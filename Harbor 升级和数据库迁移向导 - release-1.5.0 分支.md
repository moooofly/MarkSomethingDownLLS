# Harbor 升级和数据库迁移向导 - release-1.5.0 分支

> ref: https://github.com/goharbor/harbor/blob/release-1.5.0/docs/migration_guide.md

本文仅摘取关键差异部分；


----------


> If your install Harbor for the first time, or the database version is the same as that of the lastest version, you do not need any database migration.

首次安装 Harbor 或数据库版本和最新版本相同，则无需进行数据库迁移；

> NOTE:
>
> - From v1.5.0 on, the migration tool add support for the `harbor.cfg` migration, which supports upgrade from v1.2.x, v1.3.x and v1.4.x.
> - From v1.2 on, you need to use the release version as the tag of the migrator image. 'latest' is no longer used for new release.
> - You must back up your data before any data migration.
> - To migrate harbor OVA, please refer [migrate OVA guide](https://github.com/goharbor/harbor/blob/release-1.5.0/docs/migrate_ova_guide.md)

注意：

- 从 v1.5.0 开始，迁移工具增加了针对 `harbor.cfg` 迁移的支持；支持以 v1.2.x, v1.3.x 和 v1.4.x 为基础版本的迁移；
- 从 v1.2 开始，你需要使用 release version 作为 migrator 镜像的 tag ；'latest' 这个 tag 已经不再用作新 release 来使用；
- 在任何数据迁移前，你必须备份数据；


> NOTE: Upgrade from harbor 1.2 or older to harbor 1.3 must use `vmware/migratorharbor-db-migrator:1.2`. Because DB engine replaced by `MariaDB` in harbor 1.3

注意：从 harbor 1.2 或之前的版本升级到 harbor 1.3 必须使用 `vmware/migratorharbor-db-migrator:1.2` ，因为 DB 引擎在 harbor 1.3 中被替换成了 `MariaDB` ；

- **Back up** database/`harbor.cfg` to a directory such as `/path/to/backup`.

```
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg -v ${backup_path}:/harbor-migration/backup vmware/harbor-migrator:[tag] backup
```

> NOTE: By default, the migrator handles the backup for DB and CFG. If you want to backup DB or CFG only, refer to the following commands:

```
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql -v ${backup_path}:/harbor-migration/backup vmware/harbor-migrator:[tag] --db backup

docker run -it --rm -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg -v ${backup_path}:/harbor-migration/backup vmware/harbor-migrator:[tag] --cfg backup
```

默认情况下，migrator 会同时备份 DB 和 CFG ；可以通过上述两个命令进行单独备份；


- **Upgrade** database schema, `harbor.cfg` and migrate data.

```
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg vmware/harbor-migrator:[tag] up
```

> NOTE: By default, the migrator handles the upgrade for DB and CFG. If you want to upgrade DB or CFG only, refer to the following commands:

```
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql vmware/harbor-migrator:[tag] --db up

docker run -it --rm -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg vmware/harbor-migrator:[tag] --cfg up
```

> NOTE: Some errors like

```
[ERROR] Missing system table mysql.roles_mapping; please run mysql_upgrade to create it
[ERROR] Incorrect definition of table mysql.event: expected column 'sql_mode' at position ... ...
[ERROR] mysqld: Event Scheduler: An error occurred when initializing system tables. Disabling the Event Scheduler.
[Warning] Failed to load slave replication state from table mysql.gtid_slave_pos: 1146: Table 'mysql.gtid_slave_pos' doesn't exist
```

> will be occurred during upgrading **from harbor 1.2 to harbor 1.3**, just ignore them if harbor can start successfully.

可以忽略的错误信息；

> NOTE: **Rollback from harbor 1.3 to harbor 1.2** should delete `/data/database` directory first, then create new database directory by `docker-compose up -d && docker-compose stop`. And must use `vmware/harbor-db-migrator:1.2` to restore. Because of DB engine change.

从 harbor 1.3 回滚到 harbor 1.2 要求：先删除 `/data/database` 目录，之后通过 `docker-compose up -d && docker-compose stop` 创建新的数据库目录；同时必须使用 `vmware/harbor-db-migrator:1.2` 进行恢复；


> Use `test` command to test mysql connection:

```
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg vmware/harbor-migrator:[tag] test
```

可以使用 `test` 命令测试 mysql 连接；
