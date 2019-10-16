# Registry Mirror 与 --registry-mirror

## Docker Hub 拉取镜像慢解决方案

- Registry 中包含一个或多个 Repository ；
- Repository 中包含一个或多个 Image ；
- Image 用 GUID 标识，有一个或多个 Tag 与之关联；
- index.docker.io 其实是一个 Index Server ，和 Registry 有区别；
- **配置 Registry Mirror 之后，Docker 拉取镜像的过程**：Docker CLI 会试图获得授权，例如会向 https://index.docker.io/v1 请求认证，认证完成后，认证服务器会返回一个对应的 Token 。注意，这里用户认证与配置的 Registry Mirror 完全无关，这样我们就不用担心使用 Mirror 的安全问题了；
- 不幸的是，**Docker Hub 并没有在国内放服务器或者用国内的 CDN** ；为了克服跨洋网络延迟，一般有两个解决方案：一是使用**私有 Registry** ，另外是使用 **Registry Mirror** ；
    - 方案一：搭建或者使用现有的私有 Registry ，通过定期和 Docker Hub 同步热门的镜像，私有 Registry 上保存了一些镜像的副本，然后大家可以通过 `docker pull private-registry.com/user-name/ubuntu:latest`，从这个私有 Registry 上拉取镜像。因为这个方案需要定期同步 Docker Hub 镜像，因此它比较适合于使用的镜像相对稳定，或者都是私有镜像的场景。而且用户需要显式的映射官方镜像名称到私有镜像名称，私有 Registry 更多被大家应用在企业内部场景。私有 Registry 部署也很方便，可以直接在 Docker Hub 上下载 Registry 镜像，即拉即用，具体部署可以参考官方文档。
    - 方案二：使用 Registry Mirror ，它的原理类似于缓存，如果镜像在 Mirror 中命中则直接返回给客户端，否则从存放镜像的 Registry 上拉取并自动缓存在 Mirror 中。最酷的是，是否使用 Mirror 对 Docker 使用者来讲是透明的，也就是说在配置 Mirror 以后，大家可以仍然输入 `docker pull ubuntu` 来拉取 Docker Hub 镜像，除了速度变快了，和以前没有任何区别。


----------

## DaoCloud 提供的 Registry Mirror 服务

下面的例子，使用的是由 DaoCloud 提供的 Registry Mirror 服务，在申请开通 Mirror 服务后你会得到一个 Mirror 地址（我的地址：http://99a63370.m.daocloud.io ），然后我们要做的就是把这个地址配置在 Docker Server 启动脚本中，重启 Docker 服务后 Mirror 配置就生效了；


简单介绍下**如何在 DaoCloud 申请一个 Mirror 服务**：

- 账号注册：http://www.daocloud.io/
- 加速器：https://www.daocloud.io/mirror#accelerator-doc ，启动成功后，你就拥有了一个你**专用的 Registry Mirror 地址**了，加速器链接就是你要设置 "`--registry-mirror`" 的地址。目前每个用户有 10G 的加速流量（Tips：如果流量不够用可以邀请好友获得奖励流量）

**加速器说明**：

