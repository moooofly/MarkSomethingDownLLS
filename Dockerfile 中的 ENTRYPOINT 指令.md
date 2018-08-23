# Dockerfile 中的 ENTRYPOINT 指令

Ref:

- [dockerfile_best-practices/#entrypoint](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#entrypoint)
- [reference/builder/#entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint)

## 示例脚本

[Postgres Official Image](https://hub.docker.com/_/postgres/) 使用如下脚本内容作为其 ENTRYPOINT ：

> **Set options directly on the run line**. The `entrypoint` script is made so that any options passed to the docker command will be passed along to the postgres server daemon.

好处在于，可以在执行 `docker run` 等命令时，指定需要传入到内部具体应用的 options ；

> If you need to write a starter script for a single executable, you can **ensure that the final executable receives the Unix signals** by using `exec` and `gosu` commands:

在使用了 `exec` 和 `gosu` 后，能确保目标应用程序收到 Unix signals ；

```
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

注意：`gosu` 后面的参数 postgres 为 `user-spec` ；

## 基础

`ENTRYPOINT` 指令具有两种使用形式：

- `ENTRYPOINT ["executable", "param1", "param2"]` -- `exec` 形式，推荐
- `ENTRYPOINT command param1 param2` -- `shell` 形式

若存在多条 ENTRYPOINT 指令，则只有最后一条生效；

## 常规用法

> The best use for `ENTRYPOINT` is to set the image’s main command, allowing that image to be run as though it was that command (and then use `CMD` as the default flags).

在 `ENTRYPOINT` 指令中设置镜像主命令；在 `CMD` 中设置默认 flags ；

> This is useful because the image name can double as a reference to the binary as shown in the command above.

好处在于，可以直接镜像名字当做二进制程序命令来用；

## 高级用法

> The `ENTRYPOINT` instruction can also be used in combination with a **helper script**, allowing it to function in a similar way to the command above, even when starting the tool may require more than one step.

将 `ENTRYPOINT` 指令和 script 脚本结合使用；

> The helper script is copied into the container and run via `ENTRYPOINT` on container start:

使用套路如下

```
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```

使用了上述脚本后，用户可以通过如下几种方式和 Postgres 进行销户：

- 简单启动 Postgres

```
$ docker run postgres
```

- 启动 Postgres 的同时传递一些 parameters 到内部

```
$ docker run postgres postgres --help
```

注：`postgres --help` 将作为替代 Dockerfile 中 CMD 指令的内容使用；

- 启动 Bash 等其他工具（**方便调试使用**）

```
$ docker run --rm -it postgres bash
```

## Q&A

### [The exec builtin command](http://wiki.bash-hackers.org/commands/builtin/exec)

> The `exec` builtin command is used to
>
> - **replace** the shell with a given program (executing it, **not as new process**)
> - set redirections for the program to execute or for the current shell

exec 的用途：

- 使用指定程序替代当前 shell 进程（不创建新进程）；
- 用于重定向指定程序的 stdin/stdout/stderr ；

> If only redirections are given, the redirections affect the current shell without executing any program.

### Why we want to configure app as PID 1

> This script **uses the `exec` Bash command** so that the final running application **becomes the container’s PID 1**. This **allows the application to receive any Unix signals sent to the container**.

- 使用 exec 执行目标程序后，目标程序将会成为容器的 PID 1 进程；
- 容器的 PID 1 进程可以接收到任何发送给容器的 Unix signals ；

> The `shell` form (of `ENTRYPOINT`) **prevents** any `CMD` or `run` command line arguments from being used, but has the **disadvantage** that your `ENTRYPOINT` will be started as a subcommand of `/bin/sh -c`, which **does not pass signals**. This means that the executable **will not be the container’s PID 1** - and **will not receive Unix signals** - so your executable **will not receive a SIGTERM from `docker stop <container>`**.

- shell 形式的 ENTRYPOINT 指令在运行指定程序时，会将其作为 `/bin/sh -c` 的子命令（子进程）；
- 因为常规 shell 进程不会负责 signals 转发，通常情况下 signals 转发是由具有 daemon 特性的程序来负责的，例如 init 等；因此，无法进行 Unix signals 的传递就没啥奇怪的了；
- 理论上讲，PID 1 的特殊性在于其作为容器内外沟通的桥梁，例如针对 signals 的处理，所以，要么将业务进程直接作为 PID 1 运行，要么就需要使用一个具有 signals 处理功能的 daemon 进程来管理业务进程；很明显，常规 shell 不具备该能力；


### Understand how CMD and ENTRYPOINT interact

Both `CMD` and `ENTRYPOINT` instructions define what command gets executed when running a container. There are few rules that describe their co-operation.

- Dockerfile should specify at least one of `CMD` or `ENTRYPOINT` commands.
- `ENTRYPOINT` should be defined when using the container as an executable.
- `CMD` should be used as a way of defining default arguments for an `ENTRYPOINT` command or for executing an ad-hoc command in a container.
- `CMD` will be overridden when running the container with alternative arguments.

The table below shows what command is executed for different `ENTRYPOINT` / `CMD` combinations:

| - | No ENTRYPOINT | ENTRYPOINT exec_entry p1_entry | ENTRYPOINT [“exec_entry”, “p1_entry”] |
| -- | -- | -- | -- |
| No CMD | error, not allowed | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry |
| CMD [“exec_cmd”, “p1_cmd”] | exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry    | exec_entry p1_entry exec_cmd p1_cmd |
| CMD [“p1_cmd”, “p2_cmd”] | p1_cmd p2_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry p1_cmd p2_cmd |
| CMD exec_cmd p1_cmd | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd |

