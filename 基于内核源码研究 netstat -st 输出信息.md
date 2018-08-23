# 基于内核源码研究 netstat -st 输出信息

> 以下内容基于 linux-3.13.11 源码

## TCPBacklogDrop

含义：**由于内存不足导致 skb 被 drop** ；

TCPBacklogDrop 对应 LINUX_MIB_TCPBACKLOGDROP

在 `tcp_v4_rcv()` 中有

```
2015     } else if (unlikely(sk_add_backlog(sk, skb,
2016                        sk->sk_rcvbuf + sk->sk_sndbuf))) {
2017         bh_unlock_sock(sk);
2018         NET_INC_STATS_BH(net, LINUX_MIB_TCPBACKLOGDROP);
```

在 `include/net/sock.h` 中有

```
 788 /*
 789  * Take into account size of receive queue and backlog queue
 790  * Do not take into account this skb truesize,
 791  * to allow even a single big packet to come.
 792  */
 793 static inline bool sk_rcvqueues_full(const struct sock *sk, const struct sk_buff *skb,
 794                      unsigned int limit)
 795 {
 796     unsigned int qsize = sk->sk_backlog.len + atomic_read(&sk->sk_rmem_alloc);
 797
 798     return qsize > limit;
 799 }
 800
 801 /* The per-socket spinlock must be held here. */
 802 static inline __must_check int sk_add_backlog(struct sock *sk, struct sk_buff *skb,
 803                           unsigned int limit)
 804 {
 805     if (sk_rcvqueues_full(sk, skb, limit))
 806         return -ENOBUFS;
 807
 808     __sk_add_backlog(sk, skb);
 809     sk->sk_backlog.len += skb->truesize;
 810     return 0;
 811 }
```

在 `arch/alpha/include/uapi/asm/errno.h` 中有

```
 31 #define ENOBUFS     55  /* No buffer space available */
```

其中

- `sk->sk_rcvbuf` 对应 size of receive buffer in bytes
- `sk->sk_sndbuf` 对应 size of send buffer in bytes

综上，该参数的含义（大致意思）是由于内存不足导致 skb 被 drop ；



## ListenOverflows

含义：**当收到 TCP 三次握手的第一个 SYN 或最后一次握手的 ACK 时，若发现 accept queue 已满，则说明发生了 overflow 的问题**；

ListenOverflows 对应 LINUX_MIB_LISTENOVERFLOWS

在 `tcp_v4_conn_request()` 中有

```
1476     /* Accept backlog is full. If we have already queued enough
1477      * of warm entries in syn queue, drop request. It is better than
1478      * clogging syn queue with openreqs with exponentially increasing
1479      * timeout.
1480      */
1481     if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
1482         NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
1483         goto drop;
1484     }
```

- `sk_acceptq_is_full(sk)` 判定 accept queue 是否已满；
- `inet_csk_reqsk_queue_young(sk)` 获取 SYN 队列中还没有握手完成的请求数，也就是 young request sock 的数量；

> `tcp_v4_conn_request()` 实现了针对 listen 状态下收到三次握手的第一个 SYN 的处理；

在 `tcp_v4_syn_recv_sock()` 中有

```
1635     if (sk_acceptq_is_full(sk))
1636         goto exit_overflow;
...
1703 exit_overflow:
1704     NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
```

> `tcp_v4_syn_recv_sock()` 实现了处于 SYN_RECV 状态下收到一个（三次握手最后的）合法 ACK 后，新建一个 socket 的处理；

综上，该参数的含义是当收到 TCP 三次握手的第一个 SYN 或最后一次握手的 ACK 时，若发现 accept queue 已满，则说明发生了 overflow 的问题；

## ListenDrops

含义：可以看到，很多分支都会触发 drop 操作；

ListenDrops 对应 LINUX_MIB_LISTENDROPS

在 `tcp_v4_conn_request()` 中有

```
1461     /* Never answer to SYNs send to broadcast or multicast */
1462     if (skb_rtable(skb)->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST))
1463         goto drop;
1464
1465     /* TW buckets are converted to open requests without
1466      * limitations, they conserve resources and peer is
1467      * evidently real one.
1468      */
1469     if ((sysctl_tcp_syncookies == 2 ||
1470          inet_csk_reqsk_queue_is_full(sk)) && !isn) {
1471         want_cookie = tcp_syn_flood_action(sk, skb, "TCP");
1472         if (!want_cookie)
1473             goto drop;
1474     }
1475
1476     /* Accept backlog is full. If we have already queued enough
1477      * of warm entries in syn queue, drop request. It is better than
1478      * clogging syn queue with openreqs with exponentially increasing
1479      * timeout.
1480      */
1481     if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
1482         NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
1483         goto drop;
1484     }
1485
1486     req = inet_reqsk_alloc(&tcp_request_sock_ops);
1487     if (!req)
1488         goto drop;
...
1611 drop:
1612     NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENDROPS);
```

在 `tcp_v4_syn_recv_sock()` 中有

```
1697     if (__inet_inherit_port(sk, newsk) < 0)
1698         goto put_and_exit;
...
1707 exit:
1708     NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENDROPS);
1709     return NULL;
1710 put_and_exit:
1711     inet_csk_prepare_forced_close(newsk);
1712     tcp_done(newsk);
1713     goto exit;
```


----------


相关代码：

- include/uapi/linux/snmp.h
- ./net/ipv4/tcp_ipv4.c
- include/net/sock.h








