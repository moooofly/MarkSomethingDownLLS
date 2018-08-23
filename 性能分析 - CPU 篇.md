# 性能分析 - CPU 篇

## Prerequisites

### CPU Utilization 和 CPU Load

详见《[CPU Utilization 和 CPU Load Average](https://github.com/moooofly/MarkSomethingDown/blob/e5fb6bf5c489647fbac34265a038ea7870849b02/Linux/CPU%20Utilization%20%E5%92%8C%20CPU%20Load.md)》

key notes：

- CPU Utilization 如何计算
- per-process CPU Utilization 如何计算
- per-thread CPU Utilization 如何计算
- CPU Load Average 都包括哪些内容

### CPU softirq 和 CPU IRQ

- 发生中断时会打断当前“指令流水线”去处理中断；
- 中断花费时间越长，代码被打断程度越严重；
- 中断越频繁，一般来说上下文切换越频繁（每次中断都要发生很多切换）；
- 中断越频繁，中断消耗的 CPU 时间越长；
- IRQ 多说明被物理设备打断的次数越多；一般来说，主要是由于网卡上包量非常大导致（网卡 pps 越高，中断数越大，导致软硬中断所要求的 CPU 处理时间越长，导致性能下降）；因为在服务器上一般不会有那么多其它硬件事件发生；
- IRQ 达到一定值后一定会导致用户进程调度被延迟，没有办法避免；
需要关注中断发生在哪颗 CPU 上；大部分中断都是发生在 0 核心上，50% 左右；
- 目前使用 Intel 网卡（替代了原来的 Broadcom 网卡）为多队列网卡，即能中断到多少 CPU 核心就中断到多少；但事实上，并不会全部占用（一般为前xx个）；
- CPU 节能功能会导致 Linux 在调度时尽量将所有任务往一个 CPU 核心上调度；副作用就是可能导致你的目标任务发生调度延迟；
- 包量导致的延迟计算公式：100w pps --> 中断 100w 次 --> 延迟 50ms（每次中断的上下文切换时间成本为 50ns）
- 基于 DMA 解决小包问题只是优化策略，并不解决根本问题；
- pps 上升，也会造成 softirq 的上升，虽然内核调度策略会尽量将 softirq 往空核上调度，但空核不足时，一样处理不了；
- 网卡限制：缓冲区大小限制，最低 8k ，最高不确定；缓冲区大小影响 DMA 聚合的极限，超过极限会导致丢包；

### CPU 调度器

调度器类型

- RT
- O(1)
- CFS

调度器策略：

- RR
- FIFO
- NORMAL
- BATCH

调度器主要功能：

- 分时：可运行线程之间的多任务，优先执行最高优先级任务；
- 抢占：一旦有高优先级线程变为可运行状态，调度器能够抢占当前运行的线程，这样较高优先级的线程可以马上开始运行；
- 负载均衡：把可运行的线程移到空闲或者不繁忙的 CPU 队列中；

其他：

- voluntary_ctxt_switches 和 nonvoluntary_ctxt_switches
- 静态优先级和动态优先级


### Throughput & Latency

从业务目标角度来说，通常的瓶颈问题是指**吞吐量（Throughput）**和**时延（Latency）**；而吞吐量和时延是相互影响的两个因素，**当吞吐量达到系统上限，时延就会大幅提高**；

而 Throughput & Latency 的问题又和队列理论有关；

### 队列理论

- 最简模型

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AE%BF%E9%97%AE%E5%BB%B6%E6%97%B6%20-%201.jpg)

- 单队列（简化模型）

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AE%BF%E9%97%AE%E5%BB%B6%E6%97%B6%20-%202.jpg)

- 多队列（实际模型）

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AE%BF%E9%97%AE%E5%BB%B6%E6%97%B6%20-%203.jpg)


## 经验之谈

