# Netfilter 与 Connection Tracking

问题列表：

- Netfilter 的构成
- Connection tracking 是什么？如何工作？
- iptables 和 netfilter 的关系
- netfilter 和 Connection tracking 的关系
- k8s 和 netfilter 的关系
- `/proc/net/nf_conntrack` 和 `/proc/net/tcp[6]` 中内容的关系


文章列表：

- [wiki/Netfilter](https://en.wikipedia.org/wiki/Netfilter)
- [Connection tracking](https://www.rigacci.org/wiki/lib/exe/fetch.php/doc/appunti/linux/sa/iptables/conntrack.html)
- [解决 nf_conntrack: table full, dropping packet 的几种思路](http://jaseywang.me/2012/08/16/%E8%A7%A3%E5%86%B3-nf_conntrack-table-full-dropping-packet-%E7%9A%84%E5%87%A0%E7%A7%8D%E6%80%9D%E8%B7%AF/)
- [iptables与netfilter的关系简单讲解](http://blog.csdn.net/x532943257/article/details/50750347)
- [Linux协议栈-netfilter(5)-iptables](http://blog.csdn.net/jasonchen_gbd/article/details/44877543)
- [一个复杂的nf_conntrack实例全景解析](http://blog.csdn.net/dog250/article/details/78372576)
- [nf_conntrack连接跟踪模块](http://blog.csdn.net/u010472499/article/details/78292811)
- [details of /proc/net/nf_conntrack](https://stackoverflow.com/questions/16034698/details-of-proc-net-ip-conntrack-nf-conntrack)
- [如何理解Netfilter中的连接跟踪机制](http://blog.csdn.net/dandelionj/article/details/8535990)
- [netfilter之conntrack笔记](http://blog.csdn.net/appletreesujie/article/details/6871011)
- [Connection tracking - CLOSE_WAIT and FIN_WAIT](http://lists.netfilter.org/pipermail/netfilter/2003-April/043510.html)
- [connection tracking query](http://lists.netfilter.org/pipermail/netfilter/2003-April/043383.html)
- [无状态TCP的ip_conntrack](http://blog.csdn.net/dog250/article/details/9318843)
- [networking/nf_conntrack](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)


----------


## [wiki/Netfilter](https://en.wikipedia.org/wiki/Netfilter)

> Netfilter is a **framework** provided by Linux that allows various **networking-related operations** to be implemented in the form of **customized handlers**. Netfilter offers various functions and operations for **packet filtering**, **network address translation**, and **port translation**, which provide the functionality required for directing packets through a network, as well as for providing ability to prohibit packets from reaching sensitive locations within a computer network.

- Netfilter 是 linux 提供的框架，基于定制化 handlers 形式提供各种网络相关操作；
- Netfilter 提供用于实现 packet filtering、NAT、port translation 等功能的各种函数和操作；进而实现了对 packets 流向的控制和阻止 packets 到达计算机网络内部特定位置的能力；

> Netfilter represents a set of **hooks** inside the Linux kernel, allowing specific kernel modules to **register callback functions with the kernel's networking stack**. Those functions, usually applied to the traffic in the form of **filtering** and **modification** rules, are called for every packet that traverses the respective hook within the networking stack.

- Netfilter 代表了一组 Linux kernel 内的 hooks ，允许特定 kernel 模块注册回调函数到 kernel 网络协议栈中；
- 这些函数通常以 **filtering** 和 **modification** 规则的形式作用于网络流量；在网络协议栈内部，对经过指定 hook 的每一个 packet 进行函数调用；

### Userspace utility programs

#### iptables

> The kernel modules named `ip_tables`, `ip6_tables`, `arp_tables` (the underscore is part of the name), and `ebtables` are some of the significant parts of the **Netfilter hook system**. They provide **a table-based system for defining firewall rules** that can **filter** or **transform** packets. The tables can be administered through the user-space tools `iptables`, `ip6tables`, `arptables`, and `ebtables`. Notice that although both the kernel modules and userspace utilities have similar names, each of them is a different entity with different functionality.

- Netfilter hook system 由 `ip_tables`, `ip6_tables`, `arp_tables` 和 `ebtables` 构成；
- 基于这套 hook 系统提供了一种 table-based 的 firewall 规则定义方案，用于对 packets 进行 filter 和 transform ；

> Each table is actually its own hook, and **each table was introduced to serve a specific purpose**. As far as Netfilter is concerned, it runs a particular table in a specific order with respect to other tables. Any table can call itself and it also can execute its own rules, which enables possibilities for additional processing and iteration.

- 每一种 table 都有自己的 hook 函数，每一种 table 的引入都为了一种特定的目的；
- 就 Netfilter 而言，其使用的 table ，以及使用的顺序，和其他使用是不同的；

> **Rules are organized into chains**, or in other words, "chains of rules". These chains are named with predefined titles, including INPUT, OUTPUT and FORWARD. These chain titles help describe the origin of the Netfilter stack. Packet reception, for example, falls into **PREROUTING**, while the **INPUT** represents locally delivered data, and forwarded traffic falls into the **FORWARD** chain. Locally generated output passes through the **OUTPUT** chain, and packets to be sent out are in **POSTROUTING** chain. **Netfilter modules** not organized into tables (see below) **are capable of checking for the origin to select their mode of operation**.

- Rules 被组织成 chains ，也即常说的“**规则链**”；
- 存在一些**预定义 chains** ，其名字用于辅助确定 origin 是哪里：
    - PREROUTING 对应 Packet reception
    - INPUT 对应 locally delivered data
    - FORWARD 对应 forwarded traffic
    - OUTPUT 对应 Locally generated output
    - POSTROUTING 对应 packets to be sent out
- Netfilter 模块能够检查出 origin 并选择相应的操作模式；
    
> - `iptable_raw` module
When loaded, registers a hook that will be called before any other Netfilter hook. It provides a table called `raw` that can be used to **filter** packets before they reach more memory-demanding operations such as **Connection Tracking**.

`iptable_raw` 模块：

- 注册一个 hook
- 在任何其他 Netfilter hook 之前被调用
- 提供一个 `raw` table 用于对 packet 进行 filter
- 在 Connection Tracking 之前执行

> - `iptable_mangle` module
Registers a hook and `mangle` table to run after **Connection Tracking** (see below) (but still before any other table), so that modifications can be made to the packet. This enables additional modifications by rules that follow, such as NAT or further filtering.

`iptable_mangle` 模块：

- 注册一个 hook
- 提供一个 `mangle` table
- 在 Connection Tracking 之后执行，但在任何其他 table 之前执行
- 可用于对 packet 进行修改

> - `iptable_nat` module
Registers two hooks: **DNAT-based transformations** (or "Destination NAT") are applied before the filter hook, **SNAT-based transformations** (for "Source NAT") are applied afterwards. The `nat` table (or "network address translation") that is made available to iptables is merely a "configuration database" for NAT mappings only, and not intended for filtering of any kind.

`iptable_nat` 模块：

-  注册两个 hook
    - **DNAT**（在 filter hook 之前执行）
    - **SNAT**（在 filter hook 之后执行）
- 提供一个 `nat` table ，但仅用作 iptables 实用程序的、NAT 映射配置数据库，而不被用于 filtering 功能；

> - `iptable_filter` module
Registers the `filter` table, used for general-purpose filtering (firewalling).

`iptable_filter` 模块：

- 提供一个 `filter` table
- 用于通用目的 filtering 功能（firewalling）

> - `security_filter` module
Used for Mandatory Access Control (MAC) networking rules, such as those enabled by the SECMARK and CONNSECMARK targets. (These so-called "targets" refer to Security-Enhanced Linux markers.) Mandatory Access Control is implemented by Linux Security Modules such as SELinux. The security table is called following the call of the filter table, allowing any Discretionary Access Control (DAC) rules in the filter table to take effect before any MAC rules. This table provides the following built-in chains: INPUT (for packets coming into the computer itself), OUTPUT (for altering locally-generated packets before routing), and FORWARD (for altering packets being routed through the computer).

`iptable_filter` 模块：

- 略

#### nftables

> `nftables` is intended to replace Netfilter, as the new general-purpose in-kernel packet classification engine. `nft`, as the new userspace utility, is intended to replace iptables, ip6tables, arptables and ebtables.

### Packet defragmentation

> The `nf_defrag_ipv4` module will **defragment** IPv4 packets before they reach Netfilter's **connection tracking** (`nf_conntrack_ipv4` module). This is necessary for the in-kernel connection tracking and NAT helper modules (which are a form of "mini-ALGs") that only work reliably on entire packets, not necessarily on fragments.

> The IPv6 defragmenter is not a module in its own right, but is integrated into the `nf_conntrack_ipv6` module.

- `nf_defrag_ipv4` 模块用于 IPv4 packets 进入 connection tracking 前对其进行**分片重组**；
- 该重组对于 connection tracking helper 和 NAT helper 模块非常必要，因为它们均依赖完整的 packets 工作；

### Connection tracking

> One of the important features built on top of the Netfilter framework is **connection tracking**. Connection tracking allows the kernel to keep track of all logical network connections or sessions, and thereby relate all of the packets which may make up that connection. `NAT` relies on this information to translate all related packets in the same way, and `iptables` can use this information to act as a stateful firewall.

- connection tracking 构建于 Netfilter framework 之上；
- Connection tracking 允许 kernel 跟踪所有逻辑网络链接或会话，由此将构成 connection 的所有 packets 分别关联起来；
- **NAT 依赖 Connection tracking 中的信息对所有相关 packets 进行 translate ，之后 iptables 使用 NAT 信息构建 stateful firewall** ；

> **The connection state however is completely independent of any upper-level state**, such as **TCP**'s or **SCTP**'s state. Part of the reason for this is that when merely forwarding packets, i.e. no local delivery, the TCP engine may not necessarily be invoked at all. Even connectionless-mode transmissions such as **UDP**, **IPsec** (AH/ESP), **GRE** and other tunneling protocols have, at least, a pseudo connection state. The heuristic for such protocols is often based upon a preset timeout value for inactivity, after whose expiration a Netfilter connection is dropped.

- connection state 和任何上层协议（例如 TCP 等）状态是完全不相关的（例如 forwarding 功能完全不需要经过 TCP 协议栈）；
- connection state 至少为无连接传输协议（例如 UDP、IPsec (AH/ESP)、GRE 和其他隧道协议）提供了 pseudo 连接状态；因此针对不同协议的启发式算法就可以非活跃时间进行超时判定，并在超时后丢弃相应的 Netfilter connection ；

> **Each `Netfilter connection` is uniquely identified by a** (layer-3 protocol, source address, destination address, layer-4 protocol, layer-4 key) **tuple**. The layer-4 key depends on the transport protocol; for TCP/UDP it is the port numbers, for tunnels it can be their tunnel ID, but otherwise is just zero, as if it were not part of the tuple. To be able to inspect the TCP port in all cases, packets will be mandatorily defragmented.

- 每一个 Netfilter connection 由一个唯一的 tuple 确定；

> Netfilter connections can be manipulated with the user-space tool `conntrack`.

> iptables can make use of checking the connection's information such as **states**, **statuses** and more to make packet filtering rules more powerful and easier to manage. The most common states are:

> - **NEW**
> trying to create a new connection
>
> - **ESTABLISHED**
> part of an already-existing connection
>
> - **RELATED**
> assigned to a packet that is initiating a new connection and which has been "expected"; the aforementioned mini-ALGs set up these expectations, for example, when the `nf_conntrack_ftp` module sees an FTP "PASV" command
>
> - **INVALID**
> the packet was found to be invalid, e.g. it would not adhere to the TCP state diagram
>
> - **UNTRACKED**
a special state that can be assigned by the administrator to bypass connection tracking for a particular packet (see `raw` table, above).
> 
> A normal example would be that the first packet the conntrack subsystem sees will be classified "**new**", the reply would be classified "**established**" and an ICMP error would be "**related**". An ICMP error packet which did not match any known connection would be "**invalid**".

#### Connection tracking helpers

> Through the use of plugin modules, **connection tracking can be given knowledge of application-layer protocols** and thus understand that two or more distinct connections are "related". For example, consider the **FTP** protocol. A control connection is established, but whenever data is transferred, a separate connection is established to transfer it. When the `nf_conntrack_ftp` module is loaded, the first packet of an FTP data connection will be classified as "related" instead of "new", as it is logically part of an existing connection.

- connection tracking 能够“了解”应用层协议内容，由此对不同 connections 进行 related ；

> **The helpers only inspect one packet at a time, so if vital information for connection tracking is split across two packets, either due to `IP fragmentation` or `TCP segmentation`, the helper will not necessarily recognize patterns and therefore not perform its operation**. IP fragmentation is dealt with the connection tracking subsystem requiring **defragmentation**, though TCP segmentation is not handled. In case of FTP, segmentation is deemed not to happen "near" a command like PASV with standard segment sizes, so is not dealt with in Netfilter either.

- helpers 一次只检测一个 packet ，因此，若 connection tracking 相关的关键信息恰好被查分到两个 packet 中时，helper 将无法识别模式信息，也就无法正常完成工作了；

### Network Address Translation

> Each connection has a set of **original addresses** and **reply addresses**, which initially start out the same. **NAT** in Netfilter is implemented by simply changing the reply address, and where desired, port. When packets are received, their connection tuple will also be compared against the reply address pair (and ports). **Being fragment-free is also a requirement for NAT**. (If need be, IPv4 packets may be refragmented by the normal, non-Netfilter, IPv4 stack.)

- 不能分片（fragment-free）也是 NAT 的要求之一；

#### NAT helpers

> Similar to connection tracking helpers, NAT helpers will do a packet inspection and substitute original addresses by reply addresses in the payload.

NAT helpers 提供了 packet 探测和源地址替换功能；

PNG 版本

![Netfilter Components](https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Netfilter-components.svg/1280px-Netfilter-components.svg.png)

SVG 版本

![Netfilter Components](https://upload.wikimedia.org/wikipedia/commons/d/dd/Netfilter-components.svg)

PNG 版本

![Packet flow in Netfilter and General Networking](https://upload.wikimedia.org/wikipedia/commons/thumb/3/37/Netfilter-packet-flow.svg/1450px-Netfilter-packet-flow.svg.png)

SVG

![Packet flow in Netfilter and General Networking](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)


----------


## [Connection tracking](https://www.rigacci.org/wiki/lib/exe/fetch.php/doc/appunti/linux/sa/iptables/conntrack.html)

### What is connection tracking?

> `Connection tracking` refers to the ability to maintain state information about a connection in memory tables, such as source and destination ip address and port number pairs (known as socket pairs), protocol types, connection state and timeouts. **Firewalls that do this are known as stateful**. `Stateful firewalling` is inherently more secure than its "stateless" counterpart .... simple packet filtering.

- Connection tracking 的含义
- Stateful firewalling 和 stateless firewalling 的区别

> Connection tracking is accomplished with the `state` option in `iptables`. From the `iptables` manpage:
>
>> state 
>>   This module, when combined with **connection tracking**, allows access to the 
>>   connection tracking state for this packet.
>>
>>   --state state 
>>    Where state is a comma separated list of the connection states to 
>>    match.  Possible states are **INVALID** meaning that the packet is 
>>    associated with no known connection, **ESTABLISHED** meaning that the 
>>    packet is associated with a connection which has seen packets in 
>>    both directions, **NEW** meaning that the packet has started a new con­ 
>>    nection, or otherwise associated with a connection which has not 
>>    seen packets in both directions,  and **RELATED** meaning that the 
>>    packet is starting a new connection, but is associated with an 
>>    existing connection, such as an FTP data transfer, or an ICMP 
>>    error.

- **Connection tracking 是通过 iptables 的 state 选项实现的**；
- state 选项和 connection tracking 功能结合后能够针对 packet 的状态进行跟踪；
- 可用状态包括：INVALID、ESTABLISHED、NEW 和 RELATED ；

> Connection tracking is done either in the **PREROUTING** chain, or the **OUTPUT** chain for locally generated packets. 

- 连接跟踪发生的位置为 **PREROUTING** 或 **OUTPUT** 链中；

> **Connection tracking defragments all packets before tracking their state**. This explains why there is no `ip_always_defrag` switch as there was in the 2.2 kernel.

> The state table for `udp` and `tcp` connections is maintained in `/proc/net/ip_conntrack`. We will discuss what its contents look below.

- 维护 udp 和 tcp 连接信息的状态表位于 `/proc/net/ip_conntrack` ；

> The maximum number of connections the state table can contain is stored in `/proc/sys/net/ipv4/ip_conntrack_max`. This value is determined initially by how much physical memory you have (on my 128Mb machine, `ip_conntrack_max = 8184` by default). 

- 已经变更为 `/proc/sys/net/nf_conntrack_max` ；
 
## How does connection tracking work?

### A quick overview

> Lets get our bearings first with respect to the whole `netfilter` framework before we delve deeper. For a packet forwarded between interfaces the sequence of chain negotiation would be:
>
> - `PREROUTING chain` - **DNAT** the packet if necessary. **Mangle** the packet if necessary. Connection tracking now **defragments** and **tracks** (classifies) the packet in some way:
>
>> If the packet matches a entry in the state table it is part of an **ESTABLISHED** connection. If it is `icmp` traffic it might be **RELATED** to a udp/tcp connection already in the state table. The packet might be starting a **NEW** connection, or it might be unrelated to any connection in which case it is deemed **INVALID**.
> 
> - `FORWARD chain` - Compare the packet state against the ruleset in the filter table until the first match, or until the 
default policy of the chain is executed.
> 
> - `POSTROUTING chain` - **SNAT** the packet if necessary.
>
> Note that **all packets are compared against the ruleset in the filter table**. This is easily proved - If you have entries in the state table and you change the rules to deny all traffic, then although the entries in the state table remain, all traffic is indeed denied as it should be.

### More detail

> We will consider each of the three protocols, udp, tcp and icmp in turn.

#### UDP

> Because it lacks sequence numbers, udp is known as a "stateless" protocol . However, this does not mean we can't track udp connections. There is still other useful information we can utilize. Here is an example state table entry for a newly formed udp connection:

> ```
> udp      17 19 src=192.168.1.2 dst=192.168.1.50 sport=1032 dport=53 [UNREPLIED] src=192.168.1.50 dst=192.168.1.2 sport=53 dport=1032 use=1
> ```

> This state table entry can only be made if there is an `iptables` filter rule specifying **NEW** connections, something like the following ruleset, which **allows NEW connections outbound only** (as is often wise):
>
> ```
> iptables -A INPUT  -p udp -m state --state ESTABLISHED -j ACCEPT 
> iptables -A OUTPUT -p udp -m state --state NEW,ESTABLISHED -j ACCEPT
> ```
>
> Things we can tell from the state table entry are as follows:
> 
> - The protocol is udp (IP protocol **number 17**).
> - The state table entry has **19 seconds** until it expires.
> - Source and destination addresses and ports of **original** query.
> - Source and destination addresses and ports of **expected** reply. The connection is marked **UNREPLIED** so this has not been received yet.

> **Udp timeouts** are set in `/usr/src/linux/net/ipv4/netfilter/ip_conntrack_proto_udp.c` at compile time. 
>
> Here is the relevant section of code:
> 
> ```
> #define UDP_TIMEOUT (30*HZ) 
> #define UDP_STREAM_TIMEOUT (180*HZ)
> ```

UDP 超时常量；

> A **single request** will enter into the state for 30*HZ (generally 30 seconds). In the example above, where we have 19 seconds left, 11 seconds have already elapsed without a reply being received. Once a reply is received, and allowed by a rule permitting ESTABLISHED connections, the timeout is reset to 30 seconds and the UNREPLIED mark is removed. Here we see the connection a couple of seconds after this has taken place:
>
> ```
> udp      17 28 src=192.168.1.2 dst=192.168.1.50 sport=1032 dport=53 src=192.168.1.50 dst=192.168.1.2 sport=53 dport=1032 use=1
> ```

single request 受 UDP_TIMEOUT 超时时间限制；

> If **multiple requests** and replies occur between the same socket pairs, the entry is considered to be a stream and the timeout changes to 180 seconds. At this point the entry is marked ASSURED (once connections become ASSURED they are not dropped under heavy load ). Here we see the connection a few of seconds after this has taken place:
> 
>```
> udp      17 177 src=192.168.1.2 dst=192.168.1.50 sport=1032 dport=53 src=192.168.1.50 dst=192.168.1.2 sport=53 dport=1032 [ASSURED] use=1
>```
>
> There is no absolute timeout for a udp connection (or a tcp connection for that matter), provided traffic keeps flowing. 

multiple requests 构成 stream ，受 UDP_STREAM_TIMEOUT 超时时间限制；

#### TCP

> A tcp connection is initiated via a three-way handshake involving a synchronization request from the client, a synchronization and an acknowledgement from the server, and finally an acknowledgement from the client. Subsequent traffic flowing between server and client is acknowledged in all cases. The sequence looks like
>
> ```
>     Client                     Server
>
>      SYN      ---> 
>                     <---     SYN+ACK 
>      ACK      ---> 
>                     <---         ACK 
>      ACK      ---> 
>                     ......... 
>                     .........
> ```
>
> SYN and ACK refer to flags set in the tcp header. There are also 32 bit sequence and acknowledgement numbers stored in the tcp header which are passed back and forth and updated during the session.

基础知识

> **To get connection tracking to work for a tcp connection** you need a ruleset like this:
>
> ```
> iptables -A INPUT  -p tcp -m state --state ESTABLISHED -j ACCEPT 
> iptables -A OUTPUT -p tcp -m state --state NEW,ESTABLISHED -j ACCEPT
> ```

针对 tcp 的连接跟踪功能；

##### Walkthrough

以下内容详细描述了 tcp 连接建立过程中各个阶段上连接状态表中的内容；

> What we are going to do now is walk and talk through the establishment of a normal tcp connection and look at the state table at each stage:

> - **Once an initial SYN is sent in the OUTPUT chain, and accepted by out rule that allows the NEW connection**, the connection table entry may look something like:
>
> ```
> tcp      6 119 SYN_SENT src=140.208.5.62 dst=207.46.230.218 sport=1311 dport=80 [UNREPLIED] src=207.46.230.218 dst=140.208.5.62 sport=80 dport=1311 use=1
> ```
>
> The tcp connection status is `SYN_SENT` and the connection is marked **UNREPLIED**.

> - We are now **waiting for a SYN+ACK to arrive** at which point the tcp connection state changes to `SYN_RECV` and the **UNREPLIED** disappears:
>
> ```
> tcp      6 57 SYN_RECV src=140.208.5.62 dst=207.46.230.218 sport=1311 dport=80 src=207.46.230.218 dst=140.208.5.62 sport=80 dport=1311 use=1
> ```

> - We are now **waiting for the final part of the handshake, an ACK**. When it arrives, we check that it's sequence number matches the ACK of the handshake from the server to the client. The tcp connection state now becomes `ESTABLISHED` and the state table entry is marked **ASSURED** (ASSURED connections are not dropped from the state table when the connection is under load). Here we see the ESTABLISHED connection:
>
> ```
> tcp      6 431995 ESTABLISHED src=140.208.5.62 dst=207.46.230.218 sport=1311 dport=80 src=207.46.230.218 dst=140.208.5.62 sport=80 dport=1311 [ASSURED] use=1
> ```

##### Connection tracking's perspective on the state table

> We just talked a lot about tcp connection states. Now let's think about this from the perspective of the connection tracking:
>
> Connection tracking only knows about **NEW**, **ESTABLISHED**, **RELATED** and **INVALID**, classified as described above and in the iptables manpage. To quote Joszef Kadlecsik, who helped me out with a confusion I had initially about this very subject:
>
>> When a packet with the SYN+ACK flags set arrives in response to a packet with SYN set, the connection tracking thinks: "I have been just seeing a packet with SYN+ACK which answers a SYN I had previously seen, so this is an ESTABLISHED connection."
>
> The important point here is that **the conntrack states are not equivalent to tcp states**. We have already seen that a connection doesn't achive the tcp connection status of ESTABLISHED until the ACK after the SYN+ACK has been received.

conntrack states 和 tcp states 是不等价的；

> **The representation of the tcp connection states in the state table is purely for timeouts**. You can prove this to yourself by sending an ACK packet through your firewall to a non-existent machine (so that you don't get the RST back). It will create a state table entry no problem because it is the first packet of a connection and so is treated as NEW (the entry will not be marked as ASSURED though).  Checkpoint's Firewall-1 version 4.1 SP1 allows connection initiation by ACK packets too (see Lance Spitzner's [whitepaper](http://www.enteract.com/~lspitz/fwtable.html) for details).

连接状态表中针对 tcp 连接的描述纯粹是用作 timeout 用途；

> In the light of the fact that **ACK packets can create state table entries**, the following contribution from Henrik Nordstrum is insightful: **To make sure that NEW tcp connections are packets with SYN set**, use the following rule:
>
> ```
> iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
> ```

- ACK 包能够导致 state table entries 的创建
- 为了确保 NEW 状态的 tcp connections 是基于 SYN 包创建的，则需要设置上述 iptables 规则；

> Note that **doing this will prevent idle sessions from continuing once they have expired from the conntrack table**. In the normal "relaxed" view such connections initiated from the correct direction (i.e. the direction you allow NEW packets through) can normally continue even if expired from conntrack, provided that the first data/ack packet that resumes the connection comes from the correct direction.
>
> If you want real stateful filtering that requires correct connection initiation and tracks sequence numbers, apply the 
tcp-window-tracking patch from patch-o-matic. A very detailed paper describing the patch can be found [here](http://www.iae.nl/users/guido/papers/tcp_filtering.ps.gz).

##### Timeouts

> Something to note is that **timeouts are reset to the maximum each time a connection sees traffic**. Timeouts are set in `/usr/src/linux/net/ipv4/netfilter/ip_conntrack_proto_tcp.c` at compile time. Here is the relevant section of code:
>
> ```
> static unsigned long tcp_timeouts[] 
> = { 30 MINS,    /*      TCP_CONNTRACK_NONE,     */ 
>     5 DAYS,     /*      TCP_CONNTRACK_ESTABLISHED,      */ 
>     2 MINS,     /*      TCP_CONNTRACK_SYN_SENT, */ 
>     60 SECS,    /*      TCP_CONNTRACK_SYN_RECV, */ 
>     2 MINS,     /*      TCP_CONNTRACK_FIN_WAIT, */ 
>     2 MINS,     /*      TCP_CONNTRACK_TIME_WAIT,        */ 
>     10 SECS,    /*      TCP_CONNTRACK_CLOSE,    */ 
>     60 SECS,    /*      TCP_CONNTRACK_CLOSE_WAIT,       */ 
>     30 SECS,    /*      TCP_CONNTRACK_LAST_ACK, */ 
>     2 MINS,     /*      TCP_CONNTRACK_LISTEN,   */ 
> };
> ```

> There is no absolute timeout for a connection.

##### Connection termination

> Connection termination occurs in two ways. **Natural termination** at the end of a session occurs when the client sends a packet with the FIN and ACK flags set. The closure proceeds as follows:
>
> ```
       Client                    Server 
                   ......... 
                   ......... 
        FIN+ACK   ---> 
                        <---        ACK 
                        <---     FIN+ACK 
         ACK      --->
> ```

> Sometime during, or at the end of this sequence the state table connection status changes to **TIME_WAIT** and the entry is removed after 2 minutes by default.
> 
> Another way for connection termination to occur is if either party sends a packet with the RST (reset) flag set. RST's are not acknowledged. In this case the state table connection status changes to **CLOSE** and times out from the state table after 10 seconds. **This often happens with http entries**, where the server sends an RST after a period of inactivity. 

#### ICMP

> In `iptables` parlance, there are **only four types of icmp** that can be categorized as **NEW**, or **ESTABLISHED**:
>
> - **Echo request** (ping, 8) and echo reply (pong, 0). 
> - **Timestamp request** (13)and reply (14). 
> - **Information request** (15) and reply (16). 
> - **Address mask request** (17) and reply (18).

> The request in each case is classified as **NEW** and the reply as **ESTABLISHED**. 

只有四种类型的 icmp 可以被归类为 NEW 或 ESTABLISHED ；

> Other types of icmp are not request-reply based and can only be **RELATED** to other connections.

其他类型 icmp 只能被归为 RELATED ；

> Let us consider a sample ruleset and a few examples:
>
> ```
> iptables -A OUTPUT -p icmp -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT 
> iptables -A INPUT  -p icmp -m state --state ESTABLISHED,RELATED  -j ACCEPT
> ```

> - An icmp **echo request** is **NEW** and so is allowed in the OUTPUT chain.
> - An icmp **echo reply**, provided it is in response to an echo request, is **ESTABLISHED** and so is allowed in the INPUT chain. An echo reply cannot be allowed in the OUTPUT chain for the rules above because there is no NEW in the INPUT chain to allow echo requests and a reply has to be in response to a request.
> - An icmp **redirect**, because it is not request-reply based, is RELATED and so can be allowed in both the INPUT and the OUTPUT chains provided there is already a tcp or udp connection in the state table already that it can be matched against. 


#### Connection tracking and ftp

> Firstly, you need to load the `ip_conntrack_ftp` module.

> Assuming you have a single-homed box, a simple ruleset to allow an ftp connection would be:
>
> ```
> iptables -A INPUT  -p tcp --sport 21 -m state --state ESTABLISHED -j ACCEPT 
> iptables -A OUTPUT -p tcp --dport 21 -m state --state NEW,ESTABLISHED -j ACCEPT
>```

> (Please note, I am assuming here you have a separate ruleset to allow any icmp **RELATED** to the connection. Please see my example ruleset for this).

> This is not the whole story. An ftp connection also needs a data-channel, which can be provided in one of two ways:

> - Active ftp
>> The ftp client sends a port number over the ftp channel via a PORT command to the ftp server. The ftp server then connects from port 20 to this port to send data, such as a file, or the output from an ls command. The ftp-data connection is in the opposite sense from the original ftp connection.
>>
>> To allow active ftp without knowing the port number that has been passed we need a general rule which allows connections from port 20 on remote ftp servers to high ports (port numbers > 1023) on ftp clients. This is simply too general to ever be secure.
>>
>> Enter the `ip_conntrack_ftp` module. This module is able to recognize the PORT command and pick-out the port number. As such, the ftp-data connection can be classified as RELATED to the original outgoing connection to port 21 so we don't need NEW as a state match for the connection in the INPUT chain. The following rules will serve our purposes grandly:
>>
>> ```
>> iptables -A INPUT     -p tcp --sport 20 -m state --state ESTABLISHED,RELATED -j ACCEPT 
>> iptables -A OUTPUT -p tcp --dport 20 -m state --state ESTABLISHED -j ACCEPT
>> ```

> - Passive ftp
>> A PORT command is again issued, but this time it is from the server to the client. The client connects to the server for data transfer. Since the connection is in the same sense as the original ftp connection,  passive ftp is inherently more secure than active ftp, but note that this time we know even less about the port numbers. Now we have a connection between almost arbitrary port numbers.
>>
>> Enter the ip_conntrack_ftp module once more. Again, this module is able to recognize the PORT command and pick-out the port number. Instead of NEW in the state match for the OUTPUT chain, we can use RELATED. The following rules will suffice:
>>
>> ```
>> iptables -A INPUT     -p tcp --sport 1024: --dport 1024:  -m state --state ESTABLISHED -j ACCEPT 
>> iptables -A OUTPUT -p tcp --sport 1024: --dport 1024:  -m state --state ESTABLISHED,RELATED -j ACCEPT 
>> ```



----------


## [解决 nf_conntrack: table full, dropping packet 的几种思路](http://jaseywang.me/2012/08/16/%E8%A7%A3%E5%86%B3-nf_conntrack-table-full-dropping-packet-%E7%9A%84%E5%87%A0%E7%A7%8D%E6%80%9D%E8%B7%AF/)

- `nf_conntrack` 工作在 3 层（网络层），支持 IPv4 和 IPv6，而 `ip_conntrack` 只支持 IPv4 ；
- 目前，大多的 `ip_conntrack_*` 已被 `nf_conntrack_*` 取代，很多 `ip_conntrack_*` 仅仅是个 alias，原先的 `ip_conntrack` 的 `/proc/sys/net/ipv4/netfilter/` 依然存在，但是新的 `nf_conntrack` 在 `/proc/sys/net/netfilter/` 中，这个应该是做个向下的兼容；
- **`nf_conntrack`/`ip_conntrack` 跟 nat 有关，用来跟踪连接条目**，它会使用一个哈希表来记录 established 的记录。`nf_conntrack` 在 2.6.15 被引入，而 `ip_conntrack` 在 2.6.22 被移除，如果该哈希表满了，就会出现："**nf_conntrack: table full, dropping packet**" ；
- 解决上述问题的几种思路：
    - 不使用（移除） nf_conntrack 模块；
    - 调大 `/proc/sys/net/netfilter/nf_conntrack_max` 的值；缩小 `/proc/sys/net/ipv4/netfilter/ip_conntrack_tcp_timeout_established` 的值；
    - 使用 raw 表，不跟踪连接：**iptables 中的 raw 表跟包的跟踪有关，基本就是用来干一件事，通过 `NOTRACK` 给不需要被连接跟踪的包打标记**，也就是说，如果一个连接遇到了 `-j NOTRACK`，conntrack 就不会跟踪该连接，raw 的优先级大于 mangle, nat, filter，包含 PREROUTING 和 OUTPUT 链。当执行 `-t raw` 时，系统会自动加载 `iptable_raw` 模块（需要该模块存在）。raw 在 2.4 以及 2.6 早期的内核中不存在，除非打了 patch，目前的系统应该都有支持；
- 上面三种方式，最有效的是 1 跟 3，第二种治标不治本；
    


----------


## [iptables与netfilter的关系简单讲解](http://blog.csdn.net/x532943257/article/details/50750347)

- **什么是 iptables？**

iptables 是 Linux 下功能强大的**应用层防火墙工具（定义规则的配置工具）**，但了解其规则原理和基础后，配置起来也非常简单。

- **什么是 Netfilter？**

说到 iptables 必然提到 Netfilter ，iptables 是应用层的，其实质是一个定义规则的配置工具，而**核心的数据包拦截和转发**是 Netfilter 。

Netfilter 是 Linux 操作系统核心层内部（即实现于 kernel 中）的一个数据包处理模块。

iptables 和 Netfilter 关系图：

![](https://files.jb51.net/file_images/article/201506/2015615161042754.jpg)

在这张图可以看出，**Netfilter 作用于网络层**，数据包通过网络层会经过 Netfilter 的五个挂载点（Hook point）：**PRE_ROUTING**、**INPUT**、**OUTPUT**、**FORWARD**、**POST_ROUTING**。

任何一个数据包，只要经过本机，必将（至少）经过这五个挂载点的其中一个。

### iptables 规则原理

iptables 的规则组成，又被称为**四表五链**：

- 四张表（table） + 五个挂载点（hook） + 规则（rule）
- 四张表：`filter` 表、`nat` 表、`mangle` 表、`raw` 表
- 五个挂载点：**PRE_ROUTING**、**INPUT**、**OUTPUT**、**FORWARD**、**POST_ROUTING**
- 具体来说，就是 **iptables 设置的每一条允许、拒绝或转发的规则必须选择一个挂载点，关联一张表**；
- **规则**代表了对数据包的具体操作，**挂载点**代表了操作的位置，**表**代表了作用的目的；

iptables 四张表的用途（现在用的比较多的表是前两个）：

- `filter` 表用于过滤；
- `nat` 表用于地址转换；
- `mangle` 表用于修改数据包；
- `raw` 表一般用于不再让 iptables 做数据包的链接跟踪（connection tracking）处理，以便跳过其他表，提高性能；

### 数据包在规则表、挂载点的匹配流程图

下图是**数据包经过挂载点的流向图**，在每个挂载点上标识了有哪些表可用于定义规则：

![](https://files.jb51.net/file_images/article/201506/2015615161106573.jpg)

- `filter` 表一般只能用于 INPUT、FORWARD、OUTPUT ；
- `nat` 表一般也只能用于 PREROUTING、OUTPUT、POSTROUTING ；


----------


## [Linux协议栈-netfilter(5)-iptables](http://blog.csdn.net/jasonchen_gbd/article/details/44877543)

> 原文结合源码进行的讲解，非常清晰；

**iptables 是用户态的配置工具，用于实现网络层的防火墙**，用户可以通过 iptables 命令设置一系列的过滤规则，来截获特定的数据包并进行过滤或其他处理。

**iptables 命令通过与内核中的 netfilter 交互来起作用**。我们知道 netfilter 通过挂在每个 hook 点上的 hook 函数来过滤数据包，并且将过滤规则存放在几个表中供 hook 函数使用。相应的，iptables 工具也定义了同样的几张规则表来对应 netfilter 中的表，以及定义了不同的链（chain）来对应 netfilter 中的 hook 点。这样，通过 iptables 命令生成的规则可以很容易的作用到内核中的 netfilter 模块，netfilter 根据这些规则做真正的过滤工作。

> 信息梳理：
> 
> - netfilter 和 iptables 分别定义了规则表，用于相互对应；
> - netfilter 中的 hook 点对应了 iptables 中定义的链（chain）；

**netfilter 模块中，不同的协议族定义了各自的一套 hook 点**，例如 IPv4、IPv6、ARP、BRIDGE 等协议族都分别定义了各自的 netfilter 处理流程以及各自的 hook 点，我们比较常见的就是 IPv4 协议定义的 5 个 hook 点（**PRE_ROUTING**，**LOCAL_IN**，**LOCAL_OUT**，**FORWARD** 和 **POST_ROUTING**）。同样的，为了和 netfilter 交互，在用户态分别有 iptables、ip6tables、arptables、ebtables 等命令来定义各协议族的过滤规则。

> 可以看到 netfilter 中 hook 点的命名和 iptables 说的 hook 点的命名是有略微不同的；

要使用 iptables 命令，则必须将 netfilter 模块编进内核。本文内容基于内核版本 2.6.31 。

### 数据结构

- netfilter 中的每个表都定义成一个 `struct xt_table` 类型的结构体；
    - struct xt_table  *iptable_filter;
    - struct xt_table  *iptable_mangle;
    - struct xt_table  *iptable_raw;
    - struct xt_table  *nat_table;
- 表中的实际规则就放在 xt_table 结构中的 `struct xt_table_info *private;` 成员里；通过 private 就可以定位到规则的实际内容了；

相关结构细节如下

```
struct xt_table  
{  
    struct list_head list;  
    /* 在哪些 hook 点上注册了 hook 函数，是一个位图 */  
    unsigned int valid_hooks;  
    /* 表的实际数据（规则） */  
    struct xt_table_info *private;  
    struct module *me;  
    /* 所属协议族 */  
    u_int8_t af;  
    /* 表名，供用户空间设置 iptables 规则，或者内核匹配 iptables 规则 */  
    const char name[XT_TABLE_MAXNAMELEN];  
...
/* The table itself */  
struct xt_table_info  
{  
    /* 表的大小 */  
    unsigned int size;  
    /* Number of entries，即规则的数量 */  
    unsigned int number;  
    /* Initial number of entries，一般为上一次修改规则时的 number */  
    unsigned int initial_entries;  
  
    /* 在每个 hook 点作用的 entry 的偏移（注意是相对于最后一个参数 entry 的偏移，即第一个 hook 的第一个 ipt_entry 的 hook_entry 为 0） */  
    unsigned int hook_entry[NF_INET_NUMHOOKS];  
  
    /* 规则表的最大下界 */  
    unsigned int underflow[NF_INET_NUMHOOKS];  
  
    /* 规则表入口，即真正的规则存储结构. 在遍历一个规则表时，以此作为表的起始（即第一个 ipt_entry）。由定义可知这是一个数组，每个元素对应每个 CPU 上的规则 入口 */  
    void *entries[1];  
}; 
```

例如 NAT 表的定义如下

```
static struct xt_table nat_table = {  
    .name       = "nat",  
    .valid_hooks    = (1 << NF_INET_PRE_ROUTING) | \  
             (1 << NF_INET_POST_ROUTING) | \  
             (1 << NF_INET_LOCAL_OUT),  
    .me     = THIS_MODULE,  
    .af     = AF_INET,  
};  
```

由此可知，nat_table 在定义时初始化好了 4 个成员。而在注册时会赋值 list 成员，在添加新规则时会给 private 赋值。

- 一条 iptables 规则包括三个部分：ipt_entry、ipt_entry_matches、ipt_entry_target。
    - ipt_entry_matches 由多个 ipt_entry_match 组成；
    - ipt_entry 结构主要保存标准匹配的内容；
    - ipt_entry_match 结构主要保存扩展匹配的内容；
    - ipt_entry_target 结构主要保存规则的动作；

```
struct ipt_entry  
{  
    /* 将要进行匹配动作的 IP 数据报报头的描述 */  
    struct ipt_ip ip;  
    /* 经过这个规则后数据报的状态：未改变，已改变，不确定 */  
    unsigned int nfcache;  
    /* Size of ipt_entry + matches，因为 target 在 matches 的后面，即得到 target 的位置 */  
    u_int16_t target_offset;  
    /* Size of ipt_entry + matches + target，即下一个 ipt_entry 的开始 */  
    u_int16_t next_offset;  
    /* 指向数据报所经历的上一个规则地址。还可以作为 hook mask 表示规则作用于那个 hook 点上 */  
    unsigned int comefrom;  
    /* 匹配这个规则的数据报的计数以及字节计数 */  
    struct xt_counters counters;  
    /* 存放 matches（一条规则可有 0 到多个 match）和一个 target */  
    unsigned char elems[0];  
}; 
```

由此可知，结构体中的 elems 成员用于定位规则的 matches ，target_offset 用于定位规则的 target ，next_offset 指向下一条规则入口即下一个 ipt_entry 。这样，只要定位到一个表中第一个规则的 ipt_entry ，就可以找到这个表中的所有规则了。

**一个表中可以有多条规则**，下图以 IPv4 上注册的表以及 nat 表的规则为例，说明了表中规则的存放形式，该图显示了 IPv4 协议有 filter 和 nat 等规则表，并显示了 nat 表的规则的存放位置。

![](http://img.blog.csdn.net/20150404223930712?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFzb25jaGVuX2diZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

NAT 表允许在 PRE_ROUTING、LOCAL_OUT、POST_ROUTING 三个链上设置规则，所以，`struct xt_table_info` 的 `hook_entry[]` 和 `underflow[]` 成员分别有三个数组元素，用来定位每个链（即 hook 点）上的第一条规则和最后一条规则。

每个链可以有多条规则，每条规则都是由 entry + matches + target 组成的，所以在遍历每个链上的规则时，就根据 `struct ipt_entry` 来定位每条规则的位置。

- 用 iptables 命令还可以**创建自定义的子链**，例如用户新建一个自定义子链 NEW_PRE_CHAIN ：

```
iptables -N NEW_PRE_CHAIN
```

- 然后设置了两条规则添加到 NEW_PRE_CHAIN 子链上；
- 接着在 PREROUTING 链（对应 netfilter 的 hook 点 NF_INET_PRE_ROUTING）上追加一条跳转到 NEW_PRE_CHAIN 子链的规则，并将这条规则放到 NAT 表中，例如：

```
iptables -t nat -A PREROUTING -i eth1 -p tcp -s 192.168.2.0/24 -d 192.168.2.1 --dport 8080 -j NEW_PRE_CHAIN
```

上面这条规则的意思是：**在 nat 表的 PREROUTING 链上添加一条规则，将从 eth1 进来的 TCP 包，在经过在 PREROUTING 链时进行判断，如果源 IP 是 192.168.2.x 网段，目的 IP 是 192.168.2.1 ，目的端口为 8080 ，则跳转到 NEW_PRE_CHAIN 链上继续匹配规则**。

**说是子链，实际上仍然和原有规则放在一起**。例如上面在子链中新添加两条规则后，netfilter 的 NAT 表的规则就变成了：

![](http://img.blog.csdn.net/20150404224019790?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFzb25jaGVuX2diZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

可以看到，两条新的 PRE_ROUTING 规则被紧跟在原有规则后面存放，并且在内核中的 NAT 表中并没有关于子链 NEW_PRE_CHAIN 的信息，因为**“子链”的概念是用户态的 iptables 命令才使用的**，iptables 做一些处理后将规则传到内核，而内核中 netfilter 的工作就不会那么复杂。

**用户态的 iptables 命令传入的 match 和 target 在内核都要有对应的 match 和 target** 。内核中所有的 match 和 target 都注册在全局数组 xt 中，该数组每个元素是一个 `struct xt_af` 结构，存储一类地址族的 matches 和 targets ，如 NFPROTO_IPV4 。

```
struct xt_af {  
    struct mutex mutex;  
    struct list_head match;  // 该协议的 match 集合  
    struct list_head target; // 该协议的 target 集合  
};  
```

- iptables 工具是用户空间和内核的 netfilter 模块通信的手段，因此 **iptables 中也有“表”和“hook 点”的概念**，只是 hook 点被 iptables 称为**内建 chain**。

iptables 命令中的**内建链**与 Netfilter 中 hook 点的对应关系如下：

```
static const char *hooknames[] = {  
    [HOOK_PRE_ROUTING]  = "PREROUTING",  
    [HOOK_LOCAL_IN]     = "INPUT",  
    [HOOK_FORWARD]      = "FORWARD",  
    [HOOK_LOCAL_OUT]    = "OUTPUT",  
    [HOOK_POST_ROUTING] = "POSTROUTING",  
#ifdef HOOK_DROPPING  
    [HOOK_DROPPING]     = "DROPPING"  
#endif  
}; 
```

用户配置完 iptables 规则之后，传给内核的是一个 ipt_replace 结构，其中包含了内核所需要的所有内容：

```
/* The argument to IPT_SO_SET_REPLACE. */   
struct ipt_replace  
{  
    /* 表名 */  
    char name[IPT_TABLE_MAXNAMELEN];  
  
    /* hook mask */  
    unsigned int valid_hooks;  
  
    /* 新规则的 entry 数 */  
    unsigned int num_entries;   
  
    /* Total size of new entries */  
    unsigned int size;  
  
    /* Hook entry points. */  
    unsigned int hook_entry[NF_INET_NUMHOOKS];  
  
    /* Underflow points. */  
    unsigned int underflow[NF_INET_NUMHOOKS];  
  
    /* 旧规则的 entry 数 */  
    unsigned int num_counters;  
  
    /* The old entries' counters. */  
    struct xt_counters __user *counters;  
   
    /* 规则本身 */  
    struct ipt_entry entries[0];  
};  
```

该结构包含了表名，规则挂载的 hook 点，ipt entry 的数目等信息，该结构的最后为实际的规则内容，基本包含了内核中 `struct xt_table` 和 `struct xt_table_info` 结构所需要的内容。传递的过程通过 `getsockopt()` 和 `setsockopt()` 系统调用来完成，这两个系统调用的函数原型为：

```
int getsockopt(int s, int level, int optname, void *optval, socklen_t *optlen);

int setsockopt(ints, int level, int optname, const void *optval, socklen_t optlen);
```

其中 `getsockopt()` 的参数 `optname` 可取的值为 IPT_SO_GET_INFO、IPT_SO_GET_ENTRIES、IPT_SO_GET_REVISION_MATCH 和 IPT_SO_GET_REVISION_TARGET 。

`setsockopt()` 的参数 `optname` 可取的值为 IPT_SO_SET_REPLACE 和 IPT_SO_SET_ADD_COUNTERS，所有修改规则的动作（添加、修改、删除等）都通过 IPT_SO_SET_REPLACE 完成，而 IPT_SO_SET_ADD_COUNTERS 更新表中每个 ipt_entry 的 counters 成员。

当然这些都是 iptables 工具做的事情，我们只要会使用 iptables 命令即可。而**我们知道可以自定义链（子链），这也是 iptables 工具所的要处理的事情，实际上内核是不知道有自定义链的**。

在一个链中还可以设置默认规则，即如果所有规则都不匹配，就走默认规则。默认规则一般设置为 **NF_ACCEPT** 或 **NF_DROP** 等。**默认规则的 target 在 iptables 命令中被称为 `policy`** 。

例如，我们在 NAT 表中设置 PRE_ROUTING 链上的 policy 是 ACCEPT ：

```
iptables -t nat -P PREROUTING ACCEPT
```

那在这条链上的最后一条规则不是我们自己手动添加的，而是一条没有 match 和 target 的默认规则，verdict 被设置为 0xfffffffe ；

```
# iptables -t nat -L  
Chain PREROUTING (policy ACCEPT)
...
```

注意，**只能设置内建链的 policy ，不能设置自定义链（子链），因为 policy 是内核中要使用的，内核并不知道自定义链的存在**。另外，**NAT 表的各链的 policy 都不能为 DROP** ，因为 NAT 表本来就不是用来过滤的，想要 DROP 的规则可以放到 filter 表中。

### 关于 SNAT 和 masquerade

这是两个 iptables 规则的 target ，实际上他俩没有区别，只是 nat 配置规则的两种写法而已。我们先来看一下规则的写法：

```
iptables -t nat -I POSTROUTING 1 -o $wan_if -j MASQUERADE

iptables -t nat -I POSTROUTING 2 -s $lan_ip/$lan_mask -d $lan_ip/$lan_mask -j SNAT --to-source $wan_ip
```

规则中的 `-j` 选项指定 `target` ，这两个 target 分别对应内核的下面两个函数：

```
masquerade_tg(structsk_buff *skb, const struct xt_target_param *par);

ipt_snat_target(structsk_buff *skb, const struct xt_target_param *par);
```

`SNAT` 规则指定了应该将源 ip 变成什么地址，而 `MASQUERADE` 需要在出口设备的 IP 列表中选择一个合适的作为转换 IP 。二者都会调用 `nf_nat_setup_info()` 对 ct 进行转换。

注意，一般情况下路由器只允许内网到外网的 NAT ，所以必须做完 SNAT 后，外网的数据包才可以通过转换后的 conntrack 条目到达内网，所以我们通常不会设置 DNAT 规则。

同样的，`REDIRECT` 和 `DNAT` 这两个 target 的作用一样，都是修改数据包的目的地址。


----------


## [一个复杂的nf_conntrack实例全景解析](http://blog.csdn.net/dog250/article/details/78372576)

> 超级牛文推荐，原文精彩；

示例：大意就是 A 试图与 B 的 21 端口建立一个类似 FTP 控制通道连接，然后 B 反连 A 的 23 端口，创建一个类似 FTP 数据通道连接，中间经过一个 NAT Box 的节点，旨在实现地址隔离，将 1.1.1.1/2.2.2.2 元组转换为 172.16.1.1/192.168.2.2 元组，就这么简单。

指出一点，以上的拓扑中展示的连接细节跟 FTP 很像，但却不是 FTP ，为了指出其不同之处，我来说一下 B 得以反连 A 的 23 端口的细节。

- 首先，B 并不知道要反连哪个地址的哪个端口，这一切都是 A 在适当的时候告诉 B 的。
- 其次，A 告诉 B 的方式很简单，就是把信息封装在应用层数据中，比如把请反连 1.1.1.1 的 23 端口这句话作为 buffer 写入到应用层缓存里。

因此，由于 A 的地址 1.1.1.1 已经被 NAT Box 改成了 172.16.1.1 ，那么应用层 buffer 里的 IP 地址信息也应该做相应的改变，如果关联这一切，这就是本文的要旨，虽然说很简单（至少不会太难），但是对于 nf_conntrack 机制而言，这个案例却可以说它覆盖了其方方面面，可谓集大成者于一例，不可不学。

![](http://img.blog.csdn.net/20171028082232708?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 当来自 A 的 TCP 建链包首次从 NAT Box 网口 1 到达时

这是**首次到达的 SYN 包**，显然在此之前，NAT Box 上没有关于该连接的任何记录：

![](http://img.blog.csdn.net/20171028082336485?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 当来自 B 的 SYN/ACK 返回包从 NAT Box 网口 2 到达时

这个情景的重点在于 **conntrack 状态的更新**：

![](http://img.blog.csdn.net/20171028082624968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 当来自 A 的带有反连命令的数据包从 NAT Box 网口 1 到达时

这里的重点是**应用层数据的解析**，即 Helper 开始发挥作用：

![](http://img.blog.csdn.net/20171028082735835?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 当来自 B 的反连 A 的 SYN 包从 NAT Box 网口 2 到达时

这是 Helper 使能的核心：

![](http://img.blog.csdn.net/20171028082824100?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

注意事项：

他们总说 conntrack 效率低下，甚至使用了 conntrack 之后会急剧影响 Linux Box 的网络处理性能，然而内核社区却一直没有取消 conntrack 甚至都没有发出过类似的声音，这说明前面说的这帮人是多么不可理喻，他们某时仅仅会求教于别人彻底去除 conntrack 的方法，然而当我反问“你能列出可以替代 state match 的规则吗？如果可以，那就可以去掉 conntrack”，他们却根本不知道我在说什么，当然，我所提到的肯定跟 state match 相关，不过话又说回来，如果他们知道了 state match 的本质，他们肯定也就知道如何彻底去除 conntrack 的方法，所以说这不能怪他们。还是那句话，不与无知者辩论，沉默是金。

不管怎么说，虽然我并不认为 conntrack 存在问题，但是还是要说下**使用 conntrack 时要注意的几个问题**，我归纳了三个：

- **避开不必要的自旋锁**（问题在于，如果存在大量的短链接或者同时有大量的连接（比如 TCP Timewait）被销毁，这笔开销将是高昂且不可避免的。更有甚至，如果存在 DDoS 攻击，很多不请自来的数据包将会 confirm 海量的数据流，自旋锁将会锁死整个江湖）
- **注意 CPU 和内存操作**
- **注意 `net.nf_conntrack_max` 的大小**；这个就不啰嗦了，如果你的机器内存足够，不要吝啬，多多的用于网络协议栈是没有坏处的，特别是当你的机器用于转发设备而不是服务器时，这意味着你没有数据库的开销，没有应用服务器的开销，你的内存设置甚至都不会用于 TCP/UDP（我一再强调 TCP/UDP 属于端到端的技术范畴，而根本就不属于网络技术范畴），因为数据根本就到不了四层处理，那你的内存何用呢？


----------


## [nf_conntrack连接跟踪模块](http://blog.csdn.net/u010472499/article/details/78292811)

在 iptables 里，包是和被跟踪连接的四种不同状态（state）有关的。它们分别是 **NEW**，**ESTABLISHED**，**RELATED** 和 **INVALID**。使用 iptables 的 state 模块可以匹配操作这几种状态，我们能很容易地控制“谁或什么能发起新的会话”。**为什么需要这种状态跟踪机制呢？**比如你的 80 端口开启，而你的程序被植入**反弹式木马**，导致服务器主动从 80 端口向外部发起连接请求，这个时候你怎么控制呢。关掉 80 端口，那么你的网站也无法正常运行了。但有了连接跟踪你就可以设置只允许回复关于 80 端口的外部请求（ESATBLISHED 状态），而无法发起向外部的请求（NEW 状态）。所以有了连接跟踪就可以做到在这种层面上的限制，慢慢往下看就明白了各状态的意义。

**所有在内核中由 Netfilter 的特定框架做的连接跟踪称作 `conntrack` (connection tracking)** 。conntrack 可以作为模块安装，也可以作为内核的一部分。大部分情况下，我们想要、也需要更详细的连接跟踪，这是相比于缺省的 conntrack 而言。也因为此，conntrack 中有许多用来处理 TCP，UDP 或 ICMP 协议的部件。这些模块从数据包中提取详细的、唯一的信息，因此能保持对每一个数据流的跟踪。这些信息也告知 conntrack 流当前的状态。例如，UDP 流一般由他们的目的地址、源地址、目的端口和源端口唯一确定。

在以前的内核里，我们可以打开或关闭重组（defrag）功能。然而连接跟踪被引入内核后，这个选项就被取消了。**因为没有包的重组，连接跟踪就不能正常工作。现在重组已经整合入 conntrack ，并且在 conntrack 启动时自动启动。不要关闭重组功能，除非你要关闭连接跟踪**。

**除了本地产生的包由 OUTPUT 链处理外，所有连接跟踪都是在 PREROUTING 链里进行处理的**，意思就是，iptables 会在 PREROUTING 链里重新计算所有的状态。如果我们发送一个流的初始化包，状态就会在 OUTPUT 链里被设置为 NEW ，当我们收到回应的包时，状态就会在 PREROUTING 链里被设置为 ESTABLISHED 。如果第一个包不是本地产生的，那就会在 PREROUTING 链里被设置为 NEW 状态。综上，**所有状态的改变和计算都是在 `nat` 表中的 PREROUTING 链和 OUTPUT 链里完成的**。

`nf_conntrack` 模块根据 IP 地址可以实时追踪本机 TCP/UDP/ICMP 的连接详细信息并保存在内存中 `/proc/net/nf_conntrack` 文件里；

```
ipv4     2 tcp      6 89  SYN_SENT src=101.81.225.225 dst=112.74.99.130 sport=33952 dport=80 
src=112.74.99.130 dst=101.81.225.225 sport=80 dport=33952 [UNREPLIED] mark=0 secmark=0 use=2
ipv4     2 tcp      6 89  SYN_SENT src=101.81.225.225 dst=112.74.99.130 sport=33952 dport=80 
src=112.74.99.130 dst=101.81.225.225 sport=80 dport=33952 [UNREPLIED] mark=0 secmark=0 use=2
```

`[UNREPLIED]` 说明这个连接还没有收到任何回应。当一个连接在两个方向上都有传输时，conntrack 记录就删除 `[UNREPLIED]` 标志，然后重置。在末尾有 `[ASSURED]` 的记录说明两个方向已没有流量。

```
ipv4     2 tcp      6 431962 ESTABLISHED src=101.81.225.225 dst=112.74.99.130 sport=33952 dport=80 
src=112.74.99.130 dst=101.81.225.225 sport=80 dport=33952 [ASSURED] mark=0 secmark=0 use=2
ipv4     2 tcp      6 431962 ESTABLISHED src=101.81.225.225 dst=112.74.99.130 sport=33952 dport=80 
src=112.74.99.130 dst=101.81.225.225 sport=80 dport=33952 [ASSURED] mark=0 secmark=0 use=2
```

这样的记录是确定的，在连接跟踪表满时，是不会被删除的，没有 `[ASSURED]` 的记录就要被删除。


----------


## [details of /proc/net/nf_conntrack](https://stackoverflow.com/questions/16034698/details-of-proc-net-ip-conntrack-nf-conntrack)

连接跟踪表示例

```
root@ip-172-1-57-200:~# cat /proc/net/nf_conntrack|grep "31425"
ipv4     2 tcp      6 81 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=30881 dport=31425 src=100.97.116.23 dst=172.1.57.200 sport=50066 dport=30881 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 61 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=62795 dport=31425 src=100.97.116.23 dst=172.1.57.200 sport=50066 dport=62795 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 61 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=30681 dport=31425 src=100.97.74.176 dst=172.1.57.200 sport=50066 dport=30681 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 41 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=30485 dport=31425 src=100.97.89.202 dst=172.1.57.200 sport=50066 dport=30485 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 21 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=30285 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=30285 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 101 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=31077 dport=31425 src=100.97.91.90 dst=172.1.57.200 sport=50066 dport=31077 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 101 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=63187 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=63187 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 41 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=62591 dport=31425 src=100.97.123.135 dst=172.1.57.200 sport=50066 dport=62591 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 0 TIME_WAIT src=172.1.41.207 dst=172.1.57.200 sport=30085 dport=31425 src=100.97.108.84 dst=172.1.57.200 sport=50066 dport=30085 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 81 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=62993 dport=31425 src=100.97.91.90 dst=172.1.57.200 sport=50066 dport=62993 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 1 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=62199 dport=31425 src=100.97.89.202 dst=172.1.57.200 sport=50066 dport=62199 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 92 TIME_WAIT src=172.1.58.80 dst=100.97.16.153 sport=31425 dport=8090 src=100.97.16.153 dst=172.1.58.80 sport=8090 dport=31425 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 21 TIME_WAIT src=172.1.41.234 dst=172.1.57.200 sport=62397 dport=31425 src=100.97.116.23 dst=172.1.57.200 sport=50066 dport=62397 [ASSURED] mark=0 zone=0 use=2
root@ip-172-1-57-200:~#
```

The format of a line from `/proc/net/ip_conntrack` is the same as for `/proc/net/nf_conntrack`, except the first two columns are missing.

- **First column**: The network layer protocol name (eg. ipv4).
- **Second column**: The network layer protocol number.
- **Third column**: The transmission layer protocol name (eg. tcp).
- **Fourth column**: The transmission layer protocol number.
- **Fifth column**: The seconds until the entry is invalidated.
- **Sixth column** (Not all protocols): The connection state.

All other columns are named (`key=value`) or represent flags (`[UNREPLIED]`, `[ASSURED]`, ...). A line can contain up to two columns having the same name (eg. `src` and `dst`). Then, the first occurrence relates to the **request direction** and the second occurrence relates to the **response direction**.

Meaning of the flags:

- `[ASSURED]`: **Traffic has been seen in both (ie. request and response) direction**.
- `[UNREPLIED]`: **Traffic has not been seen in response direction yet**. In case the connection tracking cache overflows, these connections are dropped first.

Please note that some column names appear only for specific protocols (eg. `sport` and `dport` for TCP and UDP, `type` and `code` for ICMP). Other column names (eg. `mark`) appear only if the kernel was built with specific options.


Examples:

- `ipv4 2 tcp 6 300 ESTABLISHED src=1.1.1.2 dst=2.2.2.2 sport=2000 dport=80 src=2.2.2.2 dst=1.1.1.1 sport=80 dport=12000 [ASSURED] mark=0 use=2` belongs to an established TCP connection from host 1.1.1.2, port 2000, to host 2.2.2.2, port 80, from which responses are sent to host 1.1.1.1, port 12000, timing out in five minutes. For this connection, packets have been seen in both directions.
- `ipv4 2 icmp 1 3 src=1.1.1.2 dst=1.1.1.1 type=8 code=0 id=32354 src=1.1.1.1 dst=1.1.1.2 type=0 code=0 id=32354 mark=0 use=2` belongs to an ICMP echo request packet from host 1.1.1.2 to host 1.1.1.1 with an expected echo reply packet from host 1.1.1.1 to host 1.1.1.2, timing out in three seconds.

The response destination host is not necessarily the same as the request source host, as the request source address may have been **masqueraded** by the response destination host.


**Please note that the following information might not be up-to-date!**

Fields available for all entries:

- **bytes** (if accounting is enabled, request and response)
- **delta-time** (if `CONFIG_NF_CONNTRACK_TIMESTAMP` is enabled)
- **dst** (request and response)
- **mark** (if `CONFIG_NF_CONNTRACK_MARK` is enabled)
- **packets** (if accounting is enabled, request and response)
- **secctx** (if `CONFIG_NF_CONNTRACK_SECMARK` is enabled)
- **src** (request and response)
- **use**
- **zone** (if `CONFIG_NF_CONNTRACK_ZONES` is enabled)

Fields available for `dccp`, `sctp`, `tcp`, `udp` and `udplite` transmission layer protocols:

- **dport** (request and response)
- **sport** (request and response)

Fields available for `icmp` transmission layer protocol:

- **code** (request and response)
- **id** (request and response)
- **type** (request and response)

Fields available for `gre` transmission layer protocol:

- **dstkey** (request and response)
- **srckey** (request and response)
- **stream_timeout**
- **timeout**

Allowed values for the sixth field:

- `dccp` transmission layer protocol
    - CLOSEREQ
    - CLOSING
    - IGNORE
    - INVALID
    - NONE
    - OPEN
    - PARTOPEN
    - REQUEST
    - RESPOND
    - TIME_WAIT
- `sctp` transmission layer protocol
    - CLOSED
    - COOKIE_ECHOED
    - COOKIE_WAIT
    - ESTABLISHED
    - NONE
    - SHUTDOWN_ACK_SENT
    - SHUTDOWN_RECD
    - SHUTDOWN_SENT
- `tcp` transmission layer protocol
    - CLOSE
    - CLOSE_WAIT
    - ESTABLISHED
    - FIN_WAIT
    - LAST_ACK
    - NONE
    - SYN_RECV
    - SYN_SENT
    - SYN_SENT2
    - TIME_WAIT



------


- 连接跟踪表中的 CLOSE_WAIT

```
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack| grep "CLOSE_WAIT"
tcp      6 2938 CLOSE_WAIT src=100.97.16.151 dst=172.31.139.136 sport=55832 dport=6693 src=172.31.139.136 dst=172.1.57.200 sport=6693 dport=55832 [ASSURED] mark=0 use=2
tcp      6 3457 CLOSE_WAIT src=100.97.16.218 dst=172.31.10.189 sport=56876 dport=9092 src=172.31.10.189 dst=172.1.57.200 sport=9092 dport=56876 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack| grep "CLOSE_WAIT"
tcp      6 2929 CLOSE_WAIT src=100.97.16.151 dst=172.31.139.136 sport=55832 dport=6693 src=172.31.139.136 dst=172.1.57.200 sport=6693 dport=55832 [ASSURED] mark=0 use=2
tcp      6 3447 CLOSE_WAIT src=100.97.16.218 dst=172.31.10.189 sport=56876 dport=9092 src=172.31.10.189 dst=172.1.57.200 sport=9092 dport=56876 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack| grep "CLOSE_WAIT"
tcp      6 2926 CLOSE_WAIT src=100.97.16.151 dst=172.31.139.136 sport=55832 dport=6693 src=172.31.139.136 dst=172.1.57.200 sport=6693 dport=55832 [ASSURED] mark=0 use=2
tcp      6 3445 CLOSE_WAIT src=100.97.16.218 dst=172.31.10.189 sport=56876 dport=9092 src=172.31.10.189 dst=172.1.57.200 sport=9092 dport=56876 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack| grep "CLOSE_WAIT"
tcp      6 2925 CLOSE_WAIT src=100.97.16.151 dst=172.31.139.136 sport=55832 dport=6693 src=172.31.139.136 dst=172.1.57.200 sport=6693 dport=55832 [ASSURED] mark=0 use=2
tcp      6 3444 CLOSE_WAIT src=100.97.16.218 dst=172.31.10.189 sport=56876 dport=9092 src=172.31.10.189 dst=172.1.57.200 sport=9092 dport=56876 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack| grep "CLOSE_WAIT"
tcp      6 2924 CLOSE_WAIT src=100.97.16.151 dst=172.31.139.136 sport=55832 dport=6693 src=172.31.139.136 dst=172.1.57.200 sport=6693 dport=55832 [ASSURED] mark=0 use=2
tcp      6 3443 CLOSE_WAIT src=100.97.16.218 dst=172.31.10.189 sport=56876 dport=9092 src=172.31.10.189 dst=172.1.57.200 sport=9092 dport=56876 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~#
...（N 小时后）...
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack | grep "CLOSE_WAIT"
tcp      6 3025 CLOSE_WAIT src=100.97.16.151 dst=172.31.11.23 sport=58110 dport=6693 src=172.31.11.23 dst=172.1.57.200 sport=6693 dport=58110 [ASSURED] mark=0 use=2
tcp      6 3318 CLOSE_WAIT src=100.97.16.151 dst=172.31.10.173 sport=44986 dport=6693 src=172.31.10.173 dst=172.1.57.200 sport=6693 dport=44986 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack | grep "CLOSE_WAIT"
tcp      6 3023 CLOSE_WAIT src=100.97.16.151 dst=172.31.11.23 sport=58110 dport=6693 src=172.31.11.23 dst=172.1.57.200 sport=6693 dport=58110 [ASSURED] mark=0 use=2
tcp      6 3316 CLOSE_WAIT src=100.97.16.151 dst=172.31.10.173 sport=44986 dport=6693 src=172.31.10.173 dst=172.1.57.200 sport=6693 dport=44986 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack | grep "CLOSE_WAIT"
tcp      6 3022 CLOSE_WAIT src=100.97.16.151 dst=172.31.11.23 sport=58110 dport=6693 src=172.31.11.23 dst=172.1.57.200 sport=6693 dport=58110 [ASSURED] mark=0 use=2
tcp      6 3315 CLOSE_WAIT src=100.97.16.151 dst=172.31.10.173 sport=44986 dport=6693 src=172.31.10.173 dst=172.1.57.200 sport=6693 dport=44986 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~# cat /proc/net/ip_conntrack | grep "CLOSE_WAIT"
tcp      6 3021 CLOSE_WAIT src=100.97.16.151 dst=172.31.11.23 sport=58110 dport=6693 src=172.31.11.23 dst=172.1.57.200 sport=6693 dport=58110 [ASSURED] mark=0 use=2
tcp      6 3314 CLOSE_WAIT src=100.97.16.151 dst=172.31.10.173 sport=44986 dport=6693 src=172.31.10.173 dst=172.1.57.200 sport=6693 dport=44986 [ASSURED] mark=0 use=2
root@ip-172-1-57-200:~#
```

- 按上述输出信息中的端口进行查看，发现不存在对应的 CLOSE_WAIT 信息

```
root@ip-172-1-57-200:~# netstat -natp|grep 55832
root@ip-172-1-57-200:~# netstat -natp|grep 56876
root@ip-172-1-57-200:~# netstat -natp|grep 9092
root@ip-172-1-57-200:~# netstat -natp|grep 6693
```

- 同一时间系统中可以看到的信息

```
root@ip-172-1-57-200:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSE_WAIT 3082
TIME_WAIT 8
ESTABLISHED 19
root@ip-172-1-57-200:~#
root@ip-172-1-57-200:~# ss -na | awk '/^tcp/ {++S[$2]} END {for(a in S) print a, S[a]}'
CLOSE-WAIT 3082
ESTAB 19
TIME-WAIT 8
LISTEN 71
root@ip-172-1-57-200:~#
```

小结：连接跟踪表中显示的 CLOSE_WAIT 和 `ss` 或 `netstat` 命令输出的 CLOSE_WAIT 是不同的；


----------


## [如何理解Netfilter中的连接跟踪机制](http://blog.csdn.net/dandelionj/article/details/8535990)

> 牛逼好文！！建议看原文（但感觉部分内容图文不一致，可能有误）

连接跟踪定义很简单：用来记录和跟踪连接的状态。

**为什么又需要连接跟踪功能呢？**因为它是**状态防火墙**和 **NAT** 的实现基础。

**Netfilter 为了实现基于数据连接状态侦测的状态防火墙功能和 NAT 地址转换功能，才开发出了连接跟踪这套机制**。那就意思是说：如果编译内核时开启了连接跟踪选项，那么 Linux 系统就会为它收到的每个数据包维持一个连接状态用于记录这条数据连接的状态。

![](http://blog.chinaunix.net/attachment/201204/6/23069658_1333720270jYYn.jpg)

从上图中可以很明确的看到：用于实现**连接跟踪入口**的 hook 函数，以**较高的优先级**分别被注册到了 netfilter 的 `NF_IP_PRE_ROUTING` 和 `NF_IP_LOCAL_OUT` 两个 hook 点上；用于实现**连接跟踪出口**的 hook 函数，以**非常低的优先级**分别被注册到了 netfilter 的 `NF_IP_LOCAL_IN` 和 `NF_IP_POST_ROUTING` 两个 hook 点上。

> FIXME：这里关于优先级的说明没看懂；

其实 PRE_ROUTING 和 LOCAL_OUT 点可以看作是整个 netfilter 的入口，而 POST_ROUTING 和 LOCAL_IN 可以看作是其出口。在只考虑连接跟踪的情况下，一个数据包无外乎有以下三种流程可以走：

- **发送给本机的数据包**：流程为 `PRE_ROUTING -> LOCAL_IN -> 本地进程`，如果是新的包，在 PREROUTING 处生成连接记录，在 LOCAL_IN 处把生成的连接记录加到 hash ；

![](http://blog.chinaunix.net/attachment/201204/10/23069658_1334073268Umuw.jpg)

- **需要本机转发的数据包**：流程为 `PRE_ROUTING -> FORWARD -> POST_ROUTING -> 外出`，在 PREROUTING 处生成连接记录，通过 POSTROUTING 后加到 hash 表；

![](http://blog.chinaunix.net/attachment/201204/10/23069658_1334073271ttbV.jpg)

- **从本机发出的数据包**：流程为 `LOCAL_OUT -> POST_ROUTING -> 外出`，在 LOCAL_OUT 处生成连接记录，在 POSTROUTING 处把生成的连接记录加到 hash 表；

![](http://blog.chinaunix.net/attachment/201204/10/23069658_1334073276tkGn.jpg)

在开始分析连接跟踪之前，我们还是站在统帅的角度来俯视一下整个连接跟踪的布局。连接跟踪分**入口**和**出口**两个点。谨记：**入口时创建连接跟踪记录，出口时将该记录加入到连接跟踪表中**。我们分别来看看。

入口：

![](http://blog.chinaunix.net/attachment/201204/10/23069658_1334073557Epl0.jpg)

整个入口的流程简述如下：对于每个到来的 skb ，连接跟踪都将其转换成一个 tuple 结构，然后用该 tuple 去查连接跟踪表。如果该类型的数据包没有被跟踪过，将为其在连接跟踪的 hash 表里建立一个连接记录项，对于已经跟踪过了的数据包则不用此操作。紧接着，调用该报文所属协议的连接跟踪模块的所提供的 `packet()` 回调函数，最后根据状态改变连接跟踪记录的状态。

出口：

![](http://blog.chinaunix.net/attachment/201204/10/23069658_13340735623l2b.jpg)

整个出口的流程简述如下：对于每个即将离开 Netfilter 框架的数据包，如果用于处理该协议类型报文的连接跟踪模块提供了 helper 函数，那么该数据包首先会被 helper 函数处理，然后才去判断，如果该报文已经被跟踪过了，那么其所属连接的状态，决定该包是该被丢弃、或是返回协议栈继续传输，又或者将其加入到连接跟踪表中。

**连接跟踪的协议管理**：我们前面曾说过，不同协议其连接跟踪的实现是不相同的。每种协议如果要开发自己的连接跟踪模块，那么它首先必须实例化一个 `ip_conntrack_protocol{}` 结构体类型的变量，对其进行必要的填充，然后调用 `ip_conntrack_protocol_register()` 函数将该结构进行注册，其实就是根据协议类型将其设置到全局数组 `ip_ct_protos[]` 中的相应位置上。

**连接跟踪的辅助模块**：Netfilter 的连接跟踪为我们提供了一个非常有用的功能模块：`helper`。该模块可以使我们以很小的代价来**完成对连接跟踪功能的扩展**。这种应用场景需求一般是，**当一个数据包即将离开 Netfilter 框架之前，我们可以对数据包再做一些最后的处理**。从前面的图我们也可以看出来，helper 模块以**较低优先级**被注册到了 Netfilter 的 LOCAL_OUT 和 POST_ROUTING 两个 hook 点上。每一个辅助模块都是一个 `ip_conntrack_helper{}` 结构体类型的对象。注册在 Netfilter 框架里 LOCAL_OUT 和 POST_ROUTING 两个 hook 点上 `ip_conntrack_help()` 回调函数所做的事情基本也就很清晰了：那就是通过依次遍历 helpers 链表，然后调用每个 `ip_conntrack_helper{}` 对象的 `help()` 函数。

**期望连接**：Netfilter 的连接跟踪为支持诸如 FTP 这样的“活动”连接提供了一个叫做“期望连接”的机制。连接跟踪在处理这种应用场景时提出了一个“期望连接”的概念，即一条数据连接和另外一条数据连接是相关的，然后对于这种有“相关性”的连接给出自己的解决方案。每条期望连接都用一个 `ip_conntrack_expect{}` 结构体类型的对象来表示，所有的期望连接存储在由全局变量 `ip_conntrack_expect_list` 所指向的双向链表中；

**连接跟踪表**：连接跟踪表是一个用于记录所有数据包连接信息的 hash 散列表，其实连接跟踪表就是一个以数据包的 hash 值组成的一个双向循环链表数组，每条链表中的每个节点都是 `ip_conntrack_tuple_hash{}` 类型的一个对象。


----------


## [netfilter之conntrack笔记](http://blog.csdn.net/appletreesujie/article/details/6871011)

### 控制结构 sk_buff 和网络报文的存储空间

![](http://img.my.csdn.net/uploads/201110/13/0_1318501902FnBN.gif)

### 分片的网络报文与 scatter/gather IO

![](http://img.my.csdn.net/uploads/201110/13/0_1318501917og2S.gif)

网络报文在内存中不一定是连续存储的，同一个网络报文有可能被分成几片存放在内存的不同位置（不要和 IP fragmentation 混淆，IP 分片是将一个网络报文分成多个网络报文，这里是将一个网络报文分成几片存放在不同的内存空间）。

> `IP fragmentation` is an Internet Protocol (IP) process that breaks datagrams into smaller pieces (fragments), so that packets may be formed that can pass through a link with a smaller maximum transmission unit (MTU) than the original datagram size. The fragments are reassembled by the receiving host.

为了记录网络报文的长度，在 `sk_buff` 里增加了一个变量 `data_len` 。这个变量记录的是在 `frags` 和 `frag_list` 里面存储的报文的长度。原有的变量 len 记录网络报文的总长度。`truesize` 是 `head` 所指的存储区的大小。

skb 比较复杂的部分在于 `skb_shared_info` 部分，`alloc_skb()` 在为数据分配空间的时候，会在这个数据的末尾加上一个 `skb_shared_info` 结构，这个结构就是用于 scatter/gather IO 的实现的。它主要用于提高性能，避免数据的多次拷贝。例如，当用户用 `sendmsg()` 分送一个数组结构的数据时，这些数据在物理可能是不连续的（大多数情况），在不支持 scatter/gather IO 的网卡上，它只能通过重新拷贝，将它重装成连续的 skb（skb_linearize）,才可以进行 DMA 操作。而在支持 S/G IO 上，它就省去了这次拷贝。

### Netfilter 架构图与 connection tracker

![Netfilter Components](https://upload.wikimedia.org/wikipedia/commons/d/dd/Netfilter-components.svg)

Connection tracking 分为两部分：layer-3 和 layer-4 模块。也有一个 layer-5 模块的跟踪，这个涉及到“连接辅助模块 connection helper”，因为他们不仅影响到源方向的链接，也影响到未来的连接。

----------


## [Connection tracking - CLOSE_WAIT and FIN_WAIT](http://lists.netfilter.org/pipermail/netfilter/2003-April/043510.html)


I just carried out a couple of tests to see how conntrack changes the state
of a TCP stream -

```
----------- test 1 -------------------------------------------------
1. (A) ---->SYN----->(B)  FW state = SYN_SEND
2. (A) <---SYN+ACK-- (B)  FW state = SYN_RECV
3. (A) -----ACK----->(B)  FW state = ESTABLISHED (ASSURED)

4. (A) --->FIN+ACK-->(B)  FW state = FIN_WAIT
5. (B) <-----ACK---- (B)  FW state = TIME_WAIT (times out after 2 mins)

----------- test 2 -------------------------------------------------
1. (A) ---->SYN----->(B)  FW state = SYN_SEND
2. (A) <---SYN+ACK-- (B)  FW state = SYN_RECV
3. (A) -----ACK----->(B)  FW state = ESTABLISHED (ASSURED)

4. (A) <---FIN+ACK-- (B)  FW state = CLOSE_WAIT
5. (B) -----ACK----> (B)  FW state = TIME_WAIT (times out after 2 mins)
```


Now step (4) of test 1 leads to FIN_WAIT state, but step (4) of test 2 leads
to CLOSE_WAIT.


----------


## [connection tracking query](http://lists.netfilter.org/pipermail/netfilter/2003-April/043383.html)

> I was reading the Iptables tutorial by Oskar Andreasson
> http://iptables-tutorial.frozentux.net/iptables-tutorial.html.
> It says that connection tracking in done in the PREROUTING chain or
> OUTPUT chain (for locally generated packets). If connection tracking
> is done only at these two chains, what happens to the packets that
> don't belong to an already established connection? I understand that
> it will have to go through the filter rules - before the state table
> is updated for a NEW/RELATED connection. If that is the case,
> "conntrack" must be taking place at other chains too (where the filter
> is applied). The following document
> http://www.knowplace.org/netfilter/syntax.html does infact say that
> "conntrack" is happening not only in the PREROUTING and the OUTPUT
> chain, but also in INPUT and POSTROUTING chain. What I find strange
> with this is that for a packet that goes through the "FORWARD" chain,
> "conntrack" is done twice on the same packet - first in the
> "PREROUTING" chain and second in the "POSTROUTING" chain. Does anyone
> have any explanation for this?

You're mixing two things :

- **conntrack engine**
- **state matching**

`Conntrack engine` is a module which is designed to recognize if a given packet belongs or not to an existing connection, and update connection table according to its decisions.

`State matching` is the ability for a rule to match a packet state, understand the way it has been recognized by conntrack engine.

Conntrack engine has to examine packets very early. That's why it occurs in **PREROUTING** for recieved packets and **OUTPUT** for locally generated packets. In fact, **you can see `conntrack` as a particular table which is hooked on `NF_IP_PRE_ROUTING` and `NF_IP_LOCAL_OUT` with maximum precedence** (it is to be noted that `conntrack` also occurs on **POSTROUTING** and **INPUT** for new connections validation, avoiding creation of connection that would be droped by filter table).

So, once our packet is recognised, it is "flaged" with a state that can be matched in rules with state match. and for `conntrack` has precedence over any table, you can match state wherever you want to.

> If a packet is found to belong to an already ESTABLISHED
> connection, does it still have to go through the filter rules again?

Yes it does. State is just a flag. It does not imply anything on packet
filtering, unless you explicitly specify decisions into your ruleset.


----------



## [无状态TCP的ip_conntrack](http://blog.csdn.net/dog250/article/details/9318843)

Linux 的 ip_conntrack 实现得过于沉重和精细。而实际上有时候，根本不需要在 conntrack 中对 TCP 的状态进行跟踪，只把它当 UDP 好了；

Linux 提供了两个 `sysctl` 参数可以在一定程度上放弃对 TCP 状态的精细化监控，它们是：

- `/proc/sys/net/netfilter/nf_conntrack_tcp_loose`
- `/proc/sys/net/netfilter/nf_conntrack_tcp_be_liberal`

在 ip_conntrack 的逻辑中，如果一个包没有在 conntrack 哈希中找到对应的流，就会被认为是一个流的头包，流程如下：

![](http://img.blog.csdn.net/20130713160211796?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG9nMjUw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

TCP 对数据包的要求太高了，在上述的 loose 参数不为 1 的情况下，tcp_new 要求新到来的第一个包必然要有 SYN 标志，否则不予创建 conntrack，数据包将成为游离数据包，NAT 等都将失去作用，即便是中间的数据包，be_liberal 不为 1 时，tcp_packet 要求数据包必须 in_window 。这就是为何 TCP 的 establish conntrack 超时时间设置 5 天那么久的原因，还好，有上述两个参数，我们可以避开这个，将 TCP 尽可能地和 UDP 同等对待。我的方式更猛，索性将 TCP 的 nf_conntrack_l4proto 换成了 nf_conntrack_l4proto_udp4 ，这样连 sysctl 参数都不用设置了。

NAT 问题的解释：一条 TCP 连接，经过 Linux Box，被 NAT，过了 120 秒之后，连接断开！这个解释很简单，我将 `net.netfilter.nf_conntrack_tcp_timeout_established` 设置成了 120 秒，此后 conntrack 超时被删除！再有数据时，按照上面的 tcp_new 流程，数据包将成为游离数据包，NAT 依赖 conntrack ，故失效，连接依赖 NAT ，故而连接失效！


----------


## [networking/nf_conntrack](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt)

```
/proc/sys/net/netfilter/nf_conntrack_* Variables:

nf_conntrack_acct - BOOLEAN
	0 - disabled (default)
	not 0 - enabled

	Enable connection tracking flow accounting. 64-bit byte and packet
	counters per flow are added.

nf_conntrack_buckets - INTEGER
	Size of hash table. If not specified as parameter during module
	loading, the default size is calculated by dividing total memory
	by 16384 to determine the number of buckets but the hash table will
	never have fewer than 32 and limited to 16384 buckets. For systems
	with more than 4GB of memory it will be 65536 buckets.
	This sysctl is only writeable in the initial net namespace.

nf_conntrack_checksum - BOOLEAN
	0 - disabled
	not 0 - enabled (default)

	Verify checksum of incoming packets. Packets with bad checksums are
	in INVALID state. If this is enabled, such packets will not be
	considered for connection tracking.

nf_conntrack_count - INTEGER (read-only)
	Number of currently allocated flow entries.

nf_conntrack_events - BOOLEAN
	0 - disabled
	not 0 - enabled (default)

	If this option is enabled, the connection tracking code will
	provide userspace with connection tracking events via ctnetlink.

nf_conntrack_expect_max - INTEGER
	Maximum size of expectation table.  Default value is
	nf_conntrack_buckets / 256. Minimum is 1.

nf_conntrack_frag6_high_thresh - INTEGER
	default 262144

	Maximum memory used to reassemble IPv6 fragments.  When
	nf_conntrack_frag6_high_thresh bytes of memory is allocated for this
	purpose, the fragment handler will toss packets until
	nf_conntrack_frag6_low_thresh is reached.

nf_conntrack_frag6_low_thresh - INTEGER
	default 196608

	See nf_conntrack_frag6_low_thresh

nf_conntrack_frag6_timeout - INTEGER (seconds)
	default 60

	Time to keep an IPv6 fragment in memory.

nf_conntrack_generic_timeout - INTEGER (seconds)
	default 600

	Default for generic timeout.  This refers to layer 4 unknown/unsupported
	protocols.

nf_conntrack_helper - BOOLEAN
	0 - disabled (default)
	not 0 - enabled

	Enable automatic conntrack helper assignment.
	If disabled it is required to set up iptables rules to assign
	helpers to connections.  See the CT target description in the
	iptables-extensions(8) man page for further information.

nf_conntrack_icmp_timeout - INTEGER (seconds)
	default 30

	Default for ICMP timeout.

nf_conntrack_icmpv6_timeout - INTEGER (seconds)
	default 30

	Default for ICMP6 timeout.

nf_conntrack_log_invalid - INTEGER
	0   - disable (default)
	1   - log ICMP packets
	6   - log TCP packets
	17  - log UDP packets
	33  - log DCCP packets
	41  - log ICMPv6 packets
	136 - log UDPLITE packets
	255 - log packets of any protocol

	Log invalid packets of a type specified by value.

nf_conntrack_max - INTEGER
	Size of connection tracking table.  Default value is
	nf_conntrack_buckets value * 4.

nf_conntrack_tcp_be_liberal - BOOLEAN
	0 - disabled (default)
	not 0 - enabled

	Be conservative in what you do, be liberal in what you accept from others.
	If it's non-zero, we mark only out of window RST segments as INVALID.

nf_conntrack_tcp_loose - BOOLEAN
	0 - disabled
	not 0 - enabled (default)

	If it is set to zero, we disable picking up already established
	connections.

nf_conntrack_tcp_max_retrans - INTEGER
	default 3

	Maximum number of packets that can be retransmitted without
	received an (acceptable) ACK from the destination. If this number
	is reached, a shorter timer will be started.

nf_conntrack_tcp_timeout_close - INTEGER (seconds)
	default 10

nf_conntrack_tcp_timeout_close_wait - INTEGER (seconds)
	default 60

nf_conntrack_tcp_timeout_established - INTEGER (seconds)
	default 432000 (5 days)

nf_conntrack_tcp_timeout_fin_wait - INTEGER (seconds)
	default 120

nf_conntrack_tcp_timeout_last_ack - INTEGER (seconds)
	default 30

nf_conntrack_tcp_timeout_max_retrans - INTEGER (seconds)
	default 300

nf_conntrack_tcp_timeout_syn_recv - INTEGER (seconds)
	default 60

nf_conntrack_tcp_timeout_syn_sent - INTEGER (seconds)
	default 120

nf_conntrack_tcp_timeout_time_wait - INTEGER (seconds)
	default 120

nf_conntrack_tcp_timeout_unacknowledged - INTEGER (seconds)
	default 300

nf_conntrack_timestamp - BOOLEAN
	0 - disabled (default)
	not 0 - enabled

	Enable connection tracking flow timestamping.

nf_conntrack_udp_timeout - INTEGER (seconds)
	default 30

nf_conntrack_udp_timeout_stream2 - INTEGER (seconds)
	default 180

	This extended timeout will be used in case there is an UDP stream
	detected.
```

