# 系统分析 - 文件系统读写时 memory 使用分析

## 文件系统写测试

### 测试命令

```
root@backend-shared-stag-0:~# dd if=/dev/zero of=/tmp/BIGFILE bs=1024 count=10M
10485760+0 records in
10485760+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 89.3838 s, 120 MB/s
root@backend-shared-stag-0:~#
```

### 测试结论

- MemFree 减少 --> Cached 增长
    - Cached 增长 --> Active 增长
        - Active 增长 --> Active(file) 增长
- Dirty 先暴涨后变回初始值，最高值在 5600000 kB 左右；
- Writeback 先暴涨后变回初始值，最高达到 6000 kB/s 左右；
- Slab 增长 --> SReclaimable 增长
    - buffer_head 暴涨
    - radix_tree_node 暴涨
    - vm_area_struct 涨
- 重复文件系统写操作时，每次耗费时间都差不多（观察下来，应该是每次执行写命令时，会自动将 Cached/Active/Active(file)/Slab/SReclaimable/buffer_head/radix_tree_node 等内容 drop 掉，然后在重新执行一次）；

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/9%20-%20vmstat_free_meminfo%20%E8%BE%93%E5%87%BA%E5%AF%B9%E6%AF%94.png)

### 测试过程

#### /proc/meminfo 观察

- free pagecache, dentries and inodes

```
echo 3 > /proc/sys/vm/drop_caches
```

- 执行 dd 命令前，/proc/meminfo 的初始状态

```
Every 1.0s: cat /proc/meminfo                                    Wed May 23 13:43:16 2018

MemTotal:       32946256 kB
MemFree:        32326996 kB
MemAvailable:   32150868 kB
Buffers:            3540 kB
Cached:           288984 kB
SwapCached:            0 kB
Active:           169044 kB  - Active = Active(anon) + Active(file)
Inactive:         246136 kB
Active(anon):     128700 kB
Inactive(anon):   201740 kB
Active(file):      40344 kB
Inactive(file):    44396 kB
Unevictable:        8612 kB
Mlocked:            8612 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        131312 kB
Mapped:           143136 kB
Shmem:            201816 kB
Slab:              55788 kB
SReclaimable:      24752 kB
SUnreclaim:        31036 kB
KernelStack:        4768 kB
PageTables:         7936 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     944160 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     28672 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB
```

- dd 命令执行过程中（上升中）

Active = Active(anon) + Active(file)    
Active(file) 暴涨 => Active 暴涨

```
Every 1.0s: cat /proc/meminfo                                    Wed May 23 13:44:13 2018

MemTotal:       32946256 kB
MemFree:        25912712 kB   - 减少
MemAvailable:   32037624 kB   - 没变
Buffers:            4048 kB
Cached:          6532612 kB   -- 暴涨到 6G
SwapCached:            0 kB
Active:          6413232 kB   -- 暴涨
Inactive:         246200 kB
Active(anon):     128824 kB
Inactive(anon):   201740 kB
Active(file):    6284408 kB   -- 暴涨
Inactive(file):    44460 kB
Unevictable:        8612 kB
Mlocked:            8612 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:           5276176 kB   -- 暴涨
Writeback:          4696 kB   -- 基本维持在这个数值
AnonPages:        131384 kB
Mapped:           143200 kB
Shmem:            201836 kB
Slab:             226940 kB   -- 涨
SReclaimable:     195864 kB   -- 涨
SUnreclaim:        31076 kB
KernelStack:        4768 kB
PageTables:         7948 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     959684 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     28672 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB
```

- dd 命令执行过程中（下降中）