- **如果 CPU 占用率没有到 100%** ，latency 会稳定在 `len*ti` （对于现代 CPU 来说，作为一个多核多部件系统，latency 等于路径上所有队列的 `len[i]*t[i]` 的和）；len 其实就等于接收线程（或者中断）一次收入的请求数。**如果 CPU 达到 100%** ，队列的长度就无限增加，latency 也会跟着无限增加。当 Throughput 达到上限的时候，latency 会无限增加，直到发生流控；因此，一般先看 CPU 是否满载，满载就优化 CPU ，没有满载就找流控点，通过流控点调整 latency 和 Throughput ；
- 在多核的情况下，CPU 无法满载还会有一个原因，就是业务线程没有办法在多个 CPU 上展开，这需要通过增加处理线程来实现；
- 一般**看系统**是从“模块”的角度来看的，但**看系统的性能模型**，是从线程的角度来看的；从线程模型上看性能，处理模型就会比较简单：我们如果希望提高一个队列清库存的效率，只要增加在这个队列上的搬移线程，或者提高部署在上面的线程（实际上是 CPU 核）的数量，就可以平衡它的流控时间。如果能平衡所有队列的效率和长度，就比较容易控制整个系统的 Throughput 和 latency ；
- **如果有 CPU 占用率很高**，觉得不合理但又不知道原因，这时就要靠 perf 来画像了，先查基于时间的 perf 分布看看问题，最好顺便通过 perf-archive 打包相应的数据用于后续分析；
- 数据库查询缓慢（触发慢查询）要从其所花费的 CPU 时间、文件系统和所执行的磁盘 I/O 方面来考察最好；
- 执行大量计算的应用程序，更多的负载由线程承担，当 CPU 接近 100% 使用率时，由于 CPU 调度延时增加，性能开始下降；当性能接近峰值后，在 100% 使用率时，Throughput 已经开始随着更多线程的增加而下降，导致更多的上下文切换，这会消耗 CPU 资源，实际完成的任务会变少；

## 方法论

CPU 分析和调优的方法：

- 工具法：能用啥用啥
- USE 方法：针对每个 CPU 检查使用率、饱和度和错误；
- 负载特征归纳法：多用于前期准备基础数据，以便问题排查时的对照；
- 剖析+跟踪：抓取用户态+内核态的 CPU 用量数据进行分析；


----------


- 当遭遇一个 **Throughput 性能瓶颈（即“上不去了”）**的时候，通常可以按如下步骤来发现瓶颈位置：
    - **如果 CPU 没有满**，有三种可能
        - 某个队列提早流控了
            - 观察所有可能丢包的点，根据需要进行调整：
                - 网卡队列
                - per-CPU queue
                - socket buffer
            - 观察业务自身实现的队列水位
        - 调度没有充分展开（业务模型实现问题）
        - 除上面两种情况外，一种值得怀疑的高阶问题点为队列使用不饱和问题（除非有极高的性能要求，否则优化到这一步 ROI 很低）  
    - **如果 CPU 已经满**，则观察 CPU 时间花在业务进程上的比例比例是否合理（需要历史经验数据）
        - 如果比例不合理，则使用各种工具进行分析
            - 可用信息统计工具：top/vmstat/mpstat/sar/pidstat/
            - 可用剖析和跟踪工具：perf/systemtap/ftrace
        - 如果比例合理，则可以
            - 分析 CPU 是否还能够进一步被充分利用（比如基于 Top Down 模型进行分析）；
            - 分析软件算法是否可以进一步优化；可以通过 perf 和 ftrace 组合功能来分析；

> - 基于 perf 分析问题原因；可以看到 CPU 时间分布情况，例如当只有 < 20% 的 CPU 时间落在主业务流程上，则应该有大量的时间花在了锁和调度上；
> - 基于 ftrace 分析问题原因：当发现调度特别频繁的时候，可以通过 ftrace 观察每次切换的原因；


## 工具说明

信息统计工具

- uptime 检查负载变化情况
- vmstat 检查 CPU 整体使用情况
- mpstat 检查单个 CPU 的使用情况
- top/htop 常规检查
- pidstat 按进程/线程给出 CPU 使用情况
- sar 啥都能干
- /usr/bin/time 使用查看一些概况信息（排查问题时用处不大，测试和调试时有些价值） 
- cpustat 移植于 Solaris ，功能有些差异（可以不考虑使用）

bcc 中的工具

- TODO

剖析+跟踪工具

- perf
- ftrace
- systemtap


其他东东

- lscpu
- atop
- /proc/cpuinfo
- /proc/stat
- /proc/softirqs
- /proc/interrupts

### 工具差异比较

```
# dd if=/dev/zero of=/dev/null bs=1024 count=100M
104857600+0 records in
104857600+0 records out
107374182400 bytes (107 GB, 100 GiB) copied, 30.799 s, 3.5 GB/s
```

#### vmstat

