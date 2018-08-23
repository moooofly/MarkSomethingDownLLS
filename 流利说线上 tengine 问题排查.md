# 流利说线上 tengine 问题排查

结论：

- `tcp_max_tw_buckets` 设置的上限被达到（超限后，主动关闭连接的一方将不会进入 TIME_WAIT 状态，而直接进入 CLOSED 状态，会导致一些问题的出现）；
- `nf_conntrack` 达到上限（当 `nf_conntrack` 内核模块被 load 后，且服务器上连接跟踪数量超过该设定值时，系统会主动丢掉新连接建立的包，直到连接跟踪数量小于设置值后才会恢复；proxy 类服务一般不需要打开连接跟踪功能）；
- 随机端口使用达到未达到上限，但有接近趋势（目前主要和 kafka 服务相关）；
- 和 TIME_WAIT 相关的内核选项在进行配置时都涉及到权衡问题和适用场景问题，若打算使用，则必须搞清楚利弊；一般来说，针对 TCP 连接关闭问题，最好的对待方式就是维持 TCP 协议中规定的、应有的 TIME_WAIT 状态，然后努力在业务层通过连接池等方式实现连接复用（或者直接在协议使用上进行改进，如改为使用 https，http/1.1 或 http/2.0 等），进而减少 TIME_WAIT 的出现数量；
- tengine A 调整 tcp_max_tw_buckets 数值（目前调整为 180000）后，TIME_WAIT 量维持在 12w 左右，大致估算出单 tengine 上的 QPS 为 2000 （该机器 2MSL 时间为 1min），基于当前系统运行状态，可以推演请求量继续增加时的系统表现；
- 当 tcp_max_tw_buckets 为 65536 时（调整前），和 kafka 相关的 TIME_WAIT 数量在 2w~4w 左右，占比 30%~60% 左右；当 tcp_max_tw_buckets 为 18000 时（调整后），和 kafka 相关的 TIME_WAIT 数量在 5.2w 左右，占比 40% 左右；综上，非核心业务 kafka 对 tengine 网络相关资源占比较高，已经对使用该 tengine 的业务造成了影响；建议：要么优化 kafka 短连接使用现状，减少 TIME_WAIT 占比，要么将非核心业务的 kafka 隔离到一个单独的 tengine 上，保证核心业务可用资源的充足；


> 延伸问题：
>
> - 随机端口使用达到上限会造成什么问题？直接报错，无法建立连接；
> - `tcp_max_tw_buckets` 和 `nf_conntrack_max` 有关系么？没啥关系；
>    - 当 nf_conntrack 模块被 load ，且服务器上被跟踪的连接数量超过 `nf_conntrack_max` 设定值时，系统会主动丢掉新连接包，直到连接小于此设置值才会恢复；
>    - `tcp_max_tw_buckets` 只对主动关闭连接的一方是否能够进入 TIME_WAIT 状态起作用；
> - 为什么有的机器上存在 `/proc/net/nf_conntrack`，有的机器上没有？未知；


- 20180306 线上 3 个 tengine 输出对比（截图中 18000 应该是 180000）

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/20180306%20%E7%BA%BF%E4%B8%8A%E4%B8%89%E4%B8%AAtengine%E7%9A%84netstat%E8%BE%93%E5%87%BA%E5%AF%B9%E6%AF%94.png)

## ip-172-31-3-170

### 基础信息

```
root@ip-172-31-3-170:~# uname -a
Linux ip-172-31-3-170 3.13.0-48-generic #80-Ubuntu SMP Thu Mar 12 11:16:15 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
root@ip-172-31-3-170:~#
root@ip-172-31-3-170:~# lsb_release -rd
Description:	Ubuntu 14.04.2 LTS
Release:	14.04
root@ip-172-31-3-170:~#
root@ip-172-31-3-170:~#
root@ip-172-31-3-170:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:9c:fe:63:4a:48
          inet addr:172.31.3.170  Bcast:172.31.15.255  Mask:255.255.240.0
          inet6 addr: fe80::9c:feff:fe63:4a48/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:46742619382 errors:0 dropped:0 overruns:0 frame:0
          TX packets:62063132186 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:15121196627407 (15.1 TB)  TX bytes:16933437224836 (16.9 TB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:2804 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2804 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:226648 (226.6 KB)  TX bytes:226648 (226.6 KB)

root@ip-172-31-3-170:~#
```

### 网络信息

#### 整体情况

```
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSE_WAIT 1
ESTABLISHED 277
SYN_SENT 23
TIME_WAIT 65547
root@ip-172-31-3-170:~#
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 275
FIN_WAIT1 2
SYN_SENT 13
TIME_WAIT 65536
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 292
SYN_SENT 16
TIME_WAIT 57737
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 302
SYN_SENT 16
TIME_WAIT 60184
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 297
SYN_SENT 9
TIME_WAIT 62441
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 283
SYN_SENT 20
TIME_WAIT 65424
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 277
SYN_SENT 28
TIME_WAIT 65540
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 292
CLOSING 1
SYN_SENT 8
TIME_WAIT 65536
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 291
SYN_SENT 9
TIME_WAIT 65536
root@ip-172-31-3-170:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 274
SYN_SENT 13
TIME_WAIT 56889
root@ip-172-31-3-170:~#
```

小结：

- 大量 TIME_WAIT（基本保持在 65536 左右，可能是正常正确，也可能异常情况，待确认）
- 少量 SYN_SENT（需要确认是否属于正常情况）
- 少量 ESTABLISHED（正常）


#### 监听端口信息

当前主机监听端口信息（主要为 tengine 监听）

```
root@ip-172-31-3-170:~# ss -nltp
State       Recv-Q Send-Q                                   Local Address:Port                                     Peer Address:Port
LISTEN      0      128                                                  *:22                                                  *:*      users:(("sshd",1165,3))
LISTEN      0      128                                                  *:23334                                               *:*      users:(("ruby",20200,56))
LISTEN      0      128                                                  *:80                                                  *:*      users:(("nginx",32421,18),("nginx",32420,18),("nginx",32419,18),("nginx",32418,18),("nginx",32417,18),("nginx",32416,18),("nginx",32415,18),("nginx",32414,18),("nginx",1468,18))
LISTEN      0      128                                                 :::22                                                 :::*      users:(("sshd",1165,4))
LISTEN      0      128                                                 :::9100                                               :::*      users:(("node_exporter",1122,3))
root@ip-172-31-3-170:~#
```

#### TIME-WAIT 分布

某一时刻 TIME-WAIT 连接分布采样：