```
Every 1.0s: cat /proc/meminfo                                    Wed May 23 13:44:49 2018

MemTotal:       32946256 kB
MemFree:        21550612 kB
MemAvailable:   32037676 kB
Buffers:            4252 kB
Cached:         10775828 kB    -- 暴涨到 10G
SwapCached:            0 kB
Active:         10656548 kB    -- 暴涨到 10G
Inactive:         246260 kB
Active(anon):     128808 kB
Inactive(anon):   201740 kB
Active(file):   10527740 kB    -- 暴涨到 10G
Inactive(file):    44520 kB
Unevictable:        8612 kB
Mlocked:            8612 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:           5106636 kB    -- 暴涨
Writeback:          5256 kB    -- 基本维持在这个数值
AnonPages:        131384 kB
Mapped:           143200 kB
Shmem:            201852 kB
Slab:             345680 kB   -- 涨
SReclaimable:     314624 kB   -- 涨
SUnreclaim:        31056 kB
KernelStack:        4768 kB
PageTables:         7948 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     909828 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     28672 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB
```

- dd 执行完毕后（当 Writeback 为 0 时才对应 dd 执行结束）

```
Every 1.0s: cat /proc/meminfo                                       Wed May 23 13:45:53 2018

MemTotal:       32946256 kB
MemFree:        21548520 kB
MemAvailable:   32036836 kB
Buffers:            4772 kB
Cached:         10776572 kB
SwapCached:            0 kB
Active:         10657604 kB
Inactive:         246496 kB
Active(anon):     128860 kB
Inactive(anon):   201740 kB
Active(file):   10528744 kB
Inactive(file):    44756 kB
Unevictable:        8612 kB
Mlocked:            8612 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        131424 kB
Mapped:           143140 kB
Shmem:            201876 kB
Slab:             345588 kB
SReclaimable:     314636 kB
SUnreclaim:        30952 kB
KernelStack:        4752 kB
PageTables:         7916 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     924508 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     28672 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB
```

- free dentries and inodes

```
echo 2 > /proc/sys/vm/drop_caches
```

看到数值基本回到最初状态（貌似这里 echo 2 or 3 没差别，why）

```
Every 1.0s: cat /proc/meminfo                                       Wed May 23 14:03:49 2018

MemTotal:       32946256 kB
MemFree:        32323680 kB
MemAvailable:   32149420 kB
Buffers:            5264 kB
Cached:           290984 kB
SwapCached:            0 kB
Active:           172784 kB
Inactive:         246172 kB
Active(anon):     129228 kB
Inactive(anon):   201732 kB
Active(file):      43556 kB
Inactive(file):    44440 kB
Unevictable:        8420 kB
Mlocked:            8420 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        131152 kB
Mapped:           143144 kB
Shmem:            202284 kB
Slab:              56216 kB
SReclaimable:      25232 kB
SUnreclaim:        30984 kB
KernelStack:        4736 kB
PageTables:         7916 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     924140 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     28672 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB
```


#### slab 观察

执行命令

```
slabtop -s c -d 1
```

- 初始状态，即执行 `echo 3 > /proc/sys/vm/drop_caches` 后

```
 Active / Total Objects (% used)    : 202274 / 250846 (80.6%)
 Active / Total Slabs (% used)      : 6738 / 6738 (100.0%)
 Active / Total Caches (% used)     : 77 / 142 (54.2%)
 Active / Total Size (% used)       : 41167.69K / 55384.09K (74.3%)
 Minimum / Average / Maximum Object : 0.01K / 0.22K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 13748  11755  85%    0.55K    491       28      7856K inode_cache
 27615  13849  50%    0.19K   1315       21      5260K dentry
 37876  37276  98%    0.12K   1114       34      4456K kernfs_node_cache
  3960   1132  28%    1.05K    132       30      4224K ext4_inode_cache
  6860   2596  37%    0.57K    245       28      3920K radix_tree_node
  2528   2096  82%    1.00K     79       32      2528K kmalloc-1024
  4768   4768 100%    0.50K    149       32      2384K kmalloc-512
  1008    947  93%    2.00K     63       16      2016K kmalloc-2048
   495    445  89%    3.50K     55        9      1760K task_struct
  7920   7771  98%    0.20K    396       20      1584K vm_area_struct
 14586  10670  73%    0.10K    374       39      1496K buffer_head
 23296  18207  78%    0.06K    364       64      1456K kmalloc-64
   360    276  76%    4.00K     45        8      1440K kmalloc-4096
   180    163  90%    8.00K     45        4      1440K kmalloc-8192
  6447   6150  95%    0.19K    307       21      1228K kmalloc-192
   540    505  93%    2.05K     36       15      1152K idr_layer_cache
  1464   1237  84%    0.64K     61       24       976K shmem_inode_cache
   450    289  64%    2.06K     30       15       960K sighand_cache
  3584   2744  76%    0.25K    112       32       896K kmalloc-256
   810    650  80%    1.06K     27       30       864K signal_cache
  1196    920  76%    0.61K     46       26       736K proc_inode_cache
 21888  18534  84%    0.03K    171      128       684K kmalloc-32
 11900   8588  72%    0.05K    140       85       560K ftrace_event_field
  3968   3418  86%    0.12K    124       32       496K kmalloc-128
  5712   5712 100%    0.08K    112       51       448K anon_vma
  3360   2607  77%    0.09K     80       42       320K kmalloc-96
```

