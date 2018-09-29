# gRPC 知识梳理

## gRPC 背景介绍

- gRPC 是一个高性能、通用的开源 RPC 框架，其由 Google 主要面向移动应用开发并基于 HTTP/2 协议标准而设计，基于 ProtoBuf (Protocol Buffers) 序列化协议开发，且支持众多开发语言。 gRPC 提供了一种简单的方法来精确地定义服务和为 iOS、Android 和后台支持服务自动生成可靠性很强的客户端功能库；客户端充分利用高级流和链接功能，从而有助于节省带宽、降低的 TCP 链接次数、节省 CPU 使用、和电池寿命；
- gRPC 支持多种语言 支持 C++、Java、Go、Python、Ruby、C#、Node.js、Android Java、Objective-C、PHP 等编程语言；
- 各个模块是系统的筋肉，那 RPC 就是整个系统的血管，数据的流通，信令的传递，都离不开 RPC ；
- RPC 并不是一个固定的东西，可重可轻；
- 大家也都知道 Google 内部其实没怎么用 gRPC，大量使用的是 Stubby，它作为 gRPC 的前身，也是一个 Protobuf RPC 的实现，因为大量依赖了 Google 的其他基础服务所以不太方便开放出来给社区使用；
- 随着 SPDY / QUIC，乃至 HTTP/2  的成熟，Google 决定用这些更加标准的组件来构建一个新的 RPC 框架，也就是 gRPC ；
- 现在为止包括 ETCD/Kubernetes/TiDB 在内的大量社区顶级开源分布式项目都在使用它；

## 为何选择 gRPC 

从官方的 gRPC 的设计动机和原则说起：

- Google 应该是践行**服务化**的先驱之一，在业界没那么推崇微服务的时代，Google 就已经大规模的微服务化。**微服务的精髓之一就是服务之间传递的是可序列化消息**，而不是对象和引用，这个思想是和 DCOM 及 EJB 完全相反的。只有数据，不包含逻辑；这个设计的好处不用我多说也很好理解，参考 CSP ；
- Protobuf 作为一个良好的序列化方案，注意，只是序列化（尽管 pb 也有定义 rpc service 的能力，Protobuf 默认生成的代码并不包含 RPC 的实现），它并不像 Thrift 天生就带一个 RPC Framework，相对的来说比较轻。**在 gRPC 的设计中，一个很重要的原则就是 Payload agnostic**，RPC 框架不应该规定用的是什么 payload 格式，可以是 Protobuf，JSON，XML，这也让 gRPC 的设计和层次更加清晰；
- 比传统的 Request / Response 更丰富的 API Interface，这个是我们使用 gRPC 的重要理由，gRPC 不仅支持传统的一应一答的模式，更是支持三种 **Streaming** 的调用方式，现代的业务经常会需要传输大的数据流，Streaming API 的设计让这些业务写起来轻松很多；
- 有了 Streaming 就不可避免地需要引入 **Flow-control** ，这点 gRPC 的处理很聪明，**直接依赖了 HTTP/2** ，在流控这边不怎么用操心，顺带还可以用 HTTP 反向代理做负载均衡。但是另一方面也会带来更多的调优复杂度，毕竟和 Web 的使用场景不太一样，比如默认的 `INITIAL_WINDOW_SIZE` 在 gRPC 里是 64k，太小，吞吐上不去，需要人工改大；
- 另一方面由于直接使用了 HTTP/2 ，TLS 的支持就几乎是天然的，对于 TiDB 这样的商业数据库而言，传输层加密是一个很重要的功能，在 gRPC 中直接就可以支持。本着不重新造轮子的原则，直接用 gRPC 就好了；

## gRPC streaming

gRPC streaming 可以分为三类：

- 客户端流式发送（单向流）；
- 服务器流式返回（单向流）；
- 客户端／服务器同时流式处理（双向流）；

具体：

- 通过使用 streaming ，你可以向服务器或者客户端发送批量的数据， 服务器和客户端在接收这些数据的时候，可以不必等所有的消息全收到后才开始响应，而是接收到第一条消息的时候就可以及时的响应， 这显然比以前的类 HTTP 1.1 的方式更快的提供响应，从而提高性能；
- 当前 gRPC 通过 HTTP2 协议传输，可以方便的实现 streaming 功能。 如果你对 gRPC 如何通过 HTTP2 传输的感兴趣， 你可以阅读这篇文章 [gRPC over HTTP2](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md) ，它描述了 gRPC 通过 HTTP2 传输的低层格式；
- 要实现**客户端的流式发送**，只需在 proto 中的方法定义中将**请求**前面加上 `stream` 标记；
- 要实现**服务器的流式响应**，只需在 proto 中的方法定义中将**响应**前面加上 `stream` 标记；需要注意的是，服务器端代码的实现要通过流的方式发送响应；对象流（响应流）序列化后流式返回；
- 要实现**双向流**，只需在 proto 中的方法定义中将**请求**和**响应**前面都加上 `stream` ；


### 客户端 non-stream 发送请求，stream 接收应答

