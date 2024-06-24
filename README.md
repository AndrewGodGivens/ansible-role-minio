Ansible Role: Minio Server Installation and Configuration
=========

This role install and configure [Minio](http://min.io) in a linux server by docker swarm.


Requirements
------------

None

Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

- Wheter to install or not minio server and minio client

  ```yml
  minio_install_server: true
  minio_install_client: true
  ```
- Minio server installation details

  Minio UNIX user/group
  ```yml
  minio_group: root
  minio_user: root
  ```
  Minio installation directories to place server configuration (`minio_etc_dir`), TLS certificates (`minio_cert_dir`) and user access policies (`minio_policy_dir`)
  ```yml
  minio_etc_dir: /etc/minio
  minio_cert_dir: "{{ minio_etc_dir }}/ssl"
  minio_policy_dir: "{{ minio_etc_dir }}/policy"
  ```
  Minio server IP address (`minio_server_address`), if empty server listen in all available IP addresses, and server/console listening ports (`minio_server_port` and `minio_console_port`)
  ```yml
  minio_server_port: "9091"
  minio_server_addr: ""
  minio_console_port: "9092"
  ```

  Minio admin user and password
  ```yml
  minio_root_user: ""
  minio_root_password: ""
  ```

  Minio site region
  ```yml
  minio_site_region: "eu-west-1"
  ```

  Minio data directories (`minio_server_datadirs`) and whether force the creation in case they do not exist (`minio_server_make_datadirs`)

  ```yml
  minio_server_make_datadirs: true
  minio_server_datadirs:
    - /var/lib/minio
  ```

- Minio client configuration

  Connection alias name `minio_alias` and whether validate or not SSL certificates (`minio_validate_certificates`)

  ```yml
  minio_validate_certificate: true
  minio_alias: "myminio"
  ```


  ```
  `minio_url` might be needed in case MinIO Server TLS certificates do not contain any IP Subject Alternative Names (SAN). See [MINIO_SERVER_URL environment variable definition](https://min.io/docs/minio/linux/reference/minio-server/minio-server.html#envvar.MINIO_SERVER_URL).

  ```yml
  minio_url: "https://minio.ricsanfre.com:{{ minio_server_port }}"
  ```


- Buckets to be created

  Variable `minio_buckets` create the list of provided buckets, and applying a specifc policy. For creating the buckets, a modified version of Ansible Module from Alexis Facques is used (https://github.com/alexisfacques/ansible-module-s3-minio-bucket)

  ```yml
  minio_buckets:
    - name: bucket1
      policy: read-only
    - name: bucket2
      policy: read-write
      object_lock: false
    - name: bucket3
      policy: private
      object_lock: true
  ```
  > NOTE The module use remote connection to Minio Server using Python API (`minio` python package). Role ensure that PIP is installed and install `minio` package.

  During bucket creation three types of policy can be specified: `private`, `read-only` or `read-write` buckets.

  Minio object locking can also be enabled or disabled: `true` or `false`.

- Users to be created and buckets ACLs

  Users can be automatically created using  `minio_users` variable: a list of users can be provided, each user with three variables `name` (user name), `password` (user password) and `buckets_acl` list of buckets and type of access granted to each bucket (read-only or read-write).
  The role automatically create policy json files containing the user policy statements and load them into the server.

  Predefined `read-only` and `read-write` policies, containing pre-defined access statements, can be used. Custom policies can be also defined using  `custom` policy. In this case list of access statements need to be provided.


  ```yml
  minio_users:
  - name: user1
    password: supers1cret0
    buckets_acl:
      - name: bucket1
        policy: read-write
      - name: bucket2
        policy: read-only
      - name: bucket3
        policy: custom
        custom:
          - rule: |
              "Effect": "Allow",
              "Action": [
                  "s3:GetObject",
                  "s3:DeleteObject",
                  "s3:PutObject",
                  "s3:AbortMultipartUpload",
                  "s3:ListMultipartUploadParts"
              ],
              "Resource": [
                  "arn:aws:s3:::bucket3/*"
              ]
          - rule: |
              "Effect": "Allow",
              "Action": [
                  "s3:ListBucket"
              ],
              "Resource": [
                  "arn:aws:s3:::bucket3"
              ]
  ```
 The previous configuration will create the following policy.json file for the user

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:DeleteObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::bucket1",
                "arn:aws:s3:::bucket1/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket2",
                "arn:aws:s3:::bucket2/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:GetObject",
                "s3:ListMultipartUploadParts",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::bucket3/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::bucket3"
            ]
        }
    ]
}
```

- Generate Prometheus bearer token

  ```yml
  minio_prometheus_bearer_token: false
  prometheus_bearer_token_output: "{{ minio_etc_dir }}/prometheus_bearer.json"
  ```

  Setting `minio_prometheus_bearer_token` to true, generates a file `/etc/minio/prometheus_bearer.json` which contains the result of executing the command:

  `mc admin prometheus generate myminio -json`

- Install MinIO pip library to system packages

  As noted in the `pip` module documentation:
  > Python installations marked externally-managed (as defined by PEP668)
  > cannot be updated by pip versions >= 23.0.1 without the use of a
  > virtual environment or setting the environment variable
  > `PIP_BREAK_SYSTEM_PACKAGES=1`.

  This (and any other pip environment variables) can be set with
  `minio_pip_environment_vars` like so:

  ```yml
  minio_pip_environment_var:
   PIP_BREAK_SYSTEM_PACKAGES: "1"
  ```

Dependencies
------------

None

Example Playbook
----------------

The following playbook install and configure minio server and client, enabling TLS and generating self-signed SSL certificates.
It also create some buckets and users with proper ACLs

```yml
---
- name: Install and configure Minio Server
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: ansible-role-minio

```



License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre)
Bucket creation ansible module based on module from Alexix Facques (https://github.com/alexisfacques/ansible-module-s3-minio-bucket)