```
root@ip-172-31-3-170:~# ss -tan | grep "TIME-WAIT" | awk '{print $(NF)" "$(NF-1)}' | sed 's/:[^ ]*//g' | sort | uniq -c
   1615 172.1.36.132 172.31.3.170
   1640 172.1.37.106 172.31.3.170
   1621 172.1.38.108 172.31.3.170
   1648 172.1.39.66 172.31.3.170
      1 172.1.41.128 172.31.3.170
   1628 172.1.43.2 172.31.3.170
   1590 172.1.43.33 172.31.3.170
   1617 172.1.43.46 172.31.3.170
   1627 172.1.44.48 172.31.3.170
   1614 172.1.47.41 172.31.3.170
   1636 172.1.50.119 172.31.3.170
   1618 172.1.56.178 172.31.3.170
   1619 172.1.56.22 172.31.3.170
   1619 172.1.56.224 172.31.3.170
   1626 172.1.57.200 172.31.3.170   -- aws:autoscaling:groupName:
   1603 172.1.58.80 172.31.3.170       nodes.prod0-cluster-cn-north1a.llsops.com
  14414 172.31.1.240 172.31.3.170   -- 对应 kafka-1-2
     53 172.31.10.10 172.31.3.170
     43 172.31.129.89 172.31.3.170
     61 172.31.135.29 172.31.3.170
     53 172.31.139.181 172.31.3.170
     56 172.31.142.162 172.31.3.170
     52 172.31.143.7 172.31.3.170
     58 172.31.15.127 172.31.3.170
  13180 172.31.15.64 172.31.3.170  -- 对应 kafka-1-0
  13083 172.31.2.205 172.31.3.170  -- 对应 kafka-1-1
     53 172.31.7.213 172.31.3.170
     58 172.31.8.11 172.31.3.170
      1 172.31.8.47 172.31.3.170
     49 172.31.9.250 172.31.3.170
root@ip-172-31-3-170:~#
...
...(N 天后另一次采样)...
...
root@ip-172-31-3-170:~# ss -tan | grep "TIME-WAIT" | awk '{print $(NF)" "$(NF-1)}' | sed 's/:[^ ]*//g' | sort | uniq -c
   2660 172.1.36.132 172.31.3.170
   2681 172.1.37.106 172.31.3.170
   2684 172.1.38.108 172.31.3.170
   2658 172.1.39.66 172.31.3.170
      2 172.1.41.128 172.31.3.170
   2676 172.1.43.2 172.31.3.170
   2693 172.1.44.23 172.31.3.170
   2660 172.1.44.48 172.31.3.170
   2662 172.1.47.41 172.31.3.170
   2699 172.1.50.119 172.31.3.170
      2 172.1.55.148 172.31.3.170
   2675 172.1.56.178 172.31.3.170
   2669 172.1.56.22 172.31.3.170
   2695 172.1.56.224 172.31.3.170
   2683 172.1.57.200 172.31.3.170
   2649 172.1.58.80 172.31.3.170
   7574 172.31.1.240 172.31.3.170
      2 172.31.10.91 172.31.3.170
     96 172.31.13.39 172.31.3.170
     92 172.31.134.142 172.31.3.170
     89 172.31.136.115 172.31.3.170
    107 172.31.139.156 172.31.3.170
     83 172.31.139.20 172.31.3.170
   7700 172.31.15.64 172.31.3.170
      1 172.31.2.129 172.31.3.170
   7758 172.31.2.205 172.31.3.170
    102 172.31.3.11 172.31.3.170
     84 172.31.3.135 172.31.3.170
     33 172.31.4.158 172.31.3.170
     86 172.31.5.208 172.31.3.170
      1 172.31.7.117 172.31.3.170
      5 172.31.8.21 172.31.3.170
      2 172.31.8.47 172.31.3.170
root@ip-172-31-3-170:~#
```

小结：

- （第一次采样）三个 1.3w (*3) 左右的 TIME-WAIT 分布均对应 kafka ；
- （第一次采样）所有 1.6k (*15) 左右的 TIME-WAIT 分布均对应 autoscaling group ，即 k8s EC2 实例；
- 其余的均定位不到具体机器；
- 两次采样发现，有些 ip 消失了（对应 EC2 实例销毁），有些仍旧存在；
- 上述 ip 地址均对应 EC2 实例地址；
- 通过 tengine 的配置文件知道，neo、neo-cc、neo-extra、neo-pay 四个服务部署在 80 个相同的 EC2 实例上，而上述 TIME-WAIT 状态连接分布主要集中在 14 个左右 EC2 上（后续了解到，当前 tengine 配置上 80 个后端实例中已经有很多处于下线状态，需要移除）；；


### TIME_WAIT 问题

相关参数如下

```
net.ipv4.tcp_max_tw_buckets = 65536
net.ipv4.ip_local_port_range = 32768	61000

net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 0

fs.file-max = 1632410
net.core.somaxconn = 128
net.ipv4.tcp_max_syn_backlog = 512
```

> 详细参数说明：[这里](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)

小结：

- tw 的 recycle 和 reuse 功能未启用；
- 系统同一时间维护的 timewait sockets 最多为 65536 个；
- 本地可用随机端口数量 2.8w 左右；


#### tcp_tw_recycle 和 tcp_tw_reuse 功能启用问题

> 核心问题：
>
> - 什么场景下应该启用上述参数？

内核参数

```
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 0
```

> TODO

#### tcp_max_tw_buckets 数值调整问题

> 核心问题：
>
> - 多少 TIME_WAIT 算多？
> - tcp_max_tw_buckets 超限后会有什么问题？

内核参数

```
net.ipv4.tcp_max_tw_buckets = 65536
```

通过如下命令可以查看并计算当前系统中全部 TIME_WAIT 状态的连接共计占用的内存总量；

公式如下：

- **CACHE SIZE** = OBJS * SIZE
- **OBJS** = SLABS * OBJ/SLAB
- **USE** = ACTIVE / OBJS

结论为，**6.4w TIME_WAIT 连接大概占用 16.4M 内存**；

```
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 65824  65824 100%    0.25K   2057       32     16456K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 65824  65824 100%    0.25K   2057       32     16456K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 64832  58360  90%    0.25K   2026       32     16208K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 64832  60759  93%    0.25K   2026       32     16208K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 64832  63151  97%    0.25K   2026       32     16208K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 64832  64787  99%    0.25K   2026       32     16208K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 65824  65824 100%    0.25K   2057       32     16456K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 65824  65824 100%    0.25K   2057       32     16456K tw_sock_TCP
root@ip-172-31-3-170:~# slabtop -o | grep -E '(^  OBJS|tw_sock_TCP|tcp_bind_bucket)'
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 65824  65824 100%    0.25K   2057       32     16456K tw_sock_TCP
root@ip-172-31-3-170:~#
```

小结：

- 65536 数量的 TIME_WAIT 真心不算多；一般来讲，15w 以下不太需要考虑性能问题（经验之谈）；
- 由于 TIME_WAIT 的数量取决于 `net.ipv4.ip_local_port_range` 和 `net.ipv4.tcp_max_tw_buckets` 中的小值，因此 `tcp_max_tw_buckets` 的设置需要和 `ip_local_port_range` 综合在一起考虑；
- tcp_max_tw_buckets 超限后，主动关闭 socket 的一方将跳过 TIME_WAIT 状态，直接进入 CLOSED，会让 TCP 变得不再可靠；当被动关闭的一方早先发出的延迟包到达后，就可能出现类似下面的问题：
    - 旧 TCP 连接已经不存在了，系统此时只能返回 RST 包；
    - 新 TCP 连接被建立起来了，延迟包可能干扰新的连接；
- 另外，tcp_max_tw_buckets 超限后会输出 "**TCP: time wait bucket table overflow**" 日志到 `/var/log/message` 中（当前系统没有放开）


