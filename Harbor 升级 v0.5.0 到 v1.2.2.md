# Harbor 升级 v0.5.0 到 v1.2.2

## 相关信息梳理

### Database (mysql) 相关

- **Database**: Database stores the **meta** data of `projects`, `users`, `roles`, `replication policies` and `images`.
- **mysql**: **Database container** created from the official MySql image.
- After getting the username and password, the `token service` checks the **database** and authenticates the user by the data in the MySql database. 
- After receiving the request forwarded by Nginx, the `token service` queries the **database** to look up the user’s role and permissions to push the image. 
- **db_password**: The root password for the MySQL database used for `db_auth`. Change this password for any production use!
- **auth_mode**: The type of authentication that is used. By default, it is **db_auth**, i.e. the credentials are stored in a database. For LDAP authentication, set this to **ldap_auth**.


----------

- 移除（停止） Harbor 上运行的 containers ，但在文件系统中保留 **image** 数据和 Harbor 使用的 **database** 文件：

```
$ sudo docker-compose down -v
```

- 移除 Harbor 使用的 **database** 文件和保存在文件系统上的 **image** 数据（默认的 image storage driver 为 `filesystem`），相当于彻底重置，对应一次干净的重装：

```
$ rm -r /data/database
$ rm -r /data/registry
```

- 默认配置下 Harbor 将 **images** 保存在你的 **local filesystem** (`/data/registry`) 中；
- 默认配置下，**registry data** (`/data/registry`) 被**持久化**存储在宿主机的 `/data/` 目录下；该数据不会发生变化，即使 Harbor 容器被移除或重建；
- 另外，Harbor 使用 `rsyslog` 来收集每个容器中的日志；默认配置下，这些日志文件被保存在宿主机的 `/var/log/harbor/` 目录下；
- 在迁移过程中，可以通过 `du -shx /data/database` 确认 mysql 数据库大小；


----------


### 版本变更相关

- There's no change in schema from `0.4.5` to `0.5.0`
- There is no DB change from `0.5.0` to `1.1.2`.
- If you want to use `1.2-rc1`, you need to do the migration.


----------

## 迁移 + 升级

操作：

