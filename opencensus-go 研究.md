# opencensus-go 研究

github: [opencensus-go](https://github.com/census-instrumentation/opencensus-go)

一句话：A stats collection and distributed tracing framework.

> OpenCensus Go is a Go implementation of OpenCensus, a toolkit for collecting application performance and behavior monitoring data. Currently it consists of three major components: **tags**, **stats**, and **tracing**.

- opencensus-go 能够针对应用性能进行数据收集，针对应用行为进行监控；
- opencensus-go 目前主要由以下三部分构成
    - tags
    - stats
    - tracing

## 安装

```
go get -u go.opencensus.io
```

注意：不是 `go get -u github.com/census-instrumentation/opencensus-go`，原因在于 package 的引用问题；

> The API of this project is still evolving. The use of vendoring or a dependency management tool is recommended.

官方建议使用依赖管理工具来避免代码变更导致的不兼容问题；

## 项目组件

目前已存在的、集成了 OpenCensus 的 RPC framework

- net/http  -- 重点关注
- gRPC  -- 重点关注
- database/sql
- Go kit
- Groupcache
- Caddy webserver
- MongoDB
- Redis gomodule/redigo
- Redis goredis/redis
- Memcache

使用上述 RPC framework 时，能够方便的生成 instrumentation data ；

目前已经存在的 Exporters

- [Prometheus](https://godoc.org/go.opencensus.io/exporter/prometheus) for stats
- [OpenZipkin](https://godoc.org/go.opencensus.io/exporter/zipkin) for traces
- [Stackdriver](https://godoc.org/contrib.go.opencensus.io/exporter/stackdriver) Monitoring for stats and Trace for traces
- [Jaeger](https://godoc.org/go.opencensus.io/exporter/jaeger) for traces
- [AWS X-Ray](https://github.com/census-ecosystem/opencensus-go-exporter-aws) for traces
- [Datadog](https://github.com/DataDog/opencensus-go-exporter-datadog) for stats and traces

## Overview

![](https://camo.githubusercontent.com/198c3be40c5fe727bf7d00877aacc6199d3eaf68/68747470733a2f2f692e696d6775722e636f6d2f636634456c48452e6a7067)

> In a **microservices** environment, a user request may go through multiple services until there is a response. OpenCensus allows you to instrument your services and collect diagnostics data all through your services end-to-end.

在微服务环境下的使用；

## Tags

> **Tags** represent propagated key-value pairs. They are **propagated** using `context.Context` **in the same process** or can be encoded to be transmitted **on the wire**. Usually, this will be handled by an integration plugin, e.g. `ocgrpc.ServerHandler` and `ocgrpc.ClientHandler` for gRPC.

- tags 代表被传递的 key/value 对；
- tags 可在同一个进程内部通过 `context.Context` 传递，也可以编码后跨进程传递；

> Package `tag` allows **adding** or **modifying** tags in the current context.
>
> ```
> ctx, err = tag.New(ctx,
>    tag.Insert(osKey, "macOS-10.12.5"),
>    tag.Upsert(userIDKey, "cde36753ed"),
> )
> if err != nil {
>     log.Fatal(err)
> }
> ```

可以针对 tag 进行添加和修改；

## Stats

> OpenCensus is a low-overhead framework even if instrumentation is always enabled. In order to be so, **it is optimized to make recording of data points fast and separate from the data aggregation**.
>
> OpenCensus **stats** collection happens in two stages:
>
> - Definition of **measures** and recording of data points
> - Definition of **views** and aggregation of the recorded data

stats 收集在两个 stage 上发生：

- 定义 measures 时：记录 data points 时（关键词 **record**）
- 定义 views 时：针对记录（数据）进行聚合时（关键词 **snapshot**）

### Recording

> **Measurements** are data points associated with a **measure**. **Recording** implicitly tags the set of Measurements with the tags from the provided context:
> ```
> stats.Record(ctx, videoSize.M(102478))
> ```

- **Measurements** 对应了与某个 measure 关联的 recorded data points 集合；
- **Recording** 对应了基于 tags 对 Measurements 进行的分组；

### Views

> **Views** are how Measures are aggregated. You can think of them as **queries** over the set of recorded data points (**measurements**).
>
> **Views** have two parts:
>
> - the **tags** to group by, and
> - the **aggregation type** used.
>
> Currently three types of aggregations are supported:
>
> - **CountAggregation** is used to count the number of times a sample was recorded.
> - **DistributionAggregation** is used to provide a histogram of the values of the samples.
> - **SumAggregation** is used to sum up all sample values.
>
> ```
> countAgg := view.Count()
> distAgg := view.Distribution(0, 1<<32, 2<<32, 3<<32)
> sumAgg := view.Sum()
> ```

- Views 对应了 Measures 被聚合的方式；可以认为 view 是针对 measurements 的 query 查询；
- Views 由两部分构成
    - 用于进行 group by 的 tags
    - 所使用的 aggregation 类型
- aggregations 类型
    - **CountAggregation** 用于统计样本被 record 的次数；
    - **DistributionAggregation** 用于提供样本数值分布的 histogram ；
    - **SumAggregation** 用于对所有样本值进行求和；


> Here we create a **view** with the **DistributionAggregation** over our **measure**.
>
> ```
> if err := view.Register(&view.View{
>     Name:        "example.com/video_size_distribution",
>     Description: "distribution of processed video size over time",
>     Measure:     videoSize,
>     Aggregation: view.Distribution(0, 1<<32, 2<<32, 3<<32),
> }); err != nil {
>     log.Fatalf("Failed to register view: %v", err)
> }
> ```
>
> **Register** (`view.Register`) begins collecting data for the view. Registered views' data will be exported via the registered exporters.

通过 `view.Register` 进行 view 注册后，将触发针对该 view 的数据收集；与注册 view 相关的数据会通过已注册的 exporter 导出到外部；

## Traces

> A **distributed trace** tracks the progression of a single user request as it is handled by the services and processes that make up an application. Each step is called a **span** in the **trace**. Spans include **metadata** about the step, including especially the time spent in the step, called the **span’s latency**.

- trace 由 span 构成，用于跟踪指定用户请求的完整处理过程；
- span 中会包含关于当前 step 的各种 metadata 数据；其中 span 产生的 latency 是我们重点关注的；

### Spans

> **Span** is the unit step in a **trace**. Each span has a **name**, **latency**, **status** and **additional metadata**.
>
> Below we are starting a span for a cache read and ending it when we are done:
>
> ```
> ctx, span := trace.StartSpan(ctx, "cache.Get")
> defer span.End()
>
> // Do work to get from cache.
> ```

每个 span 都具有以下属性：

- name
- latency
- status
- additional metadata


### Propagation

> Spans can have **parents** or can be **root spans** if they don't have any parents. The current span is propagated **in-process** and **across the network** to allow associating new child spans with the parent.
>
> **In the same process**, `context.Context` is used to propagate spans. `trace.StartSpan` creates a new span as a root if the current context doesn't contain a span. Or, it creates a child of the span that is already in current context. The returned context can be used to keep propagating the newly created span in the current context.
>
> **Across the network**, OpenCensus provides different propagation methods for different protocols.
>
> - **gRPC integrations** uses the **OpenCensus**' [binary propagation format](https://godoc.org/go.opencensus.io/trace/propagation).
> - **HTTP integrations** uses **Zipkin**'s [B3](https://github.com/openzipkin/b3-propagation) by default but can be configured to use a custom propagation method by setting another [propagation.HTTPFormat](https://godoc.org/go.opencensus.io/trace/propagation#HTTPFormat).

- Spans 具有父子关系；
- 在同一个进程内，通过使用 `context.Context` 进行 span 信息的 propagate ；
- 在跨网络传递时，OpenCensus 提供了不同的 propagation 方法：
    - **gRPC integrations** 使用的是 **OpenCensus** 自己实现的 [binary propagation format](https://godoc.org/go.opencensus.io/trace/propagation) ；
    - **HTTP integrations** 默认使用的是 **Zipkin** 的 [B3](https://github.com/openzipkin/b3-propagation) ，但可以通过设置 [propagation.HTTPFormat](https://godoc.org/go.opencensus.io/trace/propagation#HTTPFormat) 配置成使用自定义 propagation 方法 ；


----------


## 代码阅读


在 `stats/measure.go` 中有

> **Measure** represents a single numeric value to be tracked and recorded. For example, latency, request bytes, and response bytes could be measures to collect from a server.
>
> Measures by themselves have no outside effects. **In order to be exported, the measure needs to be used in a View**. If no Views are defined over a measure, there is very little cost in recording it.

> **Measurement** is the numeric value measured when recording stats. Each measure provides methods to create measurements of their kind. For example, `Int64Measure` provides `M` to convert an `int64` into a measurement.

在 `stats/view/view.go` 中有

> **View** allows users to aggregate the recorded `stats.Measurements`. **Views need to be passed to the Register function to be before data will be collected and sent to Exporters**.

在 `trace/basetypes.go` 中有

> **Annotation** represents a text annotation with a set of attributes and a timestamp.

> **Attribute** represents a key-value pair on a span, link or annotation. Construct with one of: BoolAttribute, Int64Attribute, or StringAttribute.

> **Link** represents a reference from one span to another span.


在 `trace/trace.go` 中有

> **Span** represents a span of a trace.  It has an associated SpanContext, and stores data accumulated while the span is active.

> **SpanData** representing the current state of the Span.

