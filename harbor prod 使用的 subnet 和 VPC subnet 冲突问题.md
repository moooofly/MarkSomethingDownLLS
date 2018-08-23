# harbor prod 使用的 subnet 和 VPC subnet 冲突问题

## 背景

liang 新建的 k8s 集群中的某个 node 上访问 harbor prod 不通；经测试，该 node 访问其他服务都没有问题；该 node 的地址为 172.18.x.x ；

测试命令

```
nc -vz 172.31.2.7 80
```

## 现状

- 查看 docker 所管理的网络信息

```
root@harbor-prod:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4f3bb0e533de        bridge              bridge              local
72bb065e2d07        harbor_harbor       bridge              local
e4a5fd2f3443        host                host                local
9d6b5291548d        none                null                local
root@harbor-prod:~#
```

- 可以看到 harbor 创建了一个 `172.18.0.0/16` 子网，构成 harbor 服务的所有容器均位于该子网下

```
root@harbor-prod:~# docker network inspect harbor_harbor
[
    {
        "Name": "harbor_harbor",
        "Id": "72bb065e2d079fd958073a27a07c7b1b92211c24eb694a8f36d39dfc4272af45",
        "Created": "2017-11-30T15:20:09.970422773Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "36313ded78e45ac998e56effe40590564325a9001ece3c788cd9dceb550ac34b": {
                "Name": "registry",
                "EndpointID": "b933d314aa64edc26c4474298ba2d82e3370370684c9a7159c27a643694b07c3",
                "MacAddress": "02:42:ac:12:00:05",
                "IPv4Address": "172.18.0.5/16",
                "IPv6Address": ""
            },
            "5b19cf22184ac1d213f6526f7301763b64e4491d70ea691e62fec704860c5f36": {
                "Name": "harbor-ui",
                "EndpointID": "dd263609ebb3d3acc0377bc4b8e337dc94c4d52d19ac83d26bb6896833ede7d8",
                "MacAddress": "02:42:ac:12:00:06",
                "IPv4Address": "172.18.0.6/16",
                "IPv6Address": ""
            },
            "67b87e6eb0358a2f6a9b263113d62f46d94c7b30f0099984fce2519352a9c07d": {
                "Name": "harbor-log",
                "EndpointID": "a4a45d67c4c25b6ecaaee3aff1c7239ba95cbfa128197ddcee09178a188663ae",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "71bbdc989c5186a7fcec01f59130fa69f3bf1fd368881a096582c7d09d8b3e62": {
                "Name": "harbor-db",
                "EndpointID": "692ef80dea1466d8d60b694f5d2d9d8bf3c1ca0ffbd5379a53e9811e35401967",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
            "92ed94ed865240afe4fd20122a44c58c370b01f8377b7d225852aebc6e19e4f0": {
                "Name": "nginx",
                "EndpointID": "7789800d573793e51f9bdadacc0f22c84b8e4e3e9c597541efbf4ea1e3ec57fc",
                "MacAddress": "02:42:ac:12:00:08",
                "IPv4Address": "172.18.0.8/16",
                "IPv6Address": ""
            },
            "d805d47116815e361200fc0b6786ef7bc9000854143c3541d6f3bafd19ba532f": {
                "Name": "harbor-adminserver",
                "EndpointID": "a30754be8bdf653977350446ed58df3ac6c9134c30c28fa71586c677397f7cdf",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "e749ea38248a9b09416b743d5c673ba0700ced6a046ad8b6f36de9cb964ade8a": {
                "Name": "harbor-jobservice",
                "EndpointID": "b39b7bc864977bc0cc7c5e55c09104f57867bd3f9ac9b6cf5c65c83dcca630fe",
                "MacAddress": "02:42:ac:12:00:07",
                "IPv4Address": "172.18.0.7/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
root@harbor-prod:~#
```

- 查看网卡信息，重点关注 `ens3`/`docker0`/`br-72bb065e2d07` 这三个

