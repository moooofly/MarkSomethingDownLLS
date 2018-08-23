# 性能分析 -- Mem 基础概念篇

## 基本概念

已覆盖

- RSS & VSS & PSS
- virtual memory
- memory page type
- memory cache & disk cache
- kswapd & pdflush
- anonymous pages & file-backed pages
- active pages & inactive pages
- overcommit
- drop_caches
- swappiness
- paging & swapping
- PFRA
- OOM Killer
- hugepage & transparent hugepage


### RSS & VSS & PSS

- RSS: the amount of the mapping that is currently resident in RAM (RSS), 
- PSS: the process' proportional share of this mapping (PSS). The "proportional set size" (PSS) of a process is the count of pages it has in memory, where each page is divided by the number of processes sharing it.


### virtual memory

- Virtual memory 使用 disk 作为 RAM 的扩展；用作 virtual memory 的那部分 disk 被称作 swap space ；
- Virtual memory 按 pages 进行划分；在 X86 架构上，每一个 virtual memory page 为 4KB 大小；

### memory cache & disk cache

`cache` is a high-speed access area that can be a reserved section of main memory or on a storage device. The two main types of cache are `memory cache` and `disk cache`.

- `memory cache` is a portion of the high-speed **static RAM** (SRAM) and is effective because most programs access the same data or instructions repeatedly. By keeping as much of this information as possible in SRAM, the computer avoids accessing the slower DRAM, making the computer perform faster and more efficiently. Today, most computers come with L3 cache or L2 cache, while older computers included only L1 cache. 
- Like memory caching, `disk caching` is used to access commonly accessed data. However, instead of using high-speed SRAM, a disk cache uses conventional main memory. The most recently accessed data from a disk is stored in a memory buffer. When a program needs to access data from the disk, it first checks the disk cache to see if the data is there. Disk caching can dramatically improve the performance of applications because accessing a byte of data in RAM can be thousands of times faster than accessing a byte of data on a hard drive.

### memory page type

从 PFRA 的角度：

- **Unreclaimable** – locked, kernel, reserved pages
- **Swappable** – anonymous memory pages
- **Syncable** – pages backed by a disk file
- **Discardable** – static pages, discarded pages

从 Linux kernel 的角度：

- **Read Pages** – These are pages of data read in via disk (MPF) that are read only and backed on disk. These pages exist in the Buffer Cache and include static files, binaries, and libraries that do not change. The Kernel will continue to page these into memory as it needs them. If memory becomes short, the kernel will "steal" these pages and put them back on the free list causing an application to have to MPF to bring them back in.
- **Dirty Pages** – These are pages of data that have been modified by the kernel while in memory. These pages need to be synced back to disk at some point using the pdflush daemon. In the event of a memory shortage, kswapd (along with pdflush) will write these pages to disk in order to make more room in memory.
- **Anonymous Pages** – These are pages of data that do belong to a process, but do not have any file or backing store associated with them. They can't be synchronized back to disk. In the event of a memory shortage, kswapd writes these to the swap device as temporary storage until more RAM is free ("swapping" pages).

从用户进程的角度：：

- file-backed pages ：对应与文件关联的内存页，比如程序文件、数据文件所对应的内存页；
- anonymous pages ：对应与文件无关的内存页，比如进程的堆栈，用 malloc 申请的内存；
- file-backed pages 在发生 paging 时，将从其对应的文件读入或写出；
- anonymous pages 在发生 swapping 时，是对 swapping device 或 swapping file 进行读/写操作；


### active pages & inactive pages

查看命令

```
root@backend-shared-stag-0:~# grep -i active /proc/meminfo
Active:         10806044 kB
Inactive:         469976 kB
Active(anon):      94352 kB
Inactive(anon):   207456 kB
Active(file):   10711692 kB
Inactive(file):   262520 kB
root@backend-shared-stag-0:~#
```

从源码中可知

```
fs/proc/meminfo.c:
==================
0023 static int meminfo_proc_show(struct seq_file *m, void *v)
0024 {
...

0032         unsigned long pages[NR_LRU_LISTS];
...

0051         for (lru = LRU_BASE; lru < NR_LRU_LISTS; lru++)
0052                 pages[lru] = global_page_state(NR_LRU_BASE + lru);
...

0095                 "Active:         %8lu kB\n"
0096                 "Inactive:       %8lu kB\n"
0097                 "Active(anon):   %8lu kB\n"
0098                 "Inactive(anon): %8lu kB\n"
0099                 "Active(file):   %8lu kB\n"
0100                 "Inactive(file): %8lu kB\n"
...

0148                 K(pages[LRU_ACTIVE_ANON]   + pages[LRU_ACTIVE_FILE]),
0149                 K(pages[LRU_INACTIVE_ANON] + pages[LRU_INACTIVE_FILE]),
0150                 K(pages[LRU_ACTIVE_ANON]),
0151                 K(pages[LRU_INACTIVE_ANON]),
0152                 K(pages[LRU_ACTIVE_FILE]),
0153                 K(pages[LRU_INACTIVE_FILE]),
...
```

在 Linux 中 pages 可能处于三种状态：

- free
- active
- inactive

