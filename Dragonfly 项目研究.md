# Dragonfly 项目研究

| key | value |
| -- | -- |
| 项目名称 | Dragonfly |
| 地址 | https://github.com/dragonflyoss/Dragonfly  |
| 实现语言 | java + python + golang |
| 一句话描述 | Dragonfly 是一个智能的、基于 P2P 的、image 和 file 分发系统，主要解决以 Kubernetes 为核心的分布式应用编排系统的镜像分发难题 |
| 详细说明 | <br> 1. 力图解决 low-efficiency ，low-success rate 以及文件传输过程中的 waste of network bandwidth <br> 2. 尤其适用于大文件分发场景：例如 **application** distribution, **cache** distribution, **log** distribution, **image** distribution <br> 3. 解决云原生镜像分发的**效率**、**流控**与**安全**三大难题 <br> 4. Dragonfly 的系统架构不涉及对容器技术体系的任何改动，完全可以无缝支持容器使其拥有 P2P 镜像分发能力，以大幅提升文件分发效率 <br> 5. 通过动态压缩，可以在不影响 SuperNode 和 Peer 正常运行的情况下，对文件中最值得压缩的部分实施相应的压缩策略，从而可以节约大量的网络带宽资源，同时还能进一步提升分发速率 <br> 6. 通过 SuperNode 强大的任务调度能力，可以尽量使在同一个网络设备下的 Peer 互传分块，减少跨网络设备、跨机房的流量，从而进一步降低网络带宽成本 <br> 7. （即将支持）通过加密插件解决安全传输问题 |
| 特性支持 | <br> 1. P2P based file distribution <br> 2. Non-invasive support all kinds of container technologies <br> 3. Host level speed limit <br> 4. Passive CDN <br> 5. Strong consistency (基于 P2P 实现原理中的分片 MD5 来保证) <br> 6. Disk protection and high efficient IO <br> 7. High performance <br> 8. Exception auto isolation <br> 9. No pressure on file source (利用 P2P 中的 Supernode 概念) <br> 10. Support standard http header <br> 11. Effective concurrency control of Registry Auth <br> 12. Simple and easy to use |
| 项目使用 | 阿里巴巴、阿里云、中国移动、蚂蚁金服、菜鸟、滴滴、美团、高德地图、虎牙直播、科大讯飞、去哪儿 |
| 相关链接 | 很久之前的研究：https://phab.llsapp.com/T42675 |
| 调研日期 | 2018-11-19 |


----------

## 图示


- 整体架构