> 补充：
>
> 使用了一种“绕”的方式用于确定当前系统可达 TIME_WAIT 的上限，步骤如下
> 
> - `tcp_max_tw_buckets` 保持 65536 不变
> - 通过 `iptables -t nat -L` 间接放开 nf_conntrack 功能；
> - 通过 `sysctl -w net.netfilter.nf_conntrack_max=262144` 将其默认值 65536 调整为 262144 ；此调整要快，应为 65536 很快就会达到；
> - 观察 `net.netfilter.nf_conntrack_count` 数值，直到其稳定；稳定时的数值基本上（略大于）就是当前系统面临的 TIME_WAIT 数量上限；
> - 通过 `sysctl -w net.ipv4.tcp_max_tw_buckets=180000` 调整 tw buckets 默认 65536 到 180000 ；
> - 观察 `ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'` 输出，稳定后的 TIME_WAIT 数量和 `net.netfilter.nf_conntrack_count` 数值差不多（略小于）；
> - 通过 `modprobe -r iptable_nat nf_nat_ipv4 nf_conntrack_ipv4 nf_defrag_ipv4 nf_nat nf_conntrack` 关闭 nf_conntrack 功能；
> - 需要将针对 tcp_max_tw_buckets 的参数调整写入 `/etc/sysctl.conf` 文件；

调整后效果如下：

```
root@ip-172-31-3-170:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
LISTEN 5
ESTAB 280
SYN-SENT 26
TIME-WAIT 125708
UNCONN 9
root@ip-172-31-3-170:~#
```


#### 可用随机端口超限问题

内核参数

```
net.ipv4.ip_local_port_range = 32768	61000
```

- 调整 tcp_max_tw_buckets 前

随机端口使用量分布为（移除了大量只有一个链接的输出行）

```
root@ip-172-31-3-170:/usr/local/nginx/conf/sites-enabled# ss -tan | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
      2  :::*
      1  ::ffff:172.31.4.6:50886
      3 * *:*
      1 172.31.3.170 172.1.35.188:32426
     13 172.31.3.170 172.1.36.132:30217
    627 172.31.3.170 172.1.36.132:30720
    344 172.31.3.170 172.1.36.132:30966
      4 172.31.3.170 172.1.36.132:31889
      3 172.31.3.170 172.1.36.132:32206
   1193 172.31.3.170 172.1.36.132:32426
      3 172.31.3.170 172.1.36.132:32720
      1 172.31.3.170 172.1.36.190:32426
     17 172.31.3.170 172.1.37.106:30217
    626 172.31.3.170 172.1.37.106:30720
    363 172.31.3.170 172.1.37.106:30966
      4 172.31.3.170 172.1.37.106:31889
      3 172.31.3.170 172.1.37.106:32206
   1296 172.31.3.170 172.1.37.106:32426
      3 172.31.3.170 172.1.37.106:32720
     14 172.31.3.170 172.1.38.108:30217
    582 172.31.3.170 172.1.38.108:30720
    352 172.31.3.170 172.1.38.108:30966
      4 172.31.3.170 172.1.38.108:31889
      3 172.31.3.170 172.1.38.108:32206
   1251 172.31.3.170 172.1.38.108:32426
      4 172.31.3.170 172.1.38.108:32720
     16 172.31.3.170 172.1.39.66:30217
    598 172.31.3.170 172.1.39.66:30720
    364 172.31.3.170 172.1.39.66:30966
      4 172.31.3.170 172.1.39.66:31889
      4 172.31.3.170 172.1.39.66:32206
   1355 172.31.3.170 172.1.39.66:32426
      3 172.31.3.170 172.1.39.66:32720
      1 172.31.3.170 172.1.41.128:80
      1 172.31.3.170 172.1.41.163:32720
      1 172.31.3.170 172.1.43.114:31889
      1 172.31.3.170 172.1.43.161:32720
     15 172.31.3.170 172.1.43.2:30217
    627 172.31.3.170 172.1.43.2:30720
    361 172.31.3.170 172.1.43.2:30966
      4 172.31.3.170 172.1.43.2:31889
      3 172.31.3.170 172.1.43.2:32206
   1258 172.31.3.170 172.1.43.2:32426
      4 172.31.3.170 172.1.43.2:32720
      1 172.31.3.170 172.1.43.33:32720
      1 172.31.3.170 172.1.43.46:32720
      1 172.31.3.170 172.1.44.214:32720
     17 172.31.3.170 172.1.44.23:30217
    641 172.31.3.170 172.1.44.23:30720
    355 172.31.3.170 172.1.44.23:30966
      4 172.31.3.170 172.1.44.23:31889
      3 172.31.3.170 172.1.44.23:32206
   1323 172.31.3.170 172.1.44.23:32426
      6 172.31.3.170 172.1.44.23:32720
     11 172.31.3.170 172.1.44.48:30217
    628 172.31.3.170 172.1.44.48:30720
    366 172.31.3.170 172.1.44.48:30966
      3 172.31.3.170 172.1.44.48:31889
      3 172.31.3.170 172.1.44.48:32206
   1329 172.31.3.170 172.1.44.48:32426
      4 172.31.3.170 172.1.44.48:32720
      1 172.31.3.170 172.1.47.109:30217
      1 172.31.3.170 172.1.47.121:32720
      1 172.31.3.170 172.1.47.123:30966
     14 172.31.3.170 172.1.47.41:30217
    622 172.31.3.170 172.1.47.41:30720
    364 172.31.3.170 172.1.47.41:30966
      4 172.31.3.170 172.1.47.41:31889
      4 172.31.3.170 172.1.47.41:32206
   1223 172.31.3.170 172.1.47.41:32426
      3 172.31.3.170 172.1.47.41:32720
      1 172.31.3.170 172.1.48.77:31889
     16 172.31.3.170 172.1.50.119:30217
    579 172.31.3.170 172.1.50.119:30720
    356 172.31.3.170 172.1.50.119:30966
      4 172.31.3.170 172.1.50.119:31889
      3 172.31.3.170 172.1.50.119:32206
   1317 172.31.3.170 172.1.50.119:32426
      5 172.31.3.170 172.1.50.119:32720
      1 172.31.3.170 172.1.51.145:30217
      1 172.31.3.170 172.1.51.89:30217
      1 172.31.3.170 172.1.51.89:32720
      1 172.31.3.170 172.1.52.112:32720
      1 172.31.3.170 172.1.52.147:32426
      1 172.31.3.170 172.1.52.201:30966
      1 172.31.3.170 172.1.55.148:80
      1 172.31.3.170 172.1.55.44:30720
     13 172.31.3.170 172.1.56.178:30217
    642 172.31.3.170 172.1.56.178:30720
    350 172.31.3.170 172.1.56.178:30966
      2 172.31.3.170 172.1.56.178:31889
      4 172.31.3.170 172.1.56.178:32206
   1244 172.31.3.170 172.1.56.178:32426
      4 172.31.3.170 172.1.56.178:32720
      1 172.31.3.170 172.1.56.194:30966
     15 172.31.3.170 172.1.56.224:30217
    632 172.31.3.170 172.1.56.224:30720
    339 172.31.3.170 172.1.56.224:30966
      3 172.31.3.170 172.1.56.224:31889
      4 172.31.3.170 172.1.56.224:32206
   1343 172.31.3.170 172.1.56.224:32426
      3 172.31.3.170 172.1.56.224:32720
     16 172.31.3.170 172.1.56.22:30217
    635 172.31.3.170 172.1.56.22:30720
    360 172.31.3.170 172.1.56.22:30966
      4 172.31.3.170 172.1.56.22:31889
      2 172.31.3.170 172.1.56.22:32206
   1327 172.31.3.170 172.1.56.22:32426
      4 172.31.3.170 172.1.56.22:32720
     17 172.31.3.170 172.1.57.200:30217
    603 172.31.3.170 172.1.57.200:30720
    361 172.31.3.170 172.1.57.200:30966
      4 172.31.3.170 172.1.57.200:31889
      3 172.31.3.170 172.1.57.200:32206
   1306 172.31.3.170 172.1.57.200:32426
      3 172.31.3.170 172.1.57.200:32720
     14 172.31.3.170 172.1.58.80:30217
    621 172.31.3.170 172.1.58.80:30720
    347 172.31.3.170 172.1.58.80:30966
      4 172.31.3.170 172.1.58.80:31889
      2 172.31.3.170 172.1.58.80:32206
   1254 172.31.3.170 172.1.58.80:32426
      4 172.31.3.170 172.1.58.80:32720
      1 172.31.3.170 172.1.60.161:31889
      1 172.31.3.170 172.1.60.187:31889
      1 172.31.3.170 172.1.61.1:31889
      1 172.31.3.170 172.1.61.250:30720
      1 172.31.3.170 172.1.61.250:32720
      1 172.31.3.170 172.1.62.98:30966
      1 172.31.3.170 172.1.63.76:30217
   4809 172.31.3.170 172.31.1.240:9092
      1 172.31.3.170 172.31.10.66:52755
      ...
      1 172.31.3.170 172.31.139.20:64953
   4720 172.31.3.170 172.31.15.64:9092
      1 172.31.3.170 172.31.15.99:61406
      ...
      1 172.31.3.170 172.31.2.129:37540
   4966 172.31.3.170 172.31.2.205:9092
      1 172.31.3.170 172.31.3.11:7807
      ...
      1 172.31.3.170 172.31.8.47:32931
      1 Peer Address:Port
root@ip-172-31-3-170:/usr/local/nginx/conf/sites-enabled#
```

