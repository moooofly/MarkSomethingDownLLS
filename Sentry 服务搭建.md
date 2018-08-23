# Sentry 服务搭建

## [Server Installation](https://docs.sentry.io/server/installation/)

> Before running Sentry itself, there are a few minimum services that are required for Sentry to communicate with.

最小依赖服务集；

### Services

- **PostgreSQL**
    - Docker image `postgres:9.5`
- **Redis** (the minimum version requirement is 2.8.9, but 2.8.18, 3.0, or newer are recommended)
    - If running Ubuntu < 15.04, you’ll need to install from a different PPA. We recommend `chris-lea/redis-server`
    - Docker image `redis:3.2-alpine`.
- **A dedicated (sub)domain** to host Sentry on (i.e. sentry.yourcompany.com).

### Hardware

The main things you need to consider are:

- **TTL on events** (how long do you need to keep historical data around)
- **Average event throughput**
- **How many events get grouped together** (which means they get sampled)

### Installing Sentry Server

We support two methods of installing and running your own Sentry server. Our **recommended approach** is to use Docker, but if that’s not a supported environment, you may also setup a traditional Python environment.

- Via **Docker**
- Via **Python**


----------


## Installation with Docker

需要注意两点：

- 第一，“基于 Makefile 进行 Sentry 服务管理”包括：
    - 本地基于 Dockerfile 进行镜像构建；
    - 依赖服务的直接拉取；
    - 服务之间的依赖关系的建立和管理（通过相应的命令实现）；
    - 要求容器命名、启动命令等信息均按照相应的模式编写；
- 第二，“基于 docker-compose 进行 Sentry 服务管理”，其基本覆盖了 Makefile 中的功能，而且管理起来更为简单；需要注意容器命名、启动命令等信息和前者有所不同；

### 基于 Makefile 进行 Sentry 服务管理

> https://docs.sentry.io/server/installation/docker/

This guide will step you through setting up your own **on-premise** Sentry in Docker.

**Dependencies**: Docker version 1.10+

#### Building Container

