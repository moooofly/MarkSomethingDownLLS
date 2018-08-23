# 流利说线上 k8s pod 问题排查

## neo-prod-cc-api-v202-rfxh9

小结：

- 观察到的 CLOSE_WAIT 状态的连接会发生变化，由此可以认为该状态的出现不是因为业务卡死，而是业务自身处理“慢”的缘故（“慢”可能是业务的正常逻辑，也可能是不正常逻辑）；
- 所有 CLOSE-WAIT 状态的远端均为 19000 端口（除了一个 443 端口的情况），对应到两台 codis 服务上；CLOSE-WAIT 状态位于本地，说明 TCP 连接是 codis 主动进行关闭的（是否合理），而本地业务关闭 socket 的行为很“缓慢”；
- neo 在 4 cpu 上运行 28 线程应对 LISTEN 2/CLOSE-WAIT 36/ESTAB 131/TIME-WAIT 1786 是否合理（理论上讲应该是没问题的）；



## 排查路径

- 在 ip-172-1-57-200 节点上查看监听端口异常情况

```
root@ip-172-1-57-200:~# iptables -t nat -S|grep "^C
root@ip-172-1-57-200:~# ss -nltp | awk '{if ($2 > 0) print $0}'
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
LISTEN     93     128                      :::32078                   :::*      users:(("kube-proxy",pid=1448,fd=55))
LISTEN     129    128                      :::32430                   :::*      users:(("kube-proxy",pid=1448,fd=32))
LISTEN     129    128                      :::32206                   :::*      users:(("kube-proxy",pid=1448,fd=27))
LISTEN     129    128                      :::31407                   :::*      users:(("kube-proxy",pid=1448,fd=68))
LISTEN     129    128                      :::30672                   :::*      users:(("kube-proxy",pid=1448,fd=47))
LISTEN     15     128                      :::30225                   :::*      users:(("kube-proxy",pid=1448,fd=57))
LISTEN     76     128                      :::31889                   :::*      users:(("kube-proxy",pid=1448,fd=19))
LISTEN     129    128                      :::30899                   :::*      users:(("kube-proxy",pid=1448,fd=61))
LISTEN     129    128                      :::31123                   :::*      users:(("kube-proxy",pid=1448,fd=22))
LISTEN     38     128                      :::30931                   :::*      users:(("kube-proxy",pid=1448,fd=18))
LISTEN     129    128                      :::30613                   :::*      users:(("kube-proxy",pid=1448,fd=51))
LISTEN     129    128                      :::30966                   :::*      users:(("kube-proxy",pid=1448,fd=16))
LISTEN     129    128                      :::31095                   :::*      users:(("kube-proxy",pid=1448,fd=60))
LISTEN     3      128                      :::30328                   :::*      users:(("kube-proxy",pid=1448,fd=11))
LISTEN     129    128                      :::31641                   :::*      users:(("kube-proxy",pid=1448,fd=58))
LISTEN     129    128                      :::32634                   :::*      users:(("kube-proxy",pid=1448,fd=12))
LISTEN     36     128                      :::32348                   :::*      users:(("kube-proxy",pid=1448,fd=38))
LISTEN     15     128                      :::30140                   :::*      users:(("kube-proxy",pid=1448,fd=23))
LISTEN     129    128                      :::32381                   :::*      users:(("kube-proxy",pid=1448,fd=48))
root@ip-172-1-57-200:~#
```

- 确定 30720 所属 Service

```
root@ip-172-1-57-200:~# iptables -t nat -S|grep "30720"
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp --dport 30720 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "backend/neo-prod-ccapi-inner-lb:http" -m tcp --dport 30720 -j KUBE-SVC-RK4J2BHJ4QTMGP2E
root@ip-172-1-57-200:~#
```

- 确定对应 Service 下的 endpoints

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe svc neo-prod-ccapi-inner-lb
Name:     neo-prod-ccapi-inner-lb
Namespace:    backend
Labels:     <none>
Selector:   load-balancer-neo-prod-ccapi-inner-lb=true
Type:     NodePort
IP:     100.71.52.243
Port:     http  80/TCP
NodePort:   http  30720/TCP
Endpoints:    100.96.253.46:8080,100.97.100.240:8080,100.97.102.37:8080 + 47 more...
Session Affinity: None
No events.
[deployer@k8s-deploy-prod0]$
[deployer@k8s-deploy-prod0]$ kubectl -n backend describe ep neo-prod-ccapi-inner-lb
Name:   neo-prod-ccapi-inner-lb
Namespace:  backend
Labels:   <none>
Subsets:
  Addresses:    100.96.253.46,100.97.100.240,100.97.102.37,100.97.103.61,100.97.107.58,100.97.111.234,100.97.113.147,100.97.114.27,100.97.116.170,100.97.117.164,100.97.119.18,100.97.120.174,100.97.121.230,100.97.124.103,100.97.125.71,100.97.130.236,100.97.131.188,100.97.132.169,100.97.133.116,100.97.134.139,100.97.135.98,100.97.139.178,100.97.14.210,100.97.142.250,100.97.20.19,100.97.59.141,100.97.61.220,100.97.63.58,100.97.65.158,100.97.66.44,100.97.70.21,100.97.72.24,100.97.73.163,100.97.74.208,100.97.75.18,100.97.78.143,100.97.79.13,100.97.83.199,100.97.84.108,100.97.85.125,100.97.86.117,100.97.87.6,100.97.88.76,100.97.89.165,100.97.93.51,100.97.94.76,100.97.95.116,100.97.97.94,100.97.98.37,100.97.99.185
  NotReadyAddresses:  <none>
  Ports:
    Name  Port  Protocol
    ----  ----  --------
    http  8080  TCP