其中

- LRU_INACTIVE_ANON 和 LRU_ACTIVE_ANON 用来管理 anonymous pages ；
- LRU_INACTIVE_FILE 和 LRU_ACTIVE_FILE 用来管理 file-backed pages ；
- Active Memory (LRU_ACTIVE_ANON + LRU_ACTIVE_FILE) ：表示刚被访问过的内存页面；
- Inactive Memory (LRU_INACTIVE_ANON + LRU_INACTIVE_FILE) ：表示那些被应用程序映射着，但是长时间不用的内存页面；

背景信息：

- `vmstat` 命令直接从 `/proc/meminfo` 中获取的数据，`/proc/meminfo` 的数据是在 `fs/proc/meminfo.c` 中的 `meminfo_proc_show` 函数中生成的，用于统计所有的 LRU list ；
- LRU list 是 Linux kernel 的内存页框回收算法（Page Frame Reclaiming Algorithm, PFRA）所使用的数据结构；
- Linux kernel 设计了两种 LRU list ：active list 和 inactive list ；
- 内核线程 kswapd 会周期性地把 active list 中符合条件的页面移到 inactive list 中，这项转移工作是由 `refill_inactive_zone()` 完成的；之后从 inactive list 回收页面就变得简单了。


![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/LRU_list_active_inactive.png)

