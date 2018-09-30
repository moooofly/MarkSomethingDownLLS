# Docker 常见问题汇总

## 内容目录

- [Containerd Architecture](#containerd-architecture)
- [Image Formats](#image-formats)
- [Starting a Container](#starting-a-container)
- [Pulling an Image](#pulling-an-image)
- [What is a container manifest](#what-is-a-container-manifest)
- [What are Docker image layers?](#what-are-docker-image-layers)
- [/var/run/docker.sock](#varrundockersock)
- [--privileged 特权容器](#--privileged-特权容器)
- [alpine 是什么](#alpine-是什么)
- [parent image v.s. base image](#parent-image-vs-base-image)
- [What are Docker `<none>:<none>` images?](#what-are-docker-nonenone-images)
- [镜像源变更](#镜像源变更)
- [DNS 变更](#dns-变更)
- [时区变更](#时区变更)


## Containerd Architecture

![Containd Architecture - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%20Architecture%20-%201.png)

![Containd Architecture - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%20Architecture%20-%202.png)

## Image Formats

![Image Formats - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Image%20Formats%20-%201.png)

![Image Formats - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Image%20Formats%20-%202.png)

## Starting a Container

- Initialize a root filesystem (RootFS) from snapshot
- Setup OCI configuration (config.json)
- Use metadata from container and snapshotter to specify config and mounts
- Start process via the task service

![Starting a Contrainer](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Starting%20a%20Contrainer.png)

## Pulling an Image

- Resolve manifest or index (manifest list)
- Download all the resources referenced by the manifest
- Unpack layers into snapshots
- Register mappings between manifests and constituent resources

![Pulling an Image](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Pulling%20an%20Image.png)

## What is a container manifest

Ref: https://stackoverflow.com/questions/47006220/what-is-a-container-manifest

> **An `image` is a combination of a JSON manifest and individual layer files**. The process of pulling an image centers around retrieving these two components. 

image 由 JSON manifest 和不同的 layer files 构成；镜像拉取行为正是围绕着这两部分内容展开的；

> So when you pull an Image file:
> 
> - Get Manifest:
> ```
> GET /v2/<name>/manifests/<reference>
> ```
>
> - When the manifest is in hand, the client must verify the signature to ensure the names and layers are valid.
>
> - Then the client will then use the digests to download the individual layers. Layers are stored in as blobs in the V2 registry API, keyed by their digest.

因此，当你拉取镜像时：

- 首先，获取 Manifest ；
- 其次，验证 signature ，以便确认 names 和 layers 的有效性；
- 最后，基于 digests 下载不同的 layers ；
- （之后就是运行容器了）

在 V2 registry API 中，Layers 是以 blobs 的形态进行保存的，通过 blobs 对应的 digest 进行索引；


> The **manifest** types are effectively the JSON-represented description of a named/tagged image. This description (manifest) is meant for consumption by a container runtime, like the Docker engine.

manifest 事实上就是基于 JSON 方式描述的 named/tagged image ；这个概念主要被 Docker engine 所使用；

> Any registry or runtime that claims to have Docker distribution v2 API/v2.2 image specification support will be interacting with the various manifest types to find out:
>
> - what actual filesystem content (layers) will be needed to build the root filesystem for the container, and..
> - any specific image configuration that is necessary to know how to run a container using this image. For example, information like what command to run when starting the container (as probably represented in the `Dockerfile` that was used to build the image).

任何 registry 或 runtime 只要声称支持 Docker distribution v2 API/v2.2 image specification ，也就表明支持了如下 manifest 类型：

- 用于构建容器所需的 root filesystem 的、真实的文件系统内容（layers）；
- 任何需要了解的、基于该 image 的容器如何运行的、具体 image 配置；


> As a prior answer mentioned, a client (such as the `docker pull` implementation) talking to a registry will interact over the Docker v2 API to first fetch the manifest for a specific image/tag and then determine what to download in addition to be able to run a container based on this image. The v2 manifest format does not have signatures encoded into it, but external verification using tools like a [`notary` server](https://github.com/theupdateframework/notary) can be used to validate external signatures on the same "blob"/hash of content for full cryptographic trust. Docker calls this "**Docker Content Trust**" but does not require it when talking to a registry, nor is it part of the API flow when talking to an image registry.

> One additional detail about manifests in the v2.2 spec: there is not only a **standard manifest type**, but also a **manifest list type** which allows registries to represent support for multiple platforms (CPU or operating system variations) under a single "image:tag" reference. The manifest list simply has a list of platform entries with a redirector to an existing manifest so that an engine can go retrieve the correct components for that specific platform/architecture combination. **In DockerHub today, all official images are now actually manifest lists, allowing for many platforms to be supported using the same image name:tag combination**. I have a tool which can query entries in a registry and show whether they are manifest lists and also dump the contents of a manifest--both manifest lists and "regular" manifests. You can read more at the [manifest-tool](https://github.com/estesp/manifest-tool) GitHub repository.


## What are Docker image “layers”?

From the official Docker docs:

- Each instruction in a Dockerfile creates a layer in the image. When you change the Dockerfile and rebuild the image, only those layers which have changed are rebuilt. ([here](https://docs.docker.com/engine/docker-overview/#docker-objects)) 
- Basically, a **layer**, or **image layer** is a change on an image, or an intermediate image. Every command you specify (`FROM`, `RUN`, `COPY`, etc.) in your Dockerfile causes the previous image to change, thus creating a new layer. You can think of it as staging changes when you're using git: You add a file's change, then another one, then another one...
- The concept of layers comes in handy at the time of building images. Because layers are intermediate images, if you make a change to your Dockerfile, docker will build only the layer that was changed and the ones after that. This is called **layer caching**.
- [Since Docker v1.10](https://blog.docker.com/2016/01/docker-1-10-rc/), with introduction of the **content addressable storage**, the notion of 'layer' became quite different. Layers have no notion of an image or of belonging to an image, they become merely collections of files and directories that can be shared across images. Layers and images became separated. ([here](https://windsock.io/explaining-docker-image-ids/))
- Images are composed of layers. Each layer is a set of filesystem changes. Layers do not have configuration metadata such as environment variables or default arguments - these are properties of the image as a whole rather than any particular layer. ([here](https://github.com/moby/moby/blob/master/image/spec/v1.2.md))
- A Docker image is built up from a series of layers. Each layer represents an instruction in the image’s Dockerfile. Each layer except the very last one is read-only. 
- Each layer is only a set of differences from the layer before it. The layers are stacked on top of each other. When you create a new container, you add a new writable layer on top of the underlying layers. This layer is often called the “**container layer**”. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this thin writable container layer. 
- A **storage driver** handles the details about the way these layers interact with each other.
- Docker uses storage drivers to manage the contents of the **image layers** and the writable **container layer**. Each storage driver handles the implementation differently, but all drivers use stackable image layers and the copy-on-write (CoW) strategy.
- **The major difference between a container and an image is the top writable layer**. All writes to the container that add new or modify existing data are stored in this writable layer. When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged.


![](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)


## /var/run/docker.sock

- It’s the unix socket the Docker daemon listens on by default and it can be used to communicate with the daemon from within a container.
- When Docker platform is installed on a host, the Docker daemon listens on the `/var/run/docker.sock` unix socket by default. This can be seen from the options provided to the daemon, it should contain the following entry: `-H unix:///var/run/docker.sock`
- `/var/run/docker.sock` 是 Docker daemon 默认监听的 Unix domain socket ，（若有需要）容器中的进程可以通过它与 Docker daemon 进行通信。
- 绑定 Docker daemon 的 `/var/run/docker.sock`套接字之后，容器的权限会很高，因为其可以对 Docker daemon 进行控制。因此，这一点必须谨慎使用，只能用于足够信任的容器。

简单使用：

```
# Run an alpine container and bind mount the docker.sock
docker run -v /var/run/docker.sock:/var/run/docker.sock -ti alpine sh

# Install the curl utility
apk update && apk add curl

# We can send a HTTP request to the /events endpoint through the Docker socket. 
# The command hangs on waiting for new events from the daemon. 
# Each new events will then be streamed from the daemon.
curl --unix-socket /var/run/docker.sock http://localhost/events
```

参考：

- [关于/var/run/docker.sock](https://blog.fundebug.com/2017/04/17/about-docker-sock/)
- [About /var/run/docker.sock](https://medium.com/lucjuggery/about-var-run-docker-sock-3bfd276e12fd)
- [Can anyone explain docker.sock](https://stackoverflow.com/questions/35110146/can-anyone-explain-docker-sock)
- [The Dangers of Docker.sock](https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/)


## --privileged 特权容器

- 大约在 0.6 版，--privileged 被引入 docker ；
- 使用 --privileged 容器内的 root 拥有真正的 root 权限；否则，容器内的 root 只是外部的一个普通用户权限；
- 基于 --privileged 启动的容器，可以看到很多 host 上的设备，并且可以执行 mount ；
- 甚至允许你在 docker 容器中启动 docker 容器；
- This feature allowed bypassing a prior constraint of using containers, they were unable to access the host system’s devices. A possible use case for this, aside from running Docker inside Docker, was allowing you to trivially use things like your web-cam and such from within Docker.

实验：

- 非特权

```
docker run -it alpine sh
```

```
/ # ls /dev/
console  fd       mqueue   ptmx     random   stderr   stdout   urandom
core     full     null     pts      shm      stdin    tty      zero
/ #
/ # mkdir /home/test/
/ # mkdir /home/test2/
/ # mount -o bind /home/test  /home/test2
mount: permission denied (are you root?)
```

- 特权

```
docker run -it --privileged alpine sh
```

```
/ # ls /dev
autofs              net                 tty13               tty4                tty9                ttyS6
bsg                 network_latency     tty14               tty40               ttyS0               ttyS7
btrfs-control       network_throughput  tty15               tty41               ttyS1               ttyS8
console             null                tty16               tty42               ttyS10              ttyS9
core                port                tty17               tty43               ttyS11              ttyprintk
cpu_dma_latency     ppp                 tty18               tty44               ttyS12              uinput
ecryptfs            psaux               tty19               tty45               ttyS13              urandom
fd                  ptmx                tty2                tty46               ttyS14              vboxguest
full                pts                 tty20               tty47               ttyS15              vboxuser
fuse                random              tty21               tty48               ttyS16              vcs
hpet                rfkill              tty22               tty49               ttyS17              vcs1
hwrng               rtc0                tty23               tty5                ttyS18              vcs2
input               sda                 tty24               tty50               ttyS19              vcs3
kmsg                sda1                tty25               tty51               ttyS2               vcs4
lightnvm            sdb                 tty26               tty52               ttyS20              vcs5
loop-control        sg0                 tty27               tty53               ttyS21              vcs6
loop0               sg1                 tty28               tty54               ttyS22              vcsa
loop1               shm                 tty29               tty55               ttyS23              vcsa1
loop2               snapshot            tty3                tty56               ttyS24              vcsa2
loop3               snd                 tty30               tty57               ttyS25              vcsa3
loop4               stderr              tty31               tty58               ttyS26              vcsa4
loop5               stdin               tty32               tty59               ttyS27              vcsa5
loop6               stdout              tty33               tty6                ttyS28              vcsa6
loop7               tty                 tty34               tty60               ttyS29              vfio
mapper              tty0                tty35               tty61               ttyS3               vga_arbiter
mcelog              tty1                tty36               tty62               ttyS30              vhost-net
mem                 tty10               tty37               tty63               ttyS31              zero
memory_bandwidth    tty11               tty38               tty7                ttyS4
mqueue              tty12               tty39               tty8                ttyS5
/ # mkdir /home/test/
/ # mkdir /home/test2/
/ # mount -o bind /home/test /home/test2
/ # mount
...
none on /home/test2 type aufs (rw,relatime,si=65f2e9cc676bf3bf,dio,dirperm1)
```

参考：

- [在 Docker 中运行 Docker](https://github.com/jpetazzo/dind)
- https://blog.csdn.net/halcyonbaby/article/details/43499409


## alpine 是什么

- A minimal Docker image based on Alpine Linux with a complete package index and only 5 MB in size! 
- Alpine 操作系统是一个面向安全的轻型 Linux 发行版。它不同于通常 Linux 发行版，Alpine 采用了 musl libc 和 busybox 以减小系统的体积和运行时资源消耗，但功能上比 busybox 又完善的多，因此得到开源社区越来越多的青睐。在保持瘦身的同时，Alpine 还提供了自己的包管理工具 apk，可以通过 https://pkgs.alpinelinux.org/packages 网站上查询包信息，也可以直接通过 apk 命令直接查询和安装各种软件。
- Alpine 中软件安装包的名字可能会与其他发行版有所不同，可以在 https://pkgs.alpinelinux.org/packages 网站搜索并确定安装包名称。如果需要的安装包不在主索引内，但是在测试或社区索引中。那么可以按照以下方法使用这些安装包；
- Alpine 由非商业组织维护的，支持广泛场景的 Linux发行版，它特别为资深/重度Linux用户而优化，关注安全，性能和资源效能。Alpine 镜像可以适用于更多常用场景，并且是一个优秀的可以适用于生产的基础系统/环境。
- Alpine Docker 镜像也继承了 Alpine Linux 发行版的这些优势。相比于其他 Docker 镜像，它的容量非常小，仅仅只有 5 MB 左右（对比 Ubuntu 系列镜像接近 200 MB），且拥有非常友好的包管理机制。官方镜像来自 docker-alpine 项目。
- 目前 Docker 官方已开始推荐使用 Alpine 替代之前的 Ubuntu 做为基础镜像环境。这样会带来多个好处。包括镜像下载速度加快，镜像安全性提高，主机之间的切换更方便，占用更少磁盘空间等。

相关资源：

- [docker_practice](https://yeasy.gitbooks.io/docker_practice/cases/os/alpine.html)
- Alpine 官网：http://alpinelinux.org/
- Alpine 官方仓库：https://github.com/alpinelinux
- Alpine 官方镜像：https://hub.docker.com/_/alpine/
- Alpine 官方镜像仓库：https://github.com/gliderlabs/docker-alpine


## parent image v.s. base image

A **parent image** is the image that your image is based on. It refers to the contents of the `FROM` directive in the Dockerfile. Each subsequent declaration in the Dockerfile modifies this parent image. Most Dockerfiles start from a parent image, rather than a base image. However, the terms are sometimes used interchangeably.

A **base image** either has no `FROM` line in its Dockerfile, or has `FROM scratch`.

其他：

- [Create a base image](https://docs.docker.com/develop/develop-images/baseimages/)
- [How to build a base image from scratch?](https://forums.docker.com/t/how-to-build-a-base-image-from-scratch/34928)


## What are Docker `<none>:<none>` images?

Ref: https://www.projectatomic.io/blog/2015/07/what-are-docker-none-none-images/

相关问题：

- What are `<none>:<none>` images ?
- What are dangling images ?
- Why do I see a lot of `<none>:<none>` images when I do `docker images -a` ?
- What is the difference between `docker images` and `docker images -a` ?

Two kinds of `<none>:<none>` images:

- The Good one: They stand for **intermediate images** and can be seen using `docker images -a`. They don’t result into a disk space problem, and can be useful for caching.
- The Bad one: The **dangling images** which can cause disk space problems. A dangling file system layer in Docker is something that is unused and is not being referenced by any images. Hence we need a mechanism for Docker to clear these dangling images. Can be seen using `docker images -a`. These dangling images are produced as a result of `docker build` or `docker pull` command. 

The next command can be used to clean up dangling images (Docker doesn’t have an automatic garbage collection system as of now. For now this command can be used to do a manual garbage collection):

```
docker rmi $(docker images -f "dangling=true" -q)
```


## 镜像源变更

- [阿里云镜像](https://opsx.alibaba.com/mirror)
- [清华镜像](https://mirrors.tuna.tsinghua.edu.cn/)
- [中科大镜像](https://mirrors.ustc.edu.cn/)
- [自建镜像](https://github.com/anjia0532/alpine-package-mirror)


### alpine

```
echo -e "https://mirrors.ustc.edu.cn/alpine/latest-stable/main\nhttps://mirrors.ustc.edu.cn/alpine/latest-stable/community" > /etc/apk/repositories
```

或者

```
echo "https://mirrors.ustc.edu.cn/alpine/latest-stable/main" >> /etc/apk/repositories
echo "https://mirrors.ustc.edu.cn/alpine/latest-stable/community" >> /etc/apk/repositories
```

之后使用如下命令安装

```
apk --update add --no-cache <package>
```

另外，可以通过如下命令安装不在主索引内，但是在测试或社区索引中的包

```
echo "http://dl-4.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories
```

### ubuntu

todo


## DNS 变更

- By default, a container inherits the DNS settings of the Docker daemon, including the `/etc/hosts` and `/etc/resolv.conf`. You can override these settings on a **per-container basis**. [Ref](https://docs.docker.com/config/containers/container-networking/#dns-services)
- 如果在构建镜像过程中经常出现失败问题，则可能是 GFW 的原因；可以通过抓包确认 DNS 解析是否访问了 google-public-dns-a.google.com (8.8.8.8) ；如果是，解决办法如下
    - 修改宿主机的 hosts 文件，写死 ip 地址；这种方式为静态方式；
    - 修改 Docker 的 daemon.json 文件（指定其他 DNS 服务，如 DNSPod 提供的 public DNS 服务）；这种方式为动态方式；[Ref](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
    - 推荐使用第二种办法；
    - DNSPod 提供的公共域名解析服务 [Public DNS +](https://www.dnspod.cn/products/public.dns) ，服务IP:119.29.29.29
    - 调整 daemon.json 中的 `"dns": ["119.29.29.29"]` 后，执行 `sudo systemctl daemon-reload`


## 时区变更

Docker Store 上的官方镜像基本上都默认是 UTC 时区；即 /etc/timezone 的内容为 Etc/UTC，也就是标准的 UTC 时间，所以要简单的调整一下，变成中国标准时间。

场景：

- 基于 Dockerfile 构建自定义镜像时进行时区调整
    - 详见下面
- 启动 container 时基于主机文件进行时区调整（好处是不会修改镜像，适合需要在不同时区主机上运行的场景）
    - `docker run -v /etc/localtime:/etc/localtime <IMAGE:TAG>`
- 针对运行中的 container 进行时区调整
    - `docker exec -it <CONTAINER NAME> bash`
    - `echo "Asia/Shanghai" > /etc/timezone && dpkg-reconfigure -f noninteractive tzdata` -- ubuntu 环境下的命令


其他：

- CST – China Standard Time
- UTC - Coordinated Universal Time
- GMT - Greenwich Mean Time
- noninteractive - 方便整合到 Dockerfile 中使用


Ref:

- [Docker 中如何设置 container 的时区
](https://brickyang.github.io/2017/03/16/Docker%20%E4%B8%AD%E5%A6%82%E4%BD%95%E8%AE%BE%E7%BD%AE%20container%20%E7%9A%84%E6%97%B6%E5%8C%BA/)
- [在 Docker 中配置时区](https://tommy.net.cn/2015/02/05/config-timezone-in-docker/)

### alpine

详见 [timezone_adjustment/Dockerfile.alpine](https://github.com/moooofly/dockerfiles/blob/master/timezone_adjustment/Dockerfile.alpine)

### ubuntu

详见 [timezone_adjustment/Dockerfile.ubuntu](https://github.com/moooofly/dockerfiles/blob/master/timezone_adjustment/Dockerfile.ubuntu)



