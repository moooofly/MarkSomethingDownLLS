# 流利说线上 k8s node 问题排查

结论：

- 出现 Recv-Q 溢出问题的监听端口均对应类型为 NodePod 和 LoadBalancer 的 Service ；
- 观察到 node 上出现的 CLOSE_WAIT 连接状态不会消失，且很多处于 CLOSE_WAIT 状态的连接上的 Recv-Q 中残留有大量数据，怀疑和 k8s 动态扩缩容调度时，启停业务的行为有关（业务自身存在问题的可能性也很大）；
- 针对单个监听端口来说，观察到 CLOSE_WAIT 数量会增长（尚不能准确判定导致增长的具体行为）；
- 监听端口在 Recv-Q 满的状态下，仍旧能够提供连接建立服务（但实际上，在该状态下会概率性出现重连、连接不上、连接成功等情况）；
- kube-proxy 不会为 Service port 建立监听，只会为其添加 iptables 转发规则；kube-proxy 会为 NodePort 或 LoadBalancer 建立监听，同时会为其添加 iptables 转发规则；
- 基于现有的观测数据可以发现：
    - 存在 kube-proxy 上的监听端口消失的情况，如 30225 端口（应该是对应的 NodePod 或 Service 被删除）；
    - 存在 kube-proxy 上的监听端口的 Recv-Q 增长的情况，如 32708(92->93)/31123(122->129)/32634(50->129) ；
    - 由于之前没有对 CLOSE_WAIT 分布数据进行留存，故暂时无法比对相应数据变化；

> 遗留问题: 
> 
> - accept queue 已满，如何确定其中的内容都是什么？
> - accept queue 已满，说明什么？会有什么影响？
> - 为何看到的 CLOSE_WAIT 的数值恰好不超过 129 ？

- 20180306 观测数据（圈出来的部分为数值不相同项）

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/20180306_172.1.57.200_%E8%A7%82%E6%B5%8B%E6%95%B0%E6%8D%AE.png)

```
root@ip-172-1-57-200:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 3068
ESTAB 18
TIME-WAIT 17
LISTEN 74
root@ip-172-1-57-200:~# 
root@ip-172-1-57-200:~# netstat -st
IcmpMsg:
    InType0: 3
    InType3: 390
    OutType3: 7906
    OutType5: 978210
    OutType8: 9
Tcp:
    6465567 active connections openings
    1032824 passive connection openings
    136 failed connection attempts
    52813 connection resets received
    19 connections established
    118230733 segments received
    105664162 segments send out
    9687019 segments retransmited
    122 bad segments received.
    1093760 resets sent
    InCsumErrors: 5
UdpLite:
TcpExt:
    1 resets received for embryonic SYN_RECV sockets
    897 packets pruned from receive queue because of socket buffer overrun
    3484145 TCP sockets finished time wait in fast timer
    55 packets rejects in established connections because of timestamp
    1349990 delayed acks sent
    456 delayed acks further delayed because of locked socket
    Quick ack mode was activated 1233 times
    30833253 times the listen queue of a socket overflowed
    30833253 SYNs to LISTEN sockets dropped
    31091579 packet headers predicted
    19560618 acknowledgments not containing data payload received
    5164955 predicted acknowledgments
    5 times recovered from packet loss by selective acknowledgements
    2 congestion windows recovered without slow start by DSACK
    562 congestion windows recovered without slow start after partial ack
    TCPLostRetransmit: 4
    2 fast retransmits
    14 retransmits in slow start
    11487582 other TCP timeouts
    TCPLossProbes: 111803
    TCPLossProbeRecovery: 31
    491138 packets collapsed in receive queue due to low socket buffer
    1235 DSACKs sent for old packets
    7 DSACKs sent for out of order packets
    107130 DSACKs received
    448950 connections reset due to unexpected data
    6775 connections reset due to early user close
    40 connections aborted due to timeout
    TCPDSACKIgnoredNoUndo: 106166
    TCPSackShiftFallback: 51
    TCPBacklogDrop: 23
    TCPTimeWaitOverflow: 26
    TCPRetransFail: 1
    TCPRcvCoalesce: 7938007
    TCPOFOQueue: 398454
    TCPOFOMerge: 7
    TCPChallengeACK: 2568
    TCPSYNChallenge: 435
    TCPSpuriousRtxHostQueues: 445
    TCPAutoCorking: 2376983
    TCPWantZeroWindowAdv: 3117
    TCPSynRetrans: 9574616
    TCPOrigDataSent: 31330454
    TCPHystartTrainDetect: 74
    TCPHystartTrainCwnd: 1429
    TCPKeepAlive: 4396714
IpExt:
    InOctets: 8801171598456
    OutOctets: 17529868763518
    InNoECTPkts: 30742415236
    InECT0Pkts: 97
root@ip-172-1-57-200:~#
```

存在问题的 NodePod 信息汇总

| 端口号 | Service | Recv-Q |
| ------ | ------- | ---- |
| 30720 | backend/neo-prod-ccapi-inner-lb:http | 129 |
| 31425 | backend/llspay-prod-rpc-lb:grpc" | 129 |
| 30914 | frontend/tmsweb-prod-lb:http | 129 |
| 30563 | backend/cooper-prod-rpc-lb:grpc | 129 |
| 31460 | frontend/eelmsfe-prod:http | 3 |
| 32420 | frontend/lmsmobile-prod-lb:http | 74 |
| 31493 | algorithm/chatbot0acceptor:ws | 129 |
| 31591 | backend/darwin-prod-inner-lb:http | 129 |
| 30217 | backend/neo-prod-payapi-inner-lb:http | 24 |
| 32522 | platform/cwexporter-prod-rds:http | 129 |
| 32426 | backend/neo-prod-web-inner-lb:http | 82 |
| 30381 | frontend/cchybridweb-prod-lb:http | 129 |
| 30701 | platform/dva-server:http | 65 |
| 30605 | backend/anatawa-production:http | 129 |
| 32078 | frontend/tmsweb-prod-lb:https | 93 |
| 32430 | algorithm/lqdebug-prod-elb:http | 129 |
| 32206 | backend/neo-prod-opsapi-lb:http | 129 |
| 31407 | backend/viras-prod-api-lb:http | 129 |
| 30672 | backend/neo-prod-lmsapi:http | 129 |
| 31889 | backend/homepage-prod-api-lb:http | 76 |
| 30899 | algorithm/lqtelisdebug-prod:http | 129 |
| 31123 | backend/ielts:https | 129 |
| 30931 | backend/kensaku-searchweb:http | 38 |
| 30613 | backend/wechatgo-prod:http | 129 |
| 30966 | backend/neo-prod-extra-inner-lb:http | 129 |
| 31095 | algorithm/sherpa-qahub-prod:http | 129 |
| 30328 | backend/neo-prod-ccapi-inner-lb2:http | 3 |
| 31641 | platform/prometheus-service-v2:http | 129 |
| 32634 | backend/ielts:http | 129 |
| 32348 | backend/llspay-prod-outerhttp:tcp | 36 |
| 30140 | frontend/lms-prod-lb:http | 15 |
| 32381 | backend/godsaid-prod-lb:http | 129 |

