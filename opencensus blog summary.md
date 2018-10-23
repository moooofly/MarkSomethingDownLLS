# opencensus blog summary

## [“Hello, world!” for web servers in Go with OpenCensus](https://medium.com/@orijtech/hello-world-for-web-servers-in-go-with-opencensus-29955b3f02c6)

> Apr 18, 2018

## [OpenCensus’s journey ahead: platforms and languages](https://opensource.googleblog.com/2018/05/opencensus-journey-ahead-part-1.html)

> May 7, 2018

### Istio

OpenCensus will soon have out-of-the-box tracing and metrics collection in Istio. We’re currently working through our initial designs and implementation for integrations with the Envoy Sidecar and Istio Mixer service. Our goal is to provide Istio users with a great out of box tracing and metrics collection experience.

### Kubernetes

We have two primary use cases in mind for Kubernetes deployments: providing cluster-wide visibility via z-pages, and better labeling of traces, stats, and metrics. Cluster-wide z-pages will allow developers to view telemetry in real time across an entire Kubernetes deployment, independently of their back-end. This is incredibly useful when debugging immediate high-impact issues like service outages.

## [How Google uses Census internally](https://opensource.googleblog.com/2018/03/how-google-uses-opencensus-internally.html)

> March 7, 2018

Google adopted or invented new technologies, including distributed tracing (Dapper) and metrics processing, in order to operate some of the world’s largest web services. However, building analysis systems didn’t solve the difficult problem of instrumenting and extracting data from production services. This is what Census was created to do.

The Census project provides uniform instrumentation across most Google services, capturing trace spans, app-level metrics, and other metadata like log correlations from production applications. One of the biggest benefits of uniform instrumentation to developers inside of Google is that it’s almost entirely automatic: any service that uses gRPC automatically collects and exports basic traces and metrics.

### Incident Management

When latency problems or new errors crop up in a highly distributed environment, visibility into what’s happening is critical. For example, when the latency of a service crosses expected boundaries, we can view distributed traces in Dapper to find where things are slowing down. Or when a request is returning an error, we can look at the chain of calls that led to the error and examine the metadata captured during a trace (typically logs or trace annotations). This is effectively a bigger stack trace. In rare cases, we enable custom trigger-based sampling which allows us to focus on specific kinds of requests.

Once we know there’s a production issue, we can use Census data to determine the regions, services, and scope (one customer vs many) of a given problem. You can use service-specific diagnostics pages, called “z-pages,” to monitor problems and the results of solutions you deploy. These pages are hosted locally on each service and provide a firehose view of recent requests, stats, and other performance-related information.

### Performance Optimization

At Google’s scale, we need to be able to instrument and attribute costs for services. We use Census to help us answer questions like:

- How much CPU time does my query consume?
- Does my feature consume more storage resources than before?
- What is the cost of a particular user operation at a particular layer of the stack?
- What is the total cost of a particular user operation across all layers of the stack?

We’re obsessed with reducing the tail latency of all services, so we’ve built sophisticated analysis systems that process traces and metrics captured by Census to identify regressions and other anomalies.

### Quality of Service

Google also improves performance dynamically depending on the source and type of traffic. Using Census tags, traffic can be directed to more appropriate shards, or we can do things like load shedding and rate limiting.


## [The value of OpenCensus](https://opensource.googleblog.com/2018/03/the-value-of-opencensus.html)

> March 13, 2018

Google’s reasons for developing and promoting OpenCensus apply to partners at all levels.

Service developers reap the benefits of having automatic traces and stats collection, along with vendor-neutral APIs for manually interacting with these. Developers who use open source backends like Prometheus or Zipkin benefit from having a single set of well-supported instrumentation libraries that can export to both services at once.

For APM vendors, being able to take advantage of already-provided language support and framework integrations is huge, and the exporter API allows traces and metrics to be sent to an ingestion API without much additional work. Developers who might have been working on instrumentation code can now focus on other more important tasks, and vendors get traces and metrics back from places they previously didn’t have coverage for.

Cloud and API providers have the added benefit of being able to include OpenCensus in client libraries, allowing customers to gain insight into performance characteristics and debug issues without having to contact support. In situations where customers were still not able to diagnose their own issues, customer traces can be matched with internal traces for faster root cause analysis, regardless of which tracing or APM product they use.

## [OpenCensus with Prometheus and Kubernetes](https://kausal.co/blog/opencensus-prometheus-kausal/)

> Jan 18, 2018

This [example app](https://github.com/census-instrumentation/opencensus-go/blob/master/exporter/prometheus/example/main.go) shows how to take measurements, and then expose them to a `/metrics` endpoint for Prometheus.

Note that the concept of an “exporter” here is different to exporters in Prometheus land, where it usually means a separate process that translates between an applications native metrics interface and the prometheus exposition format, running as a sidecar or colocated in some other way with the application.

Before we can measure anything, the measure type needs to be registered. 

```
videoCount = stats.Int64("example.com/measures/video_count", "number of processed videos", stats.UnitDimensionless)
```

To record a single measurement we need to obtain a measure reference either by creating a new one, or by finding one via FindMeasure(name).

This increased the videoCount counter by 1. 

```
stats.Record(ctx, videoCount.M(1))
```

This recording would be ignored because we have not registered the measure as part of a view.

```
	if err = view.Register(
		&view.View{
			Name:        "video_count",
			Description: "number of videos processed over time",
			Measure:     videoCount,
			Aggregation: view.Count(),
		},
		...
	); err != nil {
		log.Fatalf("Cannot register the view: %v", err)
	}
```

After a view like the above uses the videoCount measure, its recordings will be kept, and by calling Subscribe() it makes its metrics available to the exporters.

```
xxx.Subscribe()
```

> NOTE: 此处，文章中的说明和实际示例代码中已经不一致；

New labels can be added via tags.

```
routeKey, err := tag.NewKey("route")
if err != nil {
    log.Fatal(err)
}

tagMap, err2 := tag.NewMap(ctx,
    tag.Insert(routeKey, "/myroute"),
)
if err2 != nil {
    log.Fatal(err2)
}
ctx = tag.NewContext(ctx, tagMap)
```

This augmented context is later used in the stats.Record() call where it passes on those tags as labels. 





## [OpenCensus: A Stats Collection and Distributed Tracing Framework](https://opensource.googleblog.com/2018/01/opencensus.html)

> January 17, 2018

OpenCensus, a vendor-neutral open source library for metric collection and tracing. OpenCensus is built to add minimal overhead and be deployed fleet wide, especially for microservice-based architectures.

OpenCensus is the open source version of Google’s Census library, written based on years of optimization experience. It aims to make the collection and submission of app metrics and traces easier for developers. It is a vendor neutral, single distribution of libraries that automatically collects traces and metrics from your app, displays them locally, and sends them to analysis tools. 

Developers can use this powerful, out-of-the box library to instrument microservices and send data to any supported backend. For an Application Performance Management (APM) vendor, OpenCensus provides free instrumentation coverage with minimal work, and affords customers a simple setup experience.