```
root@harbor-prod:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 02:d7:a0:28:7a:a4 brd ff:ff:ff:ff:ff:ff
    inet 172.31.2.7/20 brd 172.31.15.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::d7:a0ff:fe28:7aa4/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:d7:2a:00:89 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d7ff:fe2a:89/64 scope link
       valid_lft forever preferred_lft forever
100: br-72bb065e2d07: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:26:0f:af:24 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 scope global br-72bb065e2d07
       valid_lft forever preferred_lft forever
    inet6 fe80::42:26ff:fe0f:af24/64 scope link
       valid_lft forever preferred_lft forever
102: vethb434636@if101: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether 12:d2:cb:28:6e:c3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::10d2:cbff:fe28:6ec3/64 scope link
       valid_lft forever preferred_lft forever
104: veth40c79ce@if103: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether a2:8e:6a:b2:cc:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::a08e:6aff:feb2:ccf4/64 scope link
       valid_lft forever preferred_lft forever
106: vethe645f41@if105: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether fa:9b:20:46:da:56 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::f89b:20ff:fe46:da56/64 scope link
       valid_lft forever preferred_lft forever
108: vethd8ef255@if107: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether 46:19:0a:a3:86:68 brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::4419:aff:fea3:8668/64 scope link
       valid_lft forever preferred_lft forever
110: vethaf390fe@if109: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether 9e:db:96:f0:a4:58 brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::9cdb:96ff:fef0:a458/64 scope link
       valid_lft forever preferred_lft forever
112: veth9466a38@if111: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether 42:0e:7f:50:03:79 brd ff:ff:ff:ff:ff:ff link-netnsid 5
    inet6 fe80::400e:7fff:fe50:379/64 scope link
       valid_lft forever preferred_lft forever
114: vethf9fac6a@if113: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-72bb065e2d07 state UP group default
    link/ether 1e:ca:26:bf:bb:65 brd ff:ff:ff:ff:ff:ff link-netnsid 6
    inet6 fe80::1cca:26ff:febf:bb65/64 scope link
       valid_lft forever preferred_lft forever
root@harbor-prod:~#
```

- 查看该机器上的 `iptables` 配置，确实存在一些和 `172.18.0.0/16` 有关的规则（需要深入理解）

```
root@harbor-prod:~# iptables -t nat -L
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  ip-172-18-0-0.cn-north-1.compute.internal/16  anywhere
MASQUERADE  all  --  ip-172-17-0-0.cn-north-1.compute.internal/16  anywhere
MASQUERADE  tcp  --  ip-172-18-0-2.cn-north-1.compute.internal  ip-172-18-0-2.cn-north-1.compute.internal  tcp dpt:shell
MASQUERADE  tcp  --  ip-172-18-0-8.cn-north-1.compute.internal  ip-172-18-0-8.cn-north-1.compute.internal  tcp dpt:4443
MASQUERADE  tcp  --  ip-172-18-0-8.cn-north-1.compute.internal  ip-172-18-0-8.cn-north-1.compute.internal  tcp dpt:https
MASQUERADE  tcp  --  ip-172-18-0-8.cn-north-1.compute.internal  ip-172-18-0-8.cn-north-1.compute.internal  tcp dpt:http

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
RETURN     all  --  anywhere             anywhere
DNAT       tcp  --  anywhere             localhost            tcp dpt:1514 to:172.18.0.2:514
DNAT       tcp  --  anywhere             anywhere             tcp dpt:4443 to:172.18.0.8:4443
DNAT       tcp  --  anywhere             anywhere             tcp dpt:https to:172.18.0.8:443
DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:172.18.0.8:80
root@harbor-prod:~#
root@harbor-prod:~#
root@harbor-prod:~#
root@harbor-prod:~# iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.18.0.0/16 ! -o br-72bb065e2d07 -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.18.0.2/32 -d 172.18.0.2/32 -p tcp -m tcp --dport 514 -j MASQUERADE
-A POSTROUTING -s 172.18.0.8/32 -d 172.18.0.8/32 -p tcp -m tcp --dport 4443 -j MASQUERADE
-A POSTROUTING -s 172.18.0.8/32 -d 172.18.0.8/32 -p tcp -m tcp --dport 443 -j MASQUERADE
-A POSTROUTING -s 172.18.0.8/32 -d 172.18.0.8/32 -p tcp -m tcp --dport 80 -j MASQUERADE
-A DOCKER -i br-72bb065e2d07 -j RETURN
-A DOCKER -i docker0 -j RETURN
-A DOCKER -d 127.0.0.1/32 ! -i br-72bb065e2d07 -p tcp -m tcp --dport 1514 -j DNAT --to-destination 172.18.0.2:514
-A DOCKER ! -i br-72bb065e2d07 -p tcp -m tcp --dport 4443 -j DNAT --to-destination 172.18.0.8:4443
-A DOCKER ! -i br-72bb065e2d07 -p tcp -m tcp --dport 443 -j DNAT --to-destination 172.18.0.8:443
-A DOCKER ! -i br-72bb065e2d07 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.18.0.8:80
root@harbor-prod:~#
```