- 稳定后

```
 Active / Total Objects (% used)    : 2867206 / 2906345 (98.7%)
 Active / Total Slabs (% used)      : 75209 / 75209 (100.0%)
 Active / Total Caches (% used)     : 77 / 142 (54.2%)
 Active / Total Size (% used)       : 331460.64K / 342587.18K (96.8%)
 Minimum / Average / Maximum Object : 0.01K / 0.12K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
2632305 2632305 100%    0.10K  67495       39    269980K buffer_head    -- 暴涨
 44156  44156 100%    0.57K   1577       28     25232K radix_tree_node  -- 暴涨
 13748  11755  85%    0.55K    491       28      7856K inode_cache
 27615  13997  50%    0.19K   1315       21      5260K dentry
 37876  37276  98%    0.12K   1114       34      4456K kernfs_node_cache
  3960   1261  31%    1.05K    132       30      4224K ext4_inode_cache
  2528   2096  82%    1.00K     79       32      2528K kmalloc-1024
  4768   4768 100%    0.50K    149       32      2384K kmalloc-512
  1008    947  93%    2.00K     63       16      2016K kmalloc-2048
   495    445  89%    3.50K     55        9      1760K task_struct
  8200   8200 100%    0.20K    410       20      1640K vm_area_struct   -- 涨
 23296  19025  81%    0.06K    364       64      1456K kmalloc-64
   360    280  77%    4.00K     45        8      1440K kmalloc-4096
   180    163  90%    8.00K     45        4      1440K kmalloc-8192
  6447   6149  95%    0.19K    307       21      1228K kmalloc-192
   540    505  93%    2.05K     36       15      1152K idr_layer_cache
  1464   1237  84%    0.64K     61       24       976K shmem_inode_cache
   450    289  64%    2.06K     30       15       960K sighand_cache
  3584   2750  76%    0.25K    112       32       896K kmalloc-256
   810    650  80%    1.06K     27       30       864K signal_cache
  1196    920  76%    0.61K     46       26       736K proc_inode_cache
 21888  18534  84%    0.03K    171      128       684K kmalloc-32
 11900   8588  72%    0.05K    140       85       560K ftrace_event_field
  3968   3418  86%    0.12K    124       32       496K kmalloc-128
  5916   5916 100%    0.08K    116       51       464K anon_vma
```

变化点

> - buffer_head 暴涨：说明 blahblah
> - radix_tree_node 暴涨：说明 blahblah
> - vm_area_struct 涨：说明 blahblah

- 恢复到初始状态

```
echo 3 > /proc/sys/vm/drop_caches
```