No events.
[deployer@k8s-deploy-prod0]$
```

- 确定 "100.96.253.46" 这个 endpoint 所属的 node 地址

```
[deployer@k8s-deploy-prod0]$ kubectl -n backend get pods -o wide --show-all | grep "100.96.253.46"
neo-prod-cc-api-v202-rfxh9                                  1/1       Running       0          3d        100.96.253.46    ip-172-1-56-178.cn-north-1.compute.internal
[deployer@k8s-deploy-prod0]$
```

- 登录目标 node

```
[deployer@k8s-deploy-prod0]$ ssh admin@ip-172-1-56-178.cn-north-1.compute.internal -i ~/.ssh/id_rsa_prod0-cluster-cn-north1a
```

### k8s node 信息

网卡配置

```
root@ip-172-1-56-178:~# ifconfig
cbr0      Link encap:Ethernet  HWaddr 0a:58:64:60:fd:01
          inet addr:100.96.253.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::9ce2:9bff:fefb:f1ab/64 Scope:Link
          UP BROADCAST RUNNING PROMISC MULTICAST  MTU:9001  Metric:1
          RX packets:7982362091 errors:0 dropped:0 overruns:0 frame:0
          TX packets:7388836277 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2022334724737 (1.8 TiB)  TX bytes:3524130937171 (3.2 TiB)

docker0   Link encap:Ethernet  HWaddr 02:42:f8:34:46:4e
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

