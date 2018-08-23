# perf_events cheatsheet


![](https://jvns.ca/images/perf-cheat-sheet.png)

PDF 下载：[这里](https://jvns.ca/perf-cheat-sheet.pdf)


----------


## perf top: get updates live!

- Sample CPUs at 49 Hertz, show top symbols

> 按照 Shared Object -> Symbol 两级维度查看

```
perf top -F 49
```

输出

```
Samples: 15  of event 'cpu-clock', Event count (approx.): 136839704
Overhead  Shared Object     Symbol
  34.26%  node_exporter     [.] runtime.scanobject
  11.42%  node_exporter     [.] runtime.(*gcWork).put
  11.42%  node_exporter     [.] runtime.heapBitsForObject
  11.42%  node_exporter     [.] runtime.readgstatus
   7.85%  node_exporter     [.] runtime.mallocgc
   3.92%  [kernel]          [k] format_decode
   3.92%  [kernel]          [k] kmem_cache_alloc_trace
   3.92%  node_exporter     [.] runtime.heapBitsSetType
   3.92%  node_exporter     [.] runtime/internal/atomic.Xchg
   3.92%  node_exporter     [.] strconv.ParseFloat
   2.01%  perf              [.] 0x000000000008e878
   2.01%  perf              [.] 0x00000000000e6446

```

- Sample CPUs, show top process names and segments

> 按 Command -> Shared Object 两级维度查看

```
perf top -ns comm,dso
```

输出

```
Samples: 391  of event 'cpu-clock', Event count (approx.): 36435580
Overhead       Samples  Command          Shared Object
  32.41%            45  node_exporter    node_exporter
  28.57%            40  perf             perf
  15.72%            20  perf             libc-2.23.so
   7.31%             8  perf             libbfd-2.26.1-system.so
   7.02%             8  perf             [kernel]
   3.26%             1  perf             libpthread-2.23.so
   2.14%             0  perf             libelf-0.165.so
   0.84%             0  docker-containe  docker-containerd
   0.71%             0  node_exporter    [kernel]
   0.64%             0  dockerd          [kernel]
   0.54%             0  dockerd          dockerd
   0.53%             0  swapper          [kernel]
   0.31%             0  haproxy          libc-2.23.so (deleted)

```

- Count system calls by process, refreshing every 1 second

> 按 Command 查看系统调用数量分布情况

```
perf top -e raw_syscalls:sys_enter -ns comm -d 1
```

输出

```
Samples: 5K of event 'raw_syscalls:sys_enter', Event count (approx.): 2316
Overhead       Samples  Command
  39.29%           910  dockerd
  24.35%           564  node_exporter
  21.76%           504  docker-containe
   3.84%            89  perf
   3.76%            87  haproxy
   3.76%            87  sshd
   1.42%            33  iscsid
   0.82%            19  irqbalance
   0.73%            17  ntpd
   0.26%             6  gmain


```

- Count sent network packets by process, rolling output

> 按进程统计程序发包数量，滚动输出

```
stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings
```

输出

```
root@backend-shared-stag-0:~# stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
   100.00%             2  node_exporter
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    50.00%             2  haproxy
    25.00%             1  node_exporter
    25.00%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    75.00%             3  haproxy
    25.00%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    80.00%             4  haproxy
    20.00%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    83.33%             5  haproxy
    16.67%             1  sshd
   PerfTop:       2 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    75.00%             6  haproxy
    12.50%             1  sshd
    12.50%             1  swapper
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    87.50%             7  haproxy
    12.50%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    88.89%             8  haproxy
    11.11%             1  sshd
   PerfTop:       2 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    75.00%             9  haproxy
    16.67%             2  node_exporter
     8.33%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    81.82%             9  haproxy
     9.09%             1  node_exporter
     9.09%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    90.00%             9  haproxy
    10.00%             1  sshd
   PerfTop:       1 irqs/sec  kernel:100.0%  exact:  0.0% [1Hz net:net_dev_xmit],  (all, 8 CPUs)
-------------------------------------------------------------------------------
    90.00%             9  haproxy
    10.00%             1  sshd

root@backend-shared-stag-0:~#
```

## perf stat: count events! CPU counters!

- CPU counter statistics for COMMAND

> 分析具体命令执行过程中的耗时点（需要关注）

```
perf stat COMMAND
```

输出

```
root@backend-shared-stag-0:~# perf stat cat /proc/meminfo
MemTotal:       32946256 kB
MemFree:        25027304 kB
MemAvailable:   32157996 kB
Buffers:          279620 kB
Cached:          6942188 kB
SwapCached:            0 kB
Active:          4361292 kB
Inactive:        2961900 kB
Active(anon):     168656 kB
Inactive(anon):    66912 kB
Active(file):    4192636 kB
Inactive(file):  2894988 kB
Unevictable:        3652 kB
Mlocked:            3652 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        105032 kB
Mapped:            97404 kB
Shmem:            131764 kB
Slab:             483588 kB
SReclaimable:     442888 kB
SUnreclaim:        40700 kB
KernelStack:        4288 kB
PageTables:         5052 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     828320 kB
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
DirectMap4k:      161792 kB
DirectMap2M:     9275392 kB
DirectMap1G:    25165824 kB

 Performance counter stats for 'cat /proc/meminfo':

          0.363383      task-clock (msec)         #    0.593 CPUs utilized
                 0      context-switches          #    0.000 K/sec
                 0      cpu-migrations            #    0.000 K/sec
                54      page-faults               #    0.149 M/sec
   <not supported>      cycles
   <not supported>      stalled-cycles-frontend
   <not supported>      stalled-cycles-backend
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

       0.000612738 seconds time elapsed

root@backend-shared-stag-0:~#
```

- Detailed CPU counter statistics for COMMAND

```
perf stat -ddd COMMAND
```

输出

```
root@backend-shared-stag-0:~# perf stat -ddd cat /proc/meminfo
MemTotal:       32946256 kB
MemFree:        25026824 kB
MemAvailable:   32157516 kB
Buffers:          279620 kB
Cached:          6942188 kB
SwapCached:            0 kB
Active:          4362328 kB
Inactive:        2961900 kB
Active(anon):     169692 kB
Inactive(anon):    66912 kB
Active(file):    4192636 kB
Inactive(file):  2894988 kB
Unevictable:        3652 kB
Mlocked:            3652 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        106092 kB
Mapped:            96968 kB
Shmem:            131764 kB
Slab:             483604 kB
SReclaimable:     442888 kB
SUnreclaim:        40716 kB
KernelStack:        4272 kB
PageTables:         5052 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    16473128 kB
Committed_AS:     810240 kB
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
DirectMap4k:      161792 kB
DirectMap2M:     9275392 kB
DirectMap1G:    25165824 kB

 Performance counter stats for 'cat /proc/meminfo':

          0.343147      task-clock (msec)         #    0.583 CPUs utilized
                 0      context-switches          #    0.000 K/sec
                 0      cpu-migrations            #    0.000 K/sec
                55      page-faults               #    0.160 M/sec
   <not supported>      cycles
   <not supported>      stalled-cycles-frontend
   <not supported>      stalled-cycles-backend
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses
   <not supported>      L1-dcache-loads
   <not supported>      L1-dcache-load-misses
   <not supported>      LLC-loads
   <not supported>      LLC-load-misses
   <not supported>      L1-icache-loads
   <not supported>      L1-icache-load-misses
   <not supported>      dTLB-loads
   <not supported>      dTLB-load-misses
   <not supported>      iTLB-loads
   <not supported>      iTLB-load-misses
   <not supported>      L1-dcache-prefetches
   <not supported>      L1-dcache-prefetch-misses

       0.000588812 seconds time elapsed

root@backend-shared-stag-0:~#
```

- Various basic CPU statistics, system wide

> 只针对 `-e` 指定的项目进行查看

```
perf stat -e cycles,instructions,cache-misses -a
```

输出

```
root@backend-shared-stag-0:~# perf stat -e cycles,instructions,cache-misses -a
^C
 Performance counter stats for 'system wide':

   <not supported>      cycles
   <not supported>      instructions
   <not supported>      cache-misses

      14.661660529 seconds time elapsed


root@backend-shared-stag-0:~#
```

- Count system calls for PID, until Ctrl-C

> 查看指定进程的系统调用数量

```
perf stat -e 'syscalls:sys_enter_*' -p PID
```

输出

```
root@backend-shared-stag-0:~# perf stat -e 'syscalls:sys_enter_*' -p 8083
^C

 Performance counter stats for process id '8083':

                14      syscalls:sys_enter_socket
                 0      syscalls:sys_enter_socketpair
                 0      syscalls:sys_enter_bind
                 0      syscalls:sys_enter_listen
                 0      syscalls:sys_enter_accept4
                 0      syscalls:sys_enter_accept
                42      syscalls:sys_enter_connect
                 0      syscalls:sys_enter_getsockname
                 0      syscalls:sys_enter_getpeername
                 0      syscalls:sys_enter_sendto
                14      syscalls:sys_enter_recvfrom
                42      syscalls:sys_enter_setsockopt
                 0      syscalls:sys_enter_getsockopt
                 0      syscalls:sys_enter_shutdown
                 0      syscalls:sys_enter_sendmsg
                 0      syscalls:sys_enter_sendmmsg
                 0      syscalls:sys_enter_recvmsg
                 0      syscalls:sys_enter_recvmmsg
                 0      syscalls:sys_enter_getrandom
                 0      syscalls:sys_enter_ioprio_set
                 0      syscalls:sys_enter_ioprio_get
                 0      syscalls:sys_enter_add_key
                 0      syscalls:sys_enter_request_key
                 0      syscalls:sys_enter_keyctl
                 0      syscalls:sys_enter_mq_open
                 0      syscalls:sys_enter_mq_unlink
                 0      syscalls:sys_enter_mq_timedsend
                 0      syscalls:sys_enter_mq_timedreceive
                 0      syscalls:sys_enter_mq_notify
                 0      syscalls:sys_enter_mq_getsetattr
                 0      syscalls:sys_enter_shmget
                 0      syscalls:sys_enter_shmctl
                 0      syscalls:sys_enter_shmat
                 0      syscalls:sys_enter_shmdt
                 0      syscalls:sys_enter_semget
                 0      syscalls:sys_enter_semctl
                 0      syscalls:sys_enter_semtimedop
                 0      syscalls:sys_enter_semop
                 0      syscalls:sys_enter_msgget
                 0      syscalls:sys_enter_msgctl
                 0      syscalls:sys_enter_msgsnd
                 0      syscalls:sys_enter_msgrcv
                 0      syscalls:sys_enter_lookup_dcookie
                 0      syscalls:sys_enter_quotactl
                 0      syscalls:sys_enter_name_to_handle_at
                 0      syscalls:sys_enter_open_by_handle_at
                 0      syscalls:sys_enter_flock
                 0      syscalls:sys_enter_io_setup
                 0      syscalls:sys_enter_io_destroy
                 0      syscalls:sys_enter_io_submit
                 0      syscalls:sys_enter_io_cancel
                 0      syscalls:sys_enter_io_getevents
                 0      syscalls:sys_enter_userfaultfd
                 0      syscalls:sys_enter_eventfd2
                 0      syscalls:sys_enter_eventfd
                 0      syscalls:sys_enter_timerfd_create
                 0      syscalls:sys_enter_timerfd_settime
                 0      syscalls:sys_enter_timerfd_gettime
                 0      syscalls:sys_enter_signalfd4
                 0      syscalls:sys_enter_signalfd
                 0      syscalls:sys_enter_epoll_create1
                 0      syscalls:sys_enter_epoll_create
                14      syscalls:sys_enter_epoll_ctl
                70      syscalls:sys_enter_epoll_wait
                 0      syscalls:sys_enter_epoll_pwait
                 0      syscalls:sys_enter_fanotify_init
                 0      syscalls:sys_enter_fanotify_mark
                 0      syscalls:sys_enter_inotify_init1
                 0      syscalls:sys_enter_inotify_init
                 0      syscalls:sys_enter_inotify_add_watch
                 0      syscalls:sys_enter_inotify_rm_watch
                 0      syscalls:sys_enter_statfs
                 0      syscalls:sys_enter_fstatfs
                 0      syscalls:sys_enter_ustat
                 0      syscalls:sys_enter_utime
                 0      syscalls:sys_enter_utimensat
                 0      syscalls:sys_enter_futimesat
                 0      syscalls:sys_enter_utimes
                 0      syscalls:sys_enter_sync
                 0      syscalls:sys_enter_syncfs
                 0      syscalls:sys_enter_fsync
                 0      syscalls:sys_enter_fdatasync
                 0      syscalls:sys_enter_sync_file_range
                 0      syscalls:sys_enter_vmsplice
                 0      syscalls:sys_enter_splice
                 0      syscalls:sys_enter_tee
                 0      syscalls:sys_enter_setxattr
                 0      syscalls:sys_enter_lsetxattr
                 0      syscalls:sys_enter_fsetxattr
                 0      syscalls:sys_enter_getxattr
                 0      syscalls:sys_enter_lgetxattr
                 0      syscalls:sys_enter_fgetxattr
                 0      syscalls:sys_enter_listxattr
                 0      syscalls:sys_enter_llistxattr
                 0      syscalls:sys_enter_flistxattr
                 0      syscalls:sys_enter_removexattr
                 0      syscalls:sys_enter_lremovexattr
                 0      syscalls:sys_enter_fremovexattr
                 0      syscalls:sys_enter_umount
                 0      syscalls:sys_enter_mount
                 0      syscalls:sys_enter_pivot_root
                 0      syscalls:sys_enter_sysfs
                 0      syscalls:sys_enter_dup3
                 0      syscalls:sys_enter_dup2
                 0      syscalls:sys_enter_dup
                 0      syscalls:sys_enter_getcwd
                 0      syscalls:sys_enter_select
                 0      syscalls:sys_enter_pselect6
                 0      syscalls:sys_enter_poll
                 0      syscalls:sys_enter_ppoll
                 0      syscalls:sys_enter_getdents
                 0      syscalls:sys_enter_getdents64
                 0      syscalls:sys_enter_ioctl
                14      syscalls:sys_enter_fcntl
                 0      syscalls:sys_enter_mknodat
                 0      syscalls:sys_enter_mknod
                 0      syscalls:sys_enter_mkdirat
                 0      syscalls:sys_enter_mkdir
                 0      syscalls:sys_enter_rmdir
                 0      syscalls:sys_enter_unlinkat
                 0      syscalls:sys_enter_unlink
                 0      syscalls:sys_enter_symlinkat
                 0      syscalls:sys_enter_symlink
                 0      syscalls:sys_enter_linkat
                 0      syscalls:sys_enter_link
                 0      syscalls:sys_enter_renameat2
                 0      syscalls:sys_enter_renameat
                 0      syscalls:sys_enter_rename
                 0      syscalls:sys_enter_pipe2
                 0      syscalls:sys_enter_pipe
                 0      syscalls:sys_enter_newstat
                 0      syscalls:sys_enter_newlstat
                 0      syscalls:sys_enter_newfstatat
                 0      syscalls:sys_enter_newfstat
                 0      syscalls:sys_enter_readlinkat
                 0      syscalls:sys_enter_readlink
                 0      syscalls:sys_enter_lseek
                 0      syscalls:sys_enter_read
                 0      syscalls:sys_enter_write
                 0      syscalls:sys_enter_pread64
                 0      syscalls:sys_enter_pwrite64
                 0      syscalls:sys_enter_readv
                 0      syscalls:sys_enter_writev
                 0      syscalls:sys_enter_preadv
                 0      syscalls:sys_enter_pwritev
                 0      syscalls:sys_enter_sendfile64
                 0      syscalls:sys_enter_truncate
                 0      syscalls:sys_enter_ftruncate
                 0      syscalls:sys_enter_fallocate
                 0      syscalls:sys_enter_faccessat
                 0      syscalls:sys_enter_access
                 0      syscalls:sys_enter_chdir
                 0      syscalls:sys_enter_fchdir
                 0      syscalls:sys_enter_chroot
                 0      syscalls:sys_enter_fchmod
                 0      syscalls:sys_enter_fchmodat
                 0      syscalls:sys_enter_chmod
                 0      syscalls:sys_enter_fchownat
                 0      syscalls:sys_enter_chown
                 0      syscalls:sys_enter_lchown
                 0      syscalls:sys_enter_fchown
                 0      syscalls:sys_enter_open
                 0      syscalls:sys_enter_openat
                 0      syscalls:sys_enter_creat
                15      syscalls:sys_enter_close
                 0      syscalls:sys_enter_vhangup
                 0      syscalls:sys_enter_move_pages
                 0      syscalls:sys_enter_mbind
                 0      syscalls:sys_enter_set_mempolicy
                 0      syscalls:sys_enter_migrate_pages
                 0      syscalls:sys_enter_get_mempolicy
                 0      syscalls:sys_enter_swapoff
                 0      syscalls:sys_enter_swapon
                 0      syscalls:sys_enter_madvise
                 0      syscalls:sys_enter_fadvise64
                 0      syscalls:sys_enter_process_vm_readv
                 0      syscalls:sys_enter_process_vm_writev
                 0      syscalls:sys_enter_msync
                 0      syscalls:sys_enter_mremap
                 0      syscalls:sys_enter_mprotect
                 0      syscalls:sys_enter_brk
                 0      syscalls:sys_enter_munmap
                 0      syscalls:sys_enter_remap_file_pages
                 0      syscalls:sys_enter_mlock
                 0      syscalls:sys_enter_mlock2
                 0      syscalls:sys_enter_munlock
                 0      syscalls:sys_enter_mlockall
                 0      syscalls:sys_enter_munlockall
                 0      syscalls:sys_enter_mincore
                 0      syscalls:sys_enter_memfd_create
                 0      syscalls:sys_enter_readahead
                 0      syscalls:sys_enter_membarrier
                 0      syscalls:sys_enter_perf_event_open
                 0      syscalls:sys_enter_bpf
                 0      syscalls:sys_enter_seccomp
                 0      syscalls:sys_enter_kexec_file_load
                 0      syscalls:sys_enter_kexec_load
                 0      syscalls:sys_enter_acct
                 0      syscalls:sys_enter_delete_module
                 0      syscalls:sys_enter_init_module
                 0      syscalls:sys_enter_finit_module
                 0      syscalls:sys_enter_set_robust_list
                 0      syscalls:sys_enter_get_robust_list
                 0      syscalls:sys_enter_futex
                 0      syscalls:sys_enter_timer_create
                 0      syscalls:sys_enter_timer_gettime
                 0      syscalls:sys_enter_timer_getoverrun
                 0      syscalls:sys_enter_timer_settime
                 0      syscalls:sys_enter_timer_delete
                 0      syscalls:sys_enter_clock_settime
                 0      syscalls:sys_enter_clock_gettime
                 0      syscalls:sys_enter_clock_adjtime
                 0      syscalls:sys_enter_clock_getres
                 0      syscalls:sys_enter_clock_nanosleep
                 0      syscalls:sys_enter_getitimer
                 0      syscalls:sys_enter_setitimer
                 0      syscalls:sys_enter_nanosleep
                 0      syscalls:sys_enter_alarm
                 0      syscalls:sys_enter_time
               150      syscalls:sys_enter_gettimeofday
                 0      syscalls:sys_enter_settimeofday
                 0      syscalls:sys_enter_adjtimex
                 0      syscalls:sys_enter_kcmp
                 0      syscalls:sys_enter_syslog
                 0      syscalls:sys_enter_sched_setscheduler
                 0      syscalls:sys_enter_sched_setparam
                 0      syscalls:sys_enter_sched_setattr
                 0      syscalls:sys_enter_sched_getscheduler
                 0      syscalls:sys_enter_sched_getparam
                 0      syscalls:sys_enter_sched_getattr
                 0      syscalls:sys_enter_sched_setaffinity
                 0      syscalls:sys_enter_sched_getaffinity
                 0      syscalls:sys_enter_sched_yield
                 0      syscalls:sys_enter_sched_get_priority_max
                 0      syscalls:sys_enter_sched_get_priority_min
                 0      syscalls:sys_enter_sched_rr_get_interval
                 0      syscalls:sys_enter_getgroups
                 0      syscalls:sys_enter_setgroups
                 0      syscalls:sys_enter_reboot
                 0      syscalls:sys_enter_setns
                 0      syscalls:sys_enter_setpriority
                 0      syscalls:sys_enter_getpriority
                 0      syscalls:sys_enter_setregid
                 0      syscalls:sys_enter_setgid
                 0      syscalls:sys_enter_setreuid
                 0      syscalls:sys_enter_setuid
                 0      syscalls:sys_enter_setresuid
                 0      syscalls:sys_enter_getresuid
                 0      syscalls:sys_enter_setresgid
                 0      syscalls:sys_enter_getresgid
                 0      syscalls:sys_enter_setfsuid
                 0      syscalls:sys_enter_setfsgid
                 0      syscalls:sys_enter_getpid
                 0      syscalls:sys_enter_gettid
                 0      syscalls:sys_enter_getppid
                 0      syscalls:sys_enter_getuid
                 0      syscalls:sys_enter_geteuid
                 0      syscalls:sys_enter_getgid
                 0      syscalls:sys_enter_getegid
                 0      syscalls:sys_enter_times
                 0      syscalls:sys_enter_setpgid
                 0      syscalls:sys_enter_getpgid
                 0      syscalls:sys_enter_getpgrp
                 0      syscalls:sys_enter_getsid
                 0      syscalls:sys_enter_setsid
                 0      syscalls:sys_enter_newuname
                 0      syscalls:sys_enter_sethostname
                 0      syscalls:sys_enter_setdomainname
                 0      syscalls:sys_enter_getrlimit
                 0      syscalls:sys_enter_prlimit64
                 0      syscalls:sys_enter_setrlimit
                 0      syscalls:sys_enter_getrusage
                 0      syscalls:sys_enter_umask
                 0      syscalls:sys_enter_prctl
                 0      syscalls:sys_enter_getcpu
                 0      syscalls:sys_enter_sysinfo
                 0      syscalls:sys_enter_restart_syscall
                 0      syscalls:sys_enter_rt_sigprocmask
                 0      syscalls:sys_enter_rt_sigpending
                 0      syscalls:sys_enter_rt_sigtimedwait
                 0      syscalls:sys_enter_kill
                 0      syscalls:sys_enter_tgkill
                 0      syscalls:sys_enter_tkill
                 0      syscalls:sys_enter_rt_sigqueueinfo
                 0      syscalls:sys_enter_rt_tgsigqueueinfo
                 0      syscalls:sys_enter_sigaltstack
                 0      syscalls:sys_enter_rt_sigaction
                 0      syscalls:sys_enter_pause
                 0      syscalls:sys_enter_rt_sigsuspend
                 0      syscalls:sys_enter_ptrace
                 0      syscalls:sys_enter_capget
                 0      syscalls:sys_enter_capset
                 0      syscalls:sys_enter_sysctl
                 0      syscalls:sys_enter_exit
                 0      syscalls:sys_enter_exit_group
                 0      syscalls:sys_enter_waitid
                 0      syscalls:sys_enter_wait4
                 0      syscalls:sys_enter_personality
                 0      syscalls:sys_enter_set_tid_address
                 0      syscalls:sys_enter_unshare
                 0      syscalls:sys_enter_mmap
                 0      syscalls:sys_enter_iopl

      27.224890615 seconds time elapsed


root@backend-shared-stag-0:~#
```

- Count block device I/O events for the entire system, for 10 seconds

> 针对块设备的 I/O 事件进行 10 秒统计

```
perf stat -e 'block:*' -a sleep 10
```

输出

```
root@backend-shared-stag-0:~# perf stat -e 'block:*' -a sleep 10

 Performance counter stats for 'system wide':

                 0      block:block_touch_buffer                                      (100.00%)
                 1      block:block_dirty_buffer                                      (100.00%)
                 0      block:block_rq_abort                                          (100.00%)
                 0      block:block_rq_requeue                                        (100.00%)
                 5      block:block_rq_complete                                       (100.00%)
                 4      block:block_rq_insert                                         (100.00%)
                 4      block:block_rq_issue                                          (100.00%)
                 0      block:block_bio_bounce                                        (100.00%)
                 0      block:block_bio_complete                                      (100.00%)
                 1      block:block_bio_backmerge                                     (100.00%)
                 0      block:block_bio_frontmerge                                     (100.00%)
                 3      block:block_bio_queue                                         (100.00%)
                 2      block:block_getrq                                             (100.00%)
                 0      block:block_sleeprq                                           (100.00%)
                 1      block:block_plug                                              (100.00%)
                 1      block:block_unplug                                            (100.00%)
                 0      block:block_split                                             (100.00%)
                 3      block:block_bio_remap                                         (100.00%)
                 0      block:block_rq_remap

      10.000753335 seconds time elapsed

root@backend-shared-stag-0:~#
```

## Reporting

- Show perf.data in an ncurses browser

```
perf report
```

- Show perf.data as a text report

```
perf report --stdio
```

- List all events from perf.data

```
perf script
```

- Annotate assembly instructions from perf.data with percentages

```
perf annotate [--stdio]
```

## perf trace: trace system call & other events

- Trace syscalls system-wide

```
perf trace
```

输出

```
root@backend-shared-stag-0:~# perf trace
     0.000 ( 0.000 ms): iscsid/22378  ... [continued]: poll()) = 0 Timeout
     0.327 ( 0.000 ms): sshd/14484  ... [continued]: select()) = 1
     0.354 ( 0.290 ms): iscsid/22378 poll(ufds: 0x7fffee8683a0, nfds: 2, timeout_msecs: 250                ) ...
     0.369 ( 0.015 ms): sshd/14484 rt_sigprocmask(how: BLOCK, nset: 0x7fff7705d770, oset: 0x7fff7705d6f0, sigsetsize: 8) = 0
     0.396 ( 0.014 ms): sshd/14484 rt_sigprocmask(how: SETMASK, nset: 0x7fff7705d6f0, sigsetsize: 8      ) = 0
     0.427 ( 0.017 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d790               ) = 0
     0.489 ( 0.041 ms): sshd/14484 read(fd: 11</dev/ptmx>, buf: 0x7fff770596d0, count: 16384             ) = 160
     0.528 ( 0.015 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d710               ) = 0
     0.563 ( 0.018 ms): sshd/14484 select(n: 12, inp: 0x55cfc19091e0, outp: 0x55cfc1909770               ) = 2
     0.604 ( 0.024 ms): sshd/14484 rt_sigprocmask(how: BLOCK, nset: 0x7fff7705d770, oset: 0x7fff7705d6f0, sigsetsize: 8) = 0
     0.631 ( 0.014 ms): sshd/14484 rt_sigprocmask(how: SETMASK, nset: 0x7fff7705d6f0, sigsetsize: 8      ) = 0
     0.658 ( 0.014 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d790               ) = 0
     0.690 ( 0.016 ms): sshd/14484 read(fd: 11</dev/ptmx>, buf: 0x7fff770596d0, count: 16384             ) = 496
     0.729 ( 0.025 ms): sshd/14484 write(fd: 3<socket:[4138974]>, buf: 0x55cfc1929554, count: 196        ) = 196
     0.768 ( 0.014 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d710               ) = 0
     0.804 ( 0.019 ms): sshd/14484 select(n: 12, inp: 0x55cfc19091e0, outp: 0x55cfc1909770               ) = 2
     0.842 ( 0.020 ms): sshd/14484 rt_sigprocmask(how: BLOCK, nset: 0x7fff7705d770, oset: 0x7fff7705d6f0, sigsetsize: 8) = 0
     0.884 ( 0.021 ms): sshd/14484 rt_sigprocmask(how: SETMASK, nset: 0x7fff7705d6f0, sigsetsize: 8      ) = 0
     0.918 ( 0.014 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d790               ) = 0
     0.947 ( 0.015 ms): sshd/14484 read(fd: 11</dev/ptmx>, buf: 0x7fff770596d0, count: 16384             ) = 858
     0.984 ( 0.022 ms): sshd/14484 write(fd: 3<socket:[4138974]>, buf: 0x55cfc1929618, count: 532        ) = 532
     1.038 ( 0.022 ms): sshd/14484 clock_gettime(which_clock: BOOTTIME, tp: 0x7fff7705d710               ) = 0
     1.080 ( 0.019 ms): sshd/14484 select(n: 12, inp: 0x55cfc19091e0, outp: 0x55cfc1909770               ) = 2
     1.123 ( 0.019 ms): sshd/14484 rt_sigprocmask(how: BLOCK, nset: 0x7fff7705d770, oset: 0x7fff7705d6f0, sigsetsize: 8) = 0
...
```

- Trace syscalls for PID

```
perf trace -p PID
```

输出

```
root@backend-shared-stag-0:~# perf trace -p 8083
     0.000 ( 0.000 ms):  ... [continued]: epoll_wait()) = 0
     0.046 ( 0.014 ms): gettimeofday(tv: 0x707d30                                             ) = 0
     0.075 ( 0.014 ms): gettimeofday(tv: 0x707d80                                             ) = 0
     0.103 ( 0.014 ms): epoll_wait(events: 0x2773cf0, maxevents: 200                          ) = 0
     0.130 ( 0.014 ms): gettimeofday(tv: 0x707d30                                             ) = 0
     0.172 ( 0.026 ms): socket(family: INET, type: STREAM, protocol: 6                        ) = 1
     0.209 ( 0.015 ms): fcntl(fd: 1<socket:[4142749]>, cmd: SETFL, arg: 2048                  ) = 0
     0.242 ( 0.016 ms): setsockopt(fd: 1, level: 6, optname: 1, optval: 0x4af440, optlen: 4   ) = 0
     0.271 ( 0.014 ms): setsockopt(fd: 1, level: 6, optname: 12, optval: 0x4af444, optlen: 4  ) = 0
     0.336 ( 0.047 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = -1 EINPROGRESS Operation now in progress
     0.377 ( 0.014 ms): gettimeofday(tv: 0x707d80                                             ) = 0
     0.409 ( 0.018 ms): epoll_wait(events: 0x2773cf0, maxevents: 200                          ) = 0
     0.441 ( 0.017 ms): gettimeofday(tv: 0x707d30                                             ) = 0
     0.469 ( 0.014 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = -1 EALREADY Operation already in progress
     0.508 ( 0.023 ms): epoll_ctl(op: ADD, fd: 1, event: 0x6fc620                             ) = 0
     0.539 ( 0.016 ms): gettimeofday(tv: 0x707d80                                             ) = 0
     0.717 ( 0.164 ms): epoll_wait(events: 0x2773cf0, maxevents: 200, timeout: 1000           ) = 1
     0.762 ( 0.024 ms): gettimeofday(tv: 0x707d30                                             ) = 0
     0.802 ( 0.018 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = 0
     0.832 ( 0.015 ms): recvfrom(fd: 1<socket:[4142749]>, size: 2147483647, flags: TRUNC|DONTWAIT|NOSIGNAL) = -1 EAGAIN Resource temporarily unavailable
     0.865 ( 0.019 ms): setsockopt(fd: 1, level: 1, optname: 13, optval: 0x4af438, optlen: 8  ) = 0
     0.910 ( 0.018 ms): close(fd: 1<socket:[4142749]>                                         ) = 0
     0.969 ( 0.015 ms): gettimeofday(tv: 0x707d80                                             ) = 0
  1002.114 (1001.127 ms): epoll_wait(events: 0x2773cf0, maxevents: 200, timeout: 1000           ) = 0
  1002.165 ( 0.016 ms): gettimeofday(tv: 0x707d30                                             ) = 0
  1002.203 ( 0.022 ms): gettimeofday(tv: 0x707d80                                             ) = 0
  2002.367 (1000.140 ms): epoll_wait(events: 0x2773cf0, maxevents: 200, timeout: 999            ) = 0
  2002.418 ( 0.015 ms): gettimeofday(tv: 0x707d30                                             ) = 0
  2002.456 ( 0.021 ms): gettimeofday(tv: 0x707d80                                             ) = 0
  2002.506 ( 0.025 ms): epoll_wait(events: 0x2773cf0, maxevents: 200                          ) = 0
  2002.552 ( 0.018 ms): gettimeofday(tv: 0x707d30                                             ) = 0
  2002.611 ( 0.035 ms): socket(family: INET, type: STREAM, protocol: 6                        ) = 1
  2002.656 ( 0.017 ms): fcntl(fd: 1<socket:[4142750]>, cmd: SETFL, arg: 2048                  ) = 0
  2002.689 ( 0.019 ms): setsockopt(fd: 1, level: 6, optname: 1, optval: 0x4af440, optlen: 4   ) = 0
  2002.743 ( 0.022 ms): setsockopt(fd: 1, level: 6, optname: 12, optval: 0x4af444, optlen: 4  ) = 0
  2002.816 ( 0.059 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = -1 EINPROGRESS Operation now in progress
  2002.892 ( 0.028 ms): gettimeofday(tv: 0x707d80                                             ) = 0
  2002.933 ( 0.021 ms): epoll_wait(events: 0x2773cf0, maxevents: 200                          ) = 0
  2002.994 ( 0.029 ms): gettimeofday(tv: 0x707d30                                             ) = 0
  2003.025 ( 0.014 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = -1 EALREADY Operation already in progress
  2003.084 ( 0.039 ms): epoll_ctl(op: ADD, fd: 1, event: 0x6fc620                             ) = 0
  2003.121 ( 0.016 ms): gettimeofday(tv: 0x707d80                                             ) = 0
  2003.152 ( 0.017 ms): epoll_wait(events: 0x2773cf0, maxevents: 200, timeout: 1000           ) = 1
  2003.195 ( 0.025 ms): gettimeofday(tv: 0x707d30                                             ) = 0
  2003.251 ( 0.027 ms): connect(fd: 1, uservaddr: 0x273fb38, addrlen: 16                      ) = 0
  2003.296 ( 0.023 ms): recvfrom(fd: 1<socket:[4142750]>, size: 2147483647, flags: TRUNC|DONTWAIT|NOSIGNAL) = -1 EAGAIN Resource temporarily unavailable
  2003.339 ( 0.025 ms): setsockopt(fd: 1, level: 1, optname: 13, optval: 0x4af438, optlen: 8  ) = 0
  2003.386 ( 0.027 ms): close(fd: 1<socket:[4142750]>                                         ) = 0
  2003.472 ( 0.025 ms): gettimeofday(tv: 0x707d80                                             ) = 0
```

## perf record: record profiling data (records into perf.data file)

- Sample CPU functions for COMMAND, at 99 Hertz

```
perf record -F 99 COMMAND
```

- Sample CPU functions for PID, until Ctrl-C

```
perf record -p PID
```

- Sample CPU functions for PID, for 10 seconds

```
perf record -p PID sleep 10
```

- Sample CPU stack traces for PID, for 10 seconds

```
perf record -p PID -g -- sleep 10
```

- Sample CPU stack traces for PID, using DWARF to unwind stack

```
perf record -p PID --call-graph dwarf
```

## perf record: record tracing data (records into perf.data file)

- Trace new processes, until Ctrl-C

```
perf record -e sched:sched_process_exec -a
```

- Trace all context-switches, until Ctrl-C

```
perf record -e context-switches -a
```

- Trace all context-switches with stack traces, for 10 seconds

```
perf record -e context-switches -ag -- sleep 10
```

- Trace all page faults with stack traces, until Ctrl-C

```
perf record -e page-faults -ag
```

## adding new trace events

- Add a tracepoint for kernel function tcp_sendmsg()

```
perf probe 'tcp_sendmsg'
```

- Trace previously created probe

```
perf record -e -a probe:tcp_sendmsg
```

- Add a tracepoint for myfunc() return, and include the retval as a string

```
perf probe 'myfunc%return +0($retval):string'
```

- Trace previous probe when size > 0, and state is not TCP_ESTABLISHED(1)

> need kernel debuginfo

```
perf record -e -a probe:tcp_sendmsg --filter 'size > 0 && skc_state != 1' -a
```

- Add a tracepoint for do_sys_open() with the filename as a string

> need kernel debuginfo

```
perf probe 'do_sys_open filename:string'
```



----------



## Usage Examples

These example sequences have been chosen to illustrate some different ways that perf is used, from gathering to reporting.

- Performance counter summaries, including IPC, for the gzip command:

```
# perf stat gzip largefile
```

- Count all scheduler process events for 5 seconds, and count by tracepoint:

```
# perf stat -e 'sched:sched_process_*' -a sleep 5
```

- Trace all scheduler process events for 5 seconds, and count by both tracepoint and process name:

```
# perf record -e 'sched:sched_process_*' -a sleep 5
# perf report
```

- Trace all scheduler process events for 5 seconds, and dump per-event details:

```
# perf record -e 'sched:sched_process_*' -a sleep 5
# perf script
```

- Trace read() syscalls, when requested bytes is less than 10:

```
# perf record -e 'syscalls:sys_enter_read' --filter 'count < 10' -a
```

- Sample CPU stacks at 99 Hertz, for 5 seconds:

```
# perf record -F 99 -ag -- sleep 5
# perf report
```

- Dynamically instrument the kernel tcp_sendmsg() function, and trace it for 5 seconds, with stack traces:

```
# perf probe --add tcp_sendmsg
# perf record -e probe:tcp_sendmsg -ag -- sleep 5
# perf probe --del tcp_sendmsg
# perf report
```

Deleting the tracepoint (--del) wasn't necessary; I included it to show how to return the system to its original state.

## Listing Events

```
# Listing all currently known events:
perf list

# Listing sched tracepoints:
perf list 'sched:*'
```

## Counting Events

```
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

# Using raw PMC counters, eg, unhalted core cycles:
perf stat -e r003c -a sleep 5 

# PMCs: cycles and frontend stalls via raw specification:
perf stat -e cycles -e cpu/event=0x0e,umask=0x01,inv,cmask=0x01/ -a sleep 5

# Count syscalls per-second system-wide (this should be in /proc):
perf stat -e raw_syscalls:sys_enter -I 1000 -a

# Count system calls by type for the specified PID, until Ctrl-C:
perf stat -e 'syscalls:sys_enter_*' -p PID

# Count system calls by type for the entire system, for 5 seconds:
perf stat -e 'syscalls:sys_enter_*' -a sleep 5

# Count scheduler events for the specified PID, until Ctrl-C:
perf stat -e 'sched:*' -p PID

# Count scheduler events for the specified PID, for 10 seconds:
perf stat -e 'sched:*' -p PID sleep 10

# Count ext4 events for the entire system, for 10 seconds:
perf stat -e 'ext4:*' -a sleep 10

# Count block device I/O events for the entire system, for 10 seconds:
perf stat -e 'block:*' -a sleep 10

# Count all vmscan events, printing a report every second:
perf stat -e 'vmscan:*' -a -I 1000

# Show system calls by process, refreshing every 2 seconds:
perf top -e raw_syscalls:sys_enter -ns comm

# Show sent network packets by on-CPU process, rolling output (no clear):
stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings
```


## Profiling

```
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

# Perform branch tracing (needs HW support), for 1 second:
perf record -b -a sleep 1

# Sample CPUs at 49 Hertz, and show top addresses and symbols, live (no perf.data file):
perf top -F 49

# Sample CPUs at 49 Hertz, and show top process names and segments, live:
perf top -F 49 -ns comm,dso
```

## Static Tracing

```
# Trace new processes, until Ctrl-C:
perf record -e sched:sched_process_exec -a

# Trace all context-switches, until Ctrl-C:
perf record -e context-switches -a

# Trace context-switches via sched tracepoint, until Ctrl-C:
perf record -e sched:sched_switch -a

# Trace all context-switches with stack traces, until Ctrl-C:
perf record -e context-switches -ag

# Trace all context-switches with stack traces, for 10 seconds:
perf record -e context-switches -ag -- sleep 10

# Trace all CS, stack traces, and with timestamps (< Linux 3.17, -T now default):
perf record -e context-switches -ag -T

# Trace CPU migrations, for 10 seconds:
perf record -e migrations -a -- sleep 10

# Trace all connect()s with stack traces (outbound connections), until Ctrl-C:
perf record -e syscalls:sys_enter_connect -ag

# Trace all accepts()s with stack traces (inbound connections), until Ctrl-C:
perf record -e syscalls:sys_enter_accept* -ag

# Trace all block device (disk I/O) requests with stack traces, until Ctrl-C:
perf record -e block:block_rq_insert -ag

# Trace all block device issues and completions (has timestamps), until Ctrl-C:
perf record -e block:block_rq_issue -e block:block_rq_complete -a

# Trace all block completions, of size at least 100 Kbytes, until Ctrl-C:
perf record -e block:block_rq_complete --filter 'nr_sector > 200'

# Trace all block completions, synchronous writes only, until Ctrl-C:
perf record -e block:block_rq_complete --filter 'rwbs == "WS"'

# Trace all block completions, all types of writes, until Ctrl-C:
perf record -e block:block_rq_complete --filter 'rwbs ~ "*W*"'

# Trace all minor faults (RSS growth) with stack traces, until Ctrl-C:
perf record -e minor-faults -ag

# Trace all page faults with stack traces, until Ctrl-C:
perf record -e page-faults -ag

# Trace all ext4 calls, and write to a non-ext4 location, until Ctrl-C:
perf record -e 'ext4:*' -o /tmp/perf.data -a 

# Trace kswapd wakeup events, until Ctrl-C:
perf record -e vmscan:mm_vmscan_wakeup_kswapd -ag

# Add Node.js USDT probes (Linux 4.10+):
perf buildid-cache --add `which node`

# Trace the node http__server__request USDT event (Linux 4.10+):
perf record -e sdt_node:http__server__request -a
```


## Dynamic Tracing

```
# Add a tracepoint for the kernel tcp_sendmsg() function entry ("--add" is optional):
perf probe --add tcp_sendmsg

# Remove the tcp_sendmsg() tracepoint (or use "--del"):
perf probe -d tcp_sendmsg

# Add a tracepoint for the kernel tcp_sendmsg() function return:
perf probe 'tcp_sendmsg%return'

# Show available variables for the kernel tcp_sendmsg() function (needs debuginfo):
perf probe -V tcp_sendmsg

# Show available variables for the kernel tcp_sendmsg() function, plus external vars (needs debuginfo):
perf probe -V tcp_sendmsg --externs

# Show available line probes for tcp_sendmsg() (needs debuginfo):
perf probe -L tcp_sendmsg

# Show available variables for tcp_sendmsg() at line number 81 (needs debuginfo):
perf probe -V tcp_sendmsg:81

# Add a tracepoint for tcp_sendmsg(), with three entry argument registers (platform specific):
perf probe 'tcp_sendmsg %ax %dx %cx'

# Add a tracepoint for tcp_sendmsg(), with an alias ("bytes") for the %cx register (platform specific):
perf probe 'tcp_sendmsg bytes=%cx'

# Trace previously created probe when the bytes (alias) variable is greater than 100:
perf record -e probe:tcp_sendmsg --filter 'bytes > 100'

# Add a tracepoint for tcp_sendmsg() return, and capture the return value:
perf probe 'tcp_sendmsg%return $retval'

# Add a tracepoint for tcp_sendmsg(), and "size" entry argument (reliable, but needs debuginfo):
perf probe 'tcp_sendmsg size'

# Add a tracepoint for tcp_sendmsg(), with size and socket state (needs debuginfo):
perf probe 'tcp_sendmsg size sk->__sk_common.skc_state'

# Tell me how on Earth you would do this, but don't actually do it (needs debuginfo):
perf probe -nv 'tcp_sendmsg size sk->__sk_common.skc_state'

# Trace previous probe when size is non-zero, and state is not TCP_ESTABLISHED(1) (needs debuginfo):
perf record -e probe:tcp_sendmsg --filter 'size > 0 && skc_state != 1' -a

# Add a tracepoint for tcp_sendmsg() line 81 with local variable seglen (needs debuginfo):
perf probe 'tcp_sendmsg:81 seglen'

# Add a tracepoint for do_sys_open() with the filename as a string (needs debuginfo):
perf probe 'do_sys_open filename:string'

# Add a tracepoint for myfunc() return, and include the retval as a string:
perf probe 'myfunc%return +0($retval):string'

# Add a tracepoint for the user-level malloc() function from libc:
perf probe -x /lib64/libc.so.6 malloc

# Add a tracepoint for this user-level static probe (USDT, aka SDT event):
perf probe -x /usr/lib64/libpthread-2.24.so %sdt_libpthread:mutex_entry

# List currently available dynamic probes:
perf probe -l
```


## Mixed

```
# Sample stacks at 99 Hertz, and, context switches:
perf record -F99 -e cpu-clock -e cs -a -g 

# Sample stacks to 2 levels deep, and, context switch stacks to 5 levels (needs 4.8):
perf record -F99 -e cpu-clock/max-stack=2/ -e cs/max-stack=5/ -a -g 
```


## Special

```
# Record cacheline events (Linux 4.10+):
perf c2c record -a -- sleep 10

# Report cacheline events from previous recording (Linux 4.10+):
perf c2c report
```


## Reporting

```
# Show perf.data in an ncurses browser (TUI) if possible:
perf report

# Show perf.data with a column for sample count:
perf report -n

# Show perf.data as a text report, with data coalesced and percentages:
perf report --stdio

# Report, with stacks in folded format: one line per stack (needs 4.4):
perf report --stdio -n -g folded

# List all events from perf.data:
perf script

# List all perf.data events, with data header (newer kernels; was previously default):
perf script --header

# List all perf.data events, with customized fields (< Linux 4.1):
perf script -f time,event,trace

# List all perf.data events, with customized fields (>= Linux 4.1):
perf script -F time,event,trace

# List all perf.data events, with my recommended fields (needs record -a; newer kernels):
perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso 

# List all perf.data events, with my recommended fields (needs record -a; older kernels):
perf script -f comm,pid,tid,cpu,time,event,ip,sym,dso

# Dump raw contents from perf.data as hex (for debugging):
perf script -D

# Disassemble and annotate instructions with percentages (needs some debuginfo):
perf annotate --stdio
```


