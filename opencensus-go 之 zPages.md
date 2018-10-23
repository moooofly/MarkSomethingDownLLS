# opencensus 之 zPages

## zPages

> Ref: https://opencensus.io/core-concepts/z-pages/

OpenCensus provides in-process web pages that displays collected data from the process. These pages are called `zPages` and they are useful to see collected data from a specific process without having to depend on any metric collection or distributed tracing backend.

`zPages` can be useful during the development time or when the process to be inspected is known in production. `zPages` can also be used to debug exporter issues.

In order to serve `zPages`, register their handlers and start a web server. Below, there is an example how to serve these pages from 127.0.0.1:7777.

```golang
package main

import (
    "log"
    "net/http"
    "go.opencensus.io/zpages"
)

func main() {
    // Using the default serve mux, but you can create your own
    mux := http.DefaultServeMux
    zpages.Handle(mux, "/")
    log.Fatal(http.ListenAndServe("127.0.0.1:7777", mux))
}
```

Once handler is registered, there are various pages provided from the libraries:

- 127.0.0.1:7777/rpcz
- 127.0.0.1:7777/tracez

### rpcz

`/rpcz` serves stats about sent and received RPCs.

Available stats include:

- Number of RPCs made per minute, hour and in total.
- Average latency in the last minute, hour and since the process started.
- RPCs per second in the last minute, hour and since the process started.
- Input payload in KB/s in the last minute, hour and since the process started.
- Output payload in KB/s in the last minute, hour and since the process started.
- Number of RPC errors in the last minute, hour and in total.

### tracez

`/tracez` serves details about the trace spans collected in the process. It provides several sample spans per latency bucket and sample errored spans.

## code review

> TODO
