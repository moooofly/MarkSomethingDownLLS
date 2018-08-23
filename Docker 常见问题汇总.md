# Docker 常见问题汇总

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


## alpine

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


## What are Docker <none>:<none> images?

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



