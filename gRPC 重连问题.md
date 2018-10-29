# gRPC 重连问题

## 目录

- [GRPC client reconnect inside kubenetes](#grpc-client-reconnect-inside-kubenetes)
- [grpc-go: get ClientConn to reconnect forever](#grpc-go-get-clientconn-to-reconnect-forever)
- [stream auto reconnect](#stream-auto-reconnect)
- [Best practices for reusing connections, concurrency](#best-practices-for-reusing-connections-concurrency)
- [GRPC Connection Backoff Protocol](#grpc-connection-backoff-protocol)
- [gRPC Connectivity Semantics and API](#grpc-connectivity-semantics-and-api)
- [Auto-reconnect for Go clients?](#auto-reconnect-for-go-clients)


## 一个实际例子

详见 [perform indefinite background reconnection attempts](https://github.com/census-ecosystem/opencensus-go-exporter-ocagent/commit/ea77d4d5284c1f9646daf88bed6937e50f0824bf)

关键：

- 定义不同状态 sDisconnected/sConnected ，并进行相应的判定和切换；
- background 无限重连（重连超时 defaultConnReattemptPeriod 为 800 * time.Millisecond ）
- 通过增加 jitter 避免同时 retry 造成类似 DDOS 的情况；
- 当 TCP 连接出错后，通过上述无限循环重建连接，并在新连接上重建 stream ，而不是复用之前的 stream ；这里正是之前困扰我的地方（其实想通了很好理解）；


```golang
func (ae *Exporter) connect() error {
  cc, err := ae.dialToAgent()
  if err != nil {
    return err
  }
  return ae.enableConnectionStreams(cc)
}
```


## [GRPC client reconnect inside kubenetes](https://stackoverflow.com/questions/39277063/grpc-client-reconnect-inside-kubenetes)

问题：

> We define our micro service inside kubenetes Pods, do we need to instrument Grpc client reconnection if the service pod is restarting? When the pod restarts, the host name is not changed, but we cannot guarantee the IP address remains the same. So does the grpc client still be able to detect the new server to reconnect to?

回答：

> When the TCP connection is disconnected (because the old pod stopped) gRPC's channel will attempt to reconnect with exponential backoff. Each reconnect attempt implies resolving the DNS address, although it may not detect the new address immediately because of the TTL (time-to-live) of the old DNS entry. Also, I believe some implementations resolve the address when a failure is detected instead of before an attempt.
>
> This process happens naturally without your application doing anything, although it may experience RPC failures until the connection is re-established. Enabling "wait for ready" on an RPC would reduce the chances the RPC fails during this period of transition, although such an RPC generally implies you don't care about response latency.
> 
> If the DNS address is not (eventually) re-resolved, then that would be a bug and you should file an issue.

## [grpc-go: get ClientConn to reconnect forever](https://groups.google.com/forum/#!topic/grpc-io/OAdD-7tMJHY)

> Q: I'd like to know **if it's possible to have a `ClientConn` try to reconnect forever**? I have a service that needs to be resilient to one of its dependencies failing. When the dependency goes offline for some reason, I use a **circuit breaker** that gets triggered and gives me a chance to handle the failure by using a fallback solution. Eventually in the future, the dependency would come back online and the **circuit breaker** would close back, the behavior of my service returning to normal.

> Q: My problem is that after a timeout, `grpc-go` seems to consider a client connection to be shutdown and it seems it will not try to reconnect anymore. So my **circuit breaker** never closes back and all the instances of my service remain in degraded mode until I restart their process. So is there a way to have the `ClientConn` retry forever, until explicitly closed?

问题：ClientConn 能够一直自动重连么？

> A: If you do not set a connect time, `grpc-go` will keep reconnecting. The interval between reconnecting is exponential backoff.

回答：

- 如果创建 gRPC 连接时没有设置连接（超时）时间，`grpc-go` 就能够一直不断重连；
- 重连时间间隔实现为指数退避；

## [stream auto reconnect](https://groups.google.com/forum/#!topic/grpc-io/PEFwhLXT2wo)

> Q: I wonder **how gRPC auto reconnect client and server** (if server is down and restart?) I try to build a **bi-directional stream**. When running server, and starting the client, the client starts sending random int to server. And I `Ctrl-C` the server side, the client shows: "xxx". Then I restart the server, but it seems client cannot rebuild the stream. Anyway to recover a stream?

问题：在使用双向流的场景中，先确保 client 和 server 正常通信，然后停止 server ，client 侧开始报错；之后重启 server ，但是 client 并不会重建 stream ；

> A: When you receive the error on the client-side, the stream is dead. You should simply create a new RPC using the same Stub/Channel. **The Channel will automatically create a new connection to the server, but it can't re-establish any streams**.

回答：

- 当 client 侧收到 error 时，就表明当前 stream 已经 dead 了；
- 此时你应该只需在之前使用的 Stub/Channel 上发起一个新的 RPC 就能触发该 Stub/Channel 自动创建一条到 server 的新连接；但是，并不会重建任何 streams ；

> A: When **load balancing** and **proxies** are involved, a particular stream always goes to the same backend. So if that stream breaks, gRPC can't simply issue a new request automatically, because it can't necessarily get the same backend, and the backend may no longer exist (which is the case when you `Ctrl+C` the server; when you restart, it is a different server process). So **applications should just re-establish the failed stream**.

回答：

- 当存在使用 LB 和 proxies 的场景，（并且需要确保）任意一条特定的 stream 总是对应到相同的 backend 时；若发生 stream 断开的情况，则 gRPC 不能简单的、按照自动方式发起一个新的 request ，因为此时无法保证其能够到达和之前相同的 backend ；并且也无法保证之前的 backend 仍旧存在（即上面通过 `Ctrl+C` 停止 server 就属于这种情况）；
- 因此应用应该针对失效的 stream 进行重新建立；

结论：**应用应该针对失效的 stream 进行重新建立**！！

## [Best practices for reusing connections, concurrency](https://github.com/grpc/grpc-go/issues/682)

问题

> I have a fixed number of machines I wish to make potentially concurrent RPCs to.
>
> I would like maintain a pool of connections (`*grpc.ClientConn`) to these machines.
>
> Otherwise, for every RPC call I need to make a new connection, which has overhead (as I understand it) and also can exhaust the number of file descriptors when a large number of concurrent RPC's are taking place.
>
> - **What are the restrictions on using clients and connections concurrently**?
> - **Can I simply maintain a single connection and attach multiple client stubs to it dynamically (more flexible), or can I only have a single stub of a given type for a connection**?
> - My expectation is that I need to maintain my own pool with occasional heartbeats to maintain the health of the connection. Is that correct?

回答1：

> I'm not the gRPC maintainer, but have used gRPC extensively.
>
> - **AFAIK you can use the connections concurrently between clients**. The only problem may be the limitation of number of concurrent HTTP2 streams you can do, but these count in thousands.
> - Yes.
> - Yes and no.
> 
>> - gRPC go has a [`Picker`](https://github.com/grpc/grpc-go/blob/9e3a674ceba65708273cf1cd8e71a1bdce68107b/picker.go#L49) interface which deals with choosing a channel (tcp connection) to send it down to. Ideally you'd do all your pooling here. 
>> - gRPC 1.1 will have a way of healthchecking a backend, making `Picker` implementations better.

回答2：

> **You do not need to make a new connection for every RPC. A connection can be multiplexed by multiple clients or RPCs**.
> 
> We have health check service (https://github.com/grpc/grpc/blob/master/doc/health-checking.md) so that a client can detect the health of the services. We are also designing the connection level story (L7 counterpart of TCP keepalive).

回答3：

> **`ClientConns` can safely be accessed concurrently, and RPCs will be sent in parallel**.
> 
> The only reason I can think of to use a pool of grpc clients to the same backend(s) is if you're running into stream limits or per-connection throughput limits, possibly imposed by a proxy outside your control.

回答4：

> Technically, **a `ClientConn` (inappropriately named IMO) can consistent of many connections** that come and go for various reasons. However, the `ClientConn` should manage the connections itself, so if a connection is broken, it will reconnect automatically. And if you have multiple backends, it's possible to connect to multiple of them and load balance between them. 

## [GRPC Connection Backoff Protocol](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)

> When we do a connection to a backend which fails, it is typically desirable to not retry immediately (to avoid flooding the network or the server with requests) and instead do some form of exponential backoff.

- 与 backend 的连接失效后，典型作法是不立即重试（连接），以避免 flooding 网络或 server ；
- 取而代之，是采用某种形式的指数退避来实现；

> We have several parameters:
> 
> - **INITIAL_BACKOFF** (how long to wait after the first failure before retrying) -- 首次失效重试前需要等待的时间
> - **MULTIPLIER** (factor with which to multiply backoff after a failed retry)  -- 重试失败后作用在 backoff 的倍乘因子
> - **JITTER** (by how much to randomize backoffs).  -- 引入的随机 backoffs 毛刺
> - **MAX_BACKOFF** (upper bound on backoff) -- backoff 的上限
> - **MIN_CONNECT_TIMEOUT** (minimum time we're willing to give a connection to complete)  -- 最少需要等待连接完成的时间

### Proposed Backoff Algorithm

> Exponentially back off the start time of connection attempts up to a limit of **MAX_BACKOFF**, with **jitter**.

```
ConnectWithBackoff()
  current_backoff = INITIAL_BACKOFF
  current_deadline = now() + INITIAL_BACKOFF
  while (TryConnect(Max(current_deadline, now() + MIN_CONNECT_TIMEOUT))
         != SUCCESS)
    SleepUntil(current_deadline)
    current_backoff = Min(current_backoff * MULTIPLIER, MAX_BACKOFF)
    current_deadline = now() + current_backoff +
      UniformRandom(-JITTER * current_backoff, JITTER * current_backoff)
```

> With specific parameters of **MIN_CONNECT_TIMEOUT** = 20 seconds **INITIAL_BACKOFF** = 1 second **MULTIPLIER** = 1.6 **MAX_BACKOFF** = 120 seconds **JITTER** = 0.2

以上为提议的指数退避算法

> Implementations with pressing concerns (such as minimizing the number of wakeups on a mobile phone) may wish to use a different algorithm, and in particular different jitter logic.

> Alternate implementations must ensure that connection backoffs started at the same time disperse, and must not attempt connections substantially more often than the above algorithm.

指数退避算法的具体实现还需要考虑其他的因素；

### Reset Backoff

> The back off should be reset to **INITIAL_BACKOFF** at some time point, so that the reconnecting behavior is consistent no matter the connection is a newly started one or a previously disconnected one.

> We choose to reset the Backoff when the **SETTINGS** frame is received, at that time point, we know for sure that this connection was accepted by the server.

针对何时重置指数退避时间的说明；

## [gRPC Connectivity Semantics and API](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md)

> This document describes the **connectivity semantics for gRPC channels** and the corresponding impact on RPCs. We then discuss an API.

该文档描述了 gRPC channels 的 connectivity 语义，以及其对 RPCs 的影响；

> **gRPC Channels** provide the abstraction over which clients can communicate with servers. The client-side channel object can be constructed using little more than a DNS name. **Channels** encapsulate a range of functionality including name resolution, establishing a TCP connection (**with retries and backoff**) and TLS handshakes. Channels can also handle errors on established connections and reconnect, or in the case of HTTP/2 GO_AWAY, re-resolve the name and reconnect.

- gRPC Channels 是一个特定概念，抽象了 client 和 server 之间的通信；
- gRPC Channels 封装以下功能：
    - 域名解析
    - TCP 连接建立（包括连接重试+指数退避）
    - TLS 握手
    - 针对 ESTAB 连接的错误处理和重连
    - HTTP/2 相关

> To hide the details of all this activity from the **user of the gRPC API** (i.e., application code) while exposing meaningful information about the state of a channel, we use a **state machine** with five states, defined below:

> **CONNECTING**: The channel is **trying to establish** a connection and is **waiting to make progress** on one of the steps involved in name resolution, TCP connection establishment or TLS handshake. This may be used as the **initial state for channels upon creation**.

> **READY**: The channel has **successfully established** a connection all the way through TLS handshake (or equivalent) and all subsequent attempt to communicate have succeeded (or are pending without any known failure ).

> **TRANSIENT_FAILURE**: There has been some **transient failure** (such as a TCP 3-way handshake timing out or a socket error). Channels in this state will eventually switch to the CONNECTING state and **try to establish a connection again**. Since retries are done with exponential backoff, channels that fail to connect will start out spending very little time in this state but as the attempts fail repeatedly, the channel will spend increasingly large amounts of time in this state. For many non-fatal failures (e.g., TCP connection attempts timing out because the server is not yet available), the channel may spend increasingly large amounts of time in this state.

> **IDLE**: This is the state where the channel is **not even trying to create a connection** because of a lack of new or pending RPCs. New RPCs MAY be created in this state. **Any attempt to start an RPC on the channel will push the channel out of this state to connecting**. When there has been no RPC activity on a channel for a specified `IDLE_TIMEOUT`, i.e., no new or pending (active) RPCs for this period, channels that are READY or CONNECTING switch to IDLE. Additionaly, channels that receive a **GOAWAY** when there are no active or pending RPCs should also switch to IDLE to avoid connection overload at servers that are attempting to shed connections. We will use a default `IDLE_TIMEOUT` of 300 seconds (5 minutes).

> **SHUTDOWN**: This channel has started shutting down. Any **new RPCs** should fail immediately. **Pending RPCs** may continue running till the application cancels them. Channels may enter this state either because the application explicitly requested a shutdown or if a non-recoverable error has happened during attempts to connect communicate . (As of 6/12/2015, there are no known errors (while connecting or communicating) that are classified as non-recoverable) Channels that enter this state never leave this state.

描述 channel 状态的状态机的五种状态定义；

下面的表格列出了状态机的合法转换以及相应的原因；空格部分表示不允许状态转换；

<table style='border: 1px solid black'>
  <tr>
    <th>From/To</th>
    <th>CONNECTING</th>
    <th>READY</th>
    <th>TRANSIENT_FAILURE</th>
    <th>IDLE</th>
    <th>SHUTDOWN</th>
  </tr>
  <tr>
    <th>CONNECTING</th>
    <td>Incremental progress during connection establishment</td>
    <td>All steps needed to establish a connection succeeded</td>
    <td>Any failure in any of the steps needed to establish connection</td>
    <td>No RPC activity on channel for IDLE_TIMEOUT</td>
    <td>Shutdown triggered by application.</td>
  </tr>
  <tr>
    <th>READY</th>
    <td></td>
    <td>Incremental successful communication on established channel.</td>
    <td>Any failure encountered while expecting successful communication on
        established channel.</td>
    <td>No RPC activity on channel for IDLE_TIMEOUT <br>OR<br>upon receiving a GOAWAY while there are no pending RPCs.</td>
    <td>Shutdown triggered by application.</td>
  </tr>
  <tr>
    <th>TRANSIENT_FAILURE</th>
    <td>Wait time required to implement (exponential) backoff is over.</td>
    <td></td>
    <td></td>
    <td></td>
    <td>Shutdown triggered by application.</td>
  </tr>
  <tr>
    <th>IDLE</th>
    <td>Any new RPC activity on the channel</td>
    <td></td>
    <td></td>
    <td></td>
    <td>Shutdown triggered by application.</td>
  </tr>
  <tr>
    <th>SHUTDOWN</th>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
  </tr>
</table>

Channel State API -- 略


## [Auto-reconnect for Go clients?](https://github.com/grpc/grpc-go/issues/351)

- grpc has a backoff strategy for reconnecting ([doc/connection-backoff](https://github.com/grpc/grpc/blob/master/doc/connection-backoff.md)).
- The go
impl is at [here](https://github.com/grpc/grpc-go/blob/master/rpc_util.go#L288).
- see https://github.com/grpc/grpc-go/blob/master/clientconn.go#L384.