```
 Active / Total Objects (% used)    : 208676 / 257502 (81.0%)
 Active / Total Slabs (% used)      : 6915 / 6915 (100.0%)
 Active / Total Caches (% used)     : 77 / 142 (54.2%)
 Active / Total Size (% used)       : 42087.59K / 57054.17K (73.8%)
 Minimum / Average / Maximum Object : 0.01K / 0.22K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 14840  12739  85%    0.55K    530       28      8480K inode_cache
 27867  12932  46%    0.19K   1327       21      5308K dentry
  4290   1272  29%    1.05K    143       30      4576K ext4_inode_cache
 37876  37276  98%    0.12K   1114       34      4456K kernfs_node_cache
  7308   2603  35%    0.57K    261       28      4176K radix_tree_node
  2528   2096  82%    1.00K     79       32      2528K kmalloc-1024
  4768   4768 100%    0.50K    149       32      2384K kmalloc-512
  1008    947  93%    2.00K     63       16      2016K kmalloc-2048
   495    425  85%    3.50K     55        9      1760K task_struct
 16692  10600  63%    0.10K    428       39      1712K buffer_head
  8180   7969  97%    0.20K    409       20      1636K vm_area_struct
 23232  19171  82%    0.06K    363       64      1452K kmalloc-64
   360    268  74%    4.00K     45        8      1440K kmalloc-4096
   180    163  90%    8.00K     45        4      1440K kmalloc-8192
  6447   6183  95%    0.19K    307       21      1228K kmalloc-192
   540    505  93%    2.05K     36       15      1152K idr_layer_cache
  1464   1237  84%    0.64K     61       24       976K shmem_inode_cache
   450    289  64%    2.06K     30       15       960K sighand_cache
  3584   3147  87%    0.25K    112       32       896K kmalloc-256
   810    704  86%    1.06K     27       30       864K signal_cache
  1300   1047  80%    0.61K     50       26       800K proc_inode_cache
 22528  21691  96%    0.03K    176      128       704K kmalloc-32
 11900   8588  72%    0.05K    140       85       560K ftrace_event_field
  6834   6798  99%    0.08K    134       51       536K anon_vma
  3968   3418  86%    0.12K    124       32       496K kmalloc-128

```

#### /proc/buddyinfo 观察

> 背景知识：
> 
> - 需要了解 NUMA 内存架构，以及 node 和 zone 概念
> - 需要了解 DMA/DMA32/Normal 的差别和关系
> - 需要了解 buddy 工作原理
> - 需要了解 buddy 和 slab 的关系

初始

```
Every 1.0s: cat /proc/buddyinfo            Thu May 24 15:16:38 2018

Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32   3289   2490   2406   1608   1079    694    448    267    131     58    728
Node 0, zone   Normal  16913  12243  15454  12201   8631   6105   3901   1883    661    339   5635

```

稳定后

```
Every 1.0s: cat /proc/buddyinfo            Thu May 24 15:19:40 2018

Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32     89     32     10     24     10      4      6      5      7      3    613
Node 0, zone   Normal     91    145    260     47     17      7      3      1     13     25   4622

```

#### vmstat -s 输出

初始

```
Every 1.0s: vmstat -s                                                                                                                                         Thu May 24 15:17:33 2018

     32946256 K total memory
       267740 K used memory
       167204 K active memory      -- 暴涨
       243956 K inactive memory
     32331800 K free memory
         3720 K buffer memory
       342996 K swap cache      -- 暴涨
            0 K total swap
            0 K used swap
            0 K free swap
      1630634 non-nice user cpu ticks
         2806 nice user cpu ticks
       383440 system cpu ticks
   9400605468 idle cpu ticks
       177408 IO-wait cpu ticks
            0 IRQ cpu ticks
         9926 softirq cpu ticks
       153114 stolen cpu ticks
     32726740 pages paged in
    125687980 pages paged out      -- 暴涨
            0 pages swapped in
            0 pages swapped out
   1444962845 interrupts
   1726322566 CPU context switches
   1515388832 boot time
      1090045 forks

```

稳定后

```
Every 1.0s: vmstat -s                                                                                                                                         Thu May 24 15:20:03 2018

     32946256 K total memory
       269892 K used memory
     10655280 K active memory
       244196 K inactive memory
     21551868 K free memory
         4220 K buffer memory
     11120276 K swap cache
            0 K total swap
            0 K used swap
            0 K free swap
      1630754 non-nice user cpu ticks
         2806 nice user cpu ticks
       384390 system cpu ticks
   9400715250 idle cpu ticks
       185809 IO-wait cpu ticks
            0 IRQ cpu ticks
         9926 softirq cpu ticks
       153132 stolen cpu ticks
     32727204 pages paged in
    133682192 pages paged out
            0 pages swapped in
            0 pages swapped out
   1445087777 interrupts
   1726426495 CPU context switches
   1515388832 boot time
      1091397 forks

```

