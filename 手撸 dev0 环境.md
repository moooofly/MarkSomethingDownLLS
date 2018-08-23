# 手撸 dev0 环境

## 结论

- 没有发现 kube-proxy 溢出问题（监听端口的 accept queue 由 128 变更为 65536）
- kube-apiserver 侧存在一些 CLOSE_WAIT 状态（对应与 kubelet 之间的连接）
- 存在一些和 bird 和 protokube 相关的 SYN_SENT ；

## 服务和监听端口

| 端口 | 服务 | backlog |
| ---- | ---- | ---- |
| 179 | bird | 8 |
| 2380/2381/4001/4002 | etcd | 32768 |
| 3998 | dns-controller | 32768 |
| 3999 | protokube | 32768 |
| 4194/10248/10255/10250 | kubelet | 32768 |
| 8080/443 | kube-apiserver | 32768 |
| 10251 | kube-scheduler | 32768 |
| 10252 | kube-controller | 32768 |


## 排查

- nodes 列表

```
admin@bastion-dev0:~$ kubectl get nodes
NAME                                         STATUS    ROLES     AGE       VERSION
ip-10-3-32-171.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-32-238.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-33-109.cn-north-1.compute.internal   Ready     master    125d      v1.8.1
ip-10-3-33-209.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-33-51.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-34-238.cn-north-1.compute.internal   Ready     node      92d       v1.8.1
ip-10-3-35-252.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-36-71.cn-north-1.compute.internal    Ready     node      2d        v1.8.1
ip-10-3-37-0.cn-north-1.compute.internal     Ready     node      27d       v1.8.1
ip-10-3-37-112.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-39-102.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-39-241.cn-north-1.compute.internal   Ready     node      12d       v1.8.1
ip-10-3-39-254.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-39-78.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-40-2.cn-north-1.compute.internal     Ready     node      92d       v1.8.1
ip-10-3-40-69.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-41-201.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-41-72.cn-north-1.compute.internal    Ready     node      7d        v1.8.1
ip-10-3-42-98.cn-north-1.compute.internal    Ready     node      2d        v1.8.1
ip-10-3-43-104.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-43-126.cn-north-1.compute.internal   Ready     master    125d      v1.8.1
ip-10-3-44-54.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-45-15.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-45-69.cn-north-1.compute.internal    Ready     node      1d        v1.8.1
ip-10-3-46-100.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-47-49.cn-north-1.compute.internal    Ready     master    125d      v1.8.1
ip-10-3-49-149.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-50-127.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-50-76.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-50-95.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-51-16.cn-north-1.compute.internal    Ready     node      12d       v1.8.1
ip-10-3-51-209.cn-north-1.compute.internal   Ready     node      92d       v1.8.1
ip-10-3-51-219.cn-north-1.compute.internal   Ready     node      2d        v1.8.1
ip-10-3-52-255.cn-north-1.compute.internal   Ready     node      2d        v1.8.1
ip-10-3-53-140.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-53-5.cn-north-1.compute.internal     Ready     node      27d       v1.8.1
ip-10-3-54-190.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-55-12.cn-north-1.compute.internal    Ready     node      2d        v1.8.1
ip-10-3-55-76.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-56-54.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-57-6.cn-north-1.compute.internal     Ready     node      27d       v1.8.1
ip-10-3-58-191.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-58-38.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-59-125.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-59-21.cn-north-1.compute.internal    Ready     node      15d       v1.8.1
ip-10-3-61-139.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-61-150.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-61-73.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
ip-10-3-63-104.cn-north-1.compute.internal   Ready     node      16d       v1.8.1
ip-10-3-63-151.cn-north-1.compute.internal   Ready     node      27d       v1.8.1
ip-10-3-63-42.cn-north-1.compute.internal    Ready     node      27d       v1.8.1
admin@bastion-dev0:~$
```

## ip-10-3-32-171.cn-north-1.compute.internal

> node

```
root@ip-10-3-32-171:~# uname -a
Linux ip-10-3-32-171 4.4.78-k8s #1 SMP Fri Jul 28 01:28:39 UTC 2017 x86_64 GNU/Linux
root@ip-10-3-32-171:~#
root@ip-10-3-32-171:~#
root@ip-10-3-32-171:~# lsb_release -a
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux 8.9 (jessie)
Release:	8.9
Codename:	jessie
root@ip-10-3-32-171:~#
```


- SYN-SENT 过多

```
root@ip-10-3-32-171:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
ESTAB 128
TIME-WAIT 2
LISTEN 111
SYN-SENT 145
root@ip-10-3-32-171:~#
```

- protokube/kubelet/kube-proxy 的监听端口 accept queue 均调整为 32768

