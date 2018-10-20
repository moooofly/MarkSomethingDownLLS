# opencensus-service 研究

## [opencensus-service](https://github.com/census-instrumentation/opencensus-service)

> OpenCensus service is an **experimental component** that collects and exports from the **OpenCensus instrumented processes** available from **the same host machine**.

OpenCensus service 目前作为一个实验性组件提供；用于 collect 并 export 位于同一台主机上的、被 OpenCensus 所 instrument 的进程；

> Some frameworks and ecosystems are now providing out-of-the-box instrumentation by using OpenCensus but the user is still expected to register an exporter in order to export data. **This is a problem during an incident**. Even though our users can benefit from having more diagnostics data coming out of services already instrumented with OpenCensus, they have to modify their code to register an exporter and redeploy. Asking our users recompile and redeploy is not an ideal at an incident time.

> In addition, currently users need to decide which service backend they want to export to, before they distribute their binary instrumented by OpenCensus.

虽然在一些 frameworks 和 ecosystems 中都提供了开箱即用的 instrumentation 能力，以方便 OpenCensus 的使用，但某些场景下，用户仍会希望自行注册一个 exporter 来进行数据导出，例如发生事故时；

基于 frameworks 的使用方式要求用户修改代码以便注册 exporter ，并且还需要重新进行部署；在某些情况下，这些要求都是很难实现的；

> OpenCensus service is trying to eliminate these requirements.

> With OpenCensus Service, users do not need to redeploy or restart their applications as long as it has the OpenCensus Agent exporter. All they need to do is just configure and deploy OpenCensus Service separately. OpenCensus Service will then automatically collect traces and metrics and export to any backend of users' choice.

OpenCensus service 正是用于消除上述约束；在使用 OpenCensus Service 后，只要使用了 OpenCensus Agent exporter ，则用户不再需要重新部署或重启他们的应用程序；用户唯一需要做的只是单独配置和部署 OpenCensus Service 而已；之后 OpenCensus Service 就会自动搜集 traces 和 metrics 信息，并导出到用户所选的任何 backend ；


