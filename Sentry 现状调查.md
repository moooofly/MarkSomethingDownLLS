# Sentry 现状调查

## 环境差异对比

| 环境 | 安装目录 | 安装方式 | 安装版本 | Nginx | NOTE |
| ---- | -------- | -------- | -------- | ---- | -- |
| sentry-staging (staging) | /opt/apps/onpremise | a. 本地构建 docker 镜像<br>b. 基于 docker-compose 构建服务<br>c.直接使用本地基于 docker 运行的 postgres 、memcached 和 redis 服务 | Sentry 8.22.0 | 监听 80 端口 | 新搭建一套 staging 环境，老的 staging 环境成为 dev 环境 |
| zipkin-stag (dev) | /home/deployer/sentry/onpremise | a. 本地构建 docker 镜像<br>b. 基于 docker-compose 构建服务（docker-compose 被安装在了 `/home/deployer/.local/bin/docker-compose` ） | Sentry 8.14.1 | 监听 80 端口 | 存在贤三用于进行 ldap 测试的改动<br><br>老 staging 环境，现作为 dev 环境使用 |
| sentry-prod (prod) | /opt/docker/deploy | a. 基于 harbor 上的 sentry 镜像构建<br>b. 直接使用 AWS 上的 postgres 、memcached 和 redis 服务<br> | Sentry 8.15.0 | 监听 80 端口 | 发现 ens3 为宿主机网口名<br>发现宿主机上还运行了一个 pg 的容器（据说没啥用处） |
| sentry-frontend-prod (prod) | /opt/docker/deploy | a. 基于 harbor 上的 sentry 镜像构建<br>b. 直接使用 AWS 上的 postgres 和 redis 服务<br>c. 未使用 memcached（据说不使用的话，则直接用 redis） | Sentry 8.15.0 | 监听 80 和 443 端口 | sentry_web-8.14/Dockerfile 和 sentry_web-8.14/sentry.conf 被用于定制化调整，本地 test 镜像即基于此创建<br>发现 shell 脚本中使用了奇怪的 ens3 名（后确认为贤三直接拷贝另一台机器配置导致） |
| ubuntu-1604 | /opt/apps/onpremise | a. 本地构建 docker 镜像<br>b. 基于 docker-compose 构建服务 | Sentry 8.22.0 | 未安装 | 我自己的 SandBox |

> 安装版本信息取自 Web UI 上的显示；

- staging

```
root@zipkin-stag:/home/deployer/sentry/onpremise# docker images --format "table {{.Repository}}\t{{.Tag}}"
REPOSITORY                                TAG
onpremise_base                            latest         -- 本地构建
onpremise_cron                            latest         -- 本地构建
onpremise_web                             latest         -- 本地构建
onpremise_worker                          latest         -- 本地构建
sentry                                    8.14-onbuild   -- 直接拉取
postgres                                  9.5            -- 直接拉取
memcached                                 1.4            -- 直接拉取
tianon/exim4                              latest         -- 直接拉取
redis                                     3.2-alpine     -- 直接拉取
root@zipkin-stag:/home/deployer/sentry/onpremise#
```

- 拉取 sentry 8.14-onbuild 作为基础镜像；
- 在前者基础上本地构建 onpremise_base/onpremise_cron/onpremise_web/onpremise_worker 镜像；
- 拉取 postgres/memcached/exim4/redis 作为辅助服务镜像；


- frontend-prod

```
root@sentry-frontend-prod:/opt/docker/deploy# docker images --format "table {{.Repository}}\t{{.Tag}}"
REPOSITORY                                TAG
tianon/exim4                              latest         -- 直接拉取
prod-reg.llsops.com/platform-rls/sentry   8.14           -- 直接拉取（应该就是 staging 环境中拉取的 sentry 8.14-onbuild 镜像，之后推送到 harbor 作为基础版本）
root@sentry-frontend-prod:/opt/docker/deploy#
```

- prod

