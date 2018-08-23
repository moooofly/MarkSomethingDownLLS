# Docker Compose 学习

> GitHub 地址：[这里](https://github.com/docker/compose)

- 概述

> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a Compose file to configure your application's services. Then, using a single command, you create and start all the services from your configuration. 

- [特性](https://github.com/docker/docker.github.io/blob/master/compose/overview.md#features)

> - Multiple **isolated** environments on a single host
> - **Preserve** volume data when containers are created
> - **Only** recreate containers that have changed
> - Variables and **moving** a composition between environments

- [常规用例](https://github.com/docker/docker.github.io/blob/master/compose/overview.md#common-use-cases)

> - Development environments
> - Automated testing environments
> - Single host deployments

- 使用步骤

> Using Compose is basically a three-step process.
> 
> - Define your app's environment with a `Dockerfile` so it can be reproduced anywhere.
> - Define the services that make up your app in `docker-compose.yml` so they can be run together in an **isolated** environment.
> - Lastly, run `docker-compose up` and Compose will start and run your entire app.


----------


## Docker Compose CLI

### [Overview of docker-compose CLI](https://docs.docker.com/compose/reference/overview/)

可以通过 `docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]` 命令 **build** 和 **manage** 基于 Docker containers 运行的 multiple services ；

- You can supply multiple `-f` configuration files. -- 多个
- When you supply multiple files, Compose **combines** them **into** a single configuration. -- 合并
- Compose builds the configuration in the order you supply the files. Subsequent files **override** and add to their predecessors. -- 覆盖+追加
- If you don’t provide `-f` flag on the command line, Compose traverses the working directory and its parent directories looking for a `docker-compose.yml` and a `docker-compose.override.yml` file. -- 查找
- You can use `-f` flag to specify a path to Compose file that is not located in the current directory, either from the command line or by setting up a `COMPOSE_FILE` environment variable in your shell or in an environment file. -- 指定路径查找
- Use `-p` to specify a **project name**: Each configuration has a project name. If you supply a `-p` flag, you can specify a project name. If you don’t specify the flag, Compose uses the current directory name. 

### [Compose CLI environment variables](https://docs.docker.com/compose/reference/envvars/)

- `COMPOSE_PROJECT_NAME`: Sets the **project name**. This value is prepended along with the **service name** to the container on start up. -- 对应 `-p` 选项

从这里知道，如下的 harbor-xxx 命名其实正是 `<project name>-<service name>`

```
[#96#root@ubuntu-1604 /opt/apps/harbor]$docker-compose ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
[#97#root@ubuntu-1604 /opt/apps/harbor]$
```

- `COMPOSE_FILE`: Specify the path to a Compose file. If not provided, Compose looks for a file named `docker-compose.yml` in the **current** directory and then each **parent** directory in succession until a file by that name is found. -- 对应 `-f` 选项
- `DOCKER_HOST`: Sets the URL of the **docker daemon**. As with the **Docker client**, defaults to `unix:///var/run/docker.sock`.


### docker-compose command

#### docker-compose down

**Stops** containers and **removes** containers, networks, volumes, and images created by up.

By default, the only things removed are:

- **Containers** for services defined in the Compose file
- **Networks** defined in the networks section of the Compose file
- The default network, if one is used

Networks and volumes defined as external are never removed.

#### docker-compose kill

```
Usage: kill [options] [SERVICE...]

Options:
-s SIGNAL         SIGNAL to send to the container. Default signal is SIGKILL.
```

Forces running containers to stop by sending a `SIGKILL` signal. Optionally the signal can be passed, for example:

```
docker-compose kill -s SIGINT
```

#### docker-compose logs

```
Usage: logs [options] [SERVICE...]

Options:
--no-color          Produce monochrome output.
-f, --follow        Follow log output
-t, --timestamps    Show timestamps
--tail="all"        Number of lines to show from the end of the logs
                    for each container.
```

#### docker-compose restart

```
Usage: restart [options] [SERVICE...]

Options:
-t, --timeout TIMEOUT      Specify a shutdown timeout in seconds. (default: 10)
```

Restarts all **stopped** and **running** services.

If you make changes to your `docker-compose.yml` configuration, these changes will not be reflected after running this command.

For example, changes to environment variables (which are added after a container is built, but before the container’s command is executed) will not be updated after restarting.

#### docker-compose up

```
Usage: up [options] [--scale SERVICE=NUM...] [SERVICE...]

Options:
    -d                         Detached mode: Run containers in the background,
                               print new container names.
                               Incompatible with --abort-on-container-exit.
    --no-color                 Produce monochrome output.
    --no-deps                  Don't start linked services.
    --force-recreate           Recreate containers even if their configuration
                               and image haven't changed.
                               Incompatible with --no-recreate.
    --no-recreate              If containers already exist, don't recreate them.
                               Incompatible with --force-recreate.
    --no-build                 Don't build an image, even if it's missing.
    --no-start                 Don't start the services after creating them.
    --build                    Build images before starting containers.
    --abort-on-container-exit  Stops all containers if any container was stopped.
                               Incompatible with -d.
    -t, --timeout TIMEOUT      Use this timeout in seconds for container shutdown
                               when attached or when containers are already
                               running. (default: 10)
    --remove-orphans           Remove containers for services not
                               defined in the Compose file
    --exit-code-from SERVICE   Return the exit code of the selected service container.
                               Implies --abort-on-container-exit.
    --scale SERVICE=NUM        Scale SERVICE to NUM instances. Overrides the `scale`
                               setting in the Compose file if present.
```

**Builds, (re)creates, starts, and attaches to containers for a service.**

Unless they are already running, this command also starts any linked services.

The `docker-compose up` command **aggregates** the output of each container (essentially running `docker-compose logs -f`). When the command exits, all containers are stopped. Running `docker-compose up -d` starts the containers in the background and leaves them running.

If there are existing containers for a service, and the service’s configuration or image was changed after the container’s creation, `docker-compose up` picks up the changes by **stopping** and **recreating** the containers (preserving mounted volumes). To prevent Compose from picking up changes, use the `--no-recreate` flag.

If you want to force Compose to **stop** and **recreate** all containers, use the `--force-recreate` flag.

If the process encounters an error, the exit code for this command is 1.

If the process is interrupted using `SIGINT` (ctrl + C) or `SIGTERM`, the containers are stopped, and the exit code is 0.

If `SIGINT` or `SIGTERM` is sent again during this shutdown phase, the running containers are killed, and the exit code is 2.


----------


## [Compose file versions and upgrading](https://docs.docker.com/compose/compose-file/compose-versioning/)

The Compose file is a YAML file defining **services**, **networks**, and **volumes** for a Docker application.

There are currently three versions of the Compose file format:

- **Version 1**, the legacy format. This is specified by **omitting** a version key at the root of the YAML.
- **Version 2.x**. This is specified with a version: '2' or version: '2.1', etc., entry at the root of the YAML.
- **Version 3.x**, the latest and **recommended** version, designed to be cross-compatible between Compose and the Docker Engine’s [swarm mode](https://docs.docker.com/engine/swarm/). This is specified with a version: '3' or version: '3.1', etc., entry at the root of the YAML.

The [Compatibility Matrix](https://docs.docker.com/compose/compose-file/compose-versioning/#compatibility-matrix) shows Compose file versions mapped to Docker Engine releases.

> Note: If you’re using multiple Compose files or extending services, each file must be of the same version - you cannot, for example, mix version 1 and 2 in a single project.

Several things differ depending on which version you use:

- The structure and permitted configuration keys
- The minimum Docker Engine version you must be running
- Compose’s behaviour with regards to **networking**


----------


## [Docker Compose FAQ](https://docs.docker.com/compose/faq/)

- Can I control service **startup order**?
- Why do my services **take 10 seconds** to recreate or stop?
- How do I run **multiple copies** of a Compose file on the same host?
- What’s the difference between `up`, `run`, and `start`?
- Can I use **json** instead of yaml for my Compose file?
- Should I include my code with `COPY`/`ADD` or a volume?
- Where can I find example compose files?


----------

## Harbor 项目中的使用

Harbor 项目是通过 `docker-compose` 进行管理的，相应的配置文件如下（如下内容基于 [docker-compose.tpl](https://github.com/vmware/harbor/blob/v1.2.0/make/docker-compose.tpl) 生成）：

```
[#94#root@ubuntu-1604 /opt/apps/harbor]$cat docker-compose.yml
version: '2'
services:
  log:
    # Specify the image to start the container from. 
    # Can either be a repository/tag or a partial image ID.
    # If the image does not exist, Compose attempts to pull it, unless you have also specified build, in which case it builds it using the specified options and tags it with the specified tag.
    image: vmware/harbor-log:v1.2.0
    # Specify a custom container name, rather than a generated default name.
    container_name: harbor-log
    # no is the default restart policy, and it will not restart a container under any circumstance. 
    # When always is specified, the container always restarts. 
    # The on-failure policy restarts a container if the exit code indicates an on-failure error.
    restart: always
    # Mount paths or named volumes, optionally specifying a path on the host machine (HOST:CONTAINER), or an access mode (HOST:CONTAINER:ro). 
    # For version 2 files, named volumes need to be specified with the top-level volumes key.
    # You can mount a relative path on the host, which will expand relative to the directory of the Compose configuration file being used. 
    # Relative paths should always begin with . or ...
    volumes:
      # 关于 "z" 模式的说明详见下面
      # 此处挂载目的为：将 harbor-log 容器中存放 syslog 日志的目录 /var/log/docker/ 通过卷暴露到宿主机的 /var/log/harbor/ 目录中
      - /var/log/harbor/:/var/log/docker/:z
    # Expose ports. 
    # Either specify both ports (HOST:CONTAINER), or just the container port (a random host port will be chosen).
    ports:
      # 在 lo 口上对外暴露可访问的 port ，映射到容器内 syslogd 的 port 用于进行日志记录
      - 127.0.0.1:1514:514
    # Networks to join, referencing entries under the top-level networks key.
    # Version 2 file format and up. Replaces the version 1 net option.
    networks:
      - harbor
  registry:
    image: vmware/registry:2.6.2-photon
    container_name: registry
    restart: always
    volumes:
      - /data/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
    networks:
      - harbor
    # Add environment variables. You can use either an array or a dictionary. 
    # Any boolean values; true, false, yes no, need to be enclosed in quotes to ensure they are not converted to True or False by the YML parser.
    # Environment variables with only a key are resolved to their values on the machine Compose is running on, which can be helpful for secret or host-specific values.
    environment:
      - GODEBUG=netdns=cgo
    # Override the default command.
    # The command can also be a list, in a manner similar to dockerfile
    command:
      ["serve", "/etc/registry/config.yml"]
    # Version 2 file format and up.
    # Express dependency between services, which has two effects:
    # - docker-compose up will start services in dependency order. 
    # - docker-compose up SERVICE will automatically include SERVICE’s dependencies. 
    # Note: depends_on will not wait for log to be “ready” before starting registry - only until it has been started. 
    depends_on:
      - log
    # Logging configuration for the service.
    # https://docs.docker.com/engine/admin/logging/syslog/
    logging:
      # The driver name specifies a logging driver for the service’s containers, as with the --log-driver option for docker run 
      # The default value is json-file. can be set to "json-file", "syslog" and "none"
      # Note: Only the json-file and journald drivers make the logs available directly from docker-compose up and docker-compose logs. Using any other driver will not print any logs.
      driver: "syslog"
      # Specify logging options for the logging driver with the options key, as with the --log-opt option for docker run.
      options:
        # The address of an external syslog server. 
        # 基于 syslog 驱动访问外部 syslogd 服务器，这里其实是基于 lo 完成的本机跨容器通信
        syslog-address: "tcp://127.0.0.1:1514"
        # https://docs.docker.com/engine/admin/logging/log_tags/
        # The tag log option specifies how to format a tag that identifies the container’s log messages. 
        # A string that is appended to the APP-NAME in the syslog message. By default, the system uses the first 12 characters of the container ID.
        # The format of Docker’s syslog driver implements:
        # TIMESTAMP SP HOSTNAME SP APP-NAME SP PROCID SP MSGID
        tag: "registry"
  mysql:
    image: vmware/harbor-db:v1.2.0
    container_name: harbor-db
    restart: always
    volumes:
      - /data/database:/var/lib/mysql:z
    networks:
      - harbor
    # Add environment variables from a file. Can be a single value or a list.
    # If you have specified a Compose file with docker-compose -f FILE, paths in env_file are relative to the directory that file is in.
    # Environment variables declared in the environment section override these values – this holds true even if those values are empty or undefined.
    # Compose expects each line in an env file to be in VAR=VAL format. Lines beginning with # (i.e. comments) are ignored, as are blank lines.
    # Keep in mind that the order of files in the list is significant in determining the value assigned to a variable that shows up more than once. The files in the list are processed from the top down.
    env_file:
      - ./common/config/db/env
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "mysql"
  adminserver:
    image: vmware/harbor-adminserver:v1.2.0
    container_name: harbor-adminserver
    env_file:
      - ./common/config/adminserver/env
    restart: always
    volumes:
      - /data/config/:/etc/adminserver/config/:z
      - /data/secretkey:/etc/adminserver/key:z
      - /data/:/data/:z
    networks:
      - harbor
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "adminserver"
  ui:
    image: vmware/harbor-ui:v1.2.0
    container_name: harbor-ui
    env_file:
      - ./common/config/ui/env
    restart: always
    volumes:
      - ./common/config/ui/app.conf:/etc/ui/app.conf:z
      - ./common/config/ui/private_key.pem:/etc/ui/private_key.pem:z
      - /data/secretkey:/etc/ui/key:z
      - /data/ca_download/:/etc/ui/ca/:z
      - /data/psc/:/etc/ui/token/:z
      ## add two lines as below ##
      - ../src/ui/static/vendors/swagger-ui-2.1.4/dist:/harbor/static/vendors/swagger
      - ../src/ui/static/resources/yaml/swagger.yaml:/harbor/static/resources/yaml/swagger.yaml
    networks:
      - harbor
    depends_on:
      - log
      - adminserver
      - registry
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "ui"
  jobservice:
    image: vmware/harbor-jobservice:v1.2.0
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    volumes:
      - /data/job_logs:/var/log/jobs:z
      - ./common/config/jobservice/app.conf:/etc/jobservice/app.conf:z
      - /data/secretkey:/etc/jobservice/key:z
    networks:
      - harbor
    depends_on:
      - ui
      - adminserver
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  proxy:
    image: vmware/nginx-photon:1.11.13
    container_name: nginx
    restart: always
    volumes:
      - ./common/config/nginx:/etc/nginx:z
    networks:
      - harbor
    ports:
      - 80:80
      - 443:443
      - 4443:4443
    depends_on:
      - mysql
      - registry
      - ui
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
# The top-level networks key lets you specify networks to be created. 
# The default driver depends on how the Docker Engine you’re using is configured, but in most instances it will be bridge on a single host and overlay on a Swarm.
networks:
  harbor:
    # If set to true, specifies that this network has been created outside of Compose. docker-compose up will not attempt to create it, and will raise an error if it doesn’t exist.
    # external cannot be used in conjunction with other network configuration keys (driver, driver_opts, group_add, ipam, internal).
    external: false

[#95#root@ubuntu-1604 /opt/apps/harbor]$
```

### 关于 volumes 的 "z"/"Z" 设置

在 https://docs.docker.com/engine/reference/commandline/run/#mount-volumes-from-container-volumes-from 中有

> **Labeling systems** like SELinux require that proper labels are placed on volume content mounted into a container. Without a label, the security system might prevent the processes running inside the container from using the content. By default, Docker does not change the labels set by the OS.
>
> To change the label in the container context, you can add either of two suffixes `:z` or `:Z` to the volume mount. These suffixes tell Docker to relabel file objects on the shared volumes. The `z` option tells Docker that two containers **share** the volume content. As a result, Docker labels the content with a shared content label. Shared volume labels allow all containers to **read**/**write** content. The `Z` option tells Docker to label the content with a **private unshared** label. Only the current container can use a private volume.

在 https://github.com/docker/compose/releases/tag/1.4.0 中有

> When mounting volumes with the `volumes` option, you can now pass in any mode supported by the daemon, not just `:ro` or `:rw`. For example, **SELinux** users can pass `:z` or `:Z`.

在 [Using Volumes with Docker can Cause Problems with SELinux](http://www.projectatomic.io/blog/2015/06/using-volumes-with-docker-can-cause-problems-with-selinux/) 中有

> When using SELinux for controlling processes within a container, you need to make sure any content that gets volume mounted into the container is **readable**, and potentially **writable**, depending on the use case.
> 
> By default, Docker container processes run with the `system_u:system_r:svirt_lxc_net_t:s0` label. The `svirt_lxc_net_t` type is allowed to **read**/**execute** most content under `/usr`, but it is not allowed to use most other types on the system.
>
> If you want to volume mount content under `/var`, for example, into a container you need to set the labels on this content.
> 
> ```
> man docker-run
> ...
> When  using  SELinux,  be  aware that the host has no knowledge of container SELinux policy. Therefore, in the above example, if SELinux policy  is enforced,  the /var/db directory is not  writable to the container. A "Permission Denied" message will occur and an avc: message in the host's syslog.
> 
> To  work  around  this, at time of writing this man page, the following command needs to be run in order for the  proper  SELinux  policy  type label to be attached to the host directory:
> 
> # chcon -Rt svirt_sandbox_file_t /var/db
> ```
>
> This got easier recently since Docker finally merged a patch which will be showing up in **docker-1.7**. This patch adds support for `z` and `Z` as options on the volume mounts (`-v`).
> 
> If you volume mount a image with `-v /SOURCE:/DESTINATION:z` docker will automatically relabel the content for you to `s0`. If you volume mount with a `Z`, then the label will be specific to the container, and not be able to be shared between containers.

