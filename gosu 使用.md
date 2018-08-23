# gosu 使用

github 地址：https://github.com/tianon/gosu

一句话介绍：Simple Go-based setuid+setgid+setgroups+exec

## 解决的问题

> This is a simple tool grown out of the simple fact that **`su` and `sudo` have very strange and often annoying TTY and signal-forwarding behavior**. They're also somewhat **complex to setup and use** (especially in the case of `sudo`), which allows for a great deal of expressivity, but falls flat if all you need is "**run this specific application as this specific user and get out of the pipeline**".

- su 和 sudo 在处理 tty 和 signal 转发相关的东西时会有奇怪的表现；
- 由于 expressivity 太过丰富，su 和 sudo 在使用时比较复杂；在“使用指定用户身份运行特定应用后，从 pipeline 中脱离”的场景中很难使用；

## 工作原理

> The core of how `gosu` works is stolen directly from how `Docker/libcontainer` itself starts an application inside a container (and in fact, is using the `/etc/passwd` processing code directly from libcontainer's codebase).

gosu 工作原理的核心来自于 `Docker/libcontainer` 中使用的、在容器中启动应用的方法；

> Once the user/group is processed, we switch to that user, then we `exec` the specified process and `gosu` itself is no longer resident or involved in the process lifecycle at all. This avoids all the issues of signal passing and TTY, and punts them to the process invoking `gosu` and the process being invoked by `gosu`, where they belong.

一旦 user/group 的相关处理完成后，则成功切换到对应的 user 身份，之后再通过 `exec` 执行指定的进程；而 `gosu` 本身在业务进程的整个生命周期中将不会存在；通过这种方式，避免了 signal 处理和 tty 问题；

> Additionally, due to the fact that `gosu` is using Docker's own code for processing these `user:group`, it has exact 1:1 parity with Docker's own `--user` flag.

> If you're curious about the edge cases that gosu handles, see [Dockerfile.test](https://github.com/tianon/gosu/blob/master/Dockerfile.test) for the "test suite" (and the associated [test.sh](https://github.com/tianon/gosu/blob/master/test.sh) script that wraps this up for testing arbitrary binaries).

## 使用场景

> The core use case for `gosu` is to step down from `root` to a non-privileged user during container startup (specifically in the `ENTRYPOINT`, usually).

`gosu` 的核心使用场景：在容器启动的过程中（尤其是用于 `ENTRYPOINT`），将 `root` 身份降权为非特权用户身份；

> Uses of `gosu` beyond that could very well suffer from vulnerabilities such as CVE-2016-2779 (from which the Docker use case naturally shields us); see tianon/gosu#37 for some discussion around this point.

超出该使用范围，则可能容易受到漏洞问题影响；

## 安装方法

- [Docker 内安装](https://github.com/tianon/gosu/blob/master/INSTALL.md)
- [宿主机安装](https://github.com/moooofly/scaffolding/blob/master/gosu_setup.sh)

## 用法

```
$ gosu
Usage: ./gosu user-spec command [args]
   ie: ./gosu tianon bash
       ./gosu nobody:root bash -c 'whoami && id'
       ./gosu 1000:1 id

./gosu version: 1.1 (go1.3.1 on linux/amd64; gc)
```

## gosu 使用示例

- 基于 su

```
[#381#root@ubuntu-1604 ~]$docker run -it --rm ubuntu:trusty su -c 'exec ps aux'
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.2  46640  2608 pts/0    Ss+  06:57   0:00 su -c exec ps a
root         6  0.0  0.2  15580  2112 ?        Rs   06:57   0:00 ps aux
[#382#root@ubuntu-1604 ~]$
```

- 基于 sudo

```
[#382#root@ubuntu-1604 ~]$docker run -it --rm ubuntu:trusty sudo ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.3  46024  3076 pts/0    Ss+  06:58   0:00 sudo ps aux
root         7  0.0  0.2  15580  2096 pts/0    R+   06:58   0:00 ps aux
[#383#root@ubuntu-1604 ~]$
```

- 基于 gosu

```
[#383#root@ubuntu-1604 ~]$docker run -it --rm -v /usr/local/bin/gosu:/usr/local/bin/gosu:ro ubuntu:trusty gosu root ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   7148   848 pts/0    Rs+  06:59   0:00 ps aux
[#384#root@ubuntu-1604 ~]$
```

## 和 gosu 类似的工具

### [su-exec](https://github.com/ncopa/su-exec)

As mentioned in `INSTALL.md`, `su-exec` is a very minimal re-write of `gosu` in C, making for a much smaller binary, and is available in the main **Alpine** package repository.


### chroot

With the `--userspec` flag, `chroot` can provide similar benefits/behavior:

```
[#388#root@ubuntu-1604 ~]$docker run -it --rm ubuntu:trusty chroot --userspec=root / ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   7148   756 pts/0    Rs+  07:09   0:00 ps aux
[#389#root@ubuntu-1604 ~]$
```
