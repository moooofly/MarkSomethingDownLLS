# iostat cheatsheet

Background:

- `iostat` reports at **physics device**/**sector level** (i.e. beneath cache and IO scheduling)

Overall utilization:

- avg-cpu / `%iowait`: how busy is the CPU, or “amount of computation waiting for IO”
- device / `%util`: **how busy is the device**

Columns (listed per device):

- `rrqm/s` `wrqm/s`: r/w requests **merged** per second
    - **Block IO subsystem** may merge **physically adjacent** requests
- `r/s` `w/s`: number of (possibly merged) r/w requests
- `rsec/s` `wsec/s`: number of sectors read/written
    - `iostat` can also be set to use units rkB/s wkB/s rMB/s wMB/s
- `avgrq-sz`: **average size of request**, in #sectors
    - **larger size indicates sequential IO, smaller indicates random IO**
- `avgqu-sz`: **average length of request queue**, i.e. average #request waiting to be served
- `await`: average (**wait** + **serve**) time for a request, in ms
    - `r_await` / `w_await`: the same time for r/w requests
- `svctm`: average **serve** time for a request, in ms

> 需要重点关注 await 、svctm 以及 await - svctm 的数值关系；一般来讲，await 和 svctm 数值接近才是好的表现；如果 await 远超 svctm 的值，则说明 IO device 可能已经过载了；

A monitor command with iostat:

```
# -x: extended report
# -t: print time for report
# -z: omit inactive devices
# 1: report every 1 second

$ iostat -xtz 1
```

## Linux Disk IO Subsystem

> a basic background knowledge of the Disk IO Subsystem.

| Layer | Unit | Typical Unit Size |
| -- | -- | -- |
| User Space System Calls | read() , write() | |
| Virtual File System Switch (VFS) | Block | 4096 Bytes |
| Disk Caches | Page |  |
| Filesystem (For example ext3) | Blocks | 4096 Bytes (Can be set at FS creation) |
| Generic Block Layer | Page Frames / Block IO Operations (bio) |  |
| I/O Scheduler Layer | bios per block device (Which this layer may combine) |  |
| Block Device Driver | Segment | 512 Bytes |
| Hard Disk | Sector | 512 Bytes |


可以认为 iostat 工作在 I/O Scheduler Layer 之下

> There are two basic system calls, `read()` and `write()`, that a user process can make to read data from a file system. In the kernel these are handled by the Linux **Virtual Filesystem Switch (VFS)**. VFS is an abstraction to all file systems so they look the same to the user space and it also handles the interface between the file system and the block device layer. The **caching layer** provides caching of disk reads and writes in terms of memory pages. The **generic block layer** breaks down IO operations that might involve many different non-contiguous blocks into multiple IO operations. The **I/O scheduling layer** takes these IO operations and schedules them based on order on disk, priority, and/or direction. Lastly, the **device driver** handles interfacing with the hardware for the actual operations in terms of disk sectors which are usually 512 bytes.

`iostat` can break down the statistics at both the **partition level** and then **device level**.

**Device utilization** is "The percentage of time the device spent servicing requests as opposed to being idle."

It is import to note that **`iostat` shows requests to the device (or partition) and not read and write requests from user space**. So in the table above `iostat` is reading below the disk cache layer. Therefore, **`iostat` says noting about your cache hit ratio for block devices**. So it is possible that disk IO problems might be able to be resolved by memory upgrades.

> iostat 展示的是针对 device 或 partition 的读写，不是来自用户空间的读写请求；iostat 没有给出关于 cache hit/miss 的相关信息；

One of the main things to do when examining disk IO is **to determine if the disk access patterns are sequential or random**. This information can aid in our disk choices. When operations are random the seek time of the disk becomes more important. This is because physically the drive head has to jump around. Seek time is the measurement of the speed at which the heads can do this. **For small random reads solid state disks can be a huge advantage**.

Snapshot of **Random Read** Test:

```
Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00     0.00  172.67    0.00  1381.33     0.00     8.00     0.99    5.76   5.76  99.47
```

Snapshot of **Sequential Read** Test:

```
Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda              13.00     0.00  367.00    0.00 151893.33     0.00   413.88     2.46    6.71   2.72 100.00
```

> 如何模拟顺序读和随机读；

`rrqm/s` and `wrqm/s`, are read and write requests merged per second. I mentioned that that the scheduler can combine operations. This can be done **when multiple operations are physically adjacent to each other on the device**. So in sequential operation it would make sense to often see a large number of merges. In the snapshot of the random reads, we see no merges. However, the merging layer feels a little bit like “magic” and I don’t believe it is the best indicator of if the patterns are random or sequential.

read and write requests to the device (`r/s`, `w/s`), followed by the amount of sectors read and written from the device (`rsec/s`, `wsec/s`), and then the size of each request (`avgrq-sz`). However the average request size does not differentiate between reads and writes.

> avgrq-sz — average sectors per request (for both reads and writes). 

the average queue length of requests to the device (`avgqu-sz`), how long requests took to be serviced including their time in the queue (`await`), how long requests took to be serviced by the device after they left the queue (`svctm`), and lastly the utilization percentage (`util`).


> await — average response time (ms) of IO requests to a device. 
> svctim — average time (ms) a device was servicing requests. This is a component of total response time of IO requests.


Utilization (`util`), in more detail, is the `service time in ms * total IO operations / 1000 ms`. This gives the percentage of how busy the single disk was during the given time slice.


> `utilization = ( (read requests + write requests) * service time in ms / 1000 ms ) * 100%`
> 
> or
>
> `%util = ( r + w ) * svctim /10` = ( 10.18 + 9.78 ) * 8.88 = 17.72448


In the end it seems **average request size is the key to show if the disk usage patterns are random or not since this is post merging**. Taking this into the context of the layers above this might not mirror what an application is doing. This is because a read or write operations coming from user space might operate on a fragmented file in which case the generic block layer will break it up and it appears as random disk activity.


----------

参考：

- [Interpreting iostat Output](https://blog.serverfault.com/2010/07/06/777852755/)
- [Basic I/O Monitoring On Linux](https://blog.pythian.com/basic-io-monitoring-on-linux/)
- [Is there a way to get Cache Hit/Miss ratios for block devices in Linux?](https://serverfault.com/questions/157612/is-there-a-way-to-get-cache-hit-miss-ratios-for-block-devices-in-linux/157724#157724)
- [9 Linux iostat Command Examples for Performance Monitoring](https://linoxide.com/monitoring-2/find-linux-disk-utilization-iostat/)
- [5 TOOLS FOR MONITORING DISK ACTIVITY IN LINUX](https://www.opsdash.com/blog/disk-monitoring-linux.html)