可以看到，调整 tcp_max_tw_buckets 前，随机端口使用未达到上限，最多的情况才 4000+ 个端口（kafka 使用最多）；

- 调整 tcp_max_tw_buckets 后

随机端口使用量分布为（移除了大量只有一个链接的输出行）

```
root@ip-172-31-3-170:~#  ss -tan | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
      2  :::*
      1  ::ffff:172.31.4.6:41216
      3 * *:*
     23 172.31.3.170 172.1.36.132:30217
   1206 172.31.3.170 172.1.36.132:30720
    784 172.31.3.170 172.1.36.132:30966
      3 172.31.3.170 172.1.36.132:31889
      3 172.31.3.170 172.1.36.132:32206
   2438 172.31.3.170 172.1.36.132:32426
      4 172.31.3.170 172.1.36.132:32720
     23 172.31.3.170 172.1.37.106:30217
   1205 172.31.3.170 172.1.37.106:30720
    796 172.31.3.170 172.1.37.106:30966
      3 172.31.3.170 172.1.37.106:31889
      3 172.31.3.170 172.1.37.106:32206
   2431 172.31.3.170 172.1.37.106:32426
      3 172.31.3.170 172.1.37.106:32720
     26 172.31.3.170 172.1.38.108:30217
   1210 172.31.3.170 172.1.38.108:30720
    791 172.31.3.170 172.1.38.108:30966
      3 172.31.3.170 172.1.38.108:31889
      3 172.31.3.170 172.1.38.108:32206
   2427 172.31.3.170 172.1.38.108:32426
      4 172.31.3.170 172.1.38.108:32720
      1 172.31.3.170 172.1.38.244:32426
     27 172.31.3.170 172.1.39.66:30217
   1207 172.31.3.170 172.1.39.66:30720
    797 172.31.3.170 172.1.39.66:30966
      4 172.31.3.170 172.1.39.66:31889
      4 172.31.3.170 172.1.39.66:32206
   2431 172.31.3.170 172.1.39.66:32426
      4 172.31.3.170 172.1.39.66:32720
     24 172.31.3.170 172.1.43.2:30217
   1220 172.31.3.170 172.1.43.2:30720
    790 172.31.3.170 172.1.43.2:30966
      3 172.31.3.170 172.1.43.2:31889
      3 172.31.3.170 172.1.43.2:32206
   2413 172.31.3.170 172.1.43.2:32426
      4 172.31.3.170 172.1.43.2:32720
     23 172.31.3.170 172.1.44.23:30217
   1221 172.31.3.170 172.1.44.23:30720
    801 172.31.3.170 172.1.44.23:30966
      5 172.31.3.170 172.1.44.23:31889
      5 172.31.3.170 172.1.44.23:32206
   2435 172.31.3.170 172.1.44.23:32426
      3 172.31.3.170 172.1.44.23:32720
     24 172.31.3.170 172.1.44.48:30217
   1211 172.31.3.170 172.1.44.48:30720
    796 172.31.3.170 172.1.44.48:30966
      4 172.31.3.170 172.1.44.48:31889
      3 172.31.3.170 172.1.44.48:32206
   2405 172.31.3.170 172.1.44.48:32426
      3 172.31.3.170 172.1.44.48:32720
      1 172.31.3.170 172.1.47.109:30720
      1 172.31.3.170 172.1.47.109:32426
     22 172.31.3.170 172.1.47.41:30217
   1202 172.31.3.170 172.1.47.41:30720
    805 172.31.3.170 172.1.47.41:30966
      4 172.31.3.170 172.1.47.41:31889
      3 172.31.3.170 172.1.47.41:32206
   2433 172.31.3.170 172.1.47.41:32426
      3 172.31.3.170 172.1.47.41:32720
      1 172.31.3.170 172.1.47.54:32206
      1 172.31.3.170 172.1.48.120:32426
      1 172.31.3.170 172.1.48.194:32206
     25 172.31.3.170 172.1.50.119:30217
   1221 172.31.3.170 172.1.50.119:30720
    803 172.31.3.170 172.1.50.119:30966
      3 172.31.3.170 172.1.50.119:31889
      3 172.31.3.170 172.1.50.119:32206
   2421 172.31.3.170 172.1.50.119:32426
      3 172.31.3.170 172.1.50.119:32720
     22 172.31.3.170 172.1.56.178:30217
   1230 172.31.3.170 172.1.56.178:30720
    799 172.31.3.170 172.1.56.178:30966
      4 172.31.3.170 172.1.56.178:31889
      3 172.31.3.170 172.1.56.178:32206
   2434 172.31.3.170 172.1.56.178:32426
      3 172.31.3.170 172.1.56.178:32720
     20 172.31.3.170 172.1.56.224:30217
   1216 172.31.3.170 172.1.56.224:30720
    795 172.31.3.170 172.1.56.224:30966
      3 172.31.3.170 172.1.56.224:31889
      3 172.31.3.170 172.1.56.224:32206
   2434 172.31.3.170 172.1.56.224:32426
      6 172.31.3.170 172.1.56.224:32720
     24 172.31.3.170 172.1.56.22:30217
   1223 172.31.3.170 172.1.56.22:30720
    806 172.31.3.170 172.1.56.22:30966
      5 172.31.3.170 172.1.56.22:31889
      4 172.31.3.170 172.1.56.22:32206
   2430 172.31.3.170 172.1.56.22:32426
      5 172.31.3.170 172.1.56.22:32720
     27 172.31.3.170 172.1.57.200:30217
   1219 172.31.3.170 172.1.57.200:30720
    786 172.31.3.170 172.1.57.200:30966
      3 172.31.3.170 172.1.57.200:31889
      3 172.31.3.170 172.1.57.200:32206
   2419 172.31.3.170 172.1.57.200:32426
      4 172.31.3.170 172.1.57.200:32720
     23 172.31.3.170 172.1.58.80:30217
   1220 172.31.3.170 172.1.58.80:30720
    797 172.31.3.170 172.1.58.80:30966
      5 172.31.3.170 172.1.58.80:31889
      4 172.31.3.170 172.1.58.80:32206
   2422 172.31.3.170 172.1.58.80:32426
      5 172.31.3.170 172.1.58.80:32720
      1 172.31.3.170 172.1.59.231:30966
      1 172.31.3.170 172.1.60.161:32206
      1 172.31.3.170 172.1.60.192:30217
      1 172.31.3.170 172.1.63.150:30217
      1 172.31.3.170 172.1.63.76:32720
  17264 172.31.3.170 172.31.1.240:9092
      1 172.31.3.170 172.31.10.66:55258
      ...
      1 172.31.3.170 172.31.139.20:44664
  17268 172.31.3.170 172.31.15.64:9092
      1 172.31.3.170 172.31.15.99:3899
      1 172.31.3.170 172.31.15.99:4119
      1 172.31.3.170 172.31.2.129:24707
      1 172.31.3.170 172.31.2.129:24913
  17968 172.31.3.170 172.31.2.205:9092
      1 172.31.3.170 172.31.3.11:44395
      ...
      1 172.31.3.170 172.31.8.47:3408
      1 Peer Address:Port
root@ip-172-31-3-170:~#
```

