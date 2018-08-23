# Harbor HA solution proposals

> [issues/3728](https://github.com/vmware/harbor/issues/3728)  
> [issues/3582](https://github.com/vmware/harbor/issues/3582)  
> [T43199](https://phab.llsapp.com/T43199)  


----------


We will use this Page to discuss the **Harbor HA solution proposals**.

Harbor will plan to release some script to help make the HA setup easily.

Please feel free give out comments/suggestions/questions. We will adjust the proposal according to your feedback. So your voice are valuable to Harbor.

## Harbor HA solutions proposals

This document will cover the follow **four** solutions which provide different severity of Harbor HA. each solution will have it's cons and pros. We need to balance them and choose the one which is most fit the use scenario.

### Solution 1: Active-Active with scale out ability

![](https://user-images.githubusercontent.com/1715683/32586942-a92aa8da-c540-11e7-9af7-db03fc467451.png)

As seen in the figure, components involved in the architecture are:

- **VIP**: Virtual IP. The Harbor user will access Harbor through this VIP address. This VIP will only active on one load balancer node at the same time. It will automatically switch to the other node if the active loadbalancer node is down.

- **LoadBalancer01 and 02**: They together compose as a group which avoid single point failure of load balancer nodes. `Keepalived` is installed on both load balancer nodes. The two `Keepalived` instance will form a **VRRP** group to provide the VIP and ensure the VIP only shows on one node at the same time. The `LVS` component in `Keepalived` is responsible for router the requests between different Harbor servers according to the routing algorithm.

- **Harbor server 1..n**: These are the running Harbor instances. They are in active-active mode. User can setup multiple nodes according to their workload.

- **DB cluster**: The MariaDB is used by Harbor to store user authentication information, image metadata information and so on. **User should follow its best practice to make it HA protect**.

- **Shared Storage**: The shared storage is used for storing Docker Volumes used by Harbor. Images pushed by users are actually stored in this shared storage. The shared storage makes sure that multiple Harbor instances have consistent storage backend. Shared Storages can be Swift, NFS, S3, azure, GCS or OSS. **User should follow its best practice to make it HA protect**.

- **Redis**: The purpose of having Redis is to store UI session data and store the registry metadata cache. When one Harbor instance fails or the load balancer routes a user request to another Harbor instance, any Harbor instance can query the Redis to retrieve session information to make sure an end-user has a continued session. **User should follow the best practice of Redis to make it HA protect**.

> 由上面可以看出：DB cluster 和 Shared Storage 和 Redis 的 HA 都需要用户自己搞定；


#### Limitation

Currently it doesn’t support Clair and Notary.

#### Setup Prerequisites

- MariaDB cluster
- Shared Storage
- Redis cluster

> Item 1,2,3 are considered external components to Harbor. Before configuring Harbor HA, we assume these components are present and all of them are HA protected. Otherwise, any of these components can be a single point of failure.

- 2 VMs for Load balancer
- n VMs for Harbor instances (n >=2)
- n+1 static IPs (1 for VIP and the other n IPs will be used by harbor servers)


### Solution 2 Classical Active-Standby solution

![](https://user-images.githubusercontent.com/1715683/32590156-d88e6568-c553-11e7-9188-07eebbe207c7.png)

This solution is ideal for situations when the workload is not very high, this is the classical two node HA solution. The different compare with solution 1 is that it doen’t need the two extra loadblancer nodes and only need 1 static IP address. This solution doesn’t support scale out.

- **Keepalived**: The Keepalived will be installed both on VM1 and VM2, the two `Keepalived` will form a **VRRP** group to provide the VIP.
- **VIP**: User will access the harbor cluster by the VIP. Only the server that hold the VIP will provide the service.
- **Harbor instance1,2**: The harbor instance will share the VM1,2 with the Keepalived.
- **DB Cluster**: Same as the DB cluster in solution1
- **Shared storage**: Same as the shared storage in solution 1
- **Redis cluster**: Same as the Redis cluster in solution 1

#### Setup Prerequisites

- Standalone MariaDB (cluster) is needed
- Shared storage for registry is needed
- Redis (cluster) for Harbor session and registry metadata cache is needed.
- 2 Ubuntu16.04 VMs
- 1 Static IP address (used as VIP)

### Solution 3

Since the above two solution both require shared storage. For some scenario that don’t have shared storage but has some kind of HA requirement. We can use images replication.

![](https://user-images.githubusercontent.com/1715683/32590229-485dd112-c554-11e7-80ca-59308f63885d.png)

Set up a harbor as the **master** Harbor instance. All the images will be pushed to this Harbor instance. In the meanwhile, there are a group of **read-only** harbors, which serve the pull request from Docker hosts.

This solution is easy to implement and can met the requirement of load-balance and scale out.


### Solution 4

This is another solution that use the replication to achieve low level “HA”, same as solution 3, no need for shared storage external database cluster and Redis cluster. This solution can make the images under protection of single node failure.

![](https://user-images.githubusercontent.com/1715683/32590250-6f937f02-c554-11e7-849a-fa0d03ddd606.png)

As the figure shows it only need to setup a fully replication between two harbor nodes. This use case is more suitable for Geographically distributed team.


## 结论

### Classical Active-Active

- 适用于任何场景；
- 但要求的资源最多；
- 将 Harbor 拆分为无状态部分和有状态部分（数据相关）


### Classical Active-Standby

- 适用于工作负载不是很高的场景；
- 将 Harbor 拆分为无状态部分和有状态部分（数据相关）
- 不支持横向扩展，但节省了 LB 需要的机器和 static ip 等东东；
- 依赖 Keepalived (VIP) ；


### Master-Slaves Replication

- 适用于没有共享存储的场景；
- 基于主从单向复制实现 HA ；
- 一主多从，只有 master 接受 push 动作，所有 slave 只提供 read-only 的 pull 操作；
- 能够根据自定义配置决定镜像的分布情况，在一定程度上实现了负载均衡和扩展能力；但配置规则会有一定的维护成本；

### Master-Master Replication

- 适用于没有共享存储、没有外部 DB cluster 的场景；
- 适用于地理上分布在不同地点团队需要相互共享镜像的场景；
- 基于主主双向复制实现 HA ；
- 每个 Harbor 上都具有全量数据；