有问题的 Service 中 Recv-Q 已经溢出的情况

| Namespace | Num of Services | Recv-Q overflow |
| -- | --- | --- |
| backend | 19 | 13 |
| platform | 3 | 2 |
| algorithm | 4 | 4 |
| frontend | 6 | 2 |



## ip-172-1-57-200.cn-north-1.compute.internal

### 基础信息

```
root@ip-172-1-57-200:~# uname -a
Linux ip-172-1-57-200 4.4.41-k8s #1 SMP Mon Jan 9 15:34:39 UTC 2017 x86_64 GNU/Linux
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# lsb_release -rd
Description:   Debian GNU/Linux 8.7 (jessie)
Release:  8.7
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ifconfig
cbr0      Link encap:Ethernet  HWaddr 0a:58:64:61:10:01
          inet addr:100.97.16.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::d0fd:beff:fe1a:a774/64 Scope:Link
          UP BROADCAST RUNNING PROMISC MULTICAST  MTU:9001  Metric:1
          RX packets:9780559777 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8553261617 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2597650166059 (2.3 TiB)  TX bytes:3009045845412 (2.7 TiB)

docker0   Link encap:Ethernet  HWaddr 02:42:9c:a9:30:fe
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr 02:4a:26:b7:e6:84
          inet addr:172.1.57.200  Bcast:172.1.63.255  Mask:255.255.224.0
          inet6 addr: fe80::4a:26ff:feb7:e684/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:14792941937 errors:0 dropped:0 overruns:0 frame:0
          TX packets:15486409042 errors:0 dropped:4 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:4533717705097 (4.1 TiB)  TX bytes:4424381306029 (4.0 TiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:379 errors:0 dropped:0 overruns:0 frame:0
          TX packets:379 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:30256 (29.5 KiB)  TX bytes:30256 (29.5 KiB)

veth2bb611d4 Link encap:Ethernet  HWaddr 6a:3e:17:fe:51:e4
          inet6 addr: fe80::683e:17ff:fefe:51e4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:3590152 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4410908 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:837409545 (798.6 MiB)  TX bytes:4768723023 (4.4 GiB)

veth4e64bccf Link encap:Ethernet  HWaddr 52:e2:be:72:f1:b7
          inet6 addr: fe80::50e2:beff:fe72:f1b7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:61040061 errors:0 dropped:0 overruns:0 frame:0
          TX packets:43008946 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:76011283196 (70.7 GiB)  TX bytes:5198772585 (4.8 GiB)

veth502c0054 Link encap:Ethernet  HWaddr da:6e:be:84:48:2f
          inet6 addr: fe80::d86e:beff:fe84:482f/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:895915 errors:0 dropped:0 overruns:0 frame:0
          TX packets:835630 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:216169347 (206.1 MiB)  TX bytes:222118266 (211.8 MiB)

veth6be2e47b Link encap:Ethernet  HWaddr e6:fc:13:90:37:93
          inet6 addr: fe80::e4fc:13ff:fe90:3793/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:32831100 errors:0 dropped:0 overruns:0 frame:0
          TX packets:61177353 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2436955111 (2.2 GiB)  TX bytes:4190875816 (3.9 GiB)

veth79295df4 Link encap:Ethernet  HWaddr ee:61:dc:8d:6d:54
          inet6 addr: fe80::ec61:dcff:fe8d:6d54/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:5028949 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4275979 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1240930511 (1.1 GiB)  TX bytes:2498879747 (2.3 GiB)

vethdd28d6de Link encap:Ethernet  HWaddr 9e:c7:ea:a6:bb:e8
          inet6 addr: fe80::9cc7:eaff:fea6:bbe8/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:7821254 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6630888 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1911222218 (1.7 GiB)  TX bytes:1448434414 (1.3 GiB)

root@ip-172-1-57-200:~#
```


### 网络信息

#### 整体情况

```
root@ip-172-1-57-200:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSE_WAIT 2996    -- 这个值很长一段时间内不会变（可能有小增长）
TIME_WAIT 13
ESTABLISHED 16
root@ip-172-1-57-200:~#
...
...(N 天后)...
...
root@ip-172-1-57-200:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSE_WAIT 3083
TIME_WAIT 9
ESTABLISHED 19
root@ip-172-1-57-200:~#
```

Recv-Q > 0 的监听端口（短时间很难看到变化，长时间来说会增长，也有个别端口消失的情况）