- 天下容器，唯快不破：Docker Hub 提供众多镜像，你可以从中自由下载数十万计的免费应用镜像， 这些镜像作为 docker 生态圈的基石，是我们使用和学习 docker 不可或缺的资源。**为了解决国内用户使用 Docker Hub 时遇到的稳定性及速度问题 DaoCloud 推出永久免费的新一代加速器服务**。
- 原生技术：新一代 Docker 加速器采用自主研发的智能路由及缓存技术，并引入了先进的协议层优化，极大提升拉取镜像的速度和体验。完全兼容 Docker 原生的 `--registry-mirror` 参数配置方式。支持 Linux，MacOS ，Windows 三大平台。使您能够更加方便地配置和使用镜像加速功能。
- 配置 Docker 加速器：
    - **Linux**：`curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://99a63370.m.daocloud.io`，该脚本可以将 `--registry-mirror` 加入到你的 Docker 配置文件 `/etc/docker/daemon.json` 中（在 vagrant 环境中使用了一下，ps 后没找到和这个 mirror 地址相关的信息）。适用于 Ubuntu14.04、Debian、CentOS6 、CentOS7、Fedora、Arch Linux、openSUSE Leap 42.1，其他版本可能有细微不同。更多详情请[访问文档](http://guide.daocloud.io/dcs/daocloud-9153151.html)。
    - **MacOS**：**Docker For Mac** 右键点击桌面顶栏的 docker 图标，选择 Preferences ，在 Daemon 标签（Docker 17.03 之前版本为 Advanced 标签）下的 Registry mirrors 列表中加入下面的镜像地址: `http://99a63370.m.daocloud.io` ，点击 Apply & Restart 按钮使设置生效。

> 针对 set_mirror.sh 脚本内容的详细说明，见《[Docker 加速器配置脚本说明](https://gitee.com/moooofly/SecretGarden/blob/master/Docker%20%E5%8A%A0%E9%80%9F%E5%99%A8%E9%85%8D%E7%BD%AE%E8%84%9A%E6%9C%AC%E8%AF%B4%E6%98%8E.md)》；

- **Docker 加速器是什么，我需要使用吗**？使用 Docker 的时候，需要经常从官方获取镜像，但是由于显而易见的网络原因，拉取镜像的过程非常耗时，严重影响使用 Docker 的体验。因此 DaoCloud 推出了加速器工具解决这个难题，通过智能路由和缓存机制，极大提升了国内网络访问 Docker Hub 的速度，目前已经拥有了广泛的用户群体，并得到了 Docker 官方的大力推荐。如果您是在国内的网络环境使用 Docker，那么 Docker 加速器一定能帮助到您。
- **Docker 加速器对 Docker 的版本有要求吗**？需要 Docker 1.8 或更高版本才能使用，如果您没有安装 Docker 或者版本较旧，请安装或升级。
- **Docker 加速器支持什么系统**？Linux, MacOS 以及 Windows 平台。
- **Docker 加速器是否收费**？DaoCloud 为了降低国内用户使用 Docker 的门槛，提供永久免费的加速器服务，请放心使用。


----------

## Harbor 提供的 Registry Mirror 服务

> 原文地址：[Configuring Harbor as a local registry mirror](https://github.com/vmware/harbor/blob/master/contrib/Configure_mirror.md)

Harbor runs as a **local registry** by default. It can also be configured as a **registry mirror**, which caches downloaded images for subsequent use. *Note that under this setup, the Harbor registry only acts as a mirror server and no longer accepts image pushing requests.* Edit `Deploy/templates/registry/config.yml` before executing `./prepare`, and append a proxy section as follows:


```
proxy:
  remoteurl: https://registry-1.docker.io
```

In order to access **private images** on the Docker Hub, a username and a password can be supplied:

```
proxy:
  remoteurl: https://registry-1.docker.io
  username: [username]
  password: [password]
```

You will need to pass the `--registry-mirror` option to your **Docker daemon** on startup:

```
docker --registry-mirror=https://<my-docker-mirror-host> daemon
```

> 说明：
>
> - 上面提及的 "Docker daemon" 是指 docker client 侧使用（依赖）的那个；
> - 上面的 startup 命令一般会配置在不同 linux 发行版中用于进行服务管理的服务中（如 systemd）


For example, if your mirror is serving on `http://reg.yourdomain.com`, you would run:

```
docker --registry-mirror=https://reg.yourdomain.com daemon
```

Refer to the [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/) for detailed information.


----------


## 与 linux 发行版及 docker 版本相关的问题

- Docker 版本在 1.12 或更高

**创建**或**修改** `/etc/docker/daemon.json` 文件，修改为如下形式 （请将 **加速地址** 替换为在[加速器](https://www.daocloud.io/mirror#accelerator-doc)页面获取的专属地址）

```
{
    "registry-mirrors": [
        "加速地址"
    ],
    "insecure-registries": []
}
```

- Docker 版本在 1.8 与 1.11 之间

您可以找到 Docker 配置文件，不同的 Linux 发行版的配置路径不同，具体路径请参考 [Docker官方文档](https://docs.docker.com/engine/admin/)，在配置文件中的 `DOCKER_OPTS` 加入 `--registry-mirror=加速地址` 重启Docker，不同的 Linux 发行版的重启命令不一定相同，一般为 `service docker restart` ；


----------

- By running a **local registry mirror**, you can keep most of the redundant image fetch traffic on your local network.
- if the set of images you are using is well delimited, you can simply pull them manually and push them to **a simple, local, private registry**.
- if your images are all built in-house, not using the Hub at all and relying entirely on your local registry is the simplest scenario.
- It’s currently not possible to mirror another private registry. Only the central Hub can be mirrored.
- The Registry can be configured as a `pull through cache`. In this mode a Registry responds to all normal docker pull requests but stores all content locally.


----------


参考：

- [玩转Docker镜像](http://blog.daocloud.io/how-to-master-docker-image/)
- [daocloud mirror accelerator](https://www.daocloud.io/mirror#accelerator-doc) - 给出基于自身用户的 mirror 地址
- [Docker 加速器](http://guide.daocloud.io/dcs/daocloud-9153151.html)
- [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/)