![](https://image-1251900790.cos.ap-chengdu.myqcloud.com/image/p2p_1.png)

- 原理图

![](https://image-1251900790.cos.ap-chengdu.myqcloud.com/image/p2p_2.png)

- Dragonfly P2P 容器镜像分发示意图

![Dragonfly P2P 容器镜像分发示意图](https://upload-images.jianshu.io/upload_images/5765738-04579233c3d60b5d.png)

- 分发普通文件（Downloading General Files）

![分发普通文件](https://upload-images.jianshu.io/upload_images/5765738-066ccdbbf5540428.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/552/format/webp)

- 分发镜像（Downloading Container Images）

![分发镜像](https://upload-images.jianshu.io/upload_images/5765738-1c3b6f0e8ff2fed7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/713/format/webp)

- 块文件如何被下载（Downloading Blocks）

![块文件如何被下载](https://upload-images.jianshu.io/upload_images/5765738-96d668724fe53913.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/634/format/webp)

- Pouch 与 Dragonfly 的使用架构图

![Pouch 与 Dragonfly 的使用架构图](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Pouch%20%E4%B8%8E%20Dragonfly%20%E7%9A%84%E4%BD%BF%E7%94%A8%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

- Dragonfly 下载原理图

![Dragonfly 下载原理图](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Dragonfly%20%E4%B8%8B%E8%BD%BD%E5%8E%9F%E7%90%86%E5%9B%BE.png)

- Ecosystem architecture: illustrates how Pouch fits into the container ecosystem

![Pouch 在生态中的架构图](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Pouch%20%E5%9C%A8%E7%94%9F%E6%80%81%E4%B8%AD%E7%9A%84%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

## 概念

### Registry

- 容器镜像的存储仓库，每个镜像由多个镜像层组成，而每个镜像层又表现为一个普通文件；
- Harbor 就是一个企业级的 Registry ；

### Block

- 当通过 Dragonfly 下载某层镜像文件时，Dragonfly 的 SuperNode 会把整个文件拆分成一个个的块，SuperNode 中的分块称为**种子块**，种子块由若干初始客户端下载，并迅速在所有客户端之间传播，其中分块大小通过动态计算而来。
- Every file is divided into multiple blocks, which are transmitted between peers.
- 每个文件被分成多个块（block）在 peer 之间传输，一个 peer 就是一个 P2P client 。CM 将判断本地磁盘中是否存在相应的文件，如果不存在，则从文件服务器下载到集群管理器；

### SuperNode: Cluster Manager

- The Cluster Manager is also called a supernode, which is responsible for CDN and scheduling every peer to transfer blocks between each other. 
- 集群管理器（cluster manager）也被称为超级节点（SuperNode），它负责 CDN 并调度每个对等体在它们之间传输块；
- SuperNode 是 Dragonfly 的服务端，它主要负责种子块的生命周期管理以及构造 P2P 网络并调度客户端互传指定分块；
- Cluster Manager will check if the corresponding file exists in the local disk. If not, it will be downloaded into Cluster Manager from file server.
- Supernode is a long-time process with two primary responsibilities:
    - It’s the **tracker** and **scheduler** in the P2P network that choose appropriate downloading net-path for each peer.
    - It’s also a **CDN server** that caches downloaded data from source to avoid downloading same files repeatedly.

### peer: dfget

- Each peer is a P2P client. 
- dfget is the P2P client, which is also called “peer”. It’s mainly used to download and share blocks.
- dfget 是 P2P 客户端，也被称为 peer ，主要用于下载和共享块；
- Dfget is the client of Dragonfly used for downloading files. It’s similar to wget. At the same time, it also plays the role of peer, which can transfer data between each other in P2P network.
- dfget 是 Dragonfly 的客户端，安装在每台主机上，主要负责分块的上传与下载以及与容器 Daemon 的命令交互；
- 下载同一个文件的 Host 彼此之间称为 Peer ；

### dfget proxy: dfdaemon

- dfget proxy is also called `dfdaemon`, which intercepts HTTP requests from `docker pull` or `docker push`, and then decides which requests to process with dfget.
- dfget proxy 也被称为 `dfdaemon` ，它负责拦截执行 `docker pull` 或 `docker push` 的 http 请求，然后确定哪些请求需要使用 dfget 来处理。
- Dfdaemon is used for pulling images only. It establishes a proxy between dockerd/pouchd and registry.
- Dfdaemon filters out layer fetching requests from all requests sent by dockerd/pouchd when pulling images, then uses dfget to downloading these layers.

## 架构

Dragonfly 整体架构分三层：

- 第一层 Config Service ，管理所有的 Cluster Manager (SuperNode) ；
- 第二层 Cluster Manager (SuperNode) ，管理所有的 Host ，Host 就是终端；
- 第二层 dfget ，就是类似 wget 的一个客户端程序；

Config Service 主要负责 Cluster Manager 的管理、客户端节点路由、系统配置管理以及预热服务等等。简单的说，就是负责告诉 Host，离他最近的一组 Cluster Manager 的地址列表，并定期维护和更新这份列表，使 Host 总能找到离他最近的 Cluster Manager。

需要注意的是，开源版的 dragonfly 目前没有开源 Config Service 。

Cluster Manager 主要的职责有两个：

- 以 passive CDN 方式从文件源下载文件，并生成一组种子分块数据；
- 构造 P2P 网络，并调度每个 peer 之间互传指定的分块数据。

Host 上就存放着 dfget，dfget 的语法跟 wget 非常类似。主要功能包括文件下载和 P2P 共享等。

## 原理

- 两个 Host 和 CM 会组成一个 P2P 网络（这里针对的是上面的图2），首先 CM 会查看本地是否有缓存，如果没有，就会**回源下载**，文件当然会被分片，CM 会多线程下载这些分片，同时会将下载的分片提供给 Host 们下载，Host 下载完一个分片后，同时会提供出来给 peer 下载，如此类推，直到所有的 Host 全部下载完。
- 本地下载的时候会将下载分片的情况记录在 metadata 里，如果突然中断了下载，再次执行 dfget 命令，会断点续传。
- 下载结束后，还会比对 MD5，以确保下载的文件和源文件是完全一致的。
- Dragonfly 通过 HTTP cache 协议来控制 CM 端对文件的缓存时长，CM 端当然也有自己定期清理磁盘的能力，确保有足够的空间支撑长久的服务。


----------


首先，docker pull 命令会被 dfget proxy 截获。然后，由 dfget proxy 向 CM 发送调度请求，CM 在收到请求后会检查对应的下载文件是否已经被缓存到本地，如果没有被缓存，则会从 Registry 中下载对应的文件，并生成种子分块数据（种子分块数据一旦生成就可以立即被使用）；如果已经被缓存，则直接生成分块任务，请求者解析相应的分块任务，并从其他 peer 或者 SuperNode 中下载分块数据，当某个 Layer 的所有分块下载完成后，一个 Layer 也就下载完毕了，同样，当所有的 Layer 下载完成后，整个镜像也就下载完成了。


----------


- 首先由 PouchContainer 发起 Pull 镜像命令，该命令会被 DFget 代理截获。
- 然后由 DFget 向 SuperNode 发送调度请求。
- SuperNode 在收到请求后会检查对应的文件是否已经被缓存到本地，如果没有被缓存，则会从 Registry 中下载对应的文件并生成种子块数据（种子块一旦生成就可以立即传播，而并不需要等到 SuperNode 下载完成整个文件后才开始分发），如果已经被缓存，则直接生成分块任务。
- 客户端解析相应的任务并从其他 Peer 或者 SuperNode 中下载分块数据，当某个 Layer 的所有分块下载完成后，一个 Layer 也就下载完毕，此时会传递给容器引擎使用，而当所有的 Layer 下载完成后，整个镜像也就下载完成了。

## Q&A

### P2P 是什么？

P2P 网络是一种分布式的去中心化的网络，在网络中每个节点的地位都是对等的，每个节点既能充当服务器，也同时能为其他节点提供服务，同时也享有其他节点提供的服务。

### 为什么要引入 P2P 技术来加速镜像分发？

> Dragonfly 的产生背景

在大规模的容器集群内，镜像分发往往需要消耗大量时间，并且会给镜像仓库带来很大的压力负担（文件服务器扛不住大量的请求，且扩容成本也非常巨大），通过 P2P 技术将流量分担到集群的每个节点上，这样可以大大缩短下载镜像的时间，并且能非常有效的减轻镜像仓库的压力。

通过 P2P 技术，可以彻底解决镜像仓库的带宽瓶颈问题，充分利用各个 Peer 的硬件资源和网络传输能力，达到规模越大传输越快的效果。

大量来自不同 IDC 的客户端请求消耗了巨大的网络带宽，造成网络拥堵；大量的应用部署在海外，海外服务器下载要回源国内，浪费了大量的国际带宽，而且还很慢；如果传输大文件，网络环境差，失败的话又得重来一遍，效率极低。

### Dragonfly 能解决哪些问题

作为一款**通用文件分发系统**，Dragonfly 主要能够解决以下几个方面的问题：

- **大规模下载问题**：应用发布过程中需要下载软件包或者镜像文件，如果同时有大量机器需要发布，比如 1000 台，按照 500MB 大小的镜像文件计算，如果直接从镜像仓库下载，假设镜像仓库的带宽是 10000Mbps，那么理想状态下至少需要 10 分钟（我算出的是 6~7min），而且实际情况很可能是仓库早已被打挂。
- **远距离传输问题**：针对跨地域跨国际的应用，比如阿里速卖通，它既要在国内部署，又要在美国和俄罗斯部署，而存储软件包的源一般只在一个地域，比如国内上海，那么在美国或者俄罗斯的机器当要下载软件包的时候就要通过国际网络传输，但是国际网络不仅延时高而且极不稳定，严重影响传输效率，进而导致业务不能及时上线新功能或者问题补丁，由此甚至会产生业务故障。
- **带宽成本问题**：除了传输效率问题，高昂的带宽成本也是一个非常严重的问题，很多互联网公司尤其是视频相关的公司，带宽成本往往可以占据其总体成本的很大一部分。
- **安全传输问题**：据统计，每年因为网络安全问题导致的经济损失高达 4500 亿美元，所以安全必须是第一生命线，文件传输过程中如果不加入任何安全机制，文件内容很容易被嗅探到，假设文件中包含账号或者秘钥之类的数据，一旦被截获，后果将不堪设想。

### 为什么 P2P 模式下载的人越多速度越快，为什么 P2P 伤害机械硬盘

https://yq.aliyun.com/articles/493678?spm=5176.10695662.1996646101.searchclickresult.11525c1anNkqgB

### 开源版本 v.s. 企业版本

据悉，Dragonfly 有两个版本：

- 开源版支持 Apache 2.0 协议，可用于 P2P 文件分发、容器镜像分发、局部限速、磁盘容量预检；
- 企业版则还具备断点续传、全局限速、镜像预热、支持内存文件系统、智能网络流控、智能动态压缩、智能调度策略等功能，该版本内置在**云效**、阿里云容器服务（公共云、专有云）之中。
- 需要注意的是，开源版的 dragonfly 目前没有开源 Config Service ；


### dragonfly 和 bitorrent 技术原理对照

技术原理类似我们用的 BT 下载技术的 bitorrent 协议 cluster-manager 就类似于 Tracker 服务器。.meta 类似于 .torrent 文件；通过 .torrent 文件，获取其他正在下载该文件的网址名单，根据 .torrent 文件中的网址连接到 tracker 服务器，再从 tracker 服务器上获取正在下载该文件的网址名单，然后与它们取得联系，从他们那里获取文件的片端，直到整个下载完成。

### 跨地域传输问题

> 解决远距离传输问题：CDN 缓存 + 预热技术

#### CDN 缓存技术

通过 CDN 缓存技术，每个客户端可以就近从 SuperNode 中下载种子块，而无需跨地域进行网络传输

![Dragonfly 基于 CDN 缓存技术解决跨地域传输问题](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Dragonfly%20%E5%9F%BA%E4%BA%8E%20CDN%20%E7%BC%93%E5%AD%98%E6%8A%80%E6%9C%AF%E8%A7%A3%E5%86%B3%E8%B7%A8%E5%9C%B0%E5%9F%9F%E4%BC%A0%E8%BE%93%E9%97%AE%E9%A2%98.png)

**同一个**文件的**第一个**请求者会触发检查机制，根据请求信息计算出缓存位置，

- 如果缓存不存在，则触发**回源同步操作**生成种子块（即直接从 registry 下载）；
- 否则，向源站（CDN 源站）发送 HEAD 请求并带上 If-Modified-Since 字段，该字段的值为上次服务器返回的文件最后修改时间，
    - 如果响应码为 304，则表示源站中的文件目前还未被修改过，缓存文件是有效的，然后再根据缓存文件的元信息确定文件是否是完整的，
        - 如果完整，则缓存完全命中；
        - 否则需要通过断点续传方式把剩下的文件分段下载过来，断点续传的前提是源站必须支持分段下载，否则还是要同步整个文件。
    - 如果 HEAD 请求的响应码为 200，则表示源站文件已被修改过，缓存无效，此时需要进行**回源同步操作**；
    - 如果响应码既不是 304 也不是 200，则表示源站异常或地址无效，下载任务直接失败。

通过 CDN 缓存技术可以解决客户端**回源下载**以及**就近下载**的问题，但是如果缓存不命中，针对跨地域远距离传输的场景，SuperNode 回源同步的效率将会非常低，这会直接影响到整体的分发效率，为了解决该问题，Dragonfly 采用了一种**自动化层级预热机制**来最大程度的提升缓存命中率；

#### 自动化层级预热机制

其大致原理如下：通过 Push 命令把镜像文件推送到 Registry 的过程中，每推送完一层镜像就会立即触发 SuperNode 以 P2P 方式把该层镜像同步到 SuperNode 本地，通过这种方式，可以充分利用用户执行 Push 和 Pull 操作的时间间隙（大概 10 分钟左右），把镜像的各层文件同步到 SuperNode 中，这样当用户执行 Pull 命令时，就可以直接利用 SuperNode 中的缓存文件，自然而然也就没有远距离传输的问题了。

![Dragonfly 自动化层级预热机制](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Dragonfly%20%E8%87%AA%E5%8A%A8%E5%8C%96%E5%B1%82%E7%BA%A7%E9%A2%84%E7%83%AD%E6%9C%BA%E5%88%B6.png)


### 安全传输问题

在下载某些敏感类文件（比如秘钥文件或者账号数据之类的文件）时，传输的安全性必须要得到有效保障，在这方面，Dragonfly 主要做了以下几个方面的工作：

- 支持 HTTP Header 传输，以满足那些需要通过 Header 来进行权限验证的下载请求
- 通过自研的数据存储协议对数据块进行包装传输，后续还会对包装的数据进行再加密
- 即将支持安全加密功能插件化
- 通过多重校验机制，可以严格防止数据被篡改

### 使用建议

- 对一个机房或集群而言，建议至少准备 2 台 8 核、16G 内存、千兆网络的机器，用于部署超级节点；
- Docker 部署适用于在本地快速部署和测试；物理机部署适用于生产环境；


----------


## 实验

Ref: https://d7y.io/quick_start/

### test1

#### 环境准备


```
# 拉取镜像
docker pull registry.cn-hangzhou.aliyuncs.com/alidragonfly/supernode:0.2.0

# 启动 supernode
docker run -d -p 8001:8001 -p 8002:8002 registry.cn-hangzhou.aliyuncs.com/alidragonfly/supernode:0.2.0

# 下载 client
wget http://dragonfly-os.oss-cn-beijing.aliyuncs.com/df-client_0.2.0_linux_amd64.tar.gz

# 解压
tar zxvf df-client_0.2.0_linux_amd64.tar.gz

# 设置 PATH 方便使用
export PATH=/root/workspace/dragonfly/df-client:$PATH
```

#### 下载 file 测试

```
# dfget -u 'https://github.com/alibaba/Dragonfly/blob/master/docs/images/logo.png' -o /tmp/logo.png
--2018-11-20 17:33:07--  https://github.com/alibaba/Dragonfly/blob/master/docs/images/logo.png
current user[root] output path[/tmp/logo.png]
dfget version:0.2.0
workspace:/root/.small-dragonfly/ sign:25942-1542706387.250
node
download from source
download SUCCESS(0) cost(2.781s) length:56040 reason:5
```

查看 workspace 内容

```
# tree ~/.small-dragonfly
/root/.small-dragonfly
├── data
├── logs
│   └── dfclient.log
└── meta

3 directories, 1 file
# 
# cat ~/.small-dragonfly/logs/dfclient.log
[2018-11-20 17:33:07,257] INFO sign:25942-1542706387.250 lineno:51 : cmd params:Namespace(callsystem=None, console=False, dfdaemon=False, filter=None, header=None, identifier=None, locallimit=None, md5=None, node=None, notbs=False, output='/tmp/logo.png', pattern=None, showbar=False, timeout=None, totallimit=None, url='https://github.com/alibaba/Dragonfly/blob/master/docs/images/logo.png', version=False),python version:sys.version_info(major=2, minor=7, micro=12, releaselevel='final', serial=0)
[2018-11-20 17:33:07,257] INFO sign:25942-1542706387.250 lineno:107 : target file path:/tmp/logo.png
[2018-11-20 17:33:07,258] ERROR sign:25942-1542706387.250 lineno:118 : /etc/dragonfly.conf not found or has not data
Traceback (most recent call last):
  File "/root/workspace/dragonfly/df-client/env.py", line 116, in init
    nodes = configutil.client_config["node"]["address"].split(",")
KeyError: 'node'
[2018-11-20 17:33:07,258] ERROR sign:25942-1542706387.250 lineno:136 : init fail but try back down
Traceback (most recent call last):
  File "/root/workspace/dragonfly/df-client/dfget", line 128, in <module>
    register_result = env.init()
  File "/root/workspace/dragonfly/df-client/env.py", line 116, in init
    nodes = configutil.client_config["node"]["address"].split(",")
KeyError: 'node'
[2018-11-20 17:33:07,259] INFO sign:25942-1542706387.250 lineno:604 : start download logo.png from the source station
[2018-11-20 17:33:10,007] INFO sign:25942-1542706387.250 lineno:109 : move src:/tmp/tmpyMiF_p.backsource to dst:/tmp/logo.png result:True cost:0.001
[2018-11-20 17:33:10,031] INFO sign:25942-1542706387.250 lineno:94 : |UNKNOWN|https://github.com/alibaba/Dragonfly/blob/master/docs/images/logo.png|56040|56040|UNKNOWN|UNKNOWN|2.781|
[2018-11-20 17:33:10,031] INFO sign:25942-1542706387.250 lineno:111 : download SUCCESS cost:2.781s length:56040 reason:5
root@backend-shared-stag-0:~/workspace/dragonfly/test#
```

上述日志说明：

- 存在 python 程序
- 可以设置 /etc/dragonfly.conf

#### 下载 Image 测试

```
# 指定 registry 启动 dfdaemon （将监听 65001 端口）
nohup dfdaemon --registry https://index.docker.io > /dev/null 2>&1 &

# 更新 dockerd 配置文件 /etc/docker/daemon.json 的内容
"registry-mirrors": ["http://127.0.0.1:65001"]

# or 更新 /lib/systemd/system/docker.service 中的 --registry-mirror 配置

# 重启 dockerd
systemctl restart docker

# Download an image with Dragonfly（然而拉取过程似乎并没有和 Dragonfly 产生关系）
docker pull nginx:latest
```

----------


## 参考

- https://d7y.io/
- [阿里巴巴文件分发系统Dragonfly搭建](https://www.jianshu.com/p/1a85ea6b53ee)  -- 随便看看
- [使用p2p技术加速容器镜像分发](https://www.bladewan.com/2018/08/18/docker_image_p2p/)  -- 可以参考
- [12 Alibaba Techs made Open-source in 2017](https://medium.com/@alitech_2017/alibabas-open-source-core-technologies-of-2017-2734ba5c154a)  -- 随便看看
- [深度解读阿里巴巴云原生镜像分发系统 Dragonfly](https://mp.weixin.qq.com/s/UUZDIGopz5UruRpnxcOZ8Q) -- 重点
- [阿里巴巴正式开源自研容器技术Pouch](https://yq.aliyun.com/articles/283774)