```
root@ip-172-1-57-200:~# ss -nltp| awk '{ if ($2 > 0) print $0 }'
State      Recv-Q Send-Q        Local Address:Port          Peer Address:Port
LISTEN     129    128                      :::30720                   :::*      users:(("kube-proxy",pid=1448,fd=21))
LISTEN     129    128                      :::31425                   :::*      users:(("kube-proxy",pid=1448,fd=26))
LISTEN     129    128                      :::30914                   :::*      users:(("kube-proxy",pid=1448,fd=54))
LISTEN     129    128                      :::30563                   :::*      users:(("kube-proxy",pid=1448,fd=42))
LISTEN     3      128                      :::31460                   :::*      users:(("kube-proxy",pid=1448,fd=66))
LISTEN     74     128                      :::32420                   :::*      users:(("kube-proxy",pid=1448,fd=17))
LISTEN     129    128                      :::31493                   :::*      users:(("kube-proxy",pid=1448,fd=43))
LISTEN     129    128                      :::31591                   :::*      users:(("kube-proxy",pid=1448,fd=10))
LISTEN     24     128                      :::30217                   :::*      users:(("kube-proxy",pid=1448,fd=20))
LISTEN     129    128                      :::32522                   :::*      users:(("kube-proxy",pid=1448,fd=31))
LISTEN     82     128                      :::32426                   :::*      users:(("kube-proxy",pid=1448,fd=28))
LISTEN     129    128                      :::30381                   :::*      users:(("kube-proxy",pid=1448,fd=56))
LISTEN     65     128                      :::30701                   :::*      users:(("kube-proxy",pid=1448,fd=30))
LISTEN     129    128                      :::30605                   :::*      users:(("kube-proxy",pid=1448,fd=25))
LISTEN     92     128                      :::32078                   :::*      users:(("kube-proxy",pid=1448,fd=55))
LISTEN     129    128                      :::32430                   :::*      users:(("kube-proxy",pid=1448,fd=32))
LISTEN     129    128                      :::32206                   :::*      users:(("kube-proxy",pid=1448,fd=27))
LISTEN     129    128                      :::31407                   :::*      users:(("kube-proxy",pid=1448,fd=68))
LISTEN     129    128                      :::30672                   :::*      users:(("kube-proxy",pid=1448,fd=47))
LISTEN     15     128                      :::30225                   :::*      users:(("kube-proxy",pid=1448,fd=57))
LISTEN     76     128                      :::31889                   :::*      users:(("kube-proxy",pid=1448,fd=19))
LISTEN     129    128                      :::30899                   :::*      users:(("kube-proxy",pid=1448,fd=61))
LISTEN     122    128                      :::31123                   :::*      users:(("kube-proxy",pid=1448,fd=22))
LISTEN     38     128                      :::30931                   :::*      users:(("kube-proxy",pid=1448,fd=18))
LISTEN     129    128                      :::30613                   :::*      users:(("kube-proxy",pid=1448,fd=51))
LISTEN     129    128                      :::30966                   :::*      users:(("kube-proxy",pid=1448,fd=16))
LISTEN     129    128                      :::31095                   :::*      users:(("kube-proxy",pid=1448,fd=60))
LISTEN     3      128                      :::30328                   :::*      users:(("kube-proxy",pid=1448,fd=11))
LISTEN     129    128                      :::31641                   :::*      users:(("kube-proxy",pid=1448,fd=58))
LISTEN     50     128                      :::32634                   :::*      users:(("kube-proxy",pid=1448,fd=12))
LISTEN     36     128                      :::32348                   :::*      users:(("kube-proxy",pid=1448,fd=38))
LISTEN     15     128                      :::30140                   :::*      users:(("kube-proxy",pid=1448,fd=23))
LISTEN     129    128                      :::32381                   :::*      users:(("kube-proxy",pid=1448,fd=48))
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -nltp| awk '{ if ($2 > 0) print $0 }' | wc -l
34
root@ip-172-1-57-200:~#
```

可以看到：**所有监听端口异常的情况都出现 kube-proxy 上**，共计 33 个；

