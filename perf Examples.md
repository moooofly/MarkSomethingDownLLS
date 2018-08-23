# perf Examples

> 原文地址：[这里](http://www.brendangregg.com/perf.html)

perf 别名：

- perf
- perf Linux profiler
- Performance Counters for Linux (PCL)
- Linux perf events (LPE)
- perf_events (*)


**perf 常见使用场景**：

- Why is the kernel on-CPU so much? What code-paths?
- Which code-paths are causing CPU level 2 cache misses?
- Are the CPUs stalled on memory I/O?
- Which code-paths are allocating memory, and how much?
- What is triggering TCP retransmits?
- Is a certain kernel function being called, and how often?
- What reasons are threads leaving the CPU?

知识点：

- `perf_events` is an **event-oriented** observability tool, which can help you **solve advanced performance and troubleshooting functions**. 
- `perf_events` is part of the Linux kernel, under `tools/perf`. While it uses many Linux tracing features, some are not yet exposed via the `perf` command, and need to be used via the `ftrace` interface instead.
- My [perf-tools](https://github.com/brendangregg/perf-tools) collection (github) uses both `perf_events` and `ftrace` as needed.

## Background

- The `perf` tool is in the `linux-tools-common` package. Start by adding that, then running "perf" to see if you get the USAGE message. It may tell you to install another related package (`linux-tools-kernelversion`).
- To get the most out `perf`, you'll want **symbols** and **stack traces**. These may work by default in your Linux distribution, or they may require the addition of packages, or recompilation of the kernel with additional config options.
- `perf_events`, like other debug tools, needs symbol information (symbols). These are used to translate memory addresses into function and variable names, so that they can be read by us humans.
- If the software was added by packages, you may find debug packages (often "`-dbgsym`") which provide the symbols.
- I find it useful to add both `libc6-dbgsym` and `coreutils-dbgsym`, to provide some symbol coverage of **user-level OS codepaths**.
- Another way to get symbols is to compile the software yourself. `file xxx` will get "... not stripped".
- **Kernel-level symbols are in the kernel debuginfo package**, or when the kernel is compiled with `CONFIG_KALLSYMS`.
- **Always compile with `frame pointers`**. Omitting frame pointers is an evil compiler optimization that breaks debuggers, and sadly, is often the default. Without them, you may see incomplete stacks from `perf_events`, like seen in the earlier sshd symbols example. There are three ways to fix this: either using `dwarf` data to unwind the stack, using `last branch record (LBR)` if available (a processor feature), or returning the frame pointers.
- There are other stack walking techniques, like B`TS (Branch Trace Store)`, and the new `ORC` unwinder.
- with `CONFIG_FRAME_POINTER=y`, the kernel stack traces are complete.
- Since about the **3.9 kernel**, `perf_events` has supported a workaround for missing frame pointers in user-level stacks: `libunwind`, which uses `dwarf`. This can be enabled using "`--call-graph dwarf`" (or "`-g dwarf`").
- Apart from separate help for each subcommand, there is also documentation in the kernel source under `tools/perf/Documentation`.
- `perf_events` can instrument in three ways:
    - **counting events in-kernel context (by `perf stat`)**, where a summary of counts is printed by `perf`. This mode does not generate a `perf.data` file. This subcommand costs the least overhead.
    - **sampling events (by `perf record`)**, which writes event data to a kernel buffer, which is read at a gentle asynchronous rate by the `perf` command to write to the `perf.data` file. This file is then read by the `perf report` or `perf script` commands. You'll need to be a little careful about the overheads, as the capture files can quickly become hundreds of Mbytes. It depends on the rate of the event you are tracing: the more frequent, the higher the overhead and larger the `perf.data` size.
    - **bpf programs on events**, a new feature in **Linux 4.4+ kernels** that can **execute custom user-defined programs in kernel space**, which can perform efficient filters and summaries of the data. Eg, **efficiently-measured latency histograms**. To really cut down overhead and generate more advanced summaries, write BPF programs executed by `perf`.
- **The use of `-p PID` as a filter doesn't work properly on some older kernel versions (Linux 3.x)**: `perf` hits 100% CPU and needs to be killed. It's annoying. The workaround is to profile all CPUs (`-a`), and filter PIDs later.
- Special Usage (These make use of perf's existing instrumentation capabilities, recording selected events and reporting them in custom ways)
    - `perf c2c` (Linux 4.10+): cache-2-cache and cacheline false sharing analysis.
    - `perf kmem`: kernel memory allocation analysis.
    - `perf kvm`: KVM virtual guest analysis.
    - `perf lock`: lock analysis.
    - `perf mem`: memory access analysis.
    - `perf sched`: kernel scheduler statistics.


## Events

![](http://www.brendangregg.com/perf_events/perf_events_map.png)

The types of events are:

- **Hardware Events**: CPU performance monitoring counters (PMCs).
- **Software Events**: These are low level events based on kernel counters. For example, CPU migrations, minor faults, major faults, etc.
- **Kernel Tracepoint Events**: This are static kernel-level instrumentation points that are hardcoded in interesting and logical places in the kernel.
- **User Statically-Defined Tracing (USDT)**: These are static tracepoints for user-level programs and applications.
- **Dynamic Tracing**: Software can be dynamically instrumented, creating events in any location. For kernel software, this uses the `kprobes` framework. For user-level software, `uprobes`.
- **Timed Profiling**: Snapshots can be collected at an arbitrary frequency, using `perf record -FHz`. This is commonly used for CPU usage profiling, and works by creating custom timed interrupt events.

### Hardware Events (PMCs)

`perf_events` began life as a tool for instrumenting the processor's **performance monitoring unit** (`PMU`) hardware counters, also called **performance monitoring counters** (`PMCs`), or **performance instrumentation counters** (`PICs`). These instrument low-level processor activity, for example, CPU cycles, instructions retired, memory stall cycles, level 2 cache misses, etc. Some will be listed as Hardware Cache Events.

### Kernel Tracepoints

These tracepoints are hard coded in interesting and logical locations of the kernel, so that higher-level behavior can be easily traced. For example, **system calls**, **TCP events**, **file system I/O**, **disk I/O**, etc. These are grouped into libraries of tracepoints; eg, "`sock:`" for socket events, "`sched:`" for CPU scheduler events. A key value of tracepoints is that they should have a stable API, so if you write tools that use them on one kernel version, they should work on later versions as well.

Summarizing the tracepoint library names and numbers of tracepoints:

```
root@backend-shared-stag-0:~/workspace# perf list | awk -F: '/Tracepoint event/ { lib[$1]++ } END {
>     for (l in lib) { printf "  %-16.16s %d\n", l, lib[l] } }' | sort | column
    block          19	    ftrace         1	    napi           1	    regmap         15	    thermal        7
    btrfs          44	    gpio           2	    net            10	    regulator      7	    thermal_power_ 2
    clk            14	    i2c            8	    nfs            44	    rpm            4	    timer          13
    cma            2	    iommu          7	    nfs4           66	    sched          24	    tlb            1
    compaction     11	    irq            5	    nmi            1	    scsi           5	    udp            1
    drm            3	    irq_vectors    22	    oom            1	    signal         2	    vmscan         15
    exceptions     2	    jbd2           16	    pagemap        2	    skb            3	    vsyscall       1
    ext4           95	    kmem           12	    power          22	    sock           2	    workqueue      4
    fence          8	    libata         6	    printk         1	    spi            7	    writeback      28
    fib            3	    mce            1	    random         15	    sunrpc         26	    xen            35
    filelock       6	    migrate        2	    ras            4	    swiotlb        1	    xhci-hcd       9
    filemap        2	    module         5	    raw_syscalls   2	    syscalls       604
    fs             2	    mpx            5	    rcu            1	    task           2
root@backend-shared-stag-0:~/workspace#
```

These include:

- **block**: block device I/O
- **ext4**: file system operations
- **kmem**: kernel memory allocation events
- **random**: kernel random number generator events
- **sched**: CPU scheduler events
- **syscalls**: system call enter and exits
- **task**: task events

### User-Level Statically Defined Tracing (USDT)

Similar to kernel tracepoints, these are hardcoded (usually by placing macros) in the application source at logical and interesting locations, and presented (event name and arguments) as a stable API. Many applications already include tracepoints, added to support `DTrace`. However, many of these applications do not compile them in by default on Linux. Often you need to compile the application yourself using a `--with-dtrace` flag.

### Dynamic Tracing

The difference between **tracepoints** and **dynamic tracing** is shown in the following figure, which illustrates the coverage of common tracepoint libraries:

![](http://www.brendangregg.com/perf_events/perf_tracepoints_1700.png)

**While dynamic tracing can see everything, it's also an unstable interface since it is instrumenting raw code**. That means that any dynamic tracing tools you develop may break after a kernel patch or update. **Try to use the static tracepoints first**, since their interface should be much more stable. They can also be easier to use and understand, since they have been designed with a tracing end-user in mind.

**One benefit of dynamic tracing is that it can be enabled on a live system without restarting anything**. 

The overhead while dynamic tracing is in use, and extra instructions are being executed, is relative to the frequency of instrumented events multiplied by the work done on each instrumentation.

## Examples

### Disk I/O Tracing

```
# perf record -e block:block_rq_issue -ag
^C
# ls -l perf.data
-rw------- 1 root root 3458162 Jan 26 03:03 perf.data
# perf report
[...]
# Samples: 2K of event 'block:block_rq_issue'
# Event count (approx.): 2216
#
# Overhead       Command      Shared Object                Symbol
# ........  ............  .................  ....................
#
    32.13%            dd  [kernel.kallsyms]  [k] blk_peek_request
                      |
                      --- blk_peek_request
                          virtblk_request
                          __blk_run_queue
                         |          
                         |--98.31%-- queue_unplugged
                         |          blk_flush_plug_list
                         |          |          
                         |          |--91.00%-- blk_queue_bio
                         |          |          generic_make_request
                         |          |          submit_bio
                         |          |          ext4_io_submit
                         |          |          |          
                         |          |          |--58.71%-- ext4_bio_write_page
                         |          |          |          mpage_da_submit_io
                         |          |          |          mpage_da_map_and_submit
                         |          |          |          write_cache_pages_da
                         |          |          |          ext4_da_writepages
                         |          |          |          do_writepages
                         |          |          |          __filemap_fdatawrite_range
                         |          |          |          filemap_flush
                         |          |          |          ext4_alloc_da_blocks
                         |          |          |          ext4_release_file
                         |          |          |          __fput
                         |          |          |          ____fput
                         |          |          |          task_work_run
                         |          |          |          do_notify_resume
                         |          |          |          int_signal
                         |          |          |          close
                         |          |          |          0x0
                         |          |          |          
                         |          |           --41.29%-- mpage_da_submit_io
[...]
```

说明：

- trace the `block:block_rq_issue` probe, which fires when a block device I/O request is issued (disk I/O). 
- `-a` to trace all CPUs.
- `-g` to capture call graphs (stack traces).
- Trace data is written to a `perf.data` file, and tracing ended when `Ctrl-C` was hit.
- A summary of the `perf.data` file was printed using `perf report`, which builds a tree from the stack traces, coalescing common paths, and showing percentages for each path.

报告解读：

- The `perf report` output shows that 2,216 events were traced (disk I/O).
- 32% of which from a `dd` command. 
- These were issued by the kernel function `blk_peek_request()`, and walking down the stacks, about half of these 32% were from the `close()` system call.


### CPU Statistics


```
# perf stat gzip file1

 Performance counter stats for 'gzip file1':

       1920.159821 task-clock                #    0.991 CPUs utilized          
                13 context-switches          #    0.007 K/sec                  
                 0 CPU-migrations            #    0.000 K/sec                  
               258 page-faults               #    0.134 K/sec                  
     5,649,595,479 cycles                    #    2.942 GHz                     [83.43%]
     1,808,339,931 stalled-cycles-frontend   #   32.01% frontend cycles idle    [83.54%]
     1,171,884,577 stalled-cycles-backend    #   20.74% backend  cycles idle    [66.77%]
     8,625,207,199 instructions              #    1.53  insns per cycle        
                                             #    0.21  stalled cycles per insn [83.51%]
     1,488,797,176 branches                  #  775.351 M/sec                   [82.58%]
        53,395,139 branch-misses             #    3.59% of all branches         [83.78%]

       1.936842598 seconds time elapsed
```

This includes **instructions per cycle** (`IPC`), labled "**insns per cycle**", or in earlier versions, "IPC". This is a commonly examined metric, either IPC or its invert, `CPI`. **Higher IPC values mean higher instruction throughput, and lower values indicate more stall cycles**. I'd generally interpret high IPC values (eg, over 1.0) as good, indicating optimal processing of work. However, I'd want to **double check** what the instructions are, in case this is due to a spin loop: **a high rate of instructions, but a low rate of actual work completed**.

**Stalled cycles per instruction** is similar to IPC (inverted), however, only counting stalled cycles, which will be for memory or resource bus access. This makes it easy to interpret: stalls are latency, reduce stalls. I really like it as a metric, and hope it becomes as commonplace as IPC/CPI. Lets call it `SCPI`.


### Timed Profiling

`perf_events` can profile CPU usage based on sampling the **instruction pointer** or **stack trace** at a fixed interval (timed profiling).

Sampling CPU stacks at 99 Hertz (`-F 99`), for the entire system (`-a`, for all CPUs), with stack traces (`-g`, for call graphs), for 10 seconds:

```
root@backend-shared-stag-0:~/workspace# time perf record -F 99 -a -g -- sleep 30
[ perf record: Woken up 4 times to write data ]
[ perf record: Captured and wrote 2.730 MB perf.data (23760 samples) ]

real	0m30.131s
user	0m0.060s
sys	0m0.068s
root@backend-shared-stag-0:~/workspace# ll
total 2808
drwxr-xr-x 2 root root    4096 Apr 24 09:08 ./
drwx------ 5 root root    4096 Apr 24 03:42 ../
-rw------- 1 root root 2865388 Apr 24 09:08 perf.data
root@backend-shared-stag-0:~/workspace#
```

**The choice of `99` Hertz**, instead of 100 Hertz, is to avoid accidentally sampling in lockstep with some periodic activity, which would produce skewed results. This is also coarse: **you may want to increase that to higher rates (eg, up to `997` Hertz) for finer resolution**, especially if you are sampling short bursts of activity and you'd still like enough resolution to be useful. **Bear in mind that higher frequencies means higher overhead**.

The `perf.data` file can be processed in a variety of ways. On recent versions, the `perf report` command launches an `ncurses` navigator for call graph inspection. Older versions of perf (or if you use `--stdio` in the new version) print the call graph as a tree, annotated with percentages:

```
# perf report --stdio
# ========
# captured on: Mon Jan 26 07:26:40 2014
# hostname : dev2
# os release : 3.8.6-ubuntu-12-opt
# perf version : 3.8.6
# arch : x86_64
# nrcpus online : 8
# nrcpus avail : 8
# cpudesc : Intel(R) Xeon(R) CPU X5675 @ 3.07GHz
# cpuid : GenuineIntel,6,44,2
# total memory : 8182008 kB
# cmdline : /usr/bin/perf record -F 99 -a -g -- sleep 30 
# event : name = cpu-clock, type = 1, config = 0x0, config1 = 0x0, config2 = ...
# HEADER_CPU_TOPOLOGY info available, use -I to display
# HEADER_NUMA_TOPOLOGY info available, use -I to display
# pmu mappings: software = 1, breakpoint = 5
# ========
#
# Samples: 22K of event 'cpu-clock'
# Event count (approx.): 22751
#
# Overhead  Command      Shared Object                           Symbol
# ........  .......  .................  ...............................
#
    94.12%       dd  [kernel.kallsyms]  [k] _raw_spin_unlock_irqrestore
                 |
                 --- _raw_spin_unlock_irqrestore
                    |          
                    |--96.67%-- extract_buf
                    |          extract_entropy_user
                    |          urandom_read
                    |          vfs_read
                    |          sys_read
                    |          system_call_fastpath
                    |          read
                    |          
                    |--1.69%-- account
                    |          |          
                    |          |--99.72%-- extract_entropy_user
                    |          |          urandom_read
                    |          |          vfs_read
                    |          |          sys_read
                    |          |          system_call_fastpath
                    |          |          read
                    |           --0.28%-- [...]
                    |          
                    |--1.60%-- mix_pool_bytes.constprop.17
[...]
```

**This tree starts with the on-CPU functions and works back through the ancestry**. This approach is called a "`callee based call graph`". This can be flipped by using `-G` for an "`inverted call graph`", or by using the "caller" option to `-g/--call-graph`, instead of the "callee" default.

**The hottest (most frequent) stack trace in this perf call graph occurred in 90.99% of samples, which is the product of the overhead percentage and top stack leaf (94.12% x 96.67%, which are relative rates)**. `perf report` can also be run with "`-g graph`" to show absolute overhead rates, in which case "90.99%" is directly displayed on the stack leaf:

```
    94.12%       dd  [kernel.kallsyms]  [k] _raw_spin_unlock_irqrestore
                 |
                 --- _raw_spin_unlock_irqrestore
                    |          
                    |--90.99%-- extract_buf
[...]
```

If user-level stacks look incomplete, you can try `perf record` with "`--call-graph dwarf`" as a different technique to unwind them.

The output from `perf report` can be many pages long, which can become cumbersome to read. **Try generating [Flame Graphs](http://www.brendangregg.com/perf.html#FlameGraphs) from the same data**.

> 这里表明了**为何需要火焰图，以及火焰图和 perf 的关系**；

### Event Profiling

Apart from sampling at a timed interval, taking samples triggered by CPU hardware counters is another form of CPU profiling, which can be **used to shed more light on cache misses, memory stall cycles, and other low-level processor events**. The available events can be found using `perf list`:

```
# perf list | grep Hardware
```

For many of these, **gathering a stack on every occurrence would induce far too much overhead, and would slow down the system and change the performance characteristics of the target**. It's usually sufficient to only instrument a small fraction of their occurrences, rather than all of them. This can be done by specifying a threshold for triggering event collection, using "`-c`" and a count.

For example, the following one-liner **instruments Level 1 data cache load misses**, collecting a stack trace for one in every 10,000 occurrences:

```
# perf record -e L1-dcache-load-misses -c 10000 -ag -- sleep 5
```

The mechanics of "`-c count`" are implemented by the processor, which only interrupts the kernel when the threshold has been reached.


### Static Kernel Tracing

static tracing: the instrumentation of tracepoints and other static events.

#### Counting Syscalls

> 统计系统调用分布

counts system calls for the executed command, and prints a summary (of non-zero counts):

> 将 file1 替换为真实文件名

```
# perf stat -e 'syscalls:sys_enter_*' gzip file1 2>&1 | awk '$1 != 0'

 Performance counter stats for 'gzip file1':

                 1 syscalls:sys_enter_utimensat               
                 1 syscalls:sys_enter_unlink                  
                 5 syscalls:sys_enter_newfstat                
             1,603 syscalls:sys_enter_read                    
             3,201 syscalls:sys_enter_write                   
                 5 syscalls:sys_enter_access                  
                 1 syscalls:sys_enter_fchmod                  
                 1 syscalls:sys_enter_fchown                  
                 6 syscalls:sys_enter_open                    
                 9 syscalls:sys_enter_close                   
                 8 syscalls:sys_enter_mprotect                
                 1 syscalls:sys_enter_brk                     
                 1 syscalls:sys_enter_munmap                  
                 1 syscalls:sys_enter_set_robust_list         
                 1 syscalls:sys_enter_futex                   
                 1 syscalls:sys_enter_getrlimit               
                 5 syscalls:sys_enter_rt_sigprocmask          
                14 syscalls:sys_enter_rt_sigaction            
                 1 syscalls:sys_enter_exit_group              
                 1 syscalls:sys_enter_set_tid_address         
                14 syscalls:sys_enter_mmap                    

       1.543990940 seconds time elapsed
```

In this case, a `gzip` command was analyzed. The report shows that there were 3,201 `write()` syscalls, and half that number of `read()` syscalls. Many of the other syscalls will be due to process and library initialization.

A similar report can be seen using `strace -c`, the system call tracer, however **it may induce much higher overhead than `perf`, as `perf` buffers data in-kernel**.

#### perf vs strace

To explain the difference a little further: the current implementation of `strace` uses `ptrace(2)` to attach to the target process and stop it during system calls, like a debugger. This is violent, and can cause serious overhead. To demonstrate this, the following syscall-heavy program was run by itself, with `perf`, and with `strace`. I've only included the line of output that shows its performance:

- 直接运行

```
root@backend-shared-stag-0:~/workspace# dd if=/dev/zero of=/dev/null bs=512 count=10000k
10240000+0 records in
10240000+0 records out
5242880000 bytes (5.2 GB, 4.9 GiB) copied, 2.6875 s, 2.0 GB/s
root@backend-shared-stag-0:~/workspace#
```

- 基于 perf 运行

```
root@backend-shared-stag-0:~/workspace# perf stat -e 'syscalls:sys_enter_*' dd if=/dev/zero of=/dev/null bs=512 count=10000k 2>&1 | awk '$1 != 0'
10240000+0 records in
10240000+0 records out
5242880000 bytes (5.2 GB, 4.9 GiB) copied, 6.29422 s, 833 MB/s

 Performance counter stats for 'dd if=/dev/zero of=/dev/null bs=512 count=10000k':

                 2      syscalls:sys_enter_dup2
                 4      syscalls:sys_enter_newfstat
                 1      syscalls:sys_enter_lseek
          10240003      syscalls:sys_enter_read
          10240003      syscalls:sys_enter_write
                 3      syscalls:sys_enter_access
                12      syscalls:sys_enter_open
                 9      syscalls:sys_enter_close
                 4      syscalls:sys_enter_mprotect
                 3      syscalls:sys_enter_brk
                 1      syscalls:sys_enter_munmap
                 2      syscalls:sys_enter_clock_gettime
                 3      syscalls:sys_enter_rt_sigaction
                 1      syscalls:sys_enter_exit_group
                 9      syscalls:sys_enter_mmap

       6.295085946 seconds time elapsed

root@backend-shared-stag-0:~/workspace#
```

- 基于 strace 运行

```
root@backend-shared-stag-0:~/workspace# strace -c dd if=/dev/zero of=/dev/null bs=512 count=10000k

10240000+0 records in
10240000+0 records out
5242880000 bytes (5.2 GB, 4.9 GiB) copied, 1065.82 s, 4.9 MB/s
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 50.47    0.982292           0  10240003           read
 49.53    0.963951           0  10240003           write
  0.00    0.000000           0        12         6 open
  0.00    0.000000           0         9           close
  0.00    0.000000           0         4           fstat
  0.00    0.000000           0         1           lseek
  0.00    0.000000           0         9           mmap
  0.00    0.000000           0         4           mprotect
  0.00    0.000000           0         1           munmap
  0.00    0.000000           0         3           brk
  0.00    0.000000           0         3           rt_sigaction
  0.00    0.000000           0         3         3 access
  0.00    0.000000           0         2           dup2
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         2           clock_gettime
------ ----------- ----------- --------- --------- ----------------
100.00    1.946243              20480061         9 total
root@backend-shared-stag-0:~/workspace#
```

With `perf`, the program ran 2.5x slower. But with `strace`, it ran 62x slower. That's likely to be a worst-case result: **if syscalls are not so frequent, the difference between the tools will not be as great**.

Recent version of `perf` have included a `trace` subcommand, to provide some similar functionality to `strace`, but with much lower overhead.

### New Processes

> 追踪新进程的创建

调用 `perf record`后，在另外一个 console 上执行 `man ls` ：

```
root@backend-shared-stag-0:~/workspace# perf record -e sched:sched_process_exec -a
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.150 MB perf.data (9 samples) ]

root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#



root@backend-shared-stag-0:~/workspace# perf report -n --sort comm --stdio
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 9  of event 'sched:sched_process_exec'
# Event count (approx.): 9
#
# Overhead       Samples  Command
# ........  ............  .......
#
    11.11%             1  groff
    11.11%             1  grotty
    11.11%             1  locale
    11.11%             1  man
    11.11%             1  nroff
    11.11%             1  pager
    11.11%             1  preconv
    11.11%             1  tbl
    11.11%             1  troff


root@backend-shared-stag-0:~/workspace#
```

Nine different commands were executed, each once. I used `-n` to print the "**Samples**" column, and "`--sort comm`" to customize the remaining columns.

This works by tracing `sched:sched_process_exec`, when a process runs `exec()` to execute a different binary. **This is often how new processes are created, but not always**. An application may `fork()` to create a pool of worker processes, but not `exec()` a different binary. An application may also **reexec**: call `exec()` again, on itself, usually to clean up its address space. In that case, it's will be seen by this exec tracepoint, but it's not a new process.

The `sched:sched_process_fork` tracepoint can be traced to only catch new processes, created via `fork()`. The downside is that the process identified is the parent, not the new target, as the new process has yet to `exec()` it's final program.


### Outbound Connections

> 主动对外发起的连接建立

There can be times when it's useful to **double check what network connections are initiated by a server, from which processes, and why**. You might be surprised. **These connections** can be important to understand, as they **can be a source of latency**.

For this example, I have a completely idle ubuntu server, and while tracing I'll login to it using `ssh`. I'm going to trace outbound connections via the `connect()` syscall. Given that I'm performing an inbound connection over SSH, will there be any outbound connections at all?

> 测试方法：执行 `perf record` 后，在另外一个窗口 ssh 到该机器；

```
root@backend-shared-stag-0:~/workspace# perf record -e syscalls:sys_enter_connect -a
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.176 MB perf.data (56 samples) ]

root@backend-shared-stag-0:~/workspace#


root@backend-shared-stag-0:~/workspace# perf report --stdio --header
# ========
# captured on: Tue Apr 24 11:16:33 2018
# hostname : backend-shared-stag-0
# os release : 4.4.0-72-generic
# perf version : 4.4.49
# arch : x86_64
# nrcpus online : 8
# nrcpus avail : 8
# cpudesc : Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
# cpuid : GenuineIntel,6,79,1
# total memory : 32946256 kB
# cmdline : /usr/lib/linux-tools-4.4.0-72/perf record -e syscalls:sys_enter_connect -a
# event : name = syscalls:sys_enter_connect, , type = 2, size = 112, config = 0x442, { sample_period, sample_freq } = 1, sample_type =
# HEADER_CPU_TOPOLOGY info available, use -I to display
# HEADER_NUMA_TOPOLOGY info available, use -I to display
# pmu mappings: msr = 6, software = 1, tracepoint = 2, breakpoint = 5
# ========
#
#
# Total Lost Samples: 0
#
# Samples: 56  of event 'syscalls:sys_enter_connect'
# Event count (approx.): 56
#
# Overhead  Command          Shared Object       Symbol
# ........  ...............  ..................  .......................
#
    42.86%  haproxy          libc-2.23.so        [.] connect
    21.43%  sshd             libc-2.23.so        [.] connect
    14.29%  sudo             libc-2.23.so        [.] connect
     7.14%  bash             libc-2.23.so        [.] connect
     7.14%  groups           libc-2.23.so        [.] connect
     3.57%  lsb_release      libc-2.23.so        [.] connect
     3.57%  systemd-cgroups  libpthread-2.23.so  [.] __GI___libc_connect


#
# (For a higher level overview, try: perf report --sort comm,dso)
#
root@backend-shared-stag-0:~/workspace#
```

> haproxy 的出现是因为该机器上跑了 haproxy 服务；

The report shows that sshd, groups, mesg, and bash are all performing `connect()` syscalls.


The **stack traces** that led to the `connect()` can explain why:

> 该输出为没有安装 libc6-dbgsym 之前

```
root@backend-shared-stag-0:~/workspace# perf record -e syscalls:sys_enter_connect -ag
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.178 MB perf.data (44 samples) ]

root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace# perf report --stdio --header
# ========
# captured on: Tue Apr 24 11:26:32 2018
# hostname : backend-shared-stag-0
# os release : 4.4.0-72-generic
# perf version : 4.4.49
# arch : x86_64
# nrcpus online : 8
# nrcpus avail : 8
# cpudesc : Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
# cpuid : GenuineIntel,6,79,1
# total memory : 32946256 kB
# cmdline : /usr/lib/linux-tools-4.4.0-72/perf record -e syscalls:sys_enter_connect -ag
# event : name = syscalls:sys_enter_connect, , type = 2, size = 112, config = 0x442, { sample_period, sample_freq } = 1, sample_type =
# HEADER_CPU_TOPOLOGY info available, use -I to display
# HEADER_NUMA_TOPOLOGY info available, use -I to display
# pmu mappings: msr = 6, software = 1, tracepoint = 2, breakpoint = 5
# ========
#
#
# Total Lost Samples: 0
#
# Samples: 44  of event 'syscalls:sys_enter_connect'
# Event count (approx.): 44
#
# Children      Self  Command          Shared Object       Symbol
# ........  ........  ...............  ..................  .......................
#
    27.27%    27.27%  haproxy          libc-2.23.so        [.] connect
                    |
                    |--18.18%-- connect
                    |
                     --9.09%-- 0x401
                               connect

    27.27%    27.27%  sshd             libc-2.23.so        [.] connect
                       |
                       |--9.09%-- connect
                       |
                       |--9.09%-- 0x11c25
                       |          |
                       |          |--6.82%-- connect
                       |          |
                       |           --2.27%-- getaddrinfo
                       |                     0x13d217
                       |                     0x13f15d
                       |                     connect
                       |
                       |--4.55%-- 0
                       |          0x13f6b7
                       |          0x13f15d
                       |          connect
                       |
                       |--2.27%-- 0x848b41b87fc58941
                       |          0x7b904
                       |          0x13f8f2
                       |          connect
                       |
                        --2.27%-- 0x13f8f2
                                  connect

    18.18%    18.18%  sudo             libc-2.23.so        [.] connect
                       |
                       |--9.09%-- 0
                       |          |
                       |          |--6.82%-- 0x13f6b7
                       |          |          0x13f15d
                       |          |          connect
root@backend-shared-stag-0:~/workspace#
```

> 该输出为没有安装 libc6-dbgsym 之后

```
root@backend-shared-stag-0:~/workspace# perf report --stdio --header
# ========
# captured on: Tue Apr 24 11:26:32 2018
# hostname : backend-shared-stag-0
# os release : 4.4.0-72-generic
# perf version : 4.4.49
# arch : x86_64
# nrcpus online : 8
# nrcpus avail : 8
# cpudesc : Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz
# cpuid : GenuineIntel,6,79,1
# total memory : 32946256 kB
# cmdline : /usr/lib/linux-tools-4.4.0-72/perf record -e syscalls:sys_enter_connect -ag
# event : name = syscalls:sys_enter_connect, , type = 2, size = 112, config = 0x442, { sample_period, sample_freq } = 1, sample_type =
# HEADER_CPU_TOPOLOGY info available, use -I to display
# HEADER_NUMA_TOPOLOGY info available, use -I to display
# pmu mappings: msr = 6, software = 1, tracepoint = 2, breakpoint = 5
# ========
#
#
# Total Lost Samples: 0
#
# Samples: 44  of event 'syscalls:sys_enter_connect'
# Event count (approx.): 44
#
# Children      Self  Command          Shared Object       Symbol
# ........  ........  ...............  ..................  ......................
#
    27.27%    27.27%  haproxy          libc-2.23.so        [.] uselib@GLIBC_2.2.5
                    |
                    |--18.18%-- uselib@GLIBC_2.2.5
                    |
                     --9.09%-- 0x401
                               uselib@GLIBC_2.2.5

    27.27%    27.27%  sshd             libc-2.23.so        [.] uselib@GLIBC_2.2.5
                       |
                       |--9.09%-- uselib@GLIBC_2.2.5
                       |
                       |--9.09%-- 0x11c25
                       |          |
                       |          |--6.82%-- uselib@GLIBC_2.2.5
                       |          |
                       |           --2.27%-- getaddrinfo
                       |                     nscd_gethst_r
                       |                     __readall
                       |                     uselib@GLIBC_2.2.5
                       |
                       |--4.55%-- 0
                       |          __nscd_get_mapping
                       |          __readall
                       |          uselib@GLIBC_2.2.5
                       |
                       |--2.27%-- 0x848b41b87fc58941
                       |          0x7b904
                       |          __nscd_get_mapping
                       |          uselib@GLIBC_2.2.5
                       |
                        --2.27%-- __nscd_get_mapping
                                  uselib@GLIBC_2.2.5

    18.18%    18.18%  sudo             libc-2.23.so        [.] uselib@GLIBC_2.2.5
                       |
                       |--9.09%-- 0
                       |          __nscd_get_mapping
                       |          |
                       |          |--6.82%-- __readall
                       |          |          uselib@GLIBC_2.2.5
...
```


> 注意：
> 
> - 发现了一个问题，即安装了 libc6-dbgsym 后，原来显示为 connect 的符号均被显示成了 uselib@GLIBC_2.2.5 ，而文档中给出的是 __GI___connect_internal ，似乎并不太好（当然，libc6 版本的不同可能会有差别）；
> - 安装 coreutils-dbgsym 后，显示的内容没有发生变化；


### Socket Buffers

> 确定导致网络 IO 的原因

Tracing the consumption of socket buffers, and the stack traces, is one way to identify what is leading to socket or network I/O.

```
root@backend-shared-stag-0:~/workspace# perf record -e 'skb:consume_skb' -ag
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.252 MB perf.data (527 samples) ]

root@backend-shared-stag-0:~/workspace#


root@backend-shared-stag-0:~/workspace# perf report
Samples: 527  of event 'skb:consume_skb', Event count (approx.): 527
  Children      Self  Command          Shared Object                 Symbol
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] unix_stream_read_generic
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] unix_stream_recvmsg
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] sock_recvmsg
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] sock_read_iter
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] new_sync_read
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] __vfs_read
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] vfs_read
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] sys_read
+   56.93%     0.00%  dockerd          [kernel.kallsyms]             [k] entry_SYSCALL_64_fastpath
+   56.93%     0.00%  dockerd          dockerd                       [.] syscall.Syscall
+   56.93%    56.93%  dockerd          [kernel.kallsyms]             [k] consume_skb
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] unix_stream_read_generic
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] unix_stream_recvmsg
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] sock_recvmsg
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] sock_read_iter
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] new_sync_read
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] __vfs_read
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] vfs_read
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] sys_read
+   18.98%     0.00%  docker-containe  [kernel.kallsyms]             [k] entry_SYSCALL_64_fastpath
+   18.98%     0.00%  docker-containe  docker-containerd             [.] syscall.Syscall
+   18.98%    18.98%  docker-containe  [kernel.kallsyms]             [k] consume_skb
+   18.79%     0.00%  dockerd          [unknown]                     [k] 0000000000000000
+   11.95%     0.00%  dockerd          dockerd                       [.] runtime.goexit
+   11.76%     0.00%  haproxy          [unknown]                     [k] 0000000000000000
+   11.39%     0.00%  dockerd          [unknown]                     [k] 0x00007fcbdcf79cc0
+    9.68%     0.00%  dockerd          dockerd                       [.] runtime.gcBgMarkWorker
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] packet_rcv
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] dev_hard_start_xmit
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] sch_direct_xmit
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] __dev_queue_xmit
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] dev_queue_xmit
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] ip_finish_output2
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] ip_finish_output
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] ip_output
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] ip_local_out
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] ip_queue_xmit
+    9.49%     0.00%  haproxy          [kernel.kallsyms]             [k] tcp_transmit_skb
+    9.49%     9.49%  haproxy          [kernel.kallsyms]             [k] consume_skb
+    8.54%     0.00%  dockerd          dockerd                       [.] github.com/docker/docker/vendor/google.golang.org/grpc/transpo
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] __do_softirq
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] irq_exit
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] xen_evtchn_do_upcall
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] xen_hvm_callback_vector
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] cpu_startup_entry
+    7.78%     0.00%  swapper          [kernel.kallsyms]             [k] start_secondary
+    7.59%     0.00%  swapper          [kernel.kallsyms]             [k] ixgbevf_poll
+    7.59%     0.00%  swapper          [kernel.kallsyms]             [k] net_rx_action
+    7.59%     0.00%  swapper          [kernel.kallsyms]             [k] default_idle
+    7.59%     0.00%  swapper          [kernel.kallsyms]             [k] arch_cpu_idle
+    7.59%     0.00%  swapper          [kernel.kallsyms]             [k] default_idle_call
+    6.83%     6.83%  swapper          [kernel.kallsyms]             [k] napi_consume_skb
+    6.83%     0.00%  docker-containe  [unknown]                     [k] 0x0000000800000008
+    6.83%     0.00%  dockerd          [unknown]                     [k] 0x7fcbdc6f0a826b84
+    6.45%     0.00%  haproxy          [kernel.kallsyms]             [k] entry_SYSCALL_64_fastpath
+    6.26%     0.00%  haproxy          [kernel.kallsyms]             [k] ____fput
+    6.26%     0.00%  haproxy          [kernel.kallsyms]             [k] task_work_run
+    6.26%     0.00%  haproxy          [kernel.kallsyms]             [k] exit_to_usermode_loop
+    6.26%     0.00%  haproxy          [kernel.kallsyms]             [k] syscall_return_slowpath
+    6.26%     0.00%  haproxy          [kernel.kallsyms]             [k] int_ret_from_sys_call
+    6.26%     0.00%  haproxy          libc-2.23.so (deleted)        [.] 0xffff80a0b3bdbd10
+    6.07%     0.00%  haproxy          [kernel.kallsyms]             [k] __fput
+    5.69%     0.00%  dockerd          [unknown]                     [k] 0x000000c42049e738
+    5.12%     0.00%  haproxy          libc-2.23.so (deleted)        [.] 0xffff80a0b3bec540
+    4.93%     0.00%  haproxy          [kernel.kallsyms]             [k] inet_release
For a higher level overview, try: perf report --sort comm,dso
```

The `swapper` stack shows the **network receive path**, triggered by an interrupt. The `sshd` path shows **writes**.

### Static User Tracing

Support was added in later **4.x** series kernels. 

If you are on an older kernel, say, Linux **4.4-4.9**, you can probably get these to work with adjustments (I've even hacked them up with [ftrace](http://www.brendangregg.com/blog/2015-07-03/hacking-linux-usdt-ftrace.html) for older kernels), but since they have been in development, I haven't seen documentation outside of lkml, so you'll need to figure it out. (On this kernel range, you might find more documentation for tracing these with [bcc/eBPF](http://www.brendangregg.com/ebpf.html#bcc), including using the trace.py tool.)

### Dynamic Tracing

For kernel analysis, I'm using `CONFIG_KPROBES=y` and `CONFIG_KPROBE_EVENTS=y`, to **enable kernel dynamic tracing**, and `CONFIG_FRAME_POINTER=y`, for **frame pointer-based kernel stacks**. For user-level analysis, `CONFIG_UPROBES=y` and `CONFIG_UPROBE_EVENTS=y`, for user-level dynamic tracing.

#### Kernel: tcp_sendmsg()

> 追踪 kernel tcp_sendmsg() function

```
# add a new tracepoint event
root@backend-shared-stag-0:~/workspace# perf probe --add tcp_sendmsg
Added new event:
  probe:tcp_sendmsg    (on tcp_sendmsg)

You can now use it in all perf tools, such as:

	perf record -e probe:tcp_sendmsg -aR sleep 1

root@backend-shared-stag-0:~/workspace#


# Tracing this event for 5 seconds, recording stack traces
root@backend-shared-stag-0:~/workspace# perf record -e probe:tcp_sendmsg -a -g -- sleep 5
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.148 MB perf.data (2 samples) ]
root@backend-shared-stag-0:~/workspace#


root@backend-shared-stag-0:~/workspace# perf report --stdio
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 2  of event 'probe:tcp_sendmsg'
# Event count (approx.): 2
#
# Children      Self  Command        Shared Object      Symbol
# ........  ........  .............  .................  .............................
#
   100.00%     0.00%  node_exporter  [kernel.kallsyms]  [k] sock_sendmsg
            |
            ---sock_sendmsg
               tcp_sendmsg

   100.00%     0.00%  node_exporter  [kernel.kallsyms]  [k] sock_write_iter
            |
            ---sock_write_iter
               sock_sendmsg
               tcp_sendmsg

   100.00%     0.00%  node_exporter  [kernel.kallsyms]  [k] new_sync_write
            |
            ---new_sync_write
               sock_write_iter
               sock_sendmsg
               tcp_sendmsg

   100.00%     0.00%  node_exporter  [kernel.kallsyms]  [k] __vfs_write
            |
            ---__vfs_write
               new_sync_write
               sock_write_iter
               sock_sendmsg
               tcp_sendmsg

   100.00%     0.00%  node_exporter  [kernel.kallsyms]  [k] vfs_write
            |
            ---vfs_write
               __vfs_write
               new_sync_write
               sock_write_iter
...

# delete these dynamic tracepoints if you want after use
root@backend-shared-stag-0:~/workspace# perf probe --del tcp_sendmsg
Removed event: probe:tcp_sendmsg
root@backend-shared-stag-0:~/workspace#
```

#### Kernel: tcp_sendmsg() with size

> 失败

If your **kernel has debuginfo** (`CONFIG_DEBUG_INFO=y`), you can fish out kernel variables from functions.

```
root@backend-shared-stag-0:~/workspace# grep CONFIG_DEBUG_INFO /boot/config-`uname -r`
CONFIG_DEBUG_INFO=y
# CONFIG_DEBUG_INFO_REDUCED is not set
# CONFIG_DEBUG_INFO_SPLIT is not set
CONFIG_DEBUG_INFO_DWARF4=y
root@backend-shared-stag-0:~/workspace#

root@backend-shared-stag-0:~/workspace# perf probe -V tcp_sendmsg
Failed to find the path for kernel: Invalid ELF file
  Error: Failed to show vars.
root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#

root@backend-shared-stag-0:~/workspace# perf probe --add 'tcp_sendmsg size'
Failed to find the path for kernel: Invalid ELF file
  Error: Failed to add events.
root@backend-shared-stag-0:~/workspace#
```

#### Kernel: tcp_sendmsg() line number and local variable

> 失败

```
root@backend-shared-stag-0:~/workspace# perf probe -L tcp_sendmsg
Failed to find the path for kernel: Invalid ELF file
  Error: Failed to show lines.
root@backend-shared-stag-0:~/workspace#
```

#### User: malloc()


```
# Adding a libc malloc() probe
root@backend-shared-stag-0:~/workspace# perf probe -x /lib/x86_64-linux-gnu/libc-2.23.so --add malloc
Added new events:
  probe_libc:malloc    (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_1  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_2  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_3  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_4  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_5  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_6  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_7  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_8  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_9  (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_10 (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_11 (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_12 (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_13 (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)
  probe_libc:malloc_14 (on malloc in /lib/x86_64-linux-gnu/libc-2.23.so)

You can now use it in all perf tools, such as:

	perf record -e probe_libc:malloc_14 -aR sleep 1

root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#


# Tracing it system-wide
root@backend-shared-stag-0:~/workspace# perf record -e probe_libc:malloc -a
^C[ perf record: Woken up 6 times to write data ]
[ perf record: Captured and wrote 2.039 MB perf.data (26144 samples) ]

root@backend-shared-stag-0:~/workspace#


# 查看 report
root@backend-shared-stag-0:~/workspace# perf report -n
Samples: 26K of event 'probe_libc:malloc', Event count (approx.): 26144
Overhead       Samples  Command          Shared Object  Symbol
  38.12%          9965  lsb_release      libc-2.23.so   [.] malloc
  21.19%          5540  modprobe         libc-2.23.so   [.] malloc
  14.41%          3767  sshd             libc-2.23.so   [.] malloc
   3.54%           926  grep             libc-2.23.so   [.] malloc
   3.44%           900  dircolors        libc-2.23.so   [.] malloc
   3.25%           849  sudo             libc-2.23.so   [.] malloc
   3.01%           787  awk              libc-2.23.so   [.] malloc
   1.59%           416  groups           libc-2.23.so   [.] malloc
   1.38%           361  lesspipe         libc-2.23.so   [.] malloc
   1.12%           292  basename         libc-2.23.so   [.] malloc
   1.11%           290  dirname          libc-2.23.so   [.] malloc
   1.10%           288  top              libc-2.23.so   [.] malloc
   1.01%           264  mount            libc-2.23.so   [.] malloc
   1.01%           263  update-motd-fsc  libc-2.23.so   [.] malloc
   0.91%           238  dumpe2fs         libc-2.23.so   [.] malloc
   0.88%           231  run-parts        libc-2.23.so   [.] malloc
   0.84%           219  mesg             libc-2.23.so   [.] malloc
   0.24%            63  ls               libc-2.23.so   [.] malloc
   0.23%            60  cut              libc-2.23.so   [.] malloc
   0.22%            58  systemd-cgroups  libc-2.23.so   [.] malloc
   0.22%            57  date             libc-2.23.so   [.] malloc
   0.18%            48  00-header        libc-2.23.so   [.] malloc
   0.16%            41  release-upgrade  libc-2.23.so   [.] malloc
   0.12%            31  env              libc-2.23.so   [.] malloc
   0.08%            21  91-release-upgr  libc-2.23.so   [.] malloc
   0.07%            17  10-help-text     libc-2.23.so   [.] malloc
   0.07%            17  90-updates-avai  libc-2.23.so   [.] malloc
   0.06%            16  97-overlayroot   libc-2.23.so   [.] malloc
   0.05%            14  stat             libc-2.23.so   [.] malloc
   0.05%            13  sh               libc-2.23.so   [.] malloc
   0.05%            13  sort             libc-2.23.so   [.] malloc
   0.05%            13  update-motd-reb  libc-2.23.so   [.] malloc
   0.05%            12  98-fsck-at-rebo  libc-2.23.so   [.] malloc
   0.05%            12  98-reboot-requi  libc-2.23.so   [.] malloc
   0.04%            10  51-cloudguest    libc-2.23.so   [.] malloc
   0.03%             9  cat              libc-2.23.so   [.] malloc
   0.03%             9  uname            libc-2.23.so   [.] malloc
   0.03%             7  egrep            libc-2.23.so   [.] malloc
   0.03%             7  expr             libc-2.23.so   [.] malloc
```

> 在我的实验中，最多的是 malloc 是由 lsb_release 导致的；

This shows the most `malloc()` calls were by `apt-config`, while I was tracing.

#### User: malloc() with size

失败

### Scheduler Analysis

The `perf sched` subcommand provides a number of tools for **analyzing kernel CPU scheduler behavior**. You can use this to identify and quantify issues of **scheduler latency**.

The current **overhead** of this tool (as of up to Linux 4.10) may be **noticeable**, as it instruments and dumps scheduler events to the `perf.data` file for later analysis. For example:

```
# perf sched record -- sleep 1
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.886 MB perf.data (13502 samples) ]
```

That's 1.9 Mbytes for one second, including 13,502 samples. **The size and rate will be relative to your workload and number of CPUs** (this example is an 8 CPU server running a software build). How this is written to the file system has been optimized: it only woke up one time to read the event buffers and write them to disk, which greatly reduces overhead. That said, **there are still significant overheads with instrumenting all scheduler events and writing event data to the file system**.

These events:

```
# perf script --header
# ========
# captured on: Sun Feb 26 19:40:00 2017
# hostname : bgregg-xenial
# os release : 4.10-virtual
# perf version : 4.10
# arch : x86_64
# nrcpus online : 8
# nrcpus avail : 8
# cpudesc : Intel(R) Xeon(R) CPU E5-2680 v2 @ 2.80GHz
# cpuid : GenuineIntel,6,62,4
# total memory : 15401700 kB
# cmdline : /usr/bin/perf sched record -- sleep 1 
# event : name = sched:sched_switch, , id = { 2752, 2753, 2754, 2755, 2756, 2757, 2758, 2759 }, type = 2, size = 11...
# event : name = sched:sched_stat_wait, , id = { 2760, 2761, 2762, 2763, 2764, 2765, 2766, 2767 }, type = 2, size =...
# event : name = sched:sched_stat_sleep, , id = { 2768, 2769, 2770, 2771, 2772, 2773, 2774, 2775 }, type = 2, size ...
# event : name = sched:sched_stat_iowait, , id = { 2776, 2777, 2778, 2779, 2780, 2781, 2782, 2783 }, type = 2, size...
# event : name = sched:sched_stat_runtime, , id = { 2784, 2785, 2786, 2787, 2788, 2789, 2790, 2791 }, type = 2, siz...
# event : name = sched:sched_process_fork, , id = { 2792, 2793, 2794, 2795, 2796, 2797, 2798, 2799 }, type = 2, siz...
# event : name = sched:sched_wakeup, , id = { 2800, 2801, 2802, 2803, 2804, 2805, 2806, 2807 }, type = 2, size = 11...
# event : name = sched:sched_wakeup_new, , id = { 2808, 2809, 2810, 2811, 2812, 2813, 2814, 2815 }, type = 2, size ...
# event : name = sched:sched_migrate_task, , id = { 2816, 2817, 2818, 2819, 2820, 2821, 2822, 2823 }, type = 2, siz...
# HEADER_CPU_TOPOLOGY info available, use -I to display
# HEADER_NUMA_TOPOLOGY info available, use -I to display
# pmu mappings: breakpoint = 5, power = 7, software = 1, tracepoint = 2, msr = 6
# HEADER_CACHE info available, use -I to display
# missing features: HEADER_BRANCH_STACK HEADER_GROUP_DESC HEADER_AUXTRACE HEADER_STAT 
# ========
#
    perf 16984 [005] 991962.879966:       sched:sched_wakeup: comm=perf pid=16999 prio=120 target_cpu=005
[...]
```

If overhead is a problem, you can use my [eBPF/bcc Tools](http://www.brendangregg.com/ebpf.html#bcc) including `runqlat` and `runqlen` which **use in-kernel summaries of scheduler events**, reducing overhead further. **An advantage of `perf sched` dumping all events is that you aren't limited to the summary**. If you caught an intermittent event, you can analyze those recorded events in custom ways until you understood the issue, rather than needing to catch it a second time.

> 这里说明了 `perf sched` 和 `runqlat` 、`runqlen` 的使用差别；

The captured trace file can be reported in a number of ways, summarized by the help message:

```
root@backend-shared-stag-0:~# perf sched -h

 Usage: perf sched [<options>] {record|latency|map|replay|script}

    -D, --dump-raw-trace  dump raw trace in ASCII
    -i, --input <file>    input file name
    -v, --verbose         be more verbose (show symbol address, etc)

root@backend-shared-stag-0:~#
```

`perf sched latency` will **summarize scheduler latencies by task**, including average and maximum delay:

```
root@backend-shared-stag-0:~/workspace# perf sched latency

 -----------------------------------------------------------------------------------------------------------------
  Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms | Maximum delay at       |
 -----------------------------------------------------------------------------------------------------------------
  docker-containe:(3)   |      0.954 ms |       11 | avg:    0.021 ms | max:    0.045 ms | max at: 9246430.852946 s
  dockerd:(5)           |      1.475 ms |       32 | avg:    0.017 ms | max:    0.043 ms | max at: 9246431.353168 s
  kworker/7:2:6649      |      0.023 ms |        1 | avg:    0.015 ms | max:    0.015 ms | max at: 9246430.764103 s
  kworker/u30:0:11450   |      0.038 ms |        2 | avg:    0.015 ms | max:    0.022 ms | max at: 9246431.024106 s
  iscsid:(2)            |      0.105 ms |        5 | avg:    0.012 ms | max:    0.018 ms | max at: 9246431.490388 s
  kworker/6:1:4854      |      0.015 ms |        1 | avg:    0.012 ms | max:    0.012 ms | max at: 9246430.740079 s
  ntpd:22235            |      0.038 ms |        1 | avg:    0.012 ms | max:    0.012 ms | max at: 9246430.711908 s
  perf:11527            |      0.472 ms |        1 | avg:    0.011 ms | max:    0.011 ms | max at: 9246431.573316 s
  kworker/3:1:11369     |      0.019 ms |        1 | avg:    0.011 ms | max:    0.011 ms | max at: 9246430.764088 s
  sleep:11530           |      0.729 ms |        2 | avg:    0.010 ms | max:    0.011 ms | max at: 9246430.572482 s
  kworker/0:0:8384      |      0.012 ms |        1 | avg:    0.010 ms | max:    0.010 ms | max at: 9246430.812075 s
  kworker/5:0:15448     |      0.017 ms |        1 | avg:    0.010 ms | max:    0.010 ms | max at: 9246430.764092 s
  kworker/4:2:10988     |      0.027 ms |        1 | avg:    0.008 ms | max:    0.008 ms | max at: 9246430.764070 s
  haproxy:8083          |      0.019 ms |        1 | avg:    0.008 ms | max:    0.008 ms | max at: 9246430.776162 s
  kworker/1:1:7517      |      0.009 ms |        1 | avg:    0.007 ms | max:    0.007 ms | max at: 9246430.800071 s
  rcu_sched:7           |      0.050 ms |        8 | avg:    0.005 ms | max:    0.007 ms | max at: 9246430.576081 s
  migration/4:27        |      0.000 ms |        1 | avg:    0.002 ms | max:    0.002 ms | max at: 9246430.572571 s
 -----------------------------------------------------------------------------------------------------------------
  TOTAL:                |      4.002 ms |       71 |
 ---------------------------------------------------

root@backend-shared-stag-0:~/workspace#
```

Here are the raw events from `perf sched script`:


```
root@backend-shared-stag-0:~/workspace# perf sched script
         swapper     0 [001] 9246430.572061: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3994140 [ns]
         swapper     0 [001] 9246430.572062: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.572068: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=3724 [ns]
         swapper     0 [001] 9246430.572069: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.572074: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=10165 [ns] vruntime=5275746373411 [ns]
       rcu_sched     7 [001] 9246430.572074: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
            perf 11527 [003] 9246430.572465: sched:sched_stat_sleep: comm=perf pid=11530 delay=24653035 [ns]
            perf 11527 [003] 9246430.572470: sched:sched_wakeup: comm=perf pid=11530 prio=120 target_cpu=004
         swapper     0 [004] 9246430.572480: sched:sched_stat_wait: comm=perf pid=11530 delay=0 [ns]
         swapper     0 [004] 9246430.572481: sched:sched_switch: prev_comm=swapper/4 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=perf next_pid=11530 next_prio=120
            perf 11527 [003] 9246430.572529: sched:sched_stat_runtime: comm=perf pid=11527 runtime=472301 [ns] vruntime=361713165732 [ns]
            perf 11527 [003] 9246430.572530: sched:sched_switch: prev_comm=perf prev_pid=11527 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
            perf 11530 [004] 9246430.572568: sched:sched_wakeup: comm=migration/4 pid=27 prio=0 target_cpu=004
            perf 11530 [004] 9246430.572570: sched:sched_stat_runtime: comm=perf pid=11530 runtime=104265 [ns] vruntime=434496001441 [ns]
            perf 11530 [004] 9246430.572571: sched:sched_switch: prev_comm=perf prev_pid=11530 prev_prio=120 prev_state=R+ ==> next_comm=migration/4 next_pid=27 next_prio=0
     migration/4    27 [004] 9246430.572573: sched:sched_stat_wait: comm=perf pid=11530 delay=4432 [ns]
     migration/4    27 [004] 9246430.572574: sched:sched_migrate_task: comm=perf pid=11530 prio=120 orig_cpu=4 dest_cpu=5
     migration/4    27 [004] 9246430.572583: sched:sched_switch: prev_comm=migration/4 prev_pid=27 prev_prio=0 prev_state=S ==> next_comm=swapper/4 next_pid=0 next_prio=120
         swapper     0 [005] 9246430.572591: sched:sched_stat_wait: comm=perf pid=11530 delay=0 [ns]
         swapper     0 [005] 9246430.572592: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=perf next_pid=11530 next_prio=120
           sleep 11530 [005] 9246430.573101: sched:sched_stat_runtime: comm=sleep pid=11530 runtime=525287 [ns] vruntime=589867932087 [ns]
           sleep 11530 [005] 9246430.573103: sched:sched_switch: prev_comm=sleep prev_pid=11530 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.576072: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3997352 [ns]
         swapper     0 [001] 9246430.576073: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.576079: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=3952 [ns]
         swapper     0 [001] 9246430.576080: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.576083: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=8136 [ns] vruntime=5275746381547 [ns]
       rcu_sched     7 [001] 9246430.576084: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.580067: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3984109 [ns]
         swapper     0 [001] 9246430.580068: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.580073: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=2367 [ns]
         swapper     0 [001] 9246430.580074: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.580076: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=6571 [ns] vruntime=5275746388118 [ns]
       rcu_sched     7 [001] 9246430.580077: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.584066: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3989587 [ns]
         swapper     0 [001] 9246430.584067: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.584071: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=2158 [ns]
         swapper     0 [001] 9246430.584071: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.584073: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=5682 [ns] vruntime=5275746393800 [ns]
       rcu_sched     7 [001] 9246430.584074: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.588065: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3991743 [ns]
         swapper     0 [001] 9246430.588066: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.588071: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=2149 [ns]
         swapper     0 [001] 9246430.588071: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.588074: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=6402 [ns] vruntime=5275746400202 [ns]
       rcu_sched     7 [001] 9246430.588075: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.592063: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3989513 [ns]
         swapper     0 [001] 9246430.592064: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.592067: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=1465 [ns]
         swapper     0 [001] 9246430.592068: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.592069: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=4785 [ns] vruntime=5275746404987 [ns]
       rcu_sched     7 [001] 9246430.592070: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.596063: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3993227 [ns]
         swapper     0 [001] 9246430.596063: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.596067: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=1495 [ns]
         swapper     0 [001] 9246430.596067: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.596068: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=4509 [ns] vruntime=5275746409496 [ns]
       rcu_sched     7 [001] 9246430.596069: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.600061: sched:sched_stat_sleep: comm=rcu_sched pid=7 delay=3992351 [ns]
         swapper     0 [001] 9246430.600062: sched:sched_wakeup: comm=rcu_sched pid=7 prio=120 target_cpu=001
         swapper     0 [001] 9246430.600065: sched:sched_stat_wait: comm=rcu_sched pid=7 delay=1425 [ns]
         swapper     0 [001] 9246430.600065: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=rcu_sched next_pid=7 next_prio=120
       rcu_sched     7 [001] 9246430.600066: sched:sched_stat_runtime: comm=rcu_sched pid=7 runtime=4034 [ns] vruntime=5275746413530 [ns]
       rcu_sched     7 [001] 9246430.600067: sched:sched_switch: prev_comm=rcu_sched prev_pid=7 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [002] 9246430.711893: sched:sched_stat_sleep: comm=ntpd pid=22235 delay=999951061 [ns]
         swapper     0 [002] 9246430.711896: sched:sched_wakeup: comm=ntpd pid=22235 prio=120 target_cpu=002
         swapper     0 [002] 9246430.711907: sched:sched_stat_wait: comm=ntpd pid=22235 delay=0 [ns]
         swapper     0 [002] 9246430.711908: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=ntpd next_pid=22235 next_prio=120
            ntpd 22235 [002] 9246430.711930: sched:sched_stat_runtime: comm=ntpd pid=22235 runtime=37579 [ns] vruntime=925367992 [ns]
            ntpd 22235 [002] 9246430.711932: sched:sched_switch: prev_comm=ntpd prev_pid=22235 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.739379: sched:sched_stat_sleep: comm=iscsid pid=22378 delay=250304933 [ns]
         swapper     0 [001] 9246430.739380: sched:sched_wakeup: comm=iscsid pid=22378 prio=110 target_cpu=001
         swapper     0 [001] 9246430.739388: sched:sched_stat_wait: comm=iscsid pid=22378 delay=0 [ns]
         swapper     0 [001] 9246430.739388: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=iscsid next_pid=22378 next_prio=110
          iscsid 22378 [001] 9246430.739397: sched:sched_stat_runtime: comm=iscsid pid=22378 runtime=18279 [ns] vruntime=368716160 [ns]
          iscsid 22378 [001] 9246430.739398: sched:sched_switch: prev_comm=iscsid prev_pid=22378 prev_prio=110 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [006] 9246430.740065: sched:sched_stat_sleep: comm=kworker/6:1 pid=4854 delay=975968154 [ns]
         swapper     0 [006] 9246430.740067: sched:sched_wakeup: comm=kworker/6:1 pid=4854 prio=120 target_cpu=006
         swapper     0 [006] 9246430.740078: sched:sched_stat_wait: comm=kworker/6:1 pid=4854 delay=3217 [ns]
         swapper     0 [006] 9246430.740079: sched:sched_switch: prev_comm=swapper/6 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/6:1 next_pid=4854 next_prio=120
     kworker/6:1  4854 [006] 9246430.740083: sched:sched_stat_runtime: comm=kworker/6:1 pid=4854 runtime=15108 [ns] vruntime=3671528586426 [ns]
     kworker/6:1  4854 [006] 9246430.740084: sched:sched_switch: prev_comm=kworker/6:1 prev_pid=4854 prev_prio=120 prev_state=S ==> next_comm=swapper/6 next_pid=0 next_prio=120
         swapper     0 [004] 9246430.764061: sched:sched_stat_sleep: comm=kworker/4:2 pid=10988 delay=999977172 [ns]
         swapper     0 [004] 9246430.764062: sched:sched_wakeup: comm=kworker/4:2 pid=10988 prio=120 target_cpu=004
         swapper     0 [004] 9246430.764069: sched:sched_stat_wait: comm=kworker/4:2 pid=10988 delay=2810 [ns]
         swapper     0 [004] 9246430.764070: sched:sched_switch: prev_comm=swapper/4 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/4:2 next_pid=10988 next_prio=120
     kworker/4:2 10988 [004] 9246430.764073: sched:sched_stat_sleep: comm=kworker/3:1 pid=11369 delay=5987979397 [ns]
     kworker/4:2 10988 [004] 9246430.764076: sched:sched_wakeup: comm=kworker/3:1 pid=11369 prio=120 target_cpu=003
     kworker/4:2 10988 [004] 9246430.764079: sched:sched_stat_sleep: comm=kworker/5:0 pid=15448 delay=1012009692 [ns]
     kworker/4:2 10988 [004] 9246430.764082: sched:sched_wakeup: comm=kworker/5:0 pid=15448 prio=120 target_cpu=005
     kworker/4:2 10988 [004] 9246430.764084: sched:sched_stat_sleep: comm=kworker/7:2 pid=6649 delay=60035982405 [ns]
         swapper     0 [003] 9246430.764086: sched:sched_stat_wait: comm=kworker/3:1 pid=11369 delay=0 [ns]
     kworker/4:2 10988 [004] 9246430.764087: sched:sched_wakeup: comm=kworker/7:2 pid=6649 prio=120 target_cpu=007
         swapper     0 [003] 9246430.764087: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/3:1 next_pid=11369 next_prio=120
     kworker/4:2 10988 [004] 9246430.764090: sched:sched_stat_runtime: comm=kworker/4:2 pid=10988 runtime=27213 [ns] vruntime=4007564383790 [ns]
         swapper     0 [005] 9246430.764091: sched:sched_stat_wait: comm=kworker/5:0 pid=15448 delay=0 [ns]
     kworker/4:2 10988 [004] 9246430.764091: sched:sched_switch: prev_comm=kworker/4:2 prev_pid=10988 prev_prio=120 prev_state=S ==> next_comm=swapper/4 next_pid=0 next_prio=120
         swapper     0 [005] 9246430.764092: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/5:0 next_pid=15448 next_prio=120
     kworker/3:1 11369 [003] 9246430.764093: sched:sched_stat_runtime: comm=kworker/3:1 pid=11369 runtime=19397 [ns] vruntime=5454921579789 [ns]
     kworker/3:1 11369 [003] 9246430.764094: sched:sched_switch: prev_comm=kworker/3:1 prev_pid=11369 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
     kworker/5:0 15448 [005] 9246430.764096: sched:sched_stat_runtime: comm=kworker/5:0 pid=15448 runtime=16872 [ns] vruntime=4092726984820 [ns]
     kworker/5:0 15448 [005] 9246430.764097: sched:sched_switch: prev_comm=kworker/5:0 prev_pid=15448 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [007] 9246430.764101: sched:sched_stat_wait: comm=kworker/7:2 pid=6649 delay=0 [ns]
         swapper     0 [007] 9246430.764102: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/7:2 next_pid=6649 next_prio=120
     kworker/7:2  6649 [007] 9246430.764107: sched:sched_stat_runtime: comm=kworker/7:2 pid=6649 runtime=22669 [ns] vruntime=3873829845781 [ns]
     kworker/7:2  6649 [007] 9246430.764109: sched:sched_switch: prev_comm=kworker/7:2 prev_pid=6649 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
         swapper     0 [005] 9246430.776153: sched:sched_stat_sleep: comm=haproxy pid=8083 delay=1001057216 [ns]
         swapper     0 [005] 9246430.776154: sched:sched_wakeup: comm=haproxy pid=8083 prio=120 target_cpu=005
         swapper     0 [005] 9246430.776161: sched:sched_stat_wait: comm=haproxy pid=8083 delay=0 [ns]
         swapper     0 [005] 9246430.776162: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=haproxy next_pid=8083 next_prio=120
         haproxy  8083 [005] 9246430.776171: sched:sched_stat_runtime: comm=haproxy pid=8083 runtime=19075 [ns] vruntime=130042815876 [ns]
         haproxy  8083 [005] 9246430.776172: sched:sched_switch: prev_comm=haproxy prev_pid=8083 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.800063: sched:sched_stat_sleep: comm=kworker/1:1 pid=7517 delay=999988024 [ns]
         swapper     0 [001] 9246430.800063: sched:sched_wakeup: comm=kworker/1:1 pid=7517 prio=120 target_cpu=001
         swapper     0 [001] 9246430.800070: sched:sched_stat_wait: comm=kworker/1:1 pid=7517 delay=2337 [ns]
         swapper     0 [001] 9246430.800071: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/1:1 next_pid=7517 next_prio=120
     kworker/1:1  7517 [001] 9246430.800074: sched:sched_stat_runtime: comm=kworker/1:1 pid=7517 runtime=9404 [ns] vruntime=5275746330192 [ns]
     kworker/1:1  7517 [001] 9246430.800075: sched:sched_switch: prev_comm=kworker/1:1 prev_pid=7517 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [000] 9246430.812063: sched:sched_stat_sleep: comm=kworker/0:0 pid=8384 delay=999961411 [ns]
         swapper     0 [000] 9246430.812064: sched:sched_wakeup: comm=kworker/0:0 pid=8384 prio=120 target_cpu=000
         swapper     0 [000] 9246430.812073: sched:sched_stat_wait: comm=kworker/0:0 pid=8384 delay=3194 [ns]
         swapper     0 [000] 9246430.812074: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/0:0 next_pid=8384 next_prio=120
     kworker/0:0  8384 [000] 9246430.812078: sched:sched_stat_runtime: comm=kworker/0:0 pid=8384 runtime=12079 [ns] vruntime=5291051106967 [ns]
     kworker/0:0  8384 [000] 9246430.812079: sched:sched_switch: prev_comm=kworker/0:0 prev_pid=8384 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.852660: sched:sched_stat_sleep: comm=dockerd pid=3723 delay=407633374 [ns]
         swapper     0 [001] 9246430.852661: sched:sched_wakeup: comm=dockerd pid=3723 prio=120 target_cpu=001
         swapper     0 [001] 9246430.852674: sched:sched_stat_wait: comm=dockerd pid=3723 delay=0 [ns]
         swapper     0 [001] 9246430.852675: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=3723 next_prio=120
         dockerd  3723 [001] 9246430.852688: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=407329378 [ns]
         dockerd  3723 [001] 9246430.852696: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         dockerd  3723 [001] 9246430.852708: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=407506813 [ns]
         dockerd  3723 [001] 9246430.852716: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=002
         swapper     0 [000] 9246430.852718: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.852719: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  3723 [001] 9246430.852722: sched:sched_stat_sleep: comm=dockerd pid=1273 delay=407682132 [ns]
         dockerd  3723 [001] 9246430.852729: sched:sched_wakeup: comm=dockerd pid=1273 prio=120 target_cpu=005
         dockerd  3723 [001] 9246430.852736: sched:sched_stat_runtime: comm=dockerd pid=3723 runtime=76999 [ns] vruntime=1550702074219 [ns]
         dockerd  3723 [001] 9246430.852740: sched:sched_switch: prev_comm=dockerd prev_pid=3723 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [002] 9246430.852740: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         dockerd  1155 [000] 9246430.852741: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=52955 [ns] vruntime=1573789529834 [ns]
         swapper     0 [002] 9246430.852741: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1155 [000] 9246430.852744: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  1271 [002] 9246430.852760: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=51528 [ns] vruntime=1595923661411 [ns]
         swapper     0 [005] 9246430.852760: sched:sched_stat_wait: comm=dockerd pid=1273 delay=0 [ns]
         swapper     0 [005] 9246430.852762: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1273 next_prio=120
         dockerd  1271 [002] 9246430.852763: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         dockerd  1273 [005] 9246430.852890: sched:sched_stat_sleep: comm=docker-containe pid=3247 delay=499975500 [ns]
         swapper     0 [000] 9246430.852899: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=157884 [ns]
         dockerd  1273 [005] 9246430.852901: sched:sched_wakeup: comm=docker-containe pid=3247 prio=120 target_cpu=002
         swapper     0 [000] 9246430.852901: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [002] 9246430.852911: sched:sched_migrate_task: comm=dockerd pid=1271 prio=120 orig_cpu=2 dest_cpu=3
         swapper     0 [002] 9246430.852914: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=153688 [ns]
         dockerd  1273 [005] 9246430.852916: sched:sched_stat_runtime: comm=dockerd pid=1273 runtime=194043 [ns] vruntime=1264672338275 [ns]
         dockerd  1273 [005] 9246430.852919: sched:sched_switch: prev_comm=dockerd prev_pid=1273 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [000] 9246430.852920: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.852921: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         swapper     0 [002] 9246430.852922: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=003
         dockerd  1155 [000] 9246430.852933: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=34825 [ns] vruntime=1573789564659 [ns]
         dockerd  1155 [000] 9246430.852935: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [002] 9246430.852944: sched:sched_stat_wait: comm=docker-containe pid=3247 delay=0 [ns]
         swapper     0 [002] 9246430.852946: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=3247 next_prio=120
         swapper     0 [003] 9246430.852947: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [003] 9246430.852948: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1271 [003] 9246430.852959: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=45588 [ns] vruntime=1576447409657 [ns]
         dockerd  1271 [003] 9246430.852962: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
 docker-containe  3247 [002] 9246430.852966: sched:sched_stat_sleep: comm=docker-containe pid=1276 delay=500011977 [ns]
 docker-containe  3247 [002] 9246430.852975: sched:sched_wakeup: comm=docker-containe pid=1276 prio=120 target_cpu=007
 docker-containe  3247 [002] 9246430.852985: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=500051322 [ns]
 docker-containe  3247 [002] 9246430.852993: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         swapper     0 [007] 9246430.853001: sched:sched_stat_wait: comm=docker-containe pid=1276 delay=0 [ns]
         swapper     0 [007] 9246430.853002: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=1276 next_prio=120
         swapper     0 [003] 9246430.853017: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [003] 9246430.853018: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
 docker-containe  1276 [007] 9246430.853027: sched:sched_stat_runtime: comm=docker-containe pid=1276 runtime=60754 [ns] vruntime=1272541118579 [ns]
         dockerd  1274 [003] 9246430.853028: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=42541 [ns] vruntime=1576449529287 [ns]
         dockerd  1274 [003] 9246430.853031: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
 docker-containe  1276 [007] 9246430.853031: sched:sched_switch: prev_comm=docker-containe prev_pid=1276 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
 docker-containe  3247 [002] 9246430.853078: sched:sched_stat_sleep: comm=docker-containe pid=31466 delay=500173095 [ns]
 docker-containe  3247 [002] 9246430.853082: sched:sched_wakeup: comm=docker-containe pid=31466 prio=120 target_cpu=004
         swapper     0 [000] 9246430.853086: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=152677 [ns]
         swapper     0 [000] 9246430.853087: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [004] 9246430.853095: sched:sched_stat_wait: comm=docker-containe pid=31466 delay=0 [ns]
         swapper     0 [004] 9246430.853096: sched:sched_switch: prev_comm=swapper/4 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=31466 next_prio=120
         swapper     0 [000] 9246430.853097: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.853098: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  1155 [000] 9246430.853102: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=17136 [ns] vruntime=1573789581795 [ns]
         dockerd  1155 [000] 9246430.853104: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
 docker-containe 31466 [004] 9246430.853107: sched:sched_stat_runtime: comm=docker-containe pid=31466 runtime=29302 [ns] vruntime=1274753916172 [ns]
 docker-containe 31466 [004] 9246430.853109: sched:sched_switch: prev_comm=docker-containe prev_pid=31466 prev_prio=120 prev_state=S ==> next_comm=swapper/4 next_pid=0 next_prio=120
 docker-containe  3247 [002] 9246430.853157: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=129562 [ns]
 docker-containe  3247 [002] 9246430.853162: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         swapper     0 [003] 9246430.853171: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [003] 9246430.853172: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
         dockerd  1274 [003] 9246430.853181: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=77711 [ns]
         swapper     0 [007] 9246430.853181: sched:sched_stat_sleep: comm=docker-containe pid=1276 delay=153651 [ns]
         swapper     0 [007] 9246430.853182: sched:sched_wakeup: comm=docker-containe pid=1276 prio=120 target_cpu=007
 docker-containe  3247 [002] 9246430.853187: sched:sched_stat_runtime: comm=docker-containe pid=3247 runtime=297534 [ns] vruntime=1595932460466 [ns]
         dockerd  1274 [003] 9246430.853187: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
 docker-containe  3247 [002] 9246430.853188: sched:sched_switch: prev_comm=docker-containe prev_pid=3247 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246430.853192: sched:sched_stat_sleep: comm=docker-containe pid=31466 delay=84706 [ns]
         swapper     0 [007] 9246430.853196: sched:sched_stat_wait: comm=docker-containe pid=1276 delay=0 [ns]
         swapper     0 [007] 9246430.853196: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=1276 next_prio=120
         dockerd  1274 [003] 9246430.853197: sched:sched_wakeup: comm=docker-containe pid=31466 prio=120 target_cpu=004
         swapper     0 [000] 9246430.853199: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.853200: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
 docker-containe  1276 [007] 9246430.853201: sched:sched_stat_runtime: comm=docker-containe pid=1276 runtime=21575 [ns] vruntime=1272541140154 [ns]
 docker-containe  1276 [007] 9246430.853202: sched:sched_switch: prev_comm=docker-containe prev_pid=1276 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
         dockerd  1155 [000] 9246430.853208: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=27961 [ns] vruntime=1573789609756 [ns]
         swapper     0 [004] 9246430.853209: sched:sched_stat_wait: comm=docker-containe pid=31466 delay=0 [ns]
         dockerd  1155 [000] 9246430.853210: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [004] 9246430.853210: sched:sched_switch: prev_comm=swapper/4 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=31466 next_prio=120
 docker-containe 31466 [004] 9246430.853214: sched:sched_stat_runtime: comm=docker-containe pid=31466 runtime=21847 [ns] vruntime=1274753938019 [ns]
 docker-containe 31466 [004] 9246430.853215: sched:sched_switch: prev_comm=docker-containe prev_pid=31466 prev_prio=120 prev_state=S ==> next_comm=swapper/4 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246430.853225: sched:sched_migrate_task: comm=dockerd pid=1271 prio=120 orig_cpu=3 dest_cpu=0
         dockerd  1274 [003] 9246430.853226: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=268006 [ns]
         dockerd  1274 [003] 9246430.853231: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=000
         swapper     0 [000] 9246430.853236: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [000] 9246430.853237: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1271 [000] 9246430.853242: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=15762 [ns] vruntime=1573781169585 [ns]
         dockerd  1271 [000] 9246430.853243: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246430.853274: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=117237 [ns] vruntime=1576449646524 [ns]
         dockerd  1274 [003] 9246430.853276: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
         swapper     0 [000] 9246430.853362: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=154013 [ns]
         swapper     0 [000] 9246430.853363: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [000] 9246430.853365: sched:sched_migrate_task: comm=dockerd pid=1271 prio=120 orig_cpu=0 dest_cpu=1
         swapper     0 [000] 9246430.853366: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=123659 [ns]
         swapper     0 [000] 9246430.853369: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=001
         swapper     0 [000] 9246430.853377: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.853378: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         swapper     0 [001] 9246430.853379: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [001] 9246430.853379: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1155 [000] 9246430.853383: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=20936 [ns] vruntime=1573789630692 [ns]
         dockerd  1271 [001] 9246430.853384: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=17774 [ns] vruntime=1550693651822 [ns]
         dockerd  1155 [000] 9246430.853384: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  1271 [001] 9246430.853385: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [000] 9246430.853538: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=154597 [ns]
         swapper     0 [000] 9246430.853539: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [000] 9246430.853546: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246430.853547: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  1155 [000] 9246430.853549: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=11762 [ns] vruntime=1573789642454 [ns]
         dockerd  1155 [000] 9246430.853550: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [001] 9246430.989712: sched:sched_stat_sleep: comm=iscsid pid=22378 delay=250314596 [ns]
         swapper     0 [001] 9246430.989714: sched:sched_wakeup: comm=iscsid pid=22378 prio=110 target_cpu=001
         swapper     0 [001] 9246430.989728: sched:sched_stat_wait: comm=iscsid pid=22378 delay=0 [ns]
         swapper     0 [001] 9246430.989729: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=iscsid next_pid=22378 next_prio=110
          iscsid 22378 [001] 9246430.989737: sched:sched_stat_runtime: comm=iscsid pid=22378 runtime=25579 [ns] vruntime=368718903 [ns]
          iscsid 22378 [001] 9246430.989739: sched:sched_switch: prev_comm=iscsid prev_pid=22378 prev_prio=110 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [002] 9246431.024081: sched:sched_stat_sleep: comm=kworker/u30:0 pid=11450 delay=503083547 [ns]
         swapper     0 [002] 9246431.024083: sched:sched_wakeup: comm=kworker/u30:0 pid=11450 prio=120 target_cpu=002
         swapper     0 [002] 9246431.024104: sched:sched_stat_wait: comm=kworker/u30:0 pid=11450 delay=5397 [ns]
         swapper     0 [002] 9246431.024105: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/u30:0 next_pid=11450 next_prio=120
   kworker/u30:0 11450 [002] 9246431.024110: sched:sched_stat_runtime: comm=kworker/u30:0 pid=11450 runtime=24332 [ns] vruntime=5414310019366 [ns]
   kworker/u30:0 11450 [002] 9246431.024111: sched:sched_switch: prev_comm=kworker/u30:0 prev_pid=11450 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         swapper     0 [001] 9246431.070933: sched:sched_stat_sleep: comm=iscsid pid=22377 delay=1000115517 [ns]
         swapper     0 [001] 9246431.070935: sched:sched_wakeup: comm=iscsid pid=22377 prio=120 target_cpu=001
         swapper     0 [001] 9246431.070948: sched:sched_stat_wait: comm=iscsid pid=22377 delay=0 [ns]
         swapper     0 [001] 9246431.070949: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=iscsid next_pid=22377 next_prio=120
          iscsid 22377 [001] 9246431.070956: sched:sched_stat_runtime: comm=iscsid pid=22377 runtime=22835 [ns] vruntime=368863642 [ns]
          iscsid 22377 [001] 9246431.070959: sched:sched_switch: prev_comm=iscsid prev_pid=22377 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [001] 9246431.240040: sched:sched_stat_sleep: comm=iscsid pid=22378 delay=250303937 [ns]
         swapper     0 [001] 9246431.240041: sched:sched_wakeup: comm=iscsid pid=22378 prio=110 target_cpu=001
         swapper     0 [001] 9246431.240047: sched:sched_stat_wait: comm=iscsid pid=22378 delay=0 [ns]
         swapper     0 [001] 9246431.240048: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=iscsid next_pid=22378 next_prio=110
          iscsid 22378 [001] 9246431.240051: sched:sched_stat_runtime: comm=iscsid pid=22378 runtime=10754 [ns] vruntime=368720056 [ns]
          iscsid 22378 [001] 9246431.240052: sched:sched_switch: prev_comm=iscsid prev_pid=22378 prev_prio=110 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [006] 9246431.244088: sched:sched_migrate_task: comm=kworker/u30:0 pid=11450 prio=120 orig_cpu=2 dest_cpu=6
         swapper     0 [006] 9246431.244089: sched:sched_stat_sleep: comm=kworker/u30:0 pid=11450 delay=219979399 [ns]
         swapper     0 [006] 9246431.244089: sched:sched_wakeup: comm=kworker/u30:0 pid=11450 prio=120 target_cpu=006
         swapper     0 [006] 9246431.244097: sched:sched_stat_wait: comm=kworker/u30:0 pid=11450 delay=1759 [ns]
         swapper     0 [006] 9246431.244097: sched:sched_switch: prev_comm=swapper/6 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=kworker/u30:0 next_pid=11450 next_prio=120
   kworker/u30:0 11450 [006] 9246431.244104: sched:sched_stat_runtime: comm=kworker/u30:0 pid=11450 runtime=13363 [ns] vruntime=3671528680731 [ns]
   kworker/u30:0 11450 [006] 9246431.244105: sched:sched_switch: prev_comm=kworker/u30:0 prev_pid=11450 prev_prio=120 prev_state=S ==> next_comm=swapper/6 next_pid=0 next_prio=120
         swapper     0 [001] 9246431.352679: sched:sched_stat_sleep: comm=dockerd pid=3723 delay=499942761 [ns]
         swapper     0 [001] 9246431.352681: sched:sched_wakeup: comm=dockerd pid=3723 prio=120 target_cpu=001
         swapper     0 [001] 9246431.352694: sched:sched_stat_wait: comm=dockerd pid=3723 delay=0 [ns]
         swapper     0 [001] 9246431.352695: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=3723 next_prio=120
         dockerd  3723 [001] 9246431.352703: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=499152378 [ns]
         dockerd  3723 [001] 9246431.352710: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         dockerd  3723 [001] 9246431.352718: sched:sched_migrate_task: comm=dockerd pid=1271 prio=120 orig_cpu=1 dest_cpu=2
         dockerd  3723 [001] 9246431.352720: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=499335939 [ns]
         dockerd  3723 [001] 9246431.352727: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=002
         swapper     0 [000] 9246431.352730: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.352732: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  3723 [001] 9246431.352732: sched:sched_stat_sleep: comm=dockerd pid=1273 delay=499816252 [ns]
         dockerd  3723 [001] 9246431.352739: sched:sched_wakeup: comm=dockerd pid=1273 prio=120 target_cpu=005
         dockerd  1155 [000] 9246431.352743: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=40751 [ns] vruntime=1573789683205 [ns]
         dockerd  1155 [000] 9246431.352745: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  3723 [001] 9246431.352745: sched:sched_stat_runtime: comm=dockerd pid=3723 runtime=66448 [ns] vruntime=1550702140667 [ns]
         dockerd  3723 [001] 9246431.352749: sched:sched_switch: prev_comm=dockerd prev_pid=3723 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [002] 9246431.352749: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [002] 9246431.352751: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         swapper     0 [005] 9246431.352762: sched:sched_stat_wait: comm=dockerd pid=1273 delay=0 [ns]
         swapper     0 [005] 9246431.352764: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1273 next_prio=120
         dockerd  1271 [002] 9246431.352765: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=45355 [ns] vruntime=1595924083424 [ns]
         dockerd  1271 [002] 9246431.352768: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         dockerd  1273 [005] 9246431.352839: sched:sched_stat_sleep: comm=docker-containe pid=31466 delay=499624592 [ns]
         dockerd  1273 [005] 9246431.352849: sched:sched_wakeup: comm=docker-containe pid=31466 prio=120 target_cpu=004
         dockerd  1273 [005] 9246431.352860: sched:sched_stat_runtime: comm=dockerd pid=1273 runtime=127761 [ns] vruntime=1264672466036 [ns]
         dockerd  1273 [005] 9246431.352862: sched:sched_switch: prev_comm=dockerd prev_pid=1273 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [004] 9246431.352868: sched:sched_stat_wait: comm=docker-containe pid=31466 delay=0 [ns]
         swapper     0 [004] 9246431.352870: sched:sched_switch: prev_comm=swapper/4 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=31466 next_prio=120
 docker-containe 31466 [004] 9246431.352882: sched:sched_stat_sleep: comm=docker-containe pid=1276 delay=499680279 [ns]
 docker-containe 31466 [004] 9246431.352890: sched:sched_wakeup: comm=docker-containe pid=1276 prio=120 target_cpu=007
 docker-containe 31466 [004] 9246431.352900: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=499625304 [ns]
         swapper     0 [000] 9246431.352908: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=164582 [ns]
         swapper     0 [000] 9246431.352911: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
 docker-containe 31466 [004] 9246431.352911: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         swapper     0 [007] 9246431.352917: sched:sched_stat_wait: comm=docker-containe pid=1276 delay=0 [ns]
         swapper     0 [007] 9246431.352919: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=1276 next_prio=120
         swapper     0 [002] 9246431.352927: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=161140 [ns]
         swapper     0 [002] 9246431.352929: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=002
 docker-containe  1276 [007] 9246431.352937: sched:sched_stat_runtime: comm=docker-containe pid=1276 runtime=54722 [ns] vruntime=1272541194876 [ns]
         swapper     0 [000] 9246431.352938: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.352940: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
 docker-containe  1276 [007] 9246431.352940: sched:sched_switch: prev_comm=docker-containe prev_pid=1276 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
         swapper     0 [003] 9246431.352940: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [003] 9246431.352942: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
         dockerd  1274 [003] 9246431.352947: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=47665 [ns] vruntime=1576449694189 [ns]
         swapper     0 [002] 9246431.352947: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [002] 9246431.352949: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1274 [003] 9246431.352950: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
         dockerd  1155 [000] 9246431.352956: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=47711 [ns] vruntime=1573789730916 [ns]
         dockerd  1271 [002] 9246431.352957: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=30492 [ns] vruntime=1595924113916 [ns]
         dockerd  1155 [000] 9246431.352959: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  1271 [002] 9246431.352959: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
 docker-containe 31466 [004] 9246431.352980: sched:sched_stat_sleep: comm=docker-containe pid=3247 delay=499792771 [ns]
 docker-containe 31466 [004] 9246431.352988: sched:sched_wakeup: comm=docker-containe pid=3247 prio=120 target_cpu=002
         swapper     0 [002] 9246431.353006: sched:sched_stat_wait: comm=docker-containe pid=3247 delay=0 [ns]
         swapper     0 [002] 9246431.353007: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=3247 next_prio=120
 docker-containe  3247 [002] 9246431.353017: sched:sched_stat_runtime: comm=docker-containe pid=3247 runtime=37036 [ns] vruntime=1595932497502 [ns]
 docker-containe  3247 [002] 9246431.353018: sched:sched_switch: prev_comm=docker-containe prev_pid=3247 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
 docker-containe 31466 [004] 9246431.353092: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=144376 [ns]
         swapper     0 [007] 9246431.353097: sched:sched_stat_sleep: comm=docker-containe pid=1276 delay=159843 [ns]
         swapper     0 [007] 9246431.353100: sched:sched_wakeup: comm=docker-containe pid=1276 prio=120 target_cpu=007
 docker-containe 31466 [004] 9246431.353102: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         swapper     0 [000] 9246431.353123: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=166324 [ns]
         swapper     0 [007] 9246431.353124: sched:sched_stat_wait: comm=docker-containe pid=1276 delay=0 [ns]
         swapper     0 [000] 9246431.353125: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [003] 9246431.353126: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [007] 9246431.353126: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=1276 next_prio=120
         swapper     0 [003] 9246431.353127: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
         dockerd  1274 [003] 9246431.353139: sched:sched_stat_sleep: comm=docker-containe pid=3247 delay=121533 [ns]
 docker-containe  1276 [007] 9246431.353142: sched:sched_stat_runtime: comm=docker-containe pid=1276 runtime=45069 [ns] vruntime=1272541239945 [ns]
 docker-containe  1276 [007] 9246431.353145: sched:sched_switch: prev_comm=docker-containe prev_pid=1276 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246431.353165: sched:sched_wakeup: comm=docker-containe pid=3247 prio=120 target_cpu=002
         swapper     0 [000] 9246431.353167: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.353168: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
 docker-containe 31466 [004] 9246431.353168: sched:sched_stat_runtime: comm=docker-containe pid=31466 runtime=329989 [ns] vruntime=1274754268008 [ns]
 docker-containe 31466 [004] 9246431.353170: sched:sched_switch: prev_comm=docker-containe prev_pid=31466 prev_prio=120 prev_state=S ==> next_comm=swapper/4 next_pid=0 next_prio=120
         swapper     0 [002] 9246431.353177: sched:sched_stat_wait: comm=docker-containe pid=3247 delay=0 [ns]
         dockerd  1155 [000] 9246431.353177: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=55455 [ns] vruntime=1573789786371 [ns]
         swapper     0 [002] 9246431.353177: sched:sched_switch: prev_comm=swapper/2 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=3247 next_prio=120
         dockerd  1274 [003] 9246431.353178: sched:sched_migrate_task: comm=dockerd pid=1271 prio=120 orig_cpu=2 dest_cpu=1
         dockerd  1155 [000] 9246431.353179: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246431.353179: sched:sched_stat_sleep: comm=dockerd pid=1271 delay=222390 [ns]
 docker-containe  3247 [002] 9246431.353180: sched:sched_stat_runtime: comm=docker-containe pid=3247 runtime=42087 [ns] vruntime=1595932539589 [ns]
 docker-containe  3247 [002] 9246431.353181: sched:sched_switch: prev_comm=docker-containe prev_pid=3247 prev_prio=120 prev_state=S ==> next_comm=swapper/2 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246431.353183: sched:sched_wakeup: comm=dockerd pid=1271 prio=120 target_cpu=001
         swapper     0 [001] 9246431.353194: sched:sched_stat_wait: comm=dockerd pid=1271 delay=0 [ns]
         swapper     0 [001] 9246431.353194: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1271 next_prio=120
         dockerd  1271 [001] 9246431.353200: sched:sched_stat_runtime: comm=dockerd pid=1271 runtime=20989 [ns] vruntime=1550693778070 [ns]
         dockerd  1271 [001] 9246431.353201: sched:sched_switch: prev_comm=dockerd prev_pid=1271 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         dockerd  1274 [003] 9246431.353206: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=115295 [ns] vruntime=1576449809484 [ns]
         dockerd  1274 [003] 9246431.353208: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
         swapper     0 [007] 9246431.353294: sched:sched_stat_sleep: comm=docker-containe pid=1276 delay=152094 [ns]
         swapper     0 [007] 9246431.353294: sched:sched_wakeup: comm=docker-containe pid=1276 prio=120 target_cpu=007
         swapper     0 [007] 9246431.353302: sched:sched_stat_wait: comm=docker-containe pid=1276 delay=0 [ns]
         swapper     0 [007] 9246431.353303: sched:sched_switch: prev_comm=swapper/7 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=docker-containe next_pid=1276 next_prio=120
 docker-containe  1276 [007] 9246431.353307: sched:sched_stat_runtime: comm=docker-containe pid=1276 runtime=13662 [ns] vruntime=1272541253607 [ns]
 docker-containe  1276 [007] 9246431.353308: sched:sched_switch: prev_comm=docker-containe prev_pid=1276 prev_prio=120 prev_state=S ==> next_comm=swapper/7 next_pid=0 next_prio=120
         swapper     0 [000] 9246431.353331: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=154231 [ns]
         swapper     0 [000] 9246431.353332: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [000] 9246431.353340: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.353340: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  1155 [000] 9246431.353344: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=12989 [ns] vruntime=1573789799360 [ns]
         dockerd  1155 [000] 9246431.353345: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [001] 9246431.444984: sched:sched_stat_sleep: comm=dockerd pid=3723 delay=92239257 [ns]
         swapper     0 [001] 9246431.444985: sched:sched_wakeup: comm=dockerd pid=3723 prio=120 target_cpu=001
         swapper     0 [001] 9246431.444991: sched:sched_stat_wait: comm=dockerd pid=3723 delay=0 [ns]
         swapper     0 [001] 9246431.444992: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=3723 next_prio=120
         dockerd  3723 [001] 9246431.444996: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=91651476 [ns]
         dockerd  3723 [001] 9246431.445000: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         dockerd  3723 [001] 9246431.445004: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=91797536 [ns]
         dockerd  3723 [001] 9246431.445008: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         swapper     0 [000] 9246431.445010: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         dockerd  3723 [001] 9246431.445010: sched:sched_stat_sleep: comm=dockerd pid=1273 delay=92150501 [ns]
         swapper     0 [000] 9246431.445010: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  3723 [001] 9246431.445014: sched:sched_wakeup: comm=dockerd pid=1273 prio=120 target_cpu=005
         dockerd  1155 [000] 9246431.445016: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=20515 [ns] vruntime=1573789819875 [ns]
         dockerd  3723 [001] 9246431.445016: sched:sched_stat_runtime: comm=dockerd pid=3723 runtime=32375 [ns] vruntime=1550702173042 [ns]
         dockerd  1155 [000] 9246431.445017: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         dockerd  3723 [001] 9246431.445018: sched:sched_switch: prev_comm=dockerd prev_pid=3723 prev_prio=120 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [003] 9246431.445019: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [003] 9246431.445020: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
         swapper     0 [005] 9246431.445025: sched:sched_stat_wait: comm=dockerd pid=1273 delay=0 [ns]
         swapper     0 [005] 9246431.445025: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1273 next_prio=120
         dockerd  1274 [003] 9246431.445027: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=22747 [ns] vruntime=1576449832231 [ns]
         dockerd  1274 [003] 9246431.445028: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
         dockerd  1273 [005] 9246431.445031: sched:sched_stat_runtime: comm=dockerd pid=1273 runtime=20557 [ns] vruntime=1264672486593 [ns]
         dockerd  1273 [005] 9246431.445032: sched:sched_switch: prev_comm=dockerd prev_pid=1273 prev_prio=120 prev_state=S ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [000] 9246431.445171: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=154648 [ns]
         swapper     0 [000] 9246431.445172: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [000] 9246431.445179: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.445179: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         swapper     0 [003] 9246431.445183: sched:sched_stat_sleep: comm=dockerd pid=1274 delay=156217 [ns]
         swapper     0 [003] 9246431.445184: sched:sched_wakeup: comm=dockerd pid=1274 prio=120 target_cpu=003
         dockerd  1155 [000] 9246431.445185: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=14927 [ns] vruntime=1573789834802 [ns]
         dockerd  1155 [000] 9246431.445186: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [003] 9246431.445193: sched:sched_stat_wait: comm=dockerd pid=1274 delay=0 [ns]
         swapper     0 [003] 9246431.445194: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1274 next_prio=120
         dockerd  1274 [003] 9246431.445197: sched:sched_stat_runtime: comm=dockerd pid=1274 runtime=14594 [ns] vruntime=1576449846825 [ns]
         dockerd  1274 [003] 9246431.445198: sched:sched_switch: prev_comm=dockerd prev_pid=1274 prev_prio=120 prev_state=S ==> next_comm=swapper/3 next_pid=0 next_prio=120
         swapper     0 [000] 9246431.445340: sched:sched_stat_sleep: comm=dockerd pid=1155 delay=154711 [ns]
         swapper     0 [000] 9246431.445341: sched:sched_wakeup: comm=dockerd pid=1155 prio=120 target_cpu=000
         swapper     0 [000] 9246431.445348: sched:sched_stat_wait: comm=dockerd pid=1155 delay=0 [ns]
         swapper     0 [000] 9246431.445349: sched:sched_switch: prev_comm=swapper/0 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=dockerd next_pid=1155 next_prio=120
         dockerd  1155 [000] 9246431.445352: sched:sched_stat_runtime: comm=dockerd pid=1155 runtime=11511 [ns] vruntime=1573789846313 [ns]
         dockerd  1155 [000] 9246431.445352: sched:sched_switch: prev_comm=dockerd prev_pid=1155 prev_prio=120 prev_state=S ==> next_comm=swapper/0 next_pid=0 next_prio=120
         swapper     0 [001] 9246431.490368: sched:sched_stat_sleep: comm=iscsid pid=22378 delay=250316051 [ns]
         swapper     0 [001] 9246431.490370: sched:sched_wakeup: comm=iscsid pid=22378 prio=110 target_cpu=001
         swapper     0 [001] 9246431.490386: sched:sched_stat_wait: comm=iscsid pid=22378 delay=0 [ns]
         swapper     0 [001] 9246431.490387: sched:sched_switch: prev_comm=swapper/1 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=iscsid next_pid=22378 next_prio=110
          iscsid 22378 [001] 9246431.490395: sched:sched_stat_runtime: comm=iscsid pid=22378 runtime=27306 [ns] vruntime=368722984 [ns]
          iscsid 22378 [001] 9246431.490397: sched:sched_switch: prev_comm=iscsid prev_pid=22378 prev_prio=110 prev_state=S ==> next_comm=swapper/1 next_pid=0 next_prio=120
         swapper     0 [005] 9246431.573208: sched:sched_stat_sleep: comm=sleep pid=11530 delay=1000106702 [ns]
         swapper     0 [005] 9246431.573209: sched:sched_wakeup: comm=sleep pid=11530 prio=120 target_cpu=005
         swapper     0 [005] 9246431.573217: sched:sched_stat_wait: comm=sleep pid=11530 delay=0 [ns]
         swapper     0 [005] 9246431.573218: sched:sched_switch: prev_comm=swapper/5 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=sleep next_pid=11530 next_prio=120
           sleep 11530 [005] 9246431.573300: sched:sched_stat_sleep: comm=perf pid=11527 delay=1000771307 [ns]
           sleep 11530 [005] 9246431.573305: sched:sched_wakeup: comm=perf pid=11527 prio=120 target_cpu=003
           sleep 11530 [005] 9246431.573306: sched:sched_stat_runtime: comm=sleep pid=11530 runtime=99654 [ns] vruntime=589868031741 [ns]
           sleep 11530 [005] 9246431.573307: sched:sched_switch: prev_comm=sleep prev_pid=11530 prev_prio=120 prev_state=x ==> next_comm=swapper/5 next_pid=0 next_prio=120
         swapper     0 [003] 9246431.573315: sched:sched_stat_wait: comm=perf pid=11527 delay=0 [ns]
         swapper     0 [003] 9246431.573316: sched:sched_switch: prev_comm=swapper/3 prev_pid=0 prev_prio=120 prev_state=R ==> next_comm=perf next_pid=11527 next_prio=120
root@backend-shared-stag-0:~/workspace#
```

`perf sched map` shows **all CPUs and context-switch events, with columns representing what each CPU was doing and when**. It's the kind of data you see visualized in **scheduler analysis GUIs** (including `perf timechart`, with the layout rotated 90 degrees).

Example output:

```
# perf sched map
                      *A0           991962.879971 secs A0 => perf:16999
                       A0     *B0   991962.880070 secs B0 => cc1:16863
          *C0          A0      B0   991962.880070 secs C0 => :17023:17023
  *D0      C0          A0      B0   991962.880078 secs D0 => ksoftirqd/0:6
   D0      C0 *E0      A0      B0   991962.880081 secs E0 => ksoftirqd/3:28
   D0      C0 *F0      A0      B0   991962.880093 secs F0 => :17022:17022
  *G0      C0  F0      A0      B0   991962.880108 secs G0 => :17016:17016
   G0      C0  F0     *H0      B0   991962.880256 secs H0 => migration/5:39
   G0      C0  F0     *I0      B0   991962.880276 secs I0 => perf:16984
   G0      C0  F0     *J0      B0   991962.880687 secs J0 => cc1:16996
   G0      C0 *K0      J0      B0   991962.881839 secs K0 => cc1:16945
   G0      C0  K0      J0 *L0  B0   991962.881841 secs L0 => :17020:17020
   G0      C0  K0      J0 *M0  B0   991962.882289 secs M0 => make:16637
   G0      C0  K0      J0 *N0  B0   991962.883102 secs N0 => make:16545
   G0     *O0  K0      J0  N0  B0   991962.883880 secs O0 => cc1:16819
   G0 *A0  O0  K0      J0  N0  B0   991962.884069 secs 
   G0  A0  O0  K0 *P0  J0  N0  B0   991962.884076 secs P0 => rcu_sched:7
   G0  A0  O0  K0 *Q0  J0  N0  B0   991962.884084 secs Q0 => cc1:16831
   G0  A0  O0  K0  Q0  J0 *R0  B0   991962.884843 secs R0 => cc1:16825
   G0 *S0  O0  K0  Q0  J0  R0  B0   991962.885636 secs S0 => cc1:16900
   G0  S0  O0 *T0  Q0  J0  R0  B0   991962.886893 secs T0 => :17014:17014
   G0  S0  O0 *K0  Q0  J0  R0  B0   991962.886917 secs 
[...]
```

This is an 8 CPU system, and you can see the **8 columns for each CPU starting from the left**. Some CPU columns begin blank, as we've yet to trace an event on that CPU at the start of the profile. They quickly become populated.

The **two character codes** you see ("A0", "C0") **are identifiers for tasks**, which are **mapped on the right ("=>")**. This is more compact than using process (task) IDs. **The "*" shows which CPU had the context switch event, and the new event that was running**. For example, the very last line shows that at 991962.886917 (seconds) CPU 4 context-switched to K0 (a "cc1" process, PID 16945).

**Idle CPUs are shown as "."**.

Remember to **examine the timestamp column to make sense of this visualization** (GUIs use that as a dimension, which is easier to comprehend, but here the numbers are just listed). It's also only showing context switch events, and not scheduler latency. The newer `timehist` command has a visualization (`-V`) that can include wakeup events.


我的实际输出

```
root@backend-shared-stag-0:~/workspace# perf sched map
      *A0                               9246430.572070 secs A0 => rcu_sched:7
      *.                                9246430.572075 secs .  => swapper:0
       .          *B0                   9246430.572482 secs B0 => perf:11530
       .      *.   B0                   9246430.572531 secs
       .       .  *C0                   9246430.572571 secs C0 => migration/4:27
       .       .  *.                    9246430.572584 secs
       .       .   .  *B0               9246430.572592 secs
       .       .   .  *.                9246430.573103 secs
      *A0      .   .   .                9246430.576081 secs
      *.       .   .   .                9246430.576085 secs
      *A0      .   .   .                9246430.580074 secs
      *.       .   .   .                9246430.580077 secs
      *A0      .   .   .                9246430.584072 secs
      *.       .   .   .                9246430.584075 secs
      *A0      .   .   .                9246430.588072 secs
      *.       .   .   .                9246430.588075 secs
      *A0      .   .   .                9246430.592068 secs
      *.       .   .   .                9246430.592070 secs
      *A0      .   .   .                9246430.596068 secs
      *.       .   .   .                9246430.596069 secs
      *A0      .   .   .                9246430.600066 secs
      *.       .   .   .                9246430.600067 secs
       .  *D0  .   .   .                9246430.711908 secs D0 => ntpd:22235
       .  *.   .   .   .                9246430.711932 secs
      *E0  .   .   .   .                9246430.739389 secs E0 => iscsid:22378
      *.   .   .   .   .                9246430.739399 secs
       .   .   .   .   .  *F0           9246430.740079 secs F0 => kworker/6:1:4854
       .   .   .   .   .  *.            9246430.740084 secs
       .   .   .  *G0  .   .            9246430.764070 secs G0 => kworker/4:2:10988
       .   .  *H0  G0  .   .            9246430.764088 secs H0 => kworker/3:1:11369
       .   .   H0 *.   .   .            9246430.764092 secs
       .   .   H0  .  *I0  .            9246430.764092 secs I0 => kworker/5:0:15448
       .   .  *.   .   I0  .            9246430.764095 secs
       .   .   .   .  *.   .            9246430.764097 secs
       .   .   .   .   .   .  *J0       9246430.764103 secs J0 => kworker/7:2:6649
       .   .   .   .   .   .  *.        9246430.764109 secs
       .   .   .   .  *K0  .   .        9246430.776162 secs K0 => haproxy:8083
       .   .   .   .  *.   .   .        9246430.776173 secs
      *L0  .   .   .   .   .   .        9246430.800071 secs L0 => kworker/1:1:7517
      *.   .   .   .   .   .   .        9246430.800075 secs
  *M0  .   .   .   .   .   .   .        9246430.812075 secs M0 => kworker/0:0:8384
  *.   .   .   .   .   .   .   .        9246430.812079 secs
   .  *N0  .   .   .   .   .   .        9246430.852676 secs N0 => dockerd:3723
  *O0  N0  .   .   .   .   .   .        9246430.852719 secs O0 => dockerd:1155
   O0 *.   .   .   .   .   .   .        9246430.852740 secs
   O0  .  *P0  .   .   .   .   .        9246430.852742 secs P0 => dockerd:1271
  *.   .   P0  .   .   .   .   .        9246430.852745 secs
   .   .   P0  .   .  *Q0  .   .        9246430.852762 secs Q0 => dockerd:1273
   .   .  *.   .   .   Q0  .   .        9246430.852763 secs
   .   .   .   .   .  *.   .   .        9246430.852920 secs
  *O0  .   .   .   .   .   .   .        9246430.852922 secs
  *.   .   .   .   .   .   .   .        9246430.852936 secs
   .   .  *R0  .   .   .   .   .        9246430.852946 secs R0 => docker-containe:3247
   .   .   R0 *P0  .   .   .   .        9246430.852949 secs
   .   .   R0 *.   .   .   .   .        9246430.852962 secs
   .   .   R0  .   .   .   .  *S0       9246430.853003 secs S0 => docker-containe:1276
   .   .   R0 *T0  .   .   .   S0       9246430.853019 secs T0 => dockerd:1274
   .   .   R0 *.   .   .   .   S0       9246430.853031 secs
   .   .   R0  .   .   .   .  *.        9246430.853032 secs
   .   .   R0  .  *U0  .   .   .        9246430.853097 secs U0 => docker-containe:31466
  *O0  .   R0  .   U0  .   .   .        9246430.853099 secs
  *.   .   R0  .   U0  .   .   .        9246430.853104 secs
   .   .   R0  .  *.   .   .   .        9246430.853110 secs
   .   .   R0 *T0  .   .   .   .        9246430.853173 secs
   .   .  *.   T0  .   .   .   .        9246430.853189 secs
   .   .   .   T0  .   .   .  *S0       9246430.853197 secs
  *O0  .   .   T0  .   .   .   S0       9246430.853201 secs
   O0  .   .   T0  .   .   .  *.        9246430.853203 secs
  *.   .   .   T0  .   .   .   .        9246430.853210 secs
   .   .   .   T0 *U0  .   .   .        9246430.853211 secs
   .   .   .   T0 *.   .   .   .        9246430.853215 secs
  *P0  .   .   T0  .   .   .   .        9246430.853237 secs
  *.   .   .   T0  .   .   .   .        9246430.853244 secs
   .   .   .  *.   .   .   .   .        9246430.853276 secs
  *O0  .   .   .   .   .   .   .        9246430.853378 secs
   O0 *P0  .   .   .   .   .   .        9246430.853380 secs
  *.   P0  .   .   .   .   .   .        9246430.853385 secs
   .  *.   .   .   .   .   .   .        9246430.853385 secs
  *O0  .   .   .   .   .   .   .        9246430.853547 secs
  *.   .   .   .   .   .   .   .        9246430.853551 secs
   .  *E0  .   .   .   .   .   .        9246430.989729 secs
   .  *.   .   .   .   .   .   .        9246430.989739 secs
   .   .  *V0  .   .   .   .   .        9246431.024106 secs V0 => kworker/u30:0:11450
   .   .  *.   .   .   .   .   .        9246431.024112 secs
   .  *W0  .   .   .   .   .   .        9246431.070950 secs W0 => iscsid:22377
   .  *.   .   .   .   .   .   .        9246431.070959 secs
   .  *E0  .   .   .   .   .   .        9246431.240048 secs
   .  *.   .   .   .   .   .   .        9246431.240052 secs
   .   .   .   .   .   .  *V0  .        9246431.244098 secs
   .   .   .   .   .   .  *.   .        9246431.244105 secs
   .  *N0  .   .   .   .   .   .        9246431.352695 secs
  *O0  N0  .   .   .   .   .   .        9246431.352732 secs
  *.   N0  .   .   .   .   .   .        9246431.352745 secs
   .  *.   .   .   .   .   .   .        9246431.352749 secs
   .   .  *P0  .   .   .   .   .        9246431.352751 secs
   .   .   P0  .   .  *Q0  .   .        9246431.352764 secs
   .   .  *.   .   .   Q0  .   .        9246431.352769 secs
   .   .   .   .   .  *.   .   .        9246431.352863 secs
   .   .   .   .  *U0  .   .   .        9246431.352870 secs
   .   .   .   .   U0  .   .  *S0       9246431.352919 secs
  *O0  .   .   .   U0  .   .   S0       9246431.352940 secs
   O0  .   .   .   U0  .   .  *.        9246431.352940 secs
   O0  .   .  *T0  U0  .   .   .        9246431.352942 secs
   O0  .  *P0  T0  U0  .   .   .        9246431.352949 secs
   O0  .   P0 *.   U0  .   .   .        9246431.352951 secs
  *.   .   P0  .   U0  .   .   .        9246431.352959 secs
   .   .  *.   .   U0  .   .   .        9246431.352959 secs
   .   .  *R0  .   U0  .   .   .        9246431.353007 secs
   .   .  *.   .   U0  .   .   .        9246431.353019 secs
   .   .   .   .   U0  .   .  *S0       9246431.353126 secs
   .   .   .  *T0  U0  .   .   S0       9246431.353128 secs
   .   .   .   T0  U0  .   .  *.        9246431.353145 secs
  *O0  .   .   T0  U0  .   .   .        9246431.353168 secs
   O0  .   .   T0 *.   .   .   .        9246431.353171 secs
   O0  .  *R0  T0  .   .   .   .        9246431.353178 secs
  *.   .   R0  T0  .   .   .   .        9246431.353179 secs
   .   .  *.   T0  .   .   .   .        9246431.353181 secs
   .  *P0  .   T0  .   .   .   .        9246431.353195 secs
   .  *.   .   T0  .   .   .   .        9246431.353202 secs
   .   .   .  *.   .   .   .   .        9246431.353208 secs
   .   .   .   .   .   .   .  *S0       9246431.353304 secs
   .   .   .   .   .   .   .  *.        9246431.353308 secs
  *O0  .   .   .   .   .   .   .        9246431.353341 secs
  *.   .   .   .   .   .   .   .        9246431.353345 secs
   .  *N0  .   .   .   .   .   .        9246431.444992 secs
  *O0  N0  .   .   .   .   .   .        9246431.445011 secs
  *.   N0  .   .   .   .   .   .        9246431.445017 secs
   .  *.   .   .   .   .   .   .        9246431.445019 secs
   .   .   .  *T0  .   .   .   .        9246431.445021 secs
   .   .   .   T0  .  *Q0  .   .        9246431.445026 secs
   .   .   .  *.   .   Q0  .   .        9246431.445028 secs
   .   .   .   .   .  *.   .   .        9246431.445032 secs
  *O0  .   .   .   .   .   .   .        9246431.445180 secs
  *.   .   .   .   .   .   .   .        9246431.445187 secs
   .   .   .  *T0  .   .   .   .        9246431.445194 secs
   .   .   .  *.   .   .   .   .        9246431.445199 secs
  *O0  .   .   .   .   .   .   .        9246431.445349 secs
  *.   .   .   .   .   .   .   .        9246431.445353 secs
   .  *E0  .   .   .   .   .   .        9246431.490388 secs
   .  *.   .   .   .   .   .   .        9246431.490397 secs
   .   .   .   .   .  *B0  .   .        9246431.573218 secs
   .   .   .   .   .  *.   .   .        9246431.573308 secs
   .   .   .  *X0  .   .   .   .        9246431.573316 secs X0 => perf:11527
root@backend-shared-stag-0:~/workspace#
```

可以基于 perf.data 生成相应的 SVG ；

```
root@backend-shared-stag-0:~/workspace# perf timechart
Written 1.0 seconds of trace to output.svg.
root@backend-shared-stag-0:~/workspace#
```


`perf sched timehist` was added in Linux **4.10**, and **shows the scheduler latency by event**, including the time the task was waiting to be woken up (`wait time`) and the scheduler latency after wakeup to running (`sch delay`). **It's the scheduler latency that we're more interested in tuning**. 

（略）

`perf sched script` dumps all events (similar to `perf script`):

```
# perf sched script

    perf 16984 [005] 991962.879960: sched:sched_stat_runtime: comm=perf pid=16984 runtime=3901506 [ns] vruntime=165...
    perf 16984 [005] 991962.879966:       sched:sched_wakeup: comm=perf pid=16999 prio=120 target_cpu=005
    perf 16984 [005] 991962.879971:       sched:sched_switch: prev_comm=perf prev_pid=16984 prev_prio=120 prev_stat...
    perf 16999 [005] 991962.880058: sched:sched_stat_runtime: comm=perf pid=16999 runtime=98309 [ns] vruntime=16405...
     cc1 16881 [000] 991962.880058: sched:sched_stat_runtime: comm=cc1 pid=16881 runtime=3999231 [ns] vruntime=7897...
  :17024 17024 [004] 991962.880058: sched:sched_stat_runtime: comm=cc1 pid=17024 runtime=3866637 [ns] vruntime=7810...
     cc1 16900 [001] 991962.880058: sched:sched_stat_runtime: comm=cc1 pid=16900 runtime=3006028 [ns] vruntime=7772...
     cc1 16825 [006] 991962.880058: sched:sched_stat_runtime: comm=cc1 pid=16825 runtime=3999423 [ns] vruntime=7876...
```

Each of these events ("`sched:sched_stat_runtime`" etc) are **tracepoints** you can instrument directly using `perf record`.

As I've shown earlier, **this raw output can be useful for digging further than the summary commands**.

`perf sched replay` will **take the recorded scheduler events, and then simulate the workload by spawning threads with similar runtimes and context switches**. Useful for testing and developing scheduler changes and configuration. Don't put too much faith in this (and other) workload replayers: **they can be a useful load generator, but it's difficult to simulate the real workload completely**. Here I'm running replay with `-r -1`, to repeat the workload:

```
root@backend-shared-stag-0:~/workspace# perf sched replay -r -1
run measurement overhead: 324 nsecs
sleep measurement overhead: 150406 nsecs
the run test took 999976 nsecs
the sleep test took 1124745 nsecs
nr_run_events:        143
nr_sleep_events:      394
nr_wakeup_events:     71
task      0 (             swapper:         0), nr_events: 184
task      1 (             swapper:         1), nr_events: 1
task      2 (             swapper:         2), nr_events: 1
task      3 (            kthreadd:         3), nr_events: 1
task      4 (            kthreadd:         5), nr_events: 1
task      5 (            kthreadd:         7), nr_events: 17
task      6 (            kthreadd:         8), nr_events: 1
task      7 (            kthreadd:         9), nr_events: 1
task      8 (            kthreadd:        10), nr_events: 1
task      9 (            kthreadd:        11), nr_events: 1
task     10 (            kthreadd:        12), nr_events: 1
task     11 (            kthreadd:        13), nr_events: 1
task     12 (            kthreadd:        15), nr_events: 1
task     13 (            kthreadd:        16), nr_events: 1
task     14 (            kthreadd:        17), nr_events: 1
task     15 (            kthreadd:        18), nr_events: 1
task     16 (            kthreadd:        20), nr_events: 1
task     17 (            kthreadd:        21), nr_events: 1
task     18 (            kthreadd:        22), nr_events: 1
task     19 (            kthreadd:        23), nr_events: 1
task     20 (            kthreadd:        25), nr_events: 1
task     21 (            kthreadd:        26), nr_events: 1
task     22 (            kthreadd:        27), nr_events: 3
task     23 (            kthreadd:        28), nr_events: 1
task     24 (            kthreadd:        30), nr_events: 1
task     25 (            kthreadd:        31), nr_events: 1
task     26 (            kthreadd:        32), nr_events: 1
task     27 (            kthreadd:        33), nr_events: 1
task     28 (            kthreadd:        35), nr_events: 1
task     29 (            kthreadd:        36), nr_events: 1
task     30 (            kthreadd:        37), nr_events: 1
task     31 (            kthreadd:        38), nr_events: 1
task     32 (            kthreadd:        40), nr_events: 1
task     33 (            kthreadd:        41), nr_events: 1
task     34 (            kthreadd:        42), nr_events: 1
task     35 (            kthreadd:        43), nr_events: 1
task     36 (            kthreadd:        45), nr_events: 1
task     37 (            kthreadd:        46), nr_events: 1
task     38 (            kthreadd:        47), nr_events: 1
task     39 (            kthreadd:        48), nr_events: 1
task     40 (            kthreadd:        49), nr_events: 1
task     41 (            kthreadd:        50), nr_events: 1
task     42 (            kthreadd:        52), nr_events: 1
task     43 (            kthreadd:        53), nr_events: 1
task     44 (            kthreadd:        54), nr_events: 1
task     45 (            kthreadd:        55), nr_events: 1
task     46 (            kthreadd:        56), nr_events: 1
task     47 (            kthreadd:        57), nr_events: 1
task     48 (            kthreadd:        58), nr_events: 1
task     49 (            kthreadd:        59), nr_events: 1
task     50 (            kthreadd:        61), nr_events: 1
task     51 (            kthreadd:        62), nr_events: 1
task     52 (            kthreadd:        63), nr_events: 1
task     53 (            kthreadd:        72), nr_events: 1
task     54 (            kthreadd:        73), nr_events: 1
task     55 (            kthreadd:        74), nr_events: 1
task     56 (            kthreadd:        75), nr_events: 1
task     57 (            kthreadd:        91), nr_events: 1
task     58 (            kthreadd:        92), nr_events: 1
task     59 (            kthreadd:        93), nr_events: 1
task     60 (            kthreadd:        94), nr_events: 1
task     61 (            kthreadd:        95), nr_events: 1
task     62 (            kthreadd:        96), nr_events: 1
task     63 (            kthreadd:        97), nr_events: 1
task     64 (            kthreadd:        98), nr_events: 1
task     65 (            kthreadd:        99), nr_events: 1
task     66 (            kthreadd:       100), nr_events: 1
task     67 (            kthreadd:       101), nr_events: 1
task     68 (            kthreadd:       102), nr_events: 1
task     69 (            kthreadd:       103), nr_events: 1
task     70 (            kthreadd:       104), nr_events: 1
task     71 (            kthreadd:       105), nr_events: 1
task     72 (            kthreadd:       110), nr_events: 1
task     73 (            kthreadd:       123), nr_events: 1
task     74 (            kthreadd:       124), nr_events: 1
task     75 (            kthreadd:       160), nr_events: 1
task     76 (            kthreadd:       161), nr_events: 1
task     77 (            kthreadd:       162), nr_events: 1
task     78 (            kthreadd:       163), nr_events: 1
task     79 (            kthreadd:       164), nr_events: 1
task     80 (            kthreadd:       165), nr_events: 1
task     81 (            kthreadd:       166), nr_events: 1
task     82 (            kthreadd:       168), nr_events: 1
task     83 (            kthreadd:       184), nr_events: 1
task     84 (            kthreadd:       191), nr_events: 1
task     85 (            kthreadd:       213), nr_events: 1
task     86 (            kthreadd:       278), nr_events: 1
task     87 (            kthreadd:       305), nr_events: 1
task     88 (            kthreadd:       332), nr_events: 1
task     89 (            kthreadd:       333), nr_events: 1
task     90 (            kthreadd:       384), nr_events: 1
task     91 (            kthreadd:       398), nr_events: 1
task     92 (            kthreadd:       400), nr_events: 1
task     93 (            kthreadd:       410), nr_events: 1
task     94 (            kthreadd:       416), nr_events: 1
task     95 (            kthreadd:       417), nr_events: 1
task     96 (            kthreadd:       418), nr_events: 1
task     97 (            kthreadd:       419), nr_events: 1
task     98 (            kthreadd:       420), nr_events: 1
task     99 (             systemd:       421), nr_events: 1
task    100 (            kthreadd:       422), nr_events: 1
task    101 (            kthreadd:       430), nr_events: 1
task    102 (             systemd:       441), nr_events: 1
task    103 (            kthreadd:       621), nr_events: 1
task    104 (            kthreadd:       670), nr_events: 1
task    105 (             systemd:       907), nr_events: 1
task    106 (            kthreadd:      1034), nr_events: 1
task    107 (             systemd:      1072), nr_events: 1
task    108 (             dockerd:      1155), nr_events: 27
task    109 (             dockerd:      1157), nr_events: 1
task    110 (             dockerd:      1158), nr_events: 1
task    111 (             dockerd:      1159), nr_events: 1
task    112 (             dockerd:      1268), nr_events: 1
task    113 (             dockerd:      1269), nr_events: 1
task    114 (             dockerd:      1270), nr_events: 1
task    115 (             dockerd:      1271), nr_events: 15
task    116 (             dockerd:      1272), nr_events: 1
task    117 (             dockerd:      1273), nr_events: 9
task    118 (             dockerd:      1274), nr_events: 18
task    119 (             dockerd:      1430), nr_events: 1
task    120 (             dockerd:      1431), nr_events: 1
task    121 (             dockerd:      3723), nr_events: 16
task    122 (             dockerd:      9781), nr_events: 1
task    123 (             systemd:      1074), nr_events: 1
task    124 (             systemd:      1077), nr_events: 1
task    125 (             systemd:      1093), nr_events: 1
task    126 (     accounts-daemon:      1128), nr_events: 1
task    127 (     accounts-daemon:      1139), nr_events: 1
task    128 (             systemd:      1097), nr_events: 1
task    129 (             systemd:      1107), nr_events: 1
task    130 (               lxcfs:      1131), nr_events: 1
task    131 (               lxcfs:      1132), nr_events: 1
task    132 (               lxcfs:      8855), nr_events: 1
task    133 (               lxcfs:     27314), nr_events: 1
task    134 (               lxcfs:     10734), nr_events: 1
task    135 (             systemd:      1109), nr_events: 1
task    136 (       node_exporter:      1141), nr_events: 1
task    137 (       node_exporter:      1142), nr_events: 1
task    138 (       node_exporter:      1143), nr_events: 1
task    139 (       node_exporter:      1203), nr_events: 1
task    140 (       node_exporter:      1829), nr_events: 1
task    141 (       node_exporter:      1830), nr_events: 1
task    142 (       node_exporter:      1831), nr_events: 1
task    143 (       node_exporter:      1832), nr_events: 1
task    144 (       node_exporter:      1833), nr_events: 1
task    145 (       node_exporter:      3050), nr_events: 1
task    146 (       node_exporter:      3051), nr_events: 1
task    147 (       node_exporter:      7890), nr_events: 1
task    148 (       node_exporter:     14024), nr_events: 1
task    149 (       node_exporter:     14025), nr_events: 1
task    150 (             systemd:      1111), nr_events: 1
task    151 (             systemd:      1113), nr_events: 1
task    152 (            rsyslogd:      1133), nr_events: 1
task    153 (            rsyslogd:      1134), nr_events: 1
task    154 (            rsyslogd:      1136), nr_events: 1
task    155 (             systemd:      1125), nr_events: 1
task    156 (             systemd:      1190), nr_events: 1
task    157 (             polkitd:      1207), nr_events: 1
task    158 (             polkitd:      1209), nr_events: 1
task    159 (            kthreadd:      1201), nr_events: 1
task    160 (             systemd:      1248), nr_events: 1
task    161 (             systemd:      1263), nr_events: 1
task    162 (             systemd:      1264), nr_events: 1
task    163 (             dockerd:      1275), nr_events: 1
task    164 (     docker-containe:      1276), nr_events: 11
task    165 (     docker-containe:      1278), nr_events: 1
task    166 (     docker-containe:      1279), nr_events: 1
task    167 (     docker-containe:      1280), nr_events: 1
task    168 (     docker-containe:      1281), nr_events: 1
task    169 (     docker-containe:      1282), nr_events: 1
task    170 (     docker-containe:      1283), nr_events: 1
task    171 (     docker-containe:      3245), nr_events: 1
task    172 (     docker-containe:      3246), nr_events: 1
task    173 (     docker-containe:      3247), nr_events: 11
task    174 (     docker-containe:      3248), nr_events: 1
task    175 (     docker-containe:      3362), nr_events: 1
task    176 (     docker-containe:      3363), nr_events: 1
task    177 (     docker-containe:      4837), nr_events: 1
task    178 (     docker-containe:     31466), nr_events: 11
task    179 (            kthreadd:      3244), nr_events: 1
task    180 (            kthreadd:      4844), nr_events: 1
task    181 (            kthreadd:      4849), nr_events: 1
task    182 (            kthreadd:      4853), nr_events: 1
task    183 (            kthreadd:      4854), nr_events: 3
task    184 (            kthreadd:      6649), nr_events: 3
task    185 (             systemd:      6986), nr_events: 1
task    186 (             systemd:      6988), nr_events: 1
task    187 (             systemd:      6990), nr_events: 1
task    188 (                sshd:      7063), nr_events: 1
task    189 (                sshd:      7066), nr_events: 1
task    190 (                bash:      7085), nr_events: 1
task    191 (                sudo:      7086), nr_events: 1
task    192 (            kthreadd:      7517), nr_events: 3
task    193 (             systemd:      8080), nr_events: 1
task    194 (     haproxy-systemd:      8081), nr_events: 1
task    195 (             haproxy:      8083), nr_events: 3
task    196 (            kthreadd:      8384), nr_events: 3
task    197 (            kthreadd:      9340), nr_events: 1
task    198 (             systemd:     10792), nr_events: 1
task    199 (            kthreadd:     10988), nr_events: 6
task    200 (            kthreadd:     11123), nr_events: 1
task    201 (            kthreadd:     11369), nr_events: 3
task    202 (                sshd:     11370), nr_events: 1
task    203 (                sshd:     11447), nr_events: 1
task    204 (            kthreadd:     11450), nr_events: 5
task    205 (                sshd:     11454), nr_events: 1
task    206 (                bash:     11468), nr_events: 1
task    207 (                sudo:     11469), nr_events: 1
task    208 (                bash:     11527), nr_events: 4
task    209 (                perf:     11530), nr_events: 9
task    210 (            kthreadd:     13994), nr_events: 1
task    211 (            kthreadd:     15432), nr_events: 1
task    212 (            kthreadd:     15448), nr_events: 3
task    213 (            kthreadd:     15569), nr_events: 1
task    214 (            kthreadd:     19725), nr_events: 1
task    215 (            kthreadd:     19752), nr_events: 1
task    216 (            kthreadd:     19757), nr_events: 1
task    217 (            kthreadd:     20361), nr_events: 1
task    218 (            kthreadd:     20364), nr_events: 1
task    219 (            kthreadd:     20372), nr_events: 1
task    220 (            kthreadd:     20373), nr_events: 1
task    221 (            kthreadd:     20374), nr_events: 1
task    222 (            kthreadd:     20375), nr_events: 1
task    223 (            kthreadd:     20376), nr_events: 1
task    224 (            kthreadd:     20377), nr_events: 1
task    225 (            kthreadd:     20378), nr_events: 1
task    226 (            kthreadd:     20379), nr_events: 1
task    227 (            kthreadd:     20380), nr_events: 1
task    228 (            kthreadd:     20381), nr_events: 1
task    229 (             systemd:     20562), nr_events: 1
task    230 (            kthreadd:     20564), nr_events: 1
task    231 (             systemd:     20754), nr_events: 1
task    232 (             systemd:     21235), nr_events: 1
task    233 (             systemd:     21903), nr_events: 1
task    234 (             systemd:     22235), nr_events: 3
task    235 (             systemd:     22377), nr_events: 3
task    236 (             systemd:     22378), nr_events: 9
task    237 (             systemd:     26053), nr_events: 1
task    238 (               snapd:     26056), nr_events: 1
task    239 (               snapd:     26057), nr_events: 1
task    240 (               snapd:     26058), nr_events: 1
task    241 (               snapd:     26059), nr_events: 1
task    242 (               snapd:     26060), nr_events: 1
task    243 (               snapd:     26061), nr_events: 1
task    244 (               snapd:     26064), nr_events: 1
task    245 (               snapd:     28974), nr_events: 1
task    246 (               snapd:     28975), nr_events: 1
task    247 (               snapd:     28976), nr_events: 1
task    248 (               snapd:     28978), nr_events: 1
task    249 (               snapd:     28981), nr_events: 1
task    250 (               snapd:      5703), nr_events: 1
------------------------------------------------------------



#1  : 6069.221, ravg: 1774.25, cpu: 146.19 / 146.19
#2  : 6073.281, ravg: 1774.25, cpu: 154.43 / 146.19
#3  : 6073.313, ravg: 1774.25, cpu: 142.62 / 146.19
#4  : 6073.397, ravg: 1774.25, cpu: 142.44 / 146.19
#5  : 6073.619, ravg: 1774.25, cpu: 142.11 / 146.19
#6  : 6073.654, ravg: 1774.25, cpu: 157.64 / 146.19
#7  : 6073.800, ravg: 1774.25, cpu: 148.40 / 146.19
#8  : 6073.710, ravg: 1774.25, cpu: 163.57 / 146.19
#9  : 6073.750, ravg: 1774.25, cpu: 145.56 / 146.19
#10 : 6073.758, ravg: 1774.25, cpu: 164.83 / 146.19
#11 : 6073.536, ravg: 1774.25, cpu: 142.05 / 146.19
#12 : 6073.624, ravg: 1774.25, cpu: 164.02 / 146.19
#13 : 6073.490, ravg: 1774.25, cpu: 158.63 / 146.19
#14 : 6073.483, ravg: 1774.25, cpu: 153.94 / 146.19
#15 : 6073.605, ravg: 1774.25, cpu: 143.49 / 146.19
#16 : 6073.405, ravg: 1774.25, cpu: 152.95 / 146.19
#17 : 6073.260, ravg: 1774.25, cpu: 138.25 / 146.19
#18 : 6073.453, ravg: 1774.25, cpu: 145.95 / 146.19
#19 : 6073.563, ravg: 1774.25, cpu: 156.50 / 146.19
#20 : 6073.834, ravg: 1774.25, cpu: 162.31 / 146.19
#21 : 6073.731, ravg: 1774.25, cpu: 144.22 / 146.19
#22 : 6073.490, ravg: 1774.25, cpu: 147.31 / 146.19
#23 : 6073.557, ravg: 1774.25, cpu: 144.10 / 146.19
#24 : 6073.236, ravg: 1774.25, cpu: 130.90 / 146.19
#25 : 6072.648, ravg: 1774.25, cpu: 138.24 / 146.19
#26 : 6073.640, ravg: 1774.25, cpu: 144.45 / 146.19
#27 : 6073.547, ravg: 1774.25, cpu: 141.69 / 146.19
#28 : 6073.778, ravg: 1774.25, cpu: 155.02 / 146.19
#29 : 6074.010, ravg: 1774.25, cpu: 146.82 / 146.19
#30 : 6073.574, ravg: 1774.25, cpu: 152.83 / 146.19
...
（会一直持续下去）
```

可以看到此时 perf 会占用一个 CPU 核心 100% ；

```
root@backend-shared-stag-0:~/workspace# top
top - 06:47:38 up 107 days,  1:27,  2 users,  load average: 0.76, 0.25, 0.09
Tasks: 182 total,   1 running, 181 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  : 25.2 us, 74.4 sy,  0.0 ni,  0.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu6  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu7  :  0.0 us,  0.3 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 32946256 total, 30253396 free,   213068 used,  2479792 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 32168924 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
12690 root      20   0   61708   7544   4512 S 101.0  0.0   1:26.58 perf
 1109 root      20   0   27176  22180   8480 S   0.7  0.1 196:01.58 node_exporter
 1072 root      20   0  606792  45576  26528 S   0.3  0.1 189:48.53 dockerd
    1 root      20   0  185352   5972   4024 S   0.0  0.0   0:54.55 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.18 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   0:00.29 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
    7 root      20   0       0      0      0 S   0.0  0.0   2:30.05 rcu_sched
    8 root      20   0       0      0      0 S   0.0  0.0   0:00.00 rcu_bh
...
```

### eBPF

As of Linux **4.4**, `perf` has some **enhanced BPF support** (aka `eBPF` or just "`BPF`"), with more in later kernels. BPF makes `perf` tracing programmatic, and **takes `perf` from being a counting & sampling-with-post-processing tracer, to a fully in-kernel programmable tracer**.

`eBPF` is currently a little restricted and difficult to use from `perf`. It's getting better all the time. A different and currently easier way to access `eBPF` is via the `bcc` Python interface, which is described on my [eBPF Tools](http://www.brendangregg.com/ebpf.html) page. On this page, I'll discuss `perf`.

> **Prerequisites**
> Linux **4.4** at least. Newer versions have more `perf/BPF` features, so the newer the better. Also `clang` (eg, `apt-get install clang`).

失败

## Visualizations

`perf_events` has a **builtin** visualization: `timecharts`, as well as text-style visualization via its **text user interface** (`TUI`) and tree reports. The following two sections show visualizations of my own: **flame graphs** and **heat maps**. The software I'm using is open source and on github, and produces these from `perf_events` collected data.

可视化的方法：

- timecharts
- TUI
- flame graphs
- heat maps


### Flame Graphs

[Flame Graphs](http://www.brendangregg.com/flamegraphs.html) can be produced from `perf_events` profiling data using the [FlameGraph tools](https://github.com/brendangregg/FlameGraph) software. This visualizes the same data you see in `perf report`, and works with any `perf.data` file that was captured with stack traces (`-g`).

#### Example

This example CPU flame graph shows a network workload for the 3.2.9-1 Linux kernel, running as a KVM instance

![](http://www.brendangregg.com/FlameGraphs/cpu-linux-tcpsend.svg)

Flame Graphs show the **sample population** across the **x-axis**, and **stack depth** on the **y-axis**. Each function (`stack frame`) is drawn as a **rectangle**, with the **width** relative to the number of samples. See the [CPU Flame Graphs](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs) page for the full description of how these work.

You can use the mouse to **explore where kernel CPU time is spent**, quickly **quantifying code-paths** and **determining where performance tuning efforts are best spent**. This example shows that most time was spent in the `vp_notify()` code-path, spending 70.52% of all on-CPU samples performing `iowrite16()`, which is handled by the KVM hypervisor. This information has been extremely useful for directing KVM performance efforts.

A similar network workload on a bare metal Linux system looks quite different, as networking isn't processed via the `virtio-net` driver, for a start.

#### Generation

The example flame graph was generated using `perf_events` and the [FlameGraph tools](https://github.com/brendangregg/FlameGraph):

```
root@backend-shared-stag-0:~/workspace# git clone https://github.com/brendangregg/FlameGraph
Cloning into 'FlameGraph'...
remote: Counting objects: 949, done.
remote: Total 949 (delta 0), reused 0 (delta 0), pack-reused 949
Receiving objects: 100% (949/949), 1.82 MiB | 40.00 KiB/s, done.
Resolving deltas: 100% (541/541), done.
Checking connectivity... done.
root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace#
root@backend-shared-stag-0:~/workspace# ll
total 324
drwxr-xr-x 3 root root   4096 Apr 25 07:55 ./
drwx------ 5 root root   4096 Apr 25 07:15 ../
drwxr-xr-x 7 root root   4096 Apr 25 07:56 FlameGraph/
-rw-r--r-- 1 root root   1425 Apr 25 07:15 kca_from.c
-rw-r--r-- 1 root root  80960 Apr 25 06:32 output.svg
-rw------- 1 root root 230216 Apr 25 05:47 perf.data
root@backend-shared-stag-0:~/workspace# cd FlameGraph
root@backend-shared-stag-0:~/workspace/FlameGraph# ll
total 2208
drwxr-xr-x 7 root root    4096 Apr 25 07:56 ./
drwxr-xr-x 3 root root    4096 Apr 25 07:55 ../
drwxr-xr-x 8 root root    4096 Apr 25 07:56 .git/
-rw-r--r-- 1 root root     208 Apr 25 07:56 .travis.yml
-rw-r--r-- 1 root root    8439 Apr 25 07:56 README.md
-rwxr-xr-x 1 root root     638 Apr 25 07:56 aix-perf.pl*
drwxr-xr-x 2 root root    4096 Apr 25 07:56 demos/
drwxr-xr-x 2 root root    4096 Apr 25 07:56 dev/
-rwxr-xr-x 1 root root    3640 Apr 25 07:56 difffolded.pl*
drwxr-xr-x 2 root root    4096 Apr 25 07:56 docs/
-rw-r--r-- 1 root root 1382457 Apr 25 07:56 example-dtrace-stacks.txt
-rw-r--r-- 1 root root  155251 Apr 25 07:56 example-dtrace.svg
-rw-r--r-- 1 root root  110532 Apr 25 07:56 example-perf-stacks.txt.gz
-rw-r--r-- 1 root root  404484 Apr 25 07:56 example-perf.svg
-rwxr-xr-x 1 root root    1293 Apr 25 07:56 files.pl*
-rwxr-xr-x 1 root root   34109 Apr 25 07:56 flamegraph.pl*
-rwxr-xr-x 1 root root    2798 Apr 25 07:56 jmaps*
-rwxr-xr-x 1 root root    2710 Apr 25 07:56 pkgsplit-perf.pl*
-rwxr-xr-x 1 root root    4517 Apr 25 07:56 range-perf.pl*
-rwxr-xr-x 1 root root     669 Apr 25 07:56 record-test.sh*
-rwxr-xr-x 1 root root    1185 Apr 25 07:56 stackcollapse-aix.pl*
-rwxr-xr-x 1 root root    2304 Apr 25 07:56 stackcollapse-elfutils.pl*
-rwxr-xr-x 1 root root    1816 Apr 25 07:56 stackcollapse-gdb.pl*
-rwxr-xr-x 1 root root    3632 Apr 25 07:56 stackcollapse-go.pl*
-rwxr-xr-x 1 root root     514 Apr 25 07:56 stackcollapse-instruments.pl*
-rwxr-xr-x 1 root root    5159 Apr 25 07:56 stackcollapse-jstack.pl*
-rwxr-xr-x 1 root root    1859 Apr 25 07:56 stackcollapse-ljp.awk*
-rwxr-xr-x 1 root root    6527 Apr 25 07:56 stackcollapse-perf-sched.awk*
-rwxr-xr-x 1 root root   10309 Apr 25 07:56 stackcollapse-perf.pl*
-rwxr-xr-x 1 root root    2663 Apr 25 07:56 stackcollapse-pmc.pl*
-rwxr-xr-x 1 root root    1569 Apr 25 07:56 stackcollapse-recursive.pl*
-rwxr-xr-x 1 root root    7871 Apr 25 07:56 stackcollapse-sample.awk*
-rwxr-xr-x 1 root root    2310 Apr 25 07:56 stackcollapse-stap.pl*
-rwxr-xr-x 1 root root    2983 Apr 25 07:56 stackcollapse-vsprof.pl*
-rw-r--r-- 1 root root    2613 Apr 25 07:56 stackcollapse-vtune.pl
-rwxr-xr-x 1 root root    2862 Apr 25 07:56 stackcollapse.pl*
drwxr-xr-x 3 root root    4096 Apr 25 07:56 test/
-rwxr-xr-x 1 root root     861 Apr 25 07:56 test.sh*
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph# perf record -F 99 -ag -- sleep 60
[ perf record: Woken up 9 times to write data ]
[ perf record: Captured and wrote 5.361 MB perf.data (47520 samples) ]
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph# perf script | ./stackcollapse-perf.pl > out.perf-folded
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
/lib/x86_64-linux-gnu/libc-2.23.so was updated (is prelink enabled?). Restart the long running apps that use it!
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph# cat out.perf-folded | ./flamegraph.pl > perf-kernel.svg
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph#
root@backend-shared-stag-0:~/workspace/FlameGraph# ll
total 7720
drwxr-xr-x 7 root root    4096 Apr 25 07:59 ./
drwxr-xr-x 3 root root    4096 Apr 25 07:55 ../
drwxr-xr-x 8 root root    4096 Apr 25 07:56 .git/
-rw-r--r-- 1 root root     208 Apr 25 07:56 .travis.yml
-rw-r--r-- 1 root root    8439 Apr 25 07:56 README.md
-rwxr-xr-x 1 root root     638 Apr 25 07:56 aix-perf.pl*
drwxr-xr-x 2 root root    4096 Apr 25 07:56 demos/
drwxr-xr-x 2 root root    4096 Apr 25 07:56 dev/
-rwxr-xr-x 1 root root    3640 Apr 25 07:56 difffolded.pl*
drwxr-xr-x 2 root root    4096 Apr 25 07:56 docs/
-rw-r--r-- 1 root root 1382457 Apr 25 07:56 example-dtrace-stacks.txt
-rw-r--r-- 1 root root  155251 Apr 25 07:56 example-dtrace.svg
-rw-r--r-- 1 root root  110532 Apr 25 07:56 example-perf-stacks.txt.gz
-rw-r--r-- 1 root root  404484 Apr 25 07:56 example-perf.svg
-rwxr-xr-x 1 root root    1293 Apr 25 07:56 files.pl*
-rwxr-xr-x 1 root root   34109 Apr 25 07:56 flamegraph.pl*
-rwxr-xr-x 1 root root    2798 Apr 25 07:56 jmaps*
-rw-r--r-- 1 root root    1013 Apr 25 07:58 out.perf-folded
-rw-r--r-- 1 root root   16360 Apr 25 07:59 perf-kernel.svg
-rw------- 1 root root 5623796 Apr 25 07:58 perf.data
-rwxr-xr-x 1 root root    2710 Apr 25 07:56 pkgsplit-perf.pl*
-rwxr-xr-x 1 root root    4517 Apr 25 07:56 range-perf.pl*
-rwxr-xr-x 1 root root     669 Apr 25 07:56 record-test.sh*
-rwxr-xr-x 1 root root    1185 Apr 25 07:56 stackcollapse-aix.pl*
-rwxr-xr-x 1 root root    2304 Apr 25 07:56 stackcollapse-elfutils.pl*
-rwxr-xr-x 1 root root    1816 Apr 25 07:56 stackcollapse-gdb.pl*
-rwxr-xr-x 1 root root    3632 Apr 25 07:56 stackcollapse-go.pl*
-rwxr-xr-x 1 root root     514 Apr 25 07:56 stackcollapse-instruments.pl*
-rwxr-xr-x 1 root root    5159 Apr 25 07:56 stackcollapse-jstack.pl*
-rwxr-xr-x 1 root root    1859 Apr 25 07:56 stackcollapse-ljp.awk*
-rwxr-xr-x 1 root root    6527 Apr 25 07:56 stackcollapse-perf-sched.awk*
-rwxr-xr-x 1 root root   10309 Apr 25 07:56 stackcollapse-perf.pl*
-rwxr-xr-x 1 root root    2663 Apr 25 07:56 stackcollapse-pmc.pl*
-rwxr-xr-x 1 root root    1569 Apr 25 07:56 stackcollapse-recursive.pl*
-rwxr-xr-x 1 root root    7871 Apr 25 07:56 stackcollapse-sample.awk*
-rwxr-xr-x 1 root root    2310 Apr 25 07:56 stackcollapse-stap.pl*
-rwxr-xr-x 1 root root    2983 Apr 25 07:56 stackcollapse-vsprof.pl*
-rw-r--r-- 1 root root    2613 Apr 25 07:56 stackcollapse-vtune.pl
-rwxr-xr-x 1 root root    2862 Apr 25 07:56 stackcollapse.pl*
drwxr-xr-x 3 root root    4096 Apr 25 07:56 test/
-rwxr-xr-x 1 root root     861 Apr 25 07:56 test.sh*
root@backend-shared-stag-0:~/workspace/FlameGraph#
```

> 存在一些告警信息，可能需要看一下（http://lkml.iu.edu/hypermail/linux/kernel/1110.2/01190.html）
>
> locale settings 相关错误解决：在 .bashrc 中添加
>
> ```
> export LANG=en_US.UTF-8
> export LC_CTYPE="en_US.UTF-8"
> export LC_ALL=en_US.UTF-8
> TZ='Asia/Shanghai'; export TZ
> ```


The first `perf` command profiles CPU stacks, as explained earlier. I adjusted the rate to **99 Hertz** here; **I actually generated the flame graph from a 1000 Hertz profile**, but I'd only use that if you had a reason to go faster, which costs more in overhead. The samples are saved in a `perf.data` file, which can be viewed using `perf report`:

```
# perf report --stdio
[...]
# Overhead          Command          Shared Object                               Symbol
# ........  ...............  .....................  ...................................
#
    72.18%            iperf  [kernel.kallsyms]      [k] iowrite16
                      |
                      --- iowrite16
                         |          
                         |--99.53%-- vp_notify
                         |          virtqueue_kick
                         |          start_xmit
                         |          dev_hard_start_xmit
                         |          sch_direct_xmit
                         |          dev_queue_xmit
                         |          ip_finish_output
                         |          ip_output
                         |          ip_local_out
                         |          ip_queue_xmit
                         |          tcp_transmit_skb
                         |          tcp_write_xmit
                         |          |          
                         |          |--98.16%-- tcp_push_one
                         |          |          tcp_sendmsg
                         |          |          inet_sendmsg
                         |          |          sock_aio_write
                         |          |          do_sync_write
                         |          |          vfs_write
                         |          |          sys_write
                         |          |          system_call
                         |          |          0x369e40e5cd
                         |          |          
                         |           --1.84%-- __tcp_push_pending_frames
[...]
```

This tree follows the flame graph when reading it **top-down**. When using `-g/--call-graph` (for "caller", instead of the "callee" default), it generates a tree that follows the flame graph when read **bottom-up**. The **hottest stack trace** in the flame graph (@70.52%) can be seen in this perf call graph as the product of the top three nodes (72.18% x 99.53% x 98.16%).

The `perf report` tree (and the ncurses navigator) do an excellent job at presenting this information as text. However, with text there are limitations. The output often does not fit in one screen (you could say it doesn't need to, if the bulk of the samples are identified on the first page). Also, identifying the hottest code paths requires reading the percentages. **With the flame graph, all the data is on screen at once, and `the hottest code-paths` are immediately obvious as the `widest functions`**.

> 火焰图的好处

For generating the flame graph, the `perf script` command **dumps the stack samples**, which are then **aggregated** by `stackcollapse-perf.pl` and **folded** into single lines per-stack. That output is then **converted** by `flamegraph.pl` **into the SVG**. I included a gratuitous "cat" command to make it clear that `flamegraph.pl` can process the output of a pipe, which could include Unix commands to filter or preprocess (`grep`, `sed`, `awk`).

命令说明

- `perf script`: **dump the stack samples**
- `stackcollapse-perf.pl`: **aggregat stack samples**, and **folded into single lines per-stack**
- `flamegraph.pl`: **converted** output **into the SVG**


我的输出

```
root@backend-shared-stag-0:~/workspace/FlameGraph# perf report --stdio
/lib/x86_64-linux-gnu/libc-2.23.so was updated (is prelink enabled?). Restart the long running apps that use it!
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 47K of event 'cpu-clock'
# Event count (approx.): 479999995200
#
# Children      Self  Command        Shared Object           Symbol
# ........  ........  .............  ......................  ..........................................
#
    99.99%     0.00%  swapper        [kernel.kallsyms]       [k] default_idle
                  |
                  ---default_idle
                     |
                     |--99.99%-- native_safe_halt
                     |
                      --0.00%-- xen_hvm_callback_vector
                                xen_evtchn_do_upcall
                                irq_exit
                                __do_softirq
                                net_rx_action
                                ixgbevf_poll
                                ixgbevf_clean_rx_irq
                                napi_gro_receive
                                netif_receive_skb_internal
                                __netif_receive_skb
                                __netif_receive_skb_core
                                ip_rcv
                                ip_rcv_finish
                                ip_local_deliver
                                ip_local_deliver_finish
                                tcp_v4_rcv

    99.99%     0.00%  swapper        [kernel.kallsyms]       [k] arch_cpu_idle
                  |
                  ---arch_cpu_idle
                     default_idle
                     |
                     |--99.99%-- native_safe_halt
                     |
                      --0.00%-- xen_hvm_callback_vector
                                xen_evtchn_do_upcall
                                irq_exit
                                __do_softirq
                                net_rx_action
                                ixgbevf_poll
                                ixgbevf_clean_rx_irq
                                napi_gro_receive
                                netif_receive_skb_internal
                                __netif_receive_skb
                                __netif_receive_skb_core
                                ip_rcv
                                ip_rcv_finish
                                ip_local_deliver
                                ip_local_deliver_finish
                                tcp_v4_rcv

    99.99%     0.00%  swapper        [kernel.kallsyms]       [k] default_idle_call
                  |
                  ---default_idle_call
                     arch_cpu_idle
                     default_idle
                     |
                     |--99.99%-- native_safe_halt
                     |
...
```


#### Piping

A flame graph can be generated directly by piping all the steps:

```
# perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > perf-kernel.svg
```

In practice I don't do this, as I often re-run `flamegraph.pl` multiple times, and this one-liner would execute everything multiple times. The output of `perf script` can be dozens of Mbytes, taking many seconds to process. By writing `stackcollapse-perf.pl` to a file, you've cached the slowest step, and **can also edit the file (vi) to delete unimportant stacks**, such as CPU idle threads.


#### Filtering

The one-line-per-stack output of `stackcollapse-perf.pl` is also convenient for `grep(1)`. Eg:

```
# perf script | ./stackcollapse-perf.pl > out.perf-folded

# grep -v cpu_idle out.perf-folded | ./flamegraph.pl > nonidle.svg

# grep ext4 out.perf-folded | ./flamegraph.pl > ext4internals.svg

# egrep 'system_call.*sys_(read|write)' out.perf-folded | ./flamegraph.pl > rw.svg
```

**I frequently elide the `cpu_idle` threads in this way, to focus on the real threads that are consuming CPU resources**. If I miss this step, the `cpu_idle` threads can often dominate the flame graph, squeezing the interesting code paths.

Note that it would be a little more efficient to process the output of `perf report` instead of `perf script`; better still, `perf report` could have a report style (eg, "`-g folded`") that output folded stacks directly, obviating the need for `stackcollapse-perf.pl`. There could even be a `perf` mode that output the SVG directly (which wouldn't be the first one; see `perf-timechar`t), although, that would miss the value of being able to grep the folded stacks (which I use frequently).

There are more examples of `perf_events` CPU flame graphs on the [CPU flame graph](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html#Examples) page, including a [summary](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html#perf) of these instructions. I have also shared an example of using perf for a [Block Device I/O Flame Graph](http://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html#BlockIO).

### Heat Maps

Since `perf_events` can record high resolution timestamps (microseconds) for events, some **latency measurements** can be derived from trace data.

#### Example

The following heat map visualizes **disk I/O latency data** collected from `perf_events`

![](http://www.brendangregg.com/perf_events/perf_block_latencyheatmap.svg)

Mouse-over blocks to explore the latency distribution over time. The **x-axis** is the passage of time, the **y-axis** latency, and the **z-axis** (`color`) is the number of I/O at that time and latency range. The distribution is bimodal, with the dark line at the bottom showing that many disk I/O completed with sub-millisecond latency: cache hits. There is a cloud of disk I/O from about 3 ms to 25 ms, which would be caused by random disk I/O (and queueing). Both these modes averaged to the 9 ms we saw earlier.

The following `iostat` output was collected at the same time as the heat map data was collected (shows a typical one second summary):

```
# iostat -x 1
[...]
Device: rrqm/s wrqm/s    r/s   w/s   rkB/s wkB/s avgrq-sz avgqu-sz await r_await w_await svctm  %util
vda       0.00   0.00   0.00  0.00    0.00  0.00     0.00     0.00  0.00    0.00    0.00  0.00   0.00
vdb       0.00   0.00 334.00  0.00 2672.00  0.00    16.00     2.97  9.01    9.01    0.00  2.99 100.00
```

This workload has **an average I/O time (await) of 9 milliseconds, which sounds like a fairly random workload on 7200 RPM disks**. The problem is that we don't know the distribution from the `iostat` output, or any similar latency average. There could be latency outliers present, which is not visible in the average, and yet are causing problems. The heat map did show I/O up to 50 ms, which you might not have expected from that `iostat` output. There could also be multiple modes, as we saw in the heat map, which are also not visible in an average.

（略）



----------


## libc6-dbgsym 安装过程

- 直接安装，失败，找不到包

```
root@backend-shared-stag-0:~# apt install libc6-dbgsym coreutils-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
Package libc6-dbgsym is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'libc6-dbgsym' has no installation candidate
E: Unable to locate package coreutils-dbgsym
root@backend-shared-stag-0:~#
```

- 添加 apt 源，导入 key ，update 三板斧（参考《[Debug Symbol Packages](https://wiki.ubuntu.com/Debug%20Symbol%20Packages)》）

```
root@backend-shared-stag-0:~# echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse" >> /etc/apt/sources.list.d/ddebs.list
root@backend-shared-stag-0:~# cat /etc/apt/sources.list.d/ddebs.list
deb http://ddebs.ubuntu.com xenial main restricted universe multiverse
root@backend-shared-stag-0:~#
root@backend-shared-stag-0:~# apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 428D7C01 C8CAB6595FDFF622
Executing: /tmp/tmp.9RC2FSZ6Yz/gpg.1.sh --keyserver
keyserver.ubuntu.com
--recv-keys
428D7C01
C8CAB6595FDFF622
gpg: requesting key 428D7C01 from hkp server keyserver.ubuntu.com
gpg: requesting key 5FDFF622 from hkp server keyserver.ubuntu.com
gpg: key 428D7C01: public key "Ubuntu Debug Symbol Archive Automatic Signing Key <ubuntu-archive@lists.ubuntu.com>" imported
gpg: key 5FDFF622: public key "Ubuntu Debug Symbol Archive Automatic Signing Key (2016) <ubuntu-archive@lists.ubuntu.com>" imported
gpg: Total number processed: 2
gpg:               imported: 2  (RSA: 1)
root@backend-shared-stag-0:~#
root@backend-shared-stag-0:~# apt-get update
Hit:1 http://security.ubuntu.com/ubuntu xenial-security InRelease
Ign:2 http://ddebs.ubuntu.com xenial InRelease
Hit:3 http://ddebs.ubuntu.com xenial Release
Hit:4 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial InRelease
Get:5 http://ddebs.ubuntu.com xenial Release.gpg [819 B]
Hit:6 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-updates InRelease
Hit:7 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-backports InRelease
Fetched 819 B in 1s (544 B/s)
Reading package lists... Done
root@backend-shared-stag-0:~#
```

- 重新安装 libc6-dbgsym ，失败，遇到无法匹配的依赖关系

```
root@backend-shared-stag-0:~# apt-get install libc6-dbgsym coreutils-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libc6-dbgsym : Depends: libc6 (= 2.23-0ubuntu3) but 2.23-0ubuntu10 is to be installed
E: Unable to correct problems, you have held broken packages.
root@backend-shared-stag-0:~#
```

- 尝试对 libc6 进行了 2.23-0ubuntu5 到 2.23-0ubuntu10 的更新（可能并不需要）

```
root@backend-shared-stag-0:~# apt-get install libc6
Reading package lists... Done
Building dependency tree
Reading state information... Done
Suggested packages:
  glibc-doc
The following packages will be upgraded:
  libc6
1 upgraded, 0 newly installed, 0 to remove and 215 not upgraded.
Need to get 2580 kB of archives.
After this operation, 5120 B of additional disk space will be used.
Get:1 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-updates/main amd64 libc6 amd64 2.23-0ubuntu10 [2580 kB]
Fetched 2580 kB in 3s (844 kB/s)
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
Preconfiguring packages ...
(Reading database ... 67285 files and directories currently installed.)
Preparing to unpack .../libc6_2.23-0ubuntu10_amd64.deb ...
Unpacking libc6:amd64 (2.23-0ubuntu10) over (2.23-0ubuntu5) ...
Setting up libc6:amd64 (2.23-0ubuntu10) ...
Processing triggers for libc-bin (2.23-0ubuntu5) ...
root@backend-shared-stag-0:~#
```

- 再次尝试安装 libc6-dbgsym ，失败

```
root@backend-shared-stag-0:~# apt-get install libc6-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libc6-dbgsym : Depends: libc6 (= 2.23-0ubuntu3) but 2.23-0ubuntu10 is to be installed
E: Unable to correct problems, you have held broken packages.
root@backend-shared-stag-0:~#
```

- 尝试解决 "unmet dependencies" 问题（参考《[can't install libc6 package
](https://stackoverflow.com/questions/30451939/cant-install-libc6-package)》）

```
root@backend-shared-stag-0:~# apt-get clean

root@backend-shared-stag-0:~# apt-get update
Hit:1 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial InRelease
Hit:2 http://security.ubuntu.com/ubuntu xenial-security InRelease
Ign:3 http://ddebs.ubuntu.com xenial InRelease
Hit:4 http://ddebs.ubuntu.com xenial Release
Hit:5 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-updates InRelease
Get:6 http://ddebs.ubuntu.com xenial Release.gpg [819 B]
Hit:7 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-backports InRelease
Fetched 819 B in 2s (285 B/s)
Reading package lists... Done
root@backend-shared-stag-0:~#

root@backend-shared-stag-0:~# apt-get -f install
Reading package lists... Done
Building dependency tree
Reading state information... Done
0 upgraded, 0 newly installed, 0 to remove and 215 not upgraded.
root@backend-shared-stag-0:~#

root@backend-shared-stag-0:~# apt-get upgrade
Reading package lists... Done
Building dependency tree
Reading state information... Done
Calculating upgrade... Done
The following package was automatically installed and is no longer required:
  snap-confine
Use 'sudo apt autoremove' to remove it.
The following packages have been kept back:
  libdrm2 linux-headers-generic linux-headers-virtual linux-image-extra-virtual linux-image-generic linux-image-virtual linux-tools-generic linux-virtual
The following packages will be upgraded:
  apparmor apport apt apt-transport-https apt-utils base-files bash bind9-host bsdutils btrfs-tools ca-certificates cloud-guest-utils cloud-init cloud-initramfs-copymods cloud-initramfs-dyn-netconf console-setup console-setup-linux coreutils cryptsetup cryptsetup-bin
  curl distro-info-data dnsmasq-base dnsutils dpkg eject friendly-recovery gcc-5-base git git-man grub-common grub-legacy-ec2 grub-pc grub-pc-bin grub2-common hdparm init init-system-helpers initramfs-tools initramfs-tools-bin initramfs-tools-core iproute2
  isc-dhcp-client isc-dhcp-common keyboard-configuration klibc-utils kmod krb5-locales less libapparmor-perl libapparmor1 libapt-inst2.0 libapt-pkg5.0 libasn1-8-heimdal libaudit-common libaudit1 libbind9-140 libblkid1 libc-bin libcryptsetup4 libcurl3-gnutls libdb5.3
  libdns-export162 libdns162 libevent-2.0-5 libexpat1 libfdisk1 libfreetype6 libgcrypt20 libgnutls-openssl27 libgnutls30 libgssapi-krb5-2 libgssapi3-heimdal libhcrypto4-heimdal libheimbase1-heimdal libheimntlm0-heimdal libhogweed4 libhx509-5-heimdal libicu55 libidn11
  libisc-export160 libisc160 libisccc140 libisccfg140 libk5crypto3 libklibc libkmod2 libkrb5-26-heimdal libkrb5-3 libkrb5support0 libldap-2.4-2 liblwres141 liblxc1 libmount1 libmspack0 libnettle6 libnl-3-200 libnl-genl-3-200 libnuma1 libpam-modules libpam-modules-bin
  libpam-runtime libpam-systemd libpam0g libparted2 libpci3 libperl5.22 libplymouth4 libpython-stdlib libpython2.7-minimal libpython2.7-stdlib libpython3.5 libpython3.5-minimal libpython3.5-stdlib libroken18-heimdal librtmp1 libseccomp2 libsmartcols1 libssl1.0.0
  libstdc++6 libsystemd0 libtasn1-6 libtiff5 libudev1 libuuid1 libwind0-heimdal libxml2 linux-firmware linux-tools-common locales login logrotate lshw lxc-common lxcfs lxd lxd-client makedev mdadm mount multiarch-support nano ntp open-iscsi open-vm-tools openssh-client
  openssh-server openssh-sftp-server openssl os-prober overlayroot parted passwd patch pciutils perl perl-base perl-modules-5.22 plymouth plymouth-theme-ubuntu-text pollinate python python-apt python-apt-common python-minimal python2.7 python2.7-minimal python3-apport
  python3-apt python3-distupgrade python3-jwt python3-problem-report python3-software-properties python3-update-manager python3.5 python3.5-minimal resolvconf rsync sensible-utils snap-confine snapd software-properties-common sosreport squashfs-tools sudo systemd
  systemd-sysv tcpdump thermald tzdata ubuntu-core-launcher ubuntu-minimal ubuntu-release-upgrader-core ubuntu-server ubuntu-standard udev uidmap unattended-upgrades update-manager-core update-notifier-common util-linux uuid-runtime vlan wget xdg-user-dirs xfsprogs
  zlib1g
207 upgraded, 0 newly installed, 0 to remove and 8 not upgraded.
Need to get 137 MB of archives.
After this operation, 89.5 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
...

root@backend-shared-stag-0:~# apt-get -u dist-upgrade
Reading package lists... Done
Building dependency tree
Reading state information... Done
Calculating upgrade... Done
The following packages were automatically installed and are no longer required:
  linux-tools-4.4.0-119 linux-tools-4.4.0-119-generic snap-confine
Use 'sudo apt autoremove' to remove them.
The following NEW packages will be installed:
  libdrm-common linux-headers-4.4.0-121 linux-headers-4.4.0-121-generic linux-image-4.4.0-121-generic
  linux-image-extra-4.4.0-121-generic linux-tools-4.4.0-121 linux-tools-4.4.0-121-generic
The following packages will be upgraded:
  libdrm2 linux-headers-generic linux-headers-virtual linux-image-extra-virtual linux-image-generic linux-image-virtual
  linux-tools-generic linux-virtual
8 upgraded, 7 newly installed, 0 to remove and 0 not upgraded.
Need to get 70.2 MB of archives.
After this operation, 305 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
...
```

- 再次安装 libc6-dbgsym ，失败

```
root@backend-shared-stag-0:~# apt-get install libc6-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libc6-dbgsym : Depends: libc6 (= 2.23-0ubuntu3) but 2.23-0ubuntu10 is to be installed
E: Unable to correct problems, you have held broken packages.
root@backend-shared-stag-0:~#
```

- 降级 libc6 版本以解决 "unmet dependencies" 问题（参考《[Installed wrong libc6-dev Version](https://askubuntu.com/questions/749076/installed-wrong-libc6-dev-version)》）

```
root@backend-shared-stag-0:~# apt list libc6
Listing... Done
libc6/xenial-updates,xenial-security,now 2.23-0ubuntu10 amd64 [installed]
N: There is 1 additional version. Please use the '-a' switch to see it
root@backend-shared-stag-0:~# apt show libc6
Package: libc6
Version: 2.23-0ubuntu10
Priority: required
Section: libs
Source: glibc
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Original-Maintainer: GNU Libc Maintainers <debian-glibc@lists.debian.org>
Bugs: https://bugs.launchpad.net/ubuntu/+filebug
Installed-Size: 11.2 MB
Depends: libgcc1
Suggests: glibc-doc, debconf | debconf-2.0, locales
Breaks: hurd (<< 1:0.5.git20140203-1), libtirpc1 (<< 0.2.3), locales (<< 2.23), locales-all (<< 2.23), lsb-core (<= 3.2-27), nscd (<< 2.23)
Replaces: libc6-amd64
Homepage: http://www.gnu.org/software/libc/libc.html
Task: minimal
Supported: 5y
Download-Size: 2580 kB
APT-Manual-Installed: yes
APT-Sources: http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial-updates/main amd64 Packages
Description: GNU C Library: Shared libraries
 Contains the standard libraries that are used by nearly all programs on
 the system. This package includes shared versions of the standard C library
 and the standard math library, as well as many others.

N: There is 1 additional record. Please use the '-a' switch to see it
root@backend-shared-stag-0:~#


root@backend-shared-stag-0:~# apt search libc6-dbgsym
Sorting... Done
Full Text Search... Done
libc6-dbgsym/xenial 2.23-0ubuntu3 amd64
  debug symbols for package libc6

root@backend-shared-stag-0:~#



root@backend-shared-stag-0:~# apt-get install libc6=2.23-0ubuntu3
Reading package lists... Done
Building dependency tree
Reading state information... Done
Suggested packages:
  glibc-doc
The following packages will be DOWNGRADED:
  libc6
0 upgraded, 0 newly installed, 1 downgraded, 0 to remove and 0 not upgraded.
Need to get 2584 kB of archives.
After this operation, 6144 B disk space will be freed.
Do you want to continue? [Y/n] y
Get:1 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial/main amd64 libc6 amd64 2.23-0ubuntu3 [2584 kB]
Fetched 2584 kB in 3s (826 kB/s)
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
Preconfiguring packages ...
dpkg: warning: downgrading libc6:amd64 from 2.23-0ubuntu10 to 2.23-0ubuntu3
(Reading database ... 100002 files and directories currently installed.)
Preparing to unpack .../libc6_2.23-0ubuntu3_amd64.deb ...
Unpacking libc6:amd64 (2.23-0ubuntu3) over (2.23-0ubuntu10) ...
Setting up libc6:amd64 (2.23-0ubuntu3) ...
Processing triggers for libc-bin (2.23-0ubuntu10) ...
root@backend-shared-stag-0:~#
```

- 重新安装 libc6-dbgsym ，成功

```
root@backend-shared-stag-0:~# apt-get install libc6-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  libc6-dbgsym
0 upgraded, 1 newly installed, 0 to remove and 1 not upgraded.
Need to get 3568 kB of archives.
After this operation, 24.5 MB of additional disk space will be used.
WARNING: The following packages cannot be authenticated!
  libc6-dbgsym
Install these packages without verification? [y/N] y
Get:1 http://ddebs.ubuntu.com xenial/main amd64 libc6-dbgsym amd64 2.23-0ubuntu3 [3568 kB]
Fetched 3568 kB in 4s (887 kB/s)
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
Selecting previously unselected package libc6-dbgsym:amd64.
(Reading database ... 100002 files and directories currently installed.)
Preparing to unpack .../libc6-dbgsym_2.23-0ubuntu3_amd64.ddeb ...
Unpacking libc6-dbgsym:amd64 (2.23-0ubuntu3) ...
Setting up libc6-dbgsym:amd64 (2.23-0ubuntu3) ...
root@backend-shared-stag-0:~#
```

- 安装 coreutils-dbgsym ，失败，遇到同样的问题

```
root@backend-shared-stag-0:~# apt-get install coreutils-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 coreutils-dbgsym : Depends: coreutils (= 8.25-2ubuntu2) but 8.25-2ubuntu3~16.04 is to be installed
E: Unable to correct problems, you have held broken packages.
root@backend-shared-stag-0:~#
```

- 降级 coreutils-dbgsym

```
root@backend-shared-stag-0:~# apt install coreutils=8.25-2ubuntu2
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following packages will be DOWNGRADED:
  coreutils
0 upgraded, 0 newly installed, 1 downgraded, 0 to remove and 1 not upgraded.
Need to get 1175 kB of archives.
After this operation, 16.4 kB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://cn-north-1a.clouds.archive.ubuntu.com/ubuntu xenial/main amd64 coreutils amd64 8.25-2ubuntu2 [1175 kB]
Fetched 1175 kB in 2s (431 kB/s)
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
dpkg: warning: downgrading coreutils from 8.25-2ubuntu3~16.04 to 8.25-2ubuntu2
(Reading database ... 100444 files and directories currently installed.)
Preparing to unpack .../coreutils_8.25-2ubuntu2_amd64.deb ...
Unpacking coreutils (8.25-2ubuntu2) over (8.25-2ubuntu3~16.04) ...
Processing triggers for install-info (6.1.0.dfsg.1-5) ...
Processing triggers for man-db (2.7.5-1) ...
Setting up coreutils (8.25-2ubuntu2) ...
root@backend-shared-stag-0:~#
```

- 重新安装 coreutils-dbgsym ，成功

```
root@backend-shared-stag-0:~# apt-get install coreutils-dbgsym
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  coreutils-dbgsym
0 upgraded, 1 newly installed, 0 to remove and 2 not upgraded.
Need to get 2040 kB of archives.
After this operation, 20.0 MB of additional disk space will be used.
WARNING: The following packages cannot be authenticated!
  coreutils-dbgsym
Install these packages without verification? [y/N] y
Get:1 http://ddebs.ubuntu.com xenial/main amd64 coreutils-dbgsym amd64 8.25-2ubuntu2 [2040 kB]
Fetched 2040 kB in 3s (651 kB/s)
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "zh_CN.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to a fallback locale ("en_US.UTF-8").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
Selecting previously unselected package coreutils-dbgsym.
(Reading database ... 100444 files and directories currently installed.)
Preparing to unpack .../coreutils-dbgsym_8.25-2ubuntu2_amd64.ddeb ...
Unpacking coreutils-dbgsym (8.25-2ubuntu2) ...
Setting up coreutils-dbgsym (8.25-2ubuntu2) ...
root@backend-shared-stag-0:~#
```
