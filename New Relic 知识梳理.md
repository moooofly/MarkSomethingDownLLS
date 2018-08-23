# New Relic 知识梳理

## New Relic APM

一句话总结：提供应用性能监控能力；

[New Relic APM Features](https://newrelic.com/application-monitoring/features) 主要分为以下几个方面（按照功能分类）：

> 按照产品分类移步[这里](https://newrelic.com/application-monitoring/features/edition)；

- Application Monitoring (*)
- Database Monitoring
- Availability & Error
- Monitoring
- Reports
- Team Collaboration
- Security

> Some features are exclusive to **New Relic APM Pro**.

很多功能都是收费的~~

### Application Monitoring

> Application Monitoring **in one place** allows you to see your application performance **trends** at a glance – from **page load times**, to **error rates**, **slow transactions**, and a list of servers running the app.

关键：

- 基于单一视图查看应用性能变化趋势；
- 提供
   - 页面加载时间
   - 错误率
   - 慢事务
   - 构成当前 app 的运行服务器列表

#### Response Time, Throughput, and Error Rates

> **Response time** is an average of the total time spent across all web transactions occurring within a selected time frame on the server-side. **Throughput** is requests per minute through the application. **Error rates** measures the percentage of errors over a set time period for your application.

关键：

- **Response time** 针对的是：
    - 服务器侧
    - 选定时间片内
    - 全部 web transactions 上花费总时间的平均值
- **Throughput** 为特定应用上每分钟的请求数量（requests per minute, `rpm`）
- **Error rates** 给出的是目标应用在一定时间段内错误百分比；


#### Most Time-Consuming Transactions

> Transactions can be accessed both on the overview page and in the left hand navigation. This list of most time-consuming web transactions provides **aggregated details** around the surfaced slow transactions occurring during a specified time window alongside the total **response time** for each. Drill into these transactions to discover **specific details** around each individual transaction, where time is being spent, throughput and drill into **transaction traces**.

关键：

- 排序规则：基于 Transactions Time-Consuming 数值从大到小排序；
- `Transactions Time-Consuming = Avg.response time * Throughput`
- 排查问题过程中，既要考虑 **Avg.response time** 的绝对数值，也要考虑 **Throughput** 的绝对数值；
- 在未选中特定 transaction 前，展示的是 aggregated details ；选中后展示的是 specific details ；
    - aggregated details 仅适合观察最慢的几个接口都有谁，以及总体的变化趋势；
    - **specific details 适用于查看具体接口慢在那一层次的调用上**；
- **Transaction traces** 提供了代码中出现的、深层次的 slowdowns 信息，并能深入到某个特定的 method 或 database query 上，但需要升级到 New Relic Pro 才能使用；

#### Transaction Metrics and Traces

> Visualizing where your app is spending its time is a game-changer. You can't fix what you can't see. Transaction tracing extends the New Relic service to the tiniest transaction detail levels - Collecting data for your slowest individual HTTP requests, all the way down to the SQL.

用不了


#### Performance of External Services

> External service instrumentation captures calls to out-of-process services such as web services, resources in the cloud, and any other network calls. The external services dashboard provides charts with your top five external services by response time and external **calls per minute**.

关键：

- 用于排查**由于外部服务导致的访问延迟**问题；
- 需要理解当前都有哪些外部服务以及常规表现，当前看到的外部服务有：
    - **ELB**: internal-venet-internal-prod-elb-1301483333.cn-north-1.elb.amazonaws.com.cn
    - **环信**: a1.easemob.com
    - **homepage**: homepage-prod-api-lb.backend
    - **wechat-prod**: 172.31.9.10
    - **微博开放平台**:api.weibo.com
    - api.weixin.qq.com


## New Relic Insights

一句话总结：主要提供 data analysis 能力；

> Organize, visualize, evaluate. With in-depth analytics, you can better understand the end-to-end business impact of your software performance.

用不了

## New Relic Infrastructure

一句话总结：主要提供 monitoring 能力；

> Get a precise picture of your dynamically changing systems. Scale rapidly. Deploy intelligently. Run your business optimally.

用不了

## Q&A

### 0x01 我们当前使用的版本和 ESSENTIALS/PRO 的关系？

> xxx

### 0x02 Apdex 是什么？

Apdex 是一种工业标准（industry standard），**用于衡量访问 web 应用和服务时，用户对 response time 的满意程度**；可以将其认为是一种简化的 Service Level Agreement (SLA) 解决方案，由此为 application owner 提供用户满意度的参考数据；和传统的平均应答时间（average response time）等 metrics 相比，其不容易受少量的、超长时间的 responses 的影响；

以下内容取自 http://apdex.org/overview.html

> **Apdex (Application Performance Index)** is an open standard developed by an alliance of companies that defines a standardized method to **report**, **benchmark**, and **track** application performance.
>
> 评价：
> 
> "Apdex represents a new milestone in application analysis. The network management industry needs an agreed upon standard to **reflect the experience of the end-user** for everyday applications. **Apdex is a natural extension to our tried and proven Application Response Time analysis** already incorporated into our existing products."
>
> 产生背景：
> 
> Enterprises are swimming in IT performance numbers, but have no insight into how well their applications perform from a business point of view. Response time values do not reveal whether users are productive, and a greater number of samples leads to confusion. Averaging the samples washes out significant details about frustration with slow response times. Time values are not uniform across different applications. There should be a better way to analyze and measure what matters.

### 0x03 Apdex 的度量方式

Apdex 是基于一组 threshold 值来对 response time 进行度量的；其计算了 satisfactory response 次数和 unsatisfactory response 次数的比例；response time 的获取是通过计算针对某个 asset 发起请求，与收到完整应答的时间差值计算得到的；

application owner 定义一个 response time 的 threshold 为 T ；定义所有在 T 时间完成处理的 response 都是满足了 user 的；

可以针对每一个 New Relic apps 定义单独的 Apdex T 数值；还可以定义独立的 Apdex T thresholds 值用于 key transactions ；

Apdex tracks three response counts:

- **Satisfied**: The response time is **less than or equal to T**.
- **Tolerating**: The response time is **greater than T and less than or equal to 4T**.
- **Frustrated**: The response time is **greater than 4T**.

For example, if T is 1.2 seconds and a response completes in 0.5 seconds, then the user is **satisfied**. All responses greater than 1.2 seconds **dissatisfy** the user. Responses greater than 4.8 seconds **frustrate** the user.


## 参考

- https://newrelic.com/
- https://docs.newrelic.com/docs/apm