综上，基本上可以确定是由于 harbor 服务自己创建的 network 使用了 `172.18.0.0/16` 这个子网，同时设置了一些相应的 `iptables` 规则，导致 aws VPC 上具有 172.18.x.x 的地址无法和 harbor prod 正常通信；

接下来的问题是：**harbor 是如何选择子网地址的？是否可以通过配置文件进行控制？**


## 相关 issues

### [Default Bridge network overlaps with internal subnet #2634](https://github.com/vmware/harbor/issues/2634)

By default harbor creates it's own bridge network

```
root@harbor-prod:/opt/apps/harbor# cat docker-compose.yml
...
networks:
  harbor:
    external: false
```

the problem is i can't tell the docker daemon which subnet to create start from ([moby/moby#21776](https://github.com/moby/moby/issues/21776)) and unfortunately i am unlucky enough that the subnet chosen overlaps with one of our internal subnets.

I can simply hack up the compose file and fix this but won't be very portable though upgrades.

Happy to look at implementing a new config option if someone points me in the right direction

> I'm afraid currently you have to do the hack, but this is a valid request we'll consider it.

### [Cannot assign bridge IP address after Harbor start #92](https://github.com/vmware/harbor/issues/92)

After I start harbor, docker will create a bridge with IP 172.17.0.1 automatically. How can I assign the IP by myself?
I have a host with IP 172.17.1.42 can't access the harbor.

You can modify `harbor/Deploy/docker-compose.yml` to configure the default network used by Harbor before starting it.

- Add the following code to the end of `docker-compose.yml` (replace the `172.28.0.0/16` with your own subnet )

```
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

- Delete the old network and containers created previously

```
docker network rm deploy_network  
docker-compose rm
```

- Start Harbor

```
docker-compose up -d
```

References:

- https://docs.docker.com/compose/networking/#configuring-the-default-network
- https://docs.docker.com/compose/compose-file/#ipam


## 相关文章

### compose-file

#### [ipam](https://docs.docker.com/compose/compose-file/#ipam)

Specify custom IPAM config. This is an object with several properties, each of which is optional:

- `driver`: Custom IPAM driver, instead of the default.
- `config`: A list with zero or more config blocks, each containing any of the following keys:
    - `subnet`: Subnet in CIDR format that represents a network segment


A full example:

```
ipam:
  driver: default
  config:
    - subnet: 172.28.0.0/16
```

>  Note: Additional IPAM configurations, such as `gateway`, are only honored for version 2 at the moment.

### [Networking in Compose](https://docs.docker.com/compose/networking/)

以下内容适用于 version 2 or higher 版本的 Compose 文件格式；

> **By default** `Compose` **sets up a single [network](https://docs.docker.com/engine/reference/commandline/network_create/)** for your app. Each container for a service joins the default network and is both **reachable** by other containers on that network, and **discoverable** by them at a hostname identical to the container name.

- 默认情况下，docker-compose 为目标应用创建一个单独的 network ；
- 构成目标应用的所有 container 都会加入该 network ；
- 加入该 network 后，这些 container 彼此 reachable 和 discoverable ；

> Note: Your app’s network is given a name based on the “project name”, which is based on the name of the directory it lives in. You can override the project name with either the `--project-name flag` or the `COMPOSE_PROJECT_NAME` environment variable.

影响 network 命名的因素；

#### Specify custom networks

> Instead of just using the **default** app network, you can specify **your own** networks with the **top-level** `networks` key. This lets you create more complex topologies and **specify [custom network drivers](https://docs.docker.com/engine/extend/plugins_network/) and options**. You can also use it to connect services to externally-created networks which aren’t managed by Compose.

- 可以不使用 default 的 network ，而使用在 top-level `networks` key 中指定的 network ；
- 通过这种方式，可以创建出复杂的网络拓扑关系，可以指定想要的 network drivers 和 options ；
- 通过这种方式，可以让应用（即下属的所有 container）使用外部单独创建的 network ；

> Each service can specify what networks to connect to with the **service-level** `networks` key, which is a list of names referencing entries under the **top-level** `networks` key.

可以在 service 级别指定具体使用 top-level 中定义的哪个 network ；

> Here’s an example Compose file defining two custom networks. The `proxy` service is **isolated** from the `db` service, because they do not share a network in common - only `app` can talk to both.

一个通过 top-level 和 service-level 定义和选择 network 从而实现隔离的示例

```
version: "3"
services:

  proxy:
    build: ./proxy
    networks:
      - frontend
  app:
    build: ./app
    networks:
      - frontend
      - backend
  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    # Use a custom driver
    driver: custom-driver-1
  backend:
    # Use a custom driver which takes special options
    driver: custom-driver-2
    driver_opts:
      foo: "1"
      bar: "2"