```
root@ip-10-3-32-171:~# ss -nltp | awk '{print $0}'
State      Recv-Q Send-Q        Local Address:Port          Peer Address:Port
LISTEN     0      128                       *:111                      *:*      users:(("rpcbind",pid=496,fd=8))
LISTEN     0      8                         *:179                      *:*      users:(("bird",pid=2437,fd=7))
LISTEN     0      128                       *:22                       *:*      users:(("sshd",pid=646,fd=3))
LISTEN     0      32768                     *:3999                     *:*      users:(("protokube",pid=1870,fd=5))
LISTEN     0      32768             127.0.0.1:10248                    *:*      users:(("kubelet",pid=1283,fd=20))
LISTEN     0      128                       *:48936                    *:*      users:(("rpc.statd",pid=505,fd=9))
LISTEN     0      32768             127.0.0.1:10249                    *:*      users:(("kube-proxy",pid=1795,fd=4))
...
LISTEN     0      32768                    :::10255                   :::*      users:(("kubelet",pid=1283,fd=21))
LISTEN     0      128                      :::111                     :::*      users:(("rpcbind",pid=496,fd=11))
LISTEN     0      32768                    :::32368                   :::*      users:(("kube-proxy",pid=1795,fd=95))
...
LISTEN     0      32768                    :::30638                   :::*      users:(("kube-proxy",pid=1795,fd=46))
root@ip-10-3-32-171:~#
```

- 所有 SYN-SENT 都集中在 bird 服务上

```
root@ip-10-3-32-171:~# ss -natp | grep "SYN-SENT"
SYN-SENT   0      1               10.3.32.171:40036           10.3.45.28:179    users:(("bird",pid=2437,fd=83))
SYN-SENT   0      1               10.3.32.171:60423           10.3.33.41:179    users:(("bird",pid=2437,fd=70))
SYN-SENT   0      1               10.3.32.171:56999           10.3.55.58:179    users:(("bird",pid=2437,fd=64))
SYN-SENT   0      1               10.3.32.171:50708          10.3.37.233:179    users:(("bird",pid=2437,fd=123))
SYN-SENT   0      1               10.3.32.171:55733          10.3.39.203:179    users:(("bird",pid=2437,fd=132))
SYN-SENT   0      1               10.3.32.171:44915          10.3.63.228:179    users:(("bird",pid=2437,fd=239))
...
```

- bird 和 calico 相关

```
root@ip-10-3-32-171:~# ps aux|grep bird
root      2431  0.0  0.0    744     4 ?        Ss   Feb23   0:00 runsv bird6
root      2432  0.0  0.0    744     4 ?        Ss   Feb23   0:00 runsv bird
root      2435  0.0  0.0    792     4 ?        S    Feb23   5:44 bird6 -R -s /var/run/calico/bird6.ctl -d -c /etc/calico/confd/config/bird6.cfg
root      2437  0.2  0.0   3604  3196 ?        S    Feb23 100:09 bird -R -s /var/run/calico/bird.ctl -d -c /etc/calico/confd/config/bird.cfg
root      2471  0.0  0.0  12700  2072 pts/0    S+   06:07   0:00 grep bird
root@ip-10-3-32-171:~#
```

