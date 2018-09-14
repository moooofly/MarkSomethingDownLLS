# Prometheus 知识梳理

- endpoint: 即 metrics endpoint ，每一个暴露 `/metrics` 的 HTTP 服务都可理解成 endpoint ；
- targets: 有多个 metrics endpoint 构成；通过 labels 进行标识（例如进行分组）；
- job: The job name is added as a label `job=<job_name>` to any timeseries. We can group some endpoints into one job, other endpoints into another job. job 是一个更高层的概念；
- labels: 可以用于 targets 分组；


----------


## [prometheus/prometheus](https://github.com/prometheus/prometheus)

一句话：The Prometheus monitoring system and time series database.

### 基于 binary 安装

从 https://prometheus.io/download/ 下载目标平台的最新版本，解压后即可运行

```
tar xvfz prometheus-*.tar.gz
cd prometheus-*
```

### 基于 Docker 安装

Ref: https://github.com/prometheus/prometheus#docker-images

```
docker run --rm --name prometheus -d -p 9090:9090 quay.io/prometheus/prometheus
```

Ref: https://github.com/prometheus/prometheus/blob/master/docs/installation.md#using-docker

```
docker run -p 9090:9090 prom/prometheus
```

可以通过 http://11.11.11.12:9090/ 确认运行成功；


小结：