## 文件系统读测试

### 测试命令

```
root@backend-shared-stag-0:~# dd if=/tmp/BIGFILE of=/dev/null bs=1024
10485760+0 records in
10485760+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 84.7525 s, 127 MB/s
root@backend-shared-stag-0:~#
（不清除 cache 内容的情况下）
root@backend-shared-stag-0:~# dd if=/tmp/BIGFILE of=/dev/null bs=1024
10485760+0 records in
10485760+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 4.10822 s, 2.6 GB/s
root@backend-shared-stag-0:~#
```


### 测试结论

- MemFree 减少 --> Cached 增长
    - Cached 增长 --> Inactive 增长
        - Inactive 增长 --> Inactive(file) 增长
- Slab 增长 --> SReclaimable 增长
    - radix_tree_node 暴涨
    - vm_area_struct 涨
- 重复文件系统读操作时，若不清除 cache ，则能够大幅提升速度；
    

### 测试过程

#### /proc/meminfo 观察

- dd 执行前（`echo 3 > /proc/sys/vm/drop_caches`）

```
Every 1.0s: cat /proc/meminfo                                       Wed May 23 16:07:12 2018

MemTotal:       32946256 kB
MemFree:        32322776 kB
MemAvailable:   32145904 kB
Buffers:            1996 kB
Cached:           289552 kB   -- 暴涨
SwapCached:            0 kB
Active:           166548 kB
Inactive:         247432 kB   -- 暴涨
Active(anon):     128432 kB
Inactive(anon):   203800 kB
Active(file):      38116 kB
Inactive(file):    43632 kB   -- 暴涨
Unevictable:       12272 kB
Mlocked:           12272 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                16 kB
Writeback:             0 kB
AnonPages:        134720 kB
Mapped:           143344 kB
Shmem:            203832 kB
Slab:              57552 kB   -- 涨
SReclaimable:      26252 kB   -- 涨
SUnreclaim:        31300 kB
KernelStack:        4736 kB
PageTables:         7964 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     931416 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     30720 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB

```

- dd 过程中

```
Every 1.0s: cat /proc/meminfo                                       Wed May 23 16:09:48 2018

MemTotal:       32946256 kB
MemFree:        29236896 kB
MemAvailable:   32099988 kB
Buffers:            2000 kB
Cached:          3370900 kB
SwapCached:            0 kB
Active:           166768 kB
Inactive:        3328772 kB
Active(anon):     128644 kB
Inactive(anon):   203800 kB
Active(file):      38124 kB
Inactive(file):  3124972 kB
Unevictable:       12272 kB
Mlocked:           12272 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        134940 kB
Mapped:           143408 kB
Shmem:            203860 kB
Slab:              61916 kB
SReclaimable:      30684 kB
SUnreclaim:        31232 kB
KernelStack:        4752 kB
PageTables:         7992 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     948520 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     30720 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB

```

- 稳定后

```
Every 1.0s: cat /proc/meminfo                                       Wed May 23 16:12:03 2018

MemTotal:       32946256 kB
MemFree:        21813012 kB
MemAvailable:   32091560 kB
Buffers:            3312 kB
Cached:         10776456 kB
SwapCached:            0 kB
Active:           168816 kB
Inactive:       10733476 kB
Active(anon):     128580 kB
Inactive(anon):   203800 kB
Active(file):      40236 kB
Inactive(file): 10529676 kB
Unevictable:       12272 kB
Mlocked:           12272 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        134852 kB
Mapped:           143344 kB
Shmem:            203888 kB
Slab:              79176 kB
SReclaimable:      47964 kB
SUnreclaim:        31212 kB
KernelStack:        4736 kB
PageTables:         7964 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     949548 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     30720 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      163840 kB
DirectMap2M:     9273344 kB
DirectMap1G:    25165824 kB

```

