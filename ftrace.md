# ftrace

以下内容取自《[Documentation/ftrace.txt](https://lwn.net/Articles/322731/)》

> `Ftrace` is an internal tracer designed to help out developers and designers of systems to find what is going on inside the kernel. It can be used **for debugging or analyzing latencies and performance issues** that take place outside of user-space.
>
> Although `ftrace` is the function tracer, it also includes an infrastructure that allows for other types of tracing. Some of the tracers that are currently in `ftrace` include a tracer to trace context switches, the time it takes for a high priority task to run after it was woken up, the time interrupts are disabled, and more (`ftrace` allows for tracer plugins, which means that the list of tracers can always grow).

这个文档内容最全面，可以作为参考资料使用；

其他：

- [Debugging the kernel using Ftrace - part 1](https://lwn.net/Articles/365835/)
- [Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)




----------



在《[Ftrace: The hidden light switch](https://lwn.net/Articles/608497/)》中进行了 ftrace 的 case study 和总结；



----------


以下内容取自《[宋宝华：关于Ftrace的一个完整案例](https://mp.weixin.qq.com/s?__biz=MzAwMDUwNDgxOA==&mid=2652663542&idx=1&sn=1e19be71d650eba288b0341d09e164df&scene=21#wechat_redirect)》

> `Ftrace` 是 Linux 进行代码级实践分析最有效的工具之一，比如我们进行一个系统调用，出来的时间过长，我们想知道时间花哪里去了，利用 `Ftrace` 就可以追踪到一级级的时间分布；
>
> 最后，https://github.com/brendangregg/perf-tools 对 `Ftrace` 的功能进行了很好的封装和集成，建议大家用 `perf-tools` 来使用 `Ftrace` ，则效果更佳更简单。

此文给出一个完整使用示例；


----------


以下内容取自《[动态追踪技术专题](https://riboseyim.github.io/2016/11/26/DTrace/)》

> ftrace 是内核 hacker 的最爱。已经包含在内核，能够支持 `tracepoints`、`kprobes` 和 `uprobes`，并提供一些能力：事件追踪、可选择过滤器和参数、事件计数和时间采样、内核概览、基于函数的路径追踪等；

![](http://og2061b3n.bkt.clouddn.com/DTrace_Linux_Choose.png)

![](http://og2061b3n.bkt.clouddn.com/DTrace_Linux_Types.png)

以下内容取自《[动态追踪技术(三) ：Tracing your kernel Functions!](https://riboseyim.github.io/2017/04/17/DTrace_FTrace/)》

> `Ftrace` 是一个设计用来帮助开发者和设计者监视内核的追踪器，可**用于调试或分析延迟以及性能问题**。ftrace 令人印象最深刻的是作为一个 function tracer ，内核函数调用、耗时等情况一览无余。另外，ftrace 最常见的用途是事件追踪，通过内核中成百上千的静态事件点，看到系统内核的哪些部分在运行。实际上，ftrace 更是一个追踪框架，它具备丰富工具集：延迟跟踪检查、何时发生中断、任务的启用、禁用及抢占等。在 ftrace 的基线版本之上，还有很多第三方提供的开源工具，用于简化操作或者提供数据可视化等扩展应用。
>
> `Ftrace` 由两大组成部分：framework 和一系列的 tracer 。每个 tracer 完成不同的功能，它们统一由 framework 管理。`ftrace` 的 trace 信息保存在 ring buffer 中，由 framework 负责管理。Framework 利用 `debugfs` 建立 tracing 目录，并提供了一系列的控制文件。
>
> `ftrace` is a dynamic tracing system. 当你开始 "ftracing" 一个内核函数的时候，该函数的代码实际上就已经发生变化了。内核将在程序集中插入一些额外的指令，使得函数调用时可以随时通知追踪程序。
>
> WARNNING: 使用 `ftrace` 追踪内核将有可能对系统性能产生影响，追踪的函数越多，开销越大。使用者必须提前做好准备工作，生产环境必须谨慎使用。

![](http://og2061b3n.bkt.clouddn.com/DTrace_ftrace_tracers.png)

![](http://og2061b3n.bkt.clouddn.com/DTrace_ftrace_arch.png)

其他

- [rostedt/trace-cmd](https://github.com/rostedt/trace-cmd) -- utilities for Linux ftrace
- [brendangregg/perf-tools](https://github.com/brendangregg/perf-tools) -- Performance analysis tools based on Linux perf_events (aka perf) and ftrace
- KernelShark
    - [Using KernelShark to analyze the real-time scheduler](https://lwn.net/Articles/425583/)
    - [kernelshark](http://rostedt.homelinux.com/kernelshark/)
    - [ELC2011 : Ftrace GUI ( KernelShark)](https://www.youtube.com/watch?v=ABRtzVtUVBo)
    - [KernelShark (quick tutorial)](https://elinux.org/images/6/64/Elc2011_rostedt.pdf)


----------


以下内容取自《[How to surprise by being a Linux-performance “Know-it-all”](https://share.confex.com/share/123/webprogram/Handout/Session15754/Tools-V6-ShareDesign.pdf)》

- Characteristics: Complex interface, but a vast source of information
- Objective: In **kernel latency** and **activity insights**
- Usage: Access `debugfs` mount point `/tracing`
- Package: n/a (**Kernel interface**)


Shows

- Timestamp and activity name
- `Tracepoints` can provide event specific context data
- Infrastructure adds extra common context data like cpu, preempts depth, ...

Hints

- Very powerful and customizable, there are hundreds of tracepoints
    - Some tracepoints have tools to be accessed, both `perf sched` and `blktrace` base on them
    - Others need custom postprocessing
- There are much more things you can handle with tracepoints, check out
    - Kernel Documentation/**trace/tracepoint-analysis.txt** (via `perf stat`)
    - Kernel Documentation/**trace/events.txt** (custom access)


- (Simplified) Script
    - full versions tunes buffer sizes, checks files, ...

```
echo latency-format > /sys/kernel/debug/tracing/trace_options
echo net:* >> /sys/kernel/debug/tracing/set_event
echo napi:* >> /sys/kernel/debug/tracing/set_event
echo "name == ${dev}" > /sys/kernel/debug/tracing/events/net/filter
echo "dev_name == ${dev}" > /sys/kernel/debug/tracing/events/napi/filter # “
cat /sys/kernel/debug/tracing/trace >> ${output} echo !*:* > /sys/kernel/debug/tracing/set_event
```

- Output

```
#                 _------=> CPU#
#               / _-----=> irqs-off
#              | / _----=> need-resched
#              || / _---=> hardirq/softirq
#              ||| / _--=> preempt-depth
#              |||| /     delay
#  cmd    pid  ||||| time | caller
#     \   /    |||||  \    |   /
   <...>-24116 0..s. 486183281us+: net_dev_xmit: dev=eth5 skbaddr=0000000075b7e3e8 len=67 rc=0 
   <idle>-0 0..s. 486183303us+: netif_receive_skb: dev=eth5 skbaddr=000000007ecc6e00 len=53 
   <idle>-0 0.Ns. 486183306us+: napi_poll: napi poll on napi struct 000000007d2479a8 fordevice eth
   <...>-24116 0..s. 486183311us+: net_dev_queue: dev=eth5 skbaddr=0000000075b7e3e8 len=67
   <...>-24116 0..s. 486183317us+: net_dev_xmit: dev=eth5 skbaddr=0000000075b7e3e8 len=67 rc=0
```


----------


## 概述

- ftrace 通过一个循环队列跟踪内核的执行过程。这个循环队列在内存中，大小是固定的（可以动态设置），所以写入的速度可以很快，在没有 ftrace 的时候，我经常通过类似的方式人工跟踪系统的执行过程，以便定位调度引起的各种问题。调度的问题对执行时间非常敏感，所以进行跟踪需要尽量避免把IO，等待等各种额外的要素加进来。而直接写内存就成为影响最小的一种模式了，ftrace 很好地满足了这个要求；
- ftrace 通过 debugfs 对外提供接口（可以通过 mount 命令查看），所以不需要额外的工具进行支持。ftrace 在内核中的配置选项是 `CONFIG_FTRACE` （可以通过 grep CONFIG_FTRACE /boot/config-`uname -r` 查看），一般的发行版都会开启这个特性，所以大部分情况下你也不需要为了使用这个功能重新编译内核；
- debugfs 在大部分发行版中都 mount 在 `/sys/kernel/debug` 目录下，而 ftrace 就在这个目录下的 tracing 目录中；
- 一个用来跟踪的缓冲区（内存）称为一个 `instance` ，缓冲区的大小由文件 `buffer_size_kb` 和 `buffer_total_size_kb` 文件指定。有了缓冲区，你就可以启动行为跟踪，跟踪的结果会分 CPU 写到缓冲区中。缓冲区的数据可以通过 `trace` 和 `trace_pipe` 两个接口（文件）读出。前者通常用于事后读，后者是个 pipe ，可以让你动态读。为了不影响执行过程，我更推荐前一个接口；
- trace 等文件的输出是综合所有 CPU 的，如果你关心单个 CPU 可以进入 per_cpu 目录，里面有这些文件的分 CPU 版本；
- `/sys/kernel/debug/tracing` 这个目录本身就代表一个 instance ，如果你需要更多的 instance ，你可以进入到这个目录下面的 instances 目录中，创建一个任意名字的目录；通过多 instance ，你可以隔离多个独立的跟踪任务；
- ftrace 有两种主要跟踪机制可以往缓冲区中写数据，一种是**函数**，一种是**事件**。前者比较酷，后者才比较可靠实用；事件是固定插入到内核中的跟踪点，例如 Linux 代码中经常看到这种 `trace_` 开头的函数调用，这个地方就是一个事件，也就是打在程序中的一个桩，如果你使能这个桩，程序执行到这个地方就会把这个点（就是一个整数，而不是函数名），加上后面的三个参数（preempt, prev, next) 都写到缓冲区中。到后面你要输出的时候，它会用一个匹配的解释函数来把内容解释出来；
- 事件跟踪的另一个更强大的功能是可以设定跟踪条件，要做这种精细化的设置，你需要直接操作 events 目录下面的事件参数；
- 预定义`功能跟踪`：`事件跟踪`需要根据我们对Kernel业务流有清晰的认识，我们才能合理设置事件。`功能跟踪`就会简单得多，功能跟踪可以直接使能某种跟踪功能，具体用什么事件，设置什么参数等，都默认设置好，这种预定义功能在 `available_tracers` 中列出，只要选择其中一个，把对应的名字写入 `current_tracer` 文件中就可以启动这个功能；
- `函数跟踪`和事件跟踪一样，相当于在函数入口那里增加了一个 `trace_` 函数；还可以进一步加`堆栈跟踪`；
- ftrace 一个比较明显的缺点是**没有用户态的跟踪点支持**，作为补救，instance 中提供了一个文件，`trace_marker`，写这个文件可以在跟踪中产生一条记录；这个跟踪方法的缺点是需要额外的系统调用，没有内核跟踪那么高效，但聊胜于无；
- 使用 ftrace 的另一个缺点是它`会话跟踪`能力比较差；
- ftrace 还可以通过 `uprobes`/`kprobes` 设置跟踪点；
- ftrace 主要用于跟踪**时延**和**调度行为**（可用于优化程序的 CPU 使用）；
- 如果真的发生 sock 队列的丢包，ftrace 的 `sock:*` 事件可以具体跟踪到这个事件；
- 在 io 层自己的调度上，可以用 ftrace 的 blk tracer 来跟。`echo blk > current_tracer` 中就可以实施专门针对块设备层的跟踪。这个跟踪器的控制是放在每个块设备上的，你需要到 `/sys/block/<块设备>/trace/` 下面对这个块设备的跟踪进行支持（比如 `echo 1 > /sys/block/xvda/trace/enable`）
- 从 Host 一侧跟踪 KVM 的行为，可以跟踪到进入和离开 guest 的时间，这个可以通过跟踪 ftrace 的 `kvm:*` 事件得到；
- 通过 ftrace 跟踪 `sched:sched_migrate_task` 可以跟踪任务转移的频度；

## ftrace v.s. perf

- ftrace 的跟踪方法是一种**总体跟踪法**，换句话说，你统计了一个事件到下一个事件所有的时间长度，然后把它们放到时间轴上，你可以知道整个系统运行在时间轴上的分布；这种方法很准确，但跟踪成本很高。所以，我们也需要一种**抽样形态的跟踪方法**。perf 提供的就是这样的跟踪方法；
- perf 比起 ftrace 来说，最大的好处是它可以直接跟踪到整个系统的所有程序（而不仅仅是内核），所以 **perf 通常是我们分析的第一步**，我们先看到整个系统的 outline ，然后才会进去看具体的调度、时延等问题。而且 perf 本身也告诉你调度是否正常了，比如内核调度子系统的函数占用率特别高，我们可能就知道我们需要分析一下调度过程了；
- perf 的统计不能用来让你分析 CPU 占用率的。ftrace 和 top 等工具才能看 CPU 占用率，perf 是不行的；


## 场景

对于性能分析，我用得最多的是这个线程 switch 事件（还有 softirq 的一组事件）。因为**从考量吞吐量的角度，主业务 CPU 要不 idle ，要不在处理业务，要不在调度**。一个“不折腾”的系统，主业务进程应该每次都用完自己的时间片，如果它总用不完，要不是它实时性要求很高（主业务这种情况很少），要不是线程调度设计有问题。我们常常看到的一种模型是，由于业务在线程上安排不合理，导致一个线程刚执行一步，马上要等下一个线程完成，那个线程又执行一步，又要回来等前一个线程完成，这样 CPU 的时间都在切换上，整个吞吐量就很低了；

当我们发现比如 schedule 调度特别频繁的时候，我们可以通过 ftrace 观察每次切换的原因；


## 启动事件跟踪的方法

- 先查 `available_events` 文件中有哪些可以用的事件（查 events 目录也可以，但不是很方便）；
- 把那个事件的名称写进 `set_event` 文件，可以写多个，可以写 `sched:*` 这样的通配符（或者直接指定 `sched:sched_switch`）；
- 通过 `tracing_on` 文件启动跟踪（`echo 1 > tracing_on`）；启动之前可以通过比如 `tracing_cpumask` 这样的文件限制跟踪的 CPU ，通过 `set_event_pid` 设置跟踪的 pid ，或者通过其他属性进行更深入的设定；

对比 perf 的使用方式

```
perf top -e sched:sched_switch -s pid
```

## 操作

```
# 清空对应的缓冲区
echo > trace

# 启动跟踪
echo 1 > tracing_on

# 停止跟踪
echo 0 > tracing_on

# 限制只根据某个 pid 进行事件跟踪
echo <pid> > set_ftrace_pid

# 跟踪系统唤醒的时延
# 可以跟踪触发跟踪的期间里，最高优先级任务的调度最大时延
# 即任务被唤醒之后，执行了哪些动作，才轮到它再次执行
# 可用于参考优化哪些流程来提升整个系统的实时性
echo wakeup > current_tracer
echo 1 > tracing_on

# 切回默认
echo nop > current_tracer

# 仅跟踪调度器切换到 xxx 这个线程的场景
echo 'next_comm ~ "xxx"' > events/sched/sched_switch/filter


```

## README 文件

```
root@backend-shared-stag-0:/sys/kernel/debug/tracing# cat README
tracing mini-HOWTO:

# echo 0 > tracing_on : quick way to disable tracing
# echo 1 > tracing_on : quick way to re-enable tracing

 Important files:
  trace			- The static contents of the buffer
			  To clear the buffer write into this file: echo > trace
  trace_pipe		- A consuming read to see the contents of the buffer
  current_tracer	- function and latency tracers
  available_tracers	- list of configured tracers for current_tracer
  buffer_size_kb	- view and modify size of per cpu buffer
  buffer_total_size_kb  - view total size of all cpu buffers

  trace_clock		-change the clock used to order events
       local:   Per cpu clock but may not be synced across CPUs
      global:   Synced across CPUs but slows tracing down.
     counter:   Not a clock, but just an increment
      uptime:   Jiffy counter from time of boot
        perf:   Same clock that perf events use
     x86-tsc:   TSC cycle counter

  trace_marker		- Writes into this file writes into the kernel buffer
  tracing_cpumask	- Limit which CPUs to trace
  instances		- Make sub-buffers with: mkdir instances/foo
			  Remove sub-buffer with rmdir
  trace_options		- Set format or modify how tracing happens
			  Disable an option by adding a suffix 'no' to the
			  option name
  saved_cmdlines_size	- echo command number in here to store comm-pid list

  available_filter_functions - list of functions that can be filtered on
  set_ftrace_filter	- echo function name in here to only trace these
			  functions
	     accepts: func_full_name, *func_end, func_begin*, *func_middle*
	     modules: Can select a group via module
	      Format: :mod:<module-name>
	     example: echo :mod:ext3 > set_ftrace_filter
	    triggers: a command to perform when function is hit
	      Format: <function>:<trigger>[:count]
	     trigger: traceon, traceoff
		      enable_event:<system>:<event>
		      disable_event:<system>:<event>
		      stacktrace
		      snapshot
		      dump
		      cpudump
	     example: echo do_fault:traceoff > set_ftrace_filter
	              echo do_trap:traceoff:3 > set_ftrace_filter
	     The first one will disable tracing every time do_fault is hit
	     The second will disable tracing at most 3 times when do_trap is hit
	       The first time do trap is hit and it disables tracing, the
	       counter will decrement to 2. If tracing is already disabled,
	       the counter will not decrement. It only decrements when the
	       trigger did work
	     To remove trigger without count:
	       echo '!<function>:<trigger> > set_ftrace_filter
	     To remove trigger with a count:
	       echo '!<function>:<trigger>:0 > set_ftrace_filter
  set_ftrace_notrace	- echo function name in here to never trace.
	    accepts: func_full_name, *func_end, func_begin*, *func_middle*
	    modules: Can select a group via module command :mod:
	    Does not accept triggers
  set_ftrace_pid	- Write pid(s) to only function trace those pids
		    (function)
  set_graph_function	- Trace the nested calls of a function (function_graph)
  set_graph_notrace	- Do not trace the nested calls of a function (function_graph)
  max_graph_depth	- Trace a limited depth of nested calls (0 is unlimited)

  snapshot		- Like 'trace' but shows the content of the static
			  snapshot buffer. Read the contents for more
			  information
  stack_trace		- Shows the max stack trace when active
  stack_max_size	- Shows current max stack size that was traced
			  Write into this file to reset the max size (trigger a
			  new trace)
  stack_trace_filter	- Like set_ftrace_filter but limits what stack_trace
			  traces
  events/		- Directory containing all trace event subsystems:
      enable		- Write 0/1 to enable/disable tracing of all events
  events/<system>/	- Directory containing all trace events for <system>:
      enable		- Write 0/1 to enable/disable tracing of all <system>
			  events
      filter		- If set, only events passing filter are traced
  events/<system>/<event>/	- Directory containing control files for
			  <event>:
      enable		- Write 0/1 to enable/disable tracing of <event>
      filter		- If set, only events passing filter are traced
      trigger		- If set, a command to perform when event is hit
	    Format: <trigger>[:count][if <filter>]
	   trigger: traceon, traceoff
	            enable_event:<system>:<event>
	            disable_event:<system>:<event>
		    stacktrace
		    snapshot
	   example: echo traceoff > events/block/block_unplug/trigger
	            echo traceoff:3 > events/block/block_unplug/trigger
	            echo 'enable_event:kmem:kmalloc:3 if nr_rq > 1' > \
	                  events/block/block_unplug/trigger
	   The first disables tracing every time block_unplug is hit.
	   The second disables tracing the first 3 times block_unplug is hit.
	   The third enables the kmalloc event the first 3 times block_unplug
	     is hit and has value of greater than 1 for the 'nr_rq' event field.
	   Like function triggers, the counter is only decremented if it
	    enabled or disabled tracing.
	   To remove a trigger without a count:
	     echo '!<trigger> > <system>/<event>/trigger
	   To remove a trigger with a count:
	     echo '!<trigger>:0 > <system>/<event>/trigger
	   Filters can be ignored when removing a trigger.
root@backend-shared-stag-0:/sys/kernel/debug/tracing#
```

## /sys/kernel/debug/tracing 目录结构

```
root@backend-shared-stag-0:/sys/kernel/debug/tracing# ll
total 0
drwx------  7 root root 0 Jan  8 13:20 ./
drwx------ 26 root root 0 Apr 26 19:25 ../
-r--r--r--  1 root root 0 Jan  8 13:20 available_events
-r--r--r--  1 root root 0 Jan  8 13:20 available_filter_functions
-r--r--r--  1 root root 0 Jan  8 13:20 available_tracers
-rw-r--r--  1 root root 0 Jan  8 13:20 buffer_size_kb
-r--r--r--  1 root root 0 Jan  8 13:20 buffer_total_size_kb
-rw-r--r--  1 root root 0 May  9 14:34 current_tracer
-r--r--r--  1 root root 0 Jan  8 13:20 dyn_ftrace_total_info
-r--r--r--  1 root root 0 Jan  8 13:20 enabled_functions
drwxr-xr-x 67 root root 0 May  3 19:20 events/
--w-------  1 root root 0 Jan  8 13:20 free_buffer
-rw-r--r--  1 root root 0 Jan  8 13:20 function_profile_enabled
drwxr-xr-x  2 root root 0 Jan  8 13:20 instances/
-rw-r--r--  1 root root 0 Jan  8 13:20 kprobe_events
-r--r--r--  1 root root 0 Jan  8 13:20 kprobe_profile
-rw-r--r--  1 root root 0 Jan  8 13:20 max_graph_depth
drwxr-xr-x  2 root root 0 Jan  8 13:20 options/
drwxr-xr-x 17 root root 0 Jan  8 13:20 per_cpu/
-r--r--r--  1 root root 0 Jan  8 13:20 printk_formats
-r--r--r--  1 root root 0 Jan  8 13:20 README
-r--r--r--  1 root root 0 Jan  8 13:20 saved_cmdlines
-rw-r--r--  1 root root 0 Jan  8 13:20 saved_cmdlines_size
-rw-r--r--  1 root root 0 May  9 13:52 set_event
-rw-r--r--  1 root root 0 Jan  8 13:20 set_event_pid
-rw-r--r--  1 root root 0 Jan  8 13:20 set_ftrace_filter
-rw-r--r--  1 root root 0 Jan  8 13:20 set_ftrace_notrace
-rw-r--r--  1 root root 0 Jan  8 13:20 set_ftrace_pid
-r--r--r--  1 root root 0 Jan  8 13:20 set_graph_function
-r--r--r--  1 root root 0 Jan  8 13:20 set_graph_notrace
-rw-r--r--  1 root root 0 Jan  8 13:20 snapshot
-rw-r--r--  1 root root 0 Jan  8 13:20 stack_max_size
-r--r--r--  1 root root 0 Jan  8 13:20 stack_trace
-r--r--r--  1 root root 0 Jan  8 13:20 stack_trace_filter
-rw-r--r--  1 root root 0 May  9 14:34 trace
-rw-r--r--  1 root root 0 Jan  8 13:20 trace_clock
--w--w----  1 root root 0 Jan  8 13:20 trace_marker
-rw-r--r--  1 root root 0 Jan  8 13:20 trace_options
-r--r--r--  1 root root 0 Jan  8 13:20 trace_pipe
drwxr-xr-x  2 root root 0 Jan  8 13:20 trace_stat/
-rw-r--r--  1 root root 0 Jan  8 13:20 tracing_cpumask
-rw-r--r--  1 root root 0 Jan  8 13:20 tracing_max_latency
-rw-r--r--  1 root root 0 May  9 14:34 tracing_on
-rw-r--r--  1 root root 0 Jan  8 13:20 tracing_thresh
-rw-r--r--  1 root root 0 Jan  8 13:20 uprobe_events
-r--r--r--  1 root root 0 Jan  8 13:20 uprobe_profile
root@backend-shared-stag-0:/sys/kernel/debug/tracing#
```


参考：

- [在Linux下做性能分析1：基本模型](https://zhuanlan.zhihu.com/p/22124514)
- [在Linux下做性能分析2：ftrace](https://zhuanlan.zhihu.com/p/22130013)
- [在Linux下做性能分析3：perf](https://zhuanlan.zhihu.com/p/22194920)



