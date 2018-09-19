# opencensus-service 演变

## 背景

近日发现 opencensus-service 项目有了[一些新的动作](https://github.com/census-instrumentation/opencensus-service/issues?utf8=%E2%9C%93&q=is%3Aissue+agent)，主要涉及：

- [将实验性项目 opencensus-service 重命名为 opencensus-agent](https://github.com/census-instrumentation/opencensus-service/issues/22) ；
- 实现一个 [Go exporter](https://github.com/census-ecosystem/opencensus-go-exporter-ocagent) 用于和 opencensus-agent 交互：
    - Proto (agent): [opencensus/proto/agent](https://github.com/census-instrumentation/opencensus-proto/tree/master/src/opencensus/proto/agent)
    - Go exporter (agent): [opencensus-go-exporter-ocagent](https://github.com/census-ecosystem/opencensus-go-exporter-ocagent)
- agent 需要满足：
    - [**trace/interceptor**] The agent needs to be able to pull or receive traces from various sources (Zipkin/Jaeger), and then transform them into the OpenCensus Proto-defined Spans for later consumption - [issues#19](https://github.com/census-instrumentation/opencensus-service/issues/19)
    - [**metrics/interceptor**] The agent will be able to pull metrics from sources (Prometheus), and transform them into OpenCensus Metrics which can then be consumed/exported to the various backends - [issues#20](https://github.com/census-instrumentation/opencensus-service/issues/20)
    - agent architecture diagram: [here](https://github.com/census-instrumentation/opencensus-proto/blob/master/src/opencensus/proto/agent/agent-architecture.png)
    - [OpenCensus Agent Proto](https://github.com/census-instrumentation/opencensus-proto/blob/master/src/opencensus/proto/agent/README.md) - an overview on how OpenCensus Agent works with Library, Collector/Service and Monitoring backend. However, in the overview section Agent is treated as a blackbox and many implementation details are omitted.

## [opencensus-go-exporter-ocagent](https://github.com/census-ecosystem/opencensus-go-exporter-ocagent)

> This repository contains the Go implementation of the **OpenCensus Agent (OC-Agent) Exporter**. **OC-Agent** is a deamon process running in a VM that can retrieve spans/stats/metrics from OpenCensus Library, export them to other backends and possibly push configurations back to Library.

> Note: This is an experimental repository and is likely to get backwards-incompatible changes. Ultimately we may want to move the OC-Agent Go Exporter to OpenCensus Go core library.

- OC-Agent 就是之前的 opencensus-service ，后续会变为 opencensus-agent ；
- OC-Agent Go Exporter 最后会合并回 [opencensus-go/exporter](https://github.com/census-instrumentation/opencensus-go/tree/master/exporter) 下；


## [OpenCensus Agent Proto](https://github.com/census-instrumentation/opencensus-proto/blob/master/src/opencensus/proto/agent/README.md)

> This package describes the **OpenCensus Agent protocol**.

### Architecture Overview

> On a typical **VM/container**, there are user applications running in some processes/pods with **OpenCensus Library** (`Library`). Previously, `Library` did all the recording, collecting, sampling and aggregation on spans/stats/metrics, and exported them to other persistent storage backends via the **`Library` exporters**, or displayed them on local **zpages**. This pattern has **several drawbacks**, for example:
>
> - For each OpenCensus `Library`, exporters/zpages need to be **re-implemented in native languages**.
> - In some programming languages (e.g `Ruby`, `PHP`), it is **difficult to do the stats aggregation in process**.
> - To enable exporting OpenCensus spans/stats/metrics, application users **need to manually add library exporters and redeploy their binaries**. This is especially difficult when there’s already an incident and users want to use OpenCensus to investigate what’s going on right away.
> - Application users **need to take the responsibility in configuring and initializing exporters**. This is error-prone (e.g they may not set up the correct credentials\monitored resources), and users may be reluctant to “pollute” their code with OpenCensus.

基于 library 模式进行 instrument 时面临的问题；

> To resolve the issues above, we are introducing **OpenCensus Agent** (`Agent`). `Agent` runs as a daemon in the VM/container and can be **deployed independent of `Library`**. Once `Agent` is deployed and running, it should **be able to retrieve** spans/stats/metrics **from `Library`**, export them to other backends. We MAY also give `Agent` the ability to push configurations (e.g sampling probability) to `Library`. For those languages that cannot do stats aggregation in process, they should also be able to send raw measurements and have `Agent` do the aggregation.

基于 OpenCensus Agent 的实现（可以做到的东西）：

- `Agent` 在部署后，即可立即从 `Library` 获取 spans/stats/metrics 数据，并 export 到相应的各种 backends ；
- 可以通过 `Agent` 反向推送配置给 `Library` ；
- 针对不方便实现 stats aggregation 的语言，可以在 `Agent` 中完成；

> For developers/maintainers of other libraries: Agent can also be extended to accept spans/stats/metrics from other tracing/monitoring libraries, such as **Zipkin**, **Prometheus**, etc. This is done by adding specific **interceptors**.

对于其他类型的（非官方支持） `Library` ，可以通过对 `Agent` 进行扩展，进而处理相应的数据；这部分功能可以基于特定的 interceptors 实现来完成；

![](https://raw.githubusercontent.com/census-instrumentation/opencensus-proto/master/src/opencensus/proto/agent/agent-architecture.png)

> To support `Agent`, `Library` should have “**agent exporters**”, similar to the existing exporters to other backends. There should be 3 separate agent exporters for tracing/stats/metrics respectively. Agent exporters will be responsible for sending spans/stats/metrics and (possibly) receiving configuration updates from `Agent`.

这里解释了 Agent 和 Agent exporter 的关系，以及 Agent exporter 需要支持的功能；

### Communication

> Communication between `Library` and `Agent` should user a **bi-directional gRPC stream**. `Library` should initiate the connection, since there’s only one dedicated port for `Agent`, while there could be multiple processes with `Library` running. By default, `Agent` is available on port **55678**.

- 双向 gRPC stream ；
- `Library` 侧发起连接建立；
- `Agent` 监听在 55678 端口；

### Protocol Workflow

> - `Library` will try to **directly establish** connections for **Config** and **Export** streams.
> - As the **first message** in each stream, `Library` must sent its identifier. Each identifier should uniquely identify `Library` within the VM/container. Identifier is no longer needed once the streams are established.
> - If streams were disconnected and retries failed, the `Library` identifier would be considered expired on `Agent` side. `Library` needs to start a new connection with a unique identifier (MAY be different than the previous one).

### Packages

> - `common` package contains the common messages shared between different services, such as `Node`, `Service` and `Library` identifiers.
> - `trace` package contains the Trace Service protos.
> - (Coming soon) `stats` package contains the Stats Service protos.
> - (Coming soon) `metrics` package contains the Metrics Service protos.