```
root@sentry-prod:/opt/docker/deploy# docker images --format "table {{.Repository}}\t{{.Tag}}"
REPOSITORY                                TAG
prod-reg.llsops.com/platform-rls/sentry   8.14           -- 直接拉取（应该是本地调整后构建的）
postgres                                  9.5            -- 直接拉取
tianon/exim4                              latest         -- 直接拉取
root@sentry-prod:/opt/docker/deploy#
```

- ubuntu-1604

```
[#362#root@ubuntu-1604 /opt/apps/onpremise]$docker images --format "table {{.Repository}}\t{{.Tag}}"
REPOSITORY                                         TAG
onpremise_cron                                     latest
onpremise_worker                                   latest
onpremise_base                                     latest
onpremise_web                                      latest
tianon/exim4                                       latest
sentry                                             8.22-onbuild
postgres                                           9.5
redis                                              3.2-alpine
memcached                                          1.4
```

## Q&A

### 已解决

- harbor 上目前的 sentry 镜像的来源？

> 应该就是 staging 环境中拉取的 sentry 8.14-onbuild 镜像，之后推送到 harbor 作为基础版本；


- 为何要将 frontend 和 backend 的 Sentry 分开？

> 讨论后认为直接复用当前后端已经搭建的 Sentry 服务不合适**，因为后端代码比前端代码要敏感，因此隔离两套环境比较合适；
>
> 后端 Sentry 只对支持**内网**访问，前端支持**外网**访问（支持公网能上报异常的需求）；


- 为何多个环境的配置和使用姿势不同？

> 发展中公司的必经之路（？）


- sentry-frontend-prod 环境使用的 pg 是 aws 的，sentry-prod 环境使用的 pg 是本地镜像跑的（看到本地有跑 pg 的容器），为何这样设置？

> 据说这个 pg 容器应该是没有被使用的，然后还在跑着；


- 为何 zipkin-stag 环境中的 sentry 是使用了 memcached ，另外两个环境都没有？

> 哦，那应该就没有使用。直接用的redis ；配置的时候这是可选项，如果不配置，就会使用redis


- sentry-frontend-prod 环境中 nginx 提供 HTTPS 服务的证书问题

> 证书用的certbot ;其实我们买了证书，但是配置到elb上比较方便。配置到ec2上比较麻烦，所以就用了这个;是我觉得要配置到机器上，还需要找他们要证书什么的，比较麻烦;他们应该都不知道


- 9000 端口问题

