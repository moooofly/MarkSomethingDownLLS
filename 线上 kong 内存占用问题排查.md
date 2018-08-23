# 线上 kong 内存占用问题排查

## 结论

最终确认 kong 存在内存泄漏问题，升级版本后正常；但线上机器默认放开了透明大页，似乎也有所不妥~

## 基本信息

- cpu

```
root@ip-10-1-2-104:~# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                2
On-line CPU(s) list:   0,1
Thread(s) per core:    2
Core(s) per socket:    1
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 79
Model name:            Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
Stepping:              1
CPU MHz:               2300.048
BogoMIPS:              4600.09
Hypervisor vendor:     Xen
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              46080K
NUMA node0 CPU(s):     0,1
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm fsgsbase bmi1 avx2 smep bmi2 erms invpcid xsaveopt
root@ip-10-1-2-104:~#
```

- memory

```
root@ip-10-1-2-104:~# free -m
              total        used        free      shared  buff/cache   available
Mem:           7982        7372         229          20         380         320
Swap:             0           0           0
root@ip-10-1-2-104:~#
```

- transparent_hugepage （后补）

```
root@ip-10-1-2-104:~# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
root@ip-10-1-2-104:~#

root@ip-10-1-2-104:~# cat /sys/kernel/mm/transparent_hugepage/defrag
[always] madvise never
root@ip-10-1-2-104:~#
```

- 进程运行情况（后补）

```
root@ip-10-1-2-104:~# ps -A -o pid,ppid,user,stime,resident,etime,rss,size,vsize,args | grep nginx
 1147     1 root     Mar26     - 27-22:11:06  8448  4864 215240 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /opt/apps/kong -c /opt/conf/kong/nginx.conf
 1358  1147 nobody   Mar26     - 27-22:11:04 828268 820968 1031344 nginx: worker process
 1359  1147 nobody   Mar26     - 27-22:11:04 625096 619932 830308 nginx: worker process
 1360  1147 nobody   Mar26     - 27-22:11:04 1150308 1144596 1354972 nginx: worker process
 1361  1147 nobody   Mar26     - 27-22:11:04 1099724 1094188 1304564 nginx: worker process
 1362  1147 nobody   Mar26     - 27-22:11:04 796164 790872 1001248 nginx: worker process
 1363  1147 nobody   Mar26     - 27-22:11:04 953296 947736 1158112 nginx: worker process
 1364  1147 nobody   Mar26     - 27-22:11:04 855108 850160 1060536 nginx: worker process
 1365  1147 nobody   Mar26     - 27-22:11:04 844272 838916 1049292 nginx: worker process
 4990  4692 root     03:23     -       00:00  1080   344  12916 grep --color=auto nginx
root@ip-10-1-2-104:~#
```

## 现象

- top 后看到 kong 占用内存比较多

```
root@ip-10-1-2-104:~# top
top - 08:45:34 up 25 days,  3:33,  2 users,  load average: 0.00, 0.11, 0.31
Tasks: 142 total,   1 running, 141 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  0.0 sy,  0.0 ni, 99.3 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  8173692 total,   252252 free,  7365560 used,   555880 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   515700 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1387 root      20   0  457600 204724   4572 S   1.0  2.5 540:45.35 ruby
 1363 nobody    20   0 1144988 940064   5772 S   0.3 11.5  24:59.56 nginx
    1 root      20   0   37660   5192   3468 S   0.0  0.1   0:11.16 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.06 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:01.75 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      20   0       0      0      0 S   0.0  0.0   4:22.44 rcu_sched
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
    ...
```



## 问题排查

- 进程树

```
root@ip-10-1-2-104:~# ps ajxf |grep nginx
    1  1147  1147  1147 ?           -1 Ss       0   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /opt/apps/kong -c /opt/conf/kong/nginx.conf
 1147  1358  1147  1147 ?           -1 S    65534  24:33  \_ nginx: worker process
 1147  1359  1147  1147 ?           -1 S    65534  25:07  \_ nginx: worker process
 1147  1360  1147  1147 ?           -1 S    65534  27:27  \_ nginx: worker process
 1147  1361  1147  1147 ?           -1 S    65534  26:58  \_ nginx: worker process
 1147  1362  1147  1147 ?           -1 S    65534  25:52  \_ nginx: worker process
 1147  1363  1147  1147 ?           -1 S    65534  24:59  \_ nginx: worker process
 1147  1364  1147  1147 ?           -1 S    65534  25:57  \_ nginx: worker process
 1147  1365  1147  1147 ?           -1 S    65534  25:25  \_ nginx: worker process
root@ip-10-1-2-104:~#
```

- 简单查看进程 1147 的内存分布

```
root@ip-10-1-2-104:~# pmap 1147
1147:   nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /opt/apps/kong -c /opt/conf/kong/nginx.conf
0000000000400000   4092K r-x-- nginx
00000000009fe000      4K r---- nginx
00000000009ff000    192K rw--- nginx
0000000000a2f000    152K rw---   [ anon ]
0000000001dee000    628K rw---   [ anon ]
0000000040214000    128K rw---   [ anon ]
0000000040282000    128K rw---   [ anon ]
00000000403d4000     68K rw---   [ anon ]
0000000040428000    236K rw---   [ anon ]
00000000404ad000    128K rw---   [ anon ]
0000000040535000    128K rw---   [ anon ]
0000000040603000    128K rw---   [ anon ]
0000000040a44000    128K rw---   [ anon ]
0000000040af6000    128K rw---   [ anon ]
0000000040c0c000    128K rw---   [ anon ]
0000000040d42000    128K rw---   [ anon ]
0000000040f5d000    128K rw---   [ anon ]
0000000040fb1000    128K rw---   [ anon ]
000000004100c000    128K rw---   [ anon ]
0000000041039000    128K rw---   [ anon ]
00000000410ce000    128K rw---   [ anon ]
0000000041283000    128K rw---   [ anon ]
0000000041304000    128K rw---   [ anon ]
00000000413d6000    128K rw---   [ anon ]
00000000413f8000    128K rw---   [ anon ]
0000000041526000    128K rw---   [ anon ]
000000004180d000    128K rw---   [ anon ]
000000004186d000    128K rw---   [ anon ]
0000000041a21000    128K rw---   [ anon ]
0000000041c3a000    128K rw---   [ anon ]
0000000041ca8000    128K rw---   [ anon ]
0000000041d6c000    128K rw---   [ anon ]
00007efe6eee7000      8K r-x-- lsyslog.so
00007efe6eee9000   2044K ----- lsyslog.so
00007efe6f0e8000      4K r---- lsyslog.so
00007efe6f0e9000      4K rw--- lsyslog.so
00007efe6f0ea000     12K r-x-- lua_pack.so
00007efe6f0ed000   2044K ----- lua_pack.so
00007efe6f2ec000      4K r---- lua_pack.so
00007efe6f2ed000      4K rw--- lua_pack.so
00007efe6f2ee000     92K r-x-- libresolv-2.23.so
00007efe6f305000   2048K ----- libresolv-2.23.so
00007efe6f505000      4K r---- libresolv-2.23.so
00007efe6f506000      4K rw--- libresolv-2.23.so
00007efe6f507000      8K rw---   [ anon ]
00007efe6f509000     20K r-x-- libnss_dns-2.23.so
00007efe6f50e000   2048K ----- libnss_dns-2.23.so
00007efe6f70e000      4K r---- libnss_dns-2.23.so
00007efe6f70f000      4K rw--- libnss_dns-2.23.so
00007efe6f710000   2764K r-x-- _openssl.so
00007efe6f9c3000   2044K ----- _openssl.so
00007efe6fbc2000    132K r---- _openssl.so
00007efe6fbe3000     96K rw--- _openssl.so
00007efe6fbfb000     16K rw---   [ anon ]
00007efe6fbff000     40K r-x-- lpeg.so
00007efe6fc09000   2044K ----- lpeg.so
00007efe6fe08000      4K r---- lpeg.so
00007efe6fe09000      4K rw--- lpeg.so
00007efe6fe0a000      8K r-x-- random.so
00007efe6fe0c000   2044K ----- random.so
00007efe7000b000      4K r---- random.so
00007efe7000c000      4K rw--- random.so
00007efe7000d000     28K r-x-- cjson.so
00007efe70014000   2048K ----- cjson.so
00007efe70214000      4K r---- cjson.so
00007efe70215000      4K rw--- cjson.so
00007efe70216000     16K r-x-- lfs.so
00007efe7021a000   2044K ----- lfs.so
00007efe70419000      4K r---- lfs.so
00007efe7041a000      4K rw--- lfs.so
00007efe7041b000      4K r-x-- lua_system_constants.so
00007efe7041c000   2048K ----- lua_system_constants.so
00007efe7061c000      4K r---- lua_system_constants.so
00007efe7061d000      4K rw--- lua_system_constants.so
00007efe7061e000     56K r-x-- core.so
00007efe7062c000   2044K ----- core.so
00007efe7082b000      4K r---- core.so
00007efe7082c000      4K rw--- core.so
00007efe7082d000   5120K rw-s- zero (deleted)
00007efe70d2d000   5120K rw-s- zero (deleted)
00007efe7122d000   5120K rw-s- zero (deleted)
00007efe7172d000 131072K rw-s- zero (deleted)
00007efe7972d000   5120K rw-s- zero (deleted)
00007efe79c2d000     44K r-x-- libnss_files-2.23.so
00007efe79c38000   2044K ----- libnss_files-2.23.so
00007efe79e37000      4K r---- libnss_files-2.23.so
00007efe79e38000      4K rw--- libnss_files-2.23.so
00007efe79e39000     24K rw---   [ anon ]
00007efe79e3f000     44K r-x-- libnss_nis-2.23.so
00007efe79e4a000   2044K ----- libnss_nis-2.23.so
00007efe7a049000      4K r---- libnss_nis-2.23.so
00007efe7a04a000      4K rw--- libnss_nis-2.23.so
00007efe7a04b000     88K r-x-- libnsl-2.23.so
00007efe7a061000   2044K ----- libnsl-2.23.so
00007efe7a260000      4K r---- libnsl-2.23.so
00007efe7a261000      4K rw--- libnsl-2.23.so
00007efe7a262000      8K rw---   [ anon ]
00007efe7a264000     32K r-x-- libnss_compat-2.23.so
00007efe7a26c000   2044K ----- libnss_compat-2.23.so
00007efe7a46b000      4K r---- libnss_compat-2.23.so
00007efe7a46c000      4K rw--- libnss_compat-2.23.so
00007efe7a46d000     88K r-x-- libgcc_s.so.1
00007efe7a483000   2044K ----- libgcc_s.so.1
00007efe7a682000      4K rw--- libgcc_s.so.1
00007efe7a683000   1792K r-x-- libc-2.23.so
00007efe7a843000   2048K ----- libc-2.23.so
00007efe7aa43000     16K r---- libc-2.23.so
00007efe7aa47000      8K rw--- libc-2.23.so
00007efe7aa49000     16K rw---   [ anon ]
00007efe7aa4d000    100K r-x-- libz.so.1.2.8
00007efe7aa66000   2044K ----- libz.so.1.2.8
00007efe7ac65000      4K r---- libz.so.1.2.8
00007efe7ac66000      4K rw--- libz.so.1.2.8
00007efe7ac67000   1056K r-x-- libm-2.23.so
00007efe7ad6f000   2044K ----- libm-2.23.so
00007efe7af6e000      4K r---- libm-2.23.so
00007efe7af6f000      4K rw--- libm-2.23.so
00007efe7af70000    480K r-x-- libluajit-5.1.so.2.1.0
00007efe7afe8000   2048K ----- libluajit-5.1.so.2.1.0
00007efe7b1e8000      8K r---- libluajit-5.1.so.2.1.0
00007efe7b1ea000      4K rw--- libluajit-5.1.so.2.1.0
00007efe7b1eb000     36K r-x-- libcrypt-2.23.so
00007efe7b1f4000   2044K ----- libcrypt-2.23.so
00007efe7b3f3000      4K r---- libcrypt-2.23.so
00007efe7b3f4000      4K rw--- libcrypt-2.23.so
00007efe7b3f5000    184K rw---   [ anon ]
00007efe7b423000     96K r-x-- libpthread-2.23.so
00007efe7b43b000   2044K ----- libpthread-2.23.so
00007efe7b63a000      4K r---- libpthread-2.23.so
00007efe7b63b000      4K rw--- libpthread-2.23.so
00007efe7b63c000     16K rw---   [ anon ]
00007efe7b640000     12K r-x-- libdl-2.23.so
00007efe7b643000   2044K ----- libdl-2.23.so
00007efe7b842000      4K r---- libdl-2.23.so
00007efe7b843000      4K rw--- libdl-2.23.so
00007efe7b844000    152K r-x-- ld-2.23.so
00007efe7ba4d000     64K rwx--   [ anon ]
00007efe7ba5d000     24K rw---   [ anon ]
00007efe7ba68000      4K rw-s- zero (deleted)
00007efe7ba69000      4K r---- ld-2.23.so
00007efe7ba6a000      4K rw--- ld-2.23.so
00007efe7ba6b000      4K rw---   [ anon ]
00007efea5050000     64K r-x--   [ anon ]
00007ffcc05c1000    132K rw---   [ stack ]
00007ffcc05f8000      8K r----   [ anon ]
00007ffcc05fa000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total           215240K
root@ip-10-1-2-104:~#
```

可疑点：zero (deleted) 对应的数值较大，且存在 (deleted) 这个字符串，比较可疑；

- 检查查看进程 1358 的内存分布