```

> Networks can be configured with **static IP addresses** by setting the [ipv4_address and/or ipv6_address](https://docs.docker.com/compose/compose-file/#ipv4-address-ipv6-address) for each attached network.

可以为 network 配置静态 ip 地址；

> For full details of the network configuration options available, see the following references:
> 
> - [Top-level `networks` key](https://docs.docker.com/compose/compose-file/#network-configuration-reference)
> - [Service-level `networks` key](https://docs.docker.com/compose/compose-file/#networks)

#### Configure the default network

> Instead of (or as well as) specifying your own networks, you can also **change the settings of the app-wide `default` network** by defining an entry under `networks` named `default`:

可以不指定自定义 network ，而直接变更应用级别的 `default` network ；

```
version: "3"
services:

  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres

networks:
  default:
    # Use a custom driver
    driver: custom-driver-1
```

#### Use a pre-existing network

> If you want your containers to join a **pre-existing** network, use the [`external` option](https://docs.docker.com/compose/compose-file/#network-configuration-reference):

直接使用已经存在 network ；

```
networks:
  default:
    external:
      name: my-pre-existing-network
```

> Instead of attempting to create a network called `[projectname]_default`, Compose looks for a network called `my-pre-existing-network` and connect your app’s containers to it.


### [Use bridge networks](https://docs.docker.com/network/bridge/)

> In terms of networking, a `bridge network` is a **Link Layer device** which forwards traffic between network segments. A bridge can be a **hardware device** or a **software device** running within a host machine’s kernel.

- `bridge network` 属于一种链路层设备；
- bridge 可以是 hardware device ，也可以是运行在内核中的 software device ；

> In terms of Docker, a `bridge network` uses a software bridge which allows containers connected to the same bridge network to communicate, while providing isolation from containers which are not connected to that bridge network. **The Docker bridge driver automatically installs rules** in the host machine so that containers on different bridge networks cannot communicate directly with each other.

- 在 docker 中，`bridge network` 允许连接到其上的所有 container 相互通信，以及隔离未连接到其上的其他 container ；
- docker 的 bridge driver 会自动安装 iptables 规则，以确保连接在不同 bridge network 上的 container 不能直接相互通信；

> **`Bridge networks` apply to containers running on the same Docker daemon host**. For communication among containers running on different Docker daemon hosts, you can either manage routing at the OS level, or you can use an `overlay network`.

- `bridge network` 只对运行在相同 Docker daemon host 上的 container 起作用；
- 对于跨主机（different Docker daemon hosts）的通信，或者通过直接管理 OS 级别的路由，或者直接使用 `overlay network` ；

> When you start Docker, a [default bridge network](https://docs.docker.com/network/bridge/#use-the-default-bridge-network) (also called `bridge`) is created automatically, and newly-started containers connect to it unless otherwise specified. You can also create user-defined custom bridge networks. **User-defined bridge networks are superior to the default `bridge` network**.

- 启动 docker 后，会自动创建一个默认的 bridge network ，一般名为 `bridge0` ；
- 新启动的 container 默认都会连接到该 bridge 上，除非特别指定；
- 用户自定义的 bridge network 的优先级要高于默认的 bridge network ；

#### Differences between user-defined bridges and the default bridge

- **User-defined bridges provide better isolation and interoperability between containerized applications.**

更好的隔离性和互操作性

> **Containers connected to the same `user-defined` bridge network automatically expose all ports to each other, and no ports to the outside world**. This allows containerized applications to communicate with each other easily, without accidentally opening access to the outside world.

> Imagine an application with a web front-end and a database back-end. The outside world needs access to the web front-end (perhaps on port 80), but only the front-end itself needs access to the database host and port. Using a user-defined bridge, only the web port needs to be opened, and the database application doesn’t need any ports open, since the web front-end can reach it over the user-defined bridge.

> **If you run the same application stack on the `default` bridge network**, you need to open both the web port and the database port, using the `-p` or `--publish` flag for each. **This means the Docker host needs to block access to the database port by other means**.

- **User-defined bridges provide automatic DNS resolution between containers.**

自动在 container 间提供 DNS 解析能力

> **Containers on the `default` bridge network can only access each other by IP addresses, unless you use the [`--link` option](https://docs.docker.com/network/links/)**, which is considered legacy. **On a `user-defined` bridge network, containers can resolve each other by name or alias**.

> Imagine the same application as in the previous point, with a web front-end and a database back-end. If you call your containers `web` and `db`, the web container can connect to the db container at `db`, no matter which Docker host the application stack is running on.

> **If you run the same application stack on the `default` bridge network, you need to manually create links between the containers** (using the legacy `--link` flag). These links need to be created in both directions, so you can see this **gets complex with more than two containers which need to communicate**. Alternatively, you can manipulate the `/etc/hosts` files within the containers, but this **creates problems that are difficult to debug**.

- **Containers can be attached and detached from user-defined networks on the fly.**

可以随时进行 attach 和 detach 而不用停止、重建容器

> During a container’s lifetime, you can connect or disconnect it from `user-defined` networks on the fly. To remove a container from the `default` bridge network, you need to **stop** the container and **recreate** it with different network options.

- **Each user-defined network creates a configurable bridge.**

可以单独进行定制化配置

> **If your containers use the `default` bridge network**, you can configure it, but **all the containers use the same settings, such as `MTU` and `iptables` rules**. In addition, configuring the `default` bridge network happens outside of Docker itself, and **requires a restart of Docker**.

> `User-defined` bridge networks are created and configured using `docker network create`. **If different groups of applications have different network requirements, you can configure each `user-defined` bridge separately**, as you create it.

- **Linked containers on the default bridge network share environment variables.**

连接都 `default` bridge 上所有的 container 能够共享环境变量

> **Originally, the only way to share environment variables between two containers was to link them using the [`--link` flag](https://docs.docker.com/network/links/). This type of variable sharing is not possible with user-defined networks**. However, there are superior ways to share environment variables. A few ideas:

> - Multiple containers can mount a file or directory containing the shared information, using a **Docker volume**.

> - Multiple containers can be started together using **`docker-compose`** and the **compose file** can define the shared variables.

> - You can use **swarm services** instead of standalone containers, and take advantage of shared [secrets](https://docs.docker.com/engine/swarm/secrets/) and [configs](https://docs.docker.com/engine/swarm/configs/).

> **Containers connected to the same `user-defined` bridge network effectively expose all ports to each other**. For a port to be accessible to containers or non-Docker hosts on different networks, that port must be published using the `-p` or `--publish` flag.

#### Manage a user-defined bridge

> Use the `docker network create` command to create a `user-defined` bridge network.

```
$ docker network create my-net
```

> You can specify the **subnet**, the **IP address range**, the **gateway**, and other options. See the [docker network create reference](https://docs.docker.com/engine/reference/commandline/network_create/#specify-advanced-options) or the output of `docker network create --help` for details.

> Use the `docker network rm` command to remove a `user-defined` bridge network. **If containers are currently connected to the network, disconnect them first**.

```
$ docker network rm my-net
```

>> **What’s really happening?**
>>
>> When you **create** or **remove** a `user-defined` bridge or **connect** or **disconnect** a container from a `user-defined` bridge, Docker uses tools specific to the operating system to manage the underlying network infrastructure (such as adding or removing bridge devices or configuring `iptables` rules on Linux). These details should be considered implementation details. Let Docker manage your `user-defined` networks for you.


#### Connect a container to a user-defined bridge

> When you create a new container, you can specify one or more `--network` flags. This example connects a Nginx container to the `my-net` network. It also publishes port 80 in the container to port 8080 on the Docker host, so external clients can access that port. Any other container connected to the `my-net` network has access to all ports on the `my-nginx` container, and vice versa.

```
$ docker create --name my-nginx \
  --network my-net \
  --publish 8080:80 \
  nginx:latest