#### slab 观察

初始状态

```
 Active / Total Objects (% used)    : 211849 / 257334 (82.3%)
 Active / Total Slabs (% used)      : 6917 / 6917 (100.0%)
 Active / Total Caches (% used)     : 77 / 142 (54.2%)
 Active / Total Size (% used)       : 42623.83K / 56817.17K (75.0%)
 Minimum / Average / Maximum Object : 0.01K / 0.22K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 14840  12739  85%    0.55K    530       28      8480K inode_cache
 27615  13384  48%    0.19K   1315       21      5260K dentry
 37876  37381  98%    0.12K   1114       34      4456K kernfs_node_cache
  3930   1098  27%    1.05K    131       30      4192K ext4_inode_cache
  6832   2435  35%    0.57K    244       28      3904K radix_tree_node   -- 暴涨
  2528   2112  83%    1.00K     79       32      2528K kmalloc-1024
  4800   4800 100%    0.50K    150       32      2400K kmalloc-512
  1008    947  93%    2.00K     63       16      2016K kmalloc-2048
   495    425  85%    3.50K     55        9      1760K task_struct
  8740   8740 100%    0.20K    437       20      1748K vm_area_struct    -- 涨
 15912  10987  69%    0.10K    408       39      1632K buffer_head
 23232  19668  84%    0.06K    363       64      1452K kmalloc-64
   360    280  77%    4.00K     45        8      1440K kmalloc-4096
   180    163  90%    8.00K     45        4      1440K kmalloc-8192
  6447   6199  96%    0.19K    307       21      1228K kmalloc-192
   540    505  93%    2.05K     36       15      1152K idr_layer_cache
  1872   1644  87%    0.61K     72       26      1152K proc_inode_cache
  1464   1237  84%    0.64K     61       24       976K shmem_inode_cache
   450    289  64%    2.06K     30       15       960K sighand_cache
  3712   3082  83%    0.25K    116       32       928K kmalloc-256
   810    704  86%    1.06K     27       30       864K signal_cache
 22528  21691  96%    0.03K    176      128       704K kmalloc-32
  7242   7242 100%    0.08K    142       51       568K anon_vma
 11900   8588  72%    0.05K    140       85       560K ftrace_event_field
  3968   3418  86%    0.12K    124       32       496K kmalloc-128
```

稳定后

```
 Active / Total Objects (% used)    : 255155 / 294977 (86.5%)
 Active / Total Slabs (% used)      : 8257 / 8257 (100.0%)
 Active / Total Caches (% used)     : 77 / 142 (54.2%)
 Active / Total Size (% used)       : 66984.82K / 78421.03K (85.4%)
 Minimum / Average / Maximum Object : 0.01K / 0.27K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 44044  44044 100%    0.57K   1573       28     25168K radix_tree_node
 14840  12739  85%    0.55K    530       28      8480K inode_cache
 27615  14602  52%    0.19K   1315       21      5260K dentry
 37876  37381  98%    0.12K   1114       34      4456K kernfs_node_cache
  3930   1098  27%    1.05K    131       30      4192K ext4_inode_cache
  2528   2112  83%    1.00K     79       32      2528K kmalloc-1024
  4800   4800 100%    0.50K    150       32      2400K kmalloc-512
  1008    947  93%    2.00K     63       16      2016K kmalloc-2048
   495    425  85%    3.50K     55        9      1760K task_struct
  8360   8360 100%    0.20K    418       20      1672K vm_area_struct
 15912  11101  69%    0.10K    408       39      1632K buffer_head
  2600   2471  95%    0.61K    100       26      1600K proc_inode_cache
 23232  19719  84%    0.06K    363       64      1452K kmalloc-64
   360    280  77%    4.00K     45        8      1440K kmalloc-4096
   180    163  90%    8.00K     45        4      1440K kmalloc-8192
  6447   6199  96%    0.19K    307       21      1228K kmalloc-192
   540    505  93%    2.05K     36       15      1152K idr_layer_cache
  1464   1237  84%    0.64K     61       24       976K shmem_inode_cache
   450    289  64%    2.06K     30       15       960K sighand_cache
  3744   2898  77%    0.25K    117       32       936K kmalloc-256
   810    704  86%    1.06K     27       30       864K signal_cache
 22528  21691  96%    0.03K    176      128       704K kmalloc-32
  7293   7293 100%    0.08K    143       51       572K anon_vma
 11900   8588  72%    0.05K    140       85       560K ftrace_event_field
  3968   3418  86%    0.12K    124       32       496K kmalloc-128

```

