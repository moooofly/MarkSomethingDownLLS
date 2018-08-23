# strace 的使用场景

> https://eklitzke.org/strace

A lot of people know about `strace`, but in my opinion fail to use it effectively because they don’t use it at the right time.

- **Any time a program is crashing right after you start it**, you can probably figure out what’s going on with `strace`. The most common situation here is that the program is making a failed system call and then exiting after the failed system call. Frequently this will be something like a missing file or a permissions error (e.g. **ENOENT** or **EPERM**), but it could be almost any failed system call. As I said before, try to work backwards from the end of the `strace` output.

- **Any time you have a program that appears to be hung**, you should use `strace -p` to see what system call it’s stuck on (or what sequence of system calls it’s stuck on). **Any time you have a program that appears to be running really slowly**, you should use `strace` or `strace -p` and see if there are system calls that are taking a long time.

In both of the cases I just listed, there are **a few common patterns** that you’ll see:

- Programs that are **stuck in a `wait` call** are waiting for a child process to complete, proceed by tracing the child processes.
- Programs that are **stuck in `select(2)` or `epoll_wait(2)` calls** or loops without making other system calls are typically waiting forever for network data. To debug, try to figure out what file descriptors are in the `select` or `epoll` (you can typically figure this out with more `strace`, by looking in `/proc`, or using `lsof`). This is potentially kind of tricky to do with edge-triggered epoll loops, you may need to use `gdb` or restart the program if you don’t have sufficient context.
- Programs that are **stuck in calls like `connect(2)`, `read(2)`, `write(2)`, `sendto(2)`, `recvfrom(2)`, etc. are also stuck doing I/O**, and once again you should use something like `lsof` to figure out what the other end of the file descriptor is.
- Programs that are **stuck in the `futex(2)` system call** typically have some sort of **multi-threading related problem** as this is the primitive that `Pthreads` uses on Linux. You’ll see this also in higher level programs written in languages like **Python**, since Python threads are implemented on Linux using `Pthreads`. This can’t easily be debugged with `strace` but often knowing that it’s a **threading problem** is sufficient to point you in the right direction. If you need to do more low level debugging, proceed with `gdb` (I would recommend starting by attaching to the process and getting a backtrace).

Another really common use case for me is **programs that aren’t loading the right files**. Here’s one that comes up a lot. I frequently see people who have a Python program that has an import statement that is either loading the wrong module, or that is failing even though the developer thinks that the module should be able to be imported. **How can you figure out where Python is attempting to load the module from?** You can use `strace` and then look for calls to `open(2)` (or perhaps `acces(2)`), and then `grep` for the module name in question. Quickly you’ll see what files the Python interpreter is actually opening (or attempting to open), and that will dispell any misconceptions about what may or may not be in your `sys.path`, will identify stale bytecode files, etc.

**If there aren’t any slow system calls or the system calls don’t appear to be looping then the process is likely stuck on some sort of userspace thing**. For instance, if the program is computing the value of pi to trillions of decimal places it will probably not be doing any system calls (or doing very few system calls that complete quickly). In this case you need to use a userspace debugger like `gdb`.

One final piece of advice I have is to try to make an effort to learn as many system calls as you can and figure out why programs are making them. If you’ve never used `strace` before the output is going to be really overwhelming and confusing, particularly if you don’t have much Unix C programming experience. However, there actually aren’t that many system calls that are regularly used or that are useful when debugging these issues. There are also common sequences or patterns to system calls (e.g. for things like program initialization, memory allocation, network I/O, etc.) that you can pick up quickly that will make scanning `strace` output really fast. All of the system calls you see will have man pages available that explain all of the arguments and return codes, so you don’t even need to leave your terminal or have an internet connection to figure out what’s going on.

Happy debugging, and may the `strace` gods bless you with quick results.