```
root@backend-shared-stag-0:~# vmstat -w 1
procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 0  0            0     24808236       284224      7622356    0    0     0     0    0    0   0   0 100   0   0
 0  0            0     24808204       284224      7622372    0    0     0     0  304  388   0   0 100   0   0
 0  0            0     24808204       284224      7622372    0    0     0     0  681  900   0   0 100   0   0
 0  0            0     24808204       284224      7622372    0    0     0     0  333  428   0   0 100   0   0
 0  0            0     24808204       284224      7622372    0    0     0     0  241  326   0   0 100   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1168  969   2  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  577  329   3  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  547  359   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1087  820   3   9  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  590  389   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  563  309   3   9  88   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0 1015  766   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  889  646   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  577  385   2  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1054  869   3   9  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  566  393   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  654  421   2  11  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0 1057  916   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  591  345   2  11  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  560  376   2  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1050  873   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  628  336   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  615  344   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0 1009  841   3   9  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  555  354   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  551  351   2  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1209  971   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  841  678   3  10  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  637  348   3   9  87   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0 1161  978   3  10  87   0   0
procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 1  0            0     24807940       284224      7622376    0    0     0     0  672  366   2  10  88   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  671  402   2  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0 1101  923   3  10  87   0   0
 1  0            0     24808064       284224      7622376    0    0     0     0  565  364   2  10  88   0   0
 1  0            0     24807940       284224      7622376    0    0     0     0  604  334   3  10  87   0   0
 0  0            0     24807940       284224      7622376    0    0     0     0  987  889   2   9  89   0   0
 0  0            0     24807956       284224      7622376    0    0     0     0  239  312   0   0 100   0   0
 0  0            0     24807956       284224      7622376    0    0     0     0  237  364   0   0 100   0   0
 0  0            0     24807956       284224      7622376    0    0     0     0  695  988   0   0 100   0   0
 0  0            0     24807956       284224      7622376    0    0     0     0  193  314   0   0 100   0   0
 0  0            0     24807956       284224      7622376    0    0     0     0  332  421   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  614  860   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  401  587   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  188  320   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  585  840   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  192  298   0   0 100   0   0
 0  0            0     24807584       284224      7622384    0    0     0     0  438  548   0   0 100   0   0
 0  0            0     24807616       284224      7622384    0    0     0     0  565  723   0   0 100   0   0
^C
root@backend-shared-stag-0:~#
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_vmstat.png)

- 能够观察 in 和 cs 的变化（有点价值）
- 只能看到 CPU 整体情况（不足）
- 能够看到 r 和 b 的数值
- 无法直观确认哪个进程占用的 CPU（不足）


#### sar


```
root@backend-shared-stag-0:~# sar -u ALL -P ALL -I SUM -q -w 1
Linux 4.4.0-72-generic (backend-shared-stag-0) 	05/11/2018 	_x86_64_	(8 CPU)

02:57:57 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:57:58 PM     all      0.12      0.00      0.12      0.00      0.00      0.00      0.00      0.00      0.00     99.75
02:57:58 PM       0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       4      0.99      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00     99.01
02:57:58 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:58 PM       7      0.00      0.00      0.99      0.00      0.00      0.00      0.00      0.00      0.00     99.01

02:57:57 PM    proc/s   cswch/s
02:57:58 PM      1.00    162.00

02:57:57 PM      INTR    intr/s
02:57:58 PM       sum    165.00

02:57:57 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:57:58 PM         0       268      0.02      0.09      0.04         0

02:57:58 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:57:59 PM     all      0.00      0.00      0.12      0.00      0.12      0.00      0.00      0.00      0.00     99.75
02:57:59 PM       0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:57:59 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:57:58 PM    proc/s   cswch/s
02:57:59 PM      1.00    709.00

02:57:58 PM      INTR    intr/s
02:57:59 PM       sum    556.00

02:57:58 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:57:59 PM         0       269      0.02      0.09      0.04         0
...
02:58:05 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:58:06 PM     all      2.88      0.00      9.62      0.00      0.00      0.00      0.00      0.00      0.00     87.50
02:58:06 PM       0     23.76      0.00     76.24      0.00      0.00      0.00      0.00      0.00      0.00      0.00
02:58:06 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:06 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:58:05 PM    proc/s   cswch/s
02:58:06 PM      0.00    321.00

02:58:05 PM      INTR    intr/s
02:58:06 PM       sum    575.00