可以看到，调整 tcp_max_tw_buckets 后，随机端口使用未达到上限，最多的情况才 17968+ 个端口（仍旧是 kafka 使用最多）；


### SYN_SENT 问题

> 核心问题：
>
> - SYN_SENT 出现的原因？
> - SYN_SENT 出现是否合理？

数据观察：

- 采样间隔不能设置为 1s ，因为实际观察发现 500ms 的情况下才能猜到 2 个样本；
- 采样时间最好得到 2min 以上，否则很难总结出规律；
- 能够发现 dst ip + dst port 的组合按照规律出现，而对应的 src port 是随机变化的；

```
root@ip-172-31-3-170:~# while true; do ss -n -o | grep "SYN-SENT"; sleep 0.5; echo "---"; done
tcp    SYN-SENT   0      1           172.31.3.170:60128      172.1.32.128:32426  timer:(on,968ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:59933      172.1.60.118:30966  timer:(on,516ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:41741      172.1.51.145:31889  timer:(on,548ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:56082      172.1.48.120:30217  timer:(on,608ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:37874       172.1.51.89:30217  timer:(on,040ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:58522      172.1.58.208:30217  timer:(on,964ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:34101      172.1.60.188:30720  timer:(on,140ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:34886      172.1.49.156:32426  timer:(on,964ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:40156       172.1.63.76:31889  timer:(on,276ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:51343      172.1.47.109:32720  timer:(on,540ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:51717      172.1.35.188:30217  timer:(on,960ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:56443      172.1.43.161:32426  timer:(on,508ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:36613       172.1.63.76:30217  timer:(on,960ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:47568      172.1.43.161:30966  timer:(on,276ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:50576        172.1.39.4:30720  timer:(on,540ms,0)
---
tcp    SYN-SENT   0      1           172.31.3.170:60128      172.1.32.128:32426  timer:(on,444ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:58906      172.1.43.114:31889  timer:(on,940ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:51089      172.1.44.243:30966  timer:(on,616ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:35080      172.1.40.158:30720  timer:(on,940ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:41741      172.1.51.145:31889  timer:(on,024ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:56082      172.1.48.120:30217  timer:(on,084ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:58522      172.1.58.208:30217  timer:(on,440ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:33076      172.1.54.196:32720  timer:(on,612ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:43954      172.1.44.214:30720  timer:(on,936ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:34886      172.1.49.156:32426  timer:(on,436ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:51343      172.1.47.109:32720  timer:(on,016ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:51717      172.1.35.188:30217  timer:(on,436ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:41943        172.1.61.1:30966  timer:(on,708ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:36613       172.1.63.76:30217  timer:(on,436ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:50576        172.1.39.4:30720  timer:(on,016ms,0)
tcp    SYN-SENT   0      1           172.31.3.170:35410       172.1.51.89:31889  timer:(on,640ms,0)
---
...
```

当前结论：

- 基本上可以确定当前看到的 SYN-SENT 是由于 tengine 对 upstream 进行心跳探测导致；
- 需要知道的是，在网络环境很好的情况下，心跳探测交互的非常快，则很难看到 SYN-SENT 的出现，这里能看到一个“保持”状态（从定时器计时结果可以看到），**至少说明了心跳交互至少存在 500ms 的延时**（不确定应用本身的请求是否有类似问题，怀疑 tengine 针对已经不存在的 backend 节点进行健康监测时会有 500ms 延时情况）；


### conntrack 问题

> 核心问题：
>
> - conntrack 是用来干什么的？
> - tengine 机器上是否要开启该功能？
> - 开启后需要注意什么问题？
> - 如何开启 conntrack 功能？
> - 如何关闭 conntrack 功能？

和 conntrack 相关的内核参数（开启 conntrack 功能后才有）

```
root@ip-172-31-3-170:/var/log# sysctl -a|grep conntrack
net.netfilter.nf_conntrack_acct = 0
net.netfilter.nf_conntrack_buckets = 16384
net.netfilter.nf_conntrack_checksum = 1
net.netfilter.nf_conntrack_count = 65422
net.netfilter.nf_conntrack_events = 1
net.netfilter.nf_conntrack_events_retry_timeout = 15
net.netfilter.nf_conntrack_expect_max = 256
net.netfilter.nf_conntrack_generic_timeout = 600
net.netfilter.nf_conntrack_helper = 1
net.netfilter.nf_conntrack_icmp_timeout = 30
net.netfilter.nf_conntrack_log_invalid = 0
net.netfilter.nf_conntrack_max = 65536
net.netfilter.nf_conntrack_tcp_be_liberal = 0
net.netfilter.nf_conntrack_tcp_loose = 1
net.netfilter.nf_conntrack_tcp_max_retrans = 3
net.netfilter.nf_conntrack_tcp_timeout_close = 10
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_established = 432000
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_last_ack = 30
net.netfilter.nf_conntrack_tcp_timeout_max_retrans = 300
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_unacknowledged = 300
net.netfilter.nf_conntrack_timestamp = 0
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
net.nf_conntrack_max = 65536
root@ip-172-31-3-170:/var/log#
```

> 详细参数说明：[这里](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)

可以看到，nf_conntrack_count 已经接近 nf_conntrack_max 上限值；

```
net.netfilter.nf_conntrack_count = 65422
...
net.netfilter.nf_conntrack_max = 65536
```

此时 kern.log 中会间歇性在刷如下日志（通过 `dmesg` 命令也能看到）

```
root@ip-172-31-3-170:/var/log# tail -f kern.log
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.185260] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.187128] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.187284] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.187414] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.189815] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.190977] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.191152] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.191216] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.198360] nf_conntrack: table full, dropping packet
Mar  1 09:41:21 ip-172-31-3-170 kernel: [12449650.202865] nf_conntrack: table full, dropping packet

（停顿片刻）

Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.206163] net_ratelimit: 466 callbacks suppressed
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.206166] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.208071] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.208076] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.209518] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.210556] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.431035] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.520073] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.520083] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.520098] nf_conntrack: table full, dropping packet
Mar  1 09:41:26 ip-172-31-3-170 kernel: [12449655.524071] nf_conntrack: table full, dropping packet
^C
root@ip-172-31-3-170:/var/log#
```

