# Docker 常见问题汇总

## 内容目录

- [/usr/bin/ 下各种 docker 命令之间的关系](#usrbin-下各种-docker-命令之间的关系)
- [docker-container-shim](#docker-container-shim)
- [docker-containerd-ctr](#docker-containerd-ctr)
- [不同 Docker 版本生成的容器进程树对比](#不同-docker-版本生成的容器进程树对比)
- [Docker Engine](#docker-engine)
- [Pause Container](#pause-container)
- [Container Runtime](#container-runtime)
    - [CRI](#cri)
    - [Container Runtime CLI](#container-runtime-cli)
- [Kubernetes Node Performance Benchmark](#kubernetes-node-performance-benchmark)
- [Containerd](#containerd)
    - [为什么需要 Containerd](#为什么需要-containerd)
    - [Containerd 提供了什么](#containerd-提供了什么)
    - [Containerd 和 Docker 之间的关系](#containerd-和-docker-之间的关系)
    - [Containerd 和 OCI 和 runC 之间的关系](#containerd-和-oci-和-runc-之间的关系)
    - [Containerd 和 Container Orchestration Systems 之间的关系](#containerd-和-container-orchestration-systems-之间的关系)
    - [The Kubernetes containerd Integration Architecture](#the-kubernetes-containerd-integration-architecture)
    - [cri-containerd](#cri-containerd)
    - [How cri-containerd Works](#how-cri-containerd-works)
    - [Performance Improvement](#performance-improvement)
    - [Containerd 和 Docker Engine 和 Kubernetes 的关系](#containerd-和-docker-engine-和-kubernetes-的关系)
    - [Summary](#summary)
    - [Setup a Kubernetes cluster using containerd as the container runtime](#setup-a-kubernetes-cluster-using-containerd-as-the-container-runtime)
- [Image Formats](#image-formats)
- [Starting a Container](#starting-a-container)
- [Pulling an Image](#pulling-an-image)
- [What is a container manifest](#what-is-a-container-manifest)
- [What are Docker image layers?](#what-are-docker-image-layers)
- [/var/run/docker.sock](#varrundockersock)
- [--privileged 特权容器](#--privileged-特权容器)
- [alpine 是什么](#alpine-是什么)
- [parent image v.s. base image](#parent-image-vs-base-image)
- [What are Docker `<none>:<none>` images?](#what-are-docker-nonenone-images)
- [镜像源变更](#镜像源变更)
- [DNS 变更](#dns-变更)
- [时区变更](#时区变更)

## /usr/bin/ 下各种 docker 命令之间的关系

```
[#64#root@ubuntu-1604 ~]$docker --version
Docker version 18.03.1-ce, build 9ee9f40
[#65#root@ubuntu-1604 ~]$

[#62#root@ubuntu-1604 ~]$ll /usr/bin/ | grep docker
-rwxr-xr-x  1 root   root     38258864 Apr 26 15:18 docker*
-rwxr-xr-x  1 root   root     22774816 Apr 26 15:15 docker-containerd*
-rwxr-xr-x  1 root   root     18406688 Apr 26 15:15 docker-containerd-ctr*
-rwxr-xr-x  1 root   root      4324256 Apr 26 15:15 docker-containerd-shim*
-rwxr-xr-x  1 root   root     81682968 Apr 26 15:17 dockerd*
-rwxr-xr-x  1 root   root       866392 Apr 26 15:13 docker-init*
-rwxr-xr-x  1 root   root      3329080 Apr 26 15:13 docker-proxy*
-rwxr-xr-x  1 root   root     10960720 Apr 26 15:13 docker-runc*
[#63#root@ubuntu-1604 ~]$
```

关系：

```
docker     docker-containerd-ctr
  |         |
  V         V
dockerd -> docker-containerd -+---> docker-containerd-shim -> runc -> container process
                              |-- > docker-containerd-shim -> runc -> container process
                              +-- > docker-containerd-shim -> runc -> container process
```

小结：

- docker (cli) 和 dockerd (deamon) 构成 docker engine ；
- dockerd 会启动 docker-containerd 子进程；
- docker-containerd 会启动 docker-containerd-shim 子进程；
- docker-containerd-shim 会指定 runc 为 runtime ，并直接调用 runc 包中的函数启动相应的 container ；
- runc 真正控制 container 的生命周期；
- 每个 container 都有自己对应的 docker-containerd-shim 进程；
- 用户进程由 runc 进程启动；

## docker-container-shim

- shim for container lifecycle and reconnection.

Ref:

- https://github.com/containerd/containerd/blob/1ac546b3c4a3331a9997427052d1cb9888a2f3ef/reports/2017-01-27.md#L31

## docker-containerd-ctr

- docker-containerd-ctr 是 containerd 提供的一个开发和调试用的 CLI 工具；
- docker-containerd-ctr 即 containerd CLI ，常简写为 ctr ；

使用示例

```
docker-containerd-ctr --address unix:///var/run/docker/libcontainerd/docker-containerd.sock containers | grep 67292a357097
```

## 不同 Docker 版本生成的容器进程树对比

### 17.09.0-ce

```
root@harbor-service-new-stag:~# ls /usr/bin |grep docker
docker
docker-containerd
docker-containerd-ctr
docker-containerd-shim
docker-init
docker-proxy
docker-runc
dockerd
root@harbor-service-new-stag:~#
```

```sh
root@harbor-service-new-stag:~# docker version
Client:
 Version:      17.09.0-ce
 API version:  1.32
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:42:18 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.09.0-ce
 API version:  1.32 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   afdb6d4
 Built:        Tue Sep 26 22:40:56 2017
 OS/Arch:      linux/amd64
 Experimental: false
root@harbor-service-new-stag:~#
```

完整进程树信息

```sh
    1  1207  1207  1207 ?           -1 Ssl      0 5568:48 /usr/bin/dockerd --insecure-registry 10.1.0.13 -H fd://
 1207  1358  1358  1358 ?           -1 Ssl      0  90:40  \_ docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc
 1358  2265  2265  1358 ?           -1 Sl       0   0:04  |   \_ docker-containerd-shim 9a3cdcef11b22190b69558daae4b330c3c6ccd7b2f52ee57e748475586a2b5ee /var/run/docker/libcontainerd/9a3cdcef11b22190b69558daae4b330c3c6ccd7b2f52ee57e748475586a2b5ee docker-runc
 2265  2283  2283  2283 ?           -1 Ss       0   0:00  |   |   \_ /bin/sh -c crond && rm -f /var/run/rsyslogd.pid && rsyslogd -n
 2283  2357  2357  2357 ?           -1 Ss       0   0:14  |   |       \_ crond
 2283  2359  2283  2283 ?           -1 Sl       0 1002:15  |   |       \_ rsyslogd -n
 1358  2412  2412  1358 ?           -1 Sl       0   0:04  |   \_ docker-containerd-shim e4833a0bb3404b38ed7552546d844e71773b3345e84e4221c07b2c2e83287cd8 /var/run/docker/libcontainerd/e4833a0bb3404b38ed7552546d844e71773b3345e84e4221c07b2c2e83287cd8 docker-runc
 2412  2430  2430  2430 ?           -1 Ssl    999 953:58  |   |   \_ mysqld
 1358  2537  2537  1358 ?           -1 Sl       0 5395:30  |   \_ docker-containerd-shim 97fdd84906edf0534a03c042d76b279f1a8bab4a6c7aaad5e59f330654c15cad /var/run/docker/libcontainerd/97fdd84906edf0534a03c042d76b279f1a8bab4a6c7aaad5e59f330654c15cad docker-runc
 2537  2565  2565  2565 ?           -1 Ssl      0 8999:20  |   |   \_ registry serve /etc/registry/config.yml
 1358  2625  2625  1358 ?           -1 Sl       0   7:09  |   \_ docker-containerd-shim 75107f9afe381d462f4bf7a462beefa202383f39649872fc20e99c9666e97f41 /var/run/docker/libcontainerd/75107f9afe381d462f4bf7a462beefa202383f39649872fc20e99c9666e97f41 docker-runc
 2625  2654  2654  2654 ?           -1 Ssl      0   7:13  |   |   \_ /harbor/harbor_adminserver
 1358  2817  2817  1358 ?           -1 Sl       0 502:58  |   \_ docker-containerd-shim f2c44dc5d63bef45b19c43b7ed07e14fb9d3941027d3cb3d2a944f4ad04cdfe9 /var/run/docker/libcontainerd/f2c44dc5d63bef45b19c43b7ed07e14fb9d3941027d3cb3d2a944f4ad04cdfe9 docker-runc
 2817  2836  2836  2836 ?           -1 Ssl      0 4805:11  |   |   \_ /harbor/harbor_ui
 1358  3054  3054  1358 ?           -1 Sl       0 107:01  |   \_ docker-containerd-shim c942b0dd65c61efc26cf7d13a15b37b21ce7250e7883220465860986168c69c7 /var/run/docker/libcontainerd/c942b0dd65c61efc26cf7d13a15b37b21ce7250e7883220465860986168c69c7 docker-runc
 3054  3089  3089  3089 ?           -1 Ssl      0 335:21  |   |   \_ /harbor/harbor_jobservice
 1358  3059  3059  1358 ?           -1 Sl       0 325:58  |   \_ docker-containerd-shim 8ef57fdec0305471c1201745daad4a0abefdf10fb0dbdc67d854343add853419 /var/run/docker/libcontainerd/8ef57fdec0305471c1201745daad4a0abefdf10fb0dbdc67d854343add853419 docker-runc
 3059  3097  3097  3097 ?           -1 Ss       0   0:00  |       \_ nginx: master process nginx -g daemon off;
 3097  3203  3097  3097 ?           -1 S    65534   0:51  |           \_ nginx: worker process
 3097  3204  3097  3097 ?           -1 S    65534   0:50  |           \_ nginx: worker process
 3097  3205  3097  3097 ?           -1 S    65534   6:47  |           \_ nginx: worker process
 3097  3206  3097  3097 ?           -1 S    65534 218:15  |           \_ nginx: worker process
 3097  3207  3097  3097 ?           -1 S    65534   0:54  |           \_ nginx: worker process
 3097  3208  3097  3097 ?           -1 S    65534  52:59  |           \_ nginx: worker process
 3097  3209  3097  3097 ?           -1 S    65534   1:50  |           \_ nginx: worker process
 3097  3210  3097  3097 ?           -1 S    65534  78:00  |           \_ nginx: worker process
 1207  2260  1207  1207 ?           -1 Sl       0 1062:51  \_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 1514 -container-ip 172.18.0.2 -container-port 514
 1207  2976  1207  1207 ?           -1 Sl       0   0:04  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 4443 -container-ip 172.18.0.7 -container-port 4443
 1207  3005  1207  1207 ?           -1 Sl       0   0:38  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 443 -container-ip 172.18.0.7 -container-port 443
 1207  3048  1207  1207 ?           -1 Sl       0   0:04  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.7 -container-port 80
```

简化后的结构

```sh
/usr/bin/dockerd
\_ docker-containerd
|   \_ docker-containerd-shim
|   |   \_ /bin/sh -c crond && rm -f /var/run/rsyslogd.pid && rsyslogd -n
|   \_ docker-containerd-shim
|   |   \_ mysqld
|   \_ docker-containerd-shim
|   |   \_ registry serve /etc/registry/config.yml
|   \_ docker-containerd-shim
|   |   \_ /harbor/harbor_adminserver
|   \_ docker-containerd-shim
|   |   \_ /harbor/harbor_ui
|   \_ docker-containerd-shim
|   |   \_ /harbor/harbor_jobservice
|   \_ docker-containerd-shim
|       \_ nginx: master process nginx -g daemon off;
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
|           \_ nginx: worker process
\_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 1514 -container-ip 172.18.0.2 -container-port 514
\_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 4443 -container-ip 172.18.0.7 -container-port 4443
\_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 443 -container-ip 172.18.0.7 -container-port 443
\_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.7 -container-port 80
```

```sh
root@harbor-service-new-stag:~# docker ps
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
c942b0dd65c6        vmware/harbor-jobservice:v1.2.2    "/harbor/harbor_jo..."   3 months ago        Up 3 months                                                                            harbor-jobservice
8ef57fdec030        vmware/nginx-photon:1.11.13        "nginx -g 'daemon ..."   3 months ago        Up 3 months         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
f2c44dc5d63b        vmware/harbor-ui:v1.2.2            "/harbor/harbor_ui"      3 months ago        Up 3 months                                                                            harbor-ui
75107f9afe38        vmware/harbor-adminserver:v1.2.2   "/harbor/harbor_ad..."   3 months ago        Up 3 months                                                                            harbor-adminserver
97fdd84906ed        vmware/registry:2.6.2-photon       "/entrypoint.sh se..."   3 months ago        Up 3 months         5000/tcp                                                           registry
e4833a0bb340        vmware/harbor-db:v1.2.2            "docker-entrypoint..."   3 months ago        Up 3 months         3306/tcp                                                           harbor-db
9a3cdcef11b2        vmware/harbor-log:v1.2.2           "/bin/sh -c 'crond..."   3 months ago        Up 3 months         127.0.0.1:1514->514/tcp                                            harbor-log
root@harbor-service-new-stag:~#

root@harbor-service-new-stag:/opt/apps/harbor# docker-compose ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
root@harbor-service-new-stag:/opt/apps/harbor#
```

### 18.03.1-ce

```
[#48#root@ubuntu-1604 ~]$ls /usr/bin |grep docker
docker
docker-containerd
docker-containerd-ctr
docker-containerd-shim
dockerd
docker-init
docker-proxy
docker-runc
[#49#root@ubuntu-1604 ~]$
```

```sh
[#14#root@ubuntu-1604 /opt/apps/harbor]$docker version
Client:
 Version:      18.03.1-ce
 API version:  1.37
 Go version:   go1.9.5
 Git commit:   9ee9f40
 Built:        Thu Apr 26 07:17:20 2018
 OS/Arch:      linux/amd64
 Experimental: false
 Orchestrator: swarm

Server:
 Engine:
  Version:      18.03.1-ce
  API version:  1.37 (minimum version 1.12)
  Go version:   go1.9.5
  Git commit:   9ee9f40
  Built:        Thu Apr 26 07:15:30 2018
  OS/Arch:      linux/amd64
  Experimental: false
[#15#root@ubuntu-1604 /opt/apps/harbor]$
```

完整进程树信息

```sh
    1  1165  1165  1165 ?           -1 Ssl      0   6:40 /usr/bin/dockerd -H fd://
 1165  1318  1318  1318 ?           -1 Ssl      0   7:38  \_ docker-containerd --config /var/run/docker/containerd/containerd.toml
 1318  4508  4508  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/67b25c6b3a33eefcb68abbe521f96947fcbe1aba52290e63711aaf7775314c0b -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 4508  4524  4524  4524 ?           -1 Ss       0   0:00  |   |   \_ /bin/bash /usr/local/bin/start.sh
 4524  4574  4574  4574 ?           -1 Ss       0   0:00  |   |       \_ crond
 4524  4579  4524  4524 ?           -1 S        0   0:00  |   |       \_ sudo -u #10000 -E rsyslogd -n
 4579  4582  4524  4524 ?           -1 Sl   10000   0:00  |   |           \_ rsyslogd -n
 1318  4640  4640  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/2190541c1c0fec236b1be38710a0e56c1c07eaa0705bdaaadb982d849d6c480e -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 4640  4737  4737  4737 ?           -1 Ss       0   0:00  |   |   \_ sudo -u mysql -E /usr/local/bin/docker-entrypoint.sh mysqld
 4737  5105  4737  4737 ?           -1 Sl   10000   0:00  |   |       \_ mysqld
 1318  4717  4717  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/6ca0856685a7a86dfa62088b8f8936948fbe3211b194275b6d28188fc4fe9db8 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 4717  4875  4875  4875 ?           -1 Ss       0   0:00  |   |   \_ /bin/sh /harbor/start.sh
 4875  5145  4875  4875 ?           -1 S        0   0:00  |   |       \_ sudo -E -u #10000 /harbor/harbor_adminserver
 5145  5154  4875  4875 ?           -1 Sl   10000   0:00  |   |           \_ /harbor/harbor_adminserver
 1318  4765  4765  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/8c348b49787853eec39d93173c85001cc5131f4043d7cb3ab5a3c00b80a75607 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 4765  4867  4867  4867 ?           -1 Ss       0   0:00  |   |   \_ sudo -u redis redis-server /etc/redis.conf
 4867  5082  4867  4867 ?           -1 Sl     999   0:00  |   |       \_ redis-server *:6379
 1318  4791  4791  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/ea0ca3ad857dbd16cb147c69e5f9f6b60915e1af5418eb73c28942ef60850dfe -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 4791  4903  4903  4903 ?           -1 Ss       0   0:00  |   |   \_ /bin/sh /entrypoint.sh serve /etc/registry/config.yml
 4903  5081  4903  4903 ?           -1 S        0   0:00  |   |       \_ sudo -E -u #10000 registry serve /etc/registry/config.yml
 5081  5083  4903  4903 ?           -1 Sl   10000   0:00  |   |           \_ registry serve /etc/registry/config.yml
 1318  5175  5175  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bed8d048a618a3326f11e402eb1df677870e00692654824c30a6456446c0a1c5 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 5175  5230  5230  5230 ?           -1 Ss       0   0:00  |   |   \_ /bin/sh /harbor/start.sh
 5230  5301  5230  5230 ?           -1 S        0   0:00  |   |       \_ sudo -E -u #10000 /harbor/harbor_ui
 5301  5307  5230  5230 ?           -1 Sl   10000   0:00  |   |           \_ /harbor/harbor_ui
 1318  5490  5490  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/4df3d3e341b4290cb8f4fc7e0602bbf03460779bf9c5003fadccd5ed0f993634 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 5490  5513  5513  5513 ?           -1 Ss       0   0:00  |   |   \_ nginx: master process nginx -g daemon off;
 5513  5618  5513  5513 ?           -1 S    65534   0:00  |   |       \_ nginx: worker process
 5513  5619  5513  5513 ?           -1 S    65534   0:00  |   |       \_ nginx: worker process
 1318  5893  5893  1318 ?           -1 Sl       0   0:00  |   \_ docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/40b7e9636454340db81495c0953fe246d5b3824061f4e0bb555bbe044cb3837a -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
 5893  5938  5938  5938 ?           -1 Ss       0   0:00  |       \_ /bin/sh /harbor/start.sh
 5938  5995  5938  5938 ?           -1 S        0   0:00  |           \_ sudo -E -u #10000 /harbor/harbor_jobservice -c /etc/jobservice/config.yml
 5995  6000  5938  5938 ?           -1 Sl   10000   0:00  |               \_ /harbor/harbor_jobservice -c /etc/jobservice/config.yml
 1165  4477  1165  1165 ?           -1 Sl       0   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 1514 -container-ip 172.18.0.2 -container-port 10514
 1165  5431  1165  1165 ?           -1 Sl       0   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 4443 -container-ip 172.18.0.9 -container-port 4443
 1165  5467  1165  1165 ?           -1 Sl       0   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 443 -container-ip 172.18.0.9 -container-port 443
 1165  5484  1165  1165 ?           -1 Sl       0   0:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.9 -container-port 80
```

简化后的结构

```sh
/usr/bin/dockerd -H fd://
 \_ docker-containerd --config /var/run/docker/containerd/containerd.toml
 |   \_ docker-containerd-shim                                                                                                                                                                                                                                                   |   |   \_ /bin/bash /usr/local/bin/start.sh
 |   \_ docker-containerd-shim
 |   |   \_ sudo -u mysql -E /usr/local/bin/docker-entrypoint.sh mysqld
 |   |       \_ mysqld
 |   \_ docker-containerd-shim
 |   |   \_ /bin/sh /harbor/start.sh                                                                                                                                                                                                                                             |   |       \_ sudo -E -u #10000 /harbor/harbor_adminserver
 |   |           \_ /harbor/harbor_adminserver
 |   \_ docker-containerd-shim
 |   |   \_ sudo -u redis redis-server /etc/redis.conf                                                                                                                                                                                                                           |   |       \_ redis-server *:6379
 |   \_ docker-containerd-shim
 |   |   \_ /bin/sh /entrypoint.sh serve /etc/registry/config.yml
 |   |       \_ sudo -E -u #10000 registry serve /etc/registry/config.yml
 |   |           \_ registry serve /etc/registry/config.yml                                                                                                                                                                                                                      |   \_ docker-containerd-shim
 |   |   \_ /bin/sh /harbor/start.sh
 |   |       \_ sudo -E -u #10000 /harbor/harbor_ui
 |   |           \_ /harbor/harbor_ui                                                                                                                                                                                                                                            |   \_ docker-containerd-shim
 |   |   \_ nginx: master process nginx -g daemon off;
 |   |       \_ nginx: worker process
 |   |       \_ nginx: worker process
 |   \_ docker-containerd-shim                                                                                                                                                                                                                                                   |       \_ /bin/sh /harbor/start.sh
 |           \_ sudo -E -u #10000 /harbor/harbor_jobservice -c /etc/jobservice/config.yml
 |               \_ /harbor/harbor_jobservice -c /etc/jobservice/config.yml
 \_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 1514 -container-ip 172.18.0.2 -container-port 10514
 \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 4443 -container-ip 172.18.0.9 -container-port 4443                                                                                                                                                              \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 443 -container-ip 172.18.0.9 -container-port 443
 \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.9 -container-port 80
```

```
[#65#root@ubuntu-1604 ~]$docker ps
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS                 PORTS                                                              NAMES
40b7e9636454        vmware/harbor-jobservice:v1.5.0        "/harbor/start.sh"       2 hours ago         Up 2 hours                                                                                harbor-jobservice
4df3d3e341b4        vmware/nginx-photon:v1.5.0             "nginx -g 'daemon of…"   2 hours ago         Up 2 hours (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
bed8d048a618        vmware/harbor-ui:v1.5.0                "/harbor/start.sh"       2 hours ago         Up 2 hours (healthy)                                                                      harbor-ui
ea0ca3ad857d        vmware/registry-photon:v2.6.2-v1.5.0   "/entrypoint.sh serv…"   2 hours ago         Up 2 hours (healthy)   5000/tcp                                                           registry
8c348b497878        vmware/redis-photon:v1.5.0             "docker-entrypoint.s…"   2 hours ago         Up 2 hours             6379/tcp                                                           redis
6ca0856685a7        vmware/harbor-adminserver:v1.5.0       "/harbor/start.sh"       2 hours ago         Up 2 hours (healthy)                                                                      harbor-adminserver
2190541c1c0f        vmware/harbor-db:v1.5.0                "/usr/local/bin/dock…"   2 hours ago         Up 2 hours (healthy)   3306/tcp                                                           harbor-db
67b25c6b3a33        vmware/harbor-log:v1.5.0               "/bin/sh -c /usr/loc…"   2 hours ago         Up 2 hours (healthy)   127.0.0.1:1514->10514/tcp                                          harbor-log
[#66#root@ubuntu-1604 ~]$


[#67#root@ubuntu-1604 /opt/apps/harbor]$docker-compose ps
       Name                     Command                  State                                    Ports
-------------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/start.sh                 Up (healthy)
harbor-db            /usr/local/bin/docker-entr ...   Up (healthy)   3306/tcp
harbor-jobservice    /harbor/start.sh                 Up
harbor-log           /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp
harbor-ui            /harbor/start.sh                 Up (healthy)
nginx                nginx -g daemon off;             Up (healthy)   0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
redis                docker-entrypoint.sh redis ...   Up             6379/tcp
registry             /entrypoint.sh serve /etc/ ...   Up (healthy)   5000/tcp
[#68#root@ubuntu-1604 /opt/apps/harbor]$
```

## Docker Engine

Ref: https://docs.docker.com/engine/docker-overview/#docker-

Docker Engine is a client-server application with these major components:

- A **server** which is a type of long-running program called a daemon process (the `dockerd` command).
- A **REST API** which specifies interfaces that programs can use to talk to the daemon and instruct it what to do.
- A command line interface (CLI) client (the `docker` command).

The CLI uses the Docker REST API to control or interact with the Docker daemon through scripting or direct CLI commands. Many other Docker applications use the underlying API and CLI.

The daemon creates and manages Docker objects, such as images, containers, networks, and volumes.

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/docker%20engine%20major%20components.png)

Docker Engine's all components

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Docker%20Engine%20Components.png)

## Pause Container

https://www.ianlewis.org/en/almighty-pause-container

## Container Runtime

A container runtime is software that **executes containers** and **manages container images** on a node.

- Today, the most widely known container runtime is `Docker`.
- But there are other container runtimes in the ecosystem, such as [`rkt`](https://coreos.com/rkt/), [`containerd`](https://containerd.io/), and [`lxd`](https://linuxcontainers.org/lxd/).
- Containerd is an OCI compliant core container runtime designed to be embedded into larger systems.

### CRI

- CRI is short for **Container Runtime Interface**.
- [Kubernetes 1.5 introduced an internal plugin API named Container Runtime Interface (CRI)](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/) to provide easy access to different container runtimes. 
- CRI enables Kubernetes to use a variety of container runtimes without the need to recompile. 
- In theory, Kubernetes could use any container runtime that implements CRI to manage pods, containers and container images.
- Over the past 6 months, engineers from Google, Docker, IBM, ZTE, and ZJU have worked to implement CRI for containerd. The project is called [cri-containerd](https://github.com/containerd/cri), which had its [feature complete v1.0.0-alpha.0 release](https://github.com/containerd/cri/releases/tag/v1.0.0-alpha.0) on September 25, 2017. 
- With cri-containerd, users can run Kubernetes clusters using containerd as the underlying runtime without Docker installed.
- Docker CRI implementation - [`dockershim`](https://github.com/kubernetes/kubernetes/tree/v1.10.2/pkg/kubelet/dockershim)
- The Docker 18.03 CE integration uses the `dockershim`.
- The containerd 1.1 integration uses the CRI plugin built into containerd.
- The containerd CRI plugin is an open source [github project](https://github.com/containerd/cri) within containerd.
- With [containerd/cri](https://github.com/containerd/cri), you could run Kubernetes using containerd as the container runtime. 
- Containerd is used by Docker, Kubernetes CRI, and a few other projects. 

### Container Runtime CLI

- When using Docker as the container runtime for Kubernetes, system administrators sometimes login to the Kubernetes node to run **Docker commands (Docker CLI)** for collecting system and/or application information. 
- For **containerd** and all other CRI-compatible container runtimes, e.g. `dockershim`, we recommend using `crictl` as a replacement CLI over the Docker CLI for troubleshooting pods, containers, and container images on Kubernetes nodes.
- `crictl` is a tool providing a similar experience to the **Docker CLI** for Kubernetes node troubleshooting and crictl works consistently across all CRI-compatible container runtimes. It is hosted in the [kubernetes-incubator/cri-tools](https://github.com/kubernetes-sigs/cri-tools) repository
- `crictl` is designed to resemble the **Docker CLI** to offer a better transition experience for users, but it is not exactly the same. 
    - **Limited Scope - crictl is a Troubleshooting Tool**
        - The scope of `crictl` is limited to troubleshooting, it is not a replacement to `docker` or `kubectl`. Docker’s CLI provides a rich set of commands, making it a very useful development tool. But it is not the best fit for troubleshooting on Kubernetes nodes. Some Docker commands are not useful to Kubernetes, such as `docker network` and `docker build`; and some may even break the system, such as `docker rename`. crictl provides just enough commands for node troubleshooting, which is arguably safer to use on production nodes.
    - **Kubernetes Oriented**
        - `crictl` offers a more kubernetes-friendly view of containers. Docker CLI lacks core Kubernetes concepts, e.g. pod and namespace, so it can’t provide a clear view of containers and pods. One example is that `docker ps` shows somewhat obscure, long Docker container names, and shows pause containers and application containers together. 
        - However, pause containers are a pod implementation detail, where one pause container is used for each pod, and thus should not be shown when listing containers that are members of pods.
        - `crictl`, by contrast, is designed for Kubernetes. It has different sets of commands for pods and containers. For example, `crictl pods` lists pod information, and `crictl ps` only lists application container information. All information is well formatted into table columns.
        - As another example, crictl pods includes a –namespace option for filtering pods by the namespaces specified in Kubernetes.
    - [`crictl` user guide](https://github.com/containerd/cri/blob/master/docs/crictl.md)
    - [`crictl` demo video](https://asciinema.org/a/179047)


## Kubernetes Node Performance Benchmark

Part of [Kubernetes node e2e test](https://github.com/kubernetes/community/blob/master/contributors/devel/e2e-node-tests.md)

## Containerd

![Containd Architecture - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%20Architecture%20-%201.png)

![Containd Architecture - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%20Architecture%20-%202.png)

> containerd is an industry-standard container runtime with an emphasis on simplicity, robustness and portability. It is available as a daemon for Linux and Windows, which can manage the complete container lifecycle of its host system: image transfer and storage, container execution and supervision, low-level storage and network attachments, etc.

- containerd 是工业级 container runtime ；
- containerd 作为 daemon 程序运行，用于管理宿主机系统中 container 的完整生命周期，包括：镜像传输、存储、容器执行和监管、low-level 存储，以及 network attachments 等；

> containerd is designed to be embedded into a larger system, rather than being used directly by developers or end-users.

containerd 被设计为直接嵌入到大型系统使用的方式，而不是直接被开发者或端用户直接使用；

> containerd includes a daemon exposing gRPC API over a local UNIX socket. The API is a low-level one designed for higher layers to wrap and extend. It also includes a barebone CLI (ctr) designed specifically for development and debugging purpose. It uses runC to run containers according to the [OCI specification](https://www.opencontainers.org/about). The code can be found on [GitHub](https://github.com/containerd/containerd), and here are the [contribution guidelines](https://github.com/containerd/containerd/blob/master/CONTRIBUTING.md).

- containerd 实现中包含了一个在 local UNIX socket 上暴露 gRPC API 的 daemon 程序；
- 该 API 提供了偏底层功能，可供高层封装和扩展；
- containerd 实现中还包含了一个 CLI 工具（ctr）专门用于开发和调试目的；
- containerd 根据 OCI 规范，基于 runC 运行 containers ；

> containerd is based on the Docker Engine’s core container runtime to benefit from its maturity and existing contributors. (containerd is the core container runtime that forms the foundation for Docker Engine.)

containerd 是基于 Docker Engine 的 container runtime 开发起来的；

> containerd is a daemon born from extracting the container execution subset of the Docker Engine, and is used internally by Docker since the 1.11 release. containerd versions prior to v1.0.x were used in Docker 17.10 and earlier (see Docker version release notes), and Docker 17.12 is the first release to use containerd v1.0.0.

- containerd 基于 Docker Engine 的 container execution 代码子集创建；
- 从 Docker 1.11 开始 containerd 被 Docker 内部使用；
- 在 Docker 17.10 and earlier 时代，使用的 containerd 版本低于 v1.0.x ；
- 从 Docker 17.12 开始，使用的是 containerd v1.0.0 ；

![containerd - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20-%201.png)

Containerd was initiated by Docker Inc. and donated to CNCF in March of 2017. 

Containerd has a much smaller scope than Docker, provides a golang client API, and is more focused on being embeddable.

### 为什么需要 Containerd

containerd is a container daemon. It was originally built as an integration point for OCI runtimes like `runc` but over the past six months it has added a lot of functionality to bring it up to par with the needs of modern container platforms like Docker and Kubernetes.

Since there is no such thing as Linux containers in the kernelspace, containers are various kernel features tied together, when you are building a large platform or distributed system **you want an abstraction layer between your management code and the syscalls and duct tape of features to run a container**. That is where containerd lives. It provides a client layer of types that platforms can build on top of without ever having to drop down to the kernel level.  It’s so much nicer towork with Container, Task, and Snapshot types than it is to manage calls to `clone()` or `mount()`.

Containerd was designed to be used by Docker and Kubernetes as well as any other container platform that wants to abstract away syscalls or OS specific functionality to run containers on linux, windows, solaris, or other OSes. 

### Containerd 提供了什么

So what do you actually get using containerd? 

- You get push and pull functionality as well as image management.
- You get container lifecycle APIs to create, execute, and manage containers and their tasks.
- An entire API dedicated to snapshot management.

Basically everything that you need to build a container platform without having to deal with the underlying OS details.  

### Containerd 和 Docker 之间的关系

> Docker is a complete platform and programming environment for containerized applications. containerd is one of dozens of specialized components integrated into Docker. Developers and IT professionals looking to build, ship and run containerized applications should continue to use Docker. Operators and integrators looking for specialized components to swap into their platform should consider containerd.

- Docker 是一个针对容器化应用提供的完整平台和编程环境；
- containerd 是集成到 Docker 中的、具有特定功能的、众多组件中的一个；

![containerd in docker - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20in%20docker%20-%201.png)

对比

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20is%20a%20kind%20of%20container%20runtime.png)

> containerd 0.2.4 used in Docker 1.12 covers only container execution and process management.

![containerd in docker - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20in%20docker%20-%202.png)

> containerd’s roadmap is to refactor the Docker Engine codebase to extract more of its logic for distribution, networking and storage on a single host into a reusable component that Docker will use, and that can be used by other container orchestration projects or hosted container services.

containerd 的目标是重构（提取） Docker Engine 代码中的平台通用逻辑，以便得到一个 Docker 可重用组件，同时可以被其他 container 编排项目或 hosted container services 所使用；

![containerd in docker - 3](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20in%20docker%20-%203.png)

### Containerd 和 OCI 和 runC 之间的关系

> Docker donated the **OCI specification** to the Linux Foundation in 2015, along with a reference implementation called `runc`. containerd integrates **OCI/runc** into a feature-complete, production-ready core container runtime. `runc` is a component of containerd, the executor for containers. containerd has a wider scope than just executing containers: downloading container images, managing storage and network interfaces, calling `runc` with the right parameters to run containers. containerd fully leverages the Open Container Initiative’s (OCI) runtime, image format specifications and OCI reference implementation (`runc`) and will pursue OCI certification when it is available. Because of its massive adoption, containerd is the industry standard for implementing OCI.

- Docker 将 OCI 标准贡献给了 Linux Foundation ，同时还有一个 OCI 标准的参考实现，即 runC ；
- containerd 集成 OCI/runc 后，成为了一个特性完整、生产级别的 core container runtime ；
- runc 是 containerd 其中的一个组件，即 containers 的执行器；
- containerd 的功能不仅仅限于 containers 的执行，还包括：
    - 下载 container images 
    - 管理 storage 和 network interfaces
    - 调用 runc 来运行 containers
- containerd 充分利用了
    - OCI 的 runtime
    - image format specifications
    - OCI 的参考实现 runc
- 由于 containerd 被大量采用，已经成为了 OCI 实现的工业标准；

### Containerd 和 Container Orchestration Systems 之间的关系

> Kubernetes today uses Docker directly. In a future version Kubernetes can implement container support in the `Kubelet` by implementing it’s Container Runtime Interface using containerd. Mesos and other orchestration engines can leverage containerd for core container runtime functionality as well.

- 当前 Kubernetes 是直接使用 Docker 的（这里有点过时了，应该注明版本），将来版本的 Kubernetes 会在 Kubelet 中实现 container 支持，即通过 CRI 方式使用 containerd ；
- Mesos 和其他编排引擎同样也能利用 containerd 作为 container runtime ；

![containerd with container orchestration systems](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20with%20container%20orchestration%20systems.png)

对比

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20is%20a%20kind%20of%20container%20runtime.png)

### The Kubernetes containerd Integration Architecture

- You can now use containerd 1.1 as the container runtime for production Kubernetes clusters!
- Containerd 1.1 works with Kubernetes 1.10 and above, and supports all Kubernetes features. 

The Kubernetes containerd integration architecture has evolved twice.

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%201.0%20-%20CRI-Containerd.png)

> For **containerd 1.0**, a daemon called `cri-containerd` was required to operate between `Kubelet` and `containerd`. `Cri-containerd` handled the Container Runtime Interface (CRI) service requests from `Kubelet` and used `containerd` to manage containers and container images correspondingly. Compared to the Docker CRI implementation (`dockershim`), this eliminated one extra hop in the stack.

在 containerd 1.0 时期，在 `Kubelet` 和 `containerd` 之间还有一个名为 `cri-containerd` 的 daemon 程序；`cri-containerd` 负责处理来自 `Kubelet` 的 CRI 服务请求，并通过 `containerd` 进行 containers 和 container images 的管理；这种实现和 Docker 原生的 CRI 实现（即 `dockershim`）相比，消除了调用路径上额外的一跳；

> However, `cri-containerd` and containerd 1.0 were still 2 different daemons which interacted via grpc. The extra daemon in the loop made it more complex for users to understand and deploy, and introduced unnecessary communication overhead.

然而，`cri-containerd` 和 containerd 1.0 仍旧是两个不同的 daemon 程序，需要通过 grpc 进行交互；这种结构对用户来说既难以理解，也难以部署，同时还引入了不必要的通信开销；

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Containerd%201.1%20-%20CRI%20Plugin.png)

> In containerd 1.1, the `cri-containerd` daemon is now refactored to be a containerd CRI plugin. The CRI plugin is built into containerd 1.1, and enabled by default. Unlike `cri-containerd`, the CRI plugin interacts with containerd through direct function calls. This new architecture makes the integration more stable and efficient, and eliminates another grpc hop in the stack. Users can now use Kubernetes with containerd 1.1 directly. The `cri-containerd` daemon is no longer needed.

在 containerd 1.1 时期，`cri-containerd` daemon 被被重构成 containerd 的 CRI plugin ；因此，CRI plugin 是被构建在 containerd 1.1 内部的，且默认被使能；与 `cri-containerd` 不同，CRI plugin 和 containerd 的交互方式是通过之间函数调用完成的；新的架构确保了集成的稳定和高效，并消除了额外的 grpc 一跳；

用户现在已经可以在 Kubernetes 中直接使用 containerd 1.1 了；因此 `cri-containerd` daemon 已经不再需要了；

### cri-containerd

> Cri-containerd is exactly that: an implementation of CRI for containerd. It operates on the same node as the Kubelet and containerd. Layered between Kubernetes and containerd, cri-containerd handles all CRI service requests from the Kubelet and uses containerd to manage containers and container images. Cri-containerd manages these service requests in part by forming containerd service requests while adding sufficient additional function to support the CRI requirements.

- cri-containerd 是一种 CRI 实现；
- cri-containerd 运行在与 Kubelet 和 containerd 相同的 node 上；
- cri-containerd 位于 Kubernetes 和 containerd 中间，处理所有来自 Kubelet 的 CRI 服务请求，并使用 containerd 管理容器和容器镜像；

> Cri-containerd uses containerd to manage the full container lifecycle and all container images. As also shown below, cri-containerd manages pod networking via CNI (another CNCF project).

![cri-containerd architecture](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/cri-containerd%20architecture.png)

### How cri-containerd Works

Let’s use an example to demonstrate how cri-containerd works for the case when Kubelet creates a single-container pod:

1. Kubelet calls cri-containerd, via the CRI runtime service API, to create a pod;
2. cri-containerd uses containerd to create and start a special pause container (the sandbox container) and put that container inside the pod’s cgroups and namespace (steps omitted for brevity);
3. cri-containerd configures the pod’s network namespace using CNI;
4. Kubelet subsequently calls cri-containerd, via the CRI image service API, to pull the application container image;
5. cri-containerd further uses containerd to pull the image if the image is not present on the node;
6. Kubelet then calls cri-containerd, via the CRI runtime service API, to create and start the application container inside the pod using the pulled container image;
7. cri-containerd finally calls containerd to create the application container, put it inside the pod’s cgroups and namespace, then to start the pod’s new application container. After these steps, a pod and its corresponding application container is created and running.

### Performance Improvement

Performance improvement between containerd 1.1 and Docker 18.03 CE was optimized in terms of pod startup latency and daemon resource usage.

### Containerd 和 Docker Engine 和 Kubernetes 的关系

- Docker is by far the most common container runtime used in production Kubernetes environments, but Docker’s smaller offspring, **containerd**, may prove to be a better option. 
- Docker Engine is built on top of containerd. 
- The next release of [Docker Community Edition (Docker CE)](https://www.docker.com/products/docker-engine) will use containerd version 1.1. Of course, it will have the CRI plugin built-in and enabled by default. This means users will have the option to continue using Docker Engine for other purposes typical for Docker users, while also being able to configure Kubernetes to use the underlying containerd that came with and is simultaneously being used by Docker Engine on the same node. 
- Since containerd is being used by both Kubelet and Docker Engine, this means users who choose the containerd integration will not just get new Kubernetes features, performance, and stability improvements, they will also have the option of keeping Docker Engine around for other use cases.
- A containerd **namespace mechanism** is employed to guarantee that Kubelet and Docker Engine won’t see or have access to containers and images created by each other. This makes sure they won’t interfere with each other. This also means that:
    - Users won’t see Kubernetes created containers with the `docker ps` command. Please use `crictl ps` instead. And vice versa, users won’t see Docker CLI created containers in Kubernetes or with `crictl ps` command. The `crictl create` and `crictl runp` commands are only for troubleshooting. **Manually starting pod or container with `crictl` on production nodes is not recommended**.
    - Users won’t see Kubernetes pulled images with the `docker images` command. Please use the `crictl images` command instead. And vice versa, Kubernetes won’t see images created by `docker pull``, docker load` or `docker build` commands. Please use the `crictl pull` command instead, and [ctr](https://github.com/containerd/containerd/blob/master/docs/man/ctr.1.md) `cri load` if you have to load an image.
- With containerd integrated in Docker Engine, you get the next generation of runtime components, with more performance and configurability, integrated in a portable application workflow devs and ops know well, usable for any type of use case (single server, orchestrated runtime, CI/CD, IoT, etc.)
- containerd 1.1 implements Kubernetes Container Runtime Interface (CRI), so it can be used directly by Kubernetes, as well as Docker Engine.

See the architecture figure below showing the same containerd being used by Docker Engine and Kubelet:

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/containerd%20with%20Docker%20Engine%20and%20kubelet.png)

### Summary

- Containerd 1.1 natively supports CRI. It can be used directly by Kubernetes.
- Containerd 1.1 is production ready.
- Containerd 1.1 has good performance in terms of pod startup latency and system resource utilization.
- `crictl` is the CLI tool to talk with containerd 1.1 and other CRI-conformant container runtimes for node troubleshooting.
- The next stable release of Docker CE will include containerd 1.1. Users have the option to continue using Docker for use cases not specific to Kubernetes, and configure Kubernetes to use the same underlying containerd that comes with Docker.

### Setup a Kubernetes cluster using containerd as the container runtime

- For a production quality cluster on GCE brought up with kube-up.sh, see [here](https://github.com/containerd/cri/blob/v1.0.0/docs/kube-up.md).
- For a multi-node cluster installer and bring up steps using ansible and kubeadm, see [here](https://github.com/containerd/cri/blob/v1.0.0/contrib/ansible/README.md).
- For creating a cluster from scratch on Google Cloud, see [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).
- For a custom installation from release tarball, see [here](https://github.com/containerd/cri/blob/v1.0.0/docs/installation.md).
- To install using LinuxKit on a local VM, see [here](https://github.com/linuxkit/linuxkit/tree/master/projects/kubernetes).

Ref: 

- [containerd/containerd](https://github.com/containerd/containerd)
- https://containerd.io/
- [Kubernetes Containerd Integration Goes GA](https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/)
- [Docker Engine Sparked the Containerization Movement](https://www.docker.com/products/docker-engine)
- [What is containerd runtime](https://blog.docker.com/2017/08/what-is-containerd-runtime/)
- [Containerd Brings More Container Runtime Options for Kubernetes](https://kubernetes.io/blog/2017/11/containerd-container-runtime-options-kubernetes/)

## Image Formats

![Image Formats - 1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Image%20Formats%20-%201.png)

![Image Formats - 2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Image%20Formats%20-%202.png)

## Starting a Container

- Initialize a root filesystem (RootFS) from snapshot
- Setup OCI configuration (config.json)
- Use metadata from container and snapshotter to specify config and mounts
- Start process via the task service

![Starting a Contrainer](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Starting%20a%20Contrainer.png)

## Pulling an Image

- Resolve manifest or index (manifest list)
- Download all the resources referenced by the manifest
- Unpack layers into snapshots
- Register mappings between manifests and constituent resources

![Pulling an Image](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Pulling%20an%20Image.png)

## What is a container manifest

Ref: https://stackoverflow.com/questions/47006220/what-is-a-container-manifest

> **An `image` is a combination of a JSON manifest and individual layer files**. The process of pulling an image centers around retrieving these two components. 

image 由 JSON manifest 和不同的 layer files 构成；镜像拉取行为正是围绕着这两部分内容展开的；

> So when you pull an Image file:
> 
> - Get Manifest:
> ```
> GET /v2/<name>/manifests/<reference>
> ```
>
> - When the manifest is in hand, the client must verify the signature to ensure the names and layers are valid.
>
> - Then the client will then use the digests to download the individual layers. Layers are stored in as blobs in the V2 registry API, keyed by their digest.

因此，当你拉取镜像时：

- 首先，获取 Manifest ；
- 其次，验证 signature ，以便确认 names 和 layers 的有效性；
- 最后，基于 digests 下载不同的 layers ；
- （之后就是运行容器了）

在 V2 registry API 中，Layers 是以 blobs 的形态进行保存的，通过 blobs 对应的 digest 进行索引；


> The **manifest** types are effectively the JSON-represented description of a named/tagged image. This description (manifest) is meant for consumption by a container runtime, like the Docker engine.

manifest 事实上就是基于 JSON 方式描述的 named/tagged image ；这个概念主要被 Docker engine 所使用；

> Any registry or runtime that claims to have Docker distribution v2 API/v2.2 image specification support will be interacting with the various manifest types to find out:
>
> - what actual filesystem content (layers) will be needed to build the root filesystem for the container, and..
> - any specific image configuration that is necessary to know how to run a container using this image. For example, information like what command to run when starting the container (as probably represented in the `Dockerfile` that was used to build the image).

任何 registry 或 runtime 只要声称支持 Docker distribution v2 API/v2.2 image specification ，也就表明支持了如下 manifest 类型：

- 用于构建容器所需的 root filesystem 的、真实的文件系统内容（layers）；
- 任何需要了解的、基于该 image 的容器如何运行的、具体 image 配置；

> As a prior answer mentioned, a client (such as the `docker pull` implementation) talking to a registry will interact over the Docker v2 API to first fetch the manifest for a specific image/tag and then determine what to download in addition to be able to run a container based on this image. The v2 manifest format does not have signatures encoded into it, but external verification using tools like a [`notary` server](https://github.com/theupdateframework/notary) can be used to validate external signatures on the same "blob"/hash of content for full cryptographic trust. Docker calls this "**Docker Content Trust**" but does not require it when talking to a registry, nor is it part of the API flow when talking to an image registry.

> One additional detail about manifests in the v2.2 spec: there is not only a **standard manifest type**, but also a **manifest list type** which allows registries to represent support for multiple platforms (CPU or operating system variations) under a single "image:tag" reference. The manifest list simply has a list of platform entries with a redirector to an existing manifest so that an engine can go retrieve the correct components for that specific platform/architecture combination. **In DockerHub today, all official images are now actually manifest lists, allowing for many platforms to be supported using the same image name:tag combination**. I have a tool which can query entries in a registry and show whether they are manifest lists and also dump the contents of a manifest--both manifest lists and "regular" manifests. You can read more at the [manifest-tool](https://github.com/estesp/manifest-tool) GitHub repository.

## What are Docker image “layers”?

From the official Docker docs:

- Each instruction in a Dockerfile creates a layer in the image. When you change the Dockerfile and rebuild the image, only those layers which have changed are rebuilt. ([here](https://docs.docker.com/engine/docker-overview/#docker-objects)) 
- Basically, a **layer**, or **image layer** is a change on an image, or an intermediate image. Every command you specify (`FROM`, `RUN`, `COPY`, etc.) in your Dockerfile causes the previous image to change, thus creating a new layer. You can think of it as staging changes when you're using git: You add a file's change, then another one, then another one...
- The concept of layers comes in handy at the time of building images. Because layers are intermediate images, if you make a change to your Dockerfile, docker will build only the layer that was changed and the ones after that. This is called **layer caching**.
- [Since Docker v1.10](https://blog.docker.com/2016/01/docker-1-10-rc/), with introduction of the **content addressable storage**, the notion of 'layer' became quite different. Layers have no notion of an image or of belonging to an image, they become merely collections of files and directories that can be shared across images. Layers and images became separated. ([here](https://windsock.io/explaining-docker-image-ids/))
- Images are composed of layers. Each layer is a set of filesystem changes. Layers do not have configuration metadata such as environment variables or default arguments - these are properties of the image as a whole rather than any particular layer. ([here](https://github.com/moby/moby/blob/master/image/spec/v1.2.md))
- A Docker image is built up from a series of layers. Each layer represents an instruction in the image’s Dockerfile. Each layer except the very last one is read-only. 
- Each layer is only a set of differences from the layer before it. The layers are stacked on top of each other. When you create a new container, you add a new writable layer on top of the underlying layers. This layer is often called the “**container layer**”. All changes made to the running container, such as writing new files, modifying existing files, and deleting files, are written to this thin writable container layer. 
- A **storage driver** handles the details about the way these layers interact with each other.
- Docker uses storage drivers to manage the contents of the **image layers** and the writable **container layer**. Each storage driver handles the implementation differently, but all drivers use stackable image layers and the copy-on-write (CoW) strategy.
- **The major difference between a container and an image is the top writable layer**. All writes to the container that add new or modify existing data are stored in this writable layer. When the container is deleted, the writable layer is also deleted. The underlying image remains unchanged.


![](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)

## /var/run/docker.sock

- It’s the unix socket the Docker daemon listens on by default and it can be used to communicate with the daemon from within a container.
- When Docker platform is installed on a host, the Docker daemon listens on the `/var/run/docker.sock` unix socket by default. This can be seen from the options provided to the daemon, it should contain the following entry: `-H unix:///var/run/docker.sock`
- `/var/run/docker.sock` 是 Docker daemon 默认监听的 Unix domain socket ，（若有需要）容器中的进程可以通过它与 Docker daemon 进行通信。
- 绑定 Docker daemon 的 `/var/run/docker.sock`套接字之后，容器的权限会很高，因为其可以对 Docker daemon 进行控制。因此，这一点必须谨慎使用，只能用于足够信任的容器。

简单使用：

```
# Run an alpine container and bind mount the docker.sock
docker run -v /var/run/docker.sock:/var/run/docker.sock -ti alpine sh

# Install the curl utility
apk update && apk add curl

# We can send a HTTP request to the /events endpoint through the Docker socket. 
# The command hangs on waiting for new events from the daemon. 
# Each new events will then be streamed from the daemon.
curl --unix-socket /var/run/docker.sock http://localhost/events
```

参考：

- [关于/var/run/docker.sock](https://blog.fundebug.com/2017/04/17/about-docker-sock/)
- [About /var/run/docker.sock](https://medium.com/lucjuggery/about-var-run-docker-sock-3bfd276e12fd)
- [Can anyone explain docker.sock](https://stackoverflow.com/questions/35110146/can-anyone-explain-docker-sock)
- [The Dangers of Docker.sock](https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/)

## --privileged 特权容器

- 大约在 0.6 版，--privileged 被引入 docker ；
- 使用 --privileged 容器内的 root 拥有真正的 root 权限；否则，容器内的 root 只是外部的一个普通用户权限；
- 基于 --privileged 启动的容器，可以看到很多 host 上的设备，并且可以执行 mount ；
- 甚至允许你在 docker 容器中启动 docker 容器；
- This feature allowed bypassing a prior constraint of using containers, they were unable to access the host system’s devices. A possible use case for this, aside from running Docker inside Docker, was allowing you to trivially use things like your web-cam and such from within Docker.

实验：

- 非特权

```
docker run -it alpine sh
```

```
/ # ls /dev/
console  fd       mqueue   ptmx     random   stderr   stdout   urandom
core     full     null     pts      shm      stdin    tty      zero
/ #
/ # mkdir /home/test/
/ # mkdir /home/test2/
/ # mount -o bind /home/test  /home/test2
mount: permission denied (are you root?)
```

- 特权

```
docker run -it --privileged alpine sh
```

```
/ # ls /dev
autofs              net                 tty13               tty4                tty9                ttyS6
bsg                 network_latency     tty14               tty40               ttyS0               ttyS7
btrfs-control       network_throughput  tty15               tty41               ttyS1               ttyS8
console             null                tty16               tty42               ttyS10              ttyS9
core                port                tty17               tty43               ttyS11              ttyprintk
cpu_dma_latency     ppp                 tty18               tty44               ttyS12              uinput
ecryptfs            psaux               tty19               tty45               ttyS13              urandom
fd                  ptmx                tty2                tty46               ttyS14              vboxguest
full                pts                 tty20               tty47               ttyS15              vboxuser
fuse                random              tty21               tty48               ttyS16              vcs
hpet                rfkill              tty22               tty49               ttyS17              vcs1
hwrng               rtc0                tty23               tty5                ttyS18              vcs2
input               sda                 tty24               tty50               ttyS19              vcs3
kmsg                sda1                tty25               tty51               ttyS2               vcs4
lightnvm            sdb                 tty26               tty52               ttyS20              vcs5
loop-control        sg0                 tty27               tty53               ttyS21              vcs6
loop0               sg1                 tty28               tty54               ttyS22              vcsa
loop1               shm                 tty29               tty55               ttyS23              vcsa1
loop2               snapshot            tty3                tty56               ttyS24              vcsa2
loop3               snd                 tty30               tty57               ttyS25              vcsa3
loop4               stderr              tty31               tty58               ttyS26              vcsa4
loop5               stdin               tty32               tty59               ttyS27              vcsa5
loop6               stdout              tty33               tty6                ttyS28              vcsa6
loop7               tty                 tty34               tty60               ttyS29              vfio
mapper              tty0                tty35               tty61               ttyS3               vga_arbiter
mcelog              tty1                tty36               tty62               ttyS30              vhost-net
mem                 tty10               tty37               tty63               ttyS31              zero
memory_bandwidth    tty11               tty38               tty7                ttyS4
mqueue              tty12               tty39               tty8                ttyS5
/ # mkdir /home/test/
/ # mkdir /home/test2/
/ # mount -o bind /home/test /home/test2
/ # mount
...
none on /home/test2 type aufs (rw,relatime,si=65f2e9cc676bf3bf,dio,dirperm1)
```

参考：

- [在 Docker 中运行 Docker](https://github.com/jpetazzo/dind)
- https://blog.csdn.net/halcyonbaby/article/details/43499409


## alpine 是什么

- A minimal Docker image based on Alpine Linux with a complete package index and only 5 MB in size! 
- Alpine 操作系统是一个面向安全的轻型 Linux 发行版。它不同于通常 Linux 发行版，Alpine 采用了 musl libc 和 busybox 以减小系统的体积和运行时资源消耗，但功能上比 busybox 又完善的多，因此得到开源社区越来越多的青睐。在保持瘦身的同时，Alpine 还提供了自己的包管理工具 apk，可以通过 https://pkgs.alpinelinux.org/packages 网站上查询包信息，也可以直接通过 apk 命令直接查询和安装各种软件。
- Alpine 中软件安装包的名字可能会与其他发行版有所不同，可以在 https://pkgs.alpinelinux.org/packages 网站搜索并确定安装包名称。如果需要的安装包不在主索引内，但是在测试或社区索引中。那么可以按照以下方法使用这些安装包；
- Alpine 由非商业组织维护的，支持广泛场景的 Linux发行版，它特别为资深/重度Linux用户而优化，关注安全，性能和资源效能。Alpine 镜像可以适用于更多常用场景，并且是一个优秀的可以适用于生产的基础系统/环境。
- Alpine Docker 镜像也继承了 Alpine Linux 发行版的这些优势。相比于其他 Docker 镜像，它的容量非常小，仅仅只有 5 MB 左右（对比 Ubuntu 系列镜像接近 200 MB），且拥有非常友好的包管理机制。官方镜像来自 docker-alpine 项目。
- 目前 Docker 官方已开始推荐使用 Alpine 替代之前的 Ubuntu 做为基础镜像环境。这样会带来多个好处。包括镜像下载速度加快，镜像安全性提高，主机之间的切换更方便，占用更少磁盘空间等。

相关资源：

- [docker_practice](https://yeasy.gitbooks.io/docker_practice/cases/os/alpine.html)
- Alpine 官网：http://alpinelinux.org/
- Alpine 官方仓库：https://github.com/alpinelinux
- Alpine 官方镜像：https://hub.docker.com/_/alpine/
- Alpine 官方镜像仓库：https://github.com/gliderlabs/docker-alpine


## parent image v.s. base image

A **parent image** is the image that your image is based on. It refers to the contents of the `FROM` directive in the Dockerfile. Each subsequent declaration in the Dockerfile modifies this parent image. Most Dockerfiles start from a parent image, rather than a base image. However, the terms are sometimes used interchangeably.

A **base image** either has no `FROM` line in its Dockerfile, or has `FROM scratch`.

其他：

- [Create a base image](https://docs.docker.com/develop/develop-images/baseimages/)
- [How to build a base image from scratch?](https://forums.docker.com/t/how-to-build-a-base-image-from-scratch/34928)


## What are Docker `<none>:<none>` images?

Ref: https://www.projectatomic.io/blog/2015/07/what-are-docker-none-none-images/

相关问题：

- What are `<none>:<none>` images ?
- What are dangling images ?
- Why do I see a lot of `<none>:<none>` images when I do `docker images -a` ?
- What is the difference between `docker images` and `docker images -a` ?

Two kinds of `<none>:<none>` images:

- The Good one: They stand for **intermediate images** and can be seen using `docker images -a`. They don’t result into a disk space problem, and can be useful for caching.
- The Bad one: The **dangling images** which can cause disk space problems. A dangling file system layer in Docker is something that is unused and is not being referenced by any images. Hence we need a mechanism for Docker to clear these dangling images. Can be seen using `docker images -a`. These dangling images are produced as a result of `docker build` or `docker pull` command. 

The next command can be used to clean up dangling images (Docker doesn’t have an automatic garbage collection system as of now. For now this command can be used to do a manual garbage collection):

```
docker rmi $(docker images -f "dangling=true" -q)
```


## 镜像源变更

- [阿里云镜像](https://opsx.alibaba.com/mirror)
- [清华镜像](https://mirrors.tuna.tsinghua.edu.cn/)
- [中科大镜像](https://mirrors.ustc.edu.cn/)
- [自建镜像](https://github.com/anjia0532/alpine-package-mirror)


### alpine

```
echo -e "https://mirrors.ustc.edu.cn/alpine/latest-stable/main\nhttps://mirrors.ustc.edu.cn/alpine/latest-stable/community" > /etc/apk/repositories
```

或者

```
echo "https://mirrors.ustc.edu.cn/alpine/latest-stable/main" >> /etc/apk/repositories
echo "https://mirrors.ustc.edu.cn/alpine/latest-stable/community" >> /etc/apk/repositories
```

之后使用如下命令安装

```
apk --update add --no-cache <package>
```

另外，可以通过如下命令安装不在主索引内，但是在测试或社区索引中的包

```
echo "http://dl-4.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories
```

### ubuntu

todo


## DNS 变更

- By default, a container inherits the DNS settings of the Docker daemon, including the `/etc/hosts` and `/etc/resolv.conf`. You can override these settings on a **per-container basis**. [Ref](https://docs.docker.com/config/containers/container-networking/#dns-services)
- 如果在构建镜像过程中经常出现失败问题，则可能是 GFW 的原因；可以通过抓包确认 DNS 解析是否访问了 google-public-dns-a.google.com (8.8.8.8) ；如果是，解决办法如下
    - 修改宿主机的 hosts 文件，写死 ip 地址；这种方式为静态方式；
    - 修改 Docker 的 daemon.json 文件（指定其他 DNS 服务，如 DNSPod 提供的 public DNS 服务）；这种方式为动态方式；[Ref](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
    - 推荐使用第二种办法；
    - DNSPod 提供的公共域名解析服务 [Public DNS +](https://www.dnspod.cn/products/public.dns) ，服务IP:119.29.29.29
    - 调整 daemon.json 中的 `"dns": ["119.29.29.29"]` 后，执行 `sudo systemctl daemon-reload`


## 时区变更

Docker Store 上的官方镜像基本上都默认是 UTC 时区；即 /etc/timezone 的内容为 Etc/UTC，也就是标准的 UTC 时间，所以要简单的调整一下，变成中国标准时间。

场景：

- 基于 Dockerfile 构建自定义镜像时进行时区调整
    - 详见下面
- 启动 container 时基于主机文件进行时区调整（好处是不会修改镜像，适合需要在不同时区主机上运行的场景）
    - `docker run -v /etc/localtime:/etc/localtime <IMAGE:TAG>`
- 针对运行中的 container 进行时区调整
    - `docker exec -it <CONTAINER NAME> bash`
    - `echo "Asia/Shanghai" > /etc/timezone && dpkg-reconfigure -f noninteractive tzdata` -- ubuntu 环境下的命令


其他：

- CST – China Standard Time
- UTC - Coordinated Universal Time
- GMT - Greenwich Mean Time
- noninteractive - 方便整合到 Dockerfile 中使用


Ref:

- [Docker 中如何设置 container 的时区
](https://brickyang.github.io/2017/03/16/Docker%20%E4%B8%AD%E5%A6%82%E4%BD%95%E8%AE%BE%E7%BD%AE%20container%20%E7%9A%84%E6%97%B6%E5%8C%BA/)
- [在 Docker 中配置时区](https://tommy.net.cn/2015/02/05/config-timezone-in-docker/)

### alpine

详见 [timezone_adjustment/Dockerfile.alpine](https://github.com/moooofly/dockerfiles/blob/master/timezone_adjustment/Dockerfile.alpine)

### ubuntu

详见 [timezone_adjustment/Dockerfile.ubuntu](https://github.com/moooofly/dockerfiles/blob/master/timezone_adjustment/Dockerfile.ubuntu)