```

> To connect a **running** container to an existing `user-defined` bridge, use the `docker network connect` command. The following command connects an already-running `my-nginx` container to an already-existing `my-net` network:

```
$ docker network connect my-net my-nginx
```

#### Disconnect a container from a user-defined bridge

> To disconnect a **running** container from a `user-defined` bridge, use the `docker network disconnect` command. The following command disconnects the `my-nginx` container from the `my-net` network.

```
$ docker network disconnect my-net my-nginx
```

#### Enable forwarding from Docker containers to the outside world

从 Docker container 内部向外部通信的能力默认是被关闭的

> **By default, traffic from containers connected to the `default` bridge network is not forwarded to the outside world**. To enable forwarding, you need to change two settings. These are not Docker commands and they affect the Docker host’s kernel.

> - Configure the Linux kernel to allow IP forwarding.
> ```
> $ sysctl net.ipv4.conf.all.forwarding=1
> ```

> - Change the policy for the `iptables` FORWARD policy from DROP to ACCEPT.
> ```
> $ sudo iptables -P FORWARD ACCEPT
> ```

> These settings do not persist across a reboot, so you may need to add them to a start-up script.


#### Use the default bridge network

> **The `default` bridge network** is considered a legacy detail of Docker and is **not recommended for production use**. Configuring it is a manual operation, and it has technical shortcomings.

##### Connect a container to the default bridge network

> If you do not specify a network using the `--network` flag, and you do specify a network driver, your container is connected to the `default` bridge network by `default`. Containers connected to the `default` bridge network can communicate, but only by IP address, unless they are linked using the legacy `--link` flag.

#####Configure the default bridge network

> To configure the `default` bridge network, you specify options in `daemon.json`. Here is an example `daemon.json` with several options specified. Only specify the settings you need to customize.

```
{
  "bip": "192.168.1.5/24",
  "fixed-cidr": "192.168.1.5/25",
  "fixed-cidr-v6": "2001:db8::/64",
  "mtu": 1500,
  "default-gateway": "10.20.1.1",
  "default-gateway-v6": "2001:db8:abcd::89",
  "dns": ["10.20.1.2","10.20.1.3"]
}
```

> Restart Docker for the changes to take effect.


----------


## 解决方案

- 第一步，停止当前所有由 docker-compose 管理的容器进程

```
root@harbor-prod:/opt/apps/harbor# docker-compose down -v
Stopping nginx              ... done
Stopping harbor-jobservice  ... done
Stopping harbor-ui          ... done
Stopping registry           ... done
Stopping harbor-db          ... done
Stopping harbor-adminserver ... done
Stopping harbor-log         ... done
Removing nginx              ... done
Removing harbor-jobservice  ... done
Removing harbor-ui          ... done
Removing registry           ... done
Removing harbor-db          ... done
Removing harbor-adminserver ... done
Removing harbor-log         ... done
Removing network harbor_harbor
root@harbor-prod:/opt/apps/harbor#