小结：

- 系统中默认应该不会打开 nf_conntrack 功能；
- nf_conntrack 和 NAT 功能有关，用来跟踪连接条目；
- 执行一些和 NAT 相关的 iptables 命令会导致内核模块 nf_conntrack 被 load ，相应功能的打开，例如 `iptables -t nat -L` 命令；
- 在 proxy 类机器上不应该打开该功能；


**补充信息**：

在 dmesg 的输出内容中还可以看到（前提是 kern.log 中的内容没有被大量错误信息给覆盖掉）

```
...
[    0.656799] TCP established hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.660962] TCP bind hash table entries: 65536 (order: 8, 1048576 bytes)
[    0.664468] TCP: Hash tables configured (established 131072 bind 65536)
[    0.667852] TCP: reno registered
[    0.669794] UDP hash table entries: 8192 (order: 6, 262144 bytes)
[    0.672929] UDP-Lite hash table entries: 8192 (order: 6, 262144 bytes)
...
[    1.088294] TCP: cubic registered
```

其中：

- "TCP established hash table" 即 TCP 连接存储哈希表；
- "TCP bind hash table" 即 TCP 绑定端口哈希表；
- 连接存储哈希表中维护了多种 TCP 状态的连接（都有哪些状态？）
- 影响绑定端口哈希表中数量的访问类型（inbound or outbound）


----------

## tengine 配置

- /usr/local/nginx/conf/nginx.conf

```
root@ip-172-31-3-170:~# cat /usr/local/nginx/conf/nginx.conf
user www-data;
worker_processes  8;

pid /var/run/tengine.pid;

worker_rlimit_nofile 16384;
events {
  worker_connections 16384;
}

http {

  include       /usr/local/nginx/conf/mime.types;
  default_type  application/octet-stream;

  log_format upstream '$time_iso8601 $remote_addr $host "$upstream_response_time" $request $status "$upstream_addr"';
  #log_format elb_combined '$http_x_forwarded_for - $remote_user [$time_local] "$request_without_token" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$upstream_response_time" $upstream_http_x_login';
  log_format elb_combined '$http_x_forwarded_for - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$upstream_response_time" $upstream_http_x_login';

  access_log /var/log/nginx/access.log elb_combined;
  access_log /var/log/nginx/upstream.log upstream;
  error_log /var/log/nginx/error.log;

  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;

  keepalive_timeout  65;

  gzip  on;
  gzip_http_version 1.0;
  gzip_comp_level 2;
  gzip_proxied any;
  gzip_vary off;
  gzip_types text/plain text/css application/x-javascript text/xml application/xml application/rss+xml application/atom+xml text/javascript application/javascript application/json text/mathml;
  gzip_min_length  1000;
  gzip_disable     "MSIE [1-6]\.";

  server_names_hash_bucket_size 64;
  types_hash_max_size 2048;
  types_hash_bucket_size 64;
  check_shm_size 20M;

  include /usr/local/nginx/conf/sites-enabled/*;
}
root@ip-172-31-3-170:~#
```

- /usr/local/nginx/conf/sites-enabled/neo

