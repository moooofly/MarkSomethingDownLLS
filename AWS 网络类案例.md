# AWS 网络类案例

## 目录

- [有些机器访问这台机器的网络有问题](#有些机器访问这台机器的网络有问题)
- [这个 ELB 有一个 IP 不 work 了](#这个-elb-有一个-ip-不-work-了)
- [中国移动的网络连通性的问题](#中国移动的网络连通性的问题)
- [如何查看 ELB 的内网 ip](#如何查看-elb-的内网-ip)
- [想了解AWS的移动线路能力](#想了解aws的移动线路能力)
- [天津部分用户连接超时](#天津部分用户连接超时)
- [一些海外用户无法正常访问 ELB](#一些海外用户无法正常访问-elb)
- [用户请求不到 ELB](#用户请求不到-elb)
- [用户反馈通过 wifi 访问服务超时切换到移动网络就可以](#用户反馈通过-wifi-访问服务超时切换到移动网络就可以)


## 有些机器访问这台机器的网络有问题

### 0x01 提供的信息

i-fde8fac4 这台机器上运行了一个自己搭的 Redis 的服务，最近发现一些机器连这个 Redis 的时候会出现超时的情况。而对于另外一些同样是自己搭的 Redis 的连接，并没有出现这个问题。

我们简单做了测试，在两台 instances i-01f334f18e22b1a42 和 i-078a45397b84f60bb 上运行了脚本，每隔三秒执行一次 `redis-cli -h xx.xx.xx.xx PING`。一共执行 1000 次。最后发现 i-01f334f18e22b1a42 一直是正常的，但这台 i-078a45397b84f60bb 出现过 10 次左右的超时（127 秒都没有连上）。

因此怀疑可能是网络问题。麻烦帮忙查一下，网络方面是否有问题。多谢

Instance ID(s): i-fde8fac4

另外，补充一些。因为这两台机器是通过同一个安全组起的，VPC、AZ 什么的应该都一样。而且我是在同一时间段进行的测试，所以比较怀疑是网络的问题；

### 0x02 分析

> 我检查了您的三台实例的配置情况，发现**安全组**、**路由表**等相关配置都没有问题，发现您的**测试中的源实例和目的实例是在两个 VPC 内，而您在两个 VPC 之间搭建了 VPC-PEER** ，因此您在执行测试的时候应该是直接测试目的**实例的内网地址**，如果我以上的阐述不符合您的实际情况，请您指正。
>
> 我之后检查了您这三台实例的**底层硬件**状况，在近几个小时内没有发现丢包的现象，所以想询问您一下，您的这个问题目前是一直存在吗？从大约什么时间出现？是偶尔出现还是目前一直有超时连接的情况？

要点：

- 检查项：安全组、路由表、底层硬件的丢包情况

问题：没有说明

### 0x03 信息补充

是在两个 VPC，我们是用的内网 ip，网络也肯定是连通的，不然也不会有成功的请求是吧。

我搜了一下已有日志，是从上个月的 4.28 号开始出现的。


### 0x04 分析

> 目前的情况来看，您的两个客户端的配置几乎一样，我查看了两台实例的底层也均没有问题，CloudWatch 指标也没有明显异常，所以目前也无法定位到访问超时是因为 TCP 连接没有建立还是因为上层的 Redis 的问题。为了进一步定位分析问题，麻烦您提供一下以下信息：
> 1. 按照您的回复，访问不成功的情况是从 4.28 号开始出现，那么这个情况目前一直存在吗？问题存在的这段时间内是持续性的还是偶尔的呢？
> 2. 您在回复中提到查看日志，您的这个日志是 Redis 的服务端的访问日志吗？您可否将具体的日志信息截图或者打包成文件发送给我们呢，我们帮您分析。
> 3. 您尝试在您的客户端 `telnet` 服务器端的 6379 端口，会出现连接不上的情况吗？


### 0x05 信息补充

1. 一直存在
2. 我查到的日志其实没有跟 Redis 连接的日志，是应用层超时的日志（所以说，是应用日志），所以对分析应该帮助不大；
3. 我刚刚在跑脚本，还没跑完，但已经出现了一些 error。我是用 `nc` 来测试的 `nc -vz 172.31.9.182 6379` ，然后在那台有问题的机器上出现了 tcp 连接超时（差不多 127s 多），nc: connect to 172.31.9.182 port 6379 (tcp) failed: Connection timed out。所以应该不是 Redis 的问题；


### 0x06 分析

> 为了进一步的定位问题，能否麻烦您在出问题的客户端和服务端同时进行抓包，然后讲抓包信息回复给我们，我们帮助您分析。基本的方案如下：
> - 在有问题的客户端上，运行您的脚本开始对服务端的 6379 端口探测，然后同时在客户端和服务端进行抓包，参考命令如下：
>
> ```
> tcpudmp -i any -s0 -w client.pcap
> tcpudmp -i any -s0 -w server.pcap
> ```
>
> 然后您可以将 client.pcap 和 server.pcap 这两个文件回复给我们。

### 0x07 信息补充

好了。这次一直到 393 次才连续出现了两次

客户端命令是 `nc -vz 172.31.9.182 6379`
客户端抓包是 `sudo tcpdump -i any -s0 host 172.31.9.182 -w /tmp/client.pcap`
服务端抓包是 `sudo tcpdump -i any -s0 host 172.1.39.26 -w /tmp/server.pcap`


### 0x08 分析

> 经过查看您的抓包信息，我们发现有两次当客户端发出建立 TCP 连接的 SYN 包的时候，服务器端没有返回 SYN+ACK，因此导致了 TCP 的重传，当超过了重传次数的时候出现了 TIMEOUT 的情况，根据这种情况，想向您求证如下问题：
> 1. 您的这个 REDIS EC2 是否处于您的生产环境，也就是在您进行测试的时候，是否有其他的客户端正在进行连接？
> 2. 您的 SERVER 端的实例的内核参数：`net.ipv4.tcp_tw_recycle`、`net.ipv4.tcp_max_syn_backlog`、`net.core.somaxconn` 的值分别为多少？

### 0x09 信息补充

1. 是生产环境的。有其他客户端在连接
2. 内核参数如下

```
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_max_syn_backlog = 2048
net.core.somaxconn = 8192
```
不过我个人觉得不太像是 server 的问题，因为两个不同的机器的结果一直是不同的；

### 0x10 分析

> 首先，经过抓包的信息来看，在 client 出现 timeout 的时候，client 端已经发出了 TCP 的 SYN 包，于此同时，Server 端也接到了 client 发来的 SYN 包，但是没有回复 SYN+ACK 的包，因此原理上这可以证明 client 端的 TCP 没有问题，中间网络没有问题，只是 Server 端收到了 SYN 包但是没有回，也就是 drop 掉了（来自 client 的 SYN 包）。
> 
> 因为您的这个 Server 端是生产环境中的，有其他的客户端在连接，因此有一种可能的情况是，当您的这台 client 进行连接的时候， Server 端正好达到了连接的限制，这只是一种可能，为了印证这种可能原因，麻烦您提供一下以下信息：
> 1. 您的两个 client 的测试过程是什么样子的？是同时运行您的脚本测试？还是有先后顺序？
> 2. server 端文件系统的单个进程的最大连接数，参考命令：`ulimit -a`
> 3. server 目前的连接情况，首先您在 client 端运行您的脚本（发起）测试，然后在 Server 端运行以下参考命令：
>
> ```
> sudo netstat -natp
> ss -slt
> ```
>
> 需要注意的是，以上命令的结果是随着您的连接不断增多而可变的，所以在您的测试的时候，麻烦您可以运行几次来查看变化。
> 4. 您的测试脚本完成之后，查看一下系统日志 `/var/log/messages` 有没有报错信息。
>
> 您可以将以上命令的结果截图或者将文件发送给我们，我们帮您分析，您也可以留一个电话联系方式给我们，我们电话沟通一下。

### 0x11 信息补充

- 同时在两台机器上运行脚本，所以应该没什么特定的先后顺序


- ulimit

```
$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 983543
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 51200
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 983543
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

- 附件里的是每隔十秒运行 `netstat` 和 `ss` 的结果，文件名是 UTF-8 的时间。出问题的时间点是在 2017-05-16 07:28:18 +0000

```
"56 2017-05-16 07:28:09 +0000 0.001226376"
Connection to 172.31.9.182 6379 port [tcp/*] succeeded!
"57 2017-05-16 07:28:12 +0000 0.001327063"
Connection to 172.31.9.182 6379 port [tcp/*] succeeded!
"58 2017-05-16 07:28:15 +0000 0.001321123"
Connection to 172.31.9.182 6379 port [tcp/*] succeeded!
"59 2017-05-16 07:28:18 +0000 0.001460173"
nc: connect to 172.31.9.182 port 6379 (tcp) failed: Connection timed out
"slow"
"60 2017-05-16 07:30:28 +0000 127.335538117"
Connection to 172.31.9.182 6379 port [tcp/*] succeeded!
"61 2017-05-16 07:30:31 +0000 0.001380226"
Connection to 172.31.9.182 6379 port [tcp/*] succeeded!
"62 2017-05-16 07:30:34 +0000 0.001211711"
```

4. 没有 `/var/log/messages`，用 `dmesg -T | less` 看了，没有什么问题；


### 0x12 分析

> 首先，通常来讲，client 出现 timeout，并且此时 client 端已经发出了 TCP 的 SYN 包，与此同时 Server 端也接到了 client 发来的 SYN 包，但是没有回复 SYN+ACK 的包，出现上述现象的原因大概有如下三点：
> 1. 如果半连接队列满了，且 syncookie disable，则把 SYN 直接丢掉；
> 2. 如果完成队列满了，把 SYN 直接丢掉； 
> 3. 在 `tcp_tw_recycle`/`tcp_timestamps` 都开启的条件下，60s 内同一源 ip 主机的 socket connect 请求中的 timestamp 必须是递增的。
>
> 根据您之前发来的一些信息，我逐渐排除了第二个和第三个原因，判断依据如下：
> 1. 针对第三个原因，虽然查看到您的 `tcp_timestamps` 开启了，但是 `net.ipv4.tcp_tw_recycle = 0`，表示 `tcp_tw_recycle` 已经关闭。
> 2. 针对第二个原因，根据您发来的 `netstat`,`ss`,`ulimit` 信息，您的 TCP 连接数量没有超出 FD 的数量限制，并且每个 TCP 连接的 Recv-Q 都为 0，说明这些连接都已经被应用层拿走，而不再在已完成队列中，因此远远小于 `net.core.somaxconn = 8192` 这个值。
> 3. 由于我在您的信息中没有办法看到您的半连接队列的状态，并且不知道您是否开启了 `syncookie` ，所以无法获知是否因为第一个原因导致了 Server 端不回复 SYN+ACK 的包，但是不管怎样，Server 端在对 client 端回复 SYN+ACK 的时候，是不会针对特定的 client IP 的。
>
> 根据上面的分析，麻烦您提供一下测试信息：
>
> 1. server 端如下参数的值 
>
> ```
> net.ipv4.tcp_syncookies，
> net.ipv4.tcp_synack_retries
> net.ipv4.tcp_syn_retries
> ```
> 
> 2. 在您同时在两个 client 端进行测试的时候，请在两个 client 端同时抓包，并且在 Server 端抓包的时候将过滤条件指定为抓取两个 client host IP 的包（您之前给我的 Server 端的抓包只有一个 client 的），需要注意的是为了保证测试结果，请您在两个 client 上同时运行测试脚本。
> 3. 您的 server 端的内核版本是什么？参考命令：`uname -a`
> 4. 您的 Server 端或者 client 有没有配置什么 iptables 规则。

### 0x13 信息补充

昨晚我们发现这个 redis 的连接数比较多，并且很多是不存在的连接，但 Redis server 端还保持着。然后查到是因为我们 Redis timeout 的设置是 0，所以即使连接断掉也会一直保持，直到操作系统把它关掉。我们把这个 timeout 设置成一天，就发现连接数释放了很多，并且我们之前观察到的问题也没了。所以应该是这个原因导致的。

我还是把你需要的信息发你一下吧。但因为问题没了，所以抓包就没有办法发了。

```
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_syn_retries = 6
```

```
$ uname -a
Linux neo-cache-prod-0 3.13.0-29-generic #53-Ubuntu SMP Wed Jun 4 21:00:20 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

server 上没有配置，我们都是靠安全组来控制访问的

```
$ sudo iptables --list
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

client 上有一些跟 docker 相关的

```
$ sudo iptables --list
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-FIREWALL  all  --  anywhere             anywhere

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  anywhere             anywhere
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
KUBE-FIREWALL  all  --  anywhere             anywhere

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination
DROP       all  --  anywhere             anywhere             /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000

Chain KUBE-SERVICES (1 references)
target     prot opt source               destination
REJECT     tcp  --  anywhere             100.69.253.189       /* platform/newreclicexporter:http has no endpoints */ tcp dpt:9126 reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             100.71.102.67        /* backend/homepage-prod-api-lb:http has no endpoints */ tcp dpt:http reject-with icmp-port-unreachable
```

### 0x14 分析

> 经过您将 Redis timeout 设置为 1 天之后，很多连接释放了，很高兴您的问题也得到解决，**因为您的 `net.ipv4.tcp_syncookies` 已经开启，因此半开队列数的限制应该不会影响您的问题**，因此发生该问题的主要原因就是您的 REDIS server 上的最大连接数（请注意这个和之前的半开队列数和完成连接数还是不一样的，而且一般这个不会设置成很小）达到上限,在您释放了部分链接之后，TCP 可以正常的建立了，影响最大连接数的可能原因有以下几点：
> 1. 文件描述符 fd 的限制：
>     - 进程级别：`ulimit –n` 或 `setrlimit()`
>     - 系统级别：`/proc/sys/fs/file-max`
> 2. 系统本身可用物理内存的大小：
>     - Connection track limit ，enable iptables 之后的 connection track ，数值请参考 `net.netfilter.nf_conntrack_max`
>
> 如果您方便的话，可以将上述的参数值回复给我们，我们再帮您看看哪些地方的设置可能导致了问题。

## 这个 ELB 有一个 IP 不 work 了

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1383225384&language=zh

### 0x01 提供的信息

internal ELB 对一个两个 ip 地址

```
$ dig internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn

; <<>> DiG 9.9.5-3ubuntu0.2-Ubuntu <<>> internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39947
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. IN A

;; ANSWER SECTION:
internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. 60 IN A 172.1.51.93
internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. 60 IN A 172.1.57.214
```

其中一个 ip 是不通的

```
ubuntu@ip-172-31-14-3:~$ telnet 172.1.51.93 80
Trying 172.1.51.93...
telnet: Unable to connect to remote host: Connection refused


ubuntu@ip-172-31-14-3:~$ telnet 172.1.57.214 80
Trying 172.1.57.214...
Connected to 172.1.57.214.
Escape character is '^]'.
^]

telnet> q
Connection closed.


ubuntu@ip-172-31-14-3:~$ telnet 172.1.51.93 80
Trying 172.1.51.93...
telnet: Unable to connect to remote host: Connection refused
```

ELB FQDN(s): a87dff023241411e7b48102ab8baa063

### 0x02 分析

> 分析过程：
> 
> 帮您检查了一下，ELB 今天由于流量下降有 ScaleDown 的行为，从 ELB 的状态来看，目前暂未发现异常信息，应该能正常提供服务才对。
> 
> 刚才再次检查发现 ELB IP 发生了变化，目前为 172.1.54.246，172.1.51.93，从您刚才测试 IP 地址 172.31.14.3 来看，虽然跨 VPC 访问 ELB ，但应该是能访问的。
> 
> 不知您能否再用 `telnet` 或 `nc -vz 172.1.51.93 80` 测试一下? 如果可以的话能否在您 172.1.51.144 这台机器上验证一下？

要点：

- ELB 由于流量下降有 ScaleDown 的行为（这里的问题是：后续看仍旧存在两个 ip 地址，是否说明又 ScaleUp 回来了？）
- 测试命令的使用：
    - `telnet xxx.xxx.xxx.xxx 80`
    - `nc -vz xxx.xxx.xxx.xxx 80`
- aws 建议在不跨 VPC 的机器上再次测试；


### 0x03 信息补充

> 现在确实是通了（这里指的是 `curl` ELB 的域名）。不过并不是因为 IP 发生变化而恢复的吧，因为 172.1.51.93 这个 ip 是没变的。另外，我是在 172.31.14.3 这台机器上测试的。
> 
> 虽然现在看上去是恢复了，但为啥会出现这个问题呢？怎么感觉 ELB 很不靠谱呢？
> 
> 还有一个问题是，我们一开始用 ELB 的 DNS 对这个 ELB 进行 `curl` 的时候发现一直都是正常的，但对 ip 却不行。这个是说明 ELB 本身会做一些处理吗？ 但问题是 DNS 却没有及时更新。麻烦这个解释一下吧
> 
> `curl` 的那个也有问题，但自己处理了

```
$ curl -v internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn/api/v1/cc/mine
* Hostname was NOT found in DNS cache
*   Trying 172.1.51.93...
* connect to 172.1.51.93 port 80 failed: Connection refused
*   Trying 172.1.57.214...
* Connected to internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn (172.1.57.214) port 80 (#0)
> GET /api/v1/cc/mine?appVer=6 HTTP/1.1
> User-Agent: curl/7.35.0
> Host: internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn
> Accept: */*
>
< HTTP/1.1 420 CUSTOM
< Cache-Control: no-cache
< Content-Language: en
< Content-Type: application/json
< Vary: Origin
< X-Login: 0
< X-Request-Id: 95837d8e-fe27-4e4e-afbc-20078169ca0b
< X-Runtime: 0.006138
< Content-Length: 28
< Connection: keep-alive
<
* Connection #0 to host internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn left intact
```

要点：

- `curl` 在针对域名进行处理时的逻辑：先进行 DNS 域名解析，然后针对解析的结果 ip 地址（可能多个），逐一进行 `curl` 操作（从上面的日志中可以看出，先访问了 172.1.51.93 失败，然后访问了 172.1.57.214 成功）；


### 0x04 分析

> 经进一步检查，ELB: internal-a87dff023241411e7b48102ab8baa063 在今天 11:41 触发了 ScaleDown ，之后 ELB 将节点进行了替换。在正常情况下，替换的节点在正常提供服务之后，老的节点退出服务，替换过程不会中断业务流量。
>
> 但今天这个 ELB 替换的节点 172.1.51.93 虽然成功启动了，但转发流量有些问题，从而您在 `telnet` 或 `curl` 测试时，无法正确的访问，在一段时间之后 ELB 自动监测到问题并更新了状态，才正常的提供服务。 
>
> 如电话中沟通，我已将这个问题以及您的测试结果提供给后台团队。后台团队已将ELB的节点做了替换，目前所有ELB节点暂时正常，对于出问题的节点，后台团队会继续调查，在收到相关调查结果后，我们会及时回复您。
>
> 非常抱歉对于 ELB: internal-a87dff023241411e7b48102ab8baa063-1424689483 出现的一个 IP 不能接受请求的问题，经后台团队反复排查暂还不能确定问题原因，因为这个问题在出现一个小时左右之后又恢复了，而目前还无法复现这个问题，以至于无法深入的排查。建议您如果再次遇到这个问题的话可以联系我们，我们会请求后台团队再次协助调查分析。

### 0x05 新的情况

刚刚 9:25 左右，我们 ELB 前边的 tengine（nginx）出现很大 error

```
2017/08/14 13:25:49 [error] 11020#0: *182655034 no live upstreams while connecting to upstream, client: 172.31.1.12, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11020#0: *182655034 no live upstreams while connecting to upstream, client: 172.31.1.12, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11016#0: *182650101 no live upstreams while connecting to upstream, client: 172.31.1.12, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11016#0: *182650101 no live upstreams while connecting to upstream, client: 172.31.1.12, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11015#0: *182654303 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request:
2017/08/14 13:25:49 [error] 11017#0: *182659258 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request:
2017/08/14 13:25:49 [error] 11015#0: *182654655 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11017#0: *182655522 no live upstreams while connecting to upstream, client: 172.31.1.12, server: apineo.llsapp.com, request:
2017/08/14 13:25:49 [error] 11016#0: *182653016 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request:
2017/08/14 13:25:49 [error] 11016#0: *182653021 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request: 
2017/08/14 13:25:49 [error] 11016#0: *182653024 no live upstreams while connecting to upstream, client: 172.31.133.151, server: apineo.llsapp.com, request: 
```

随后我们收到一些报警，查看的时候发现新的 ELB ip 是两个不同的

```
$ dig internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn

; <<>> DiG 9.9.5-3ubuntu0.2-Ubuntu <<>> internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58112
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. IN A

;; ANSWER SECTION:
internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. 26 IN A 172.1.56.177
internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn. 26 IN A 172.1.45.9

;; Query time: 0 msec
;; SERVER: 172.31.0.2#53(172.31.0.2)
;; WHEN: Mon Aug 14 13:39:16 UTC 2017
;; MSG SIZE  rcvd: 145
```

这两个 ip 都是通的（跟上次不太一样）。之后 reload tengine(nginx) 解决。看起来是 tengine 没有及时更新，以及 ELB 同时两台都有问题，导致 tengine 没来得及切换？ （如果只是一台的话，应该还能等另外一个 ip 失效？）

麻烦帮忙看看 9:20-9:50 的时候 ELB 背后的机器状态？


### 0x06 新情况分析

> 经过 Review 您的 CLB (Classic LB) ，我注意到在 2017-08-14 13:25 UTC (21:25 UTC+8) 的时候，两个老的 ELB 节点被停止了，这应该是导致您观察到 tengine 出错的原因。
> 
> 但整个过程是这样的：
>
> - 2017-08-14 11:52:54 UTC: CLB ScaleUp ，在这个过程中，两个新的（更大的）ELB 节点 172.1.56.177 和 172.1.45.9 被加入 ELB 的 DNS ，而两个老的节点 IP 172.1.56.212, 172.1.39.190 被从 ELB 的 DNS 中移除。
> - 由于 **ELB 采用一种叫 Graceful Shutdown 的机制**，我们在从 DNS 上移除老的节点后，由于节点上可能有未完成的连接/请求，我们并不会直接终止节点，而是保留节点运行一段时间后再终止。
> - 接下来，在 2017-08-14 13:25:20 UTC ，老的节点 172.1.56.212, 172.1.39.190 被终止。这也就解释了为什么您在 tengine 上看到有错误；重启 tengine 后，由于 tengine 会重新进行 DNS 解析，故可以拿到新 ELB 节点的 IP ，因此您此时发现 nginx 访问两个新的节点都是正常的。
> - 同时，我会建议您检查 tengine 的 upstream 的配置。因为原生的 Nginx，在配置 upsteam 的时候，如果不使用变量，那么在给 upstream 的 BE 发请求时，是不会重新解析 IP 地址的，而只会在 Nginx 启动时进行一次 DNS 解析，之后则一直使用一开始得到的 IP 地址；我猜想，您的 tengine 遇到的问题可能和这个有关。
> - 针对这个问题，Nginx 的解决方案是配置变量，并将 upstream 指向这个变量。更详细的信息您可以参考以下文档：https://tenzer.dk/nginx-with-dynamic-upstreams/
> - 在 ELB scale up 之后，解析得到的 IP 还是两个，新的节点加入 DNS ，而旧的会被移除。不会出现解析到 4 个 ELB ip 地址的情况；

要点：

- ELB scaleup 的逻辑；
- tengine (nginx) 支持 dynamic upstreams 方式基于 ip 地址变更问题；
- ELB 通过 Graceful Shutdown 机制关停节点时，若节点上可能有未完成的连接/请求，则会保留节点运行一段时间后再终止；换那句话说，如果未完成的连接/请求很多，则一定会触发 nginx (tengine) 上的报错；之后基于 dynamic upstreams 机制完整新 ip 地址的获取；


### 0x07 信息补充

我们看了晚上那段时间的 log，发现 8 点左右，流量已经都从老的 ELB 进入新的 ELB 了。

图一是查询 nginx upstream 日志中老的 ELB 的，到 8 点就没有了；

![Kibana - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Kibana%20-%201.png)

图二是查询新的 ELB 的日志，从 8 点开始，一直持续到 11 点左右，而且 9 点半左右明显有个低峰，应该就是出事故的时候；

![Kibana - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Kibana%20-%202.png)

图三是对图二进行放大后的、出事故时的图

![Kibana - 3](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Kibana%20-%203.png)

> kibana 搜索命令 source_:"nginx.common.upstream" AND ("172.1.56.177" OR "172.1.45.9")

可以看到 9:25 到 9:40（估计就是我 reload tengine 的时候），是完全没有流量的，再结合详细日志会看到很多 502：

```
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" GET /api/v1/cc/learning_goals/status? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" GET /api/v1/cc/learning_goals/status? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.1.12 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.1.12 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.1.12 apineo.llsapp.com "0.000" GET /api/v1/cc/mine? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup? HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.1.12 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup HTTP/1.1 502 "neo-cc"
2017-08-14T13:25:49+00:00 172.31.133.151 apineo.llsapp.com "0.000" POST /api/v1/cc/answerup HTTP/1.1 502 "neo-cc"

```

### 0x08 新情况分析

> 我们再次 review 了您的 ELB ，发现昨天 21:26pm-21:38pm 这段时间，172.1.56.177 和 172.1.45.9 接收到的平均流量为 6.28KiB/s ，这个流量基本上是对您 63 个后端实例做健康检查的流量的量级。另外，同时段 HTTPCode_ELB_5XX 的值为 0 ，从这两点大致判断，这段时间两个节点没有接收到来自 tengine 的流量。
>
> 我注意到您开启了 ELB access log ，要进一步确认的话，您可以调取这段时间的 ELB access log ，看一下是否有收到来自 tengine 的流量的记录，以及量级。
> 
> 我们对这两个 ELB 节点的网络指标（丢包／延迟等）进行了检查，这段时间都是正常的。且这段时间，两个节点的 CPU 利用率也瞬间降低到 0.8% 左右，这也从侧面说明这两个节点没有大量流量需要处理，处于较为空闲的状态。
>
> 在 21:38pm ，两个节点的 CPU 利用率，入网流量等同时同步飙升。这个时间点唯一发生的变量是您执行了 tengine 的 reload 动作，综合这些信息，我们怀疑问题是发生在 tengine 上，但是具体是 tengine 上发生了什么导致的这个问题，基于目前的日志和数据无法判断。
>
> 目前可以确定的是，172.1.56.177 和 172.1.45.9 在 14 号 21:26pm-21:38pm 这段时间突然没有接收到流量，和已经退出使用的两个老节点 172.1.56.212／172.1.39.190 没有关系，且这两个新节点在这段时间本身也没有发现异常。
> 
> 我们建议您把排查的重点放在 tengine 服务器上。如果这个问题再次复现的话，在 tengine 这台服务器上对 ELB 做下抓包，结合 ELB 的 access log ，我们可以判断一下流量是否有发出去，以及如果没有发出去，大概是什么原因造成的。同时不要急于 reload ，（而是要）检查 tengine 服务（和服务器）的运行状况，看是否有什么报错记录或异常发生。
>
> 当然，也请您及时联系我们，我们会同时从后台对 ELB 进行检查。
>
> 如之前工程师的建议，如果该问题再次出现的话，可以结合 ELB 的 access log，判断一下 ELB 是否有收到前端的流量以及量级。或者是结合抓包看一下前端所请求的 IP 是否是 ELB ScaleUp 之后的 IP 。当然您也可以随时联系我们，我们会同时从后台对协助进行检查。


要点：

- 平均流量 6.28 KiB/s 基本上对应 63 个后端实例做健康检查的流量的量级；即**每个后端实例健康检查的流量在 0.1 KiB/s 左右**；
- `HTTPCode_ELB_5XX` 是一个需要关注的指标；
- ELB access log 是一个需要关注的文件；需要自行开启；
- ELB 节点需要关注的指标：
    - 丢包
    - 延迟
    - CPU


## 中国移动的网络连通性的问题

### 0x01 提供的信息

腾讯华佗诊断分析系统

### 0x02 分析

根据后台团队的排查结果，我们能够确定**问题出现在上海市的中国移动运营商网络内**，具体可以参考附件中的截图。

我们会就这个问题向相应运营商进行投诉。同时，根据以往经验，来自最终用户的直接投诉在运营商处理时更为有效。因此我们建议您使用工信部电信用户申诉的方式进行投诉，具体过程如下：

1. 打开工信部电信用户申诉受理中心网站： http://www.chinatcc.gov.cn:8080/cms/shensus/
2. 填写必填信息，其中申诉问题涉及号码需要提供出现这个问题时使用的号码。
3. 在申诉内容中，将我们之前的分析结果和数据（比如 `mtr`/`traceroute` 等）提交上去。

申诉提交后，被投诉运营商会在数个工作日内联系投诉者，您可以详细的描述您的问题，并要求提供一个邮箱，将我们之前的分析过程、结论、数据及如何解决都发送给运营商。

我们也会进一步和相关运营商沟通改善网络质量。感谢您的理解和支持。



## 如何查看 ELB 的内网 ip

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1407748164&language=zh

### 0x01 提供的信息

我们在 ELB 的后端实例上看到一些日志中包含的 ip 地址，怀疑是 ELB 的 ip 地址，但因为查不到 ELB 自身的 ip，所以不能很好定位问题。

比如如何根据 ELB 的 DNS 查内网 ip（特别是 internet-facing 的），如何根据内网 ip 来查到具体是哪个 ELB ；

### 0x02 分析

- 对于内部 ELB ，您直接 dig 这个 ELB 的 DNS 域名就可以的得出它的内部 IP 地址；
- 对于面向互联网的 ELB ，您只能得到它的公网 IP ，无法得到私网 IP ；
- 如果这个节点已经退出服务，即便是内部的 ELB 您也是查询不到这个 IP 的；

所以最保险的方式，是您有分析需求的时候，及时联系我们帮您确定 IP 。需要注意的是一定要及时联系，因为我们这边的记录保留的时间也有限，如果时间过久也就查不到了。

还有一个思路，因为 ELB 是和后端实例在同一个 VPC 的，您可以关注一下这些不能确定身份的 IP 地址所归属的网段是否是这个 VPC 的网段，如果是的话，并且您也没有其他内部实例访问后端服务器的话，则比较可能是 ELB 的节点 IP 。当然这种方式只能用来大致判断，不能 100% 确定。

```
root@harbor-prod:~# dig neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn.

; <<>> DiG 9.10.3-P4-Ubuntu <<>> neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn.
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56017
;; flags: qr rd ra; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. IN A

;; ANSWER SECTION:
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 54.222.222.173
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 52.80.63.136
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 52.80.92.163
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 52.80.202.80
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 52.81.3.221
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 54.222.158.246
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 54.222.194.182
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 18 IN A 54.222.221.90

;; Query time: 1 msec
;; SERVER: 172.31.0.2#53(172.31.0.2)
;; WHEN: Thu Nov 29 10:47:59 UTC 2018
;; MSG SIZE  rcvd: 215

root@harbor-prod:~#
```


## 想了解AWS的移动线路能力

### 0x01 提供的信息

我们有很多移动用户反馈网络问题，想知道 AWS 对移动线路的优化等；

### 0x02 分析

AWS 中国区和中国国内的主要运营商均采用 BGP 互联。依靠 BGP 动态路由协议的选路规则选择最优路径进行数据传输。

流量会根据目的地址动态的选择最优路径。因此，对于分布在 AWS 中国区所、直连中国国内主要运营商的终端，能够直接访问到 AWS 资源而不需要经过多个运营商转发。

由于不知道您提到的“移动用户反馈网络问题”具体的现象以及发生时间和地点，我们还无法深入的分析原因。

如果您发现访问异常的情况，我们建议您提供问题发生的时间点和具体现象、通信双方公网 IP 地址和双向 `mtr`/`traceroute` ，我们会帮助您进一步定位问题所在。


## 天津部分用户连接超时

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1413946414&language=zh

### 0x01 提供的信息

我们收到天津的部分用户反馈说，连不上我们的服务 apineo.llsapp.com ，这个是 CNAME 到 ELB: neo-app-prod-elb 的。

> 这里的 dig 是由我后续补充的，为了确认 apineo.llsapp.com CNAME 到 neo-app-prod-elb 的事实，因此，这里给出的 ip 地址和 case 中是不同的；

```
root@harbor-prod:~# dig apineo.llsapp.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> apineo.llsapp.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49277
;; flags: qr rd ra; QUERY: 1, ANSWER: 9, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;apineo.llsapp.com.		IN	A

;; ANSWER SECTION:
apineo.llsapp.com.	60	IN	CNAME	neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn.
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 54.222.222.173
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 54.223.19.202
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 54.223.26.218
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 52.80.92.163
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 52.80.116.82
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 52.80.150.13
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 54.222.142.31
neo-app-prod-elb-936082747.cn-north-1.elb.amazonaws.com.cn. 60 IN A 54.222.221.90

;; Query time: 5 msec
;; SERVER: 172.31.0.2#53(172.31.0.2)
;; WHEN: Thu Nov 29 09:24:24 UTC 2018
;; MSG SIZE  rcvd: 246

root@harbor-prod:~#
```

比如，通过我们在客户端收集到的 log 显示，用户在 2:15-2:20 的时候连接这几个 IP 有问题

- 54.222.177.24
- 52.80.9.251
- 54.223.59.254
- 52.80.63.144
- 52.80.208.1

这几个 IP 应该都是 ELB 的 IP

ELB FQDN(s): neo-app-prod-elb


### 0x02 分析

> 请问您说的时间 2:15-2:20 是指北京时间 14:15-14:20 吧？
>
> 我查看了 ELB 今天的状态和处理的流量，在今天 14:15-14:20，并没有特殊的情况发生。
>
> 后续步骤：
>
> 你提供的 5 个 IP 都是 ELB: neo-app-prod-elb 的 IP。为进一步排查问题，我们需要跟您收集以下信息
>
> 1. 请问用户连不上的现象是什么？是否有相应的截图？
> 2. 是否可以把客户端收集到的 log 发送给我们？是否可以提供一下用户的公网 IP？
> 3. 我看到您这边启用了 ELB 的 AccessLog，请您提供一下今天的 ELB 的 AccessLog 。


## 一些海外用户无法正常访问 ELB

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1420393104&language=zh

### 0x01 提供的信息

时间：北京时间9月27日14:30左右
现象：86.41.119.4 这个 ip 的用户访问我们服务有问题（通过 ELB: neo-app-prod-elb），看了 ELB log，北京时间9月27日 14:30 左右没有这个 ip 的 log，但同时访问其他 ELB 能够收到请求。说明用户的网络没有问题，而是访问这个 ELB 出现了网络问题。
ELB FQDN(s): neo-app-prod-elb

### 0x02 分析

> [问题描述]
>
> 您案例里提到了来自**爱尔兰**的用户（ 86.41.119.4 ）在北京时间今天（9月27日）下午14:30分左右访问 ELB 出现了问题，但同时访问其他 ELB 是正常的，您想知道原因。如果我的理解有误，还请您指正。

> [分析过程及解决方案]
> 
> 我们**检查了您 neo-app-prod-elb 这个 ELB 以及对应后端实例的状态**，没有发现异常情况。同时，对于您的描述我们有几点不太清楚的情况，想跟您确认一下并且收集一些信息以便我们进一步的定位问题所在：
>
> 1. 您说访问出现了问题，能具体描述一下是什么问题吗？如果方便的话您可否在重现问题的时候打开浏览器开发者工具（Chrome 和 Firefox 按下 F12 即可打开），点击 Network 一栏，将具体发生错误的连接详细信息截图，以便我们确认访问具体问题所在；
> 2. 您能否做一个**从遇到问题客户端到 ELB 节点的（双向） `traceroute`** ，以便我们排查网络方面的问题；
> 3. 您在案例里提到从 86.41.119.4 这个 IP 访问其他的 ELB 是可以收到请求的，这里其他的 ELB 指的是中国 Region 的 ELB 吗？
>
> 由于**国际间的网络传输环境比较复杂**，可能有很多原因导致问题的出现，您提供的信息将对我们排查问题有非常大的帮助，我们会根据您提供的信息尽力为您排查。

要点：

- PC 端可以通过浏览器提供的开发者工具进行问题排查（手机上用不了）；
- 需要提供客户端到 ELB 节点的双向 `traceroute` 或抓包；

### 0x03 信息补充

1. 就是访问 API 一直连不上，因为是手机端，所以信息不是很多
2. 现在还不行
3. 是的。同一个账号里的其他 ELB

### 0x04 分析

> 您的这个用户（ 86.41.119.4 ）现在可以正常访问了吗？我们在北京做了 curl 访问 ELB：neo-app-prod-elb 是正常的。
> 
> 经过和后台网络团队沟通，我们得知，**在北京时间9月27日下午14:30分中国到爱尔兰的网络有拥塞的情况发生**，您能否告知我们连接失败的具体时间段，以便我们确认连接失败与拥塞的时间是否吻合。
>
> 由于您暂时无法提供**用户到服务器之间双向 traceroute 的信息**以及**客户端和服务端的抓包记录**，我们能够用来判断的信息有限，您能否提供给我们发生问题对应时间段 ELB：neo-app-prod-elb 以及您之前提到的另一个正常 ELB 的访问日志，以便我们进一步分析。

### 0x05 信息补充

9月23日出现，然后测试时间是 9.27日14:31。

能请求到的那个日志放在附件里了。（ELB: kong-llsapp-lb）

neo-api 日志太多了，这里放不了（全部 gzip 压缩后有 75.2MB），但我找了，确实没有：

```
$ ls
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.144.69_4zbbzph4.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.154.228_62lvltjp.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.248.43_5dvsmar6.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.157.95_386m2y6n.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.31.11_2xzjgqse.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.177.24_59fxsvda.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.49.63_5x1vn2w7.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.194.144_1a098i83.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.92.163_5vex76fe.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.124.196_3l7s9vqm.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.81.4.171_53vnzd0w.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.184.231_56pkna9b.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.81.8.215_wu8faezd.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.251.198_4ssg0b8j.log
$ ag 86.41.119.4
(blank)
```

### 0x06 分析

> 结合您提供日志中的时间段，我们检查了对应时间下 ELB 及后端实例的各项指标，没有发现异常。从北京到爱尔兰的网络拥塞时间段（北京时间9月27日下午2点半到3点）的这个时间段和您用户无法访问服务的时间段是吻合的，由此我们怀疑很可能是网络的原因导致了问题的发生。**对于分析这类网络问题，最有帮助的同时也是我们还没有得到的是问题客户端与服务端的抓包日志和双向的 mtr 记录**。
>
> 我们现在观察到北京到爱尔兰的网络拥塞情况已经有所缓解，不知道您的这个用户现在是否可以正常访问 ELB: neo-app-prod-elb ，如果依然不能正常访问的话，我们还是希望您能帮忙**提供客户端 ip 下的双向 mtr 记录**以**及客户端和服务端的抓包日志**，以便我们查找产生问题的原因。如果没有详细的日志，我们能做到的分析也会比较有限，还望您能谅解。
>
> 您的用户不能正常访问服务，我们明白您的感受，同时国际间的网络环境的确比较复杂，我们会尽可能的根据现有信息帮您分析，还望您能理解。如果有进一步问题，欢迎随时与我们联系。谢谢！

## 用户请求不到 ELB

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1420556564&language=zh

### 0x01 提供的信息

时间：9月27日 14:25 左右
现象：ip 为 183.238.79.10 的用户请求发不到 kong-llsapp-lb 这个 ELB ，一直超时，重试多次都没用；切 wifi 和 4G 都不行，并且 ELB log 中没有这个 ip，而同时这个用户却可以正常请求到 neo-app-prod-elb 这个 ELB 。


### 0x02 分析

> 我们检查了您 **ELB: kong-llsapp-lb** 以及对应后端实例（的状态），并且将  **ELB: kong-llsapp-lb** 与 **ELB: neo-app-prod-elb** 的 metrics 做了一个对比，没有发现异常。同时，我们对您的 ELB 做了一个 curl，显示结果如下，是正常的：

```
[ec2-user@ip-172-31-15-229 ~]$ curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn
* Rebuilt URL to: https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn/
*   Trying 54.223.53.86...
* TCP_NODELAY set
* Connected to kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn (54.223.53.86) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: OU=Domain Control Validated; OU=EssentialSSL Wildcard; CN=*.llsapp.com
*  start date: Nov 29 00:00:00 2017 GMT
*  expire date: Dec 10 23:59:59 2019 GMT
*  issuer: C=GB; ST=Greater Manchester; L=Salford; O=COMODO CA Limited; CN=COMODO RSA Domain Validation Secure Server CA
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x25d5e60)
> GET / HTTP/2
> Host: kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn
> User-Agent: curl/7.55.1
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 404
< date: Fri, 28 Sep 2018 10:31:53 GMT
< content-type: application/json; charset=utf-8
<
{"message":"no route and no API found with those values"}
* Connection #0 to host kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn left intact
```

> 令我们困惑的是，您说这个 **183.238.79.10** 用户通过 wifi 和 4G 都访问不了，这两种访问方式应该是有对应 2 个 IP ，我们想知道另一个 IP 以及它的访问方式（是 4G 还是 wifi），这样就我们就可以确认两种访问方式是否是来自同一个运营商。
>
> 您在案例里提到这个用户在9月7日下午 14:25 分左右访问不到 ELB，我们想知道这个现象大概持续了多久，现在是否已经可以访问了？您能否方便向我们提供一下 **ELB: neo-app-prod-elb** 以及 **ELB: kong-llsapp-lb** 在问题发生期间的 Log ，这将有助于我们去排查这是 ELB 的问题还是网络的问题，以便做进一步的分析。
>
> 您如果方便的话，能否在您的出问题用户的 wifi 环境下做个 `curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn` ，并将结果发给我们。

要点：

- 针对不同 ELB 进行 metrics 对比
- 通过 `curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn` 确认 ELB 是否能被正常访问；
- 在出问题的用户环境中执行上述 curl 命令（有困难）


## 用户反馈通过 wifi 访问服务超时切换到移动网络就可以

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1419913324&language=zh

### 0x01 提供的信息

服务：apineo.llsapp.com
服务对应的 ELB FQDN(s)：neo-app-prod-elb
现象：最近主要集中在福州，已经收到了好几个福州的用户反馈有相似的问题

### 0x02 分析

> 我们理解您的用户在使用 wifi 的情况访问您的业务超时，但使用手机移动网络就是正常的。
>
> 客户使用的 wifi 和他的移动网络可能并不是同一个网络运营商，譬如客户的 wifi 可能是中国联通的，但手机移动网络用的是中国移动的。这种情况下有可能是运营商造成的问题。
>
> 我们检查您的 ELB 运行都是正常的。为了进一步确认问题原因，不知您是否可以收集客户端更多的信息，例如，客户端使用 wifi 时的 ISP 是什么，使用的 IP 地址是什么，如果能够收集 `mtr`/`traceroute` 等 log 那就更好。

思路：

- 重新描述一遍客户提供的信息，以便确认信息的准确性；
- 提出可能的怀疑方向；
- 提供自身进行的检测和结论；
- 提供需要用户配合完成的测试（指明工具和使用方法）；

要点：

- 客户端使用 wifi 时的 ISP 是什么？
- 使用的 IP 地址是什么？
- 收集 `mtr`/`traceroute` 等 log

### 0x03 信息补充

一些遇到问题的终端用户的 IP 地址：

> 可能是通过 https://www.ipip.net/ip.html 查得的

| 地点 | ISP | IP |
| -- | -- | -- |
| 福建省福州市 | 铁通 | 111.142.134.165|
| 福建省三明市 | 铁通 | 111.142.73.78 |
| 福建省福州市 | 铁通 | 111.146.32.113 |
| 福建省南平市 | 铁通 | 111.128.16.139 |

### 0x04 分析

> 我们的 network 团队确认此**问题是由于终端用户使用的铁通运营商有问题导致**。我们检测**从 AWS 到这些终端用户的 IP 地址都有比较严重的丢包现象**：

```
# mtr --tcp --port 22 -r -c 100 111.128.16.139
Start: Mon Sep 17 13:44:26 2018
HOST: ip-10-0-1-117.cn-north-1.co Loss% Snt Last Avg Best Wrst StDev
1.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
2.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
3.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0


4.|-- 100.65.0.129       0.0% 100 0.5 0.7 0.4 4.6 0.6
5.|-- 54.222.0.139       0.0% 100 3.2 2.1 1.4 6.5 0.5
6.|-- 54.222.24.167      0.0% 100 5.8 7.6 1.4 64.6 6.5
7.|-- 54.222.0.162       0.0% 100 3.9 2.2 1.4 5.3 0.6
8.|-- 124.65.226.133     0.0% 100 3.7 10.2 3.6 79.5 17.5
9.|-- 202.96.13.117      52.0% 100 3.8 5.6 3.5 7.6 1.0
10.|-- 124.65.194.81     0.0% 100 4.6 16.3 4.3 1021. 101.6
11.|-- 219.158.101.22    0.0% 100 4.9 5.1 4.2 37.8 3.3
12.|-- 202.97.10.236     0.0% 100 3.6 3.8 3.5 4.2 0.0
13.|-- 61.237.127.229    0.0% 100 5.4 7.0 4.8 9.8 1.1
14.|-- 61.237.121.70     0.0% 100 22.6 22.2 21.6 30.6 1.3
15.|-- 61.237.126.106    0.0% 100 36.4 37.1 36.0 53.8 2.7


16.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
17.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0


18.|-- 218.207.222.54    86.0% 100 56.6 56.6 56.2 57.3 0.0
19.|-- 218.207.222.54    51.0% 100 57.0 325.0 50.3 7159. 1109.4
20.|-- 218.207.222.166   35.7% 98 7961. 1378. 50.0 7961. 2182.7
21.|-- 112.5.255.89      42.9% 98 1898. 2238. 49.9 8047. 2209.8
22.|-- 112.5.255.89      51.5% 97 8205. 2039. 49.8 8205. 2858.6
23.|-- 112.5.255.94      45.7% 94 8154. 2395. 49.8 8154. 2783.4
24.|-- 112.5.255.94      45.7% 94 3096. 1323. 49.4 8077. 2423.2
25.|-- 112.5.255.89      53.8% 93 1051. 2250. 49.9 7975. 2384.0
26.|-- 112.5.255.89      46.2% 93 3090. 1304. 50.0 8015. 2437.6
27.|-- 112.5.255.94      64.1% 92 7988. 2722. 49.8 8138. 2828.2
28.|-- 112.5.255.89      46.7% 92 7349. 1136. 49.6 8178. 2364.1
29.|-- 211.143.158.250   57.8% 90 3966. 2808. 49.8 8162. 3129.5
30.|-- 112.5.255.94      53.3% 90 1081. 812.9 49.7 8083. 1734.7
```

> 通过 https://bgp.he.net/ 可以查到，**111.142.134.165** 和 **61.237.126.106** 是**中国铁通**（`AS9394`）的地址，而 **218.207.222.54** 和下面的 **112.5.255.89** 等地址都是**中国移动**（`AS9808`）的地址。从这里看，AWS 发出的数据报已经到达了**目的地址**的网内（`AS9394`），但铁通又将数据报转发给了**中国移动**（`AS9808`），并最终路由不通。
> 
> 我们**怀疑是铁通内部的路由配置有误导致的这个问题**。由于我们与 `AS9394` 的网络运营商没有直连渠道，我们建议您让您的终端客户向 `AS9394` ，即这些客户使用的运营商“**福建铁通**”投诉，以修复这个问题。

思路：

- 将 mtr 的输出分为几个段（上面已经通过空行进行了拆分）
- 确定路由转发的走向（基于 IP 查 AS，结合 Loss% 信息）

要点：

- 通过 `mtr` 检测从 AWS 到终端用户的 IP 地址是否有比较严重的丢包现象；
- 要能读懂 `mtr` 输出信息的含义；
- 要能理解 `mtr` 测试参数的选择（`--tcp --port 22`）；
- 要了解如何使用 https://bgp.he.net/ 这个网站上提供的信息；
- 要能知道 ASxxxx 对应的 ISP 是谁，要能理解不同 ISP 互通时可能存在的问题；



