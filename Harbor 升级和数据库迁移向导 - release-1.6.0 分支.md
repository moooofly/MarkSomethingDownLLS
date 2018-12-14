# Harbor 升级和数据库迁移向导 - release-1.6.0 分支

> ref: https://github.com/goharbor/harbor/blob/release-1.6.0/docs/migration_guide.md

本文仅摘取关键差异部分；

----------

> NOTE:
> 
> - Please use `goharbor/harbor-migrator:1.6.3` instead if you're performing migration, as the v1.6.3 includes a fix for issue https://github.com/goharbor/harbor/issues/6465.
> - **From v1.6.0 on, Harbor migrates DB from `MariaDB` to `Postgresql`**, and combines Harbor, Notary and Clair DB into one.
> - **From v1.5.0 on**, the migration tool add support for the `harbor.cfg` migration, which supports upgrade from v1.2.x, v1.3.x and v1.4.x.
> - **From v1.2 on**, you need to use the release version as the tag of the migrator image. 'latest' is no longer used for new release.
> - You must back up your data before any data migration.
> - To migrate harbor OVA, please refer [migrate OVA guide](https://github.com/goharbor/harbor/blob/release-1.6.0/docs/migrate_ova_guide.md).

注意：

- 数据库迁移工具需要使用 [`goharbor/harbor-migrator:1.6.3`](https://hub.docker.com/r/goharbor/harbor-migrator/tags/) ；
- 从 v1.6.0 开始，Harbor 将自身使用的 DB 从 `MariaDB` 迁移成 `Postgresql` ；

## Upgrading Harbor and migrating data

> NOTE: **[Before harbor 1.5](https://hub.docker.com/r/goharbor/harbor-db-migrator/tags/)** , image name of the migration tool is `goharbor/harbor-db-migrator:[tag]`

```
docker pull goharbor/harbor-migrator:[tag]
```

> NOTE: Upgrade from harbor 1.2 or older to harbor 1.3 must use `goharbor/harbor-db-migrator:1.2`. Because DB engine replaced by MariaDB in harbor 1.3

> NOTE: **In v1.6.0, you needs to DO three sequential steps to fully migrate Harbor, `Notary` and `Clair`'s DB**. The migration of `Notary` and `Clair`'s DB depends on Harbor's DB, you need to first upgrade Harbor's DB, then upgrade `Notary` and `Clair`'s DB. The following command handles the upgrade for Harbor DB and CFG, not include `Notary` and `Clair` DB.

注意：在 v1.6.0 版本中，你需要按顺序执行如下三个步骤：先迁移 Harbor 数据库，之后才是 `Notary` 和 `Clair` 的数据库；

```
# Harbor 的 DB (MariaDB) 和 CFG 迁移
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:[tag] up
```

> NOTE: **You must run migration of Notary and Clair's DB before launch Harbor**. If you want to upgrade Notary and Clair DB, refer to the following commands:

```
# 迁移 notary-db (postgresql)
docker run -it --rm -e DB_USR=root -v /data/notary-db/:/var/lib/mysql -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up

# 迁移 clair-db (postgresql)
docker run -it --rm -v /data/clair-db/:/clair-db -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up
```

> NOTE: If you want to upgrade DB or CFG only, refer to the following commands:

```
# 迁移 Harbor 的 DB
docker run -it --rm -e DB_USR=root -e DB_PWD={db_pwd} -v ${harbor_db_path}:/var/lib/mysql goharbor/harbor-migrator:[tag] --db up

# 迁移 notary 的 DB
docker run -it --rm -e DB_USR=root -v /data/notary-db/:/var/lib/mysql -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up

# 迁移 clair 的 DB
docker run -it --rm -v /data/clair-db/:/clair-db -v /data/database:/var/lib/postgresql/data goharbor/harbor-migrator:${tag} --db up

# 迁移 Harbor 的 CFG
docker run -it --rm -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:[tag] --cfg up
```

DB 和 CFG 分开迁移的玩法；

> NOTE: Some errors like

```
[ERROR] Missing system table mysql.roles_mapping; please run mysql_upgrade to create it
[ERROR] Incorrect definition of table mysql.event: expected column 'sql_mode' at position ... ...
[ERROR] mysqld: Event Scheduler: An error occurred when initializing system tables. Disabling the Event Scheduler.
[Warning] Failed to load slave replication state from table mysql.gtid_slave_pos: 1146: Table 'mysql.gtid_slave_pos' doesn't exist
```

will be occurred during **upgrading from harbor 1.2 to harbor 1.3**, just ignore them if harbor can start successfully.

```
/usr/lib/python2.7/site-packages/psycopg2/__init__.py:144: UserWarning: The psycopg2 wheel package will be renamed from release 2.8; in order to keep installing from binary please use "pip install psycopg2-binary" instead. For details see: <http://initd.org/psycopg/docs/install.html#binary-install-from-pypi>.
```

will be occurred during **upgrading from harbor <= v1.5.0 to harbor v1.6.0**, just ignore them if harbor can start successfully.

可能遇到的错误，如果 harbor 能够成功启动，则直接忽略；

## Roll back from an upgrade

> NOTE: **Roll back doesn't support upgrade across v1.5.0, like from v1.2.0 to v1.6.0**. It's because Harbor changes DB to `Postgresql` from v1.6.0, the migrator cannot roll back data to `MariaDB`.

注意：回滚操作不支持跨 v1.5.0 版本的情况（例如从 v1.2.0 到 v1.6.0）；因为 Harbor 从 v1.6.0 开始将自身使用的数据库变成了 `PostgreSQL` ；而迁移工具无法将数据重新回滚成 `MariaDB` 格式；

