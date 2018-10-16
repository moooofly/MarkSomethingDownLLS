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

详见：https://github.com/moooofly/MarkSomethingDownLLS/blob/master/OpenCensus%20Agent.md#opencensus-agent-1

### OpenCensus Collector

详见：https://github.com/moooofly/MarkSomethingDownLLS/blob/master/OpenCensus%20Agent.md#opencensus-collector


----------

## 实际测试

- server

```
# 编译+安装
[#121#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$go build -v && go install
[#122#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$ll /go/bin/opencensusd
-rwxr-xr-x 1 root root 18248976 Jul 30 20:13 /go/bin/opencensusd*
[#123#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$date
Mon Jul 30 20:13:54 CST 2018

# 启动
[#125#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$opencensusd
2018/07/30 20:14:39 endpointFile path: /root/.config/opencensus.endpoint

```

- client

```
[#61#root@ubuntu-1604 /go/src/github.com/census-instrumentation/opencensus-service/cmd/opencensusd]$OPENCENSUS_DEBUG=1 go run "$(go env GOPATH)/src/github.com/census-instrumentation/opencensus-service/example/main.go"
Looking for the endpoint file
Dialing [127.0.0.1:35246]...
Sleeping for a minute...
Exporting span [[{9b826b38005d0a8a8e692c2d2c63aaf1 f3ef0a9e888ea738 1}]]
Exporting span [[{69d8cf7a7c892bf0642fdcb8be73238d 84ee8d1b4dfa55fc 1}]]
Exporting span [[{9cae079607dd3d855e531c888e6b0fcc 15ed1099116604c0 1}]]
Exporting span [[{88d8e385c354202f0ae926bee18abb12 a6eb9316d6d1b283 1}]]
Exporting span [[{84b7718e3ab0fe464b98b11bc5e2d8a1 37ea16949a3d6147 1}]]
Exporting span [[{7571fb807d592930390fc96c6549da4d c8e899115fa90f0b 1}]]
Exporting span [[{47a6ff00127fef010bb63621b402282e 59e71c8f2315bece 1}]]
Exporting span [[{0e0b6f2aee551d62a083b9eac90a4651 eae59f0ce8806c92 1}]]
Exporting span [[{229c19b1140b6d91d5cedb1ce4bf44cc 7be4228aacec1a56 1}]]
Exporting span [[{ae193dbcccd194bfce7b33f74a3a7741 0ce3a5077158c919 1}]]
Exporting span [[{ec91248d8e8557fd4831594dcb48a45a 9de1288535c477dd 1}]]
Exporting span [[{c0bd2050515e617f28d125dd1e70fa6d 2ee0ab02fa2f26a1 1}]]
Exporting span [[{9f5f77627c0980953665d5ab9dd380fe bfde2e80be9bd464 1}]]
Exporting span [[{d81fc8c55478f2dc7d6da7301ec242f2 50ddb1fd82078328 1}]]
Exporting span [[{464afaa7b66820bf0224a4e3396cd489 e1db347b477331ec 1}]]
Exporting span [[{2333b3b59d3ab96d35f493f81a47936e 72dab7f80bdfdfaf 1}]]
Exporting span [[{e0cb0c87ad55d596374565529abac1f2 03d93a76d04a8e73 1}]]
Exporting span [[{c875674ec14e1bfb157805fdf4204262 94d7bdf394b63c37 1}]]
Exporting span [[{cbbf2316457c09ebb2eb83442d3b03af 25d640715922ebfa 1}]]
Exporting span [[{9972a289097b23be4172cdd52e0f3687 b6d4c3ee1d8e99be 1}]]
Exporting span [[{0371f63944fd28fb988915c24e755631 47d3466ce2f94782 1}]]
Exporting span [[{ea2020503ff00035dbeb4f04f630597b d8d1c9e9a665f645 1}]]
Exporting span [[{61d50bac6795b91da8d173b6fa358ae5 69d04c676bd1a409 1}]]
Exporting span [[{5327a218b8f5211f9bcdeacd6867711a facecfe42f3d53cd 1}]]
Exporting span [[{e5015fecaf1708629502b7b59022e851 8bcd5262f4a80191 1}]]
Exporting span [[{a556214bfb0b2680a253eeb622dc0279 1cccd5dfb814b054 1}]]
Exporting span [[{5041ddb6f5fe45ae78d4452e71d96415 adca585d7d805e18 1}]]
Exporting span [[{97b6c1fd66449f434ae3dcab9b6a8e83 3ec9dbda41ec0cdc 1}]]
Exporting span [[{b1f9e94c634ec4fc6bf07926d6e53f07 cfc75e580658bb9f 1}]]
Exporting span [[{cb98a1e4998ae9d4b1d95dd22dc66e1b 60c6e1d5cac36963 1}]]
Exporting span [[{df02b46fc1c1345c6ce16de386dc174b f1c464538f2f1827 1}]]
Exporting span [[{b2c49c83ea58c29eefb486934a67f764 82c3e7d0539bc6ea 1}]]
Exporting span [[{c84f092f05e080611507a5bf2bb5b324 13c26a4e180775ae 1}]]
Exporting span [[{dd32e93aef7a494865f85679579294cd a4c0edcbdc722372 1}]]
Exporting span [[{c901393172764a69190e896170870119 35bf7049a1ded135 1}]]
Exporting span [[{d1652a3bf219fde7a260b75f8272847b c6bdf3c6654a80f9 1}]]
...
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