```
root@ip-172-31-3-170:~# cat /usr/local/nginx/conf/sites-enabled/neo
upstream neo {
    # dynamic_resolve fallback=stale fail_timeout=30s;
    # prod0_n1a neo-prod-web-inner-lb
    # server internal-a49e0bc1607ba11e7b48102ab8baa063-1634739253.cn-north-1.elb.amazonaws.com.cn:80;

    server 172.1.55.22:32426;
    server 172.1.47.123:32426;
    server 172.1.56.194:32426;
    server 172.1.63.76:32426;
    server 172.1.35.188:32426;
    server 172.1.36.82:32426;
    server 172.1.36.190:32426;
    server 172.1.47.109:32426;
    server 172.1.61.250:32426;
    server 172.1.44.243:32426;
    server 172.1.58.225:32426;
    server 172.1.39.19:32426;
    server 172.1.60.188:32426;
    server 172.1.48.194:32426;
    server 172.1.54.196:32426;
    server 172.1.51.144:32426;
    server 172.1.47.102:32426;
    server 172.1.61.1:32426;
    server 172.1.38.244:32426;
    server 172.1.49.156:32426;
    server 172.1.43.161:32426;
    server 172.1.38.108:32426;
    server 172.1.48.120:32426;
    server 172.1.37.106:32426;
    server 172.1.59.231:32426;
    server 172.1.41.163:32426;
    server 172.1.36.193:32426;
    server 172.1.40.121:32426;
    server 172.1.62.98:32426;
    server 172.1.44.27:32426;
    server 172.1.51.89:32426;
    server 172.1.58.208:32426;
    server 172.1.32.128:32426;
    server 172.1.47.83:32426;
    server 172.1.60.192:32426;
    server 172.1.52.158:32426;
    server 172.1.36.132:32426;
    server 172.1.60.115:32426;
    server 172.1.58.80:32426;
    server 172.1.47.54:32426;
    server 172.1.44.214:32426;
    server 172.1.39.70:32426;
    server 172.1.44.23:32426;
    server 172.1.55.44:32426;
    server 172.1.48.77:32426;
    server 172.1.63.150:32426;
    server 172.1.54.150:32426;
    server 172.1.34.190:32426;
    server 172.1.60.104:32426;
    server 172.1.40.158:32426;
    server 172.1.52.112:32426;
    server 172.1.56.178:32426;
    server 172.1.60.161:32426;
    server 172.1.38.207:32426;
    server 172.1.50.119:32426;
    server 172.1.55.37:32426;
    server 172.1.52.147:32426;
    server 172.1.55.225:32426;
    server 172.1.56.224:32426;
    server 172.1.39.66:32426;
    server 172.1.43.33:32426;
    server 172.1.56.22:32426;
    server 172.1.47.41:32426;
    server 172.1.43.2:32426;
    server 172.1.39.4:32426;
    server 172.1.43.46:32426;
    server 172.1.57.200:32426;
    server 172.1.47.127:32426;
    server 172.1.37.196:32426;
    server 172.1.60.118:32426;
    server 172.1.43.114:32426;
    server 172.1.61.72:32426;
    server 172.1.60.187:32426;
    server 172.1.47.121:32426;
    server 172.1.48.9:32426;
    server 172.1.60.44:32426;
    server 172.1.51.145:32426;
    server 172.1.52.201:32426;
    server 172.1.32.148:32426;
    server 172.1.44.48:32426;
    check interval=30000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
}

upstream neo-cc {
    # dynamic_resolve fallback=stale fail_timeout=30s;
    # prod0_n1a neo-prod-ccapi-inner-lb
    # server internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn:80;

    server 172.1.55.22:30720;
    server 172.1.47.123:30720;
    server 172.1.56.194:30720;
    server 172.1.63.76:30720;
    server 172.1.35.188:30720;
    server 172.1.36.82:30720;
    server 172.1.36.190:30720;
    server 172.1.47.109:30720;
    server 172.1.61.250:30720;
    server 172.1.44.243:30720;
    server 172.1.58.225:30720;
    server 172.1.39.19:30720;
    server 172.1.60.188:30720;
    server 172.1.48.194:30720;
    server 172.1.54.196:30720;
    server 172.1.51.144:30720;
    server 172.1.47.102:30720;
    server 172.1.61.1:30720;
    server 172.1.38.244:30720;
    server 172.1.49.156:30720;
    server 172.1.43.161:30720;
    server 172.1.38.108:30720;
    server 172.1.48.120:30720;
    server 172.1.37.106:30720;
    server 172.1.59.231:30720;
    server 172.1.41.163:30720;
    server 172.1.36.193:30720;
    server 172.1.40.121:30720;
    server 172.1.62.98:30720;
    server 172.1.44.27:30720;
    server 172.1.51.89:30720;
    server 172.1.58.208:30720;
    server 172.1.32.128:30720;
    server 172.1.47.83:30720;
    server 172.1.60.192:30720;
    server 172.1.52.158:30720;
    server 172.1.36.132:30720;
    server 172.1.60.115:30720;
    server 172.1.58.80:30720;
    server 172.1.47.54:30720;
    server 172.1.44.214:30720;
    server 172.1.39.70:30720;
    server 172.1.44.23:30720;
    server 172.1.55.44:30720;
    server 172.1.48.77:30720;
    server 172.1.63.150:30720;
    server 172.1.54.150:30720;
    server 172.1.34.190:30720;
    server 172.1.60.104:30720;
    server 172.1.40.158:30720;
    server 172.1.52.112:30720;
    server 172.1.56.178:30720;
    server 172.1.60.161:30720;
    server 172.1.38.207:30720;
    server 172.1.50.119:30720;
    server 172.1.55.37:30720;
    server 172.1.52.147:30720;
    server 172.1.55.225:30720;
    server 172.1.56.224:30720;
    server 172.1.39.66:30720;
    server 172.1.43.33:30720;
    server 172.1.56.22:30720;
    server 172.1.47.41:30720;
    server 172.1.43.2:30720;
    server 172.1.39.4:30720;
    server 172.1.43.46:30720;
    server 172.1.57.200:30720;
    server 172.1.47.127:30720;
    server 172.1.37.196:30720;
    server 172.1.60.118:30720;
    server 172.1.43.114:30720;
    server 172.1.61.72:30720;
    server 172.1.60.187:30720;
    server 172.1.47.121:30720;
    server 172.1.48.9:30720;
    server 172.1.60.44:30720;
    server 172.1.51.145:30720;
    server 172.1.52.201:30720;
    server 172.1.32.148:30720;
    server 172.1.44.48:30720;
    check interval=30000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
}

upstream neo-extra {
    # dynamic_resolve fallback=stale fail_timeout=30s;
    # prod0_n1a neo-prod-extra-inner-lb
    # server internal-a8b47d928241711e7b48102ab8baa063-1087606731.cn-north-1.elb.amazonaws.com.cn:80;

    server 172.1.55.22:30966;
    server 172.1.47.123:30966;
    server 172.1.56.194:30966;
    server 172.1.63.76:30966;
    server 172.1.35.188:30966;
    server 172.1.36.82:30966;
    server 172.1.36.190:30966;
    server 172.1.47.109:30966;
    server 172.1.61.250:30966;
    server 172.1.44.243:30966;
    server 172.1.58.225:30966;
    server 172.1.39.19:30966;
    server 172.1.60.188:30966;
    server 172.1.48.194:30966;
    server 172.1.54.196:30966;
    server 172.1.51.144:30966;
    server 172.1.47.102:30966;
    server 172.1.61.1:30966;
    server 172.1.38.244:30966;
    server 172.1.49.156:30966;
    server 172.1.43.161:30966;
    server 172.1.38.108:30966;
    server 172.1.48.120:30966;
    server 172.1.37.106:30966;
    server 172.1.59.231:30966;
    server 172.1.41.163:30966;
    server 172.1.36.193:30966;
    server 172.1.40.121:30966;
    server 172.1.62.98:30966;
    server 172.1.44.27:30966;
    server 172.1.51.89:30966;
    server 172.1.58.208:30966;
    server 172.1.32.128:30966;
    server 172.1.47.83:30966;
    server 172.1.60.192:30966;
    server 172.1.52.158:30966;
    server 172.1.36.132:30966;
    server 172.1.60.115:30966;
    server 172.1.58.80:30966;
    server 172.1.47.54:30966;
    server 172.1.44.214:30966;
    server 172.1.39.70:30966;
    server 172.1.44.23:30966;
    server 172.1.55.44:30966;
    server 172.1.48.77:30966;
    server 172.1.63.150:30966;
    server 172.1.54.150:30966;
    server 172.1.34.190:30966;
    server 172.1.60.104:30966;
    server 172.1.40.158:30966;
    server 172.1.52.112:30966;
    server 172.1.56.178:30966;
    server 172.1.60.161:30966;
    server 172.1.38.207:30966;
    server 172.1.50.119:30966;
    server 172.1.55.37:30966;
    server 172.1.52.147:30966;
    server 172.1.55.225:30966;
    server 172.1.56.224:30966;
    server 172.1.39.66:30966;
    server 172.1.43.33:30966;
    server 172.1.56.22:30966;
    server 172.1.47.41:30966;
    server 172.1.43.2:30966;
    server 172.1.39.4:30966;
    server 172.1.43.46:30966;
    server 172.1.57.200:30966;
    server 172.1.47.127:30966;
    server 172.1.37.196:30966;
    server 172.1.60.118:30966;
    server 172.1.43.114:30966;
    server 172.1.61.72:30966;
    server 172.1.60.187:30966;
    server 172.1.47.121:30966;
    server 172.1.48.9:30966;
    server 172.1.60.44:30966;
    server 172.1.51.145:30966;
    server 172.1.52.201:30966;
    server 172.1.32.148:30966;
    server 172.1.44.48:30966;
    check interval=30000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
}

upstream neo-pay {
    # dynamic_resolve fallback=stale fail_timeout=30s;
    # prod0_n1a neo-prod-payapi-inner-lb
    # server internal-a6d51411b574011e7b48102ab8baa063-28810528.cn-north-1.elb.amazonaws.com.cn:80;

    server 172.1.55.22:30217;
    server 172.1.47.123:30217;
    server 172.1.56.194:30217;
    server 172.1.63.76:30217;
    server 172.1.35.188:30217;
    server 172.1.36.82:30217;
    server 172.1.36.190:30217;
    server 172.1.47.109:30217;
    server 172.1.61.250:30217;
    server 172.1.44.243:30217;
    server 172.1.58.225:30217;
    server 172.1.39.19:30217;
    server 172.1.60.188:30217;
    server 172.1.48.194:30217;
    server 172.1.54.196:30217;
    server 172.1.51.144:30217;
    server 172.1.47.102:30217;
    server 172.1.61.1:30217;
    server 172.1.38.244:30217;
    server 172.1.49.156:30217;
    server 172.1.43.161:30217;
    server 172.1.38.108:30217;
    server 172.1.48.120:30217;
    server 172.1.37.106:30217;
    server 172.1.59.231:30217;
    server 172.1.41.163:30217;
    server 172.1.36.193:30217;
    server 172.1.40.121:30217;
    server 172.1.62.98:30217;
    server 172.1.44.27:30217;
    server 172.1.51.89:30217;
    server 172.1.58.208:30217;
    server 172.1.32.128:30217;
    server 172.1.47.83:30217;
    server 172.1.60.192:30217;
    server 172.1.52.158:30217;
    server 172.1.36.132:30217;
    server 172.1.60.115:30217;
    server 172.1.58.80:30217;
    server 172.1.47.54:30217;
    server 172.1.44.214:30217;
    server 172.1.39.70:30217;
    server 172.1.44.23:30217;
    server 172.1.55.44:30217;
    server 172.1.48.77:30217;
    server 172.1.63.150:30217;
    server 172.1.54.150:30217;
    server 172.1.34.190:30217;
    server 172.1.60.104:30217;
    server 172.1.40.158:30217;
    server 172.1.52.112:30217;
    server 172.1.56.178:30217;
    server 172.1.60.161:30217;
    server 172.1.38.207:30217;
    server 172.1.50.119:30217;
    server 172.1.55.37:30217;
    server 172.1.52.147:30217;
    server 172.1.55.225:30217;
    server 172.1.56.224:30217;
    server 172.1.39.66:30217;
    server 172.1.43.33:30217;
    server 172.1.56.22:30217;
    server 172.1.47.41:30217;
    server 172.1.43.2:30217;
    server 172.1.39.4:30217;
    server 172.1.43.46:30217;
    server 172.1.57.200:30217;
    server 172.1.47.127:30217;
    server 172.1.37.196:30217;
    server 172.1.60.118:30217;
    server 172.1.43.114:30217;
    server 172.1.61.72:30217;
    server 172.1.60.187:30217;
    server 172.1.47.121:30217;
    server 172.1.48.9:30217;
    server 172.1.60.44:30217;
    server 172.1.51.145:30217;
    server 172.1.52.201:30217;
    server 172.1.32.148:30217;
    server 172.1.44.48:30217;
    check interval=30000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
}

upstream neo-ncc {
    dynamic_resolve fallback=stale fail_timeout=30s;
    # prod0_n1a darwin-prod-inner-lb
    server internal-a36f4c99b780e11e7b48102ab8baa063-220306410.cn-north-1.elb.amazonaws.com.cn:80;
    # check interval=10000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
}

# limit_req_zone $nginx_version zone=neo_flood_limit:10m rate=200r/s;

server {
    listen 80 default_server;

    # Alibaba Public DNS
    resolver 223.5.5.5 223.6.6.6 valid=300s;
    resolver_timeout 10s;

    client_max_body_size 5m;
    client_body_buffer_size 3m;
    keepalive_timeout 10;
    proxy_http_version 1.1;

    server_name apineo.llsapp.com;
    root /opt/nginx/static/public;

    server_tokens off;

    # limit_req zone=neo_flood_limit;

    location / {
        proxy_pass http://neo;

        proxy_set_header X-Real-IP $http_x_forwarded_for;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-Queue-Start "t=${msec}000";
        proxy_redirect off;

        # This actually doesn't work with an ELB behind nginx
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout      10s;

        proxy_buffer_size          512k;
        proxy_buffers              8 512k;
        proxy_busy_buffers_size    1m;
        proxy_temp_file_write_size 1m;

        location /api/v1/cc {
          proxy_pass http://neo-cc;
        }
        location /api/v1/pt {
          proxy_pass http://neo-cc;
        }
        location /api/v1/user_klasses {
          proxy_pass http://neo-cc;
        }
        location /api/v1/live_sessions {
          proxy_pass http://neo-cc;
        }
        location /api/v1/klasses {
          proxy_pass http://neo-cc;
        }
        location /api/v1/klass_sessions {
          proxy_pass http://neo-cc;
        }

        location /api/data {
          proxy_pass http://neo-extra;
        }
        location /api/v1/userevents {
          proxy_pass http://neo-extra;
        }
        location /api/v1/trace  {
          proxy_pass http://neo-extra;
        }
        location /api/personas/ios/exist  {
          proxy_pass http://neo-extra;
        }

        # ========= neo-pay ==============
        location /api/payment {
          proxy_pass http://neo-pay;
        }
        location /api/payway {
          proxy_pass http://neo-pay;
        }
        location /api/gift_card {
          proxy_pass http://neo-pay;
        }
        location /api/v1/cc/packages {
          proxy_pass http://neo-pay;
        }
        location /api/v1/pronco/packages {
          proxy_pass http://neo-pay;
        }
        location /api/v1/cc/learning_goals/reward {
          proxy_pass http://neo-pay;
        }
        # ======== end of neo-pay ========
        # ======== ncc ========
        location /api/v1/ncc {
          proxy_pass http://neo-ncc;
        }
        # ======== end of ncc ========
    }

    if ($request_method !~ ^(GET|HEAD|PUT|POST|DELETE|OPTIONS)$ ){
        return 405;
    }
    location ~ \.(php|cgi)$ {
        return 405;
    }
}
root@ip-172-31-3-170:~#
```

