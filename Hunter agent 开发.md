# Hunter agent 开发

## 项目地址

- [moooofly/hunter-agent](https://github.com/moooofly/hunter-agent)
- [hunter-agent 完整功能列表](https://github.com/moooofly/hunter-agent/issues/1)

## 梳理

可能需要解决的问题：

- 和 kafka 之间连接的处理：鉴权，SSL 等
- 端口使用问题
- 暴露 prometheus metrics
- daemon + unix socket + tcp socket + signal 处理
- 配置热加载
- 配置文件处理
    - [spf13/viper](https://github.com/spf13/viper) --
Viper is a complete configuration solution for Go applications including 12-Factor apps. It is designed to work within an application, and can handle all types of configuration needs and formats. Viper can be thought of as a registry for all of your applications configuration needs.
    - [toml-lang/toml](https://github.com/toml-lang/toml) -- TOML aims to be a minimal configuration file format that's easy to read due to obvious semantics. TOML is designed to map unambiguously to a hash table. TOML should be easy to parse into data structures in a wide variety of languages.
- 控制能力
    - 可以控制代理特定的请求
    - 屏蔽特定的请求
    - 甚至可以重写特定的请求
- 日志处理
    - [Sirupsen/logrus](https://github.com/Sirupsen/logrus) --
Logrus is a structured logger for Go (golang), completely API compatible with the standard library logger.
    - [uber-go/zap](https://github.com/uber-go/zap) --
Blazing fast, structured, leveled logging in Go.
    - [golang/glog](https://github.com/golang/glog) -- Leveled execution logs for Go
    - [natefinch/lumberjack](https://github.com/natefinch/lumberjack) -- a Go package for writing logs to rolling files.
- 可能需要保活探测功能
- 基于 docker-compose 进行本地服务的构建（[这里](https://github.com/openzipkin/docker-zipkin#docker-compose)）


## hunter-agent 目录结构


```shell
[#827#root@ubuntu-1604 /go/src/github.com/moooofly/hunter-agent]$tree . -I vendor
.
├── agent.json.template
├── CHANGELOG.md
├── cli
│   ├── cobra.go
│   ├── debug
│   │   ├── debug.go
│   │   └── debug_test.go
│   ├── error.go
│   └── required.go
├── client
├── cmd
│   └── agent
│       ├── agent.go
│       ├── config.go
│       ├── daemon.go
│       ├── metrics.go
│       ├── options.go
│       └── server.go
├── daemon
│   ├── config
│   │   └── config.go
│   ├── daemon.go
│   ├── daemon_unix.go
│   ├── listeners
│   │   └── listeners_linux.go
│   └── reload.go
├── Dockerfile
├── docs
│   ├── 20180912_test_report.md
│   ├── callchain.png
│   ├── callchain.svg
│   ├── protocol.png
│   ├── protocol.svg
│   ├── whole.png
│   └── whole.svg
├── gen-go
│   └── dumpproto
│       └── dump.pb.go
├── glide.lock
├── glide.yaml
├── LICENSE
├── Makefile
├── mkgogen.sh
├── opencensus
│   └── proto
│       └── trace
│           └── trace.proto
├── opts
│   ├── hosts_unix.go
│   ├── opts.go
│   └── opts_test.go
├── pkg
│   ├── fileutils
│   │   └── fileutils.go
│   ├── pidfile
│   │   ├── pidfile.go
│   │   ├── pidfile_test.go
│   │   └── pidfile_unix.go
│   ├── signal
│   │   ├── signal.go
│   │   ├── signal_linux.go
│   │   ├── signal_linux_test.go
│   │   ├── signal_test.go
│   │   ├── trap.go
│   │   └── trap_linux_test.go
│   ├── system
│   │   └── filesys.go
│   └── term
│       ├── tc.go
│       ├── term.go
│       ├── termios_linux.go
│       └── winsize.go
├── proto
│   └── dump.proto
├── proxy
├── README.md
├── version
│   └── version.go
└── VERSION

23 directories, 56 files
[#828#root@ubuntu-1604 /go/src/github.com/moooofly/hunter-agent]$
```

目录结构：

- cli:
- client: 用于和 hunter-agent 进行交互的 cli client
- cmd:
    - agent: hunter-agent 主程序
- daemon:
    - config: agent 配置处理
- opts: 通用功能实现
- docs: 各类文档


----------


## TODO

使用了很多以前没有使用过的 package ，需要理解

- github.com/spf13/cobra
- github.com/pkg/errors
- github.com/docker/go-units
- github.com/imdario/mergo


----------


- [opencensus-proto/opencensus/proto/exporter/exporter.proto](https://github.com/census-instrumentation/opencensus-proto/blob/master/opencensus/proto/exporter/exporter.proto)

```
syntax = "proto3";

// NOTE: This proto is experimental and is subject to change at this point.
// Please do not use it at the moment.

// 指定包名
// 当外部 .proto 文件需要访问当前文件中定义的内容时，需要使用该名字
package opencensus.proto.exporter;

// 引用其他 .proto 文件
// 当引用其他文件中定义的内容时，需要使用其 .proto 中定义的包名
import "opencensus/proto/trace/trace.proto";
import "opencensus/proto/metrics/metrics.proto";

// java 相关，可以不必关注
option java_multiple_files = true;
option java_package = "io.opencensus.proto.exporter";
option java_outer_classname = "ExporterProto";

// 设置生成 go package 时的 package 名字
// 生成路径和 --go_out=plugins=grpc:$OUTDIR 的内容对应，即生成 exporter.pb.go 到 
// $OUTDIR/github.com/census-instrumentation/opencensus-proto/gen-go/exporterproto 目录下面
option go_package = "github.com/census-instrumentation/opencensus-proto/gen-go/exporterproto";

message ExportSpanRequest {
  // 这里引用了 trace.proto 中定义的内容
  repeated opencensus.proto.trace.Span spans = 1;
}

message ExportSpanResponse {
}

message ExportMetricsRequest {
  // 这里引用了 metrics.proto 中定义的内容
  repeated opencensus.proto.metrics.Metric metrics = 1;
}

message ExportMetricsResponse {
}

service Export {
  rpc ExportSpan (stream ExportSpanRequest) returns (stream ExportSpanResponse) {}
  rpc ExportMetrics (stream ExportMetricsRequest) returns (stream ExportMetricsResponse) {}
}
```

关键字说明：

- repeated
- message
- service
- stream


----------


- [opencensus-proto/gen-go/exporterproto/exporter.pb.go](https://github.com/census-instrumentation/opencensus-proto/blob/master/gen-go/exporterproto/exporter.pb.go)

```golang
// Code generated by protoc-gen-go. DO NOT EDIT.
// source: opencensus/proto/exporter/exporter.proto

package exporterproto // import "github.com/census-instrumentation/opencensus-proto/gen-go/exporterproto"

import proto "github.com/golang/protobuf/proto"
import fmt "fmt"
import math "math"

// 对应了 .proto 文件中 import "opencensus/proto/trace/metrics.proto";
// 导入 package 名字取决于 option go_package 定义
import metricsproto "github.com/census-instrumentation/opencensus-proto/gen-go/metricsproto"
// 对应了 .proto 文件中 import "opencensus/proto/trace/trace.proto";
// 导入 package 名字取决于 option go_package 定义
import traceproto "github.com/census-instrumentation/opencensus-proto/gen-go/traceproto"
...
```


----------


## 对比

### message 关键字

`exporter.proto` 中的

```
message ExportSpanRequest {
  repeated opencensus.proto.trace.Span spans = 1;
}
```

对应 `exporter.pb.go` 中的

```golang
type ExportSpanRequest struct {
    Spans                []*traceproto.Span `protobuf:"bytes,1,rep,name=spans,proto3" json:"spans,omitempty"`
    XXX_NoUnkeyedLiteral struct{}           `json:"-"`
    XXX_unrecognized     []byte             `json:"-"`
    XXX_sizecache        int32              `json:"-"`
}
```

可以看出：`repeated` 对应了 `[]xxx` 切片类型；

### service 关键字

`exporter.proto` 中的

```
service Export {
  rpc ExportSpan (stream ExportSpanRequest) returns (stream ExportSpanResponse) {}
  rpc ExportMetrics (stream ExportMetricsRequest) returns (stream ExportMetricsResponse) {}
}
```

对应

- client 侧

```golang
// ExportClient is the client API for Export service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please refer to https://godoc.org/google.golang.org/grpc#ClientConn.NewStream.

// 1. rpc client 需要实现的接口
type ExportClient interface {
    ExportSpan(ctx context.Context, opts ...grpc.CallOption) (Export_ExportSpanClient, error)
    ExportMetrics(ctx context.Context, opts ...grpc.CallOption) (Export_ExportMetricsClient, error)
}

// 2. 实现了上述接口的 client 结构
type exportClient struct {
    cc *grpc.ClientConn
}

// 3. client 实例化函数
func NewExportClient(cc *grpc.ClientConn) ExportClient {
    return &exportClient{cc}
}

// 4. 转化为 client stream 的函数
func (c *exportClient) ExportSpan(ctx context.Context, opts ...grpc.CallOption) (Export_ExportSpanClient, error) {
    stream, err := c.cc.NewStream(ctx, &_Export_serviceDesc.Streams[0], "/opencensus.proto.exporter.Export/ExportSpan", opts...)
    if err != nil {
        return nil, err
    }
    x := &exportExportSpanClient{stream}
    return x, nil
}

// 5. 定义 client stream 需要实现的接口
type Export_ExportSpanClient interface {
    Send(*ExportSpanRequest) error
    Recv() (*ExportSpanResponse, error)
    grpc.ClientStream
}

// 6. 实现了上述接口的 client stream 结构
type exportExportSpanClient struct {
    grpc.ClientStream
}

// 7. client stream send (request)
func (x *exportExportSpanClient) Send(m *ExportSpanRequest) error {
    return x.ClientStream.SendMsg(m)
}

// 8. client stream recv (response)
func (x *exportExportSpanClient) Recv() (*ExportSpanResponse, error) {
    m := new(ExportSpanResponse)
    if err := x.ClientStream.RecvMsg(m); err != nil {
        return nil, err
    }
    return m, nil
}

func (c *exportClient) ExportMetrics(ctx context.Context, opts ...grpc.CallOption) (Export_ExportMetricsClient, error) {
    stream, err := c.cc.NewStream(ctx, &_Export_serviceDesc.Streams[1], "/opencensus.proto.exporter.Export/ExportMetrics", opts...)
    if err != nil {
        return nil, err
    }
    x := &exportExportMetricsClient{stream}
    return x, nil
}

type Export_ExportMetricsClient interface {
    Send(*ExportMetricsRequest) error
    Recv() (*ExportMetricsResponse, error)
    grpc.ClientStream
}

type exportExportMetricsClient struct {
    grpc.ClientStream
}

func (x *exportExportMetricsClient) Send(m *ExportMetricsRequest) error {
    return x.ClientStream.SendMsg(m)
}

func (x *exportExportMetricsClient) Recv() (*ExportMetricsResponse, error) {
    m := new(ExportMetricsResponse)
    if err := x.ClientStream.RecvMsg(m); err != nil {
        return nil, err
    }
    return m, nil
}
```

- server 侧

```golang
// ExportServer is the server API for Export service.

// 1. rpc server 需要实现的接口
type ExportServer interface {
    ExportSpan(Export_ExportSpanServer) error
    ExportMetrics(Export_ExportMetricsServer) error
}

// 2. 将符合 rpc server 接口的结构注册到 grpc server 的函数
func RegisterExportServer(s *grpc.Server, srv ExportServer) {
    s.RegisterService(&_Export_serviceDesc, srv)
}

// 3. 符合 ExportServer 接口的 srv 在 stream 上调用 ExportSpan ，即需要自行实现的那部分代码
// 作为回调函数使用
func _Export_ExportSpan_Handler(srv interface{}, stream grpc.ServerStream) error {
    return srv.(ExportServer).ExportSpan(&exportExportSpanServer{stream})
}

// 4. 定义 server stream 需要实现的接口
type Export_ExportSpanServer interface {
    Send(*ExportSpanResponse) error
    Recv() (*ExportSpanRequest, error)
    grpc.ServerStream
}

// 5. 实现了上述接口的 server stream 结构
type exportExportSpanServer struct {
    grpc.ServerStream
}

// 6. server stream send (response)
func (x *exportExportSpanServer) Send(m *ExportSpanResponse) error {
    return x.ServerStream.SendMsg(m)
}

// 7. server stream recv (request)
func (x *exportExportSpanServer) Recv() (*ExportSpanRequest, error) {
    m := new(ExportSpanRequest)
    if err := x.ServerStream.RecvMsg(m); err != nil {
        return nil, err
    }
    return m, nil
}

// 8. 符合 ExportServer 接口的 srv 在 stream 上调用 ExportMetrics ，即需要自行实现的那部分代码
// 作为回调函数使用
func _Export_ExportMetrics_Handler(srv interface{}, stream grpc.ServerStream) error {
    return srv.(ExportServer).ExportMetrics(&exportExportMetricsServer{stream})
}

type Export_ExportMetricsServer interface {
    Send(*ExportMetricsResponse) error
    Recv() (*ExportMetricsRequest, error)
    grpc.ServerStream
}

type exportExportMetricsServer struct {
    grpc.ServerStream
}

func (x *exportExportMetricsServer) Send(m *ExportMetricsResponse) error {
    return x.ServerStream.SendMsg(m)
}

func (x *exportExportMetricsServer) Recv() (*ExportMetricsRequest, error) {
    m := new(ExportMetricsRequest)
    if err := x.ServerStream.RecvMsg(m); err != nil {
        return nil, err
    }
    return m, nil
}

// service 的完整定义
// rpc client 侧通过 _Export_serviceDesc.Streams[n] 进行 ExportSpan/ExportMetrics 调用
// rpc server 侧
var _Export_serviceDesc = grpc.ServiceDesc{
    // 对应 .proto 中的 package opencensus.proto.exporter; 和 service Export {}
    ServiceName: "opencensus.proto.exporter.Export",
    HandlerType: (*ExportServer)(nil),
    Methods:     []grpc.MethodDesc{},
    Streams: []grpc.StreamDesc{
        {
            // 对应 grpc 的通信类型
            // 对应 .proto 中 rpc ExportSpan(stream ...) returns (stream ...) {}
            StreamName:    "ExportSpan",
            Handler:       _Export_ExportSpan_Handler,
            ServerStreams: true,
            ClientStreams: true,
        },
        {
            StreamName:    "ExportMetrics",
            Handler:       _Export_ExportMetrics_Handler,
            ServerStreams: true,
            ClientStreams: true,
        },
    },
    Metadata: "opencensus/proto/exporter/exporter.proto",
}
```