# 可以看到由 docker-compose 所创建的 network 均被销毁了
root@harbor-prod:/opt/apps/harbor# ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:d7:2a:00:89
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:d7ff:fe2a:89/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:258 (258.0 B)

ens3      Link encap:Ethernet  HWaddr 02:d7:a0:28:7a:a4
          inet addr:172.31.2.7  Bcast:172.31.15.255  Mask:255.255.240.0
          inet6 addr: fe80::d7:a0ff:fe28:7aa4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:2816571661 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1965901056 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1859060950160 (1.8 TB)  TX bytes:1373476494879 (1.3 TB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:2224923504 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2224923504 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:753122069031 (753.1 GB)  TX bytes:753122069031 (753.1 GB)

root@harbor-prod:/opt/apps/harbor#


# 可以看到相应的 iptables 规则也被删除了
root@harbor-prod:/opt/apps/harbor# iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A DOCKER -i docker0 -j RETURN
root@harbor-prod:/opt/apps/harbor#
```

- 第二步，调整 network 配置

```
root@harbor-prod:/opt/apps/harbor# vi docker-compose.yml
...
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

重新启动失败，发现 network 的名字不对

```
root@harbor-prod:/opt/apps/harbor# docker-compose up -d
ERROR: Service "log" uses an undefined network "harbor"
root@harbor-prod:/opt/apps/harbor#
```

再次调整

```
root@harbor-prod:/opt/apps/harbor# vi docker-compose.yml
...
networks:
  harbor:
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

- 第三步，重新启动

```
root@harbor-prod:/opt/apps/harbor# docker-compose up -d
Creating network "harbor_harbor" with the default driver
Creating harbor-log ...
Creating harbor-log ... done
Creating harbor-db ...
Creating harbor-adminserver ...
Creating registry ...
Creating harbor-db
Creating registry
Creating harbor-adminserver ... done
Creating harbor-ui ...
Creating harbor-ui ... done
Creating harbor-jobservice ...
Creating harbor-db ... done
Creating nginx ...
Creating nginx ... done
root@harbor-prod:/opt/apps/harbor#


# 可以看到容器进程都成功启动了
root@harbor-prod:/opt/apps/harbor# docker ps
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
1246f770a686        vmware/nginx-photon:1.11.13        "nginx -g 'daemon ..."   25 seconds ago      Up 24 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
f9abf98c5cc0        vmware/harbor-jobservice:v1.2.2    "/harbor/harbor_jo..."   26 seconds ago      Up 25 seconds                                                                          harbor-jobservice
26a9a797a346        vmware/harbor-ui:v1.2.2            "/harbor/harbor_ui"      26 seconds ago      Up 25 seconds                                                                          harbor-ui
06603e5c58a8        vmware/harbor-adminserver:v1.2.2   "/harbor/harbor_ad..."   27 seconds ago      Up 25 seconds                                                                          harbor-adminserver
58acb13c76f9        vmware/harbor-db:v1.2.2            "docker-entrypoint..."   27 seconds ago      Up 24 seconds       3306/tcp                                                           harbor-db
97c4521acf90        vmware/registry:2.6.2-photon       "/entrypoint.sh se..."   27 seconds ago      Up 25 seconds       5000/tcp                                                           registry
7f3a26a3c886        vmware/harbor-log:v1.2.2           "/bin/sh -c 'crond..."   27 seconds ago      Up 26 seconds       127.0.0.1:1514->514/tcp                                            harbor-log
root@harbor-prod:/opt/apps/harbor#


# 可以看到 network 被重新建立
root@harbor-prod:/opt/apps/harbor# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4f3bb0e533de        bridge              bridge              local
e0f31f5113f8        harbor_harbor       bridge              local
e4a5fd2f3443        host                host                local
9d6b5291548d        none                null                local
root@harbor-prod:/opt/apps/harbor#
root@harbor-prod:/opt/apps/harbor# docker network inspect harbor_harbor
[
    {
        "Name": "harbor_harbor",
        "Id": "e0f31f5113f82e48cdf0a0b2b363fc82fabeae3f17e0078800db8c4f86a845f9",
        "Created": "2018-04-27T10:48:14.261025865Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.28.0.0/16"   # 和之前相比，这里缺少了一个 gateway 设置，不过貌似也没啥问题
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "06603e5c58a8397863244569ee30ae901cd6032a6eff18a8a58ddddd4d43cfcd": {
                "Name": "harbor-adminserver",
                "EndpointID": "4941ead677487b24274d91f421e3fa1b189c7da9de082b16b7c93e8901b73976",
                "MacAddress": "02:42:ac:1c:00:05",
                "IPv4Address": "172.28.0.5/16",
                "IPv6Address": ""
            },
            "1246f770a6866b37fcfda7abb4986927c7324cfe6b068d08c8734dcaa93129b1": {
                "Name": "nginx",
                "EndpointID": "98193c72f99283d8679a665849691ab37e26431651cccd320ddbef4571366e67",
                "MacAddress": "02:42:ac:1c:00:08",
                "IPv4Address": "172.28.0.8/16",
                "IPv6Address": ""
            },
            "26a9a797a346e95d7da52ed78dfa3c1415a136fef21080fc84dfd192bdc405ca": {
                "Name": "harbor-ui",
                "EndpointID": "5495f01855a78c6c18d0965e10138bfc9461a08fb1b79416f7a9d9ccea3c55da",
                "MacAddress": "02:42:ac:1c:00:06",
                "IPv4Address": "172.28.0.6/16",
                "IPv6Address": ""
            },
            "58acb13c76f92e5ca60262c4255ce4516930c1b9f0fbcc588cbfe39d2a0a8f92": {
                "Name": "harbor-db",
                "EndpointID": "0af27cb9209ee0b38540b7fc21aa48eaaf9711bd92b793db5ae5ac36e5a56b3d",
                "MacAddress": "02:42:ac:1c:00:04",
                "IPv4Address": "172.28.0.4/16",
                "IPv6Address": ""
            },
            "7f3a26a3c886b699089c9c71da3238b3cb5f8a28ba797a1518413db7af2ddd73": {
                "Name": "harbor-log",
                "EndpointID": "e17304541235ea3fddfa2f871f3efe0333892f731b7dc2e040e9c9955bcdb2a0",
                "MacAddress": "02:42:ac:1c:00:02",
                "IPv4Address": "172.28.0.2/16",
                "IPv6Address": ""
            },
            "97c4521acf90ea2f3f856972de473937238fa39f377fc42071953a4175ceeee1": {
                "Name": "registry",
                "EndpointID": "a2cbd21c1b9c99bfd624550bc3c3b08faeb5bbefe1ae756dc95e387fb4887054",
                "MacAddress": "02:42:ac:1c:00:03",
                "IPv4Address": "172.28.0.3/16",
                "IPv6Address": ""
            },
            "f9abf98c5cc084e712f2292f9bf73442d4d25493a941f2f4a49e928c6f8b35f0": {
                "Name": "harbor-jobservice",
                "EndpointID": "53a92f05f38390d8ad501830b08997596bd5f4730a80d66c8c26f234054389e7",
                "MacAddress": "02:42:ac:1c:00:07",
                "IPv4Address": "172.28.0.7/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
root@harbor-prod:/opt/apps/harbor#
```

查看网卡信息和 iptables 信息

```
root@harbor-prod:/opt/apps/harbor# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 02:d7:a0:28:7a:a4 brd ff:ff:ff:ff:ff:ff
    inet 172.31.2.7/20 brd 172.31.15.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::d7:a0ff:fe28:7aa4/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:d7:2a:00:89 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d7ff:fe2a:89/64 scope link
       valid_lft forever preferred_lft forever
115: br-e0f31f5113f8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:f2:5c:15:88 brd ff:ff:ff:ff:ff:ff
    inet 172.28.0.1/16 scope global br-e0f31f5113f8
       valid_lft forever preferred_lft forever
    inet6 fe80::42:f2ff:fe5c:1588/64 scope link
       valid_lft forever preferred_lft forever
117: vethce5c042@if116: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether fa:bf:58:d2:bd:a9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::f8bf:58ff:fed2:bda9/64 scope link
       valid_lft forever preferred_lft forever
119: veth3c9127a@if118: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether ae:4d:3d:d5:fb:6e brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::ac4d:3dff:fed5:fb6e/64 scope link
       valid_lft forever preferred_lft forever
121: veth4a00474@if120: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether ae:3f:3d:d7:fd:33 brd ff:ff:ff:ff:ff:ff link-netnsid 5
    inet6 fe80::ac3f:3dff:fed7:fd33/64 scope link
       valid_lft forever preferred_lft forever
123: vethdadea9c@if122: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether 9a:ac:71:22:41:ac brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::98ac:71ff:fe22:41ac/64 scope link
       valid_lft forever preferred_lft forever
125: veth603b5d1@if124: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether 72:78:36:45:aa:cb brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::7078:36ff:fe45:aacb/64 scope link
       valid_lft forever preferred_lft forever
127: vethe39eacc@if126: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether 66:8f:04:4a:bd:9a brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::648f:4ff:fe4a:bd9a/64 scope link
       valid_lft forever preferred_lft forever
129: veth189e29a@if128: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e0f31f5113f8 state UP group default
    link/ether 3a:e4:64:25:3c:72 brd ff:ff:ff:ff:ff:ff link-netnsid 6
    inet6 fe80::38e4:64ff:fe25:3c72/64 scope link
       valid_lft forever preferred_lft forever
root@harbor-prod:/opt/apps/harbor#
root@harbor-prod:/opt/apps/harbor#
root@harbor-prod:/opt/apps/harbor# iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.28.0.0/16 ! -o br-e0f31f5113f8 -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.28.0.2/32 -d 172.28.0.2/32 -p tcp -m tcp --dport 514 -j MASQUERADE
-A POSTROUTING -s 172.28.0.8/32 -d 172.28.0.8/32 -p tcp -m tcp --dport 4443 -j MASQUERADE
-A POSTROUTING -s 172.28.0.8/32 -d 172.28.0.8/32 -p tcp -m tcp --dport 443 -j MASQUERADE
-A POSTROUTING -s 172.28.0.8/32 -d 172.28.0.8/32 -p tcp -m tcp --dport 80 -j MASQUERADE
-A DOCKER -i br-e0f31f5113f8 -j RETURN
-A DOCKER -i docker0 -j RETURN
-A DOCKER -d 127.0.0.1/32 ! -i br-e0f31f5113f8 -p tcp -m tcp --dport 1514 -j DNAT --to-destination 172.28.0.2:514
-A DOCKER ! -i br-e0f31f5113f8 -p tcp -m tcp --dport 4443 -j DNAT --to-destination 172.28.0.8:4443
-A DOCKER ! -i br-e0f31f5113f8 -p tcp -m tcp --dport 443 -j DNAT --to-destination 172.28.0.8:443
-A DOCKER ! -i br-e0f31f5113f8 -p tcp -m tcp --dport 80 -j DNAT --to-destination 172.28.0.8:80
root@harbor-prod:/opt/apps/harbor#
root@harbor-prod:/opt/apps/harbor#
```