以下关系基于 `/usr/local/nginx/conf/sites-enabled/neo` 中的配置进行梳理

| upstream  | port  | lb (prod0_n1a) | server |
| -- | -- | -- | -- |
| **neo**       | **32426** | neo-prod-web-inner-lb | internal-a49e0bc1607ba11e7b48102ab8baa063-1634739253.cn-north-1.elb.amazonaws.com.cn:80 |
| **neo-cc**    | **30720** | neo-prod-ccapi-inner-lb | internal-a87dff023241411e7b48102ab8baa063-1424689483.cn-north-1.elb.amazonaws.com.cn:80 |
| **neo-extra** | **30966** | neo-prod-extra-inner-lb | internal-a8b47d928241711e7b48102ab8baa063-1087606731.cn-north-1.elb.amazonaws.com.cn:80 |
| **neo-pay**   | **30217** | neo-prod-payapi-inner-lb | internal-a6d51411b574011e7b48102ab8baa063-28810528.cn-north-1.elb.amazonaws.com.cn:80 |
| **neo-ncc**   | N/A   | darwin-prod-inner-lb | internal-a36f4c99b780e11e7b48102ab8baa063-220306410.cn-north-1.elb.amazonaws.com.cn:80 |

并看到了每个 upstream 都有如下配置

```
check interval=30000 fall=3 rise=2 timeout=1000 default_down=false type=tcp;
```


以下关系基于 `/usr/local/nginx/conf/sites-enabled/neo_ops` 中的配置进行梳理

| upstream  | port  | lb (prod0_n1a) | server |
| -- | -- | -- | -- |
| neo-ops | 32206 | neo-prod-opsapi-lb | internal-a6146bc402bbc11e7b48102ab8baa063-1889674119.cn-north-1.elb.amazonaws.com.cn:80 |

以下关系基于 `/usr/local/nginx/conf/sites-enabled/homepage_api` 中的配置进行梳理

| upstream  | port  | lb (prod0_n1a) | server |
| -- | -- | -- | -- |
| homepage-api | 31889 | homepage-prod-api-lb | internal-a9cc0c476dfc611e6b48102ab8baa063-899340309.cn-north-1.elb.amazonaws.com.cn:80 |

以下关系基于 `/usr/local/nginx/conf/sites-enabled/link_agent_web` 中的配置进行梳理

| upstream  | port  | lb (prod0_n1a) | server |
| -- | -- | -- | -- |
| link-agent-web | 32720 | linkagent-prod-web-lb | internal-aa4e8d7a071db11e7b48102ab8baa063-61910742.cn-north-1.elb.amazonaws.com.cn:80 |