# Harbor v1.6.0 升级和数据库迁移向导

> ref: https://github.com/goharbor/harbor/blob/c9d51f2a7534a4d63f35e865cc1510dddbd91468/docs/migration_guide.md

该文档适用于从 v1.6.0 迁移到更高的版本

> When upgrading your existing Harbor instance to a newer version, you may need to **migrate the data in your database and the settings in `harbor.cfg`**. Since the migration may alter the database schema and the settings of `harbor.cfg`, you should always back up your data before any migration.

- 升级已存在的 harbor 实例到更新的版本时，需要迁移数据库中的数据和 `harbor.cfg` 的配置；因为迁移过程会导致数据库 schema 以及 `harbor.cfg` 中的配置发生变更；
- 进行迁移前一定要进行数据库的备份；

> NOTE:
> 
> - Again, you must **back up your data** before any data migration.
> - This guide only covers the **migration from v1.6.0 to current version**, if you are upgrading from earlier versions please refer to the migration guide in release branch to upgrade to v1.6.0 and follow this guide to do the migration to later version.
> - From v1.6.0 on, Harbor will automatically try to do the migrate the DB schema when it starts, so **if you are upgrading from v1.6.0 or above it's not necessary to call the migrator tool to migrate the schema**.
> - From v1.6.0 on, **Harbor migrates DB from `MariaDB` to `PostgreSQL`**, and combines Harbor, Notary and Clair DB into one.
> - For the change in Database schema please refer to change log.

注意：

- 数据迁移前一定要进行备份；
- 该向导仅覆盖了从 v1.6.0 迁移到最新版本的步骤；如果您打算基于更早的版本进行迁移，则需要参考对应 release 分支中的迁移向导，先完成向 v1.6.0 版本的迁移；之后在基于当前向导迁移到更新的版本；
- 从 v1.6.0 版本开始，Harbor 将在启动后自动进行 DB schema 迁移的尝试，因此，如果你打算你从 v1.6.0 或更高的版本进行迁移，则没有必要再调用迁移工具进行 schema 迁移了；
- 从 v1.6.0 开始，Harbor 将 DB 从 `MariaDB` 变更为 `PostgreSQL` ，并且将 `Harbor`, `Notary` 和 `Clair` 的 DB 合并成了一个（即均使用 PostgreSQL）；
- 具体的数据库 schema 变更详见 [change log](https://github.com/goharbor/harbor/blob/c9d51f2a7534a4d63f35e865cc1510dddbd91468/tools/migration/db/changelog.md) ；

## Upgrading Harbor and migrating data

- 登录 + 停止运行中的 Harbor

```
cd harbor
docker-compose down
```

- 备份 Harbor 相关文件以便必要时回滚

```
mv harbor /my_backup_dir/harbor
```

- 备份数据库（默认位于 `/data/database`）

```
cp -r /data/database /my_backup_dir/
```

- 获取最新 Harbor 发布包：https://github.com/goharbor/harbor/releases

- 进行 Harbor 升级前需要先完成数据库迁移操作；迁移工具通过 docker image 提供，需要从 docker hub 上进行下载；在使用如下命令时，替换 `[tag]` 为指定的 release version（例如 v1.5.0）：

```
docker pull goharbor/harbor-migrator:[tag]
```

- 升级 `harbor.cfg` 文件；注意：`${harbor_cfg}` 会被覆盖，因此你必须将先备份，之后再拷贝回安装目录；

```
docker run -it --rm -v ${harbor_cfg}:/harbor-migration/harbor-cfg/harbor.cfg goharbor/harbor-migrator:[tag] --cfg up
```

注意：schema upgrade 和数据库的数据迁移均在 Harbor 启动时由 adminserver 完成，如果迁移发生了失败，请查看 adminserver 的日志进行 debug ；

- 进入 `./harbor` 目录，运行 `./install.sh` 脚本以安装新的 Harbor 实例；如果你打算安装具有 `Notary`, `Clair` 和 `chartmuseum` 等组件的 Harbor ，请参考 [Installation & Configuration Guide](https://github.com/goharbor/harbor/blob/c9d51f2a7534a4d63f35e865cc1510dddbd91468/docs/installation_guide.md) 文档；

## Roll back from an upgrade

> For any reason, if you want to roll back to the previous version of Harbor, follow the below steps:
>
> **NOTE**: **Roll back doesn't support upgrade across v1.5.0**, like from v1.2.0 to v1.7.0. This is because Harbor changes DB to PostgreSQL from v1.7.0, the migrator cannot roll back data to MariaDB.

注意：回滚操作不支持跨 v1.5.0 版本的情况（例如从 v1.2.0 到 v1.7.0）；因为 Harbor 从 v1.7.0 开始将数据库变成了 PostgreSQL ；而迁移工具无法将数据重新回滚为 MariaDB ；

> - 停止并移除当前正在运行的 Harbor 服务

```
cd harbor
docker-compose down
```

> - 删除当前 Harbor instance 目录

```
rm -rf harbor
```

> - 恢复打算回滚的目标 Harbor 旧版本

```
mv /my_backup_dir/harbor harbor
```

> - 恢复数据库，从备份目录中拷贝数据文件到你的数据卷中，默认为 `/data/database` ；
> - 使用之前（备份）的配置重启 Harbor 服务；如果之前的 Harbor 是基于 release build 安装的，则可以执行

```
cd harbor
./install.sh
```