相关链接：[调查 calico node 很多 SYN_SENT 状态连接问题](https://phab.llsapp.com/T47011)

涉及问题：

- ICMP
- localhost
- SYN 重传
- etcd 元数据


## ip-10-3-32-238.cn-north-1.compute.internal

> node

结论同上

## ip-10-3-33-109.cn-north-1.compute.internal

> master

- CLOSE-WAIT 和 SYN-SENT 较多

```
root@ip-10-3-33-109:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 46
ESTAB 705
TIME-WAIT 39
LISTEN 120
SYN-SENT 239
root@ip-10-3-33-109:~#
```

- 无 accept queue 溢出

```
root@ip-10-3-33-109:~# ss -nltp | awk '{ if ($2 > 0) print $0 }'
State      Recv-Q Send-Q        Local Address:Port          Peer Address:Port
root@ip-10-3-33-109:~#
```

- SYN-SENT 主要集中在 bird 上，少部分在 protokube（似乎 protokube 间相互探测，然而目标地址不在 nodes 信息中）

```
root@ip-10-3-33-109:~# ss -natp | grep "SYN-SENT" |grep -v bird
SYN-SENT   0      1               10.3.33.109:37121           10.3.39.22:3999   users:(("protokube",pid=11427,fd=12))
SYN-SENT   0      1               10.3.33.109:39891          10.3.53.170:3999   users:(("protokube",pid=11427,fd=13))
SYN-SENT   0      1               10.3.33.109:45150           10.3.0.250:3999   users:(("protokube",pid=11427,fd=46))
SYN-SENT   0      1               10.3.33.109:53619           10.3.48.41:3999   users:(("protokube",pid=11427,fd=15))
root@ip-10-3-33-109:~#
```

> protokube 是干啥的？https://github.com/kubernetes/kops/tree/master/protokube
>
>> Protokube acts as a proving ground for code that we will likely want in kubelet, for easy bring-up of a cluster. As we are still figuring out some of these things, it would be inefficient and premature to propose them into kubelet immediately. However, long-term the hope is that we can move some or all of protokube into kubelet itself.
>>
>> Protokube has three main roles:
>>
>> - Mount and discover master volumes
>> - Configures DNS for simple discovery
>> - Applies component configuration

- CLOSE-WAIT 都对应到 kube-apiserver ，且为稳定不变状态；远端端口 10250 为 kubelet 监听端口（**说明 kube-apiserver 主动连接 kubelet ；说明 kubelet 主动关闭连接；说明 kube-apiserver 存在问题**）

```
root@ip-10-3-33-109:~# ss -natp | grep "CLOSE-WAIT"
CLOSE-WAIT 1      0               10.3.33.109:50718          10.3.63.151:10250  users:(("kube-apiserver",pid=27807,fd=179))
CLOSE-WAIT 0      0               10.3.33.109:59948           10.3.61.73:10250  users:(("kube-apiserver",pid=27807,fd=193))
CLOSE-WAIT 0      0               10.3.33.109:60506          10.3.63.151:10250  users:(("kube-apiserver",pid=27807,fd=185))
CLOSE-WAIT 0      0               10.3.33.109:48928          10.3.39.241:10250  users:(("kube-apiserver",pid=27807,fd=210))
CLOSE-WAIT 1      0               10.3.33.109:35242          10.3.59.125:10250  users:(("kube-apiserver",pid=27807,fd=205))
...
CLOSE-WAIT 0      0               10.3.33.109:49788          10.3.39.241:10250  users:(("kube-apiserver",pid=27807,fd=183))
CLOSE-WAIT 0      0               10.3.33.109:57318          10.3.43.104:10250  users:(("kube-apiserver",pid=27807,fd=211))
root@ip-10-3-33-109:~#
```

> 可以确定：kubelet 和 kube-apiserver 之间似乎存在某种问题（未知），导致 kube-apiserver 侧无法正常关闭 socket ；理论上讲，持续累积会导致随机端口耗尽，可用 fd 数据不足等问题；

- kube-apiserver 8 线程跑在单核 CPU 上，可以看出 CPU 资源不足，steal 较大，r 有积压

```
root@ip-10-3-33-109:~# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                1
On-line CPU(s) list:   0
Thread(s) per core:    1
Core(s) per socket:    1
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 62
Model name:            Intel(R) Xeon(R) CPU E5-2670 v2 @ 2.50GHz
Stepping:              4
CPU MHz:               2500.054
BogoMIPS:              5000.10
Hypervisor vendor:     Xen
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              25600K
NUMA node0 CPU(s):     0
root@ip-10-3-33-109:~#


root@ip-10-3-33-109:~# vmstat 1 60 -w
procs ---------------memory-------------- ---swap-- -----io---- -system-- ------cpu-----
 r  b     swpd     free     buff    cache   si   so    bi    bo   in   cs us sy id wa st
 1  0        0   194884   384088  1892160    0    0    49   495    2    1 14  3 70  1 12
 0  0        0   197968   384088  1892228    0    0     0  1044 2743 6308 14  7 73  2  4
 0  0        0   197728   384088  1892332    0    0     0   716 2313 5654 11  3 80  2  4
 0  0        0   197728   384088  1892380    0    0     0   804 1926 4854  7  2 86  2  3
 0  0        0   197604   384092  1892404    0    0     0   440 1711 5028  7  3 85  1  4
 1  0        0   197604   384092  1892424    0    0     0   460 1801 4904  3  2 93  1  1
 4  0        0   198472   384092  1892440    0    0     0   260 1593 4816  6  3 87  1  3
 0  0        0   198472   384092  1892504    0    0     0   928 2002 5096  6  2 86  2  4
 5  0        0   197092   384092  1892540    0    0     0   516 1798 5233  9  8 75  1  7
 0  0        0   198348   384096  1892612    0    0     0   892 1845 4970  7  8 76  3  6
 0  0        0   198100   384100  1892632    0    0     0   288 1812 4702  4  1 92  1  2
 3  0        0   198100   384100  1892692    0    0     0   732 2076 4941 11  2 79  2  5
 5  0        0   192704   384100  1892812    0    0     0  1044 1924 4372 52  6  1  0 41
 0  0        0   197852   384100  1892864    0    0     0   640 2103 6006 20  4 60  2 14
 0  0        0   197852   384100  1892896    0    0     0   516 1713 5117 14  3 73  1  8
 0  0        0   197728   384100  1892908    0    0     0   236 1427 4144  5  1 89  1  4
 0  0        0   197728   384100  1892920    0    0     0   440 1557 4772  4  1 93  1  1
 0  0        0   197728   384100  1893132    0    0     0   816 1825 5063  7  2 89  2  0
 0  0        0   197612   384100  1893156    0    0     0   396 1544 4859  6  1 90  1  2
 0  0        0   197488   384104  1893232    0    0     0   936 1880 5367  9  3 80  3  5
 0  0        0   197488   384104  1893248    0    0     0   280 1645 4777  5  2 88  1  4
 2  0        0   197396   384104  1893304    0    0     0   776 2033 5512 13  3 77  2  5
 1  0        0   196124   384104  1893416    0    0     0  1160 2679 7279 14  5 72  3  6
 0  0        0   196124   384104  1893468    0    0     0   796 2078 5972  6  2 86  2  3
 6  0        0   196124   384104  1893520    0    0     0   700 2038 5301  6  3 88  2  1
 2  0        0   196124   384104  1893528    0    0     0   176 1667 4899  8  1 85  0  6
 1  0        0   196124   384104  1893548    0    0     0   368 1779 5106  6  1 89  1  2
 2  0        0   196124   384104  1893604    0    0     0  1928 2308 5528 19  2 63  4 12
 3  0        0   196000   384104  1893664    0    0     0   504 2012 5481  6  3 88  1  2
 2  0        0   196000   384104  1893716    0    0     0  1028 2046 5622  6  2 88  3  1
 1  0        0   196000   384104  1893760    0    0     0   440 1845 5563  8  2 85  1  3
 2  1        0   195628   384104  1893800    0    0     0   740 2265 5717  8  4 84  2  2
 1  0        0   195628   384104  1893916    0    0     0  1224 2773 6816 13  3 78  3  2
 2  0        0   195628   384104  1893936    0    0     0   420 1569 4864  3  3 88  1  5
 2  0        0   195504   384104  1894008    0    0     0   724 1925 5496  9  5 81  2  3
 1  0        0   195504   384108  1894016    0    0     0   228 1826 5090  4  3 88  1  4
 4  0        0   195360   384108  1894044    0    0     0   316 1498 4657  3  2 92  0  3
 0  0        0   195380   384108  1894100    0    0     0   912 1870 5074  6  3 83  3  5
 0  0        0   195288   384108  1894152    0    0     0   512 1661 4568  6  3 87  1  3
 1  0        0   195216   384112  1894196    0    0     0   956 1993 5411 11 11 64  3 11
 3  0        0   195248   384116  1894240    0    0     0   368 1697 4360  6  2 91  1  0
 1  0        0   195248   384116  1894288    0    0     0   828 1994 4984  8  0 89  2  1
 1  0        0   195248   384116  1894392    0    0     0   944 2359 5840 18  6 64  2 11
 9  0        0   195000   384116  1894420    0    0     0   676 1667 4833  5  2 90  2  1
 1  0        0   194980   384116  1894492    0    0     0   584 1658 4068  8  1 83  2  6
 1  0        0   195124   384116  1894492    0    0     0   268 1552 3971  3  3 91  0  3
 4  0        0   195000   384116  1894516    0    0     0   244 1461 3972  2  3 93  1  1
 7  0        0   195000   384116  1894564    0    0     0  1000 1876 4606  6  2 86  3  3
 2  0        0   195000   384116  1894624    0    0     0   788 1654 4688  6  2 87  2  3
 3  0        0   194876   384120  1894664    0    0     0   756 2058 5588 10  5 79  2  4
 0  0        0   194876   384120  1894732    0    0     0   572 1807 5152  6  2 88  2  2
13  0        0   192040   384120  1894780    0    0     0   784 2126 5655 13 14 54  1 18
 1  0        0   194596   384120  1894892    0    0     0  1232 2539 5850 18  8 57  3 14
 1  0        0   194628   384120  1894924    0    0     0   504 1530 4355  4  2 91  1  2
 1  0        0   194504   384124  1894992    0    0     0  1352 2121 5397  5  3 86  3  2
 1  0        0   194504   384124  1894996    0    0     0   180 2071 5368  5  3 88  1  3
 1  0        0   194504   384124  1895028    0    0     0   304 1873 4946  5  2 89  1  3
 0  0        0   194380   384124  1895068    0    0     0   548 2003 5396  9  5 79  1  6
 0  0        0   194388   384124  1895120    0    0     0   820 2003 5122  9  1 84  2  4
 0  0        0   194388   384124  1895164    0    0     0   848 2020 5330  8  3 83  4  2
root@ip-10-3-33-109:~#
```