```
root@ip-172-1-57-200:~# ss -nltp| awk '{print $0}' | grep "kube-proxy"
LISTEN     0      128               127.0.0.1:10249                    *:*      users:(("kube-proxy",pid=1448,fd=6))
LISTEN     0      128                      :::31806                   :::*      users:(("kube-proxy",pid=1448,fd=72))
LISTEN     0      128                      :::32543                   :::*      users:(("kube-proxy",pid=1448,fd=73))
LISTEN     0      128                      :::32703                   :::*      users:(("kube-proxy",pid=1448,fd=46))
LISTEN     129    128                      :::30720                   :::*      users:(("kube-proxy",pid=1448,fd=21))
LISTEN     129    128                      :::31425                   :::*      users:(("kube-proxy",pid=1448,fd=26))
LISTEN     129    128                      :::30914                   :::*      users:(("kube-proxy",pid=1448,fd=54))
LISTEN     0      128                      :::31555                   :::*      users:(("kube-proxy",pid=1448,fd=33))
LISTEN     129    128                      :::30563                   :::*      users:(("kube-proxy",pid=1448,fd=42))
LISTEN     3      128                      :::31460                   :::*      users:(("kube-proxy",pid=1448,fd=66))
LISTEN     0      128                      :::32068                   :::*      users:(("kube-proxy",pid=1448,fd=65))
LISTEN     74     128                      :::32420                   :::*      users:(("kube-proxy",pid=1448,fd=17))
LISTEN     129    128                      :::31493                   :::*      users:(("kube-proxy",pid=1448,fd=43))
LISTEN     0      128                      :::31813                   :::*      users:(("kube-proxy",pid=1448,fd=13))
LISTEN     129    128                      :::31591                   :::*      users:(("kube-proxy",pid=1448,fd=10))
LISTEN     0      128                      :::30184                   :::*      users:(("kube-proxy",pid=1448,fd=44))
LISTEN     0      128                      :::31177                   :::*      users:(("kube-proxy",pid=1448,fd=34))
LISTEN     0      128                      :::31753                   :::*      users:(("kube-proxy",pid=1448,fd=14))
LISTEN     24     128                      :::30217                   :::*      users:(("kube-proxy",pid=1448,fd=20))
LISTEN     0      128                      :::32746                   :::*      users:(("kube-proxy",pid=1448,fd=70))
LISTEN     129    128                      :::32522                   :::*      users:(("kube-proxy",pid=1448,fd=31))
LISTEN     82     128                      :::32426                   :::*      users:(("kube-proxy",pid=1448,fd=28))
LISTEN     0      128                      :::30891                   :::*      users:(("kube-proxy",pid=1448,fd=69))
LISTEN     0      128                      :::31691                   :::*      users:(("kube-proxy",pid=1448,fd=63))
LISTEN     0      128                      :::31020                   :::*      users:(("kube-proxy",pid=1448,fd=36))
LISTEN     129    128                      :::30381                   :::*      users:(("kube-proxy",pid=1448,fd=56))
LISTEN     65     128                      :::30701                   :::*      users:(("kube-proxy",pid=1448,fd=30))
LISTEN     129    128                      :::30605                   :::*      users:(("kube-proxy",pid=1448,fd=25))
LISTEN     0      128                      :::30702                   :::*      users:(("kube-proxy",pid=1448,fd=15))
LISTEN     92     128                      :::32078                   :::*      users:(("kube-proxy",pid=1448,fd=55))
LISTEN     129    128                      :::32430                   :::*      users:(("kube-proxy",pid=1448,fd=32))
LISTEN     129    128                      :::32206                   :::*      users:(("kube-proxy",pid=1448,fd=27))
LISTEN     129    128                      :::31407                   :::*      users:(("kube-proxy",pid=1448,fd=68))
LISTEN     129    128                      :::30672                   :::*      users:(("kube-proxy",pid=1448,fd=47))
LISTEN     0      128                      :::32752                   :::*      users:(("kube-proxy",pid=1448,fd=45))
LISTEN     0      128                      :::32720                   :::*      users:(("kube-proxy",pid=1448,fd=37))
LISTEN     15     128                      :::30225                   :::*      users:(("kube-proxy",pid=1448,fd=57))
LISTEN     76     128                      :::31889                   :::*      users:(("kube-proxy",pid=1448,fd=19))
LISTEN     0      128                      :::31122                   :::*      users:(("kube-proxy",pid=1448,fd=59))
LISTEN     129    128                      :::30899                   :::*      users:(("kube-proxy",pid=1448,fd=61))
LISTEN     0      128                      :::30579                   :::*      users:(("kube-proxy",pid=1448,fd=50))
LISTEN     122    128                      :::31123                   :::*      users:(("kube-proxy",pid=1448,fd=22))
LISTEN     38     128                      :::30931                   :::*      users:(("kube-proxy",pid=1448,fd=18))
LISTEN     0      128                      :::31380                   :::*      users:(("kube-proxy",pid=1448,fd=71))
LISTEN     0      128                      :::30196                   :::*      users:(("kube-proxy",pid=1448,fd=29))
LISTEN     129    128                      :::30613                   :::*      users:(("kube-proxy",pid=1448,fd=51))
LISTEN     0      128                      :::31926                   :::*      users:(("kube-proxy",pid=1448,fd=62))
LISTEN     129    128                      :::30966                   :::*      users:(("kube-proxy",pid=1448,fd=16))
LISTEN     0      128                      :::30071                   :::*      users:(("kube-proxy",pid=1448,fd=64))
LISTEN     129    128                      :::31095                   :::*      users:(("kube-proxy",pid=1448,fd=60))
LISTEN     3      128                      :::30328                   :::*      users:(("kube-proxy",pid=1448,fd=11))
LISTEN     0      128                      :::31225                   :::*      users:(("kube-proxy",pid=1448,fd=74))
LISTEN     129    128                      :::31641                   :::*      users:(("kube-proxy",pid=1448,fd=58))
LISTEN     0      128                      :::30554                   :::*      users:(("kube-proxy",pid=1448,fd=24))
LISTEN     50     128                      :::32634                   :::*      users:(("kube-proxy",pid=1448,fd=12))
LISTEN     36     128                      :::32348                   :::*      users:(("kube-proxy",pid=1448,fd=38))
LISTEN     15     128                      :::30140                   :::*      users:(("kube-proxy",pid=1448,fd=23))
LISTEN     129    128                      :::32381                   :::*      users:(("kube-proxy",pid=1448,fd=48))
root@ip-172-1-57-200:~# ss -nltp| awk '{print $0}' | grep "kube-proxy" | wc -l
58
root@ip-172-1-57-200:~#
```

可以看到：kube-proxy 共计监听 57 个端口；

> 问题：kube-proxy 监听端口的数量由什么因素决定？那就是……哈哈哈

系统当前配置的一些 tcp 内核参数

```
fs.file-max = 762200
net.core.somaxconn = 128
net.ipv4.tcp_max_syn_backlog = 256
net.ipv4.tcp_abort_on_overflow = 0
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 0
net.ipv4.tcp_tw_recycle = 0
```

#### 30720 端口情况

