# Harbor 信息汇总

## 文档汇总

- [Harbor 升级 v0.5.0 到 v1.2.2](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%20v0.5.0%20%E5%88%B0%20v1.2.2.md)
- [Harbor 升级和数据库迁移向导 - release-1.2.0 分支](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB%E5%90%91%E5%AF%BC%20-%20release-1.2.0%20%E5%88%86%E6%94%AF.md)
- [Harbor 升级和数据库迁移向导 - release-1.5.0 分支](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB%E5%90%91%E5%AF%BC%20-%20release-1.5.0%20%E5%88%86%E6%94%AF.md)
- [Harbor 升级和数据库迁移向导 - release-1.6.0 分支](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB%E5%90%91%E5%AF%BC%20-%20release-1.6.0%20%E5%88%86%E6%94%AF.md)
- [Harbor 升级和数据库迁移向导 - master 分支](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB%E5%90%91%E5%AF%BC%20-%20master%20%E5%88%86%E6%94%AF.md)
- [Harbor 升级问题汇总](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Harbor%20%E5%8D%87%E7%BA%A7%E9%97%AE%E9%A2%98%E6%B1%87%E6%80%BB.md)


## 数据库变更

- Harbor DB 引擎 `MySQL` 在 harbor 1.3 中被替换成了 `MariaDB` ；
- 从 v1.6.0 开始，Harbor 将自身使用的 DB 从 `MariaDB` 迁移成 `Postgresql` ； 

## migrator tools 变更

- https://hub.docker.com/r/vmware/harbor-db-migrator/tags/ -- 基本已废弃，除非有人真的使用了老版本，因为这个还是 harbor 项目未进入 CNCF 前维护的内容；涉及到的 tags 有
    - 0.4.5
    - 1.2
    - 1.3
    - 1.4
- https://hub.docker.com/r/vmware/harbor-migrator/tags -- 过渡用？
    - v1.5.0
- https://hub.docker.com/r/goharbor/harbor-db-migrator/tags/ -- 这个应该是 harbor 加入 CNCF 后将之前的 migrator tools 迁移过来的内容，理论上有了这个后，上面那个就没有用了
    - 1.2
    - 1.3
    - 1.4
- https://hub.docker.com/r/goharbor/harbor-migrator/tags/ -- 这里提供了更新版本的工具
    - v1.5.0
    - v1.6.0
    - v1.6.1
    - v1.6.3

## migrator tools 的使用

[从 v1.1.1 升级到 v1.5.0](https://github.com/goharbor/harbor/issues/5745)

- 先使用 `vmware/harbor-db-migrator:1.2` 从 v1.1.1 升级到 v1.2.0
- 再使用 `vmware/harbor-migrator:v1.5.0` 从 v1.2.0 升级到 v1.5.0

[从 1.2.0 升级到高版本](https://github.com/goharbor/harbor/issues/5232)

- 首先使用 `vmware/harbor-db-migrator:1.2` 备份数据；
- 再根据目标版本的 tag 选择相应的 migrator 版本进行升级；

[从 1.2.2 升级到 1.3.0](https://github.com/goharbor/harbor/issues/3949)

- 使用 **migrator v1.2** 备份数据；
- 删除 `/data/database`
- Start and stop harbor-db v1.3.0 to initiate an empty `mariadb` instance
- 使用 **migrator v1.3** 恢复之前的数据库备份；
- 使用 **migrator v1.3** 进行数据库升级；

[从 1.2.2 升级到 1.6.0](https://github.com/goharbor/harbor/issues/6139)

- 使用 `goharbor/harbor-db-migrator:1.2` 备份数据；
- 使用 `goharbor/harbor-migrator:v1.6.0` 升级；

[从 1.5.1 升级到 1.6.0](https://github.com/goharbor/harbor/issues/6004)

- 使用 `goharbor/harbor-migrator:v1.6.0` 升级；