02:58:05 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:58:06 PM         1       271      0.02      0.09      0.04         0

02:58:06 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:58:07 PM     all      3.38      0.00      9.14      0.00      0.00      0.00      0.00      0.00      0.00     87.48
02:58:07 PM       0     27.27      0.00     72.73      0.00      0.00      0.00      0.00      0.00      0.00      0.00
02:58:07 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:07 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:58:06 PM    proc/s   cswch/s
02:58:07 PM      0.00    343.00

02:58:06 PM      INTR    intr/s
02:58:07 PM       sum    584.00

02:58:06 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:58:07 PM         1       271      0.10      0.10      0.05         0

02:58:07 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:58:08 PM     all      2.75      0.00      9.89      0.00      0.00      0.00      0.00      0.00      0.00     87.36
02:58:08 PM       0     20.79      0.00     79.21      0.00      0.00      0.00      0.00      0.00      0.00      0.00
02:58:08 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:08 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:58:07 PM    proc/s   cswch/s
02:58:08 PM      0.00    836.00

02:58:07 PM      INTR    intr/s
02:58:08 PM       sum   1092.00

02:58:07 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:58:08 PM         1       271      0.10      0.10      0.05         0

02:58:08 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:58:09 PM     all      2.88      0.00      9.62      0.00      0.00      0.00      0.00      0.00      0.00     87.50
02:58:09 PM       0     24.00      0.00     76.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
02:58:09 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:09 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:58:08 PM    proc/s   cswch/s
02:58:09 PM      0.00    401.00

02:58:08 PM      INTR    intr/s
02:58:09 PM       sum    608.00

02:58:08 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:58:09 PM         1       271      0.10      0.10      0.05         0
...
02:58:35 PM     CPU      %usr     %nice      %sys   %iowait    %steal      %irq     %soft    %guest    %gnice     %idle
02:58:36 PM     all      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       3      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       4      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       5      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       6      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00
02:58:36 PM       7      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00    100.00

02:58:35 PM    proc/s   cswch/s
02:58:36 PM      0.00    326.00

02:58:35 PM      INTR    intr/s
02:58:36 PM       sum    249.00

02:58:35 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:58:36 PM         0       270      0.41      0.18      0.08         0
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_sar.png)

- 能够针对 per-CPU 展示使用情况
- 能够基于 cswch/s 和 intr/s 看出变化情况，但由于输出信息的形式，不方便进行变化比较；
- 无法直观确认哪个进程占用的 CPU（不足）


#### top

```
root@backend-shared-stag-0:~# top -b -n 100
top - 14:57:58 up 123 days,  1:37,  5 users,  load average: 0.02, 0.09, 0.04
Tasks: 199 total,   1 running, 198 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu6  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu7  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32946256 total, 24808500 free,   231180 used,  7906576 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 32135792 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0  185764   6432   4024 S   0.0  0.0   1:00.38 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.20 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:00.37 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      20   0       0      0      0 S   0.0  0.0   3:18.29 rcu_sched
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
    9 root      rt   0       0      0      0 S   0.0  0.0   0:00.85 migration/0
   10 root      rt   0       0      0      0 S   0.0  0.0   0:43.38 watchdog/0
   11 root      rt   0       0      0      0 S   0.0  0.0   0:39.86 watchdog/1
   12 root      rt   0       0      0      0 S   0.0  0.0   0:00.76 migration/1
   13 root      20   0       0      0      0 S   0.0  0.0   0:00.28 ksoftirqd/1
   15 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/1:0H
   16 root      rt   0       0      0      0 S   0.0  0.0   0:37.83 watchdog/2
   17 root      rt   0       0      0      0 S   0.0  0.0   0:00.40 migration/2
   18 root      20   0       0      0      0 S   0.0  0.0   0:00.24 ksoftirqd/2
   20 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/2:0H
...

top - 14:58:07 up 123 days,  1:37,  5 users,  load average: 0.10, 0.10, 0.05
Tasks: 202 total,   2 running, 200 sleeping,   0 stopped,   0 zombie
%Cpu0  : 22.9 us, 77.1 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu6  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu7  :  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32946256 total, 24807940 free,   231716 used,  7906600 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 32135232 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 7509 root      20   0    6048    848    780 R 100.0  0.0   0:03.26 dd
 1072 root      20   0  606792  45608  26528 S   0.3  0.1 217:31.06 dockerd
 1248 root      20   0   19608   2084   1740 S   0.3  0.0   7:22.11 irqbalance
 1275 root      20   0  670016  13848   6684 S   0.3  0.0 121:51.88 docker-containe
 7025 root      20   0       0      0      0 S   0.3  0.0   0:00.09 kworker/u30:0
 7505 root      20   0   11416   1968   1752 S   0.3  0.0   0:00.01 sadc
    1 root      20   0  185764   6432   4024 S   0.0  0.0   1:00.38 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.20 kthreadd
...
top - 14:58:16 up 123 days,  1:37,  5 users,  load average: 0.17, 0.12, 0.05
Tasks: 202 total,   2 running, 200 sleeping,   0 stopped,   0 zombie
%Cpu0  : 19.3 us, 80.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu6  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu7  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32946256 total, 24808064 free,   231592 used,  7906600 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 32135356 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 7509 root      20   0    6048    848    780 R 100.0  0.0   0:12.28 dd
 1072 root      20   0  606792  45608  26528 S   0.3  0.1 217:31.07 dockerd
 6766 deployer  20   0   92800   3316   2392 S   0.3  0.0   0:00.02 sshd
 7506 root      20   0   40520   3780   3180 R   0.3  0.0   0:00.03 top
    1 root      20   0  185764   6432   4024 S   0.0  0.0   1:00.38 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.20 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:00.37 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      20   0       0      0      0 S   0.0  0.0   3:18.29 rcu_sched
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
...

```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_top.png)