- [`getsentry/onpremise`](https://github.com/getsentry/onpremise): 基于该 github repo 中的内容构建的镜像是你定制化镜像的 base 版本
- `sentry.conf.py` and `config.yml`: 用于定制化配置
- `requirements.txt`: 用于安装插件
- `REPOSITORY=registry.example.com/sentry make build push`: 构建定制化镜像
- If you plan on running the dependent services in Docker as well, we support linking containers.

#### Running Dependent Services

可以基于 Docker 运行 Sentry 的依赖服务，也可以直接运行相应的依赖服务；基于 Docker 运行时利用了 linking containers ；


- **Redis**

```
docker run \
  --detach \
  --name sentry-redis \
  redis:3.2-alpine
```

- **PostgreSQL**

```
docker run \
  --detach \
  --name sentry-postgres \
  --env POSTGRES_PASSWORD=secret \
  --env POSTGRES_USER=sentry \
  postgres:9.5
```

- **Outbound Email**

```
docker run \
  --detach \
  --name sentry-smtp \
  tianon/exim4
```

> 这里没有提及 `memcached` 服务，但在 `docker-compose.yml` 文件中能够看到相应的配置；

#### Running Sentry Services

> **Note**
> 
> The image that is built, acts as the entrypoint for all running pieces for the Sentry application and the **same image** must be used for all containers.
> 
> If different components are running with different configurations, Sentry will likely have unexpected behaviors.

上文意思是：基于 Makefile 中 `docker build --rm -t $(REPOSITORY):$(TAG) .` 命令构建的镜像，将作为构成 Sentry 应用的所有组成部分的 entrypoint 而存在（即基础镜像），同时必须保证构成 Sentry 应用的所有组成部分使用相同的基础镜像；这段话的含义也可以从 `docker-compose.yml` 中依赖关系定义看出；

```
[#308#root@ubuntu-1604 /opt/apps/onpremise]$cat Makefile
REPOSITORY?=sentry-onpremise
TAG?=latest

OK_COLOR=\033[32;01m
NO_COLOR=\033[0m

build:
	@echo "$(OK_COLOR)==>$(NO_COLOR) Building $(REPOSITORY):$(TAG)"
	@docker build --rm -t $(REPOSITORY):$(TAG) .

$(REPOSITORY)_$(TAG).tar: build
	@echo "$(OK_COLOR)==>$(NO_COLOR) Saving $(REPOSITORY):$(TAG) > $@"
	@docker save $(REPOSITORY):$(TAG) > $@

push: build
	@echo "$(OK_COLOR)==>$(NO_COLOR) Pushing $(REPOSITORY):$(TAG)"
	@docker push $(REPOSITORY):$(TAG)

all: build push

.PHONY: all build push
[#309#root@ubuntu-1604 /opt/apps/onpremise]$
```

一些可以借鉴使用的命令：

> `${REPOSITORY}` 的值默认为 Makefile 中的 sentry-onpremise

- 测试镜像能正常工作

```
docker run --rm ${REPOSITORY} --help
```

- 生成一个 `secret-key` 值（可以将该值保存到 `config.yml` 中，或者作为环境变量 `SENTRY_SECRET_KEY` 的值；如果采用的是前者，则必须 **rebuild** 你的镜像）

```
docker run --rm ${REPOSITORY} config generate-secret-key
```

- The **base** for running commands (For all future Sentry command invocations, you must have all the necessary **container links**, **mounted volumes**, and the **same environment variables**)

```
docker run \
  --detach \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  ${REPOSITORY} \
  <command>
```

> 此处的 link 关系可以对比一下 `docker-compose.yml` 中的定义；


以下命令均基于上面的 base 命令：

- **Running Migrations** (During the upgrade, you will be prompted to create the **initial user** which will act as the **superuser**.)

```
docker run \
  --rm \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  -it \
  ${REPOSITORY} \
  upgrade
```

- **Starting the Web Service** (test the web service by visiting http://localhost:9000/)

```
docker run \
  --detach \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  --name sentry-web-01 \
  --publish 9000:9000 \
  ${REPOSITORY} \
  run web
```

- **Starting Background Workers** (A large amount of Sentry’s work is managed via background workers)

```
docker run \
  --detach \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  --name sentry-worker-01 \
  ${REPOSITORY} \
  run worker
```

详见：[Asynchronous Workers](https://docs.sentry.io/server/queue/)

- **Starting the Cron Process**

```
docker run \
  --detach \
  --link sentry-redis:redis \
  --link sentry-postgres:postgres \
  --link sentry-smtp:smtp \
  --env SENTRY_SECRET_KEY=${SENTRY_SECRET_KEY} \
  --name sentry-cron \
  ${REPOSITORY} \
  run cron
```


### 基于 docker-compose 进行 Sentry 服务管理

> https://github.com/getsentry/onpremise

**Requirements**

- Docker 1.10.0+
- Compose 1.6.0+ (optional)

可以对 `docker-compose.yml` 进行定制化修改，以满足你的需求；如下指令为常规需要完成的步骤；

- `mkdir -p data/{sentry,postgres}` - Make our local database and sentry config directories. This directory is bind-mounted with postgres so you don't lose state!
- `docker-compose run --rm web config generate-secret-key` - Generate a secret key. Add it to `docker-compose.yml` in base as `SENTRY_SECRET_KEY`.
- `docker-compose run --rm web upgrade` - Build the database. Use the interactive prompts to create a user account.
- `docker-compose up -d` - Lift all services (detached/background mode).
- Access your instance at `localhost:9000`!

Note that as long as you have your database bind-mounted, you should be fine stopping and removing the containers without worry.


按照 `docker-compose` 方式进行服务管理后，能够看到如下信息：

- `onpremise_base_1` 和 `onpremise_cron_1` 和 `onpremise_worker_1` 的 Image Id 相同；
- `xx_yy_z` 命名风格（默认）；
- 存在两个运行 `/entrypoint.sh run web` 命令的容器；
- 基于 `tini` 实现 docker 中的多进程运行和管理；


```
[#311#root@ubuntu-1604 /opt/apps/onpremise]$docker-compose images
      Container           Repository        Tag         Image Id      Size
----------------------------------------------------------------------------
onpremise_base_1        onpremise_base   latest       19764d09673d   511 MB
onpremise_cron_1        onpremise_base   latest       19764d09673d   511 MB
onpremise_memcached_1   memcached        1.4          bdb0ceca47d8   55.9 MB
onpremise_postgres_1    postgres         9.5          cea87be7b432   252 MB
onpremise_redis_1       redis            3.2-alpine   f60c2c2ed490   18.9 MB
onpremise_smtp_1        tianon/exim4     latest       f0e5295455c1   165 MB
onpremise_web_1         onpremise_web    latest       28521fc03d66   511 MB
onpremise_worker_1      onpremise_base   latest       19764d09673d   511 MB
[#312#root@ubuntu-1604 /opt/apps/onpremise]$
[#312#root@ubuntu-1604 /opt/apps/onpremise]$docker-compose ps
        Name                       Command               State           Ports
---------------------------------------------------------------------------------------
onpremise_base_1        /entrypoint.sh run web           Up      9000/tcp
onpremise_cron_1        /entrypoint.sh run cron          Up      9000/tcp
onpremise_memcached_1   docker-entrypoint.sh memcached   Up      11211/tcp
onpremise_postgres_1    docker-entrypoint.sh postgres    Up      5432/tcp
onpremise_redis_1       docker-entrypoint.sh redis ...   Up      6379/tcp
onpremise_smtp_1        entrypoint.sh tini -- exim ...   Up      25/tcp
onpremise_web_1         /entrypoint.sh run web           Up      0.0.0.0:9000->9000/tcp
onpremise_worker_1      /entrypoint.sh run worker        Up      9000/tcp
[#313#root@ubuntu-1604 /opt/apps/onpremise]$
[#313#root@ubuntu-1604 /opt/apps/onpremise]$docker-compose top
onpremise_base_1
UID    PID    PPID    C   STIME   TTY     TIME               CMD
--------------------------------------------------------------------------
999   474     32749   0   04:58   ?     00:00:01   [Sentry] uWSGI worker 1
999   475     32749   0   04:58   ?     00:00:02   [Sentry] uWSGI worker 2
999   476     32749   0   04:58   ?     00:00:01   [Sentry] uWSGI worker 3
999   32419   32369   0   04:58   ?     00:00:00   tini -- sentry run web
999   32749   32419   0   04:58   ?     00:00:02   [Sentry] uWSGI master

onpremise_cron_1
UID    PID    PPID    C   STIME   TTY     TIME               CMD
--------------------------------------------------------------------------
999   429     32708   0   04:58   ?     00:00:17   [celery beat] run cron
999   32708   32633   0   04:58   ?     00:00:00   tini -- sentry run cron

onpremise_memcached_1
UID    PID    PPID    C   STIME   TTY     TIME        CMD
------------------------------------------------------------
999   29961   29780   0   04:38   ?     00:00:01   memcached

onpremise_postgres_1
UID    PID    PPID    C   STIME   TTY     TIME                            CMD
-----------------------------------------------------------------------------------------------------
999   471     29981   0   04:58   ?     00:00:00   postgres: postgres postgres 172.24.0.9(56348) idle
999   509     29981   0   04:58   ?     00:00:00   postgres: postgres postgres 172.24.0.6(55556) idle
999   511     29981   0   04:58   ?     00:00:00   postgres: postgres postgres 172.24.0.6(55558) idle
999   512     29981   0   04:58   ?     00:00:00   postgres: postgres postgres 172.24.0.6(55560) idle
999   29981   29781   0   04:38   ?     00:00:00   postgres
999   30527   29981   0   04:38   ?     00:00:00   postgres: checkpointer process
999   30528   29981   0   04:38   ?     00:00:00   postgres: writer process
999   30529   29981   0   04:38   ?     00:00:00   postgres: wal writer process
999   30530   29981   0   04:38   ?     00:00:00   postgres: autovacuum launcher process
999   30531   29981   0   04:38   ?     00:00:00   postgres: stats collector process

onpremise_redis_1
  UID       PID    PPID    C   STIME   TTY     TIME         CMD
--------------------------------------------------------------------
systemd+   29957   29843   0   04:38   ?     00:00:20   redis-server

onpremise_smtp_1
 UID      PID    PPID    C   STIME   TTY     TIME             CMD
-------------------------------------------------------------------------
root     29959   29795   0   04:38   ?     00:00:00   tini -- exim -bd -v
syslog   30433   29959   0   04:38   ?     00:00:00   exim -bd -v

onpremise_web_1
UID   PID   PPID    C   STIME   TTY     TIME               CMD
------------------------------------------------------------------------
999   308   32735   0   04:58   ?     00:00:00   tini -- sentry run web
999   465   308     0   04:58   ?     00:00:02   [Sentry] uWSGI master
999   484   465     0   04:58   ?     00:00:02   [Sentry] uWSGI worker 1
999   485   465     0   04:58   ?     00:00:02   [Sentry] uWSGI worker 2
999   486   465     0   04:58   ?     00:00:02   [Sentry] uWSGI worker 3

onpremise_worker_1
UID    PID    PPID    C   STIME   TTY     TIME                                   CMD
-------------------------------------------------------------------------------------------------------------------
999   408     32603   0   04:58   ?     00:00:24   [celeryd: celery@a5264a1bed69:MainProcess] -active- (run worker)
999   477     408     0   04:58   ?     00:00:15   [celeryd: celery@a5264a1bed69:Worker-1]
999   479     408     0   04:58   ?     00:00:07   [celeryd: celery@a5264a1bed69:Worker-2]
999   32603   32488   0   04:58   ?     00:00:00   tini -- sentry run worker
[#314#root@ubuntu-1604 /opt/apps/onpremise]$
```

#### 关于 Sentry with SSL/TLS

> If you'd like to protect your Sentry install with SSL/TLS, there are fantastic SSL/TLS proxies like `HAProxy` and `Nginx`.

这里直接建议使用 `HAProxy` 和 `Nginx` 这样的代理实现 SSL/TLS 了；


----------


## Installation with Python

> https://docs.sentry.io/server/installation/python/

这种安装方式比较繁琐，且和 docker 安装方式相比已经过时了，但其中有些信息有价值；

> **Note**: This method of installation is **deprecated** in favor of Docker.

### 有用信息整理

- virtualenv 的使用；
- 基于 `pip install -U sentry` 安装 sentry ；
- 基于源码安装 sentry
    - `$ python setup.py develop`
    - `$ pip install -e git+https://github.com/getsentry/sentry.git@master#egg=sentry-dev`
    - `$ pip install -e git+https://github.com/getsentry/sentry.git@___SHA___#egg=sentry-dev`
- 使用 `sentry init /path/to/configfile/` 创建默认配置 sentry.conf.py 和 config.yml 到指定目录（Starting with 8.0.0）；
- Redis 的使用（作为各种 backend services 的默认实现）
    - time-series storage
    - SQL update buffers
    - rate limiting
- 建议运行两个独立的 Redis clusters ，因为可以基于不同的配置实现 aggressive/optimized LRU ：
    - one for persistent data (TSDB)
    - one for temporal data (buffers, rate limits). 


### 和 redis 有关的几个功能

- [queue](https://docs.sentry.io/server/queue/) - Asynchronous Workers
- [update buffers](https://docs.sentry.io/server/buffer/) - Write Buffers
- [quotas](https://docs.sentry.io/server/throttling/) - Throttles and Rate Limiting
- [time-series storage](https://docs.sentry.io/server/tsdb/) - Time-series Storage

### 和 Migrations 有关的内容

- make sure you’ve created the database

```
# If you kept the database ``NAME`` as ``sentry``
$ createdb -E utf-8 sentry
```

- create the initial schema using the upgrade command

```
$ SENTRY_CONF=/etc/sentry sentry upgrade
```

- create the first user, which will act as a superuser

```
# create a new user
$ SENTRY_CONF=/etc/sentry sentry createuser
```

> All schema changes and database upgrades are handled via the `upgrade` command, and this is the first thing you’ll want to run when upgrading to future versions of Sentry.
> Internally this uses `South` to manage database migrations. `South` is a tool to provide consistent, easy-to-use and database-agnostic migrations for Django applications.


### Sentry 主服务

- Starting the **Web Service**

Sentry provides a built-in webserver (powered by `uWSGI`) to get you off the ground quickly.

To start the built-in webserver run sentry run web:

```
SENTRY_CONF=/etc/sentry sentry run web
```

- Starting **Background Workers**

A large amount of Sentry’s work is managed via background workers. These need run in addition to the web service workers:

```
SENTRY_CONF=/etc/sentry sentry run worker
```

> `Celery` is an open source task framework for Python.

- Starting the **Cron Process**

Sentry also needs a cron process:

```
SENTRY_CONF=/etc/sentry sentry run cron
```

### 为 Sentry 增加反向代理

> By default, Sentry runs on port **9000**. Even if you change this, under normal conditions you won’t be able to bind to port **80**. To get around this (and to avoid running Sentry as a privileged user, which you shouldn’t), we recommend you setup a simple web proxy.

- Sentry 默认监听 9000 端口；
- 常规情况下，变更 Sentry 监听端口到 80 不太容易；
- 因此建议使用 proxy 来解决问题；

#### Proxying with Nginx

You’ll use the builtin `HttpProxyModule` within `Nginx` to handle proxying:

```
location / {
  proxy_pass         http://localhost:9000;
  proxy_redirect     off;

  proxy_set_header   Host              $host;
  proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
  proxy_set_header   X-Forwarded-Proto $scheme;
}
```

详细配置见 [Deploying Sentry with Nginx](https://docs.sentry.io/server/nginx/)

#### Enabling SSL

> If you are planning to use SSL, you will also need to ensure that you’ve enabled detection within the reverse proxy (see the instructions above), as well as within the Sentry configuration:

正确使用 SSL 的前提：确保使能了方向代理的 detection 功能；

```
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### Running Sentry as a Service

#### Configure supervisord

略

#### Configure systemd

略

### Removing Old Data

One of the most important things you’re going to need to be aware of is storage costs. You’ll want to setup a cron job that runs to automatically trim stale data. **This won’t guarantee space is reclaimed** (i.e. by SQL), **but it will try to minimize the footprint**. This task is designed to run under various environments so it doesn’t delete things in the most optimal way possible, but as long as you run it routinely (i.e. daily) you should be fine.

```
$ crontab -e
0 3 * * * sentry cleanup --days=30
```


----------


## 概念梳理

### What does 'on-premises' mean?

On-Premise

> Traditional method of **installing** and **customizing** software on the customer’s own computers that reside inside their own data center.

On-premises software

> (often abbreviated as **on-prem software**, and also called **“on-premises” software**) is installed and run on computers on the premises (in the building) of the person or organisation using the software, rather than at a remote facility, such as at a server farm or cloud somewhere on the internet. **On-premises software** is sometimes referred to as “shrinkwrap” software, and **off-premises software** is commonly called “software as a service” or “computing in the cloud”.


### DSN

After you complete setting up a project in Sentry, you’ll be given a value which we call a `DSN`, or **Data Source Name**. It looks a lot like a standard URL, but it’s actually just a representation of the configuration required by the Sentry SDKs. It consists of a few pieces, including the **protocol**, **public and secret keys**, the **server address**, and the project identifier.

The `DSN` can be found in Sentry by navigating to `[Project Name] -> Project Settings -> Client Keys (DSN)`. Its template resembles the following:

```
'{PROTOCOL}://{PUBLIC_KEY}:{SECRET_KEY}@{HOST}/{PATH}{PROJECT_ID}'
```

It is composed of five important pieces:

- The **Protocol** used. This can be one of the following: http or https.
- The **public and secret keys** to authenticate the SDK.
- The **destination** Sentry server.
- The **project ID** which the authenticated user is bound to.

### Membership

> https://docs.sentry.io/learn/membership/

**Membership in Sentry is handled at the organizational level**. The system is designed so each user has a singular account which can be reused across multiple organizations (even those using SSO). Each user of Sentry should have their own account, and will then be able to set their own personal preferences in addition to receiving notifications for events.

### Roles

Access to organizations is dictated by roles. **Your role is scoped to an entire organization**.

Roles include:

- Owner
- Manager
- Admin
- Member
- Billing

### Breadcrumbs

- **Breadcrumbs** show you events that lead to errors. Reproduce errors without user feedback
    - **Client side**: Sentry's JavaScript **breadcrumbs** include most interaction events, from clicks and inputs to page navigation and console messages—even earlier-occurring errors.
    - **Server side**: Sentry server-side SDKs, including Python and PHP, provide **breadcrumbs** for logging messages, network requests, database queries, and more.
- **Breadcrumbs** don't include personal user information. For example, JavaScript breadcrumbs will show that an input was used, but not what was typed.

### 其他

- Use **issue graphs** to understand the frequency, scope, and impact of errors and prioritize what needs to be fixed.
- Track problems by **release version** so you can focus on debugging live errors and get alerted if things regress in the future.
- Link **commit data** to your error monitoring workflow so you can easily determine the cause of a bug and resolve with release.
- Understand the **context** that contributed to errors with tags and relevant information about your software, environment, and users.
- **Teams** group members' access to a specific focus, e.g. a major product or application that may have sub-projects.
- **Projects** allow you to scope events to a specific application in your organization. For example, you might have separate projects for production vs development instances, or separate projects for your web app and mobile app.
- Sentry provides an abstraction called **Releases** which you can **attach source artifacts to**.
- **Release artifacts** 可以理解为 release 衍生物，用于在后续 event 处理过程中提供更多的信息；
- Release artifacts are only considered at time of event processing.
- Releases are shared across the organization.
- Releases work on projects.
- **Authentication tokens** allow you to perform actions against the Sentry API on behalf of your account. They're the easiest way to get started using the API.


两种信息组织结构：

- Organizations -> Projects => 所有 project 都属于某个 organization ；
- Teams -> Projects => 所有 project 都通过 team 来进行划分，以便不同用户角色进行不同管理；


----------

## 相关链接

- https://sentry.io/welcome/
- https://hub.docker.com/\_/sentry/
- https://docs.sentry.io/
- https://docs.sentry.io/clients/go/
- https://docs.sentry.io/server/config/
- https://github.com/getsentry/raven-go
- https://phab.llsapp.com/w/engineer/ops/sentry/
- https://sentry.io/vs/logging/
- [getsentry/sentry](https://github.com/getsentry/sentry) -- Sentry is a cross-platform crash reporting and aggregation platform
- [Sentry On-Premise setup](https://github.com/getsentry/onpremise) -- 基于 docker-compose + Docker
- [getsentry/docker-sentry](https://github.com/getsentry/docker-sentry/) -- Docker Official Image packaging for Sentry