```
root@ip-172-1-57-200:~# ss -antp| grep "30720"
LISTEN     129    128                      :::30720                   :::*      users:(("kube-proxy",pid=1448,fd=21))
CLOSE-WAIT 746    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39745
CLOSE-WAIT 775    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40103
CLOSE-WAIT 2963   0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:52693
CLOSE-WAIT 705    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:54134
CLOSE-WAIT 748    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51454
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:30720   ::ffff:172.1.51.65:37411
CLOSE-WAIT 457    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38536
CLOSE-WAIT 439    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:52216
CLOSE-WAIT 762    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39027
CLOSE-WAIT 762    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:36883
CLOSE-WAIT 666    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40230
CLOSE-WAIT 432    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37841
CLOSE-WAIT 1057   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38735
CLOSE-WAIT 746    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38574
CLOSE-WAIT 774    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38462
CLOSE-WAIT 798    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51773
CLOSE-WAIT 1041   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39567
CLOSE-WAIT 1003   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39883
CLOSE-WAIT 1064   0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:52484
CLOSE-WAIT 711    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39891
CLOSE-WAIT 758    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38287
CLOSE-WAIT 438    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40246
CLOSE-WAIT 449    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38421
CLOSE-WAIT 784    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40494
CLOSE-WAIT 846    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38877
CLOSE-WAIT 796    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40128
CLOSE-WAIT 756    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:52131
CLOSE-WAIT 1062   0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53177
CLOSE-WAIT 755    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39872
CLOSE-WAIT 760    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:52806
CLOSE-WAIT 748    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39719
CLOSE-WAIT 528    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53622
CLOSE-WAIT 525    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38116
CLOSE-WAIT 486    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39725
CLOSE-WAIT 1011   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39228
CLOSE-WAIT 764    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40506
CLOSE-WAIT 1062   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38980
CLOSE-WAIT 489    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40371
CLOSE-WAIT 1031   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39760
CLOSE-WAIT 774    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38363
CLOSE-WAIT 53818  0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40164
CLOSE-WAIT 468    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38723
CLOSE-WAIT 103876 0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37054
CLOSE-WAIT 778    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40658
CLOSE-WAIT 477    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:54051
CLOSE-WAIT 781    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39520
CLOSE-WAIT 32480  0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38229
CLOSE-WAIT 1016   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40514
CLOSE-WAIT 86310  0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40740
CLOSE-WAIT 1108   0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53734
CLOSE-WAIT 1014   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40454
CLOSE-WAIT 795    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39062
CLOSE-WAIT 801    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39973
CLOSE-WAIT 804    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51726
CLOSE-WAIT 64618  0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39398
CLOSE-WAIT 780    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40357
CLOSE-WAIT 1042   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37085
CLOSE-WAIT 798    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39831
CLOSE-WAIT 765    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38665
CLOSE-WAIT 810    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39321
CLOSE-WAIT 763    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51075
CLOSE-WAIT 770    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40066
CLOSE-WAIT 828    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40567
CLOSE-WAIT 458    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:50843
CLOSE-WAIT 763    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40698
CLOSE-WAIT 760    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39785
CLOSE-WAIT 1015   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37195
CLOSE-WAIT 794    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53200
CLOSE-WAIT 473    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51711
CLOSE-WAIT 763    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40772
CLOSE-WAIT 755    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53464
CLOSE-WAIT 1024   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40385
CLOSE-WAIT 479    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39981
CLOSE-WAIT 761    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:51265
CLOSE-WAIT 785    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40729
CLOSE-WAIT 1013   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37470
CLOSE-WAIT 843    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39766
CLOSE-WAIT 735    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38971
CLOSE-WAIT 770    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53816
CLOSE-WAIT 521    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:50697
CLOSE-WAIT 756    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39830
CLOSE-WAIT 1022   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37659
CLOSE-WAIT 456    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40018
CLOSE-WAIT 1021   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40095
CLOSE-WAIT 1244   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39673
CLOSE-WAIT 1064   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39888
CLOSE-WAIT 480    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40161
CLOSE-WAIT 761    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37702
CLOSE-WAIT 31372  0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40186
CLOSE-WAIT 718    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53759
CLOSE-WAIT 778    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:53105
CLOSE-WAIT 756    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:40787
CLOSE-WAIT 443    0       ::ffff:172.1.57.200:30720 ::ffff:172.31.12.249:50448
CLOSE-WAIT 766    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:38169
CLOSE-WAIT 777    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39829
CLOSE-WAIT 1062   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37253
CLOSE-WAIT 1019   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37624
CLOSE-WAIT 1014   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39186
CLOSE-WAIT 847    0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:37119
CLOSE-WAIT 1079   0       ::ffff:172.1.57.200:30720  ::ffff:172.31.3.170:39442
root@ip-172-1-57-200:~#
```

输出解析：

- 监听端口的 accept queue 已满（129）；
- 存在 100 个 CLOSE-WAIT 状态的连接；
- "172.31.3.170" 存在 75 个（该地址属于 tengine A）；
- "172.31.12.249" 存在 24 个（该地址属于 tengine B）；
- "172.1.51.65" 存在 1 个（该地址查不到）；
- 几乎全部 CLOSE-WAIT 状态的连接中都残留大量数据（Recv-Q）；
- 发现和 tengine 相关的 Recv-Q 中均留有很多字节未被获取（**之前一直觉得 Recv-Q 中有数据堆积是 kube-proxy 的问题，现在觉得应该是后端服务不正常导致的问题，因为 kube-proxy 本身只负责转发而已**）；

![生产环境 tengine 信息](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%20tengine%20%E4%BF%A1%E6%81%AF.png)

由此看来，大部分 CLOSE-WAIT 状态发生似乎和 tengine 有所关联（应该只说明了上下游关系，并非因果关系）；

**之前认为，当 30720 监听端口 accept queue 满（Recv-Q 129）后，该端口将无法正常提供服务，结果发现，这个结论是错误的！！**


实际抓包发现：

- 11.2s 抓取了 6,575 个 packets ；

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%201.png)

- 其中包含了 587 条 tcp 流（HTTP）；

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%202.png)


- 平均一次 HTTP request-response 在 20ms~30ms 左右（大约 50 qps）；

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%203.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%204.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%205.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%206.png)

- 仅出现一处包异常处理

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/30720%20%E5%88%86%E6%9E%90%20-%207.png)

当前结论：

- **30720 端口在 Recv-Q 满的状态下，仍旧能够提供服务（但实际上，此时可能会有重连、连接不上、概率性成功等问题）**；
- **accept queue 中有堆积并不会影响后续连接的处理（即没有所谓的先后顺序）**；


查看和 30720 相关的 iptables 规则