- 输出的信息有点多，略微对问题有点干扰（小不足）
- 能够展示 per-CPU 使用情况，但需要事先定制展示模式（小不足）
- 能够直接确认占用 CPU 多的对象


#### mpstat

```
root@backend-shared-stag-0:~# mpstat -u -I SUM -P ALL 1
Linux 4.4.0-72-generic (backend-shared-stag-0) 	05/11/2018 	_x86_64_	(8 CPU)

02:57:58 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:57:59 PM  all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:57:59 PM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

02:57:58 PM  CPU    intr/s
02:57:59 PM  all    196.00
02:57:59 PM    0     41.00
02:57:59 PM    1     40.00
02:57:59 PM    2     32.00
02:57:59 PM    3     36.00
02:57:59 PM    4     56.00
02:57:59 PM    5     70.00
02:57:59 PM    6     41.00
02:57:59 PM    7     29.00
...

02:58:04 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:58:05 PM  all    2.88    0.00    9.62    0.00    0.00    0.00    0.00    0.00    0.00   87.50
02:58:05 PM    0   23.00    0.00   77.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
02:58:05 PM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:05 PM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

02:58:04 PM  CPU    intr/s
02:58:05 PM  all    604.00
02:58:05 PM    0    695.00
02:58:05 PM    1     59.00
02:58:05 PM    2     53.00
02:58:05 PM    3     36.00
02:58:05 PM    4     91.00
02:58:05 PM    5     86.00
02:58:05 PM    6     21.00
02:58:05 PM    7     45.00

02:58:05 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:58:06 PM  all    2.88    0.00    9.64    0.00    0.00    0.00    0.00    0.00    0.00   87.48
02:58:06 PM    0   22.22    0.00   77.78    0.00    0.00    0.00    0.00    0.00    0.00    0.00
02:58:06 PM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    5    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    6    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
02:58:06 PM    7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

02:58:05 PM  CPU    intr/s
02:58:06 PM  all    575.00
02:58:06 PM    0    679.00
02:58:06 PM    1     44.00
02:58:06 PM    2     45.00
02:58:06 PM    3     46.00
02:58:06 PM    4     96.00
02:58:06 PM    5     70.00
02:58:06 PM    6     19.00
02:58:06 PM    7     28.00
...
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_mpstat.png)


- 和前面的大同小异，但 intr/s 的数据输出形式有有点费解；


#### pidstat

```
root@backend-shared-stag-0:~# pidstat -u -l 1
Linux 4.4.0-72-generic (backend-shared-stag-0) 	05/11/2018 	_x86_64_	(8 CPU)

03:38:19 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command

03:38:20 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command

03:38:21 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:22 PM     0      7664    0.00    1.00    0.00    1.00     6  pidstat -u -l 1

03:38:22 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command

03:38:23 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:24 PM     0      7664    1.00    0.00    0.00    1.00     6  pidstat -u -l 1