- 镜像同时存在于 https://quay.io/repository/prometheus/prometheus 和 https://hub.docker.com/u/prom/
- 这里没有说明基于 docker 的方式下，如何定制化配置文件内容，详见 [docs/installation.md](https://github.com/prometheus/prometheus/blob/master/docs/installation.md) 中的说明；


### 配置 Prometheus 监控自身

> Ref: https://github.com/prometheus/prometheus/blob/master/docs/getting_started.md

Prometheus collects metrics from monitored targets by scraping `metrics` HTTP endpoints on these targets. Since Prometheus also exposes data in the same manner about itself, it can also scrape and monitor its own health.

Save the following basic Prometheus configuration as a file named `prometheus.yml`:

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
```

（基于 binary 方式）指定配置文件运行：

```
./prometheus --config.file=prometheus.yml
```

可以通过 `metrics` endpoint `localhost:9090/metrics` 确认 Prometheus 已经能够正确提供针对自身的 metrics 输出；

> 注意：
>
> - `localhost:9090/metrics` 暴露出的内容其实就是可用于 expression console 上进行查询的内容；
> - `# HELP xxx` 给出 metric 的简单描述
> - `# TYPE xxx` 给出 metric 的类型

### 基于 expression browser 进行 metrics 查看

为了使用 Prometheus 内置的 expression browser 只需访问 `http://11.11.11.12:9090/graph` ，并选择 "Graph" -> "Console" ；

从 `localhost:9090/metrics` 上得到的数据可知，Prometheus 导出的 metric 之一为 `prometheus_target_interval_length_seconds` ，即两次 target scrapes 之间实际的时间花费；在 expression console 中输入

```
prometheus_target_interval_length_seconds
```

执行后，会返回一组不同的 time series 以及相应的 latest value ，所有内容都具有相同的 metric name `prometheus_target_interval_length_seconds` ，同时具有不同的 labels ；这些 labels 用于指定了不同的 latency percentiles 以及 target group intervals ；

![prom_1](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom_1.png)

如果我们仅对 **99th percentile latencies** 感兴趣，可以使用如下 query 获取信息：

```
prometheus_target_interval_length_seconds{quantile="0.99"}
```

若想要计算返回的 time series 的总数，可以执行：

```
count(prometheus_target_interval_length_seconds)
```

完整的 query 命令详见：[docs/querying/basics.md](https://github.com/prometheus/prometheus/blob/master/docs/querying/basics.md)


除了使用 expression console 之外，还可以使用 graph expressions ：即访问 "Graph" 标签进行查看：

如下 expression 用于绘制 per-second rate of chunks being created in the self-scraped Prometheus ：

```
rate(prometheus_tsdb_head_chunks_created_total[1m])
```

### 运行一些更实际的 demo

The [Go client library](https://github.com/prometheus/client_golang) includes an example which **exports fictional RPC latencies for three services with different latency distributions**.


```
# Fetch the client library code and compile example.
git clone https://github.com/prometheus/client_golang.git
cd client_golang/examples/random
go get -d
go build

# Start 3 example targets in separate terminals:
./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

之后可以访问 http://11.11.11.12:8080/metrics, http://11.11.11.12:8081/metrics 和 http://11.11.11.12:8082/metrics ；

![prom_2](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom_2.png)

![prom_3](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom_3.png)

![prom_4](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom_4.png)

![prom_5](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/prom_5.png)

从上图中可以看出，prometheus 在进行 query 时会对所有 metric endpoints 的内容进行某种聚合；并在查询结果中结合了 `prometheus.yml` 配置文件中设置的内容，代码中定义的内容（如 `rpc_durations_seconds`/`rpc_durations_histogram_seconds` 和 `uniform`/`normal`/`exponential`）；

小结：

- 在 prometheus 的使用中，需要理解 summary/histogram 等概念，已经如何构建他们；
- 在 prometheus 的使用中，需要熟悉其使用中的常规步骤；

### 基于 rules 将采集的数据进一步聚合为新 time series

> Though not a problem in our example, queries that aggregate over thousands of time series can get slow when computed ad-hoc. To make this more efficient, Prometheus allows you to prerecord expressions into completely new persisted time series via configured recording rules.

- 当执行 query 时面对的数据量超过数以千计的 time series 时，这种 ad-hoc 计算的方式会变得很慢；
- 更加高效的方式为，通过事先设置 recording rules ，可以将基于 prerecord expressions 生成的数据保存到全新的持久化 time series 中；

> Let's say we are interested in recording the **per-second rate of example RPCs** (`rpc_durations_seconds_count`) **averaged over all instances** (but preserving the **job** and **service** dimensions) **as measured over a window of 5 minutes**. We could write this as:

```
avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

> To record the time series resulting from this expression into a new metric called `job_service:rpc_durations_seconds_count:avg_rate5m`, create a file with the following recording rule and save it as `prometheus.rules.yml`:

```
groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)
```

> To make Prometheus pick up this new rule, add a `rule_files` statement to the global configuration section in your `prometheus.yml`. The config should now look like this:

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s # Evaluate rules every 15 seconds.

  # Attach these extra labels to all timeseries collected by this Prometheus instance.
  external_labels:
    monitor: 'codelab-monitor'

rule_files:
  - 'prometheus.rules.yml'

scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```




## [prometheus/client_golang](https://github.com/prometheus/client_golang)

一句话:Prometheus instrumentation library for Go applications

### [Collector](https://github.com/prometheus/client_golang/blob/master/prometheus/collector.go#L27)

```golang
// Collector is the interface implemented by anything that can be used by
// Prometheus to collect metrics. A Collector has to be registered for
// collection.
...
type Collector interface {
	// Describe sends the super-set of all possible descriptors of metrics
	// collected by this Collector to the provided channel and returns once
	// the last descriptor has been sent.
	...
	Describe(chan<- *Desc)
	// Collect is called by the Prometheus registry when collecting
	// metrics.
	// ...
	Collect(chan<- Metric)
}
```

要点：

- 只要实现了 Collector interface 就能够被 Prometheus 收集 metrics ；
- Collector 需要在注册后才能被收集；


### [Registerer](https://github.com/prometheus/client_golang/blob/master/prometheus/registry.go#L91)


```golang
// Registerer is the interface for the part of a registry in charge of
// registering and unregistering.
// ...
type Registerer interface {
	// Register registers a new Collector to be included in metrics
	// collection.
	// ...
	Register(Collector) error
	// MustRegister works like Register but registers any number of
	// Collectors and panics upon the first registration that causes an
	// error.
	MustRegister(...Collector)
	// Unregister unregisters the Collector that equals the Collector passed
	// in as an argument.
	// ...
	Unregister(Collector) bool
}
```

要点：

- Registerer interface 作为 registry 实现的一部分使用；
- Registerer 负责注册和注销；

### [Gatherer](https://github.com/prometheus/client_golang/blob/master/prometheus/registry.go#L138)


```golang
// Gatherer is the interface for the part of a registry in charge of gathering
// the collected metrics into a number of MetricFamilies.
// ...
type Gatherer interface {
	// Gather calls the Collect method of the registered Collectors and then
	// gathers the collected metrics into a lexicographically sorted slice
	// of uniquely named MetricFamily protobufs.
	// ...
	Gather() ([]*dto.MetricFamily, error)
}
```

要点：

- Gatherer interface 作为 registry 实现的一部分使用；
- Gatherer 负责将已收集的 metrics 汇聚成多种 MetricFamilies ；
- Gatherer 通过调用已注册 Collectors 的 Collect 方法来汇聚已收集的 metrics ，并将其保存到按字典序排序过的、类型为 MetricFamily 的 slice 中；


### [Registry](https://github.com/prometheus/client_golang/blob/master/prometheus/registry.go#L253)

```golang
// Registry registers Prometheus collectors, collects their metrics, and gathers
// them into MetricFamilies for exposition. It implements both Registerer and
// Gatherer. The zero value is not usable. Create instances with NewRegistry or
// NewPedanticRegistry.
type Registry struct {
	mtx                   sync.RWMutex
	collectorsByID        map[uint64]Collector // ID is a hash of the descIDs.
	descIDs               map[uint64]struct{}
	dimHashesByName       map[string]uint64
	uncheckedCollectors   []Collector
	pedanticChecksEnabled bool
}
```

要点：

- Registry 实现了 collectors 的注册；
- Registry 负责收集已注册 collectors 的 metrics ；
- Registry 负责汇聚 metrics 到 MetricFamilies 以便展示；
- Registry 同时实现了 Registerer 和 Gatherer ；

