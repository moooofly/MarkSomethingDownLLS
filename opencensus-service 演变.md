# OpenCensus Agent

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

### Packages

> - `common` package contains the common messages shared between different services, such as `Node`, `Service` and `Library` identifiers.
> - `trace` package contains the Trace Service protos.
> - `metrics` package contains the Metrics Service protos.
> - (Coming soon) `stats` package contains the Stats Service protos.


## [OpenCensus Agent](https://github.com/census-instrumentation/opencensus-service#opencensus-agent)

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

old:

![ocagent-architecture-old](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ocagent-architecture-old.png)

new:

![ocagent-architecture-new](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ocagent-architecture-new.png)

> To support `Agent`, `Library` should have “**agent exporters**”, similar to the existing exporters to other backends. There should be 3 separate agent exporters for tracing/stats/metrics respectively. Agent exporters will be responsible for sending spans/stats/metrics and (possibly) receiving configuration updates from `Agent`.

这里解释了 Agent 和 Agent exporter 的关系，以及 Agent exporter 需要支持的功能；

### Communication

> Communication between `Library` and `Agent` should user a **bi-directional gRPC stream**. `Library` should initiate the connection, since there’s only one dedicated port for `Agent`, while there could be multiple processes with `Library` running. By default, `Agent` is available on port **55678**.

- 双向 gRPC stream ；
- `Library` 侧发起连接建立；
- `Agent` 监听在 55678 端口；

### Protocol Workflow

> - `Library` will try to **directly establish** connections for **Config** and **Export** streams.
> - As the **first message** in each stream, `Library` must sent its identifier. Each identifier should uniquely identify `Library` within the VM/container. If there is no identifier in the first message, Agent should drop the whole message and return an error to the client. In addition, the first message MAY contain additional data (such as `Span`s). As long as it has a valid identifier assoicated, Agent should handle the data properly, as if they were sent in a subsequent message. Identifier is no longer needed once the streams are established.
> - If streams were disconnected and retries failed, the `Library` identifier would be considered expired on `Agent` side. `Library` needs to start a new connection with a unique identifier (MAY be different than the previous one).

### Implementation details of Agent Server

> This section describes the in-process implementation details of OC-Agent.

![ocagent-in-process-details](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ocagent-in-process-details.png)

> Note: Red arrows represent RPCs or HTTP requests. Black arrows represent local method invocations.

> The Agent consists of three main parts:
> 
> - **The interceptors of different instrumentation libraries**, such as **OpenCensus**, **Zipkin**, **Istio Mixer**, **Prometheus client**, etc. Interceptors act as the “frontend” or “gateway” of Agent. In addition, there MAY be one special receiver for receiving configuration updates from outside.
> - **The core Agent module**. It acts as the “brain” or “dispatcher” of Agent.
> - **The exporters to different monitoring backends or collector services**, such as **Omnition Collector**, **Stackdriver Trace**, **Jaeger**, **Zipkin**, etc.

Agent 由三部分构成：

- interceptors
- core module
- exporters

#### Interceptors

> Each interceptor can be connected with multiple instrumentation libraries. The communication protocol between interceptors and libraries is the one we described in the proto files (for example trace_service.proto). When a library opens the connection with the corresponding interceptor, the first message it sends must have the Node identifier. The interceptor will then cache the Node for each library, and Node is not required for the subsequent messages from libraries.

- 每个 interceptor 可以和多个 instrumentation libraries 建立连接；
- Interceptors 和 libraries 之间的通信协议由 proto 文件定义（例如 trace_service.proto）；
- 当一个 library 打开了一个和某个 interceptor 对应的连接，第一条消息必须带有 Node identifier ；之后 interceptor 会为缓存 Node 信息以便所有 library 可用，因此来自 libraries 的后续消息将不在需要 Node 信息了；

#### Agent Core

> Most functionalities of Agent are in Agent Core. Agent Core's responsibilies include:

> - **Accept `SpanProto` from each interceptor**. Note that the `SpanProto`s that are sent to Agent Core must have Node associated, so that Agent Core can differentiate and group SpanProtos by each Node.
> - **Store and batch `SpanProto`s**.
> - **Augment the `SpanProto` or `Node` sent from the interceptor**. For example, in a Kubernetes container, Agent Core can detect the namespace, pod id and container name and then add them to its record of Node from interceptor.
> - For some configured period of time, Agent Core will **push `SpanProto`s (grouped by Nodes) to Exporters**.
> - **Display the currently stored `SpanProto`s on local zPages**.
> - **MAY accept the updated configuration from Config Receiver, and apply it to all the config service clients**.
> - **MAY track the status of all the connections of Config streams**. Depending on the language and implementation of the Config service protocol, Agent Core MAY either store a list of active Config streams (e.g gRPC-Java), or a list of last active time for streams that cannot be kept alive all the time (e.g gRPC-Python).