03:38:24 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:25 PM  1001      5566    1.00    0.00    0.00    1.00     5  sshd: deployer@pts/3
03:38:25 PM     0      7665    0.00    2.00    0.00    2.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:25 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:26 PM     0      7664    0.00    1.00    0.00    1.00     6  pidstat -u -l 1
03:38:26 PM     0      7665   22.00   77.00    0.00   99.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:26 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:27 PM     0      1072    1.00    0.00    0.00    1.00     1  /usr/bin/dockerd -H fd:// --registry-mirror=http://a9d2d4bd.m.daocloud.io
03:38:27 PM     0      1109    2.00    0.00    0.00    2.00     5  /opt/monitor/node-exporter/node_exporter
03:38:27 PM     0      7665   18.00   83.00    0.00  101.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:27 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:28 PM     0      7665   18.00   82.00    0.00  100.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:28 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:29 PM     0      7664    0.00    1.00    0.00    1.00     6  pidstat -u -l 1
03:38:29 PM     0      7665   22.00   78.00    0.00  100.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:29 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:30 PM     0      7665   22.00   77.00    0.00   99.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:30 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:31 PM     0      7665   21.00   79.00    0.00  100.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:31 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:32 PM     0      7665   19.00   81.00    0.00  100.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M

03:38:32 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:38:33 PM     0      7664    0.00    1.00    0.00    1.00     6  pidstat -u -l 1
03:38:33 PM     0      7665   16.00   85.00    0.00  101.00     7  dd if=/dev/zero of=/dev/null bs=1024 count=100M
...
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_pidstat.png)

- CPU 分类没有那么详细（不足）
- 没有 in 数据输出（不足）
- 非常适合迅速、准确的定位导致 CPU 占用的进程或线程（亮点）
- 能够按 context switch 类型进行细分统计


#### /usr/bin/time


```
root@backend-shared-stag-0:~/workspace# /usr/bin/time -v dd if=/dev/zero of=/dev/null bs=1024 count=100M
104857600+0 records in
104857600+0 records out
107374182400 bytes (107 GB, 100 GiB) copied, 30.7698 s, 3.5 GB/s
	Command being timed: "dd if=/dev/zero of=/dev/null bs=1024 count=100M"
	User time (seconds): 6.18
	System time (seconds): 24.58
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:30.77
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 2044
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 79
	Voluntary context switches: 1
	Involuntary context switches: 41
	Swaps: 0
	File system inputs: 0
	File system outputs: 0
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
root@backend-shared-stag-0:~/workspace#
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/dd_time.png)

- 只能在运行结束后看到概况信息，无法实时查看
- 用处不大


#### perf

```
########################
###   Counting Events
########################

# CPU counter statistics for the specified command:
perf stat command

# Detailed CPU counter statistics (includes extras) for the specified command:
perf stat -d command

# CPU counter statistics for the specified PID, until Ctrl-C:
perf stat -p PID

# CPU counter statistics for the entire system, for 5 seconds:
perf stat -a sleep 5

# Various basic CPU statistics, system wide, for 10 seconds:
perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 10

# Various CPU level 1 data cache statistics for the specified command:
perf stat -e L1-dcache-loads,L1-dcache-load-misses,L1-dcache-stores command

# Various CPU data TLB statistics for the specified command:
perf stat -e dTLB-loads,dTLB-load-misses,dTLB-prefetch-misses command

# Various CPU last level cache statistics for the specified command:
perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches command

# PMCs: cycles and frontend stalls via raw specification:
perf stat -e cycles -e cpu/event=0x0e,umask=0x01,inv,cmask=0x01/ -a sleep 5

# Show sent network packets by on-CPU process, rolling output (no clear):
stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings


########################
###   Profiling
########################

# Sample on-CPU functions for the specified command, at 99 Hertz:
perf record -F 99 command

# Sample on-CPU functions for the specified PID, at 99 Hertz, until Ctrl-C:
perf record -F 99 -p PID

# Sample on-CPU functions for the specified PID, at 99 Hertz, for 10 seconds:
perf record -F 99 -p PID sleep 10

# Sample CPU stack traces (via frame pointers) for the specified PID, at 99 Hertz, for 10 seconds:
perf record -F 99 -p PID -g -- sleep 10

# Sample CPU stack traces for the PID, using dwarf (dbg info) to unwind stacks, at 99 Hertz, for 10 seconds:
perf record -F 99 -p PID --call-graph dwarf sleep 10