eth0      Link encap:Ethernet  HWaddr 02:de:d8:6d:5d:da
          inet addr:172.1.56.178  Bcast:172.1.63.255  Mask:255.255.224.0
          inet6 addr: fe80::de:d8ff:fe6d:5dda/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:18122688959 errors:0 dropped:0 overruns:0 frame:0
          TX packets:18330086338 errors:0 dropped:12 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:6439254230367 (5.8 TiB)  TX bytes:5276372237184 (4.7 TiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:494 errors:0 dropped:0 overruns:0 frame:0
          TX packets:494 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:39560 (38.6 KiB)  TX bytes:39560 (38.6 KiB)
```

docker 容器

```
root@ip-172-1-56-178:~# docker ps
CONTAINER ID        IMAGE                                                                        COMMAND                  CREATED             STATUS              PORTS               NAMES
af45de2d3dfd        prod-reg.llsops.com/backend-rls/neo_production:6.9.1_1.2.23                  "/entrypoint.sh puma"    3 days ago          Up 3 days                               k8s_neo-web.5cfc341d_neo-prod-cc-api-v202-rfxh9_backend_934835a3-1dc7-11e8-b481-02ab8baa063e_381d31f1
2d6e237f6d33        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 3 days ago          Up 3 days                               k8s_POD.d8dbe16c_neo-prod-cc-api-v202-rfxh9_backend_934835a3-1dc7-11e8-b481-02ab8baa063e_2468705c
ee13541edc14        prod-reg.llsops.com/backend-rls/neo_production:6.9.1_1.2.23                  "/entrypoint.sh puma"    3 days ago          Up 3 days                               k8s_neo-web.d4455bd8_neo-prod-api-v205-ahfcc_backend_69170c00-1dc7-11e8-b481-02ab8baa063e_d656ab22
b4508b4b74ef        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 3 days ago          Up 3 days                               k8s_POD.d8dbe16c_neo-prod-api-v205-ahfcc_backend_69170c00-1dc7-11e8-b481-02ab8baa063e_09c5d069
6b7e9d5d4250        prod-reg.llsops.com/library/k8s-fluentd-kafka-prod:v2.0.4                    "/bin/sh -c '/run.sh "   9 days ago          Up 9 days                               k8s_fluentd-kafka.a81d43b6_fluentd-kafka-v1-v1vdt_kube-system_feb05d33-1931-11e8-b481-02ab8baa063e_c75c29b1
3e0bab7a92df        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 9 days ago          Up 9 days                               k8s_POD.d8dbe16c_fluentd-kafka-v1-v1vdt_kube-system_feb05d33-1931-11e8-b481-02ab8baa063e_0e016c07
831cb55912f2        prod-reg.llsops.com/gcr-io-rls/google_containers-kube-state-metrics:v0.5.0   "/kube-state-metrics "   3 months ago        Up 3 months                             k8s_kube-state-metrics.abccef2c_kubestate-prod-metrics-v001-b0rph_platform_967d5cd9-d026-11e7-b481-02ab8baa063e_ed573815
64d46bd3aae0        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 3 months ago        Up 3 months                             k8s_POD.d8dbe16c_kubestate-prod-metrics-v001-b0rph_platform_967d5cd9-d026-11e7-b481-02ab8baa063e_18174801
32745ba0ce28        prod-reg.llsops.com/backend-rls/anatawa:master-dc7795f9                      "sidekiq"                5 months ago        Up 5 months                             k8s_backend-rls-anatawa.3a7d2dcb_anatawa-production-sidekiq-v004-vzebm_backend_d0055095-a25f-11e7-b481-02ab8baa063e_61235d81
6485019a721a        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 5 months ago        Up 5 months                             k8s_POD.d8dbe16c_anatawa-production-sidekiq-v004-vzebm_backend_d0055095-a25f-11e7-b481-02ab8baa063e_96fd3c6d
8203f4f76e62        gcr.io/google_containers/kube-proxy:v1.4.6                                   "/bin/sh -c 'kube-pro"   6 months ago        Up 6 months                             k8s_kube-proxy.94fe1699_kube-proxy-ip-172-1-56-178.cn-north-1.compute.internal_kube-system_f4ed6116beda8195ebb67e2e60c5e90d_03749f68
70d70555da99        gcr.io/google_containers/pause-amd64:3.0                                     "/pause"                 6 months ago        Up 6 months                             k8s_POD.d8dbe16c_kube-proxy-ip-172-1-56-178.cn-north-1.compute.internal_kube-system_f4ed6116beda8195ebb67e2e60c5e90d_54a69074
root@ip-172-1-56-178:~#
```


### k8s pod 信息

```
root@ip-172-1-56-178:~# docker exec -it af45de2d3dfd /bin/bash
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

网络和 CPU 信息

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if553: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 0a:58:64:60:fd:2e brd ff:ff:ff:ff:ff:ff
    inet 100.96.253.46/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a486:44ff:fe19:a94c/64 scope link
       valid_lft forever preferred_lft forever
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#

root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    2
Core(s) per socket:    2
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 63
Model name:            Intel(R) Xeon(R) CPU E5-2666 v3 @ 2.90GHz
Stepping:              2
CPU MHz:               2900.110
BogoMIPS:              5800.22
Hypervisor vendor:     Xen
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              25600K
NUMA node0 CPU(s):     0-3
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

业务进程信息

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1 22.3 16.6 3100456 1278828 ?     Ssl  Mar02 1042:03 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root      9791  0.0  0.0  22184  3744 ?        Ss   16:30   0:00 /bin/bash
root     12721  0.0  0.0  19208  2336 ?        R+   16:49   0:00 ps aux
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ps -eLf
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
root         1     0     1  0   27 Mar02 ?        00:00:10 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    14  0   27 Mar02 ?        00:00:32 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    27  0   27 Mar02 ?        00:00:17 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    33  0   27 Mar02 ?        00:00:07 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    42  0   27 Mar02 ?        00:01:05 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    43  0   27 Mar02 ?        00:00:10 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    44  0   27 Mar02 ?        00:00:00 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    45  0   27 Mar02 ?        00:05:29 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    47  0   27 Mar02 ?        00:00:02 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    56  0   27 Mar02 ?        00:07:19 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    88  0   27 Mar02 ?        00:01:52 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    89  0   27 Mar02 ?        00:00:00 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0    90  0   27 Mar02 ?        00:02:48 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  5356  0   27 Mar03 ?        00:00:26 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  4089  0   27 Mar04 ?        00:01:02 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0 14047  0   27 13:15 ?        00:00:01 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  1136  0   27 15:34 ?        00:00:00 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  1137  0   27 15:34 ?        00:00:00 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  1138  0   27 15:34 ?        00:00:00 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  6867  1   27 16:12 ?        00:00:25 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  6869  1   27 16:12 ?        00:00:20 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9893  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9896  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9899  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9901  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9909  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root         1     0  9910  0   27 16:31 ?        00:00:06 puma 3.10.0 (tcp://0.0.0.0:8080) [neo]
root      9791     0  9791  0    1 16:30 ?        00:00:00 /bin/bash
root     11796  9791 11796  0    1 16:43 ?        00:00:00 ps -eLf
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

监听端口信息

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -nltp | awk '{print $0}'
State      Recv-Q Send-Q        Local Address:Port          Peer Address:Port
LISTEN     0      128                       *:23333                    *:*      users:(("ruby",pid=1,fd=11))
LISTEN     0      128                       *:8080                     *:*      users:(("ruby",pid=1,fd=26))
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

网络链接分布情况（这个输出结果随时间有较大波动）

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 43
ESTAB 123
TIME-WAIT 625
LISTEN 2
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#

root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
LAST-ACK 6
CLOSE-WAIT 35
ESTAB 130
TIME-WAIT 636
LISTEN 2
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#


root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 31
ESTAB 134
TIME-WAIT 685
LISTEN 2
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#

...（N 长时间后的一次抽样）...

root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 22
ESTAB 148
TIME-WAIT 1542
LISTEN 2
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```


可以看到各种连接状态的量都不是很高，唯一需要查看的就是 CLOSE-WAIT 状态的连接情况；


#### CLOSE-WAIT 情况

第一次采样（35 个）

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | grep "CLOSE-WAIT"
tcp    CLOSE-WAIT 1      0          100.96.253.46:36018     172.31.13.216:19000   -- 第二次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:49968     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:55328     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:36044     172.31.13.216:19000   -- 第二次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:51822     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:56928     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35808      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51916     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:54522      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:36028     172.31.13.216:19000   -- 第二次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:52182     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35838      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51458     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50402     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42200      172.31.3.148:19000
tcp    CLOSE-WAIT 32     0          100.96.253.46:40708      54.222.49.26:443
tcp    CLOSE-WAIT 1      0          100.96.253.46:33008      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:60030     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58434     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:47218     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35226     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58492     172.31.13.216:19000   -- 第二次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:36048     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:40918      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50956     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41070     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42238     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58800      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35834      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:45540      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:40912      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41162      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35804      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50950     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:57174     172.31.13.216:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

第二次采样（31 个）

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | grep "CLOSE-WAIT"
tcp    CLOSE-WAIT 1      0          100.96.253.46:49968     172.31.13.216:19000  -- 第三次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:55328     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51822     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:56928     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35808      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51916     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:54522      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:52182     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35838      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51458     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50402     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42200      172.31.3.148:19000
tcp    CLOSE-WAIT 32     0          100.96.253.46:40708      54.222.49.26:443
tcp    CLOSE-WAIT 1      0          100.96.253.46:33008      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:60030     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58434     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:47218     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35226     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:36048     172.31.13.216:19000  -- 第三次中没有
tcp    CLOSE-WAIT 1      0          100.96.253.46:40918      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50956     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41070     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42238     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58800      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35834      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:45540      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:40912      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41162      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35804      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50950     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:57174     172.31.13.216:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

第三次采样（35 个）

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | grep "CLOSE-WAIT"
tcp    CLOSE-WAIT 1      0          100.96.253.46:45108      172.31.3.148:19000  -- 新增（在下面 ESTAB 中出现过）
tcp    CLOSE-WAIT 1      0          100.96.253.46:55328     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51822     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:56928     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:33916      172.31.3.148:19000  -- 新增
tcp    CLOSE-WAIT 1      0          100.96.253.46:35808      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51916     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:54522      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:52182     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:49662     172.31.13.216:19000  -- 新增
tcp    CLOSE-WAIT 1      0          100.96.253.46:35838      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:51458     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50402     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42200      172.31.3.148:19000
tcp    CLOSE-WAIT 32     0          100.96.253.46:40708      54.222.49.26:443
tcp    CLOSE-WAIT 1      0          100.96.253.46:33008      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:60030     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:32802     172.31.13.216:19000  -- 新增
tcp    CLOSE-WAIT 1      0          100.96.253.46:58434     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:47218     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35226     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:37260     172.31.13.216:19000  -- 新增（在下面 ESTAB 中出现过）
tcp    CLOSE-WAIT 1      0          100.96.253.46:40196      172.31.3.148:19000  -- 新增
tcp    CLOSE-WAIT 1      0          100.96.253.46:40918      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50956     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41070     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:42238     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:58800      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35834      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:45540      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:40912      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:41162      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:35804      172.31.3.148:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:50950     172.31.13.216:19000
tcp    CLOSE-WAIT 1      0          100.96.253.46:57174     172.31.13.216:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

分布情况如下（某次采样数据）

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "CLOSE-WAIT" | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
     19 100.96.253.46 172.31.13.216:19000
     16 100.96.253.46 172.31.3.148:19000
      1 100.96.253.46 54.222.49.26:443
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

通过 playdock 确认

- 172.31.13.216 对应 neo-cache-codis-proxy-2xlarge-prod-0
- 172.31.3.148 对应 neo-store-codis-proxy-2xlarge-prod-1


#### TIME-WAIT 情况

TIME-WAIT 的分布情况没有什么特别的，几乎都是唯一一个；

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | grep "TIME-WAIT"
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:52961
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:45491
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:60014
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42167
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46373
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:48160
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:36820
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:51261
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42367
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:37419
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46108
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:39383
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:35707
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:53875
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:34855
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:47573
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:53417
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:44615
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:41482
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:53069
tcp    TIME-WAIT  0      0          100.96.253.46:23333      100.96.253.1:34698
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48020
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:59906
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:35103
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:32961
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:33245
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:37251
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:55126
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:34467
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:34497
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:42147
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:43828
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:55794
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:59145
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45850
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:47146
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48960
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:50678
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:32916
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:53505
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:59797
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:43826
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:42283
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:39866
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:36396
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:39646
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:56961
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:34697
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:49929
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:39461
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:40324
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:42277
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:48647
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:41066
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:40326
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:33806
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:32916
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45479
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:42116
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:45547
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:45367
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:44751
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:41832
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:42434
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:44865
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:55034
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:58415
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:47915
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:43629
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:60078
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:53767
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:40744
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:38904
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:44528
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:45972
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:51348
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:35944
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:57325
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:47110
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:57959
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:47787
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51355
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:36416
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:48812
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:35754
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:46780
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:42405
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:38570
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:44213
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:34957
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:48083
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:47628
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:50384
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:45666
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:39647
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:35099
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:37973
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:40568
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51372
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:32956
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:40141
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:37477
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:59558
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:56674
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:51188
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:58504
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:51003
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:44152
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:33633
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:56051
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:59128
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:52218
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:44266
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:59420
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:45380
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:54635
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:41275
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:42957
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:44957
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:54134
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:51752
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:58132
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:40023
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:43450
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:54975
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:44447
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:40399
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:39490
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:35113
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:38554
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:35296
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:35918
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:40848
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:43780
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:49906
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:52202
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:49907
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:47748
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:58279
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:43451
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:60809
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:60697
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:49363
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:46582
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:54774
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42249
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:33960
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:43792
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:57016
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:51457
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:44070
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:54708
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:43996
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:38511
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:58314
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:43528
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:59372
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:48617
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:40848
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:37837
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:37908
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46879
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:54211
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:47430
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:34004
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:39856
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48755
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:38671
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:33327
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:48511
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:44841
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:45774
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:55295
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:59475
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:33038
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46333
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:34768
tcp    TIME-WAIT  0      0          100.96.253.46:23333      100.96.253.1:34102
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:60095
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:44371
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:35840
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:54270
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:47339
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:55830
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:58733
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:48120
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:57664
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:45618
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:60655
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:48763
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:36283
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:45283
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:37476
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:52006
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:36352
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:43661
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:51773
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:46865
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48979
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:42689
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:46462
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51208
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:50016
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:46405
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:49959
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:35780
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:56448
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:56484
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:37615
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:42930
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:38575
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:49330
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:54470
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:48340
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:41116
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45662
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:43643
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:46382
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:51211
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:34701
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:45859
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:60422
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:48367
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:53419
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:59467
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46440
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:45353
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:53353
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:53893
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:32801
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:44523
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:47642
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:47097
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:55796
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:43486
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:47092
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:48427
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:54197
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:45076
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46161
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:60536
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:49687
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:32939
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:41283
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:40781
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:57443
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:36603
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:55865
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:44666
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:57145
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:44572
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45794
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:54479
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:40977
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:33043
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:47664
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:49683
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:45233
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:33469
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:52882
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:56388
tcp    TIME-WAIT  0      0          100.96.253.46:23333      100.96.253.1:34990
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:52454
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:33887
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:46966
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51356
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:37277
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:35941
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:34282
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:42225
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:44865
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:59176
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:58464
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:54744
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:48868
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:57119
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:37890
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:49621
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:55637
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:55587
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:32922
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:35976
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:38995
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:59943
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:47767
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:40932
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:49375
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:41144
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:42392
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:60003
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:32864
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:42747
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:45889
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48262
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:49797
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:51983
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:40638
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:50316
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:38756
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:43309
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:35744
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51108
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:49383
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:43407
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:47220
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:60348
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:54574
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:49400
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:42895
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:51128
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:54304
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:37458
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:53272
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:50440
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:43163
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:41409
tcp    TIME-WAIT  0      0          100.96.253.46:23333      100.96.253.1:34320
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:34224
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:50916
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:59254
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:46504
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:58704
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:52673
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:37456
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:56774
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:45994
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:33973
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:34270
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:39228
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:53769
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:46774
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:53995
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:41849
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46146
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:42050
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:57892
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:39688
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:45489
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:60889
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:42533
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:50028
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:48885
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:37567
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:60822
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:45770
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:47955
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:58917
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:35731
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:59502
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:52645
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:43690
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:52224
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:36583
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42237
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:47935
tcp    TIME-WAIT  0      0          100.96.253.46:23333      100.96.253.1:33890
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:59252
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:41128
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:54720
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:55040
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:35570
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:42991
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:46736
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:52472
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:37578
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:53936
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:43637
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:49272
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:59755
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:58191
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:32979
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:50917
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:40411
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:41007
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:44035
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:39563
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:53959
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:60390
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:53129
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:57716
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:51663
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42754
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:44627
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:58197
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:43161
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:58318
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:41967
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:60910
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:43696
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:60425
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:60162
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:37776
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:38356
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46164
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:46857
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:56361
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:60886
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:54419
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:40070
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:35620
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:47968
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:48098
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:33638
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:38859
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:39752
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:38141
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:33796
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:41339
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:32910
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:43429
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:59719
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:56356
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:33678
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:33346
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:45292
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:46772
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:55589
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:57803
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:47882
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:49965
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:37305
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:40942
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:35866
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:42241
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:43029
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:33347
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:41810
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:44982
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:44872
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:52557
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:49133
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:53725
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:33943
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:49546
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:42001
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:54312
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:38750
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:47712
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:36465
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:46762
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45847
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:55145
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:42656
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:43224
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:52096
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:46892
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:48893
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:40830
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:53928
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45394
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:59767
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:52865
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:41685
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:42147
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:48523
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:34522
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:52089
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:52445
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:53071
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:50845
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:42916
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:45671
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:33840
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:55350
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:46622
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:41812
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:45947
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:37167
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:33321
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:44858
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:42506
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:32809
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:54480
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:34109
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:43390
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:55856
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:53800
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:54165
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:57404
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:53182
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:41751
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:60467
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:38692
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:43130
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:47578
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:47255
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:51937
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:47522
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:34662
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:51336
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:45919
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:45204
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:45655
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:38838
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:39439
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:35046
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:57035
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:50193
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:33335
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:33806
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:49518
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:52683
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:37398
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:47339
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:35827
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:56614
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:34040
tcp    TIME-WAIT  0      0          100.96.253.46:33530    172.31.131.106:50052
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:47663
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:37499
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:36945
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:45591
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:33988
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:53687
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:57546
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:60998
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:41462
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:32888
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:37891
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:45280
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:53406
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:54432
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:37808
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:55030
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:53870
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:35679
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:42504
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:44138
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:52323
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:33320
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:40188
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:55801
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:56239
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:36693
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:36975
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:32993
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:34123
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:35500
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:49393
tcp    TIME-WAIT  0      0          100.96.253.46:32856    172.31.131.106:50052
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:35954
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:53585
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:34469
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:47211
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:51107
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:44001
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:41484
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:45643
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:52752
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:43809
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:40916
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:54019
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46325
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:59336
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:51820
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:42474
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.58.80:59177
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:43065
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:42819
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:56739
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:40858
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:47012
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:41906
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:38367
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:45983
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:44812
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:33988
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:48998
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:52828
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:35385
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:40430
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:43315
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:40151
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:38847
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:38103
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:57520
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:42092
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:60819
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.36.132:44055
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:33335
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:43916
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:43561
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:55288
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:42796
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:57606
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:54864
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:60229
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:43277
tcp    TIME-WAIT  0      0          100.96.253.46:32944    172.31.131.106:50052
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:40434
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:42687
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:46023
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:48962
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:49476
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:32087
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:41959
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:46070
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46930
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:39028
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:52358
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:41111
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:60674
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:44147
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:58362
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:60192
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:50062
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:51541
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:51031
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:52426
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:58872
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:39140
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:49241
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:57285
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:36896
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:48146
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:39460
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:36856
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:49957
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:39603
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:47525
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:55606
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46544
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:57871
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:59932
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:35847
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:54116
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.48:57937
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:43961
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:42827
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:43254
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:52346
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:36475
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:57048
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:45358
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:35266
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:55083
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:58206
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.56.22:51797
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:46883
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:60736
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:34272
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:48826
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:55470
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:47591
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:38446
tcp    TIME-WAIT  0      0          100.96.253.46:8080         172.1.43.2:60411
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.37.106:44116
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:59817
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:40174
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:43200
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.47.41:51811
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:36614
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:56918
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.56.224:39471
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.44.23:45063
tcp    TIME-WAIT  0      0          100.96.253.46:8080        172.1.39.66:40452
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:43561
tcp    TIME-WAIT  0      0          100.96.253.46:8080       100.96.253.1:53584
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.38.108:56852
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.50.119:33654
tcp    TIME-WAIT  0      0          100.96.253.46:8080       172.1.57.200:34340
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

#### ESTAB 情况

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -na | grep "ESTAB"
tcp    ESTAB      0      0          100.96.253.46:49664     172.31.13.216:19000   -- neo-cache-codis-proxy-2xlarge-prod-0 (cache_redis)
tcp    ESTAB      0      0          100.96.253.46:45108      172.31.3.148:19000   -- neo-store-codis-proxy-2xlarge-prod-1
tcp    ESTAB      0      0          100.96.253.46:51546      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:46666      172.31.14.73:6379    -- neo-redis-prod-0-master (sidekiq_redis)
tcp    ESTAB      0      0          100.96.253.46:34638      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:59154      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:59830     50.31.164.146:443
tcp    ESTAB      0      0          100.96.253.46:52280     172.31.136.58:6693
tcp    ESTAB      0      0          100.96.253.46:52832      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:46732      172.31.8.152:6693
tcp    ESTAB      0      0          100.96.253.46:54052      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:39694      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:54052      172.31.9.204:6693
tcp    ESTAB      0      0          100.96.253.46:52300      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:38218      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:52796      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:23333      100.97.115.7:46824   -- pod 2 pod
tcp    ESTAB      0      0          100.96.253.46:39688      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:52274      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:32808     172.31.13.216:19000   -- cache_redis
tcp    ESTAB      0      0          100.96.253.46:37336      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:39458      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:48968      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:56186      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:53802      172.31.8.188:6693
tcp    ESTAB      0      0          100.96.253.46:39716      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:37264     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:52814      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:47200     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:33932      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:43418     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39704      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:51298     172.31.15.253:443
tcp    ESTAB      0      0          100.96.253.46:56184      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:46950      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:32806     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:37346      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:33914      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:39468      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:40212     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:60526     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39686      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:56204      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:58666    100.66.143.181:80
tcp    ESTAB      0      0          100.96.253.46:49662     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:37266     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:37278      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:58496     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:8080       172.1.57.200:52819
tcp    ESTAB      0      0          100.96.253.46:34636      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:36060     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39692      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:45110      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:49810      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:37516      172.31.14.73:6379
tcp    ESTAB      0      0          100.96.253.46:39474      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:36026     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39706      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:38226      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:48272      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:40200      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:38204      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:32802     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:52828      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:43726      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:49694     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:53040      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:45400      172.31.8.152:6693
tcp    ESTAB      0      0          100.96.253.46:36544    172.31.137.135:6693
tcp    ESTAB      0      0          100.96.253.46:36978      172.31.14.73:6379
tcp    ESTAB      0      0          100.96.253.46:39682      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39722      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:40488      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:47220     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:34296     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39720      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:37316      172.31.10.51:6693
tcp    ESTAB      0      0          100.96.253.46:43416     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:53382      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:37260     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39710      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39184      172.31.14.73:6379
tcp    ESTAB      0      0          100.96.253.46:36030     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:38232      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:42204        172.31.8.4:6693
tcp    ESTAB      0      0          100.96.253.46:60528     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39708      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:60522     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:43286      172.31.10.71:6693
tcp    ESTAB      0      0          100.96.253.46:54072      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:37262     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:48206      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:39676      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:43392     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:39684      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:48320       172.31.0.28:6693
tcp    ESTAB      0      0          100.96.253.46:54062      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:35982      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:47222     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:32804     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:47132      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:33928      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:39718      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:49666     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:59148      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:52826      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39728      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:52278      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:52812      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39700      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39714      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:34014     172.31.11.163:19000  -- leaderboard_redis
tcp    ESTAB      0      0          100.96.253.46:36016     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:33930      172.31.3.148:19000
tcp    ESTAB      0      0          100.96.253.46:39712      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:39696      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:38522      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:39702      172.31.5.120:6693
tcp    ESTAB      0      0          100.96.253.46:54066      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:37330      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:8080       172.1.57.200:53451
tcp    ESTAB      0      0          100.96.253.46:37008      172.31.8.152:6693
tcp    ESTAB      0      0          100.96.253.46:34276     172.31.13.216:19000
tcp    ESTAB      0      0          100.96.253.46:53386      172.31.13.32:6693
tcp    ESTAB      0      0          100.96.253.46:46006     172.31.136.58:6693
tcp    ESTAB      0      0          100.96.253.46:59660     172.31.136.58:6693
tcp    ESTAB      0      0          100.96.253.46:39698      172.31.5.120:6693
tcp    ESTAB      0      0       ::ffff:100.96.253.46:36156 ::ffff:172.31.4.7:50055
tcp    ESTAB      0      0       ::ffff:100.96.253.46:50946 ::ffff:100.70.205.110:50066
tcp    ESTAB      0      0       ::ffff:100.96.253.46:36380 ::ffff:100.67.167.150:50067
tcp    ESTAB      0      0       ::ffff:100.96.253.46:44456 ::ffff:100.67.144.153:51060
tcp    ESTAB      0      17      ::ffff:100.96.253.46:59538 ::ffff:100.71.144.163:50052
tcp    ESTAB      0      0       ::ffff:100.96.253.46:34394  ::ffff:100.68.155.73:50051
tcp    ESTAB      0      0       ::ffff:100.96.253.46:39608   ::ffff:100.71.42.82:50055
tcp    ESTAB      0      0       ::ffff:100.96.253.46:39552   ::ffff:100.71.42.82:50055
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

分布情况如下

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "ESTAB" | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
      1  ::ffff:100.67.144.153:51060
      1  ::ffff:100.67.167.150:50067
      1  ::ffff:100.68.155.73:50051
      1  ::ffff:100.70.205.110:50066
      1  ::ffff:100.71.144.163:50052
      2  ::ffff:100.71.42.82:50055
      1  ::ffff:172.31.4.7:50055
      1 100.96.253.46 100.66.143.181:80
      1 100.96.253.46 100.96.253.1:33835
      1 100.96.253.46 100.97.115.7:46824
      1 100.96.253.46 172.1.39.66:43729
      1 100.96.253.46 172.1.56.224:53442
      1 100.96.253.46 172.31.0.28:6693
      1 100.96.253.46 172.31.10.51:6693
      1 100.96.253.46 172.31.10.71:6693
      1 100.96.253.46 172.31.11.163:19000
     37 100.96.253.46 172.31.13.216:19000  -- redis
     32 100.96.253.46 172.31.13.32:6693    -- aws mysql
      3 100.96.253.46 172.31.136.58:6693
      1 100.96.253.46 172.31.137.135:6693
      4 100.96.253.46 172.31.14.73:6379    -- redis
      1 100.96.253.46 172.31.15.253:443
     12 100.96.253.46 172.31.3.148:19000   -- redis
     32 100.96.253.46 172.31.5.120:6693    -- aws mysql
      3 100.96.253.46 172.31.8.152:6693
      1 100.96.253.46 172.31.8.188:6693
      1 100.96.253.46 172.31.8.4:6693
      1 100.96.253.46 172.31.9.204:6693
      1 100.96.253.46 50.31.164.146:443
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

#### 6693 端口情况

在 `/opt/apps/neo/config/database.yml` 中能够查到所有 **6693** 端口均对应 aws 上的 mysql 服务；

分布情况如下

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "6693"
ESTAB      0      0             100.96.253.46:38368         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:52112         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38398         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:43364         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:59154         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52048         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38396         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:52280        172.31.136.58:6693
ESTAB      0      0             100.96.253.46:46732         172.31.8.152:6693
ESTAB      0      0             100.96.253.46:38400         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:54052         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:39694         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:54052         172.31.9.204:6693
ESTAB      0      0             100.96.253.46:45872         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52274         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:53802         172.31.8.188:6693
ESTAB      0      0             100.96.253.46:39716         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:43356         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38372         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:37346         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38404         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:56204         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:38374         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:45840         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52054         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38370         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:38378         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:34636         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:54726         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38384         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:43444         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:43388         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:54724         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38386         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:38388         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:48272         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38402         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:45400         172.31.8.152:6693
ESTAB      0      0             100.96.253.46:36544       172.31.137.135:6693
ESTAB      0      0             100.96.253.46:39682         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:37316         172.31.10.51:6693
ESTAB      0      0             100.96.253.46:53382         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38392         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:39710         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:42204           172.31.8.4:6693
ESTAB      0      0             100.96.253.46:52034         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:39708         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:52108         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:43286         172.31.10.71:6693
ESTAB      0      0             100.96.253.46:54722         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:48320          172.31.0.28:6693
ESTAB      0      0             100.96.253.46:38382         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:52114         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38390         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:38406         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:54062         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:35982         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:47132         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:39718         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:59148         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52826         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:39728         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52278         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:52812         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:39700         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:54720         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:43394         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38376         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:39712         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:38522         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:38380         172.31.5.120:6693
ESTAB      0      0             100.96.253.46:37330         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:37008         172.31.8.152:6693
ESTAB      0      0             100.96.253.46:53386         172.31.13.32:6693
ESTAB      0      0             100.96.253.46:46006        172.31.136.58:6693
ESTAB      0      0             100.96.253.46:59660        172.31.136.58:6693
ESTAB      0      0             100.96.253.46:39698         172.31.5.120:6693
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "6693" | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
      1 100.96.253.46 172.31.0.28:6693
      1 100.96.253.46 172.31.10.51:6693
      1 100.96.253.46 172.31.10.71:6693
     32 100.96.253.46 172.31.13.32:6693
      3 100.96.253.46 172.31.136.58:6693
      1 100.96.253.46 172.31.137.135:6693
     32 100.96.253.46 172.31.5.120:6693
      3 100.96.253.46 172.31.8.152:6693
      1 100.96.253.46 172.31.8.188:6693
      1 100.96.253.46 172.31.8.4:6693
      1 100.96.253.46 172.31.9.204:6693
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

#### 19000 端口情况

```
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000"
ESTAB      0      0             100.96.253.46:49664        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:51546         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:36690        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36692        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:38864         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:40856        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:40858        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:58188        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36716        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36708        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36702        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:56110        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:46978         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:50644        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:55328        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:40854        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:38810         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:55442         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:36688        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:56928        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:34018        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:35808         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:51916        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:33914         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:40212        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:34016        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:38866         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35838         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:36714        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:40602        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:55746         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:36704        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:50402        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:42200         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:34026        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:33008         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:60030        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:33982        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36686        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36684        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36710        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:49476        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36718        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:52772         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35226        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:36694        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:56116        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:40918         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:40604        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:43392        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:41070        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:46060         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:58186        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:58800         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35834         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:38784         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:45540         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:34024        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:41162         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35804         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:38870         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:34014        172.31.11.163:19000
ESTAB      0      0             100.96.253.46:58184        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:33930         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:46620         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:36696        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:40600        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:58190        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:38808         172.31.3.148:19000
ESTAB      0      0             100.96.253.46:34020        172.31.13.216:19000
ESTAB      0      0             100.96.253.46:50646        172.31.13.216:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000" | grep "CLOSE-WAIT"
CLOSE-WAIT 1      0             100.96.253.46:55328        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:56928        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:35808         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:51916        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:35838         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:50402        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:42200         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:33008         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:60030        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:35226        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:40918         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:41070        172.31.13.216:19000
CLOSE-WAIT 1      0             100.96.253.46:58800         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35834         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:38784         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:45540         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:41162         172.31.3.148:19000
CLOSE-WAIT 1      0             100.96.253.46:35804         172.31.3.148:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000" |grep "CLOSE-WAIT" | wc -l
18
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000" |grep "ESTAB" | wc -l
53
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000" | wc -l
71
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo# ss -tan | grep "19000" | awk '{print $(NF-1)" "$(NF)}' | sed 's/:[^ ]* / /g' | sort | uniq -c
      1 100.96.253.46 172.31.11.163:19000
     45 100.96.253.46 172.31.13.216:19000
     25 100.96.253.46 172.31.3.148:19000
root@neo-prod-cc-api-v202-rfxh9:/opt/apps/neo#
```

在 `/opt/apps/neo/config/redis_settings.yml` 中能看到

```
redis: <%= "redis://#{ ENV['STORE_REDIS_HOSTS'] ? ENV['STORE_REDIS_HOSTS'].split(';').sample : '172.31.11.163' }:19000/0" %>
cache_redis: <%= "redis://#{ ENV['CACHE_REDIS_HOSTS'] ? ENV['CACHE_REDIS_HOSTS'].split(';').sample : '172.31.13.216' }:19000/0" %>
leaderboard_redis: <%= "redis://#{ ENV['STORE_REDIS_HOSTS'] ? ENV['STORE_REDIS_HOSTS'].split(';').sample : '172.31.11.163' }:19000/3" %>
notification_redis: <%= "redis://#{ ENV['STORE_REDIS_HOSTS'] ? ENV['STORE_REDIS_HOSTS'].split(';').sample : '172.31.11.163' }:19000/5" %>
campaign_redis: <%= "redis://#{ ENV['STORE_REDIS_HOSTS'] ? ENV['STORE_REDIS_HOSTS'].split(';').sample : '172.31.11.163' }:19000/4" %>
rollout_redis: <%= "redis://#{ ENV['STORE_REDIS_HOSTS'] ? ENV['STORE_REDIS_HOSTS'].split(';').sample : '172.31.11.163' }:19000/6" %>
```

简化后有

- **redis**:              '172.31.11.163' 19000/0"
- **cache_redis**:        '172.31.13.216' 19000/0"
- **leaderboard_redis**:  '172.31.11.163' 19000/3"
- **notification_redis**: '172.31.11.163' 19000/5"
- **campaign_redis**:     '172.31.11.163' 19000/4"
- **rollout_redis**:      '172.31.11.163' 19000/6"


另外，还有

```
./config/redis_settings.yml:2:  redis: redis://localhost:6379/0
./config/redis_settings.yml:3:  leaderboard_redis: redis://localhost:6379/0
./config/redis_settings.yml:4:  cache_redis: redis://localhost:6379/0
./config/redis_settings.yml:5:  sidekiq_redis: redis://localhost:6379/0
./config/redis_settings.yml:6:  notification_redis: redis://localhost:6379/0
./config/redis_settings.yml:7:  search_update_redis: redis://localhost:6379/0
./config/redis_settings.yml:8:  campaign_redis: redis://localhost:6379/0
./config/redis_settings.yml:9:  algorithm_recommend_redis: redis://localhost:6379/0
./config/redis_settings.yml:10:  pusher_redis: redis://localhost:6379/0
./config/redis_settings.yml:45:  sidekiq_redis: redis://172.31.14.73:6379/2
./config/redis_settings.yml:48:  search_update_redis: redis://search-index-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/0
./config/redis_settings.yml:50:  algorithm_recommend_redis: redis://neo-algorithm-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/0
./config/redis_settings.yml:51:  algorithm_recommend_course_redis: redis://neo-algorithm-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/0
./config/redis_settings.yml:52:  algorithm_recommend_home_essential_redis: redis://neo-algorithm-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/1
./config/redis_settings.yml:53:  algorithm_recommend_topic_redis: redis://neo-algorithm-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/1
./config/redis_settings.yml:54:  pusher_redis: redis://172.31.10.108:6379/0
./config/redis_settings.yml:55:  rack_attack_redis: redis://attack-redis.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/0
./config/redis_settings.yml:56:  weekly_report_redis: redis://lingome-wr-prod.qh1egb.0001.cnn1.cache.amazonaws.com.cn:6379/0
```