```
root@ip-10-1-2-104:~# pmap 1358
1358:   nginx: worker process
0000000000400000   4092K r-x-- nginx
00000000009fe000      4K r---- nginx
00000000009ff000    192K rw--- nginx
0000000000a2f000    152K rw---   [ anon ]
0000000001dee000    588K rw---   [ anon ]
0000000001e81000   1008K rw---   [ anon ]
0000000040005000    128K rw---   [ anon ]
0000000040041000    512K rw---   [ anon ]
00000000400cd000    640K rw---   [ anon ]
0000000040171000    640K rw---   [ anon ]
0000000040214000    128K rw---   [ anon ]
0000000040234000    256K rw---   [ anon ]
0000000040282000    128K rw---   [ anon ]
00000000402a5000    256K rw---   [ anon ]
00000000402ec000    128K rw---   [ anon ]
0000000040317000    256K rw---   [ anon ]
000000004036c000    128K rw---   [ anon ]
0000000040392000    128K rw---   [ anon ]
00000000403b3000    128K rw---   [ anon ]
00000000403ea000    128K rw---   [ anon ]
0000000040428000    236K rw---   [ anon ]
0000000040463000    256K rw---   [ anon ]
00000000404ad000    128K rw---   [ anon ]
00000000404cd000    128K rw---   [ anon ]
00000000404ef000    256K rw---   [ anon ]
0000000040535000    128K rw---   [ anon ]
000000004056d000    128K rw---   [ anon ]
00000000405a8000    256K rw---   [ anon ]
0000000040603000    128K rw---   [ anon ]
0000000040623000    640K rw---   [ anon ]
00000000406d2000    256K rw---   [ anon ]
0000000040722000    512K rw---   [ anon ]
00000000407be000    128K rw---   [ anon ]
00000000407e3000    128K rw---   [ anon ]
0000000040806000    256K rw---   [ anon ]
0000000040859000    384K rw---   [ anon ]
00000000408c8000    128K rw---   [ anon ]
0000000040900000    256K rw---   [ anon ]
0000000040947000    128K rw---   [ anon ]
0000000040969000    128K rw---   [ anon ]
0000000040991000    256K rw---   [ anon ]
00000000409eb000    256K rw---   [ anon ]
0000000040a44000    128K rw---   [ anon ]
0000000040a64000    384K rw---   [ anon ]
0000000040ac5000    128K rw---   [ anon ]
0000000040af6000    128K rw---   [ anon ]
0000000040b16000    640K rw---   [ anon ]
0000000040bb7000    256K rw---   [ anon ]
0000000040c0c000    128K rw---   [ anon ]
0000000040c2c000    128K rw---   [ anon ]
0000000040c62000    512K rw---   [ anon ]
0000000040cf5000    256K rw---   [ anon ]
0000000040d42000    128K rw---   [ anon ]
0000000040d62000    512K rw---   [ anon ]
0000000040de6000    256K rw---   [ anon ]
0000000040e3a000    640K rw---   [ anon ]
0000000040edf000    256K rw---   [ anon ]
0000000040f23000    128K rw---   [ anon ]
0000000040f5d000    128K rw---   [ anon ]
0000000040f7d000    128K rw---   [ anon ]
0000000040fb1000    128K rw---   [ anon ]
0000000040fe0000    128K rw---   [ anon ]
000000004100c000    128K rw---   [ anon ]
0000000041039000    128K rw---   [ anon ]
0000000041059000    384K rw---   [ anon ]
00000000410ce000    128K rw---   [ anon ]
00000000410ee000    128K rw---   [ anon ]
000000004111e000   1152K rw---   [ anon ]
0000000041240000    256K rw---   [ anon ]
0000000041283000    128K rw---   [ anon ]
00000000412b4000    256K rw---   [ anon ]
0000000041304000    128K rw---   [ anon ]
0000000041324000    128K rw---   [ anon ]
0000000041358000    256K rw---   [ anon ]
000000004139d000    128K rw---   [ anon ]
00000000413d6000    128K rw---   [ anon ]
00000000413f8000    128K rw---   [ anon ]
0000000041418000    384K rw---   [ anon ]
0000000041479000    384K rw---   [ anon ]
00000000414f3000    128K rw---   [ anon ]
0000000041526000    128K rw---   [ anon ]
0000000041546000    256K rw---   [ anon ]
0000000041587000    128K rw---   [ anon ]
00000000415b8000    128K rw---   [ anon ]
00000000415d9000    128K rw---   [ anon ]
000000004160d000    256K rw---   [ anon ]
0000000041667000    512K rw---   [ anon ]
00000000416ff000    128K rw---   [ anon ]
0000000041739000    256K rw---   [ anon ]
0000000041788000    512K rw---   [ anon ]
000000004180d000    128K rw---   [ anon ]
000000004182d000    256K rw---   [ anon ]
000000004186d000    128K rw---   [ anon ]
000000004188d000    128K rw---   [ anon ]
00000000418b0000    512K rw---   [ anon ]
000000004193a000    128K rw---   [ anon ]
000000004195b000    256K rw---   [ anon ]
00000000419ab000    384K rw---   [ anon ]
0000000041a21000    128K rw---   [ anon ]
0000000041a41000    384K rw---   [ anon ]
0000000041ab4000    256K rw---   [ anon ]
0000000041b06000    128K rw---   [ anon ]
0000000041b3b000    640K rw---   [ anon ]
0000000041be8000    128K rw---   [ anon ]
0000000041c0e000    128K rw---   [ anon ]
0000000041c3a000    128K rw---   [ anon ]
0000000041c5a000    256K rw---   [ anon ]
0000000041ca8000    128K rw---   [ anon ]
0000000041cc8000    128K rw---   [ anon ]
0000000041d07000    384K rw---   [ anon ]
0000000041d6c000    128K rw---   [ anon ]
0000000041d8c000    512K rw---   [ anon ]
0000000041e25000    256K rw---   [ anon ]
0000000041e67000    512K rw---   [ anon ]
0000000041ee8000    128K rw---   [ anon ]
0000000041f24000    384K rw---   [ anon ]
0000000041f8c000    128K rw---   [ anon ]
0000000041fb5000  13640K rw---   [ anon ]
0000000042d19000   9216K rw---   [ anon ]
000000004361a000   9984K rw---   [ anon ]
0000000043fdb000   5120K rw---   [ anon ]
00000000444dc000  11136K rw---   [ anon ]
0000000044fbd000  24960K rw---   [ anon ]
000000004681e000  50820K rw---   [ anon ]
00000000499c0000  82176K rw---   [ anon ]
000000004ea01000 168524K rw---   [ anon ]
0000000058e95000 144896K rw---   [ anon ]
0000000061c26000  38912K rw---   [ anon ]
0000000064237000 217604K rw---   [ anon ]
00007efe6e935000   5832K rw---   [ anon ]
00007efe6eee7000      8K r-x-- lsyslog.so
00007efe6eee9000   2044K ----- lsyslog.so
00007efe6f0e8000      4K r---- lsyslog.so
00007efe6f0e9000      4K rw--- lsyslog.so
00007efe6f0ea000     12K r-x-- lua_pack.so
00007efe6f0ed000   2044K ----- lua_pack.so
00007efe6f2ec000      4K r---- lua_pack.so
00007efe6f2ed000      4K rw--- lua_pack.so
00007efe6f2ee000     92K r-x-- libresolv-2.23.so
00007efe6f305000   2048K ----- libresolv-2.23.so
00007efe6f505000      4K r---- libresolv-2.23.so
00007efe6f506000      4K rw--- libresolv-2.23.so
00007efe6f507000      8K rw---   [ anon ]
00007efe6f509000     20K r-x-- libnss_dns-2.23.so
00007efe6f50e000   2048K ----- libnss_dns-2.23.so
00007efe6f70e000      4K r---- libnss_dns-2.23.so
00007efe6f70f000      4K rw--- libnss_dns-2.23.so
00007efe6f710000   2764K r-x-- _openssl.so
00007efe6f9c3000   2044K ----- _openssl.so
00007efe6fbc2000    132K r---- _openssl.so
00007efe6fbe3000     96K rw--- _openssl.so
00007efe6fbfb000     16K rw---   [ anon ]
00007efe6fbff000     40K r-x-- lpeg.so
00007efe6fc09000   2044K ----- lpeg.so
00007efe6fe08000      4K r---- lpeg.so
00007efe6fe09000      4K rw--- lpeg.so
00007efe6fe0a000      8K r-x-- random.so
00007efe6fe0c000   2044K ----- random.so
00007efe7000b000      4K r---- random.so
00007efe7000c000      4K rw--- random.so
00007efe7000d000     28K r-x-- cjson.so
00007efe70014000   2048K ----- cjson.so
00007efe70214000      4K r---- cjson.so
00007efe70215000      4K rw--- cjson.so
00007efe70216000     16K r-x-- lfs.so
00007efe7021a000   2044K ----- lfs.so
00007efe70419000      4K r---- lfs.so
00007efe7041a000      4K rw--- lfs.so
00007efe7041b000      4K r-x-- lua_system_constants.so
00007efe7041c000   2048K ----- lua_system_constants.so
00007efe7061c000      4K r---- lua_system_constants.so
00007efe7061d000      4K rw--- lua_system_constants.so
00007efe7061e000     56K r-x-- core.so
00007efe7062c000   2044K ----- core.so
00007efe7082b000      4K r---- core.so
00007efe7082c000      4K rw--- core.so
00007efe7082d000   5120K rw-s- zero (deleted)
00007efe70d2d000   5120K rw-s- zero (deleted)
00007efe7122d000   5120K rw-s- zero (deleted)
00007efe7172d000 131072K rw-s- zero (deleted)
00007efe7972d000   5120K rw-s- zero (deleted)
00007efe79c2d000     44K r-x-- libnss_files-2.23.so
00007efe79c38000   2044K ----- libnss_files-2.23.so
00007efe79e37000      4K r---- libnss_files-2.23.so
00007efe79e38000      4K rw--- libnss_files-2.23.so
00007efe79e39000     24K rw---   [ anon ]
00007efe79e3f000     44K r-x-- libnss_nis-2.23.so
00007efe79e4a000   2044K ----- libnss_nis-2.23.so
00007efe7a049000      4K r---- libnss_nis-2.23.so
00007efe7a04a000      4K rw--- libnss_nis-2.23.so
00007efe7a04b000     88K r-x-- libnsl-2.23.so
00007efe7a061000   2044K ----- libnsl-2.23.so
00007efe7a260000      4K r---- libnsl-2.23.so
00007efe7a261000      4K rw--- libnsl-2.23.so
00007efe7a262000      8K rw---   [ anon ]
00007efe7a264000     32K r-x-- libnss_compat-2.23.so
00007efe7a26c000   2044K ----- libnss_compat-2.23.so
00007efe7a46b000      4K r---- libnss_compat-2.23.so
00007efe7a46c000      4K rw--- libnss_compat-2.23.so
00007efe7a46d000     88K r-x-- libgcc_s.so.1
00007efe7a483000   2044K ----- libgcc_s.so.1
00007efe7a682000      4K rw--- libgcc_s.so.1
00007efe7a683000   1792K r-x-- libc-2.23.so
00007efe7a843000   2048K ----- libc-2.23.so
00007efe7aa43000     16K r---- libc-2.23.so
00007efe7aa47000      8K rw--- libc-2.23.so
00007efe7aa49000     16K rw---   [ anon ]
00007efe7aa4d000    100K r-x-- libz.so.1.2.8
00007efe7aa66000   2044K ----- libz.so.1.2.8
00007efe7ac65000      4K r---- libz.so.1.2.8
00007efe7ac66000      4K rw--- libz.so.1.2.8
00007efe7ac67000   1056K r-x-- libm-2.23.so
00007efe7ad6f000   2044K ----- libm-2.23.so
00007efe7af6e000      4K r---- libm-2.23.so
00007efe7af6f000      4K rw--- libm-2.23.so
00007efe7af70000    480K r-x-- libluajit-5.1.so.2.1.0
00007efe7afe8000   2048K ----- libluajit-5.1.so.2.1.0
00007efe7b1e8000      8K r---- libluajit-5.1.so.2.1.0
00007efe7b1ea000      4K rw--- libluajit-5.1.so.2.1.0
00007efe7b1eb000     36K r-x-- libcrypt-2.23.so
00007efe7b1f4000   2044K ----- libcrypt-2.23.so
00007efe7b3f3000      4K r---- libcrypt-2.23.so
00007efe7b3f4000      4K rw--- libcrypt-2.23.so
00007efe7b3f5000    184K rw---   [ anon ]
00007efe7b423000     96K r-x-- libpthread-2.23.so
00007efe7b43b000   2044K ----- libpthread-2.23.so
00007efe7b63a000      4K r---- libpthread-2.23.so
00007efe7b63b000      4K rw--- libpthread-2.23.so
00007efe7b63c000     16K rw---   [ anon ]
00007efe7b640000     12K r-x-- libdl-2.23.so
00007efe7b643000   2044K ----- libdl-2.23.so
00007efe7b842000      4K r---- libdl-2.23.so
00007efe7b843000      4K rw--- libdl-2.23.so
00007efe7b844000    152K r-x-- ld-2.23.so
00007efe7b8cc000   1540K rw---   [ anon ]
00007efe7ba4d000     64K rwx--   [ anon ]
00007efe7ba5d000     24K rw---   [ anon ]
00007efe7ba68000      4K rw-s- zero (deleted)
00007efe7ba69000      4K r---- ld-2.23.so
00007efe7ba6a000      4K rw--- ld-2.23.so
00007efe7ba6b000      4K rw---   [ anon ]
00007efea4fe0000    448K r-x--   [ anon ]
00007efea5050000     64K r-x--   [ anon ]
00007ffcc05c1000    132K rw---   [ stack ]
00007ffcc05f8000      8K r----   [ anon ]
00007ffcc05fa000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total          1025012K
root@ip-10-1-2-104:~#
```

- 对比 lsof 的输出