#### /proc/buddyinfo 观察


初始

```
Every 1.0s: cat /proc/buddyinfo             Wed May 23 16:46:38 2018

Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32   3230   3558   2622   1710   1203    785    485    287    157     76    699
Node 0, zone   Normal  16792  18297  16358  12665   8591   6357   4021   1957    725    413   5536

```

稳定后

```
Every 1.0s: cat /proc/buddyinfo             Wed May 23 16:48:57 2018

Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32    128    167     54     99     72     43     27     23     12      4    612
Node 0, zone   Normal   1230   1041    188    552    497    309    189    106     61     26   4617

```


其他

```
root@backend-shared-stag-0:~# echo m > /proc/sysrq-trigger
root@backend-shared-stag-0:~# dmesg -T
...
[Wed May 23 16:58:12 2018] sysrq: SysRq : Show Memory
[Wed May 23 16:58:12 2018] Mem-Info:
[Wed May 23 16:58:12 2018] active_anon:32678 inactive_anon:51073 isolated_anon:0
                            active_file:10481 inactive_file:2632612 isolated_file:0
                            unevictable:3068 dirty:0 writeback:0 unstable:0
                            slab_reclaimable:11869 slab_unreclaimable:7827
                            mapped:35844 shmem:51192 pagetables:2005 bounce:0
                            free:5452280 free_pcp:2069 free_cma:0
[Wed May 23 16:58:12 2018] Node 0 DMA free:15904kB min:32kB low:40kB high:48kB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB isolated(anon):0kB isolated(file):0kB present:15988kB managed:15904kB mlocked:0kB dirty:0kB writeback:0kB mapped:0kB shmem:0kB slab_reclaimable:0kB slab_unreclaimable:0kB kernel_stack:0kB pagetables:0kB unstable:0kB bounce:0kB free_pcp:0kB local_pcp:0kB free_cma:0kB writeback_tmp:0kB pages_scanned:0 all_unreclaimable? yes
[Wed May 23 16:58:12 2018] lowmem_reserve[]: 0 3745 32158 32158 32158
[Wed May 23 16:58:12 2018] Node 0 DMA32 free:2561416kB min:7792kB low:9740kB high:11688kB active_anon:15840kB inactive_anon:21312kB active_file:5168kB inactive_file:1208288kB unevictable:1312kB isolated(anon):0kB isolated(file):0kB present:3915776kB managed:3835116kB mlocked:1312kB dirty:0kB writeback:0kB mapped:18408kB shmem:21364kB slab_reclaimable:4380kB slab_unreclaimable:3952kB kernel_stack:608kB pagetables:1104kB unstable:0kB bounce:0kB free_pcp:4356kB local_pcp:112kB free_cma:0kB writeback_tmp:0kB pages_scanned:0 all_unreclaimable? no
[Wed May 23 16:58:12 2018] lowmem_reserve[]: 0 0 28413 28413 28413
[Wed May 23 16:58:12 2018] Node 0 Normal free:19231800kB min:59756kB low:74692kB high:89632kB active_anon:114872kB inactive_anon:182980kB active_file:36756kB inactive_file:9322160kB unevictable:10960kB isolated(anon):0kB isolated(file):0kB present:29622272kB managed:29095236kB mlocked:10960kB dirty:0kB writeback:0kB mapped:124968kB shmem:183404kB slab_reclaimable:43096kB slab_unreclaimable:27356kB kernel_stack:4176kB pagetables:6916kB unstable:0kB bounce:0kB free_pcp:3920kB local_pcp:136kB free_cma:0kB writeback_tmp:0kB pages_scanned:0 all_unreclaimable? no
[Wed May 23 16:58:12 2018] lowmem_reserve[]: 0 0 0 0 0
[Wed May 23 16:58:12 2018] Node 0 DMA: 0*4kB 0*8kB 0*16kB 1*32kB (U) 2*64kB (U) 1*128kB (U) 1*256kB (U) 0*512kB 1*1024kB (U) 1*2048kB (M) 3*4096kB (M) = 15904kB
[Wed May 23 16:58:12 2018] Node 0 DMA32: 194*4kB (UME) 152*8kB (UE) 56*16kB (UM) 94*32kB (UE) 72*64kB (UE) 43*128kB (UME) 27*256kB (UME) 22*512kB (UE) 12*1024kB (UME) 4*2048kB (UME) 612*4096kB (ME) = 2561416kB
[Wed May 23 16:58:12 2018] Node 0 Normal: 1316*4kB (UME) 1046*8kB (UME) 194*16kB (UM) 535*32kB (UME) 498*64kB (UME) 309*128kB (UME) 188*256kB (UE) 106*512kB (UME) 60*1024kB (UE) 25*2048kB (UE) 4617*4096kB (ME) = 19231552kB
[Wed May 23 16:58:12 2018] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=1048576kB
[Wed May 23 16:58:12 2018] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=2048kB
[Wed May 23 16:58:12 2018] 2695777 total pagecache pages
[Wed May 23 16:58:12 2018] 0 pages in swap cache
[Wed May 23 16:58:12 2018] Swap cache stats: add 0, delete 0, find 0/0
[Wed May 23 16:58:12 2018] Free swap  = 0kB
[Wed May 23 16:58:12 2018] Total swap = 0kB
[Wed May 23 16:58:12 2018] 8388509 pages RAM
[Wed May 23 16:58:12 2018] 0 pages HighMem/MovableOnly
[Wed May 23 16:58:12 2018] 151945 pages reserved
[Wed May 23 16:58:12 2018] 0 pages cma reserved
[Wed May 23 16:58:12 2018] 0 pages hwpoisoned
root@backend-shared-stag-0:~#
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/6%20-%20sysrq%20%E6%9F%A5%E7%9C%8B%20memory%20%E4%BF%A1%E6%81%AF.png)


#### vmstat -s 输出

初始

```
Every 1.0s: vmstat -s                                                                                                                                         Wed May 23 16:47:01 2018

     32946256 K total memory
       275060 K used memory
       170564 K active memory
       248472 K inactive memory   -- 暴涨
     32319372 K free memory
         3440 K buffer memory
       348384 K swap cache     -- 暴涨
            0 K total swap
            0 K used swap
            0 K free swap
      1616766 non-nice user cpu ticks
         2806 nice user cpu ticks
       375851 system cpu ticks
   9335839914 idle cpu ticks
       169438 IO-wait cpu ticks
            0 IRQ cpu ticks
         9782 softirq cpu ticks
       151891 stolen cpu ticks
     22077184 pages paged in   -- 暴涨
    125649144 pages paged out
            0 pages swapped in
            0 pages swapped out
   1432593262 interrupts
   1711981461 CPU context switches
   1515388832 boot time
       968887 forks
```

稳定后

```
Every 1.0s: vmstat -s                                                                                                                                         Wed May 23 16:49:30 2018

     32946256 K total memory
       274280 K used memory
       170764 K active memory
     10734308 K inactive memory
     21812976 K free memory
         3452 K buffer memory
     10855548 K swap cache
            0 K total swap
            0 K used swap
            0 K free swap
      1616830 non-nice user cpu ticks
         2806 nice user cpu ticks
       376056 system cpu ticks
   9335949966 idle cpu ticks
       176855 IO-wait cpu ticks
            0 IRQ cpu ticks
         9782 softirq cpu ticks
       151918 stolen cpu ticks
     32563028 pages paged in
    125649144 pages paged out
            0 pages swapped in
            0 pages swapped out
   1432724248 interrupts
   1712187015 CPU context switches
   1515388832 boot time
       970220 forks

```