> 参考：《[解读VMSTAT中的ACTIVE/INACTIVE MEMORY](http://linuxperf.com/?p=97)》


### PFRA

- PFRA 负责释放内存；
- PFRA 根据 memory page 类型选择要释放的内存页；除了 “unreclaimable” 之外的其它类型 pages 均可以通过 PFRA 进行回收； 
- PFRA 主要基于如下两个函数进行内存回收，即
    - `kswapd` 内核线程
    - “Low On Memory Reclaiming” function ；

### pdflush & kswapd

- 差别：kswapd 的作用是**内存管理**，pdflush 的作用是**缓存管理**（同步内存和磁盘）；
- kswapd 试图保证内存永远都是可满足用户要求的；pdflush 试图保证内存和磁盘的数据是同步的；
- pdflush 负责将 file-backed page 同步到 disk 上；
- pdflush 的同步行为受内核参数 `vm.dirty_background_ratio` 的影响；
- 大多数情况下，pdflush 独立于 PFRA 工作；即 kernel 触发 LMR 算法，LMR 会通过 pdflush 将 dirty pages 刷出，同时也会调用其它 page 释放程序；
- kswapd 的回收目标对象：
    - anonymous pages ，即属于某个进程但是又和任何文件无关联，不能被同步到硬盘上，内存不足的时候由 kswapd 负责将它们写到交换分区并释放内存；
    - file-backed pages ，即与文件相关的 pages（和 pdflush 有重叠的地方）；
- 当内存不足时候，kswapd 和 pdflush 会共同负责把数据写回硬盘并释放内存；


### paging & swapping

- paging 是指将 memory 按照一定的时间间隔、同步（synching）到 disk 的过程；在某些时间点上，kernel 必须扫描（scan）memory 并回收（reclaim）不再使用的 pages ，以便其能够分配给其它应用使用；
- Kernel 通过 pdflush 进行 paging ；
- paging 是一种常规活动，不要和 swapping 搞混淆；
- paging 包括 page in 和 page out 两种形式；
- 有些时候你可能会运行超过 memory 可支持数量的应用程序（进程），在这种情况下，kernel 会将使用较少的 process 搬移到一个 a slower area ，例如 disk 上；该行为被称作 `swapping` ；swapping 的意义在于允许你能够运行超过所需物理内存的程序数量；


### swappiness

> 详见《[swappiness](https://github.com/moooofly/MarkSomethingDown/blob/8c35004dc517aa303131b0dd9fd34773bc05a528/Linux/swappiness.md)》


### dirty & writeback

查看命令

```
root@backend-shared-stag-0:~# grep -E -i "dirty|writeback" /proc/meminfo
Dirty:                40 kB
Writeback:             0 kB
WritebackTmp:          0 kB
root@backend-shared-stag-0:~#
```

相关参数

```
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 10
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 20
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200
```


- `dirty_ratio`: the number of pages at which a process which is generating disk writes will itself start writing out dirty data.
- `dirty_expire_centisecs`: This tunable is used to define when dirty data is old enough to be eligible for writeout by the kernel flusher threads.  It is expressed in 100'ths of a second.  Data which has been dirty in-memory for longer than this interval will be written out next time a flusher thread wakes up.
- `dirty_writeback_centisecs`: The kernel flusher threads will periodically wake up and write 'old' data out to disk.  This tunable expresses the interval between those wakeups, in 100'ths of a second. Setting this to zero disables periodic writeback altogether.
- `dirty_background_ratio`: the number of pages at which the background kernel flusher threads will start writing out dirty data.



### drop_caches

系统参数

```
vm.drop_caches = 0
vfs_cache_pressure = 100
```

- Writing to drop_caches will cause the kernel to drop clean caches, as well as reclaimable slab objects like dentries and inodes.
- This is a non-destructive operation and will not free any dirty objects.
- use outside of a testing or debugging environment is not recommended.
- `vfs_cache_pressure`: This percentage value controls the tendency of the kernel to reclaim the memory which is used for caching of directory and inode objects. At the default value of vfs_cache_pressure=100 the kernel will attempt to reclaim dentries and inodes at a "fair" rate with respect to pagecache and swapcache reclaim.


> 详见《[linux 系统调优之 drop_caches](https://github.com/moooofly/MarkSomethingDown/blob/e73ab424a126720b75591bd85a724d00181fddbb/Linux/linux%20%E7%B3%BB%E7%BB%9F%E8%B0%83%E4%BC%98%E4%B9%8B%20drop_caches.md)》


### overcommit

查看命令

```
root@backend-shared-stag-0:~# grep -i commit /proc/meminfo
CommitLimit:    16473128 kB
Committed_AS:     882040 kB
root@backend-shared-stag-0:~#
```

系统参数

```
vm.nr_overcommit_hugepages = 0
vm.overcommit_kbytes = 0
vm.overcommit_memory = 0
vm.overcommit_ratio = 50
```

- Memory Overcommit 的意思是操作系统承诺给进程的内存大小超过了实际可用的内存；
- 按照 UNIX/Linux 的算法，物理内存页的分配发生在使用的瞬间，而不是在申请的瞬间；
- commit 或 overcommit **针对的是内存申请**，内存申请**不等于内存分配**，内存只在实际用到的时候才分配；
- Linux 是允许 memory overcommit 的，只要你来申请内存我就给你，寄希望于进程实际上用不到那么多内存；万一真用到那么多，Linux 设计了一个 OOM killer 机制来处理这种危机：挑选一个进程出来杀死，以腾出部分内存，如果还不够就继续杀；也可通过设置内核参数 `vm.panic_on_oom` 使得发生 OOM 时自动重启系统。这都是有风险的机制，重启有可能造成业务中断，杀死进程也有可能导致业务中断；所以 Linux 2.6 之后允许通过内核参数 `vm.overcommit_memory` 禁止 memory overcommit ；
- 内核参数 `vm.overcommit_memory` 接受三种取值：
    - 0 -- **Heuristic overcommit handling**. 这是缺省值，它允许 overcommit ，但过于明目张胆的 overcommit 会被拒绝，比如 malloc 一次性申请的内存大小就超过了系统总内存。Heuristic 的意思是“试探式的”，内核利用某种算法猜测你的内存申请是否合理，它认为不合理就会拒绝 overcommit ；Heuristic overcommit 算法：单次申请的内存大小不能超过 `free memory + free swap + pagecache 的大小 + SLAB 中可回收的部分`，否则本次申请就会失败；
    - 1 -- **Always overcommit**. 允许 overcommit ，对内存申请来者不拒；
    - 2 -- **Don’t overcommit**. 禁止 overcommit ；
- 禁止 overcommit (vm.overcommit_memory=2) 的触发前提为申请的内存总数超过了 overcommit 的阈值，即 `grep -i commit /proc/meminfo` 输出的 `CommitLimit` 值；
    - 未使用 huge pages ，计算公式为 `CommitLimit = (Physical RAM * vm.overcommit_ratio / 100) + Swap`
    - 使用了 huge pages ，则需要从物理内存中减去，计算公式变为 `CommitLimit = ([total RAM] – [total huge TLB RAM]) * vm.overcommit_ratio / 100 + swap`

> 参考《[理解LINUX的MEMORY OVERCOMMIT](http://linuxperf.com/?p=102)》


`/proc/meminfo` 中的参数说明：

- `Committed_AS` 表示所有进程**已经申请**的内存总大小（注意是已经申请的，**不是已经分配**的），如果 `Committed_AS` 超过 `CommitLimit` 就表示发生了 overcommit，超出越多表示 overcommit 越严重；
- `Committed_AS` 的含义换一种说法就是，如果要绝对保证不发生 OOM (out of memory) 需要多少物理内存；
- `CommitLimit` %lu (since Linux 2.6.10) -- This is **the total amount of memory currently available to be allocated on the system**, expressed in kilobytes. This limit is **adhered to only if** strict overcommit accounting is enabled (mode 2 in  `/proc/sys/vm/overcommit_memory`).
- `Committed_AS` %lu -- **The amount of memory presently allocated on the system**. The committed memory is **a sum of all** of the memory which has been allocated by **processes**, even if it has not been "used" by them as of yet. With strict overcommit enabled on the system (mode 2 in IR `/proc/sys/vm/overcommit_memory`), allocations which would exceed the `CommitLimit` will not be permitted. This is useful if one needs to guarantee that processes will not fail due to lack of memory once that memory has been successfully allocated.

通过 sar 命令查看

```
root@backend-shared-stag-0:~# sar -r 1 3
Linux 4.4.0-72-generic (backend-shared-stag-0)  05/22/2018  _x86_64_    (8 CPU)

04:35:37 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
04:35:38 PM  24722428   8223828     24.96    284632   7197604    864216      2.62   4583508   2996612        32
04:35:39 PM  24722428   8223828     24.96    284632   7197604    864216      2.62   4583508   2996612        32
04:35:40 PM  24722428   8223828     24.96    284632   7197604    864216      2.62   4583508   2996612        32
Average:     24722428   8223828     24.96    284632   7197604    864216      2.62   4583508   2996612        32
root@backend-shared-stag-0:~#
```

- 通过 `sar -r` 查看内存使用状况时，输出结果中有两项与 overcommit 有关：`kbcommit` 和 `%commit` ；
- `kbcommit` 对应 `/proc/meminfo` 中的 Committed_AS；
- `%commit` 的计算公式并没有采用 `CommitLimit` 作分母，而是 `Committed_AS/(MemTotal+SwapTotal)` ，意思是“内存申请占物理内存与交换区之和的百分比”。


### buddy & node & zone

`/proc/buddyinfo` 信息如下

```
root@backend-shared-stag-0:~# cat /proc/buddyinfo
Node 0, zone      DMA      0      0      0      1      2      1      1      0      1      1      3
Node 0, zone    DMA32    403     48     65     42     20     11      6      5      6      9    596
Node 0, zone   Normal    467    432   1208    384     96     42      8      2     20     61   4503
root@backend-shared-stag-0:~#
```

`/proc/pagetypeinfo` 信息如下

```
root@backend-shared-stag-0:~# cat /proc/pagetypeinfo
Page block order: 9
Pages per block:  512

Free pages count per migrate type at order       0      1      2      3      4      5      6      7      8      9     10
Node    0, zone      DMA, type    Unmovable      0      0      0      1      2      1      1      0      1      0      0
Node    0, zone      DMA, type      Movable      0      0      0      0      0      0      0      0      0      1      3
Node    0, zone      DMA, type  Reclaimable      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone      DMA, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone      DMA, type          CMA      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone      DMA, type      Isolate      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone    DMA32, type    Unmovable     64     34     30     20      9      5      4      4      6      1      0
Node    0, zone    DMA32, type      Movable    961     76     35     20     14      5      2      2      0      2    519
Node    0, zone    DMA32, type  Reclaimable     14      8      2      0      0      1      0      0      0      7     19
Node    0, zone    DMA32, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone    DMA32, type          CMA      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone    DMA32, type      Isolate      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone   Normal, type    Unmovable    468    103     65     24     35     16      7      2     10     16      0
Node    0, zone   Normal, type      Movable   7657    244     59     38     34     22     20      8      6      5   3921
Node    0, zone   Normal, type  Reclaimable      2     11      5      1      1      0      0      0      0     38    149
Node    0, zone   Normal, type   HighAtomic      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone   Normal, type          CMA      0      0      0      0      0      0      0      0      0      0      0
Node    0, zone   Normal, type      Isolate      0      0      0      0      0      0      0      0      0      0      0

Number of blocks type     Unmovable      Movable  Reclaimable   HighAtomic          CMA      Isolate
Node 0, zone      DMA            1            7            0            0            0            0
Node 0, zone    DMA32           14         1830           68            0            0            0
Node 0, zone   Normal          102        13856          506            0            0            0
root@backend-shared-stag-0:~#
```


`/proc/zoneinfo` 信息如下

```
root@backend-shared-stag-0:~# cat /proc/zoneinfo | grep -A 4 "Node "
Node 0, zone      DMA
  pages free     3976
        min      8
        low      10
        high     12
--
Node 0, zone    DMA32
  pages free     619169
        min      1948
        low      2435
        high     2922
--
Node 0, zone   Normal
  pages free     4660342
        min      14939
        low      18673
        high     22408
root@backend-shared-stag-0:~#
```

其中

- buddy system 通过 node 进行管理
    - DMA zone
    - DMA32 zone
    - Normal zone


相关内核参数

- `min_free_kbytes`: This is used to force the Linux VM to keep a minimum number of kilobytes free.  The VM uses this number to compute a watermark[WMARK_MIN] value for each lowmem zone in the system. Each lowmem zone gets a number of reserved free pages based proportionally on its size.
- `lowmem_reserve_ratio`: The tunable determines how aggressive the kernel is in defending these lower zones.


Linux 的最底层的内存分配算法叫做 Buddy 算法，它以 2 的 n 次方页为单位对空闲内存进行管理。当然并不是每一次用户空间申请内存都会引起底层的内存分配，slab 算法就可以在 buddy 算法的基础上对内存进行二次管理，分配更小的内存空间；Buddy 算法的优点是避免了内存的外部碎片，但是长期运行后，大片的内存会比较少，而 1 页，2 页，4 页这种内存会非常多，当我们分配大片连续内存的时候就会出问题；在 linux 系统中，我们可以通过 `/proc/buddy` 文件来查看当前系统空闲的连续内存空间剩余情况。



### slab

查看命令

```
root@backend-shared-stag-0:~# grep -E -i "slab|reclaim" /proc/meminfo
Slab:             499060 kB
SReclaimable:     457472 kB
SUnreclaim:        41588 kB
root@backend-shared-stag-0:~#
```

内核参数

```
vm.min_slab_ratio = 5
```

> 参考《[陈延伟：任督二脉之内存管理总结笔记](https://mp.weixin.qq.com/s/VpieQfuaRqimHpxVUglZ6w)》

Linux 的最底层，由 Buddy 算法管理着所有的空闲页面，最小单位是 2 的 0 次方页，就是 1 页，4K ，但是很多时候，我们为一个结构分配空间，也只需要几十个字节，按页分配无疑是浪费空间；另外，当我们频繁的分配和释放一个结构，我们希望在释放的时候，这部分内存不要立刻还给 Buddy ，而是提供一种类似 C 库的管理机制，在下一次在分配的时候还可以拿到同一块内存且保留着基本的数据结构。基于上面两点，Slab 应运而生。总结一下，Slab 主要提供以下两个功能：

- 对从 Buddy 拿到的内存进行二次管理，以更小的单位进行分配和回收（注意，是回收而不是释放），防止了空间的浪费；
- 让频繁使用的对象尽量分配在同一块内存区间并保留基本数据结构，提高程序效率；


> 参考《[怎样诊断SLAB泄露问题](http://linuxperf.com/?p=148)》

- Slab allocator 是 Linux kernel 的内存分配机制，各内核子系统、模块、驱动程序都可以使用；但用完应该记得释放，忘记释放就会造成“内存泄露”；
- 通过观察 `/proc/meminfo` 中 MemTotal/MemFree/SwapTotal/SwapFree/Slab/SReclaimable/SUnreclaim 的数值相对关系，确认内存占用是否和 swap 或 slab 相关；
- Linux kernel 自 2.6.23 之后就已经从 Slab 进化成 Slub 了，它自带原生的诊断功能，比 Slab 更方便。判断kernel是否在使用 slub 有一个简单的方法，就是看 `/sys/kernel/slab` 目录是否存在，如果存在的话就是 slub ，否则就是 slab ；
    - 如果是 slab 问题诊断，可以
        - 利用 debug kernel 的 slab leak 辅助功能，需要 kernel 编译时打开 `CONFIG_DEBUG_SLAB_LEAK` 选项才行，默认是没打开的；
        - 利用 systemtap 等工具，对内核适当的位置植入探针；
    - 如果是 slub 问题诊断，则参考《[如何诊断SLUB问题](http://linuxperf.com/?p=184)》


> 参考《[如何诊断SLUB问题](http://linuxperf.com/?p=184)》

- 无论是 slab 还是 slub ，可能会出现的问题无非是以下几类：
    - **内存泄露**（leak），alloc 之后忘了 free，导致内存占用不断增长；
    - **越界**（overrun），访问了 alloc 分配的区域之外的内存，覆盖了不属于自己的数据；
    - **使用已经释放的内存**（use after free），正常情况下，已经被 free 释放的内存是不应该再被读写的，否则就意味着程序有 bug ；
    - **使用未经初始化的数据**（use uninitialised bytes），缺省模式下 alloc 分配的内存是不被初始化的，内存值是随机的，直接使用的话后果可能是灾难性的；


`/proc/meminfo` 中的参数说明：

- `Slab` %lu -- In-kernel data structures cache.
- `SReclaimable` %lu (since Linux 2.6.19) -- Part of Slab, that might be reclaimed, such as caches.
- `SUnreclaim` %lu (since Linux 2.6.19) -- Part of Slab, that cannot be reclaimed on memory pressure.

补充说明：

- 可回收的 slab objects 有 dentries 和 inodes ；
- A `dentries` is a data structure that represents a **directory**.
- An `inode` in your context is a data structure that represents a **file**.


### OOM Killer

procfs 下的相关文件

```
/proc/<pid>/oom_adj
/proc/<pid>/oom_score
/proc/<pid>/oom_score_adj
```

sysctl 相关参数

```
vm.oom_dump_tasks = 1
vm.oom_kill_allocating_task = 0
vm.panic_on_oom = 0
```

以下内容取自《[内存管理二](https://unordered.org/content/58b6015c61800000)》

当我们使用 malloc 分配内存时，系统并没有真正的分配内存，而是采用欺骗性的手段，拖延分配内存的时机，防止无谓的内存消耗，只有在我们写入的时候，产生 page fault 才会拿到真实的内存，且是写多少才分配多少，那么，问题来了，当我们通过 malloc 获取一片内存并成功返回，然后开始逐步使用内存，系统也逐步的分配真实的内存给进程，但这个过程中，另外一个耗内存的程序快速的拿走所有的内存，导致我的进程在逐步写入的过程发现刚刚说好给我的内存现在没有了，这种情况就是 OOM ，out of memory！

那么有没有什么办法可以调整进程的 oom score ，就算这个进程比较耗内存，但是在 OOM 时候，这个进程仍然不会干掉。

系统提供两个接口文件给用户：

- `/proc/pid/oom_adj`：可配置范围是 -17 到 15 ，设置为 15 ，oom score 最大，最容易被干掉，设置 -16 ，oom score 最小，设置 -17 为禁止使用 OOM 杀死该进程。
- `/proc/pid/oom_score_adj`：oom score 会加上这个值，也可以设置负数，但如果负数的绝对值大于 oom score ，oom score 最小为 0 。


一个案例（取自《[怎样避免MYSQLD被OOM-KILLER杀死？](http://linuxperf.com/?p=94)》）

- 当出现内存紧张的情况时，oom-killer 会自动选择合适的进程牺牲掉，oom-killer 会通过比较每个进程的 oom_score 来挑选要出局的进程，**数值越大就越容易被选中**，而手工调整 oom_score 是通过 oom_score_adj 来实现的，例如 `echo "-100" > /proc/<pid>/oom_score_adj` ；但这种修改，重启后会失效；
- 如何修改 systemd 的服务脚本：
    - `/usr/lib/systemd/system/xxx.service` 就是目标文件，可以直接修改，但不建议这么做，因为将来升级 xxx 有可能会覆盖掉你的改动；
    - 更好的方式是：创建一个新目录 `/etc/systemd/system/xxx.service.d/`，把需要改动的内容放在该目录下的`.conf` 文件里，文件名可以随便，但必须以 `.conf` 结尾（实际情况似乎并不完全相同）。这个文件起的作用是对 systemd 的 service 文件做出补充；
    - 在 systemd 相关配置文件中，命令必须使用全路径，因为 systemd 不提供设置好的 PATH 环境变量，例如 `ExecStartPost=/bin/sudo /usr/local/bin/oom_mysql.sh`；
- 如何设置不用输入密码而且没有 tty 的 sudo ：
    - 修改 sudo 配置的命令是 visudo ，如果不加 -f 参数，它默认修改配置文件 `/etc/sudoers` ，但直接改动 `/etc/sudoers` 不太稳妥，万一改坏了什么地方就有麻烦，所以推荐的方法是：把你要增加的 sudoers 配置放在 `/etc/sudoers.d/` 目录下的文件中，文件名可以随意。



### hugepage & transparent hugepage

查看命令

```
root@backend-shared-stag-0:~# grep -i huge /proc/meminfo
AnonHugePages:     28672 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
root@backend-shared-stag-0:~#
```

系统参数

```
vm.hugepages_treat_as_movable = 0
vm.nr_hugepages = 0
vm.nr_hugepages_mempolicy = 0
vm.nr_overcommit_hugepages = 0
```

#### Transparent HugePages (THP)

- "AnonHugePages" shows the ammount of memory backed by transparent hugepage.
- AnonHugePages 统计的是 Transparent HugePages (THP)，THP 与 Hugepages 不是一回事，区别很大。在 `/proc/<pid>/smaps` 中也有单个进程的统计，这个统计值与进程的 RSS/PSS 是有重叠的，如果用户进程用到了 THP ，进程的 RSS/PSS 也会相应增加，这与 Hugepages 是不同的；
- THP 也可以用于 shared memory 和 tmpfs ，缺省是禁止的，打开的方法如下（详见 https://www.kernel.org/doc/Documentation/vm/transhuge.txt）：
    - mount 时加上 "huge=always" 等选项
    - 通过 `/sys/kernel/mm/transparent_hugepage/shmem_enabled` 来控制
- 缺省情况下 shared memory 和 tmpfs 不使用 THP ；

#### Hugepages

Hugepages 在 `/proc/meminfo` 中是被独立统计的，与其它统计项不重叠，既不计入进程的 RSS/PSS 中，又不计入 LRU Active/Inactive，也不会计入 cache/buffer 。如果进程使用了 Hugepages ，它的 RSS/PSS 不会增加；

HugePages_Total 对应内核参数 `vm.nr_hugepages` ，也可以在运行中的系统上直接修改 `/proc/sys/vm/nr_hugepages`，修改的结果会立即影响空闲内存 MemFree 的大小，因为 HugePages 在内核中独立管理，只要一经定义，无论是否被使用，都不再属于 free memory ；

用户程序在申请 Hugepages 的时候，其实是 reserve 了一块内存，并未真正使用，此时 `/proc/meminfo` 中的 HugePages_Rsvd 会增加，而 HugePages_Free 不会减少；等到用户程序真正读写 Hugepages 的时候，它才被消耗掉了，此时 HugePages_Free 会减少，HugePages_Rsvd 也会减少；

使用 Hugepages 有三种方式：(详见 https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)

- mount 一个特殊的 hugetlbfs 文件系统，在上面创建文件，然后用 `mmap()` 进行访问，如果要用 `read()` 访问也是可以的，但是 `write()` 不行；
- 通过 `shmget`/`shmat` 也可以使用 Hugepages ，调用 shmget 申请共享内存时要加上 SHM_HUGETLB 标志；
- 通过 `mmap()`，调用时指定 MAP_HUGETLB 标志也可以使用 Huagepages ；

其他信息详见：《[线上 kong 内存占用问题排查](https://gitee.com/moooofly/SecretGarden/blob/master/%E7%BA%BF%E4%B8%8A%20kong%20%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5.md#%E6%9C%89%E4%BB%B7%E5%80%BC%E7%9A%84%E4%BF%A1%E6%81%AF)》


### major fault & minor fault

- 阻塞当前进程的缺页异常处理称为主缺页 (major falut)，也称为大缺页；
- 不会阻塞进程的缺页，称为次缺页 (minor fault)，也称为小缺页；
- 被访问的页还没有被放在任何一个页框中，内核分配一新的页框并适当初始化来满足调用请求，这种情况称为 Demand Paging ；


### /dev/zero (deleted) & tmpfs

On Linux, a **shared anonymous map** is actually `file-based`. The kernel creates a file in a `tmpfs` (an instance of `/dev/zero`). **The file is immediately unlinked so it cannot be accessed by any other processes unless they inherited the map (by forking)**. This is quite clever since the sharing is done through the file layer the same way it’s done for **shared file-backed mappings**.

tmpfs 是一个基于内存的文件系统，提供可以接近零延迟的快速存储区域。


## 系统文件

There are system-wide files in `/proc` :

- `/proc/buddyinfo` -- This file contains information which is used for diagnosing memory fragmentation issues.
- `/proc/zoneinfo` -- This file display information about memory zones.
- `/proc/pagetypeinfo` -- Additional page allocator information, more information relevant to external fragmentation can be found in pagetypeinfo.
- `/proc/vmstat` -- This file displays various virtual memory statistics.
- `/proc/vmallocinfo` -- Provides information about vmalloced/vmaped areas.
- `/proc/meminfo` -- This file reports statistics about memory usage on the system.
- `/proc/slabinfo` -- Information about kernel caches.
- `/proc/[pid]/stat` -- Status information about the process.
- `/proc/[pid]/statm` -- Provides information about memory usage, measured in pages (exposes some memory related statistics).
- `/proc/[pid]/status` -- Provides much of the information in `/proc/[pid]/stat` and `/proc/[pid]/statm` in a format that's easier for humans to parse.
- `/proc/[pid]/maps` -- a simple list of mappings (The source of pmap data)
- `/proc/[pid]/smaps` -- a more detailed version with a paragraph per mapping (The source of pmap data)


## 杂七杂八

- **虚拟内存**作为一种逻辑层，处于应用程序请求与硬件内存管理单元（MMU）之间；
- **虚拟内存子系统**的主要部分为**虚拟地址空间**概念，进程所使用的内存地址即**进程虚拟地址空间**中的**虚拟内存地址**；
- **虚拟内存子系统**必须解决的一个主要问题事**内存碎片问题**；Linux 上的内核内存分配器（KMA）通过在**伙伴系统**之上采用 **slab 分配算法**解决该问题；
- 基于**请求调页**（demand paging）的内存分配策略，进程可以在它的页还没有在内存时就开始执行；当进程真正访问该页时，再由 MMU 中的分页单元引发**缺页异常**，进而完成空闲页的分配和初始化；
- 内核分配给进程的虚拟地址空间有一下内存区组成：
    - 程序的代码段（text）
    - 程序的初始化数据段（data）
    - 程序的未初始化数据段（bss）
    - 初始程序栈（stack）
    - 所引用共享库的可执行代码和数据（.so 的 text + data + bss）
    - 堆（heap）
- 内核通过一组**页表**（page table）数据结构来映射线性地址和物理地址之间的对应关系；页表存放在主存中；
- DRAM 的划分：
    - 一部分专门（永久的）用于存放内核代码和内核静态数据结构（0～16MB 中的一部分）
    - 其余部分称作动态内存（dynamic memory），由**虚拟内存子系统**管理
        - 满足内核对缓冲区、描述符及其他动态内核数据结构的请求；
        - 满足进程对一般内存区的请求及对文件内存映射的请求；
        - 借助于高速缓存（cache）从磁盘及其他缓冲设备获得较好的性能；
- MMU 由**分段单元**和**分页单元**构成；
    - 分段单元负责将**逻辑地址**转换为**线性地址**；线性地址被划分成固定长度的**页**（page）；
    - 分页单元负责将**线性地址**转换为**物理地址**；物理地址被划分成固定长度的**页框**（page frame），即**物理页**；
- **硬件高速缓存**（L1/L2/L3 cache）是为了缩小 CPU 和 RAM 之间的速度不匹配问题而存在的；理论基础为**局部性原理**；物理实现上对应片上 SRAM ；
- 硬件高速缓存位于分页单元和主存之间；
- 多处理器系统的每一个处理器都有一个单独的硬件高速缓存，因此需要额外的硬件电路用于保持内容同步；
- 硬件高速缓存相关的性能指标有
    - cache hit
    - cache miss
- Linux 实现中，对于所有的页框都启用高速缓存，对于写操作总是采用**回写**（write-back）策略；回写方式只更新高速缓存行，不改变 RAM 的内容，提供了更快的速度；在回写结束后，通过其他方式触发 RAM 的最终更新；
- 转换后援缓冲期（TLB）也是一种高速缓存，用于加快线性地址的转换；在多处理器系统中，每个 CPU 都有自己的 TLB ；这些 TLB 上的内容不存在需要同步的问题；
- 在 NUMA 内存模型中，给定 CPU 对不同内存单元的访问时间是不同的；
- 在 NUMA 内存模型中，物理内存首先按照**节点**（node）进行划分，每个节点中的物理内存又被划分为几个**管理区**（zone）；在每个管理区内，通过使用不同的**伙伴系统**进行页框的管理；
- 由于实际的计算机体系结构存在硬件上的制约，因此，Linux 将每个内存节点划分成 3 个管理区：
    - ZONE_DMA：包含 0~16MB 的内存页框；
    - ZONE_NORMAL：包含 16MB~896MB 的内存页框；
    - ZONE_HIGHMEM：包含 >896MB 的内存页框；
- Linux 采用伙伴系统算法来解决**外碎片**问题；通过 slab 分配器模式解决**内碎片**问题；
- slab 分配器将**对象**（object）分组（按 slab 大小分组）放进**高速缓存**（cache）中，高速缓存是主存的一部分；
- 磁盘高速缓存（disk cache）包括：
    - 目录项高速缓存（directory cache）：存放描述文件系统路径名的目录项对象；
    - 索引高速缓存（inode cache）：存放描述磁盘索引节点的索引节点对象；
    - 页高速缓存（page cache）：一种对完整的数据也进行操作的磁盘高速缓存；
- 在绝大多数情况下，内核在读写磁盘时都引用页高速缓存；几乎所有的文件读和写操作都依赖于页高速缓存，只有在 O_DIRECT 标志被置位而进程打开文件的情况下才会出现例外；
- 页高速缓存中的每个页所包含的数据肯定属于某个文件；
- 页高速缓存中的页的类型：
    - 含有普通文件数据的页；
    - 含有目录的页；
    - 含有直接从块设备文件（跳过文件系统层）读出的数据的页；
    - 含有用户态进程数据的页，但页中的数据已经被交换到磁盘；
    - 属于特殊文件系统文件的页；
- 页高速缓存的核心数据结构是 address_space 对象；为了实现页高速缓存的高效查找，Linux 2.6 使用了大量的搜索树，其中每个 address_space 对象对应一颗搜索树；address_space 对象的 page_tree 字段是基树（radix tree）的根；基树上的每个节点由 radix_tree_node 数据结构表示；
- 在 Linux 内核的旧版本中，主要有两种不同磁盘高速缓存：
    - 页高速缓存（page cache）：用来存放访问磁盘文件内容时生成的磁盘数据页；
    - 缓冲区高速缓存（buffer cache）：通过 VFS 访问的块的内容保留在内存中，因此也被称作“**块缓冲区**”；
- 从 2.4.10 的稳定版本开始，缓冲区高速缓存其实就不存在了；而是在页高速缓存中划分出叫做**缓冲区页**的区域，用来作为之前的缓冲区高速缓存；
- 缓冲区页在形式上就是与称作“缓冲区首部”的附加描述符相关的数据页，其主要目的是快速确定页中的一个块在磁盘中的位置；
- 每个块缓冲区都有 buffer_head 类型的缓冲区首部描述符；
- 内核不断用包含块设备数据的页填充页高速缓冲区，只要进程修改了数据，相应的页就被标记为脏页；
- Unix 系统允许把脏缓冲区写入块设备的操作延迟执行；
- 一个脏页可能直到系统关闭时，都一直逗留在主存之中；
- 把脏页刷新（写入）到磁盘的条件：
    - 页高速缓存变得太满，但还需要更多的页，或者脏页的数量已经太多；
    - 自从页变成脏页以来已经过去太长时间；
    - 进程请求对块设备或者特定文件任何待定的变化都进程刷新（通过 sync()/fsync()/fdatasync() 系统调用来实现）
- 用户应用程序把脏缓冲区刷新到磁盘会使用的三个系统调用：
    - sync()：允许进程把所有的脏缓冲区刷新到磁盘
    - fsync()：允许进程把属于特定打开文件的所有块刷新到磁盘；
    - fdatasync()：于 fsync() 非常相似，但不刷新文件的索引节点块；
- 按照 Linux 的设计，迟早所有空闲内存都将被分配给进程和高速缓存；因此，Linux 通过页框回收算法（PFRA）采取从用户态进程和内核高速缓存“窃取”页框的办法补充伙伴系统的空闲块列表；
- 所谓“映射页”是指该页映射了一个文件的某个部分，映射页差不多都是可同步的；
- 所谓“匿名页”是指它属于一个进程的某匿名线性区（如进程的用户态堆和栈中的所有页），为回收页框，内核必须将页中内容保存到一个专门的磁盘分区或磁盘文件上，称作**交换区**；所有匿名页都是可交换的；
- 通常，特殊文件系统中的页是不可回收的，唯一的例外是 tmpfs 特殊文件系统的页。它可以被保存在交换区后被回收；
- PFRA 存在几个入口：
    - 内存紧缺回收
    - 睡眠回收
    - 周期回收
        - kswapd 内核线程：用于检查某个 zone 中空闲页框数是否已低于 pages_high 的值，然后从 LRU inactive list 中回收页；
        - events 内核线程（cache_reap 函数）：用于回收 slab 分配器处理的位于内存高速缓存中的所有空闲（未用） slab ；
- 属于进程用户态地址空间或页高速缓存的所有页被分为两组：
    - 活动 LRU 链表 active list
    - 非活动 LRU 链表 inactive list
- 页的 active list 和 inactive list 是 PFRA 的核心数据结构；PFRA 必须从 inactive list 中窃取页；