```
root@ip-172-1-57-200:~# iptables -t nat -S
...
-N KUBE-MARK-MASQ
-N KUBE-NODEPORTS
...
-N KUBE-SEP-HOTTGIVHNU7PF2IZ
(略)
...
-N KUBE-SERVICES
...
-N KUBE-SVC-RK4J2BHJ4QTMGP2E
...
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
...
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp --dport 30720 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp --dport 30720 -j KUBE-SVC-RK4J2BHJ4QTMGP2E
...
-A KUBE-SERVICES -d 100.71.52.243/32 -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-RK4J2BHJ4QTMGP2E
...
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
...
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02499999991 -j KUBE-SEP-HOTTGIVHNU7PF2IZ
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02564000012 -j KUBE-SEP-KAMGFW7GBX2ZEWNG
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02632000018 -j KUBE-SEP-KWJZCBMS26B6TN3E
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02703000000 -j KUBE-SEP-2HGWRPDMJ5VN7IYG
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02778000012 -j KUBE-SEP-AQD544UMQK7YXHWI
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02857000008 -j KUBE-SEP-2V2SKOGLRWBDEIK3
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.02940999996 -j KUBE-SEP-HDGZH2K2Q5KYK6NI
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03030000022 -j KUBE-SEP-B5QS2JVDGXLX3GNU
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03125000000 -j KUBE-SEP-VKO3XG5XDMHS5Q6Z
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03225999977 -j KUBE-SEP-CF26G2Q3UU23YVC3
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03333000001 -j KUBE-SEP-H5XF2J7YYQIWDT72
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03447999991 -j KUBE-SEP-F3MK5ZZ5HMOEKOUT
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03570999997 -j KUBE-SEP-6TVTHK3L2D7XQMQ2
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03703999985 -j KUBE-SEP-CCG65K5G3FJWZAIZ
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.03845999995 -j KUBE-SEP-L6CQAJ4PHCURPTLX
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04000000004 -j KUBE-SEP-OWYSYJ7JDBK4GIW4
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04167000018 -j KUBE-SEP-Q3IE6XRCWUSLECHF
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04347999999 -j KUBE-SEP-WSVC4Q27PD4HF72A
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04545000009 -j KUBE-SEP-X6XIVDRI35SZ7FXK
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04761999985 -j KUBE-SEP-NQLPKTYR4TUJ34B2
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.04999999981 -j KUBE-SEP-Q3E2IYV5SW2ZEJO5
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.05262999982 -j KUBE-SEP-7XMEMF5BY5SUWLZP
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.05555999978 -j KUBE-SEP-HMUJYHHRZF6KGFKY
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.05881999992 -j KUBE-SEP-IBCIOSRMSWR2CVEQ
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.06250000000 -j KUBE-SEP-AFPA5JSWVO3IWAN2
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.06667000009 -j KUBE-SEP-XA43BKIZOXWBXF4A
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.07143000001 -j KUBE-SEP-F3IBY5CTRFUTYRT3
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.07691999990 -j KUBE-SEP-4BM56RNS7OXJS2K5
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.08332999982 -j KUBE-SEP-MR5XDDJACS7SBLHK
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.09090999980 -j KUBE-SEP-MTPUVDW4XJC46CS4
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.10000000009 -j KUBE-SEP-AOTTVU6HRBQ7PGIS
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.11110999994 -j KUBE-SEP-DQFHR5WZ3PZVZICM
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.12500000000 -j KUBE-SEP-WZ4YDT64BIIEFLGB
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.14286000002 -j KUBE-SEP-NUUEOMKFYDG5OYRX
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.16667000018 -j KUBE-SEP-DOW7HFKXQXNVEDWN
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.20000000019 -j KUBE-SEP-PZZTKAZNCFYU3EGH
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.25000000000 -j KUBE-SEP-Q6D6RVOSSK5HWTTY
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-YP6N75D43VPTTURV
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-2ZXTAKNSHGAEW3P2
-A KUBE-SVC-RK4J2BHJ4QTMGP2E -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -j KUBE-SEP-HJVPNYJ43247FIZT
...
-A KUBE-SEP-HOTTGIVHNU7PF2IZ -s 100.96.179.196/32 -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-HOTTGIVHNU7PF2IZ -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp -j DNAT --to-destination 100.96.179.196:8080
...
-A KUBE-SEP-KAMGFW7GBX2ZEWNG -s 100.97.100.231/32 -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-KAMGFW7GBX2ZEWNG -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp -j DNAT --to-destination 100.97.100.231:8080
...
```

反过来查看对应的 service 信息，可以看出和上述 iptables 规则的关系；

> 注意：Endpoints 中的内容（地址信息+数量）是动态变化的

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe svc neo-prod-ccapi-inner-lb
Name:               neo-prod-ccapi-inner-lb
Namespace:          backend
Labels:             <none>
Selector:      load-balancer-neo-prod-ccapi-inner-lb=true
Type:               NodePort
IP:            100.71.52.243
Port:               http 80/TCP
NodePort:      http 30720/TCP
Endpoints:          100.96.179.196:8080,100.97.100.231:8080,100.97.101.244:8080 + 37 more...
Session Affinity:   None
No events.
[deployer@k8s-deploy-prod0]$
```

发现还存在另外一个 service 也同样指向相同的 endpoints ；

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe svc neo-prod-ccapi-inner-lb2
Name:               neo-prod-ccapi-inner-lb2
Namespace:          backend
Labels:             <none>
Selector:      load-balancer-neo-prod-ccapi-inner-lb2=true
Type:               NodePort
IP:            100.68.156.255
Port:               http 80/TCP
NodePort:      http 30328/TCP
Endpoints:          100.96.179.196:8080,100.97.100.231:8080,100.97.101.244:8080 + 37 more...
Session Affinity:   None
No events.
[deployer@k8s-deploy-prod0]$
```

结论增加：

- **kube-proxy 不会为 Service port 建立监听，只会为其添加 iptables 转发规则**；
- **kube-proxy 会为 Node port 建立监听，同时会为其添加 iptables 转发规则**；
- 通过命令 `ss -nltp| awk '{ if ($2 > 0) print $0 }'` 看到的所有 Recv-Q > 0 的监听端口，经确认，均为 kube-proxy 创建的 NodePort 监听端口（Service type 为 `NodePort` 或者 `LoadBalancer`）


#### 31425 端口情况

```
root@ip-172-1-57-200:~# ss -antp| grep "31425"
LISTEN     129    128                      :::31425                   :::*      users:(("kube-proxy",pid=1448,fd=26))
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:2128
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:44086
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:51400
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48150
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48756
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:51818
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:50768
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54917
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:44492
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:39172
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:38966
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:63405
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:9010
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:61701
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:9212
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:9412
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:57005
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54261
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:59443
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:52226
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:62989
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:41830
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:52634
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47428
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:40048
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:3996
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:46770
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47636
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54045
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:2546
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:62359
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:39802
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:24438
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:24026
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:46150
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:61493
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:3378
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:59035
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:49813
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:2958
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:49603
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:58417
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:49316
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47848
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:58623
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:46562
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48352
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48684
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48888
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:58829
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:39600
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:64445
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:45322
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47008
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48060
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:40720
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48474
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:1096
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:55553
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47552
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:3584
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:53841
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:49940
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:52432
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:60275
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:60065
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:3166
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:39388
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:10014
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:45732
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:8806
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:59653
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:24234
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:64025
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:49150
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:50017
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:59237
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47946
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:62149
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:61945
CLOSE-WAIT 620    0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:39364
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:56385
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:50566
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47750
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:59857
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48264
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:9814
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:50356
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:43664
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:49106
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:50531
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:63615
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:50984
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:24968
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:50227
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:52844
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54707
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:1508
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:62573
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54505
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:1716
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:57209
CLOSE-WAIT 521    0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:54029
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48954
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:62777
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:42652
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:8610
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:43876
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:51200
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:44700
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:64235
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:52018
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:8408
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:42450
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:9612
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:48548
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:1920
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:44908
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:61291
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:47224
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:60477
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:46354
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:2342
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:45526
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:3788
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:63201
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.207:63823
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:44292
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:31425  ::ffff:172.1.41.234:45110
root@ip-172-1-57-200:~#
```