![](https://camo.githubusercontent.com/65cd9faa62d0bbea03c22498deb7633ae29ff455/68747470733a2f2f692e696d6775722e636f6d2f367075796e77342e706e67)

```
stream, err := c.SayHello1(context.Background(), &pb.HelloRequest{Name: *name})
if err != nil {
	log.Fatalf("could not greet: %v", err)
}
for {
	reply, err := stream.Recv()
	if err == io.EOF {
		break
	}
	if err != nil {
		log.Printf("failed to recv: %v", err)
	}
	log.Printf("Greeting: %s", reply.Message)
}
```

### 客户端 stream 发送请求，non-stream 接收应答

![](https://camo.githubusercontent.com/5cbd779f64c3819afc62ae7c23ac85bc89e4c69e/68747470733a2f2f692e696d6775722e636f6d2f7778446c67704d2e706e67)

- client 侧

客户端读取的方法是 `stream.CloseAndRecv()` ，读取完毕会关闭这个流的发送，这个方法返回最终结果。注意**客户端只负责关闭流的发送**。

```
	stream, err := c.SayHello2(context.Background())
	for i := 0; i < 100; i++ {
		if err != nil {
			log.Printf("failed to call: %v", err)
			break
		}
		stream.Send(&pb.HelloRequest{Name: *name + strconv.Itoa(i)})
	}
	reply, err := stream.CloseAndRecv()     -- 注意这里
	if err != nil {
		fmt.Printf("failed to recv: %v", err)
	}
```

- server 侧

服务器端收到每条消息都进行了处理，这里的处理简化为增加到一个 slice 中。一旦它检测的客户端关闭了流的发送，它则把最终结果发送给客户端，通过关闭这个流。流的关闭通过 `io.EOF` 这个 error 来区分。

```
	for {
		in, err := gs.Recv()
		if err == io.EOF {   -- 当客户端侧关闭流的发送时，服务器侧也关闭流的发送
			gs.SendAndClose(&pb.HelloReply{Message: "Hello " + strings.Join(names, ",")})
			return nil
		}
		if err != nil {
			log.Printf("failed to recv: %v", err)
			return err
		}
		names = append(names, in.Name)
	}
```


### 双向流

- client 侧

通过 `stream.Send` 发送请求，通过 `stream.Recv` 读取响应。客户端可以通过 `CloseSend` 方法关闭发送流；

```
	stream, err := c.SayHello3(context.Background())
	if err != nil {
		log.Printf("failed to call: %v", err)
		return
	}
	var i int64
	for {
		stream.Send(&pb.HelloRequest{Name: *name + strconv.FormatInt(i, 10)})
		if err != nil {
			log.Printf("failed to send: %v", err)
			break
		}
		reply, err := stream.Recv()
		if err != nil {
			log.Printf("failed to recv: %v", err)
			break
		}
		log.Printf("Greeting: %s", reply.Message)
		i++
	}
```

- server 侧

服务器端代码也是通过 Send 发送响应，通过 Recv 响应；

```
	for {
		in, err := gs.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			log.Printf("failed to recv: %v", err)
			return err
		}
		gs.Send(&pb.HelloReply{Message: "Hello " + in.Name})
	}
```



## 经验

- 基于 HTTP/2 的 flow control ，和参数 `INITIAL_WINDOW_SIZE` 有关，默认 64k，调大有助于提高吞吐；
- HTTP/2 是单连接的，实际测试发现也制约了吞吐；实践中不管是 TiDB 连接 TiKV 还是 TiKV 之间的连接都是采用多个 gRPC client 的方式来同时建立多个 HTTP/2 连接；
- 如果你知道自己的 workload 的大小，通过适当的调整 `GRPC_WRITE_BUFFER_HINT` 改变 write buffer 的大小也能显著减少 syscall 的调用，详见 [grpc/issues#9121](https://github.com/grpc/grpc/issues/9121)；
- `GRPC_ARG_MAX_CONCURRENT_STREAMS` 规定在一个 HTTP/2 连接中最多存在多少 stream，在 gRPC 中一次 RPC 就是一个 stream。在 TiKV 的应用场景中，适当调高该参数同样有助于提高吞吐；
- gRPC 本身不适用于传送大文件的场景，详见 [grpc-go/issues#414](https://github.com/grpc/grpc-go/issues/414) ；TiKV 之间发送 snapshot 就是采用 issue 中推荐的方案，把大文件拆成多个 chunk 后使用 client streaming 发送；


调优示例代码（Ref: [here](https://github.com/etcd-io/etcd/blob/1a282a72bec1fb55692698bf3cd06d7d191d1906/functional/agent/server.go#L95)）

```golang
const (
	maxRequestBytes   = 1.5 * 1024 * 1024
	grpcOverheadBytes = 512 * 1024
	maxStreams        = math.MaxUint32
	maxSendBytes      = math.MaxInt32
)
...
	var opts []grpc.ServerOption
	opts = append(opts, grpc.MaxRecvMsgSize(int(maxRequestBytes+grpcOverheadBytes)))
	opts = append(opts, grpc.MaxSendMsgSize(maxSendBytes))
	opts = append(opts, grpc.MaxConcurrentStreams(maxStreams))
	srv.grpcServer = grpc.NewServer(opts...)
```

## Q&A

### 什么是 RPC

RPC 代指远程过程调用（Remote Procedure Call），它的调用包含了传输协议和编码（对象序列号）协议等等。允许运行于一台计算机的程序调用另一台计算机的子程序，而开发人员无需额外地为这个交互作用编程；

### 什么是 RPC 框架

一个完整的 RPC 框架，应包含**负载均衡**、**服务注册和发现**、**服务治理**等功能，并**具有可拓展性**，便于流量监控系统等接入，那么它才算完整的，当然了。有些较单一的 RPC 框架，通过组合多组件也能达到这个标准；

### 为什么要 RPC

简单、通用、安全、效率

### RPC 可以基于 HTTP 吗

RPC 是代指远程过程调用，是可以基于 HTTP 协议的；

肯定会有人说效率优势，我可以告诉你，那是基于 HTTP/1.1 来讲的，HTTP/2 优化了许多问题（当然也存在新的问题），所以你看到了本文的主题 gRPC ；

### Protobuf 是什么

Protobuf 即 Protocol Buffers ，是一种与语言、平台无关，可扩展的序列化结构化数据的方法，常用于通信协议，数据存储等等。相较于 JSON、XML，它更小、更快、更简单，因此也更受开发人员的青眯；

### 相较 Protobuf ，为什么不使用 XML ？

- 更简单
- 数据描述文件只需原来的 1/10 至 1/3
- 解析速度是原来的 20 倍至 100 倍
- 减少了二义性
- 生成了更易使用的数据访问类

### gRPC 的特点是什么

gRPC 是一个高性能、开源和通用的 RPC 框架，面向移动和 HTTP/2 设计；特点如下：

- HTTP/2
- Protobuf
- 客户端、服务端基于同一份 IDL
- 移动网络的良好支持
- 支持多语言

### 什么场景下不适合使用 Protobuf，而适合使用 JSON、XML？

> todo

### gRPC channel 是什么？

A **gRPC channel** provides a connection to a gRPC server on a specified host and port and is used when creating a **client stub** (or just “client” in some languages). Clients can specify channel arguments to modify gRPC’s default behaviour, such as switching on and off message compression. A channel has state, including `connected` and `idle`.

How gRPC deals with closing down channels is language-dependent. Some languages also permit querying channel state.


### `stream.Send()` 方法能发送多少次？有没有大小限制？

Send 内部的 SendMsg 方法，该方法涉及以下过程:

- 消息体（对象）序列化
- 压缩序列化后的消息体
- 对正在传输的消息体增加 5 个字节的 header
- 判断压缩+序列化后的消息体总字节长度是否大于预设的 `maxSendMessageSize`（预设值为 `math.MaxInt32`），若超出则提示错误
- 写入给流的数据集

### `stream.Recv()` 方法在什么情况下返回 `io.EOF` ？什么情况下存在错误信息呢？

Recv 内部调用 RecvMsg ，后者会从流中读取完整的 gRPC 消息体，另外通过阅读源码可得知：

- RecvMsg 是**阻塞**等待的；
- RecvMsg 当流成功/结束（对端调用了 Close）时，会返回 `io.EOF` ；
- RecvMsg 当流出现任何错误时，流会被中止，错误信息会包含 RPC 错误码；
- 需要注意，默认的 `MaxReceiveMessageSize` 值为 1024 * 1024 * 4，建议不要超出；

可能出现的错误有：

- io.EOF
- io.ErrUnexpectedEOF
- transport.ConnectionError
- https://google.golang.org/grpc/codes 中定义的内容



## 图片

![](https://grpc.io/img/landing-2.svg)

![](https://res.infoq.com/articles/tidb-and-grpc/zh/resources/00.jpg)

![](https://res.infoq.com/articles/tidb-and-grpc/zh/resources/01.jpg)

![](https://camo.githubusercontent.com/27cdbcffc801a125c279b0b0beddfce10380c5e7/68747470733a2f2f692e696d6775722e636f6d2f676367324938742e706e67)


参考：

- [TiDB与gRPC的那点事](http://www.infoq.com/cn/articles/tidb-and-grpc) -- by 黄东旭
- [gRPC的那些事 - streaming](https://colobu.com/2017/04/06/dive-into-gRPC-streaming/)
- [带入gRPC：gRPC及相关介绍](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-09-23-%E5%B8%A6%E5%85%A5gRPC%E4%B8%80-gRPC%E5%8F%8A%E7%9B%B8%E5%85%B3%E4%BB%8B%E7%BB%8D.md)
- [带入gRPC：gRPC Client and Server](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-09-23-%E5%B8%A6%E5%85%A5gRPC%E4%BA%8C-gRPC-Client-and-Server.md)
- [带入gRPC：gRPC Streaming, Client and Server](https://github.com/EDDYCJY/blog/blob/master/golang/gRPC/2018-09-24-%E5%B8%A6%E5%85%A5gRPC-gRPC-Streaming-Client-and-Server.md)




