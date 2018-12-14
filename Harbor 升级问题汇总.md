# Harbor 升级问题汇总

## [where I can get goharbor/harbor-db-migrator:1.6.3 as per release-1.6.0/docs/migration_guide.md](https://github.com/goharbor/harbor/issues/6528)

migrator tools 的变更历史：

- https://hub.docker.com/r/vmware/harbor-db-migrator/tags/ -- almost deprecated, but can work well in some cases, not suggest to use. It is a hub keeping migrator tools before harbor entering into CNCF. Tags involved:
    - 0.4.5
    - 1.2
    - 1.3
    - 1.4
- https://hub.docker.com/r/vmware/harbor-migrator/tags -- for what use?
    - v1.5.0
- https://hub.docker.com/r/goharbor/harbor-db-migrator/tags/ -- This hub is used to keep migrator tools after harbor entering into CNCF, as you can see, the contents are almost the same as above. So it is just a copy. Tags involved:
    - 1.2
    - 1.3
    - 1.4
- https://hub.docker.com/r/goharbor/harbor-migrator/tags/ -- This hub keeps the most recently migrator tools.  Tags involved:
    - v1.5.0
    - v1.6.0
    - v1.6.1
    - v1.6.3

## [upgrade harbor from 1.2.2 to 1.6.0，but occur error while database backup](https://github.com/goharbor/harbor/issues/6139)

要点：

- please use the [`goharbor/harbor-db-migrator:1.2`](https://hub.docker.com/r/goharbor/harbor-db-migrator/tags) to **backup** data, and follow the [guide on branch release-1.2.0](https://github.com/goharbor/harbor/blob/release-1.2.0/docs/migration_guide.md)
- "**How should I rollback from 1.6.0 to 1.2.2** because the guide said don't support it.", the harbor doesn't support rollback (这里应该是指“不支持从 1.6.0 回滚到 1.2.2 版本，因为底层数据库发生了变更”). But, I think you can do it manually. Just clean your data folder and import the dumped data into Db.
- "**import the dumped data into Db** -- Do you mean that login the db container and exec the dumped sql ?". The easiest way to backup data is just to copy files. But, I suggest you just use the `harbor-db-migrator:1.2` to backup `MySQL` data. If any issue during upgrade, you can rollback the data with MySQL command in the DB container, no need to login.

这里没有回答清楚究竟应该使用哪个 hub 上的 1.2 ；已经在 issue 中提问；

## [Upgrade from v1.3.0-rc4 to v1.6.3 fails](https://github.com/goharbor/harbor/issues/6523)

By default, the `harbor.cfg` used by the `install.sh` is the one in the offline installer, please either to modify it or to replace it with upgraded one.

> 疯狂更新中

## [Unknown error has occured - Error Message](https://github.com/goharbor/harbor/issues/5975)

问题现象：

- 在 ui.log 中

```
...
Oct  1 13:44:42 172.18.0.1 ui[597]: 2018-10-01T11:44:42Z [WARNING] Harbor is not deployed with Clair, it's not impossible to get vulnerability details.
Oct  1 13:44:42 172.18.0.1 ui[597]: 2018/10/01 11:44:42 #033[1;44m[D] [server.go:2619] |     10.3.0.133|#033[41m 503 #033[0m|    701.153µs|   match|#033[44m GET     #033[0m /api/repositories/windows/visual-studio-2017-build-tools/tags/latest/vulnerability/details   r:/api/repositories/*/tags/:tag/vulnerability/details#033[0m
...
```

- 在 proxy.log 中

```
Oct 18 08:37:17 172.18.0.1 proxy[616]: 10.3.0.133 - "GET /api/repositories/windows/visual-studio-2017-build-tools/tags/latest/vulnerability/details HTTP/1.1" 503 1 "https://10.100.152.165/harbor/projects/5/repositories/windows%2Fvisual-studio-2017-build-tools/tags/latest" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.100 Safari/537.36" 0.000 0.000 .
```

> - This error message appears after **upgrading harbor from v1.1.2 to v1.5.0**.
> - There were no errors while updating habor.
> - I tried to **Update harbor from v1.5.0 to v1.6.0** but the error message still appears.

通过 web console 可以看到调用 /vulnerability/details 这个 API 时触发了 503 ；


问题原因：

> This 503 issue was because UI tries to **get information from Clair while Clair is not installed**, this should be fixed in 1.6, could you confirm you still see this issue after 1.6.0?

> So the error was because the browser tries to load the vulnerability of an image when Clair is not installed, **this was a known issue in 1.5 and fixed in 1.6**

> If you keep seeing this in 1.6, it's **maybe due to browser cache**, could you clean up the cache and retry? If the problem persists please upload the complete logs again.


## [Is it possible to migrate from 1.5.1 to 1.6.0 version](https://github.com/goharbor/harbor/issues/6004)

you can **use `goharbor/harbor-migrator:v1.6.0` to migrate from 1.5.1 to 1.6.0**

## [harbor upgrade directly from v1.1.1 to v1.5.0, only to v1.4.0](https://github.com/goharbor/harbor/issues/5745)

问题现象：

> I am using harbor v1.1.1 and I want to upgrade to v1.5.0
>
> I was upgrading harbor using the command below, but it just upgrade to v1.4.0.

```
docker run -it --rm -e DB_USR=root -e DB_PWD=root123 -v /data/database:/var/lib/mysql -v /root/harbor_test/harbor/harbor.cfg:/harbor-migration/harbor-cfg/harbor.cfg vmware/harbor-migrator:v1.5.0 up
```

问题原因：

> Can you please use the v1.2.0 migrator to upgrade to v1.2.0 DB scheme firstly?
>
> And then to upgrade to v1.5.0, it seems that there is a conflict in the alchemy framework.

## [No image to migrate from v1.2.0 to higher version for Harbor](https://github.com/goharbor/harbor/issues/5232)

> So you should probably try to use `vmware/harbor-db-migrator:1.2` (**only for backing up your DB**)
> 
> If you want to upgrade your DB, use the command from the upgrade notes with the matching tag (for example "v1.5.0" for version 1.5)

> The `vmware/harbor-db-migrator:1.2` image is just used for backup

## [DB schema upgrade problem from 1.2.2 to 1.3.0](https://github.com/goharbor/harbor/issues/3949)

> you should 
>
> - （这里缺失了 backup 步骤）
> - delete `/data/database` dir first. 
> - And start harbor with empty MySQL. 
> - Then restore with migrator v1.2

这个是 vmware 的人给的方法，然后说的不清楚，也不完整；

> I faced the same problem last week. It works with the following process:
> 
> - **Backup** your database with **migrator v1.2**
> - Delete `/data/database`
> - Start and stop harbor-db v1.3.0 to initiate an empty mariadb instance
> - **Restore** your backup with **migrator v1.3**
> - Upgrade with migrator v1.3

这是另外一个人给的完整步骤，这个应该是 ok 的；