可以发现存在 129 个 CLOSE-WAIT 状态

- "172.1.41.207" 存在 65 个（该地址查不到）；
- "172.1.41.234" 存在 64 个（该地址查不到）；
- 几乎所有连接的 Recv-Q 上只有 1 字节残留（仅两个例外）；


抓包发现：

- "172.1.41.207" 和 "172.1.41.234" 每隔 20s 向 `172.1.57.200:31425` 建立一次连接，然后在主动断开（FIN）连接；在链接建立后，`172.1.57.200:31425` 会主动发送 63 字节 tcp 数据，显示为乱码；
- 经确认，该连接上使用的是 grpc 协议，而非 HTTP 协议；据说为心跳协议实现方式；

```
Frame 4: 131 bytes on wire (1048 bits), 131 bytes captured (1048 bits)
Linux cooked capture
Internet Protocol Version 4, Src: 172.1.57.200, Dst: 172.1.41.207
Transmission Control Protocol, Src Port: 31425, Dst Port: 33581, Seq: 1, Ack: 1, Len: 63
Data (63 bytes)
    Data: 000018040000000000000400400000000500400000000600...
    [Length: 63]

--- 具体为

00:00:18:04:00:00:00:00:00:00:04:00:40:00:00:00:05:00:40:00:00:00:06:00:00:20:00:fe:03:00:00:00:01:00:00:04:08:00:00:00:00:00:00:3f:00:01:00:00:08:06:00:00:00:00:00:00:00:00:00:00:00:00:00
```

- 在 conntrack 中也能发现证据："172.1.41.234" 和 "172.1.41.207" 的连接各存在 6 个，刚好对应 20s 一次建链和 120s 状态超时时间；

```
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack|grep "31425"
tcp      6 66 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=14532 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=14532 [ASSURED] mark=0 use=2
tcp      6 46 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=14330 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=14330 [ASSURED] mark=0 use=2
tcp      6 25 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=46031 dport=31425 src=100.97.74.176 dst=172.1.57.200 sport=50066 dport=46031 [ASSURED] mark=0 use=2
tcp      6 5 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=13936 dport=31425 src=100.97.91.90 dst=172.1.57.200 sport=50066 dport=13936 [ASSURED] mark=0 use=2
tcp      6 45 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=46229 dport=31425 src=100.97.116.23 dst=172.1.57.200 sport=50066 dport=46229 [ASSURED] mark=0 use=2
tcp      6 105 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=14932 dport=31425 src=100.97.89.202 dst=172.1.57.200 sport=50066 dport=14932 [ASSURED] mark=0 use=2
tcp      6 85 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=14738 dport=31425 src=100.97.89.202 dst=172.1.57.200 sport=50066 dport=14738 [ASSURED] mark=0 use=2
tcp      6 65 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=46429 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=46429 [ASSURED] mark=0 use=2
tcp      6 85 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=46633 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=46633 [ASSURED] mark=0 use=2
tcp      6 105 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=46833 dport=31425 src=100.97.116.23 dst=172.1.57.200 sport=50066 dport=46833 [ASSURED] mark=0 use=2
tcp      6 25 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=14136 dport=31425 src=100.97.89.202 dst=172.1.57.200 sport=50066 dport=14136 [ASSURED] mark=0 use=2
tcp      6 5 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=45833 dport=31425 src=100.97.91.90 dst=172.1.57.200 sport=50066 dport=45833 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~#
```

- 通过 `curl -v -i 172.1.57.200:31425` 测试也能返回乱码信息（因为其不是 HTTP 协议，而是 grpc 协议）；

```
➜  ~ curl -v -i 172.1.57.200:31425
* Rebuilt URL to: 172.1.57.200:31425/
*   Trying 172.1.57.200...
* TCP_NODELAY set
* Connected to 172.1.57.200 (172.1.57.200) port 31425 (#0)
> GET / HTTP/1.1
> Host: 172.1.57.200:31425
> User-Agent: curl/7.54.0
> Accept: */*
>
* Connection #0 to host 172.1.57.200 left intact
@@ %                                                                                                                                   ➜  ~
```

- 确认 31425 NodePort 对应 Service 下的所有 endpoints ；

```
root@ip-172-1-57-200:~# iptables -t nat -S |grep "31425"
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/llspay-prod-rpc-lb:grpc" -m tcp --dport 31425 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/llspay-prod-rpc-lb:grpc" -m tcp --dport 31425 -j KUBE-SVC-J5WKZTVXLKWSTUZM
root@ip-172-1-57-200:~#
```

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe svc llspay-prod-rpc-lb
Name:               llspay-prod-rpc-lb
Namespace:          backend
Labels:             <none>
Selector:      load-balancer-llspay-prod-rpc-lb=true
Type:               LoadBalancer
IP:            100.70.205.110
LoadBalancer Ingress:    internal-aaa4700043ec711e7b48102ab8baa063-288575930.cn-north-1.elb.amazonaws.com.cn
Port:               grpc 50066/TCP
NodePort:      grpc 31425/TCP
Endpoints:          100.97.108.84:50066,100.97.116.23:50066,100.97.123.135:50066 + 3 more...
Session Affinity:   None
No events.
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get ep llspay-prod-rpc-lb
NAME                 ENDPOINTS                                                  AGE
llspay-prod-rpc-lb   100.97.108.84:50066,100.97.116.23:50066,100.97.123.135:50066 + 3 more...   282d
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe ep llspay-prod-rpc-lb
Name:          llspay-prod-rpc-lb
Namespace:     backend
Labels:        <none>
Subsets:
  Addresses:        100.97.108.84,100.97.116.23,100.97.123.135,100.97.74.176,100.97.89.202,100.97.91.90
  NotReadyAddresses:     <none>
  Ports:
    Name  Port Protocol
    ----  ---- --------
    grpc  50066     TCP

No events.
[deployer@k8s-deploy-prod0]$