```
root@ip-10-1-2-104:~# lsof -p 1147
COMMAND  PID USER   FD   TYPE             DEVICE  SIZE/OFF   NODE NAME
nginx   1147 root  cwd    DIR              202,1      4096      2 /
nginx   1147 root  rtd    DIR              202,1      4096      2 /
nginx   1147 root  txt    REG              202,1  17054480  19363 /usr/local/openresty/nginx/sbin/nginx
nginx   1147 root  mem    REG              202,1     12872  20192 /usr/local/lib/lua/5.1/lsyslog.so
nginx   1147 root  mem    REG              202,1     17232  20191 /usr/local/lib/lua/5.1/lua_pack.so
nginx   1147 root  mem    REG              202,1    101200 401664 /lib/x86_64-linux-gnu/libresolv-2.23.so
nginx   1147 root  mem    REG              202,1     27000 401670 /lib/x86_64-linux-gnu/libnss_dns-2.23.so
nginx   1147 root  mem    REG              202,1   3372536  20194 /usr/local/lib/lua/5.1/_openssl.so
nginx   1147 root  mem    REG              202,1     52064  20195 /usr/local/lib/lua/5.1/lpeg.so
nginx   1147 root  mem    REG              202,1     13424  20193 /usr/local/lib/lua/5.1/random.so
nginx   1147 root  mem    REG              202,1    129648  19274 /usr/local/openresty/lualib/cjson.so
nginx   1147 root  mem    REG              202,1     24888  20190 /usr/local/lib/lua/5.1/lfs.so
nginx   1147 root  mem    REG              202,1     12936  20189 /usr/local/lib/lua/5.1/lua_system_constants.so
nginx   1147 root  mem    REG              202,1     75992  20188 /usr/local/lib/lua/5.1/socket/core.so
nginx   1147 root  DEL    REG                0,5            15856 /dev/zero
nginx   1147 root  DEL    REG                0,5            15855 /dev/zero
nginx   1147 root  DEL    REG                0,5            15854 /dev/zero
nginx   1147 root  DEL    REG                0,5            15853 /dev/zero
nginx   1147 root  DEL    REG                0,5            15852 /dev/zero
nginx   1147 root  mem    REG              202,1     47600 401673 /lib/x86_64-linux-gnu/libnss_files-2.23.so
nginx   1147 root  mem    REG              202,1     47648 401677 /lib/x86_64-linux-gnu/libnss_nis-2.23.so
nginx   1147 root  mem    REG              202,1     93128 401657 /lib/x86_64-linux-gnu/libnsl-2.23.so
nginx   1147 root  mem    REG              202,1     35688 401668 /lib/x86_64-linux-gnu/libnss_compat-2.23.so
nginx   1147 root  mem    REG              202,1     89696 396885 /lib/x86_64-linux-gnu/libgcc_s.so.1
nginx   1147 root  mem    REG              202,1   1868984 401660 /lib/x86_64-linux-gnu/libc-2.23.so
nginx   1147 root  mem    REG              202,1    104824 396956 /lib/x86_64-linux-gnu/libz.so.1.2.8
nginx   1147 root  mem    REG              202,1   1088952 401656 /lib/x86_64-linux-gnu/libm-2.23.so
nginx   1147 root  mem    REG              202,1   2941184  19332 /usr/local/openresty/luajit/lib/libluajit-5.1.so.2.1.0
nginx   1147 root  mem    REG              202,1     39224 401679 /lib/x86_64-linux-gnu/libcrypt-2.23.so
nginx   1147 root  mem    REG              202,1    138696 401659 /lib/x86_64-linux-gnu/libpthread-2.23.so
nginx   1147 root  mem    REG              202,1     14608 401662 /lib/x86_64-linux-gnu/libdl-2.23.so
nginx   1147 root  mem    REG              202,1    162632 401658 /lib/x86_64-linux-gnu/ld-2.23.so
nginx   1147 root  DEL    REG                0,5            15131 /dev/zero
nginx   1147 root    0r   CHR                1,3       0t0      6 /dev/null
nginx   1147 root    1u  unix 0xffff880204ef7400       0t0  13512 type=STREAM
nginx   1147 root    2w   REG              202,1    979526  22360 /opt/logs/kong/error.log
nginx   1147 root    3u  unix 0xffff880037442800       0t0  15132 type=STREAM
nginx   1147 root    4w   REG              202,1 346670325  20630 /opt/logs/kong/access.log
nginx   1147 root    5w   REG              202,1 289370983  22368 /opt/logs/kong/upstream.log
nginx   1147 root    6w   REG              202,1    219724  21855 /opt/logs/kong/admin_access.log
nginx   1147 root    7w   REG              202,1         0  17688 /opt/logs/kong/admin_error.log
nginx   1147 root    9u  IPv4              15129       0t0    TCP *:http (LISTEN)
nginx   1147 root   10u  IPv4              15130       0t0    TCP *:8001 (LISTEN)
nginx   1147 root   11u  unix 0xffff880037443000       0t0  15133 type=STREAM
nginx   1147 root   12u  unix 0xffff880037445c00       0t0  15134 type=STREAM
nginx   1147 root   13u  unix 0xffff880037445400       0t0  15135 type=STREAM
nginx   1147 root   14u  unix 0xffff880037442c00       0t0  15136 type=STREAM
nginx   1147 root   15u  unix 0xffff880037446c00       0t0  15137 type=STREAM
nginx   1147 root   16u  unix 0xffff880037443800       0t0  15138 type=STREAM
nginx   1147 root   17u  unix 0xffff880037441800       0t0  15139 type=STREAM
nginx   1147 root   18u  unix 0xffff880037456000       0t0  15140 type=STREAM
nginx   1147 root   19u  unix 0xffff880037452800       0t0  15141 type=STREAM
nginx   1147 root   20u  unix 0xffff880037455c00       0t0  15142 type=STREAM
nginx   1147 root   21u  unix 0xffff8800ea7bf400       0t0  15143 type=STREAM
nginx   1147 root   22u  unix 0xffff880204ef0400       0t0  15144 type=STREAM
nginx   1147 root   23u  unix 0xffff880204ef3000       0t0  15145 type=STREAM
nginx   1147 root   24u  unix 0xffff880037486c00       0t0  15146 type=STREAM
nginx   1147 root   25u  unix 0xffff880037482400       0t0  15147 type=STREAM
nginx   1147 root   26w   REG              202,1    979526  22360 /opt/logs/kong/error.log
root@ip-10-1-2-104:~#
```

可疑点：依旧是 /dev/zero 的 FD 项被标识为 DEL ，值得排查；

- 针对 nginx (kong) 进程树中每一项查看总体内存占用

```
root@ip-10-1-2-104:~# pmap 1147 | tail -n 1
 total           215240K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1358 | tail -n 1
 total          1025012K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1359 | tail -n 1
 total           821092K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1360 | tail -n 1
 total          1338040K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1361 | tail -n 1
 total          1289624K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1362 | tail -n 1
 total           989140K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1363 | tail -n 1
 total          1144988K
root@ip-10-1-2-104:~# pmap 1364 | tail -n 1
 total          1053812K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap 1365 | tail -n 1
 total          1043848K
root@ip-10-1-2-104:~#
```

- 重点查看进程 1360 的内存使用情况（从上面得知其内存占用量最大）