> Currently OpenCensus Service consists of two components, [OpenCensus Agent](https://github.com/census-instrumentation/opencensus-service#opencensus-agent) and [OpenCensus Collector](https://github.com/census-instrumentation/opencensus-service#opencensus-collector).

> Goals
>
> - **Allow enabling of configuring the exporters lazily**. After deploying code, optionally run a daemon on the host and it will read the collected data and upload to the configured backend.
> - **Binaries can be instrumented without thinking about the exporting story**. Allows open source binary projects (e.g. web servers like Caddy or Istio Mixer) to adopt OpenCensus without having to link any exporters into their binary.
> - **Easier to scale the exporter development**. Not every language has to implement support for each backend.
> - **Custom daemons containing only the required exporters compiled in can be created**.

目标：

- 允许先部署应用，后配置 exporters ；即懒惰模式；
- 可以针对开源 binary 项目进行 instrument 动作，无需额外的处理；
- 针对 exporters 种类的扩展更容易；
- 基于 daemon 模式的实现定制化程度更高；


### OpenCensus Agent

详见：[opencensus-agent](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/OpenCensus%20Agent.md#opencensus-agent-1)

### OpenCensus Collector

详见：[opencensus-collector](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/OpenCensus%20Agent.md#opencensus-collector)


----------

## 实际测试

- server

```sh
[#301#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   cmd/ocagent/config.yaml

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	stackdriver-moooofly-fa114f37b0b2.json

no changes added to commit (use "git add" and/or "git commit -a")
[#302#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$
[#302#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$cat cmd/ocagent/config.yaml
interceptors:
    opencensus:
        address: "127.0.0.1:55678"

exporters:
    stackdriver:
        project: "stackdriver-moooofly"
        enable_tracing: true

    zipkin:
        endpoint: "http://localhost:9411/api/v2/spans"

zpages:
    port: 55679
[#303#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$
[#303#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$
[#303#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$rm bin/ocagent_linux && GOOS=linux go build -o bin/ocagent_linux ./cmd/ocagent
[#304#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$
[#304#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$GOOGLE_APPLICATION_CREDENTIALS=./stackdriver-moooofly-fa114f37b0b2.json ./bin/ocagent_linux -c cmd/ocagent/config.yaml
2018/10/19 15:06:54 "stackdriver" trace-exporter enabled
2018/10/19 15:06:54 "zipkin" trace-exporter enabled
2018/10/19 15:06:54 Running OpenCensus interceptor as a gRPC service at "127.0.0.1:55678"
2018/10/19 15:06:54 Running zPages at ":55679"
```

- client

```
[#190#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service]$go run "$(go env GOPATH)/src/github.com/census-instrumentation/opencensus-service/example/main.go"
```

### 异常信息记录

- client 运行过程中，停止 server

```
[#62#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$OPENCENSUS_DEBUG=1 go run "$(go env GOPATH)/src/github.com/census-instrumentation/opencensus-service/example/main.go"
Looking for the endpoint file
Dialing [127.0.0.1:35246]...
Sleeping for a minute...
Exporting span [[{025b40c1b53f08fe05afdeddc5a56d01 f4f1dae4764f558f 1}]]
Exporting span [[{bb7204400393128944128501b63d37e2 d5273036e7eac840 1}]]
Exporting span [[{fc88f1460f2c5e788358b0a7c44e3208 b65d858757863cf2 1}]]
Exporting span [[{116faf98f7796b821ae2c606a66d305d 9793dad8c721b0a3 1}]]
Exporting span [[{2644d80a49d8af006fd97ed64a0945fe 78c92f2a38bd2355 1}]]
Exporting span [[{6f43da28896ee28ae333918af8bea7c0 59ff847ba8589706 1}]]
Exporting span [[{39dd362d56dff9e3e8ee3c6c955774ad 3a35dacc18f40ab8 1}]]
...
Exporting span [[{a6045ad712409cca0f74e76d757895e7 695dd890ad0fd11d 1}]]
Connection is unavailable; will try to reconnect in a minute


Looking for the endpoint file
Dialing [127.0.0.1:41577]...
Sleeping for a minute...
Exporting span [[{a97256666ec296e6882789364288cb19 06f6919c13fa934d 1}]]
Exporting span [[{01e212fa9b1ef41bceee7ab88eea280b e72be7ed839507ff 1}]]
```

- client 和 server 正常运行过程中，停止 zipkin（此时 client 侧仍旧输出正常的打印）

```
[#126#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$opencensusd
2018/07/30 20:17:59 endpointFile path: /root/.config/opencensus.endpoint
2018/07/30 20:19:12 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:13 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:14 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:15 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:16 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:17 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:18 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:19 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:20 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:21 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:21 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
...
```

- 重启 zipkin

```
...
2018/07/30 20:19:34 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:35 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:35 failed to send the request: Post http://localhost:9411/api/v2/spans: dial tcp 127.0.0.1:9411: connect: connection refused
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38088->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38096->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38100->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38108->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: net/http: HTTP/1.x transport connection broken: write tcp 127.0.0.1:38120->127.0.0.1:9411: write: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: net/http: HTTP/1.x transport connection broken: write tcp 127.0.0.1:38140->127.0.0.1:9411: write: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: net/http: HTTP/1.x transport connection broken: write tcp 127.0.0.1:38148->127.0.0.1:9411: write: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: net/http: HTTP/1.x transport connection broken: write tcp 127.0.0.1:38160->127.0.0.1:9411: write: broken pipe
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38168->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38172->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38176->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:38 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
...
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38520->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38524->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: read tcp 127.0.0.1:38532->127.0.0.1:9411: read: connection reset by peer
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: net/http: HTTP/1.x transport connection broken: write tcp 127.0.0.1:38540->127.0.0.1:9411: write: connection reset by peer
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: http: server closed idle connection
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
2018/07/30 20:19:46 failed to send the request: Post http://localhost:9411/api/v2/spans: EOF
（卡住）
```


----------


### Zipkin 启动

> Ref: https://github.com/openzipkin/zipkin#quick-start

![](https://zipkin.io/public/img/architecture-1.png)

- You can also start Zipkin via Docker.

```
docker run -d -p 9411:9411 openzipkin/zipkin
```

- Once the server is running, you can view traces with the Zipkin UI at http://11.11.11.12:9411/zipkin/.
- If your applications aren't sending traces, yet, configure them with [Zipkin instrumentation](https://zipkin.io/pages/existing_instrumentations.html) or try one of our [examples](https://github.com/openzipkin?utf8=%E2%9C%93&q=example).

