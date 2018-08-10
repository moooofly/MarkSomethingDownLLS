# Hunter 项目介绍

项目主页：https://phab.llsapp.com/w/engineer/ops/hunter/

## 项目价值

- 耗时问题
- 服务依赖关系展示
- 组件失败展示，失败原因展示
- opencensus 和 opentracing 相比，标准定义更详细，约束更明确；通过 opencensus 的 stats 可以统一化 metric 的上报格式问题；

## 系统架构

Ver1 结构图

![](https://phab.llsapp.com/file/data/n5oxrxqonv7rpub2br6d/PHID-FILE-6clrcshyobcewttn5d55/image.png)

Ver2 结构图

![](https://phab.llsapp.com/file/data/ik4gzber3dcnjocwxdyo/PHID-FILE-c5vbf5fctmpaeqlczjwb/image.png)

## 技术栈

- [opencensus](https://opencensus.io/)
    - [opencensus-go](https://github.com/census-instrumentation/opencensus-go)
    - [opencensus-ruby](https://github.com/census-instrumentation/opencensus-ruby)
    - opencensus [exporter](https://github.com/census-instrumentation/opencensus-go/tree/master/exporter) -- 是 opencensus-go 的一部分，目前不支持对接 kafka ，**需要自行实现**（moooofly）；
    - [go-kit/tracing/opencensus](https://github.com/go-kit/kit/tree/master/tracing/opencensus)  -- Go kit uses the opencensus-go implementation to power its middlewares. -- 用于学习参考；
- kafka
- spark
    - spark streaming -- 从 kafka 消费 opencensus 数据，进行流式处理后，再写入 kafka ；**需要自行实现**；
    - spark job -- 应该是类似 Map-Reduce 的东西，从 cassandra 中获取当天的数据计算服务之间的拓扑关系，以便 jaeger ui 展示需要；**需要自行实现**；
- [vizceral](https://github.com/Netflix/vizceral) -- 能够从全局的角度展示服务之间的拓扑关系以及流量和错误率等信息；
    - vizceral collector -- **需要自行实现**，从 kafka 获取输出，转换成 vizceral 需要的格式后，提供给 vizceral ；
- hunter adapter：从 kafka 消费 opencensus 数据，转换成 jaeger collector 需要的数据格式，输出到 jaeger collector ；**需要自行实现**；
- jaeger -- jaeger clients are language specific implementations of the OpenTracing API.
    - [jaeger collector](https://github.com/jaegertracing/jaeger/tree/master/cmd/collector): 从 hunter adapter 处获取 opencensus 数据，对数据进行加工处理，之后存储到 cassandra 中；**需要自行实现**；
    - [jaeger-client-go](https://github.com/jaegertracing/jaeger-client-go) -- 应该会用到；
    - [jaeger query](https://github.com/jaegertracing/jaeger/tree/master/cmd/query) -- 从 cassandra 查询数据，供 jaeger-ui 展示使用；
    - [jaeger-ui](https://github.com/jaegertracing/jaeger-ui)
- [cassandra](https://github.com/apache/cassandra)

> 注意：v2 中已经将 hunter adapter 和 jaeger collector 合并为 hunter collector ；

## 关键点

- 第一阶段只考虑 trace ，不考虑 stats 问题；
- 针对 exporter 的实现，需要考虑
    - 针对数据库的 query 和 update 操作进行 wrap ；
    - 针对那些操作不需要 wrap ；
- 涉及到的 trace 点
    - grpc -- 重点
    - http -- 次重点
    - redis
    - mysql


## Related

- [opencensus-go](https://github.com/census-instrumentation/opencensus-go)
- [opencensus-specs](https://github.com/census-instrumentation/opencensus-specs) -- 标准文档
- [Shopify/sarama](https://github.com/Shopify/sarama)  -- 业务组使用的 kafka 库；
- [elastic/beats](https://github.com/elastic/beats/) -- 包含对 Shopify/sarama 的使用，参考使用；


## protobuf

可以对比看

- https://github.com/census-instrumentation/opencensus-proto/blob/master/opencensus/proto/trace/trace.proto
- https://developers.google.com/protocol-buffers/docs/reference/proto3-spec