```
root@ip-10-1-2-104:~# pmap 1360
1360:   nginx: worker process
0000000000400000   4092K r-x-- nginx
00000000009fe000      4K r---- nginx
00000000009ff000    192K rw--- nginx
0000000000a2f000    152K rw---   [ anon ]
0000000001dee000    628K rw---   [ anon ]
0000000001e8b000    852K rw---   [ anon ]
00000000129bf000  26496K rw---   [ anon ]
000000003d147000  47872K rw---   [ anon ]
0000000040015000    128K rw---   [ anon ]
000000004003e000    128K rw---   [ anon ]
0000000040074000    128K rw---   [ anon ]
00000000400a6000    640K rw---   [ anon ]
000000004015e000    256K rw---   [ anon ]
00000000401ab000    128K rw---   [ anon ]
00000000401e6000    128K rw---   [ anon ]
0000000040214000    128K rw---   [ anon ]
0000000040234000    256K rw---   [ anon ]
0000000040282000    128K rw---   [ anon ]
00000000402a2000    128K rw---   [ anon ]
00000000402d1000    384K rw---   [ anon ]
0000000040333000    256K rw---   [ anon ]
0000000040383000    128K rw---   [ anon ]
00000000403ad000    128K rw---   [ anon ]
00000000403d6000    256K rw---   [ anon ]
0000000040428000    236K rw---   [ anon ]
0000000040463000    256K rw---   [ anon ]
00000000404ad000    128K rw---   [ anon ]
00000000404cd000    128K rw---   [ anon ]
0000000040504000    128K rw---   [ anon ]
0000000040535000    128K rw---   [ anon ]
0000000040555000    128K rw---   [ anon ]
000000004058a000    128K rw---   [ anon ]
00000000405b9000    128K rw---   [ anon ]
00000000405de000    128K rw---   [ anon ]
0000000040603000    128K rw---   [ anon ]
0000000040623000    128K rw---   [ anon ]
0000000040657000    128K rw---   [ anon ]
000000004067c000    512K rw---   [ anon ]
0000000040702000    256K rw---   [ anon ]
0000000040746000    640K rw---   [ anon ]
00000000407f2000    256K rw---   [ anon ]
000000004084a000    896K rw---   [ anon ]
0000000040935000    512K rw---   [ anon ]
00000000409c3000    256K rw---   [ anon ]
0000000040a14000    128K rw---   [ anon ]
0000000040a44000    128K rw---   [ anon ]
0000000040a64000    128K rw---   [ anon ]
0000000040a8c000    384K rw---   [ anon ]
0000000040af6000    128K rw---   [ anon ]
0000000040b16000    256K rw---   [ anon ]
0000000040b64000    128K rw---   [ anon ]
0000000040b8b000    512K rw---   [ anon ]
0000000040c0c000    128K rw---   [ anon ]
0000000040c2c000    640K rw---   [ anon ]
0000000040cd2000    128K rw---   [ anon ]
0000000040cf9000    256K rw---   [ anon ]
0000000040d42000    128K rw---   [ anon ]
0000000040d62000    384K rw---   [ anon ]
0000000040dcf000    384K rw---   [ anon ]
0000000040e4d000    768K rw---   [ anon ]
0000000040f0f000    256K rw---   [ anon ]
0000000040f5d000    128K rw---   [ anon ]
0000000040f88000    128K rw---   [ anon ]
0000000040fb1000    128K rw---   [ anon ]
0000000040fd1000    128K rw---   [ anon ]
000000004100c000    128K rw---   [ anon ]
0000000041039000    128K rw---   [ anon ]
0000000041059000    128K rw---   [ anon ]
0000000041090000    128K rw---   [ anon ]
00000000410ce000    128K rw---   [ anon ]
00000000410ee000    768K rw---   [ anon ]
00000000411b1000    256K rw---   [ anon ]
00000000411fa000    128K rw---   [ anon ]
0000000041220000    384K rw---   [ anon ]
0000000041283000    128K rw---   [ anon ]
00000000412a3000    384K rw---   [ anon ]
0000000041304000    128K rw---   [ anon ]
0000000041324000    384K rw---   [ anon ]
000000004138c000    256K rw---   [ anon ]
00000000413d6000    128K rw---   [ anon ]
00000000413f8000    128K rw---   [ anon ]
0000000041418000    384K rw---   [ anon ]
0000000041484000    256K rw---   [ anon ]
00000000414cb000    256K rw---   [ anon ]
0000000041526000    128K rw---   [ anon ]
0000000041546000    256K rw---   [ anon ]
000000004159a000    512K rw---   [ anon ]
000000004161a000    128K rw---   [ anon ]
0000000041648000    128K rw---   [ anon ]
0000000041675000    128K rw---   [ anon ]
000000004169b000    256K rw---   [ anon ]
00000000416e3000    256K rw---   [ anon ]
0000000041730000    768K rw---   [ anon ]
000000004180d000    128K rw---   [ anon ]
000000004183c000    128K rw---   [ anon ]
000000004186d000    128K rw---   [ anon ]
000000004188d000    256K rw---   [ anon ]
00000000418d8000    128K rw---   [ anon ]
00000000418f9000    512K rw---   [ anon ]
000000004197d000    256K rw---   [ anon ]
00000000419c5000    256K rw---   [ anon ]
0000000041a21000    128K rw---   [ anon ]
0000000041a41000    128K rw---   [ anon ]
0000000041a7c000    256K rw---   [ anon ]
0000000041acd000    256K rw---   [ anon ]
0000000041b14000    384K rw---   [ anon ]
0000000041b85000    128K rw---   [ anon ]
0000000041baf000    128K rw---   [ anon ]
0000000041bd2000    256K rw---   [ anon ]
0000000041c19000    128K rw---   [ anon ]
0000000041c3a000    128K rw---   [ anon ]
0000000041c5a000    256K rw---   [ anon ]
0000000041ca8000    128K rw---   [ anon ]
0000000041cc8000    256K rw---   [ anon ]
0000000041d10000    256K rw---   [ anon ]
0000000041d6c000    128K rw---   [ anon ]
0000000041d8c000  15560K rw---   [ anon ]
0000000042cd0000   6656K rw---   [ anon ]
0000000043351000  14720K rw---   [ anon ]
00000000441b2000   2816K rw---   [ anon ]
0000000044473000  12160K rw---   [ anon ]
0000000045054000  23936K rw---   [ anon ]
00000000467b5000  42112K rw---   [ anon ]
00000000490d6000  90244K rw---   [ anon ]
000000004e8f8000 158924K rw---   [ anon ]
000000005842c000 477700K rw---   [ anon ]
00000000756be000   4224K rw---   [ anon ]
0000000075adf000  63616K rw---   [ anon ]
0000000079910000 105344K rw---   [ anon ]
00007efe585b0000    320K r-x--   [ anon ]
00007efe6e935000   5832K rw---   [ anon ]
00007efe6eee7000      8K r-x-- lsyslog.so
00007efe6eee9000   2044K ----- lsyslog.so
00007efe6f0e8000      4K r---- lsyslog.so
00007efe6f0e9000      4K rw--- lsyslog.so
00007efe6f0ea000     12K r-x-- lua_pack.so
00007efe6f0ed000   2044K ----- lua_pack.so
00007efe6f2ec000      4K r---- lua_pack.so
00007efe6f2ed000      4K rw--- lua_pack.so
00007efe6f2ee000     92K r-x-- libresolv-2.23.so
00007efe6f305000   2048K ----- libresolv-2.23.so
00007efe6f505000      4K r---- libresolv-2.23.so
00007efe6f506000      4K rw--- libresolv-2.23.so
00007efe6f507000      8K rw---   [ anon ]
00007efe6f509000     20K r-x-- libnss_dns-2.23.so
00007efe6f50e000   2048K ----- libnss_dns-2.23.so
00007efe6f70e000      4K r---- libnss_dns-2.23.so
00007efe6f70f000      4K rw--- libnss_dns-2.23.so
00007efe6f710000   2764K r-x-- _openssl.so
00007efe6f9c3000   2044K ----- _openssl.so
00007efe6fbc2000    132K r---- _openssl.so
00007efe6fbe3000     96K rw--- _openssl.so
00007efe6fbfb000     16K rw---   [ anon ]
00007efe6fbff000     40K r-x-- lpeg.so
00007efe6fc09000   2044K ----- lpeg.so
00007efe6fe08000      4K r---- lpeg.so
00007efe6fe09000      4K rw--- lpeg.so
00007efe6fe0a000      8K r-x-- random.so
00007efe6fe0c000   2044K ----- random.so
00007efe7000b000      4K r---- random.so
00007efe7000c000      4K rw--- random.so
00007efe7000d000     28K r-x-- cjson.so
00007efe70014000   2048K ----- cjson.so
00007efe70214000      4K r---- cjson.so
00007efe70215000      4K rw--- cjson.so
00007efe70216000     16K r-x-- lfs.so
00007efe7021a000   2044K ----- lfs.so
00007efe70419000      4K r---- lfs.so
00007efe7041a000      4K rw--- lfs.so
00007efe7041b000      4K r-x-- lua_system_constants.so
00007efe7041c000   2048K ----- lua_system_constants.so
00007efe7061c000      4K r---- lua_system_constants.so
00007efe7061d000      4K rw--- lua_system_constants.so
00007efe7061e000     56K r-x-- core.so
00007efe7062c000   2044K ----- core.so
00007efe7082b000      4K r---- core.so
00007efe7082c000      4K rw--- core.so
00007efe7082d000   5120K rw-s- zero (deleted)
00007efe70d2d000   5120K rw-s- zero (deleted)
00007efe7122d000   5120K rw-s- zero (deleted)
00007efe7172d000 131072K rw-s- zero (deleted)
00007efe7972d000   5120K rw-s- zero (deleted)
00007efe79c2d000     44K r-x-- libnss_files-2.23.so
00007efe79c38000   2044K ----- libnss_files-2.23.so
00007efe79e37000      4K r---- libnss_files-2.23.so
00007efe79e38000      4K rw--- libnss_files-2.23.so
00007efe79e39000     24K rw---   [ anon ]
00007efe79e3f000     44K r-x-- libnss_nis-2.23.so
00007efe79e4a000   2044K ----- libnss_nis-2.23.so
00007efe7a049000      4K r---- libnss_nis-2.23.so
00007efe7a04a000      4K rw--- libnss_nis-2.23.so
00007efe7a04b000     88K r-x-- libnsl-2.23.so
00007efe7a061000   2044K ----- libnsl-2.23.so
00007efe7a260000      4K r---- libnsl-2.23.so
00007efe7a261000      4K rw--- libnsl-2.23.so
00007efe7a262000      8K rw---   [ anon ]
00007efe7a264000     32K r-x-- libnss_compat-2.23.so
00007efe7a26c000   2044K ----- libnss_compat-2.23.so
00007efe7a46b000      4K r---- libnss_compat-2.23.so
00007efe7a46c000      4K rw--- libnss_compat-2.23.so
00007efe7a46d000     88K r-x-- libgcc_s.so.1
00007efe7a483000   2044K ----- libgcc_s.so.1
00007efe7a682000      4K rw--- libgcc_s.so.1
00007efe7a683000   1792K r-x-- libc-2.23.so
00007efe7a843000   2048K ----- libc-2.23.so
00007efe7aa43000     16K r---- libc-2.23.so
00007efe7aa47000      8K rw--- libc-2.23.so
00007efe7aa49000     16K rw---   [ anon ]
00007efe7aa4d000    100K r-x-- libz.so.1.2.8
00007efe7aa66000   2044K ----- libz.so.1.2.8
00007efe7ac65000      4K r---- libz.so.1.2.8
00007efe7ac66000      4K rw--- libz.so.1.2.8
00007efe7ac67000   1056K r-x-- libm-2.23.so
00007efe7ad6f000   2044K ----- libm-2.23.so
00007efe7af6e000      4K r---- libm-2.23.so
00007efe7af6f000      4K rw--- libm-2.23.so
00007efe7af70000    480K r-x-- libluajit-5.1.so.2.1.0
00007efe7afe8000   2048K ----- libluajit-5.1.so.2.1.0
00007efe7b1e8000      8K r---- libluajit-5.1.so.2.1.0
00007efe7b1ea000      4K rw--- libluajit-5.1.so.2.1.0
00007efe7b1eb000     36K r-x-- libcrypt-2.23.so
00007efe7b1f4000   2044K ----- libcrypt-2.23.so
00007efe7b3f3000      4K r---- libcrypt-2.23.so
00007efe7b3f4000      4K rw--- libcrypt-2.23.so
00007efe7b3f5000    184K rw---   [ anon ]
00007efe7b423000     96K r-x-- libpthread-2.23.so
00007efe7b43b000   2044K ----- libpthread-2.23.so
00007efe7b63a000      4K r---- libpthread-2.23.so
00007efe7b63b000      4K rw--- libpthread-2.23.so
00007efe7b63c000     16K rw---   [ anon ]
00007efe7b640000     12K r-x-- libdl-2.23.so
00007efe7b643000   2044K ----- libdl-2.23.so
00007efe7b842000      4K r---- libdl-2.23.so
00007efe7b843000      4K rw--- libdl-2.23.so
00007efe7b844000    152K r-x-- ld-2.23.so
00007efe7b8cc000   1540K rw---   [ anon ]
00007efe7ba4d000     64K rwx--   [ anon ]
00007efe7ba5d000     24K rw---   [ anon ]
00007efe7ba68000      4K rw-s- zero (deleted)
00007efe7ba69000      4K r---- ld-2.23.so
00007efe7ba6a000      4K rw--- ld-2.23.so
00007efe7ba6b000      4K rw---   [ anon ]
00007ffcc05c1000    132K rw---   [ stack ]
00007ffcc05f8000      8K r----   [ anon ]
00007ffcc05fa000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total          1338432K
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# lsof -p 1360
COMMAND  PID   USER   FD      TYPE             DEVICE  SIZE/OFF   NODE NAME
nginx   1360 nobody  cwd       DIR              202,1      4096      2 /
nginx   1360 nobody  rtd       DIR              202,1      4096      2 /
nginx   1360 nobody  txt       REG              202,1  17054480  19363 /usr/local/openresty/nginx/sbin/nginx
nginx   1360 nobody  mem       REG              202,1     12872  20192 /usr/local/lib/lua/5.1/lsyslog.so
nginx   1360 nobody  mem       REG              202,1     17232  20191 /usr/local/lib/lua/5.1/lua_pack.so
nginx   1360 nobody  mem       REG              202,1    101200 401664 /lib/x86_64-linux-gnu/libresolv-2.23.so
nginx   1360 nobody  mem       REG              202,1     27000 401670 /lib/x86_64-linux-gnu/libnss_dns-2.23.so
nginx   1360 nobody  mem       REG              202,1   3372536  20194 /usr/local/lib/lua/5.1/_openssl.so
nginx   1360 nobody  mem       REG              202,1     52064  20195 /usr/local/lib/lua/5.1/lpeg.so
nginx   1360 nobody  mem       REG              202,1     13424  20193 /usr/local/lib/lua/5.1/random.so
nginx   1360 nobody  mem       REG              202,1    129648  19274 /usr/local/openresty/lualib/cjson.so
nginx   1360 nobody  mem       REG              202,1     24888  20190 /usr/local/lib/lua/5.1/lfs.so
nginx   1360 nobody  mem       REG              202,1     12936  20189 /usr/local/lib/lua/5.1/lua_system_constants.so
nginx   1360 nobody  mem       REG              202,1     75992  20188 /usr/local/lib/lua/5.1/socket/core.so
nginx   1360 nobody  DEL       REG                0,5            15856 /dev/zero
nginx   1360 nobody  DEL       REG                0,5            15855 /dev/zero
nginx   1360 nobody  DEL       REG                0,5            15854 /dev/zero
nginx   1360 nobody  DEL       REG                0,5            15853 /dev/zero
nginx   1360 nobody  DEL       REG                0,5            15852 /dev/zero
nginx   1360 nobody  mem       REG              202,1     47600 401673 /lib/x86_64-linux-gnu/libnss_files-2.23.so
nginx   1360 nobody  mem       REG              202,1     47648 401677 /lib/x86_64-linux-gnu/libnss_nis-2.23.so
nginx   1360 nobody  mem       REG              202,1     93128 401657 /lib/x86_64-linux-gnu/libnsl-2.23.so
nginx   1360 nobody  mem       REG              202,1     35688 401668 /lib/x86_64-linux-gnu/libnss_compat-2.23.so
nginx   1360 nobody  mem       REG              202,1     89696 396885 /lib/x86_64-linux-gnu/libgcc_s.so.1
nginx   1360 nobody  mem       REG              202,1   1868984 401660 /lib/x86_64-linux-gnu/libc-2.23.so
nginx   1360 nobody  mem       REG              202,1    104824 396956 /lib/x86_64-linux-gnu/libz.so.1.2.8
nginx   1360 nobody  mem       REG              202,1   1088952 401656 /lib/x86_64-linux-gnu/libm-2.23.so
nginx   1360 nobody  mem       REG              202,1   2941184  19332 /usr/local/openresty/luajit/lib/libluajit-5.1.so.2.1.0
nginx   1360 nobody  mem       REG              202,1     39224 401679 /lib/x86_64-linux-gnu/libcrypt-2.23.so
nginx   1360 nobody  mem       REG              202,1    138696 401659 /lib/x86_64-linux-gnu/libpthread-2.23.so
nginx   1360 nobody  mem       REG              202,1     14608 401662 /lib/x86_64-linux-gnu/libdl-2.23.so
nginx   1360 nobody  mem       REG              202,1    162632 401658 /lib/x86_64-linux-gnu/ld-2.23.so
nginx   1360 nobody  DEL       REG                0,5            15131 /dev/zero
nginx   1360 nobody    0r      CHR                1,3       0t0      6 /dev/null
nginx   1360 nobody    1u     unix 0xffff880204ef7400       0t0  13512 type=STREAM
nginx   1360 nobody    2w      REG              202,1    983458  22360 /opt/logs/kong/error.log
nginx   1360 nobody    3u     unix 0xffff880037442800       0t0  15132 type=STREAM
nginx   1360 nobody    4u     IPv4            3777335       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:55562->ip-10-1-6-42.cn-north-1.compute.internal:postgresql (ESTABLISHED)
nginx   1360 nobody    5w      REG              202,1 347087110  20630 /opt/logs/kong/access.log
nginx   1360 nobody    6w      REG              202,1    983458  22360 /opt/logs/kong/error.log
nginx   1360 nobody    7w      REG              202,1 289647463  22368 /opt/logs/kong/upstream.log
nginx   1360 nobody    8u     IPv4            3773321       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:58602->ip-10-1-1-189.cn-north-1.compute.internal:9093 (ESTABLISHED)
nginx   1360 nobody    9u     IPv4              15129       0t0    TCP *:http (LISTEN)
nginx   1360 nobody   10u     IPv4              15130       0t0    TCP *:8001 (LISTEN)
nginx   1360 nobody   11u     unix 0xffff880037443800       0t0  15138 type=STREAM
nginx   1360 nobody   12u     unix 0xffff880037445c00       0t0  15134 type=STREAM
nginx   1360 nobody   13u     unix 0xffff880037456000       0t0  15140 type=STREAM
nginx   1360 nobody   14u     unix 0xffff880037455c00       0t0  15142 type=STREAM
nginx   1360 nobody   15u     unix 0xffff880037446c00       0t0  15137 type=STREAM
nginx   1360 nobody   16u  a_inode               0,11         0   8105 [eventpoll]
nginx   1360 nobody   17u  a_inode               0,11         0   8105 [eventfd]
nginx   1360 nobody   18u     unix 0xffff880204ef0400       0t0  15144 type=STREAM
nginx   1360 nobody   19u     unix 0xffff880037486c00       0t0  15146 type=STREAM
nginx   1360 nobody   20w      REG              202,1    236704  21855 /opt/logs/kong/admin_access.log
nginx   1360 nobody   21w      REG              202,1         0  17688 /opt/logs/kong/admin_error.log
nginx   1360 nobody   23u     IPv4            3779751       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:38526->ip-10-128-39-245.cn-north-1.compute.internal:9090 (ESTABLISHED)
nginx   1360 nobody   24u     IPv4            3770729       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:36460->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   25u     IPv4            3777394       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34708->ip-10-5-50-224.cn-north-1.compute.internal:32643 (ESTABLISHED)
nginx   1360 nobody   27u     IPv4            3778521       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:38496->ip-10-1-1-189.cn-north-1.compute.internal:9091 (ESTABLISHED)
nginx   1360 nobody   30u     IPv4            3770735       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:36464->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   32u     IPv4            3779758       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:49760->ip-10-4-53-234.cn-north-1.compute.internal:30875 (ESTABLISHED)
nginx   1360 nobody   34u     IPv4            3778930       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:37126->ip-10-1-1-189.cn-north-1.compute.internal:9090 (ESTABLISHED)
nginx   1360 nobody   35u     IPv4            3774748       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:52776->ip-10-3-42-103.cn-north-1.compute.internal:30744 (ESTABLISHED)
nginx   1360 nobody   36u     IPv4            3779764       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:33804->ip-10-3-51-209.cn-north-1.compute.internal:31218 (ESTABLISHED)
nginx   1360 nobody   38u     IPv4            3761508       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:58776->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   39u     IPv4            3773309       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:42052->ip-10-128-39-245.cn-north-1.compute.internal:9093 (ESTABLISHED)
nginx   1360 nobody   40u     IPv4            3779028       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:43562->ip-10-3-51-209.cn-north-1.compute.internal:32724 (ESTABLISHED)
nginx   1360 nobody   42u     IPv4            3765319       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:32792->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   43u     IPv4            3779770       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:51752->ip-10-128-39-245.cn-north-1.compute.internal:9091 (ESTABLISHED)
nginx   1360 nobody   46u     IPv4            3780649       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:8001->ip-10-1-4-122.cn-north-1.compute.internal:38349 (ESTABLISHED)
nginx   1360 nobody   47u     IPv4            3778948       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:52890->ip-10-4-57-188.cn-north-1.compute.internal:30875 (ESTABLISHED)
nginx   1360 nobody   48u     IPv4            3765321       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:32794->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   49u     IPv4            3768949       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:35554->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   50u     IPv4            3780675       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:8001->ip-10-1-15-21.cn-north-1.compute.internal:54667 (ESTABLISHED)
nginx   1360 nobody   51u     IPv4            3755929       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:55922->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   52u     IPv4            3778055       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:39612->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   53u     IPv4            3778056       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:39614->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   55u     IPv4            3778058       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:39616->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   58u     IPv4            3767648       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34904->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   59u     IPv4            3778061       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:39618->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   60u     IPv4            3778062       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:39620->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   65u     IPv4            3778928       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:43774->ip-10-3-42-103.cn-north-1.compute.internal:32724 (ESTABLISHED)
nginx   1360 nobody   68u     IPv4            3778939       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:57194->ip-10-3-51-209.cn-north-1.compute.internal:30744 (ESTABLISHED)
nginx   1360 nobody   76u     IPv4            3767644       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34902->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   82u     IPv4            3773104       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:38276->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   91u     IPv4            3768446       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34918->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   95u     IPv4            3768450       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34922->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
nginx   1360 nobody   99u     IPv4            3767661       0t0    TCP ip-10-1-2-104.cn-north-1.compute.internal:34926->ip-10-1-1-189.cn-north-1.compute.internal:3000 (ESTABLISHED)
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~#
root@ip-10-1-2-104:~# pmap -x 1360
1360:   nginx: worker process
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000    4092     616       0 r-x-- nginx
0000000000400000       0       0       0 r-x-- nginx
00000000009fe000       4       4       4 r---- nginx
00000000009fe000       0       0       0 r---- nginx
00000000009ff000     192     116     112 rw--- nginx
00000000009ff000       0       0       0 rw--- nginx
0000000000a2f000     152      40      40 rw---   [ anon ]
0000000000a2f000       0       0       0 rw---   [ anon ]
0000000001dee000     628     624     624 rw---   [ anon ]
0000000001dee000       0       0       0 rw---   [ anon ]
0000000001e8b000     852     420     420 rw---   [ anon ]
0000000001e8b000       0       0       0 rw---   [ anon ]
00000000129bf000   26496   26444   26444 rw---   [ anon ]
00000000129bf000       0       0       0 rw---   [ anon ]
000000003d147000   47872   47872   47872 rw---   [ anon ]
000000003d147000       0       0       0 rw---   [ anon ]
0000000040015000     128     128     128 rw---   [ anon ]
0000000040015000       0       0       0 rw---   [ anon ]
000000004003e000     128     128     128 rw---   [ anon ]
000000004003e000       0       0       0 rw---   [ anon ]
0000000040074000     128     128     128 rw---   [ anon ]
0000000040074000       0       0       0 rw---   [ anon ]
00000000400a6000     640     640     640 rw---   [ anon ]
00000000400a6000       0       0       0 rw---   [ anon ]
000000004015e000     256     256     256 rw---   [ anon ]
000000004015e000       0       0       0 rw---   [ anon ]
00000000401ab000     128     128     128 rw---   [ anon ]
00000000401ab000       0       0       0 rw---   [ anon ]
00000000401e6000     128     128     128 rw---   [ anon ]
00000000401e6000       0       0       0 rw---   [ anon ]
0000000040214000     128     128     128 rw---   [ anon ]
0000000040214000       0       0       0 rw---   [ anon ]
0000000040234000     256     256     256 rw---   [ anon ]
0000000040234000       0       0       0 rw---   [ anon ]
0000000040282000     128     128     128 rw---   [ anon ]
0000000040282000       0       0       0 rw---   [ anon ]
00000000402a2000     128     128     128 rw---   [ anon ]
00000000402a2000       0       0       0 rw---   [ anon ]
00000000402d1000     384     384     384 rw---   [ anon ]
00000000402d1000       0       0       0 rw---   [ anon ]
0000000040333000     256     256     256 rw---   [ anon ]
0000000040333000       0       0       0 rw---   [ anon ]
0000000040383000     128     128     128 rw---   [ anon ]
0000000040383000       0       0       0 rw---   [ anon ]
00000000403ad000     128     128     128 rw---   [ anon ]
00000000403ad000       0       0       0 rw---   [ anon ]
00000000403d6000     256     256     256 rw---   [ anon ]
00000000403d6000       0       0       0 rw---   [ anon ]
0000000040428000     236     236     236 rw---   [ anon ]
0000000040428000       0       0       0 rw---   [ anon ]
0000000040463000     256     256     256 rw---   [ anon ]
0000000040463000       0       0       0 rw---   [ anon ]
00000000404ad000     128     128     128 rw---   [ anon ]
00000000404ad000       0       0       0 rw---   [ anon ]
00000000404cd000     128     128     128 rw---   [ anon ]
00000000404cd000       0       0       0 rw---   [ anon ]
0000000040504000     128     128     128 rw---   [ anon ]
0000000040504000       0       0       0 rw---   [ anon ]
0000000040535000     128     128     128 rw---   [ anon ]
0000000040535000       0       0       0 rw---   [ anon ]
0000000040555000     128     128     128 rw---   [ anon ]
0000000040555000       0       0       0 rw---   [ anon ]
000000004058a000     128     128     128 rw---   [ anon ]
000000004058a000       0       0       0 rw---   [ anon ]
00000000405b9000     128     128     128 rw---   [ anon ]
00000000405b9000       0       0       0 rw---   [ anon ]
00000000405de000     128     128     128 rw---   [ anon ]
00000000405de000       0       0       0 rw---   [ anon ]
0000000040603000     128     128     128 rw---   [ anon ]
0000000040603000       0       0       0 rw---   [ anon ]
0000000040623000     128     128     128 rw---   [ anon ]
0000000040623000       0       0       0 rw---   [ anon ]
0000000040657000     128     128     128 rw---   [ anon ]
0000000040657000       0       0       0 rw---   [ anon ]
000000004067c000     512     512     512 rw---   [ anon ]
000000004067c000       0       0       0 rw---   [ anon ]
0000000040702000     256     256     256 rw---   [ anon ]
0000000040702000       0       0       0 rw---   [ anon ]
0000000040746000     640     640     640 rw---   [ anon ]
0000000040746000       0       0       0 rw---   [ anon ]
00000000407f2000     256     256     256 rw---   [ anon ]
00000000407f2000       0       0       0 rw---   [ anon ]
000000004084a000     896     896     896 rw---   [ anon ]
000000004084a000       0       0       0 rw---   [ anon ]
0000000040935000     512     508     508 rw---   [ anon ]
0000000040935000       0       0       0 rw---   [ anon ]
00000000409c3000     256     256     256 rw---   [ anon ]
00000000409c3000       0       0       0 rw---   [ anon ]
0000000040a14000     128     128     128 rw---   [ anon ]
0000000040a14000       0       0       0 rw---   [ anon ]
0000000040a44000     128     128     128 rw---   [ anon ]
0000000040a44000       0       0       0 rw---   [ anon ]
0000000040a64000     128     128     128 rw---   [ anon ]
0000000040a64000       0       0       0 rw---   [ anon ]
0000000040a8c000     384     384     384 rw---   [ anon ]
0000000040a8c000       0       0       0 rw---   [ anon ]
0000000040af6000     128     128     128 rw---   [ anon ]
0000000040af6000       0       0       0 rw---   [ anon ]
0000000040b16000     256     256     256 rw---   [ anon ]
0000000040b16000       0       0       0 rw---   [ anon ]
0000000040b64000     128     128     128 rw---   [ anon ]
0000000040b64000       0       0       0 rw---   [ anon ]
0000000040b8b000     512     512     512 rw---   [ anon ]
0000000040b8b000       0       0       0 rw---   [ anon ]
0000000040c0c000     128     128     128 rw---   [ anon ]
0000000040c0c000       0       0       0 rw---   [ anon ]
0000000040c2c000     640     640     640 rw---   [ anon ]
0000000040c2c000       0       0       0 rw---   [ anon ]
0000000040cd2000     128     128     128 rw---   [ anon ]
0000000040cd2000       0       0       0 rw---   [ anon ]
0000000040cf9000     256     256     256 rw---   [ anon ]
0000000040cf9000       0       0       0 rw---   [ anon ]
0000000040d42000     128     128     128 rw---   [ anon ]
0000000040d42000       0       0       0 rw---   [ anon ]
0000000040d62000     384     384     384 rw---   [ anon ]
0000000040d62000       0       0       0 rw---   [ anon ]
0000000040dcf000     384     384     384 rw---   [ anon ]
0000000040dcf000       0       0       0 rw---   [ anon ]
0000000040e4d000     768     768     768 rw---   [ anon ]
0000000040e4d000       0       0       0 rw---   [ anon ]
0000000040f0f000     256     256     256 rw---   [ anon ]
0000000040f0f000       0       0       0 rw---   [ anon ]
0000000040f5d000     128     128     128 rw---   [ anon ]
0000000040f5d000       0       0       0 rw---   [ anon ]
0000000040f88000     128     128     128 rw---   [ anon ]
0000000040f88000       0       0       0 rw---   [ anon ]
0000000040fb1000     128     128     128 rw---   [ anon ]
0000000040fb1000       0       0       0 rw---   [ anon ]
0000000040fd1000     128     128     128 rw---   [ anon ]
0000000040fd1000       0       0       0 rw---   [ anon ]
000000004100c000     128     128     128 rw---   [ anon ]
000000004100c000       0       0       0 rw---   [ anon ]
0000000041039000     128     128     128 rw---   [ anon ]
0000000041039000       0       0       0 rw---   [ anon ]
0000000041059000     128     128     128 rw---   [ anon ]
0000000041059000       0       0       0 rw---   [ anon ]
0000000041090000     128     128     128 rw---   [ anon ]
0000000041090000       0       0       0 rw---   [ anon ]
00000000410ce000     128     128     128 rw---   [ anon ]
00000000410ce000       0       0       0 rw---   [ anon ]
00000000410ee000     768     768     768 rw---   [ anon ]
00000000410ee000       0       0       0 rw---   [ anon ]
00000000411b1000     256     256     256 rw---   [ anon ]
00000000411b1000       0       0       0 rw---   [ anon ]
00000000411fa000     128     128     128 rw---   [ anon ]
00000000411fa000       0       0       0 rw---   [ anon ]
0000000041220000     384     384     384 rw---   [ anon ]
0000000041220000       0       0       0 rw---   [ anon ]
0000000041283000     128     128     128 rw---   [ anon ]
0000000041283000       0       0       0 rw---   [ anon ]
00000000412a3000     384     384     384 rw---   [ anon ]
00000000412a3000       0       0       0 rw---   [ anon ]
0000000041304000     128     128     128 rw---   [ anon ]
0000000041304000       0       0       0 rw---   [ anon ]
0000000041324000     384     384     384 rw---   [ anon ]
0000000041324000       0       0       0 rw---   [ anon ]
000000004138c000     256     256     256 rw---   [ anon ]
000000004138c000       0       0       0 rw---   [ anon ]
00000000413d6000     128     128     128 rw---   [ anon ]
00000000413d6000       0       0       0 rw---   [ anon ]
00000000413f8000     128     128     128 rw---   [ anon ]
00000000413f8000       0       0       0 rw---   [ anon ]
0000000041418000     384     384     384 rw---   [ anon ]
0000000041418000       0       0       0 rw---   [ anon ]
0000000041484000     256     256     256 rw---   [ anon ]
0000000041484000       0       0       0 rw---   [ anon ]
00000000414cb000     256     256     256 rw---   [ anon ]
00000000414cb000       0       0       0 rw---   [ anon ]
0000000041526000     128     128     128 rw---   [ anon ]
0000000041526000       0       0       0 rw---   [ anon ]
0000000041546000     256     256     256 rw---   [ anon ]
0000000041546000       0       0       0 rw---   [ anon ]
000000004159a000     512     512     512 rw---   [ anon ]
000000004159a000       0       0       0 rw---   [ anon ]
000000004161a000     128     128     128 rw---   [ anon ]
000000004161a000       0       0       0 rw---   [ anon ]
0000000041648000     128     128     128 rw---   [ anon ]
0000000041648000       0       0       0 rw---   [ anon ]
0000000041675000     128     128     128 rw---   [ anon ]
0000000041675000       0       0       0 rw---   [ anon ]
000000004169b000     256     256     256 rw---   [ anon ]
000000004169b000       0       0       0 rw---   [ anon ]
00000000416e3000     256     256     256 rw---   [ anon ]
00000000416e3000       0       0       0 rw---   [ anon ]
0000000041730000     768     768     768 rw---   [ anon ]
0000000041730000       0       0       0 rw---   [ anon ]
000000004180d000     128     128     128 rw---   [ anon ]
000000004180d000       0       0       0 rw---   [ anon ]
000000004183c000     128     128     128 rw---   [ anon ]
000000004183c000       0       0       0 rw---   [ anon ]
000000004186d000     128     128     128 rw---   [ anon ]
000000004186d000       0       0       0 rw---   [ anon ]
000000004188d000     256     256     256 rw---   [ anon ]
000000004188d000       0       0       0 rw---   [ anon ]
00000000418d8000     128     128     128 rw---   [ anon ]
00000000418d8000       0       0       0 rw---   [ anon ]
00000000418f9000     512     512     512 rw---   [ anon ]
00000000418f9000       0       0       0 rw---   [ anon ]
000000004197d000     256     256     256 rw---   [ anon ]
000000004197d000       0       0       0 rw---   [ anon ]
00000000419c5000     256     256     256 rw---   [ anon ]
00000000419c5000       0       0       0 rw---   [ anon ]
0000000041a21000     128     128     128 rw---   [ anon ]
0000000041a21000       0       0       0 rw---   [ anon ]
0000000041a41000     128     128     128 rw---   [ anon ]
0000000041a41000       0       0       0 rw---   [ anon ]
0000000041a7c000     256     256     256 rw---   [ anon ]
0000000041a7c000       0       0       0 rw---   [ anon ]
0000000041acd000     256     256     256 rw---   [ anon ]
0000000041acd000       0       0       0 rw---   [ anon ]
0000000041b14000     384     384     384 rw---   [ anon ]
0000000041b14000       0       0       0 rw---   [ anon ]
0000000041b85000     128     128     128 rw---   [ anon ]
0000000041b85000       0       0       0 rw---   [ anon ]
0000000041baf000     128     128     128 rw---   [ anon ]
0000000041baf000       0       0       0 rw---   [ anon ]
0000000041bd2000     256     256     256 rw---   [ anon ]
0000000041bd2000       0       0       0 rw---   [ anon ]
0000000041c19000     128     128     128 rw---   [ anon ]
0000000041c19000       0       0       0 rw---   [ anon ]
0000000041c3a000     128     124     124 rw---   [ anon ]
0000000041c3a000       0       0       0 rw---   [ anon ]
0000000041c5a000     256     256     256 rw---   [ anon ]
0000000041c5a000       0       0       0 rw---   [ anon ]
0000000041ca8000     128     128     128 rw---   [ anon ]
0000000041ca8000       0       0       0 rw---   [ anon ]
0000000041cc8000     256     256     256 rw---   [ anon ]
0000000041cc8000       0       0       0 rw---   [ anon ]
0000000041d10000     256     256     256 rw---   [ anon ]
0000000041d10000       0       0       0 rw---   [ anon ]
0000000041d6c000     128     128     128 rw---   [ anon ]
0000000041d6c000       0       0       0 rw---   [ anon ]
0000000041d8c000   15560   15560   15560 rw---   [ anon ]
0000000041d8c000       0       0       0 rw---   [ anon ]
0000000042cd0000    6656    6656    6656 rw---   [ anon ]
0000000042cd0000       0       0       0 rw---   [ anon ]
0000000043351000   14720   14720   14720 rw---   [ anon ]
0000000043351000       0       0       0 rw---   [ anon ]
00000000441b2000    2816    2816    2816 rw---   [ anon ]
00000000441b2000       0       0       0 rw---   [ anon ]
0000000044473000   12160   12160   12160 rw---   [ anon ]
0000000044473000       0       0       0 rw---   [ anon ]
0000000045054000   23936   23936   23936 rw---   [ anon ]
0000000045054000       0       0       0 rw---   [ anon ]
00000000467b5000   42112   42112   42112 rw---   [ anon ]
00000000467b5000       0       0       0 rw---   [ anon ]
00000000490d6000   90244   90244   90244 rw---   [ anon ]
00000000490d6000       0       0       0 rw---   [ anon ]
000000004e8f8000  158924  158924  158924 rw---   [ anon ]
000000004e8f8000       0       0       0 rw---   [ anon ]
000000005842c000  477700  477700  477700 rw---   [ anon ]
000000005842c000       0       0       0 rw---   [ anon ]
00000000756be000    4224    4224    4224 rw---   [ anon ]
00000000756be000       0       0       0 rw---   [ anon ]
0000000075adf000   63616   63616   63616 rw---   [ anon ]
0000000075adf000       0       0       0 rw---   [ anon ]
0000000079910000  105344  105344  105344 rw---   [ anon ]
0000000079910000       0       0       0 rw---   [ anon ]
00007efe585b0000     320     268     268 r-x--   [ anon ]
00007efe585b0000       0       0       0 r-x--   [ anon ]
00007efe6e935000    5832    5828    5828 rw---   [ anon ]
00007efe6e935000       0       0       0 rw---   [ anon ]
00007efe6eee7000       8       0       0 r-x-- lsyslog.so
00007efe6eee7000       0       0       0 r-x-- lsyslog.so
00007efe6eee9000    2044       0       0 ----- lsyslog.so
00007efe6eee9000       0       0       0 ----- lsyslog.so
00007efe6f0e8000       4       4       4 r---- lsyslog.so
00007efe6f0e8000       0       0       0 r---- lsyslog.so
00007efe6f0e9000       4       4       4 rw--- lsyslog.so
00007efe6f0e9000       0       0       0 rw--- lsyslog.so
00007efe6f0ea000      12       0       0 r-x-- lua_pack.so
00007efe6f0ea000       0       0       0 r-x-- lua_pack.so
00007efe6f0ed000    2044       0       0 ----- lua_pack.so
00007efe6f0ed000       0       0       0 ----- lua_pack.so
00007efe6f2ec000       4       4       4 r---- lua_pack.so
00007efe6f2ec000       0       0       0 r---- lua_pack.so
00007efe6f2ed000       4       4       4 rw--- lua_pack.so
00007efe6f2ed000       0       0       0 rw--- lua_pack.so
00007efe6f2ee000      92       0       0 r-x-- libresolv-2.23.so
00007efe6f2ee000       0       0       0 r-x-- libresolv-2.23.so
00007efe6f305000    2048       0       0 ----- libresolv-2.23.so
00007efe6f305000       0       0       0 ----- libresolv-2.23.so
00007efe6f505000       4       4       4 r---- libresolv-2.23.so
00007efe6f505000       0       0       0 r---- libresolv-2.23.so
00007efe6f506000       4       4       4 rw--- libresolv-2.23.so
00007efe6f506000       0       0       0 rw--- libresolv-2.23.so
00007efe6f507000       8       0       0 rw---   [ anon ]
00007efe6f507000       0       0       0 rw---   [ anon ]
00007efe6f509000      20       0       0 r-x-- libnss_dns-2.23.so
00007efe6f509000       0       0       0 r-x-- libnss_dns-2.23.so
00007efe6f50e000    2048       0       0 ----- libnss_dns-2.23.so
00007efe6f50e000       0       0       0 ----- libnss_dns-2.23.so
00007efe6f70e000       4       4       4 r---- libnss_dns-2.23.so
00007efe6f70e000       0       0       0 r---- libnss_dns-2.23.so
00007efe6f70f000       4       4       4 rw--- libnss_dns-2.23.so
00007efe6f70f000       0       0       0 rw--- libnss_dns-2.23.so
00007efe6f710000    2764       0       0 r-x-- _openssl.so
00007efe6f710000       0       0       0 r-x-- _openssl.so
00007efe6f9c3000    2044       0       0 ----- _openssl.so
00007efe6f9c3000       0       0       0 ----- _openssl.so
00007efe6fbc2000     132     132     132 r---- _openssl.so
00007efe6fbc2000       0       0       0 r---- _openssl.so
00007efe6fbe3000      96      96      96 rw--- _openssl.so
00007efe6fbe3000       0       0       0 rw--- _openssl.so
00007efe6fbfb000      16       4       4 rw---   [ anon ]
00007efe6fbfb000       0       0       0 rw---   [ anon ]
00007efe6fbff000      40      28       0 r-x-- lpeg.so
00007efe6fbff000       0       0       0 r-x-- lpeg.so
00007efe6fc09000    2044       0       0 ----- lpeg.so
00007efe6fc09000       0       0       0 ----- lpeg.so
00007efe6fe08000       4       4       4 r---- lpeg.so
00007efe6fe08000       0       0       0 r---- lpeg.so
00007efe6fe09000       4       4       4 rw--- lpeg.so
00007efe6fe09000       0       0       0 rw--- lpeg.so
00007efe6fe0a000       8       0       0 r-x-- random.so
00007efe6fe0a000       0       0       0 r-x-- random.so
00007efe6fe0c000    2044       0       0 ----- random.so
00007efe6fe0c000       0       0       0 ----- random.so
00007efe7000b000       4       4       4 r---- random.so
00007efe7000b000       0       0       0 r---- random.so
00007efe7000c000       4       4       4 rw--- random.so
00007efe7000c000       0       0       0 rw--- random.so
00007efe7000d000      28      24       0 r-x-- cjson.so
00007efe7000d000       0       0       0 r-x-- cjson.so
00007efe70014000    2048       0       0 ----- cjson.so
00007efe70014000       0       0       0 ----- cjson.so
00007efe70214000       4       4       4 r---- cjson.so
00007efe70214000       0       0       0 r---- cjson.so
00007efe70215000       4       4       4 rw--- cjson.so
00007efe70215000       0       0       0 rw--- cjson.so
00007efe70216000      16       0       0 r-x-- lfs.so
00007efe70216000       0       0       0 r-x-- lfs.so
00007efe7021a000    2044       0       0 ----- lfs.so
00007efe7021a000       0       0       0 ----- lfs.so
00007efe70419000       4       4       4 r---- lfs.so
00007efe70419000       0       0       0 r---- lfs.so
00007efe7041a000       4       4       4 rw--- lfs.so
00007efe7041a000       0       0       0 rw--- lfs.so
00007efe7041b000       4       0       0 r-x-- lua_system_constants.so
00007efe7041b000       0       0       0 r-x-- lua_system_constants.so
00007efe7041c000    2048       0       0 ----- lua_system_constants.so
00007efe7041c000       0       0       0 ----- lua_system_constants.so
00007efe7061c000       4       4       4 r---- lua_system_constants.so
00007efe7061c000       0       0       0 r---- lua_system_constants.so
00007efe7061d000       4       4       4 rw--- lua_system_constants.so
00007efe7061d000       0       0       0 rw--- lua_system_constants.so
00007efe7061e000      56       0       0 r-x-- core.so
00007efe7061e000       0       0       0 r-x-- core.so
00007efe7062c000    2044       0       0 ----- core.so
00007efe7062c000       0       0       0 ----- core.so
00007efe7082b000       4       4       4 r---- core.so
00007efe7082b000       0       0       0 r---- core.so
00007efe7082c000       4       4       4 rw--- core.so
00007efe7082c000       0       0       0 rw--- core.so
00007efe7082d000    5120     220     220 rw-s- zero (deleted)
00007efe7082d000       0       0       0 rw-s- zero (deleted)
00007efe70d2d000    5120       0       0 rw-s- zero (deleted)
00007efe70d2d000       0       0       0 rw-s- zero (deleted)
00007efe7122d000    5120    1360    1360 rw-s- zero (deleted)
00007efe7122d000       0       0       0 rw-s- zero (deleted)
00007efe7172d000  131072    1092    1092 rw-s- zero (deleted)
00007efe7172d000       0       0       0 rw-s- zero (deleted)
00007efe7972d000    5120      40      40 rw-s- zero (deleted)
00007efe7972d000       0       0       0 rw-s- zero (deleted)
00007efe79c2d000      44       0       0 r-x-- libnss_files-2.23.so
00007efe79c2d000       0       0       0 r-x-- libnss_files-2.23.so
00007efe79c38000    2044       0       0 ----- libnss_files-2.23.so
00007efe79c38000       0       0       0 ----- libnss_files-2.23.so
00007efe79e37000       4       4       4 r---- libnss_files-2.23.so
00007efe79e37000       0       0       0 r---- libnss_files-2.23.so
00007efe79e38000       4       4       4 rw--- libnss_files-2.23.so
00007efe79e38000       0       0       0 rw--- libnss_files-2.23.so
00007efe79e39000      24       0       0 rw---   [ anon ]
00007efe79e39000       0       0       0 rw---   [ anon ]
00007efe79e3f000      44       0       0 r-x-- libnss_nis-2.23.so
00007efe79e3f000       0       0       0 r-x-- libnss_nis-2.23.so
00007efe79e4a000    2044       0       0 ----- libnss_nis-2.23.so
00007efe79e4a000       0       0       0 ----- libnss_nis-2.23.so
00007efe7a049000       4       4       4 r---- libnss_nis-2.23.so
00007efe7a049000       0       0       0 r---- libnss_nis-2.23.so
00007efe7a04a000       4       4       4 rw--- libnss_nis-2.23.so
00007efe7a04a000       0       0       0 rw--- libnss_nis-2.23.so
00007efe7a04b000      88       0       0 r-x-- libnsl-2.23.so
00007efe7a04b000       0       0       0 r-x-- libnsl-2.23.so
00007efe7a061000    2044       0       0 ----- libnsl-2.23.so
00007efe7a061000       0       0       0 ----- libnsl-2.23.so
00007efe7a260000       4       4       4 r---- libnsl-2.23.so
00007efe7a260000       0       0       0 r---- libnsl-2.23.so
00007efe7a261000       4       4       4 rw--- libnsl-2.23.so
00007efe7a261000       0       0       0 rw--- libnsl-2.23.so
00007efe7a262000       8       0       0 rw---   [ anon ]
00007efe7a262000       0       0       0 rw---   [ anon ]
00007efe7a264000      32       0       0 r-x-- libnss_compat-2.23.so
00007efe7a264000       0       0       0 r-x-- libnss_compat-2.23.so
00007efe7a26c000    2044       0       0 ----- libnss_compat-2.23.so
00007efe7a26c000       0       0       0 ----- libnss_compat-2.23.so
00007efe7a46b000       4       4       4 r---- libnss_compat-2.23.so
00007efe7a46b000       0       0       0 r---- libnss_compat-2.23.so
00007efe7a46c000       4       4       4 rw--- libnss_compat-2.23.so
00007efe7a46c000       0       0       0 rw--- libnss_compat-2.23.so
00007efe7a46d000      88      40       0 r-x-- libgcc_s.so.1
00007efe7a46d000       0       0       0 r-x-- libgcc_s.so.1
00007efe7a483000    2044       0       0 ----- libgcc_s.so.1
00007efe7a483000       0       0       0 ----- libgcc_s.so.1
00007efe7a682000       4       4       4 rw--- libgcc_s.so.1
00007efe7a682000       0       0       0 rw--- libgcc_s.so.1
00007efe7a683000    1792    1460       0 r-x-- libc-2.23.so
00007efe7a683000       0       0       0 r-x-- libc-2.23.so
00007efe7a843000    2048       0       0 ----- libc-2.23.so
00007efe7a843000       0       0       0 ----- libc-2.23.so
00007efe7aa43000      16      16      16 r---- libc-2.23.so
00007efe7aa43000       0       0       0 r---- libc-2.23.so
00007efe7aa47000       8       8       8 rw--- libc-2.23.so
00007efe7aa47000       0       0       0 rw--- libc-2.23.so
00007efe7aa49000      16      16      16 rw---   [ anon ]
00007efe7aa49000       0       0       0 rw---   [ anon ]
00007efe7aa4d000     100      56       0 r-x-- libz.so.1.2.8
00007efe7aa4d000       0       0       0 r-x-- libz.so.1.2.8
00007efe7aa66000    2044       0       0 ----- libz.so.1.2.8
00007efe7aa66000       0       0       0 ----- libz.so.1.2.8
00007efe7ac65000       4       4       4 r---- libz.so.1.2.8
00007efe7ac65000       0       0       0 r---- libz.so.1.2.8
00007efe7ac66000       4       4       4 rw--- libz.so.1.2.8
00007efe7ac66000       0       0       0 rw--- libz.so.1.2.8
00007efe7ac67000    1056     216       0 r-x-- libm-2.23.so
00007efe7ac67000       0       0       0 r-x-- libm-2.23.so
00007efe7ad6f000    2044       0       0 ----- libm-2.23.so
00007efe7ad6f000       0       0       0 ----- libm-2.23.so
00007efe7af6e000       4       4       4 r---- libm-2.23.so
00007efe7af6e000       0       0       0 r---- libm-2.23.so
00007efe7af6f000       4       4       4 rw--- libm-2.23.so
00007efe7af6f000       0       0       0 rw--- libm-2.23.so
00007efe7af70000     480     316       0 r-x-- libluajit-5.1.so.2.1.0
00007efe7af70000       0       0       0 r-x-- libluajit-5.1.so.2.1.0
00007efe7afe8000    2048       0       0 ----- libluajit-5.1.so.2.1.0
00007efe7afe8000       0       0       0 ----- libluajit-5.1.so.2.1.0
00007efe7b1e8000       8       8       8 r---- libluajit-5.1.so.2.1.0
00007efe7b1e8000       0       0       0 r---- libluajit-5.1.so.2.1.0
00007efe7b1ea000       4       4       4 rw--- libluajit-5.1.so.2.1.0
00007efe7b1ea000       0       0       0 rw--- libluajit-5.1.so.2.1.0
00007efe7b1eb000      36       0       0 r-x-- libcrypt-2.23.so
00007efe7b1eb000       0       0       0 r-x-- libcrypt-2.23.so
00007efe7b1f4000    2044       0       0 ----- libcrypt-2.23.so
00007efe7b1f4000       0       0       0 ----- libcrypt-2.23.so
00007efe7b3f3000       4       4       4 r---- libcrypt-2.23.so
00007efe7b3f3000       0       0       0 r---- libcrypt-2.23.so
00007efe7b3f4000       4       4       4 rw--- libcrypt-2.23.so
00007efe7b3f4000       0       0       0 rw--- libcrypt-2.23.so
00007efe7b3f5000     184       0       0 rw---   [ anon ]
00007efe7b3f5000       0       0       0 rw---   [ anon ]
00007efe7b423000      96      96       0 r-x-- libpthread-2.23.so
00007efe7b423000       0       0       0 r-x-- libpthread-2.23.so
00007efe7b43b000    2044       0       0 ----- libpthread-2.23.so
00007efe7b43b000       0       0       0 ----- libpthread-2.23.so
00007efe7b63a000       4       4       4 r---- libpthread-2.23.so
00007efe7b63a000       0       0       0 r---- libpthread-2.23.so
00007efe7b63b000       4       4       4 rw--- libpthread-2.23.so
00007efe7b63b000       0       0       0 rw--- libpthread-2.23.so
00007efe7b63c000      16       4       4 rw---   [ anon ]
00007efe7b63c000       0       0       0 rw---   [ anon ]
00007efe7b640000      12      12       0 r-x-- libdl-2.23.so
00007efe7b640000       0       0       0 r-x-- libdl-2.23.so
00007efe7b643000    2044       0       0 ----- libdl-2.23.so
00007efe7b643000       0       0       0 ----- libdl-2.23.so
00007efe7b842000       4       4       4 r---- libdl-2.23.so
00007efe7b842000       0       0       0 r---- libdl-2.23.so
00007efe7b843000       4       4       4 rw--- libdl-2.23.so
00007efe7b843000       0       0       0 rw--- libdl-2.23.so
00007efe7b844000     152     152       0 r-x-- ld-2.23.so
00007efe7b844000       0       0       0 r-x-- ld-2.23.so
00007efe7b8cc000    1540    1536    1536 rw---   [ anon ]
00007efe7b8cc000       0       0       0 rw---   [ anon ]
00007efe7ba4d000      64      16      16 rwx--   [ anon ]
00007efe7ba4d000       0       0       0 rwx--   [ anon ]
00007efe7ba5d000      24      24      24 rw---   [ anon ]
00007efe7ba5d000       0       0       0 rw---   [ anon ]
00007efe7ba68000       4       4       4 rw-s- zero (deleted)
00007efe7ba68000       0       0       0 rw-s- zero (deleted)
00007efe7ba69000       4       4       4 r---- ld-2.23.so
00007efe7ba69000       0       0       0 r---- ld-2.23.so
00007efe7ba6a000       4       4       4 rw--- ld-2.23.so
00007efe7ba6a000       0       0       0 rw--- ld-2.23.so
00007efe7ba6b000       4       4       4 rw---   [ anon ]
00007efe7ba6b000       0       0       0 rw---   [ anon ]
00007ffcc05c1000     132      40      40 rw---   [ stack ]
00007ffcc05c1000       0       0       0 rw---   [ stack ]
00007ffcc05f8000       8       0       0 r----   [ anon ]
00007ffcc05f8000       0       0       0 r----   [ anon ]
00007ffcc05fa000       8       4       0 r-x--   [ anon ]
00007ffcc05fa000       0       0       0 r-x--   [ anon ]
ffffffffff600000       4       0       0 r-x--   [ anon ]
ffffffffff600000       0       0       0 r-x--   [ anon ]
---------------- ------- ------- -------
total kB         1489988 1133264 1130240
root@ip-10-1-2-104:~#
```