- 先原样安装 v0.5.0 版本
    - 配置文件调整：基于打包数据（Harbor 工作目录 /opt/apps/harbor 打包文件 + Harbor 数据目录 /data 打包文件 + 基于备份工具导出的 /path/to/backup/registry.sql 文件）安装 v0.5.0 版本后，按需调整 `harbor.cfg` 的配置内容，如 hostname, db_password, ssl_cert/ssl_cert_key, harbor_admin_password 等；
    - 证书调整：在自测环境下，需要按需重建 HTTPS 证书，即直接基于 ip 地址创建，如 10.1.0.13 ，并将该 ip 作为 crt_commonname ；同时需要调整 `harbor.cfg` 中 "Information of your organization for certificate" 下配置的相应内容（v0.5.0 版本有，v1.2.2 中没有）；另外，由于自签名证书的缘故，需要配置 `--insecure-registry 10.1.0.13` 以解决[通过 docker cli 访问 harbor 时的证书报错](https://github.com/vmware/harbor/blob/master/docs/user_guide.md#pulling-and-pushing-images-using-docker-client)问题；另外，针对你能拿到 registry's CA certificate 的情况（例如自签名证书的情况），将 ca.crt 放到 `/etc/docker/certs.d/xx.xx.xx/ca.crt` 也是是很有必要的（原 v0.5.0 环境没有这么使用，因为其使用的证书是购买的，非自签名）；
    - 原 v0.5.0 环境配置了 `--registry-mirror=http://a9d2d4bd.m.daocloud.io` ，是因为之前该机器曾作为 CI runner 使用（会进行镜像拉取），后来才用作 harbor 服务器，故该配置已经没有用处；
- 再按照下文提及的操作升级到 v1.2.2 版本；

If you run a previous version of Harbor, you may need to **update `harbor.cfg`** and **migrate the data** to fit the new database schema. For more details, please refer to [Harbor Migration Guide](https://github.com/vmware/harbor/blob/master/docs/migration_guide.md).

### Upgrading Harbor and migrating data

> 以升级到 `v1.2.2` 为例

```
# Get the lastest Harbor release package
$ mkdir -p /opt/apps
$ cd /opt/apps/
$ wget https://github.com/vmware/harbor/releases/download/v1.2.2/harbor-offline-installer-v1.2.2.tgz

# stop and remove existing Harbor instance if it is still running
# 停止构成 Harbor 服务的 container ，以避免造成数据损毁的可能
$ cd harbor
$ docker-compose down

# Back up Harbor's current files so that you can roll back to the current version when it is necessary.
# 将原始 Harbor 版本的相关内容备份走（注意：不仅可执行程序有用，配套的脚本以及 common 目录下的配置都有用）
$ cd ..
$ mv harbor /my_backup_dir/harbor

# Before upgrading Harbor, perform database migration first.
# The migration tool is delivered as a docker image, so you should pull the image from docker hub.
# 获取用于进行数据迁移的工具
# 根据想要升级的目标版本选择工具的 tag 值，详见：https://hub.docker.com/r/vmware/harbor-db-migrator/tags/
# FIXME: 这里有个疑问，按照其他文档的说法，似乎 migrator 的 tag 应该选择 0.5 才对
$ docker pull vmware/harbor-db-migrator:1.2

# Back up database to a directory such as /path/to/backup (You need to create the directory if it does not exist).
# 备份数据库内容
# /data/database 为 harbor 的 mysql 容器对应的宿主机目录 - 1
# /var/lib/mysql 为 mysql 容器内的数据库目录 - 2
# /path/to/backup 为保存 mysql 用于保存数据库备份的宿主机目录 - 4 - 该数据可用于回滚
# /harbor-migration/backup 为用于保存 mysql 数据库备份的容器内目录 - 3
# 该命令最终会在 /path/to/backup 目录下生成 registry.sql 文件
$ docker run -ti --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql -v /path/to/backup:/harbor-migration/backup vmware/harbor-db-migrator:1.2 backup

# Upgrade database schema and migrate data
# 升级数据库 schema 并进行数据迁移（基于 registry.sql 导入数据)
$ docker run -ti --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql vmware/harbor-db-migrator:1.2 up head

# Unzip the new Harbor package and change to ./harbor as the working directory. Configure Harbor by modifying the file harbor.cfg,
$ cd /opt/apps
$ tar zxvf harbor-offline-installer-v1.2.2.tgz
$ cd harbor

# Configure Harbor by modifying the file harbor.cfg, you may need to refer to the configuration files you've backed up during step 2. 
# 建议进行人工对比，手动调整该配置文件的内容
$ vi harbor.cfg

# To assist you in migrating the harbor.cfg file from v0.5.0 to v1.1.x, a script is provided and described as below. For other versions of Harbor, you need to manually migrate the file harbor.cfg.
# NOTE: After running the script, make sure you go through harbor.cfg to verify all the settings are correct. 
$ ./upgrade --source-loc source_harbor_cfg_loc --source-version 0.5.0 --target-loc target_harbor_cfg_loc --target-version 1.1.x

# Under the directory ./harbor, run the ./install.sh script to install the new Harbor instance.
$ ./install.sh
```

### Roll back from an upgrade

```
# Stop and remove the current Harbor service if it is still running.
$ cd harbor
$ docker-compose down

# Restore database from backup file in /path/to/backup .
$ docker run -ti --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql -v /path/to/backup:/harbor-migration/backup vmware/harbor-db-migrator:1.2 restore

# Remove current Harbor instance.
$ cd ..
$ rm -rf harbor

# Restore the older version package of Harbor.
$ mv /my_backup_dir/harbor harbor

# Restart Harbor service using the previous configuration.

# If previous version of Harbor was installed by a release build:
$ cd harbor
$ ./install.sh

# If your previous version of Harbor was installed from source code:
$ cd harbor
$ docker-compose up --build -d
```


----------


## 配置相关

### 原 harbor v0.5.0 配置（docker-registry-stag）

```
root@docker-registry-stag:/opt/apps/harbor# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 14.04.5 LTS
Release:	14.04
Codename:	trusty
root@docker-registry-stag:/opt/apps/harbor#
```

```
# vi /etc/default/docker
...
DOCKER_OPTS="$DOCKER_OPTS --registry-mirror=http://a9d2d4bd.m.daocloud.io"
```

```
# ps aux|grep docker
...
root     28160  1.0  7.8 3526284 303380 ?      Ssl  Apr19 3292:22 /usr/bin/dockerd --registry-mirror=http://a9d2d4bd.m.daocloud.io --raw-logs
```

之前曾因为忽略了如下配置文件的内容，而导致排查了半天 login 保存证书未知的问题；

```
root@docker-registry-stag:/opt/apps/harbor# cat harbor.cfg
## Configuration file of Harbor

#The IP address or hostname to access admin UI and registry service.
#DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname = stag-reg.llsops.com

#The protocol for accessing the UI and token/notification service, by default it is http.
#It can be set to https if ssl is enabled on nginx.
#ui_url_protocol = http
ui_url_protocol = https

#Email account settings for sending out password resetting emails.

#Email server uses the given username and password to authenticate on TLS connections to host and act as identity.
#Identity left blank to act as username.
email_identity =

email_server = smtp.163.com
email_server_port = 25
email_username = lls_sender@163.com
email_password = lls_sender_2016
email_from = admin <admin@llsops.com>
email_ssl = false

##The initial password of Harbor admin, only works for the first time when Harbor starts.
#It has no effect after the first launch of Harbor.
#Change the admin password from UI after launching Harbor.
harbor_admin_password = 8B+Ps{rV+bF*PD

##By default the auth mode is db_auth, i.e. the credentials are stored in a local database.
#Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
auth_mode = db_auth

#The url for an ldap endpoint.
ldap_url = ldaps://ldap.mydomain.com

#A user's DN who has the permission to search the LDAP/AD server.
#If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
#ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

#the password of the ldap_searchdn
#ldap_search_pwd = password

#The base DN from which to look up a user in LDAP/AD
ldap_basedn = ou=people,dc=mydomain,dc=com

#Search filter for LDAP/AD, make sure the syntax of the filter is correct.
#ldap_filter = (objectClass=person)

# The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD
ldap_uid = uid

#the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
ldap_scope = 3

#The password for the root user of mysql db, change this before any production use.
db_password = $W]HNF*PDrhqP

#Turn on or off the self-registration feature
self_registration = on

#Determine whether the UI should use compressed js files.
#For production, set it to on. For development, set it to off.
use_compressed_js = on

#Maximum number of job workers in job service
max_job_workers = 3

#Secret key for encryption/decryption of password of remote registry, its length has to be 16 chars
#**NOTE** if this changes, previously encrypted password will not be decrypted!
#Change this key before any production use.
secret_key = F*PDrhqPYTP1LR5?

#The expiration time (in minute) of token created by token service, default is 30 minutes
token_expiration = 30

#Determine whether the job service should verify the ssl cert when it connects to a remote registry.
#Set this flag to off when the remote registry uses a self-signed or untrusted certificate.
verify_remote_cert = on

#Determine whether or not to generate certificate for the registry's token.
#If the value is on, the prepare script creates new root cert and private key
#for generating token to access the registry. If the value is off, a key/certificate must
#be supplied for token generation.
customize_crt = on

#Information of your organization for certificate
crt_country = CN
crt_state = SH
crt_location = SH
crt_organization = LLS
crt_organizationalunit = DEV
crt_commonname = stag-reg.llsops.com   # 注意：这里的内容必须和创建证书的内容保持一致
crt_email = liang@liulishuo.com        # 注意：这里的内容必须和创建证书的内容保持一致

#The flag to control what users have permission to create projects
#Be default everyone can create a project, set to "adminonly" such that only admin can create project.
project_creation_restriction = everyone

#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /data/cert/reg.llsops.com.crt
ssl_cert_key = /data/cert/reg.llsops.com.key
#############
root@docker-registry-stag:/opt/apps/harbor#
```


----------


### 新 harbor v1.2.2 配置（harbor-service-new-stag）


```
root@harbor-service-new-stag:/opt/apps/harbor# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.1 LTS
Release:	16.04
Codename:	xenial
root@harbor-service-new-stag:/opt/apps/harbor#
```

```
# vi /lib/systemd/system/docker.service
...
ExecStart=/usr/bin/dockerd --insecure-registry 10.1.0.13 -H fd://
```

```
# ps aux|grep docker
...
root     31736  0.3  0.6 613632 53240 ?        Ssl  06:48   0:06 /usr/bin/dockerd --insecure-registry 10.1.0.13 -H fd://
```


----------


### 自建 vagrant 环境配置

```
[#21#root@ubuntu-1604 ~]$lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.3 LTS
Release:	16.04
Codename:	xenial
[#22#root@ubuntu-1604 ~]$
```

```
$vi /lib/systemd/system/docker.service
...
ExecStart=/usr/bin/dockerd --insecure-registry 11.11.11.12 -H fd://
```

```
$ps aux|grep odcker
...
root      4803  0.2  4.9 552864 50496 ?        Ssl  14:59   0:04 /usr/bin/dockerd --insecure-registry 11.11.11.12 -H fd://
```

```
[#12#root@ubuntu-1604 /opt/apps/harbor]$cat harbor.cfg
## Configuration file of Harbor

#The IP address or hostname to access admin UI and registry service.
#DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname = 11.11.11.12

#The protocol for accessing the UI and token/notification service, by default it is http.
#It can be set to https if ssl is enabled on nginx.
#ui_url_protocol = http
ui_url_protocol = https

#The password for the root user of mysql db, change this before any production use.
db_password = root123

#Maximum number of job workers in job service
max_job_workers = 3

#Determine whether or not to generate certificate for the registry's token.
#If the value is on, the prepare script creates new root cert and private key
#for generating token to access the registry. If the value is off the default key/cert will be used.
#This flag also controls the creation of the notary signer's cert.
customize_crt = on

#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /data/cert/harbor_test.crt
ssl_cert_key = /data/cert/harbor_test.key

#The path of secretkey storage
secretkey_path = /data

#Admiral's url, comment this attribute, or set its value to NA when Harbor is standalone
admiral_url = NA

#The password of the Clair's postgres database, only effective when Harbor is deployed with Clair.
#Please update it before deployment, subsequent update will cause Clair's API server and Harbor unable to access Clair's database.
clair_db_password = password

#NOTES: The properties between BEGIN INITIAL PROPERTIES and END INITIAL PROPERTIES
#only take effect in the first boot, the subsequent changes of these properties
#should be performed on web ui

#************************BEGIN INITIAL PROPERTIES************************

#Email account settings for sending out password resetting emails.

#Email server uses the given username and password to authenticate on TLS connections to host and act as identity.
#Identity left blank to act as username.
#email_identity =
#
#email_server = smtp.mydomain.com
#email_server_port = 25
#email_username = sample_admin@mydomain.com
#email_password = abc
#email_from = admin <sample_admin@mydomain.com>
#email_ssl = false


email_identity =

email_server = smtp.163.com
email_server_port = 25
email_username = lls_sender@163.com
email_password = lls_sender_2016
email_from = admin <admin@llsops.com>
email_ssl = false


##The initial password of Harbor admin, only works for the first time when Harbor starts.
#It has no effect after the first launch of Harbor.
#Change the admin password from UI after launching Harbor.
harbor_admin_password = Harbor12345

##By default the auth mode is db_auth, i.e. the credentials are stored in a local database.
#Set it to ldap_auth if you want to verify a user's credentials against an LDAP server.
auth_mode = db_auth

#The url for an ldap endpoint.
ldap_url = ldaps://ldap.mydomain.com

#A user's DN who has the permission to search the LDAP/AD server.
#If your LDAP/AD server does not support anonymous search, you should configure this DN and ldap_search_pwd.
#ldap_searchdn = uid=searchuser,ou=people,dc=mydomain,dc=com

#the password of the ldap_searchdn
#ldap_search_pwd = password

#The base DN from which to look up a user in LDAP/AD
ldap_basedn = ou=people,dc=mydomain,dc=com

#Search filter for LDAP/AD, make sure the syntax of the filter is correct.
#ldap_filter = (objectClass=person)

# The attribute used in a search to match a user, it could be uid, cn, email, sAMAccountName or other attributes depending on your LDAP/AD
ldap_uid = uid

#the scope to search for users, 1-LDAP_SCOPE_BASE, 2-LDAP_SCOPE_ONELEVEL, 3-LDAP_SCOPE_SUBTREE
ldap_scope = 3

#Timeout (in seconds)  when connecting to an LDAP Server. The default value (and most reasonable) is 5 seconds.
ldap_timeout = 5

#Turn on or off the self-registration feature
self_registration = on

#The expiration time (in minute) of token created by token service, default is 30 minutes
token_expiration = 30

#The flag to control what users have permission to create projects
#The default value "everyone" allows everyone to creates a project.
#Set to "adminonly" so that only admin user can create project.
project_creation_restriction = everyone

#Determine whether the job service should verify the ssl cert when it connects to a remote registry.
#Set this flag to off when the remote registry uses a self-signed or untrusted certificate.
verify_remote_cert = on
#************************END INITIAL PROPERTIES************************
#############

[#13#root@ubuntu-1604 /opt/apps/harbor]$
```

需要注意的是，在 v1.2.2 配置文件中，默认没有下面这些内容，所以在之前搭建时没有遇到 login 证书未知问题；

```
#Information of your organization for certificate
crt_country = CN
crt_state = SH
crt_location = SH
crt_organization = LLS
crt_organizationalunit = DEV
crt_commonname = stag-reg.llsops.com   # 注意：这里的内容必须和创建证书的内容保持一致
crt_email = liang@liulishuo.com        # 注意：这里的内容必须和创建证书的内容保持一致
```


----------

## 其他

### 备份操作

在未事先执行 `docker-compose down` 的情况下，执行备份操作，报错如下

```
root@harbor-service-new-stag:/opt/apps# docker run -ti --rm -e DB_USR=root -e DB_PWD='$W]HNF*PDrhqP' -v /data/database:/var/lib/mysql -v /path/to/backup:/harbor-migration/backup vmware/harbor-db-migrator:1.2 backup
Trying to start mysql server...
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:05 0 [Note] mysqld (mysqld 5.6.33) starting as process 7 ...
2017-11-24 09:44:05 7 [Note] Plugin 'FEDERATED' is disabled.
2017-11-24 09:44:05 7 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-11-24 09:44:05 7 [Note] InnoDB: The InnoDB memory heap is disabled
2017-11-24 09:44:05 7 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-11-24 09:44:05 7 [Note] InnoDB: Memory barrier is not used
2017-11-24 09:44:05 7 [Note] InnoDB: Compressed tables use zlib 1.2.8
2017-11-24 09:44:05 7 [Note] InnoDB: Using Linux native AIO
2017-11-24 09:44:05 7 [Note] InnoDB: Using CPU crc32 instructions
2017-11-24 09:44:05 7 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-11-24 09:44:05 7 [Note] InnoDB: Completed initialization of buffer pool
2017-11-24 09:44:05 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:05 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
2017-11-24 09:44:05 7 [Note] InnoDB: Retrying to lock the first data file
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:06 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:06 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:07 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:07 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:08 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:08 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:09 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:09 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:44:10 7 [ERROR] InnoDB: Unable to lock ./ibdata1, error: 11
2017-11-24 09:44:10 7 [Note] InnoDB: Check that you do not already have another mysqld process using the same InnoDB data or log files.
MySQL Community Server 5.6.33 is not running.
...
```

正确操作输出

```
root@harbor-service-new-stag:/my_backup_dir/harbor# docker run -ti --rm -e DB_USR=root -e DB_PWD='$W]HNF*PDrhqP' -v /data/database:/var/lib/mysql -v /path/to/backup:/harbor-migration/backup vmware/harbor-db-migrator:1.2 backup
Trying to start mysql server...
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:47:05 0 [Note] mysqld (mysqld 5.6.33) starting as process 7 ...
2017-11-24 09:47:05 7 [Note] Plugin 'FEDERATED' is disabled.
2017-11-24 09:47:05 7 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-11-24 09:47:05 7 [Note] InnoDB: The InnoDB memory heap is disabled
2017-11-24 09:47:05 7 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-11-24 09:47:05 7 [Note] InnoDB: Memory barrier is not used
2017-11-24 09:47:05 7 [Note] InnoDB: Compressed tables use zlib 1.2.8
2017-11-24 09:47:05 7 [Note] InnoDB: Using Linux native AIO
2017-11-24 09:47:05 7 [Note] InnoDB: Using CPU crc32 instructions
2017-11-24 09:47:05 7 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-11-24 09:47:05 7 [Note] InnoDB: Completed initialization of buffer pool
2017-11-24 09:47:05 7 [Note] InnoDB: Highest supported file format is Barracuda.
2017-11-24 09:47:05 7 [Note] InnoDB: 128 rollback segment(s) are active.
2017-11-24 09:47:05 7 [Note] InnoDB: Waiting for purge to start
2017-11-24 09:47:05 7 [Note] InnoDB: 5.6.33 started; log sequence number 45901375
2017-11-24 09:47:05 7 [Note] Server hostname (bind-address): '*'; port: 3306
2017-11-24 09:47:05 7 [Note] IPv6 is available.
2017-11-24 09:47:05 7 [Note]   - '::' resolves to '::';
2017-11-24 09:47:05 7 [Note] Server socket created on IP: '::'.
2017-11-24 09:47:05 7 [Warning] 'proxies_priv' entry '@ root@8c6c362a1c73' ignored in --skip-name-resolve mode.
2017-11-24 09:47:05 7 [Note] Event Scheduler: Loaded 0 events
2017-11-24 09:47:05 7 [Note] mysqld: ready for connections.
Version: '5.6.33'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
MySQL Community Server 5.6.33 is running.
Performing backup...
Backup performed.
root@harbor-service-new-stag:/my_backup_dir/harbor# ll /path/to/backup
total 7340
drwxr-xr-x 2 root root    4096 Nov 24 09:47 ./
drwxr-xr-x 3 root root    4096 Nov 24 09:44 ../
-rw-r--r-- 1 root root 7504179 Nov 24 09:47 registry.sql
root@harbor-service-new-stag:/my_backup_dir/harbor#
root@harbor-service-new-stag:/my_backup_dir/harbor#
root@harbor-service-new-stag:/my_backup_dir/harbor#
root@harbor-service-new-stag:/my_backup_dir/harbor#
root@harbor-service-new-stag:/my_backup_dir/harbor# docker run -ti --rm -e DB_USR=root -e DB_PWD='$W]HNF*PDrhqP' -v /data/database:/var/lib/mysql vmware/harbor-db-migrator:1.2 up head
Please backup before upgrade.
Enter y to continue updating or n to abort:y
Trying to start mysql server...
MySQL Community Server 5.6.33 is not running.
2017-11-24 09:48:06 0 [Note] mysqld (mysqld 5.6.33) starting as process 5 ...
2017-11-24 09:48:06 5 [Note] Plugin 'FEDERATED' is disabled.
2017-11-24 09:48:06 5 [Note] InnoDB: Using atomics to ref count buffer pool pages
2017-11-24 09:48:06 5 [Note] InnoDB: The InnoDB memory heap is disabled
2017-11-24 09:48:06 5 [Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
2017-11-24 09:48:06 5 [Note] InnoDB: Memory barrier is not used
2017-11-24 09:48:06 5 [Note] InnoDB: Compressed tables use zlib 1.2.8
2017-11-24 09:48:06 5 [Note] InnoDB: Using Linux native AIO
2017-11-24 09:48:06 5 [Note] InnoDB: Using CPU crc32 instructions
2017-11-24 09:48:06 5 [Note] InnoDB: Initializing buffer pool, size = 128.0M
2017-11-24 09:48:06 5 [Note] InnoDB: Completed initialization of buffer pool
2017-11-24 09:48:06 5 [Note] InnoDB: Highest supported file format is Barracuda.
2017-11-24 09:48:06 5 [Note] InnoDB: Log scan progressed past the checkpoint lsn 45901375
2017-11-24 09:48:06 5 [Note] InnoDB: Database was not shutdown normally!
2017-11-24 09:48:06 5 [Note] InnoDB: Starting crash recovery.
2017-11-24 09:48:06 5 [Note] InnoDB: Reading tablespace information from the .ibd files...
2017-11-24 09:48:06 5 [Note] InnoDB: Restoring possible half-written data pages
2017-11-24 09:48:06 5 [Note] InnoDB: from the doublewrite buffer...
InnoDB: Doing recovery: scanned up to log sequence number 45901385
2017-11-24 09:48:06 5 [Note] InnoDB: 128 rollback segment(s) are active.
2017-11-24 09:48:06 5 [Note] InnoDB: Waiting for purge to start
2017-11-24 09:48:06 5 [Note] InnoDB: 5.6.33 started; log sequence number 45901385
2017-11-24 09:48:06 5 [Note] Server hostname (bind-address): '*'; port: 3306
2017-11-24 09:48:06 5 [Note] IPv6 is available.
2017-11-24 09:48:06 5 [Note]   - '::' resolves to '::';
2017-11-24 09:48:06 5 [Note] Server socket created on IP: '::'.
2017-11-24 09:48:06 5 [Warning] 'proxies_priv' entry '@ root@8c6c362a1c73' ignored in --skip-name-resolve mode.
2017-11-24 09:48:06 5 [Note] Event Scheduler: Loaded 0 events
2017-11-24 09:48:06 5 [Note] mysqld: ready for connections.
Version: '5.6.33'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
MySQL Community Server 5.6.33 is running.
Performing upgrade head...
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
0.4.0
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade 0.4.0 -> 1.2.0, 0.4.0 to 1.2.0
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
1.2.0 (head)
Upgrade performed.
0
root@harbor-service-new-stag:/my_backup_dir/harbor#
```


----------


### Migration tool reference

迁移工具提供 `help` 命令以方便查看可用指令：

```
docker run --rm -e DB_USR=root -e DB_PWD=xxxx vmware/harbor-db-migrator:[tag] help
```

提供 `test` 命令用于测试  mysql 连接：

```
docker run --rm -e DB_USR=root -e DB_PWD=xxxx -v /data/database:/var/lib/mysql vmware/harbor-db-migrator:[tag] test
```

## 参考

- [Installation and Configuration Guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)
- [Architecture Overview of Harbor](https://github.com/vmware/harbor/wiki/Architecture-Overview-of-Harbor)
- [Harbor upgrade and database migration guide](https://github.com/vmware/harbor/blob/master/docs/migration_guide.md)
- [Changelog for harbor database schema](https://github.com/vmware/harbor/blob/master/tools/migration/changelog.md)
