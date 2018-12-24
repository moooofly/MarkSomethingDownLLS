# Harbor 升级 v1.2.2 到 v1.6.3

## 备份

```
# 方便回滚时切换
/opt/apps/harbor

# 完整的数据目录（默认位置），备份时可以将其中的 job_logs/ 下的内容删除
/data

# 在执行 backup 命令时生成，在通过 restore 命令进行回滚时使用
/path/to/backup/registry.sql
```

## 升级

```
# Log in to the host that Harbor runs on, stop and remove existing Harbor instance if it is still running
cd /opt/apps/harbor
docker-compose down

# Back up Harbor's current files so that you can roll back to the current version when it is necessary.
cd ..
cp -rf harbor /my_backup_dir/harbor

# Get the lastest Harbor release package from https://github.com/goharbor/harbor/releases
docker pull goharbor/harbor-migrator:v1.6.3
docker pull goharbor/harbor-db-migrator:1.2

# Back up database and harbor.cfg to a directory such as /path/to/backup. 
docker run -it --rm -e DB_USR=root -e DB_PWD='xxxx' -v /data/database:/var/lib/mysql -v /opt/apps/harbor/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg -v /path/to/backup:/harbor-migration/backup goharbor/harbor-migrator:v1.6.3 backup

# （最后使用的是这个）
docker run -it --rm -e DB_USR=root -e DB_PWD='xxxx' -v /data/database:/var/lib/mysql -v /opt/apps/harbor/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg -v /path/to/backup:/harbor-migration/backup goharbor/harbor-db-migrator:1.2 backup

# Upgrade database schema, harbor.cfg and migrate data.

# The following command handles the upgrade for Harbor DB and CFG, not include Notary and Clair DB.
# （最后使用的是这个）
docker run -it --rm -e DB_USR=root -e DB_PWD='xxxx' -v /data/database:/var/lib/mysql -v /opt/apps/harbor/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:v1.6.3 up

# You must run migration of Notary and Clair's DB before launch Harbor. If you want to upgrade Notary and Clair DB, refer to the following commands
# 由于我之前没有使用 notary 和 clair ，故可以直接跳过这里
docker run -it --rm -e DB_USR=root -v /data/notary-db/:/var/lib/mysql -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up

docker run -it --rm -v /data/clair-db/:/clair-db -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up

# 解压
tar zxvf harbor-offline-installer-v1.6.3.tgz
cd harbor

# 调整配置
vi harbor.cfg
#（主要调整存储使用 s3）
vi common/templates/registry/config.yml
#（主要是 timeout 配置）
vi common/templates/nginx/nginx.http.conf
vi common/templates/nginx/nginx.https.conf

# Under the directory ./harbor, run the ./install.sh script to install the new Harbor instance. If you choose to install Harbor with components like Notary and/or Clair, refer to Installation & Configuration Guide for more information. 
# https://github.com/goharbor/harbor/blob/release-1.6.0/docs/installation_guide.md
./install.sh --with-notary --with-clair --with-chartmuseum
```

测试 mysql 访问是否正常：

```
docker run -it --rm -e DB_USR=root -e DB_PWD='xxxx' -v /data/database:/var/lib/mysql -v /opt/apps/harbor/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:v1.6.3 test
```

可能会看到的错误信息

```
/usr/lib/python2.7/site-packages/psycopg2/__init__.py:144: UserWarning: The psycopg2 wheel package will be renamed from release 2.8; in order to keep installing from binary please use "pip install psycopg2-binary" instead. For details see: <http://initd.org/psycopg/docs/install.html#binary-install-from-pypi>.
```

> will be occurred during upgrading **from harbor <= v1.5.0 to harbor v1.6.0**, just ignore them if harbor can start successfully.