```
root@ip-10-1-2-104:~# pmap -XX 1360
1360:   nginx: worker process
         Address Perm   Offset Device  Inode    Size     Rss     Pss Shared_Clean Shared_Dirty Private_Clean Private_Dirty Referenced Anonymous AnonHugePages Shared_Hugetlb Private_Hugetlb Swap SwapPss KernelPageSize MMUPageSize Locked                   VmFlagsMapping
        00400000 r-xp 00000000  ca:01  19363    4092     616      70          616            0             0             0        616         0             0              0               0    0       0              4           4      0    rd ex mr mw me dw sd  nginx
        009fe000 r--p 003fe000  ca:01  19363       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0    rd mr mw me dw ac sd  nginx
        009ff000 rw-p 003ff000  ca:01  19363     192     116      30            4           92             0            20        112       112             0              0               0    0       0              4           4      0 rd wr mr mw me dw ac sd  nginx
        00a2f000 rw-p 00000000  00:00      0     152      40      36            0            4             0            36         36        40             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        01dee000 rw-p 00000000  00:00      0     628     628     350            0          312             0           316        464       628             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  [heap]
        01e8b000 rw-p 00000000  00:00      0     852     672     672            0            0             0           672        636       672             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  [heap]
        129bf000 rw-p 00000000  00:00      0   26496   26444   26444            0            0             0         26444      19560     26444             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        3d147000 rw-p 00000000  00:00      0   47872   47872   47872            0            0             0         47872      47872     47872         47104              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40015000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4003e000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40074000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        400a6000 rw-p 00000000  00:00      0     640     640     640            0            0             0           640        640       640             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4015e000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        401ab000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        401e6000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40214000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40234000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40282000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        402a2000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        402d1000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40333000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40383000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        403ad000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        403d6000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40428000 rw-p 00000000  00:00      0     236     236      29            0          232             0             4          4       236             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40463000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        404ad000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        404cd000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40504000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40535000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40555000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4058a000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        405b9000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        405de000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40603000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40623000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40657000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4067c000 rw-p 00000000  00:00      0     512     512     512            0            0             0           512        512       512             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40702000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40746000 rw-p 00000000  00:00      0     640     640     640            0            0             0           640        640       640             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        407f2000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4084a000 rw-p 00000000  00:00      0     896     896     896            0            0             0           896        896       896             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40935000 rw-p 00000000  00:00      0     512     508     508            0            0             0           508        508       508             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        409c3000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40a14000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40a44000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40a64000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40a8c000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40af6000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40b16000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40b64000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40b8b000 rw-p 00000000  00:00      0     512     512     512            0            0             0           512        512       512             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40c0c000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40c2c000 rw-p 00000000  00:00      0     640     640     640            0            0             0           640        640       640             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40cd2000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40cf9000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40d42000 rw-p 00000000  00:00      0     128     128     120            0            8             0           120        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40d62000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40dcf000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40e4d000 rw-p 00000000  00:00      0     768     768     768            0            0             0           768        768       768             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40f0f000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40f5d000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40f88000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40fb1000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        40fd1000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4100c000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41039000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41059000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41090000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        410ce000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        410ee000 rw-p 00000000  00:00      0     768     768     768            0            0             0           768        768       768             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        411b1000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        411fa000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41220000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41283000 rw-p 00000000  00:00      0     128     128     124            0            4             0           124        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        412a3000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41304000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41324000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4138c000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        413d6000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        413f8000 rw-p 00000000  00:00      0     128     128     110            0           20             0           108        108       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41418000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41484000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        414cb000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41526000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41546000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4159a000 rw-p 00000000  00:00      0     512     512     512            0            0             0           512        512       512             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4161a000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41648000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41675000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4169b000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        416e3000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41730000 rw-p 00000000  00:00      0     768     768     768            0            0             0           768        768       768             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4180d000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4183c000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4186d000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4188d000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        418d8000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        418f9000 rw-p 00000000  00:00      0     512     512     512            0            0             0           512        512       512             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4197d000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        419c5000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41a21000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41a41000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41a7c000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41acd000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41b14000 rw-p 00000000  00:00      0     384     384     384            0            0             0           384        384       384             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41b85000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41baf000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41bd2000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41c19000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41c3a000 rw-p 00000000  00:00      0     128     124     120            0            4             0           120        124       124             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41c5a000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41ca8000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41cc8000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41d10000 rw-p 00000000  00:00      0     256     256     256            0            0             0           256        256       256             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41d6c000 rw-p 00000000  00:00      0     128     128     128            0            0             0           128        128       128             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        41d8c000 rw-p 00000000  00:00      0   15560   15560   15560            0            0             0         15560      15536     15560         14336              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        42cd0000 rw-p 00000000  00:00      0    6656    6656    6656            0            0             0          6656       6656      6656          4096              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        43351000 rw-p 00000000  00:00      0   14720   14720   14720            0            0             0         14720      14720     14720         12288              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        441b2000 rw-p 00000000  00:00      0    2816    2816    2816            0            0             0          2816       2816      2816          2048              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        44473000 rw-p 00000000  00:00      0   12160   12160   12160            0            0             0         12160      12160     12160         10240              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        45054000 rw-p 00000000  00:00      0   23936   23936   23936            0            0             0         23936      23936     23936         20480              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        467b5000 rw-p 00000000  00:00      0   42112   42112   42112            0            0             0         42112      42112     42112         40960              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        490d6000 rw-p 00000000  00:00      0   90244   90244   90244            0            0             0         90244      90244     90244         88064              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        4e8f8000 rw-p 00000000  00:00      0  158924  158924  158924            0            0             0        158924     158904    158924        157696              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        5842c000 rw-p 00000000  00:00      0  477700  477700  477700            0            0             0        477700     477688    477700        475136              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        756be000 rw-p 00000000  00:00      0    4224    4224    4224            0            0             0          4224       4172      4224          2048              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        75adf000 rw-p 00000000  00:00      0   63616   63616   63616            0            0             0         63616      63600     63616         61440              0               0    0       0              4           4      0    rd wr mr mw me ac sd
        79910000 rw-p 00000000  00:00      0  105344  105344  105344            0            0             0        105344     105256    105344        102400              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe585b0000 r-xp 00000000  00:00      0     320     268     268            0            0             0           268        252       268             0              0               0    0       0              4           4      0    rd ex mr mw me ac sd
    7efe6e935000 rw-p 00000000  00:00      0    5832    5828    5828            0            0             0          5828       5828      5828          4096              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe6eee7000 r-xp 00000000  ca:01  20192       8       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  lsyslog.so
    7efe6eee9000 ---p 00002000  ca:01  20192    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  lsyslog.so
    7efe6f0e8000 r--p 00001000  ca:01  20192       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  lsyslog.so
    7efe6f0e9000 rw-p 00002000  ca:01  20192       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  lsyslog.so
    7efe6f0ea000 r-xp 00000000  ca:01  20191      12       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  lua_pack.so
    7efe6f0ed000 ---p 00003000  ca:01  20191    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  lua_pack.so
    7efe6f2ec000 r--p 00002000  ca:01  20191       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  lua_pack.so
    7efe6f2ed000 rw-p 00003000  ca:01  20191       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  lua_pack.so
    7efe6f2ee000 r-xp 00000000  ca:01 401664      92       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libresolv-2.23.so
    7efe6f305000 ---p 00017000  ca:01 401664    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libresolv-2.23.so
    7efe6f505000 r--p 00017000  ca:01 401664       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libresolv-2.23.so
    7efe6f506000 rw-p 00018000  ca:01 401664       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libresolv-2.23.so
    7efe6f507000 rw-p 00000000  00:00      0       8       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe6f509000 r-xp 00000000  ca:01 401670      20       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libnss_dns-2.23.so
    7efe6f50e000 ---p 00005000  ca:01 401670    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libnss_dns-2.23.so
    7efe6f70e000 r--p 00005000  ca:01 401670       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libnss_dns-2.23.so
    7efe6f70f000 rw-p 00006000  ca:01 401670       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libnss_dns-2.23.so
    7efe6f710000 r-xp 00000000  ca:01  20194    2764       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  _openssl.so
    7efe6f9c3000 ---p 002b3000  ca:01  20194    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  _openssl.so
    7efe6fbc2000 r--p 002b2000  ca:01  20194     132     132      14            0          132             0             0          0       132             0              0               0    0       0              4           4      0       rd mr mw me ac sd  _openssl.so
    7efe6fbe3000 rw-p 002d3000  ca:01  20194      96      96      10            0           96             0             0         12        96             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  _openssl.so
    7efe6fbfb000 rw-p 00000000  00:00      0      16       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe6fbff000 r-xp 00000000  ca:01  20195      40      28       3           28            0             0             0         28         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  lpeg.so
    7efe6fc09000 ---p 0000a000  ca:01  20195    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  lpeg.so
    7efe6fe08000 r--p 00009000  ca:01  20195       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  lpeg.so
    7efe6fe09000 rw-p 0000a000  ca:01  20195       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  lpeg.so
    7efe6fe0a000 r-xp 00000000  ca:01  20193       8       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  random.so
    7efe6fe0c000 ---p 00002000  ca:01  20193    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  random.so
    7efe7000b000 r--p 00001000  ca:01  20193       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  random.so
    7efe7000c000 rw-p 00002000  ca:01  20193       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  random.so
    7efe7000d000 r-xp 00000000  ca:01  19274      28      24       2           24            0             0             0         24         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  cjson.so
    7efe70014000 ---p 00007000  ca:01  19274    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  cjson.so
    7efe70214000 r--p 00007000  ca:01  19274       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  cjson.so
    7efe70215000 rw-p 00008000  ca:01  19274       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  cjson.so
    7efe70216000 r-xp 00000000  ca:01  20190      16       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  lfs.so
    7efe7021a000 ---p 00004000  ca:01  20190    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  lfs.so
    7efe70419000 r--p 00003000  ca:01  20190       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  lfs.so
    7efe7041a000 rw-p 00004000  ca:01  20190       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  lfs.so
    7efe7041b000 r-xp 00000000  ca:01  20189       4       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  lua_system_constants.so
    7efe7041c000 ---p 00001000  ca:01  20189    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  lua_system_constants.so
    7efe7061c000 r--p 00001000  ca:01  20189       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  lua_system_constants.so
    7efe7061d000 rw-p 00002000  ca:01  20189       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  lua_system_constants.so
    7efe7061e000 r-xp 00000000  ca:01  20188      56       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  core.so
    7efe7062c000 ---p 0000e000  ca:01  20188    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  core.so
    7efe7082b000 r--p 0000d000  ca:01  20188       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  core.so
    7efe7082c000 rw-p 0000e000  ca:01  20188       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  core.so
    7efe7082d000 rw-s 00000000  00:05  15856    5120     220      27            0          220             0             0        192         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe70d2d000 rw-s 00000000  00:05  15855    5120       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe7122d000 rw-s 00000000  00:05  15854    5120    1360     169            0         1360             0             0        728         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe7172d000 rw-s 00000000  00:05  15853  131072    1092     134            0         1092             0             0        936         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe7972d000 rw-s 00000000  00:05  15852    5120      40       4            0           40             0             0         40         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe79c2d000 r-xp 00000000  ca:01 401673      44       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libnss_files-2.23.so
    7efe79c38000 ---p 0000b000  ca:01 401673    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libnss_files-2.23.so
    7efe79e37000 r--p 0000a000  ca:01 401673       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libnss_files-2.23.so
    7efe79e38000 rw-p 0000b000  ca:01 401673       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libnss_files-2.23.so
    7efe79e39000 rw-p 00000000  00:00      0      24       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe79e3f000 r-xp 00000000  ca:01 401677      44       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libnss_nis-2.23.so
    7efe79e4a000 ---p 0000b000  ca:01 401677    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libnss_nis-2.23.so
    7efe7a049000 r--p 0000a000  ca:01 401677       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libnss_nis-2.23.so
    7efe7a04a000 rw-p 0000b000  ca:01 401677       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libnss_nis-2.23.so
    7efe7a04b000 r-xp 00000000  ca:01 401657      88       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libnsl-2.23.so
    7efe7a061000 ---p 00016000  ca:01 401657    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libnsl-2.23.so
    7efe7a260000 r--p 00015000  ca:01 401657       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libnsl-2.23.so
    7efe7a261000 rw-p 00016000  ca:01 401657       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libnsl-2.23.so
    7efe7a262000 rw-p 00000000  00:00      0       8       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7a264000 r-xp 00000000  ca:01 401668      32       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libnss_compat-2.23.so
    7efe7a26c000 ---p 00008000  ca:01 401668    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libnss_compat-2.23.so
    7efe7a46b000 r--p 00007000  ca:01 401668       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libnss_compat-2.23.so
    7efe7a46c000 rw-p 00008000  ca:01 401668       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libnss_compat-2.23.so
    7efe7a46d000 r-xp 00000000  ca:01 396885      88      40       3           40            0             0             0         40         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libgcc_s.so.1
    7efe7a483000 ---p 00016000  ca:01 396885    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libgcc_s.so.1
    7efe7a682000 rw-p 00015000  ca:01 396885       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libgcc_s.so.1
    7efe7a683000 r-xp 00000000  ca:01 401660    1792    1460      30         1460            0             0             0       1460         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libc-2.23.so
    7efe7a843000 ---p 001c0000  ca:01 401660    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libc-2.23.so
    7efe7aa43000 r--p 001c0000  ca:01 401660      16      16       1            0           16             0             0         16        16             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libc-2.23.so
    7efe7aa47000 rw-p 001c4000  ca:01 401660       8       8       8            0            0             0             8          8         8             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libc-2.23.so
    7efe7aa49000 rw-p 00000000  00:00      0      16      16      16            0            0             0            16         16        16             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7aa4d000 r-xp 00000000  ca:01 396956     100      56       5           56            0             0             0         56         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libz.so.1.2.8
    7efe7aa66000 ---p 00019000  ca:01 396956    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libz.so.1.2.8
    7efe7ac65000 r--p 00018000  ca:01 396956       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libz.so.1.2.8
    7efe7ac66000 rw-p 00019000  ca:01 396956       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libz.so.1.2.8
    7efe7ac67000 r-xp 00000000  ca:01 401656    1056     216      19          216            0             0             0        216         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libm-2.23.so
    7efe7ad6f000 ---p 00108000  ca:01 401656    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libm-2.23.so
    7efe7af6e000 r--p 00107000  ca:01 401656       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libm-2.23.so
    7efe7af6f000 rw-p 00108000  ca:01 401656       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libm-2.23.so
    7efe7af70000 r-xp 00000000  ca:01  19332     480     316      35          316            0             0             0        316         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libluajit-5.1.so.2.1.0
    7efe7afe8000 ---p 00078000  ca:01  19332    2048       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libluajit-5.1.so.2.1.0
    7efe7b1e8000 r--p 00078000  ca:01  19332       8       8       0            0            8             0             0          8         8             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libluajit-5.1.so.2.1.0
    7efe7b1ea000 rw-p 0007a000  ca:01  19332       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libluajit-5.1.so.2.1.0
    7efe7b1eb000 r-xp 00000000  ca:01 401679      36       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libcrypt-2.23.so
    7efe7b1f4000 ---p 00009000  ca:01 401679    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libcrypt-2.23.so
    7efe7b3f3000 r--p 00008000  ca:01 401679       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libcrypt-2.23.so
    7efe7b3f4000 rw-p 00009000  ca:01 401679       4       4       0            0            4             0             0          0         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libcrypt-2.23.so
    7efe7b3f5000 rw-p 00000000  00:00      0     184       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7b423000 r-xp 00000000  ca:01 401659      96      96       2           96            0             0             0         96         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libpthread-2.23.so
    7efe7b43b000 ---p 00018000  ca:01 401659    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libpthread-2.23.so
    7efe7b63a000 r--p 00017000  ca:01 401659       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libpthread-2.23.so
    7efe7b63b000 rw-p 00018000  ca:01 401659       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libpthread-2.23.so
    7efe7b63c000 rw-p 00000000  00:00      0      16       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7b640000 r-xp 00000000  ca:01 401662      12      12       0           12            0             0             0         12         0             0              0               0    0       0              4           4      0       rd ex mr mw me sd  libdl-2.23.so
    7efe7b643000 ---p 00003000  ca:01 401662    2044       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0             mr mw me sd  libdl-2.23.so
    7efe7b842000 r--p 00002000  ca:01 401662       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0       rd mr mw me ac sd  libdl-2.23.so
    7efe7b843000 rw-p 00003000  ca:01 401662       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd  libdl-2.23.so
    7efe7b844000 r-xp 00000000  ca:01 401658     152     152       3          152            0             0             0        152         0             0              0               0    0       0              4           4      0    rd ex mr mw me dw sd  ld-2.23.so
    7efe7b8cc000 rw-p 00000000  00:00      0    1540    1536    1536            0            0             0          1536       1536      1536             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7ba4d000 rwxp 00000000  00:00      0      64      16      12            0            4             0            12         16        16             0              0               0    0       0              4           4      0 rd wr ex mr mw me ac sd
    7efe7ba5d000 rw-p 00000000  00:00      0      24      24       6            0           20             0             4         24        24             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7efe7ba68000 rw-s 00000000  00:05  15131       4       4       0            0            4             0             0          4         0             0              0               0    0       0              4           4      0 rd wr sh mr mw me ms sd  zero (deleted)
    7efe7ba69000 r--p 00025000  ca:01 401658       4       4       0            0            4             0             0          4         4             0              0               0    0       0              4           4      0    rd mr mw me dw ac sd  ld-2.23.so
    7efe7ba6a000 rw-p 00026000  ca:01 401658       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0 rd wr mr mw me dw ac sd  ld-2.23.so
    7efe7ba6b000 rw-p 00000000  00:00      0       4       4       4            0            0             0             4          4         4             0              0               0    0       0              4           4      0    rd wr mr mw me ac sd
    7ffcc05c1000 rw-p 00000000  00:00      0     136      40      40            0            0             0            40         40        40             0              0               0    0       0              4           4      0    rd wr mr mw me gd ac  [stack]
    7ffcc05f8000 r--p 00000000  00:00      0       8       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0    rd mr pf io de dd sd  [vvar]
    7ffcc05fa000 r-xp 00000000  00:00      0       8       4       0            4            0             0             0          4         0             0              0               0    0       0              4           4      0    rd ex mr mw me de sd  [vdso]
ffffffffff600000 r-xp 00000000  00:00      0       4       0       0            0            0             0             0          0         0             0              0               0    0       0              4           4      0                   rd ex  [vsyscall]
                                             ======= ======= ======= ============ ============ ============= ============= ========== ========= ============= ============== =============== ==== ======= ============== =========== ======
                                             1338436 1133520 1127284         3024         3816             0       1126680    1124840   1127780       1042432              0               0    0       0            984         984      0 KB
root@ip-10-1-2-104:~#
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/pmap_AnonHugePages%20-%201.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/pmap_AnonHugePages%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/pmap_AnonHugePages%20-%203.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/pmap_AnonHugePages%20-%204.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/pmap_AnonHugePages%20-%205.png)


- 观察对比 `/proc/1360/smaps` 中的内容（`-XX` 的内容就是参照该文件得到的）

```
root@ip-10-1-2-104:~# cat /proc/1360/smaps
...
49a37000-4e8f7000 rw-p 00000000 00:00 0
Size:              80640 kB
Rss:               80640 kB
Pss:               80640 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:     80640 kB
Referenced:        80640 kB
Anonymous:         80640 kB
AnonHugePages:     77824 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd wr mr mw me ac sd
4e8f8000-5842b000 rw-p 00000000 00:00 0
Size:             158924 kB
Rss:              158924 kB
Pss:              158924 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:    158924 kB
Referenced:       158924 kB
Anonymous:        158924 kB
AnonHugePages:    157696 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd wr mr mw me ac sd
5842c000-756ad000 rw-p 00000000 00:00 0
Size:             477700 kB
Rss:              477700 kB
Pss:              477700 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:    477700 kB
Referenced:       477700 kB
Anonymous:        477700 kB
AnonHugePages:    475136 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd wr mr mw me ac sd
756be000-75ade000 rw-p 00000000 00:00 0
Size:               4224 kB
Rss:                4224 kB
Pss:                4224 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:      4224 kB
Referenced:         4224 kB
Anonymous:          4224 kB
AnonHugePages:      2048 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd wr mr mw me ac sd
...
```

需要重点关注：

- **Private_Dirty**:    477700 kB
- **Referenced**:       477700 kB
- **Anonymous**:        477700 kB
- **AnonHugePages**:    475136 kB


- 查看系统整体 HugePages 内存使用情况

```
root@ip-10-1-2-104:~# grep Huge /proc/meminfo
AnonHugePages:   6561792 kB    # THP
HugePages_Total:       0       # HP
HugePages_Free:        0       # HP
HugePages_Rsvd:        0       # HP
HugePages_Surp:        0       # HP
Hugepagesize:       2048 kB    # HP
root@ip-10-1-2-104:~#
```

可以看到 AnonHugePages 占用了 6G 内存；

> 其他参数含义如下：
> 
> - **HugePages_Total** 为所分配的页面总数目，该值和 Hugepagesize 相乘后得到所分配的内存大小；
> - **HugePages_Free** 为从来没有被使用过的 Hugepages 数目。即使 xxx 已经分配了这部分内存，但是如果没有实际写入，那么看到的还是 Free 的。这是很容易误解的地方；
> - **HugePages_Rsvd** 为已经被分配预留但是还没有使用的 page 数目。在 xxx 刚刚启动时，大部分内存应该都是 Reserved 并且 Free 的，随着 xxx 的使用，Reserved 和 Free 都会不断的降低；
> - **HugePages_Free** 减去 HugePages_Rsvd 这部分是没有被使用到的内存，如果没有其他的 xxx 实例 ，这部分内存也许永远都不会被使用到，也就是被浪费了。




- 可以看到 THP 确实是打开的

```
root@ip-10-1-2-104:~# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
root@ip-10-1-2-104:~#
```


----------


## 有价值的信息

### **HugePages 的产生背景**

Linux 操作系统进程是通过虚拟地址对内存进行访问，而 CPU 必须把它转换成实际的物理内存地址才能真正访问内存。CPU 通过缓存最近的虚拟内存地址和物理内存地址的映射关系来提高访问效率，并且保存在一个由 CPU 维护的映射表中。**为了尽量提高内存的访问速度，需要在映射表中保存尽量多的映射关系**。我们都知道，在传统的 LINUX 内存管理模式中，内存都是以页的形式进行分配的，默认每个内存页大小为 4K 。这就产生了一个问题，在原先的**百 G 左右**的内存下，可能整个映射表还在可控范围内，而当内存增加到 **T 级别**后，内存页依旧是 4K 大小，这就会导致整个映射表变得非常庞大，导致 CPU 的检索效率大大降低，而 HugePages 就是在这样一个环境下产生；

### **如何确定 HugePages 的 pagesize**

HugePages 是在 Linux2.6 内核中被引入的，提供 4k 的 pagesize 和比较大的 pagesize 的选择。HugePages 的大小取决于所使用的操作系统的内核版本以及不同的硬件平台；可以使用 `grep Hugepagesize /proc/meminfo` 来查看 HugePages 的大小；

### 使用 HugePages 时的注意事项

- HugePages 在操作系统启动后，如果没有管理员的介入，是**不会释放和改变**的，是**属于一种常驻内存**；
- HugePages 也**不会被交换**，也就是没有换入换出过程（page-in/page-out）。HugePages 一直被 pin 在内存中，这就要求我们不能把 HugePages 设置过大。
- 由于 AMM (Automatic Memory Management) 特性与 Hugepages 不兼容，如果希望使用 HugePages 需要禁用 AMM 特性；
- Hugepages 是**在分配后就会预留出来**的；

### Hugepages v.s. Transparent HugePages

Hugepages 在 `/proc/meminfo` 中是被独立统计的，与其它统计项不重叠，既不计入进程的 RSS/PSS 中，又不计入 LRU Active/Inactive ，也不会计入 cache/buffer 。如果进程使用了 Hugepages ，它的 RSS/PSS 不会增加；

不要把 Transparent HugePages (THP) 跟 Hugepages 搞混了，THP 的统计值是 `/proc/meminfo` 中的 `AnonHugePages` ，在 `/proc/<pid>/smaps` 中也有单个进程的统计，这个统计值与进程的 RSS/PSS 是有重叠的，如果用户进程用到了 THP ，进程的 RSS/PSS 也会相应增加，这与 Hugepages 是不同的；


### pmap 输出项的含义

```
root@ip-10-1-11-119:~# pmap -x 1310
1310:   nginx: worker process
Address           Kbytes     RSS   Dirty Mode  Mapping
...
0000000042e1b000       0       0       0 rw---   [ anon ]
00000000432bc000    2816    2816    2816 rw---   [ anon ]
00000000432bc000       0       0       0 rw---   [ anon ]
000000004357d000    4736    4736    4736 rw---   [ anon ]
000000004357d000       0       0       0 rw---   [ anon ]
0000000043a1e000    9728    9728    9728 rw---   [ anon ]
0000000043a1e000       0       0       0 rw---   [ anon ]
000000004439f000   24068   24068   24068 rw---   [ anon ]
000000004439f000       0       0       0 rw---   [ anon ]
0000000045b21000   29184   29184   29184 rw---   [ anon ]
0000000045b21000       0       0       0 rw---   [ anon ]
00000000483a2000   82436   82348   82348 rw---   [ anon ]
00000000483a2000       0       0       0 rw---   [ anon ]
00007fe8a3843000    5832    5828    5828 rw---   [ anon ]
00007fe8a3843000       0       0       0 rw---   [ anon ]
00007fe8a3df5000       8       0       0 r-x-- lsyslog.so
...
```

From man:

- **Address**: start address of map
- **Kbytes**: size of map in kilobytes
- **RSS**: resident set size in kilobytes
- **Dirty**: dirty pages (both shared and private) in kilobytes
- **Mode**: permissions on map: read, write, execute, shared, private (copy on write)
- **Mapping**: file backing the map, or '[ anon ]' for allocated memory, or '[ stack ]' for the program stack


### Memory 类型区分

Memory is either `private`, meaning it is exclusive to this process, or `shared`, meaning multiple processes may have it mapped and in use (think shared library code, etc.). Memory can also be `clean` - it hasn't been modified since it was loaded from disk or provided as zero-filled pages or whatever, and so if it needs to be freed to provide memory pages for other processes, it can just be discarded, and reloaded/re-filled if it is ever needed again - or `dirty`, which means if it needs to be freed up, it must be written out to a swap area so that the modified contents can be recovered when necessary.

It's not necessarily unusual to see large amounts of private dirty data in a process. **The problem is when the sum of all private dirty data across all processes in the system becomes a significant portion** (exact numbers depend greatly on your workload and acceptable performance) **of your overall physical memory, and stuff has to start being swapped in/out**..


----------

## 三种情况下 kong 的运行对比

- 三种情况下的输出对比

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%201.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%203.png)

- 监控对比

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%204%20-%20%E7%9B%91%E6%8E%A7.png)

- 未调整情况

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%205%20-%20%E6%9C%AA%E8%B0%83%E6%95%B4%E6%83%85%E5%86%B5.png)

- 关闭 THP 的情况

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%206%20-%20%E5%85%B3%E9%97%AD%20THP%20%E7%9A%84%E6%83%85%E5%86%B5.png)

- 升级 kong 版本_未关闭 THP

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/kong%20-%207%20-%20%E5%8D%87%E7%BA%A7%20kong%20%E7%89%88%E6%9C%AC_%E6%9C%AA%E5%85%B3%E9%97%AD%20THP.png)

## 遗留问题

- `/dev/zero` (deleted) 的含义（见[这里](https://gitee.com/moooofly/SecretGarden/blob/master/%E6%80%A7%E8%83%BD%E5%88%86%E6%9E%90%20--%20Mem%20%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5%E7%AF%87.md#devzero-deleted--tmpfs)）
- `pmap` 的展示项
- `/proc/meminfo` 中的输出（详见《[/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)》）
- `/proc/<pid>/smaps` 中的输出（见[这里](https://www.kernel.org/doc/Documentation/filesystems/proc.txt)）


参考：

- [/PROC/MEMINFO之谜](http://linuxperf.com/?p=142)
- [What does “Private_Dirty” memory mean in smaps?](https://stackoverflow.com/questions/17594183/what-does-private-dirty-memory-mean-in-smaps)
- [What does `Kbytes RSS Dirty` mean for pmap?](https://serverfault.com/questions/274460/what-does-kbytes-rss-dirty-mean-for-pmap/274472)
- [Cheat sheet: understanding the pmap(1) output](http://www.software-architect.net/blog/article/date/2015/07/03/cheat-sheet-understanding-the-pmap1-output.html)
- [Memory – Part 1: Memory Types](https://techtalk.intersec.com/2013/07/memory-part-1-memory-types/)
- [Memory – Part 2: Understanding Process memory](https://techtalk.intersec.com/2013/07/memory-part-2-understanding-process-memory/)