[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.108.84"
llspay-prod-rpc-v054-l8m9g                                  1/1       Running            0          27d       100.97.108.84    ip-172-1-53-68.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.116.23"
llspay-prod-rpc-v054-dv9dt                                  1/1       Running            0          19d       100.97.116.23    ip-172-1-39-126.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.123.135"
llspay-prod-rpc-v054-6ppa3                                  1/1       Running            1          19d       100.97.123.135   ip-172-1-44-36.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.74.176"
llspay-prod-rpc-v054-gokfv                                  1/1       Running            1          19d       100.97.74.176    ip-172-1-46-244.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.89.202"
llspay-prod-rpc-v054-6zisw                                  1/1       Running            1          27d       100.97.89.202    ip-172-1-50-19.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all|grep "100.97.91.90"
llspay-prod-rpc-v054-d9ugi                                  1/1       Running            0          27d       100.97.91.90     ip-172-1-59-67.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
```

- 针对所有 endpoints 测试访问情况

```
admin@ip-172-1-57-200:~$ nc 100.97.108.84 50066
@@ ?
admin@ip-172-1-57-200:~$
admin@ip-172-1-57-200:~$ nc 100.97.116.23 50066
@@ ?
admin@ip-172-1-57-200:~$
admin@ip-172-1-57-200:~$ nc 100.97.123.135 50066
@@ ?
admin@ip-172-1-57-200:~$
admin@ip-172-1-57-200:~$ nc 100.97.74.176 50066
@@ ?
admin@ip-172-1-57-200:~$
admin@ip-172-1-57-200:~$ nc 100.97.89.202 50066
@@ ?
admin@ip-172-1-57-200:~$
admin@ip-172-1-57-200:~$ nc 100.97.91.90 50066
@@ ?
admin@ip-172-1-57-200:~$
```

- 针对 Service VIP 的连接测试

```
admin@ip-172-1-57-200:~$ nc 100.70.205.110 50066
@@ ?
admin@ip-172-1-57-200:~$
```

当前结论：

- 当 31425 监听端口 accept queue 满（Recv-Q 129 且全部为 CLOSE_WAIT）后，仍然能够建立 TCP 连接；
- 抓包发现 31425 监听端口上没有什么流量，只有心跳包；
- 6 个 endpoints 运行应该是正常的（否则业务将会出问题才对）


#### 32426 端口情况

```
root@ip-172-1-57-200:~# ss -antp| grep "32426"
LISTEN     82     128                      :::32426                   :::*      users:(("kube-proxy",pid=1448,fd=28))
CLOSE-WAIT 452    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:40884
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:43685
CLOSE-WAIT 724    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:56297
CLOSE-WAIT 465    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:59886
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:22735
CLOSE-WAIT 457    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:48514
CLOSE-WAIT 466    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:33254
CLOSE-WAIT 204    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:44834
CLOSE-WAIT 631    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:37692
CLOSE-WAIT 466    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:38359
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:22381
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:60670
CLOSE-WAIT 468    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:47756
CLOSE-WAIT 466    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:40612
CLOSE-WAIT 850    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:53107
CLOSE-WAIT 204    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:43463
CLOSE-WAIT 513    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:43597
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:60252
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:46911
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:55482
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:55360
CLOSE-WAIT 764    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:52888
CLOSE-WAIT 497    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:55630
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:28990
CLOSE-WAIT 731    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:43944
CLOSE-WAIT 815    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:56865
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:61236
CLOSE-WAIT 447    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:39979
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:29254
CLOSE-WAIT 488    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:41867
CLOSE-WAIT 720    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:41535
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:59952
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:60850
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:21965
CLOSE-WAIT 616    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:38452
CLOSE-WAIT 495    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:41922
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:55038
CLOSE-WAIT 765    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:39287
CLOSE-WAIT 450    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:55759
CLOSE-WAIT 471    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:33760
CLOSE-WAIT 475    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:42547
CLOSE-WAIT 808    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:56843
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:28906
CLOSE-WAIT 477    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:39491
CLOSE-WAIT 510    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:57318
CLOSE-WAIT 468    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:38031
CLOSE-WAIT 457    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:33297
CLOSE-WAIT 475    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:43051
CLOSE-WAIT 873    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:57716
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:60154
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:22555
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:22927
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:32807
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:61044
CLOSE-WAIT 697    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:43317
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:23121
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:61354
CLOSE-WAIT 477    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:41703
CLOSE-WAIT 793    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:42342
CLOSE-WAIT 473    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:38558
CLOSE-WAIT 593    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:59228
CLOSE-WAIT 782    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:45856
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:22221
CLOSE-WAIT 449    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:60935
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:57959
CLOSE-WAIT 478    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:48007
CLOSE-WAIT 447    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:32974
CLOSE-WAIT 754    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:41387
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426  ::ffff:172.1.44.188:55420
CLOSE-WAIT 719    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:58708
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:28930
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:29078
CLOSE-WAIT 706    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:60454
CLOSE-WAIT 727    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:46477
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:33210
CLOSE-WAIT 205    0       ::ffff:172.1.57.200:32426 ::ffff:172.31.12.249:59108
CLOSE-WAIT 465    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:40484
CLOSE-WAIT 777    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.6.140:37803
CLOSE-WAIT 1      0       ::ffff:172.1.57.200:32426    ::ffff:172.1.62.3:29170
CLOSE-WAIT 779    0       ::ffff:172.1.57.200:32426  ::ffff:172.31.3.170:53209
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -antp| grep "32426" | wc -l
81
root@ip-172-1-57-200:~#
```

可以发现存在 80 个 CLOSE-WAIT 状态

- "172.31.3.170" 存在 17 个（该地址属于 tengine A）；
- "172.31.12.249" 存在 20 个（该地址属于 tengine B）；
- "172.1.62.3" 存在 14 个（该地址查不到）；
- "172.1.44.188" 存在 11 个（该地址查不到）；
- "172.31.6.140" 存在 18 个（该地址属于 tengine C）；




和 tengine 相关的 CLOSE-WAIT 数量为 57 + 83 + 103 = 243 远远低于总量 2996

```
root@ip-172-1-57-200:~# ss -antp | grep "CLOSE-WAIT" | wc -l
2996
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -antp | grep "CLOSE-WAIT" | grep "172.31.6.140" | wc -l
57
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -antp | grep "CLOSE-WAIT" | grep "172.31.12.249" | wc -l
83
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -antp | grep "CLOSE-WAIT" | grep "172.31.3.170" | wc -l
103
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~#
```