> Sentry 默认监听 9000 端口，官方说，在常规情况下变更 Sentry 监听端口到 80 不太容易，故建议使用 proxy 来解决问题；
>
> 目前配置 nginx 监听 80 和/或 443 端口解决上述问题；
> 
> 发现当前既存在基于 9000 端口的 URL 访问，也存在基于 80 端口的 URL 访问；因此有必要统一入口访问方式，并修正[文档](https://phab.llsapp.com/w/engineer/platform/portal/)中的内容；
>


- Sentry 的 Saas 服务是指官方提供的服务么？

> 基于云主机的 saas 版本 Sentry



### 未解决

- Sentry 的两种部署方式的选择？

> on-promise 方式 vs 源码（？）

- 为什么 staging 环境名字对应 [platform/zipkin](https://git.llsapp.com/ops/rasen/tree/master/platform/zipkin) ，而 prod 环境对应 [platform/sentry](https://git.llsapp.com/ops/spiral/tree/master/platform/sentry)

> zipkin 和 sentry 之间有什么必然关系么？



## 当前需求

### 0x01 [sentry 上传文件失败排查](https://phab.llsapp.com/T42723)

### 0x02 [Sentry 的用户 URL 都是一个内网地址](https://phab.llsapp.com/T44391)

### 0x03 [sentry 集成 phab](https://phab.llsapp.com/T44225)

当前的 sentry 中没有与 phab 集成，关于出现的异常无法很快生成对应的 phab task ；

### 0x04 [sentry账号与AD集成](https://phab.llsapp.com/T40442)

TODO：本地测试 sentry 账号与 AD 集成（贤三遗留）。

> 相关：[AD集成介绍](https://phab.llsapp.com/w/engineer/ops/ad/)

### 0x05 [Sentry 邮件提醒的链接地址不正确](https://phab.llsapp.com/T46419)




## 背景信息搜集

### 0x01 [基于 Sentry 的 Saas 收集异常](https://phab.llsapp.com/T15035)

- 全局发送异常（已验证）
- 快速可上线

如果发现

- 被墙
- 收费贵
- 某些定制需求无法满足

可能再考虑切换回自己搭建 Sentry **on-Premise** 环境

### 0x02 [部署自己的 Sentry 服务](https://phab.llsapp.com/T37126)

当前用官方的 Sentry 存在两个问题：

- 收费，因此使用了较便宜的套餐，于是异常上报数量有限制；
- **上传 sourcemap 的速度**，这个直接导致 Lingome 项目中启用 sourcemap 失败

解决办法：在我们自己的服务器上部署 Sentry 服务


### 0x03 [Sentry 异常报警的区分](https://phab.llsapp.com/T18789)

当前 Sentry 的报警没有基于频次设置 threshold，导致重大异常和一般异常无法区分，容易遗漏，应该设置 threshold 区分两者

当前，所有异常都会发邮件，紧急异常（频次很高的异常会发 slack）

### 0x04 [在本地基于 Virtual Box 搭建 Sentry 服务](https://phab.llsapp.com/T14432)

基于 VirtualBox 将 Sentry 环境搭建起来，具体参见下面这片文章: 
http://dustindavis.me/setting-up-your-own-sentry-server/

> 文章有点过时，基于 python 工具安装


### 0x05 [为 LINGOME 支持 Sentry Sourcemap](https://phab.llsapp.com/T33552)

通过上传 sourcemap 文件到 Sentry 上, 使得捕捉错误时能定位到源代码位置且调用栈更加完整, 方便追踪错误.

相关：[Assets Accessible at Multiple Origins](https://docs.sentry.io/clients/javascript/sourcemaps/?_ga=2.137826352.1674430129.1501227549-1137095163.1501227549?_ga=2.137826352.1674430129.1501227549-1137095163.1501227549#assets-multiple-origins)

### 0x06 [搭建一个前端用的 Sentry 服务](https://phab.llsapp.com/T38605)

背景：

- 根据讨论的结果，**认为直接复用当前后端已经搭建的 Sentry 服务不合适**，因为后端代码比前端代码要敏感，因此直接隔离两套比较合适；
- 后端 Sentry 只对支持**内网**访问，前端支持**外网**访问；

需求：

- 希望部署一个只对前端的 Sentry，暴露端口以**支持公网能上报异常的需求**；
- 在 Task 上附上该 Sentry 服务的访问方式；


### 0x07 [AD集成介绍](https://phab.llsapp.com/w/engineer/ops/ad/)

由于基础平台系统较多（gitlab，grafana，sentry，spinnaker等），各系统都有比较独立的账号系统，独立维护比较复杂。
为了减少对账号的维护，会将部分系统与公司 AD 域集成。

Sentry 使用的是 https://github.com/Banno/getsentry-ldap-auth

需要重新生成镜像：

在 Dockerfile 中添加

```
...
RUN apt-get update && apt-get install -y libsasl2-dev python-dev libldap2-dev libssl-dev
RUN pip install sentry-ldap-auth
...
```

sentry.conf.py 中添加：

```
import ldap
from django_auth_ldap.config import LDAPSearch, GroupOfUniqueNamesType
import logging
logger = logging.getLogger('django_auth_ldap')
logger.addHandler(logging.StreamHandler())
logger.setLevel('DEBUG')
#
#
AUTH_LDAP_SERVER_URI = 'ldap://101.132.160.22'  //修改
AUTH_LDAP_BIND_DN = "cn=luoxsan,cn=Users,dc=liulishuo01,dc=partner,dc=onmschina,dc=cn"  //修改
AUTH_LDAP_BIND_PASSWORD = 'xxxx'    //修改
#
AUTH_LDAP_USER_SEARCH = LDAPSearch(
    'dc=liulishuo01,dc=partner,dc=onmschina,dc=cn',   //修改
    ldap.SCOPE_SUBTREE,
    '(&(objectClass=user)(objectClass=person)(cn=%(user)s))',   //修改
)
#
AUTH_LDAP_GROUP_SEARCH = LDAPSearch(
    'dc=liulishuo01,dc=partner,dc=onmschina,dc=cn',   //修改
    ldap.SCOPE_SUBTREE,
    '(objectClass=group)'   //修改
)
#
AUTH_LDAP_GROUP_TYPE = GroupOfUniqueNamesType()
AUTH_LDAP_REQUIRE_GROUP = None
AUTH_LDAP_DENY_GROUP = None

AUTH_LDAP_USER_ATTR_MAP = {
    'name': 'cn',     //修改
    'email': 'mail'   //修改
}
#
AUTH_LDAP_FIND_GROUP_PERMS = False
AUTH_LDAP_CACHE_GROUPS = True
AUTH_LDAP_GROUP_CACHE_TIMEOUT = 3600
#
AUTH_LDAP_DEFAULT_SENTRY_ORGANIZATION = u'Sentry'
AUTH_LDAP_SENTRY_ORGANIZATION_ROLE_TYPE = 'member'
AUTH_LDAP_SENTRY_ORGANIZATION_GLOBAL_ACCESS = True
#
AUTHENTICATION_BACKENDS = AUTHENTICATION_BACKENDS + (
    'sentry_ldap_auth.backend.SentryLdapBackend',
)
```

### 0x08 [Micro Service Introduction](https://phab.llsapp.com/w/engineer/microservice/)

kibana 是对日志流进行收集，大量的 info 日志可能会干扰我们查看一些异常。如果将程序里的一些异常发送给 sentry 服务器，在 sentry
服务器上就可以方便我们查看了，并且 sentry 也支持报警。注意：sentry 不能用来当做 log 收集平台使用，因为会产生大量的网络操作，影响服务的运行效率，也会严重依赖网络。


使用方法请参考：

- https://docs.sentry.io/clients/go/
- https://github.com/getsentry/raven-go



----------

## 网络文章汇总

### [Sentry - 实时日志收集和聚合平台](https://mp.weixin.qq.com/s/KMdzM-nSQIXbVlvUfHiWDg)

Sentry’s real-time error tracking gives you insight into production deployments and information to reproduce and fix crashes. 

> Apache Sentry 是一个 hadoop 组件

Sentry 目前在 Python 圈子里面应用比较广泛，它是基于 Django 实现的，对 Python 框架集成完善。

**为什么要使用 Sentry ?**

- Stop hoping your users will report errors.
- No need to see error logs from server.
- Know when things break.
- See the impact of each release.
- Quickly diagnose and fix issues.
- Sentry 是面向开发人员的：
    - 首先，一般情况下开发人员是没权限直接登录生产环境的服务器的，特别是 docker 环境下，查看日志比较繁琐。
    - 其次，有些异常是在特殊条件下才能触发的，有了 Sentry 就可以轻松的捕获到这些瞬间，同时有丰富的上下文信息，能在第一时间收集到报警信息。

Sentry 的简要架构图：

![]()

DSN 就是 Client Keys ，客户端发送日志的时候需要带上 DSN ，每个 Project 会自动生成，也可以重置。


### [创业公司简单粗暴之路：高效利用Sentry追踪日志发现问题](https://mp.weixin.qq.com/s/gD4x89JYn6Ybx0X4_WdkgQ)

基于日志排查问题时的困境：

- 文件过于分散
- 内容过于繁杂
- 解决问题的被动性

解决办法：

- **传统解决方案**：规定好统一的日志格式，将所有模块的日志进行适配之后统一管理起来，并建立相应的日志分类与报表，在检查到问题的时候通过邮件的形式通知运维（需要的时间和技术成本还是很大的，真正能提高日志利用的效率，还需要很长的规划与不断的总结）；
- **简单粗暴的方案（上 Sentry）** ：一个小时搭建整个平台，包含日志汇集、聚合、主动报警、漂亮的界面；


Sentry 如何解决问题：

- `dsn` (data source name) -- Sentry 的实际连接地址， Sentry 通过这个来知道到底将日志发送到哪里；
- `issues` & `events` -- 需要看的不是单单一条日志，而是一类日志；一些聚集的日志才能尽可能地反映整个错误的情况，即一个 issue ，而这些有关联的日志在 Sentry 这边就转化为这个 issue 的关联的 events ；
- `tags` -- Sentry 还帮我们加上了一些额外的 tags ，我们也需要自己去定义一些有用的 tags ，尽可能多的展现一个 issue 发生前的状况；
- `Sampling` -- Sentry 不是为了日志存储，也不会将所有日志都记录下来（毕竟使用关系型数据库作为持久化存储）。每个发送到 Sentry 的日志都是一个提供 issue 信息的事件（event），而每个项目发送到 Sentry 的事件都有一个**数量上限**，一旦超过这个上限 Sentry 就会**忽略掉重复的内容**。
Sentry 是我们所有日志的一个关于错误，问题的**分析子集**。体现在界面上的 events 信息，也是 Sentry 聚合之后的样本。
- `聚合策略` -- Sentry 主要按照以下几个方面，优先级从高到低进行日志事件的聚合（存在的已知问题：两个无关的事件但是 stacktrace 相同，那么 Sentry 会将它们分到同一个 issue 下）：
    - Stacktrace
    - Exception
    - Template
    - Messages
- `alerts digest` & limit -- 默认 Sentry 的 alerts 会发送邮件（也可以推送 slack）；Sentry 除了对重复的 alerts 进行抑制，还会追加一段时间内更新 issue 的摘要（digest）到下一个报警，这样用户邮件上接收到的信息会充分压缩，不用苦恼于过多的邮件。另外，每个用户可以根据自己的喜好自行配置 alerts 的时间间隔。

Sentry 的定位：

- **不是日志的替代**：Sentry 的目的是为了让我们专注于系统与程序的异常信息，目的是提高排查问题的效率，日志事件的量到达一个限制时甚至丢弃一些内容；官方也提倡正确设置 Sentry 接收的日志 level 的同时，用户也能继续旧的日志备份；
- **不是排查错误的万能工具**：Sentry 是带有一定策略的问题分析工具，以样本的形式展示部分原始日志的信息。信息不全面的同时，使用过程中也可能出现 Sentry 聚合所带来的负面影响，特别是日志记录质量不够的情况下。
- **不是传统监控的替代品**：与传统的监控系统相比，Sentry 更依赖于发出的日志报告，而另外一些隐藏的逻辑问题或者业务问题很可能是不会得到反馈的。

### [The Complete Guide to Error Tracking Tools: Rollbar vs Raygun vs Sentry vs Airbrake vs Bugsnag vs OverOps](https://blog.takipi.com/the-complete-guide-to-error-tracking-tools-rollbar-vs-raygun-vs-sentry-vs-airbrake-vs-bugsnag-vs-overops/)

> What started as a tiny bit of open source code grew to became a full-blown error monitoring tool, that identifies and debugs errors in production. You can use the insights to know whether you should spend time fixing a certain bug, or if you should roll back to a previous version.
>
> The dashboard lets you see stack traces, with support for source maps, along with detecting each error’s URL, parameters and session information. Each trace can be filtered with app, framework or raw error views.
> 
> You can catch errors in real-time as you deploy, and every error includes information about software, environment and the users themselves. If you believe your users are a key point at understanding errors, Sentry lets you prompt users for feedback whenever they encounter errors. That allows comparing their individual experience to the data you already have.