#### Exporters

> Once in a while, Agent Core will push `SpanProto` with `Node` to each exporter. After receiving them, each exporter will translate `SpanProto` to the format supported by the backend (e.g Jaeger Thrift Span), and then push them to corresponding backend or service.

### Usage

- Install `ocagent` if you haven't.

```
$ go get github.com/census-instrumentation/opencensus-service/cmd/ocagent
```

### 配置文件

Create a `config.yaml` file in the current directory and modify it with the exporter and interceptor configuration. 

#### Exporters

For example, following configuration exports both to Stackdriver and Zipkin.

```
# config.yaml

stackdriver:
  project: "your-project-id"
  enableTraces: true

zipkin:
  endpoint: "http://localhost:9411/api/v2/spans"
```

#### Interceptors

To modify the address that the OpenCensus interceptor runs on, please use the YAML field name `opencensus_interceptor` and it takes fields like address. For example:

```
opencensus_interceptor:
    address: "localhost:55678"
```

### Running an end-to-end example/demo

Run the example application that **collects** traces and **exports** to the daemon if it is running.

- 首先运行 `ocagent`

```
$ ocagent
```

- 之后运行 demo 应用

```
$ go run "$(go env GOPATH)/src/github.com/census-instrumentation/opencensus-service/example/main.go"
```

You should be able to see the traces in `Stackdriver` and `Zipkin`. If you stop the `ocagent`, example application will stop exporting. If you run it again, exporting will resume.


## [OpenCensus Collector](https://github.com/census-instrumentation/opencensus-service#opencensus-collector)

> The OpenCensus Collector is a component that runs “nearby” (e.g. in the same VPC, AZ, etc.) a user’s application components and receives trace spans and metrics emitted by the OpenCensus Agent or tasks instrumented with OpenCensus instrumentation (or other supported protocols/libraries). The received spans and metrics could be emitted directly by clients in instrumented tasks, or potentially routed via intermediate proxy sidecar/daemon agents (such as the OpenCensus Agent). **The collector provides a central egress point for exporting traces and metrics to one or more tracing and metrics backends, with buffering and retries as well as advanced aggregation, filtering and annotation capabilities**.

collector 提供了中央 egress 出口，用于导出 traces 和 metrics 到一个或多个 backends ；collector 提供了缓冲功能，提供了失败重试功能，提供了聚合、过滤，以及 annotation 等高级能力；

> The collector is extensible enabling it to support a range of out-of-the-box (and custom) capabilities such as:
> 
> - Retroactive (tail-based) sampling of traces 抽样率反制调整方法
> - Cluster-wide z-pages
> - Filtering of traces and metrics 过滤
> - Aggregation of traces and metrics 聚合
> - Decoration with meta-data from infrastructure provider (e.g. k8s master) 额外信息添加
> - much more ...

> The collector also serves as a control plane for agents/clients by supplying them updated configuration (e.g. trace sampling policies), and reporting agent/client health information/inventory metadata to downstream exporters.

collector 还可以作为控制面，为 agents/clients 提供配置更新功能，提供健康状态报告元数据给下游的 exporters ；


### Architecture Overview

> The OpenCensus Collector runs as a standalone instance and receives spans and metrics exporterd by one or more OpenCensus Agents or Libraries, or by tasks/agents that emit in one of the supported protocols. The Collector is configured to send data to the configured exporter(s). The following figure summarizes the deployment architecture:

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/collector-architecture.png)

> The OpenCensus Collector can also be deployed in other configurations, such as receiving data from other agents or clients in one of the formats supported by its interceptors.

### Usage

- install the collector if you haven't

```
$ go get github.com/census-instrumentation/opencensus-service/cmd/occollector
```

Create a `config.yaml` file in the current directory and modify it with the collector exporter configuration.

```
omnition:
  tenant: "your-api-key"
```

- install ocagent if you haven't

```
$ go get github.com/census-instrumentation/opencensus-service/cmd/ocagent
```

Create a `config.yaml` file in the current directory and modify it with the collector exporter configuration.

```
collector:
  endpoint: "https://collector.local"
```

- Run the example application that collects traces and exports to the daemon if it is running

```
$ go run "$(go env GOPATH)/src/github.com/census-instrumentation/opencensus-service/example/main.go"
```

- Run ocagent

```
$ ocagent
```

You should be able to see the traces in the configured tracing backend. If you stop the ocagent, example application will stop exporting. If you run it again, it will start exporting again.