# Sample CPU stack traces for the entire system, at 99 Hertz, for 10 seconds (< Linux 4.11):
perf record -F 99 -ag -- sleep 10

# Sample CPU stack traces for the entire system, at 99 Hertz, for 10 seconds (>= Linux 4.11):
perf record -F 99 -g -- sleep 10

# If the previous command didn't work, try forcing perf to use the cpu-clock event:
perf record -F 99 -e cpu-clock -ag -- sleep 10

# Sample CPU stack traces for a container identified by its /sys/fs/cgroup/perf_event cgroup:
perf record -F 99 -e cpu-clock --cgroup=docker/1d567f4393190204...etc... -a -- sleep 10

# Sample CPU stack traces for the entire system, with dwarf stacks, at 99 Hertz, for 10 seconds:
perf record -F 99 -a --call-graph dwarf sleep 10

# Sample CPU stack traces for the entire system, using last branch record for stacks, ... (>= Linux 4.?):
perf record -F 99 -a --call-graph lbr sleep 10

# Sample CPU stack traces, once every 10,000 Level 1 data cache misses, for 5 seconds:
perf record -e L1-dcache-load-misses -c 10000 -ag -- sleep 5

# Sample CPU stack traces, once every 100 last level cache misses, for 5 seconds:
perf record -e LLC-load-misses -c 100 -ag -- sleep 5

# Sample on-CPU kernel instructions, for 5 seconds:
perf record -e cycles:k -a -- sleep 5

# Sample on-CPU user instructions, for 5 seconds:
perf record -e cycles:u -a -- sleep 5

# Sample on-CPU user instructions precisely (using PEBS), for 5 seconds:
perf record -e cycles:up -a -- sleep 5

# Sample CPUs at 49 Hertz, and show top addresses and symbols, live (no perf.data file):
perf top -F 49

# Sample CPUs at 49 Hertz, and show top process names and segments, live:
perf top -F 49 -ns comm,dso


########################
###   Static Tracing
########################

# Trace CPU migrations, for 10 seconds:
perf record -e migrations -a -- sleep 10


########################
###   Mixed
########################

# Sample stacks at 99 Hertz, and, context switches:
perf record -F99 -e cpu-clock -e cs -a -g

# Sample stacks to 2 levels deep, and, context switch stacks to 5 levels (needs 4.8):
perf record -F99 -e cpu-clock/max-stack=2/ -e cs/max-stack=5/ -a -g


########################
###   Reporting
########################

# Show perf.data in an ncurses browser (TUI) if possible:
perf report

# Show perf.data with a column for sample count:
perf report -n

# List all perf.data events, with my recommended fields (needs record -a; newer kernels):
perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso

# List all perf.data events, with my recommended fields (needs record -a; older kernels):
perf script -f comm,pid,tid,cpu,time,event,ip,sym,dso
```


#### systemtap

TODO


## 案例

参考《[基于 vmstat 进行系统分析](https://github.com/moooofly/MarkSomethingDown/blob/8bfeabfe3e280341456185a620a0adc5943a0f80/Linux/%E5%9F%BA%E4%BA%8E%20vmstat%20%E8%BF%9B%E8%A1%8C%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90.md)》


----------


### docker 环境下的 CPU 分析

> TODO


### TODO

- [CPU Utilization is Wrong](http://www.brendangregg.com/blog/2017-05-09/cpu-utilization-is-wrong.html)
- [perf sched for Linux CPU scheduler analysis](http://www.brendangregg.com/blog/2017-03-16/perf-sched.html)
- [Linux eBPF Off-CPU Flame Graph](http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html)
- [Linux perf_events Off-CPU Time Flame Graph](http://www.brendangregg.com/blog/2015-02-26/linux-perf-off-cpu-flame-graph.html)
- [CPI Flame Graphs: Catching Your CPUs Napping](http://www.brendangregg.com/blog/2014-10-31/cpi-flame-graphs.html)
- [perf CPU Sampling](http://www.brendangregg.com/blog/2014-06-22/perf-cpu-sample.html)
- [The noploop CPU Benchmark](http://www.brendangregg.com/blog/2014-04-26/the-noploop-cpu-benchmark.html)
- [How much CPU does a down interface chew?](http://www.brendangregg.com/blog/2006-08-11/how-much-cpu-does-a-down-interface-chew.html)



