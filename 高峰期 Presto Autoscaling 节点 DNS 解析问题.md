# 高峰期 Presto Autoscaling 节点 DNS 解析问题

## 结论

> https://phab.llsapp.com/T77831


### 问题点

简单看了下，怀疑和 CPU 使用有关：

- 发现在使用高峰，CPU 几乎被 presto 用光；
- 另外 CPU5 和 CPU7 似乎被“绑定“专门用于处理网络包；
- 在 CPU 被耗尽的情况下（%us + %sy + %si），有可能发生
    - 操作系统调度实体排队（产生延迟）
    - 网络包得不到及时处理（可能出现丢包和延迟）
    - 当访问 dns 服务的实体或提供 dns 服务的实体恰好调度到 CPU 被跑满的核心上时，可能正常时间无法完成（触发业务超时）

### 可以采用的改进措施

- 目前 EC2 instance 为 36c ，如果可以进一步扩大，肯定对问题有缓解；
- 优化 presto 的代码，占用 CPU 的主要是 %us 说明用户态代码需要进行优化（如果确定无法进一步优化，那就只能增加 CPU core）
- 鉴于发生问题时的错误报错主要和 DNS 查询有关，可以使用措施有
    - 进行 cpu core 的使用划分（隔离），例如 0-34 给 presto 使用，35 给 dns 相关服务使用（有可能有改善）；
    - 优化业务的访问模型，从 aws monitoring 上观察，有很明显的迹象表明 CPU 的使用曲线和 network 的 in/out 曲线相吻合，如果有可能将 network 流量变的更为均衡，应该也可以解决问题；


## 信息交互

- 出错时的日志

主要报错位置

```
com.amazonaws.SdkClientException: Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn: unknown error
...
Caused by: java.net.UnknownHostException: warehouse.s3.cn-north-1.amazonaws.com.cn: unknown error
...
```

详细日志

```
com.amazonaws.SdkClientException: Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn: unknown error
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.handleRetryableException(AmazonHttpClient.java:1114)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.executeHelper(AmazonHttpClient.java:1064)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.doExecute(AmazonHttpClient.java:743)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.executeWithTimer(AmazonHttpClient.java:717)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.execute(AmazonHttpClient.java:699)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.access$500(AmazonHttpClient.java:667)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutionBuilderImpl.execute(AmazonHttpClient.java:649)
	at com.amazonaws.http.AmazonHttpClient.execute(AmazonHttpClient.java:513)
	at com.amazonaws.services.s3.AmazonS3Client.invoke(AmazonS3Client.java:4330)
	at com.amazonaws.services.s3.AmazonS3Client.invoke(AmazonS3Client.java:4277)
	at com.amazonaws.services.s3.AmazonS3Client.invoke(AmazonS3Client.java:4271)
	at com.amazonaws.services.s3.AmazonS3Client.listObjects(AmazonS3Client.java:835)
	at com.facebook.presto.hive.s3.PrestoS3FileSystem.listPrefix(PrestoS3FileSystem.java:476)
	at com.facebook.presto.hive.s3.PrestoS3FileSystem.access$000(PrestoS3FileSystem.java:141)
	at com.facebook.presto.hive.s3.PrestoS3FileSystem$1.<init>(PrestoS3FileSystem.java:264)
	at com.facebook.presto.hive.s3.PrestoS3FileSystem.listLocatedStatus(PrestoS3FileSystem.java:262)
	at org.apache.hadoop.fs.FilterFileSystem.listLocatedStatus(FilterFileSystem.java:263)
	at com.facebook.presto.hive.HadoopDirectoryLister.list(HadoopDirectoryLister.java:30)
	at com.facebook.presto.hive.util.HiveFileIterator$FileStatusIterator.<init>(HiveFileIterator.java:131)
	at com.facebook.presto.hive.util.HiveFileIterator$FileStatusIterator.<init>(HiveFileIterator.java:119)
	at com.facebook.presto.hive.util.HiveFileIterator.getLocatedFileStatusRemoteIterator(HiveFileIterator.java:108)
	at com.facebook.presto.hive.util.HiveFileIterator.computeNext(HiveFileIterator.java:101)
	at com.facebook.presto.hive.util.HiveFileIterator.computeNext(HiveFileIterator.java:38)
	at com.google.common.collect.AbstractIterator.tryToComputeNext(AbstractIterator.java:141)
	at com.google.common.collect.AbstractIterator.hasNext(AbstractIterator.java:136)
	at java.util.Spliterators$IteratorSpliterator.tryAdvance(Spliterators.java:1811)
	at java.util.stream.StreamSpliterators$WrappingSpliterator.lambda$initPartialTraversalState$0(StreamSpliterators.java:294)
	at java.util.stream.StreamSpliterators$AbstractWrappingSpliterator.fillBuffer(StreamSpliterators.java:206)
	at java.util.stream.StreamSpliterators$AbstractWrappingSpliterator.doAdvance(StreamSpliterators.java:161)
	at java.util.stream.StreamSpliterators$WrappingSpliterator.tryAdvance(StreamSpliterators.java:300)
	at java.util.Spliterators$1Adapter.hasNext(Spliterators.java:681)
	at com.facebook.presto.hive.BackgroundHiveSplitLoader.loadSplits(BackgroundHiveSplitLoader.java:251)
	at com.facebook.presto.hive.BackgroundHiveSplitLoader.access$300(BackgroundHiveSplitLoader.java:89)
	at com.facebook.presto.hive.BackgroundHiveSplitLoader$HiveSplitLoaderTask.process(BackgroundHiveSplitLoader.java:183)
	at com.facebook.presto.hive.util.ResumableTasks.safeProcessTask(ResumableTasks.java:47)
	at com.facebook.presto.hive.util.ResumableTasks.access$000(ResumableTasks.java:20)
	at com.facebook.presto.hive.util.ResumableTasks$1.run(ResumableTasks.java:35)
	at io.airlift.concurrent.BoundedExecutor.drainQueue(BoundedExecutor.java:78)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.net.UnknownHostException: warehouse.s3.cn-north-1.amazonaws.com.cn: unknown error
	at java.net.Inet4AddressImpl.lookupAllHostAddr(Native Method)
	at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:928)
	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1323)
	at java.net.InetAddress.getAllByName0(InetAddress.java:1276)
	at java.net.InetAddress.getAllByName(InetAddress.java:1192)
	at java.net.InetAddress.getAllByName(InetAddress.java:1126)
	at com.amazonaws.SystemDefaultDnsResolver.resolve(SystemDefaultDnsResolver.java:27)
	at com.amazonaws.http.DelegatingDnsResolver.resolve(DelegatingDnsResolver.java:38)
	at org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:112)
	at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.connect(PoolingHttpClientConnectionManager.java:373)
	at sun.reflect.GeneratedMethodAccessor882.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.amazonaws.http.conn.ClientConnectionManagerFactory$Handler.invoke(ClientConnectionManagerFactory.java:76)
	at com.amazonaws.http.conn.$Proxy229.connect(Unknown Source)
	at org.apache.http.impl.execchain.MainClientExec.establishRoute(MainClientExec.java:381)
	at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:237)
	at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:185)
	at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:83)
	at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)
	at com.amazonaws.http.apache.client.impl.SdkHttpClient.execute(SdkHttpClient.java:72)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.executeOneRequest(AmazonHttpClient.java:1236)
	at com.amazonaws.http.AmazonHttpClient$RequestExecutor.executeHelper(AmazonHttpClient.java:1056)
	... 39 more
Instance ID(s): 
```

- 需要提供 ASG 名称和 instance ID

> 我们正在调查您的问题，不知您这次出现问题的 autoscaling group 名称和实例 id 是多少，还请您提供一下这些信息以帮助我们进一步调查。

- 需要 dig 脚本和 tcpdump 

> 我们查看了您另外一个案例里的历史记录，其中提到您在实例上都挂后台跑脚本，同时进行 dig vpcdns 和 dig 114dns 测试并进行 tcpdump ，不知这次出现问题的实例上是否也有类似测试，如果有还请您提供以下这些测试结果。

之前使用的、每分钟执行一次的脚本

```
date +'%F %T'
dig warehouse.s3.cn-north-1.amazonaws.com.cn
dig neo-mysql-prod-slave-data-0.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn
dig warehouse.s3.cn-north-1.amazonaws.com.cn @114.114.114.114
dig neo-mysql-prod-slave-data-0.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn @114.114.114.114
echo
echo
echo
echo
```

- 关于环境差异的问题

> 感谢您的回复。我们检查您的账号和 ASG ，还有几个信息需要您帮忙确认：
> 
> 1. 您账号下一共有 39 个 ASG ，是否每个 ASG 里的实例都出现了问题？ 还是说始终只有一个或者某些特定的 ASG 里的实例有问题？
> 2. 当某一个实例出现问题时，同一 ASG 里的其他实例是否也有问题？
> 3. 实例的问题是间歇的还是一直连续的？也就是说，实例在被 ASG 里启动起来后就出现了问题，还是说在运行一段时间后才出现了问题，并且不久就恢复？
> 4. 您不在 ASG 里的实例，也就是您自己启动的实例是否也有这个问题？

- 关于环境差异的回答

> 1. 出现问题的就只有提供给你们的那个 asg 其他没有问题
> 2. 其他实例也有出现过.
> 3. 间歇出现
> 4. 自己启动的没有出现这个问题.


----------


## 上一次出现的同样的问题

- Presto 访问 S3 报错（N 个报错节点），同时出现了连不上 RDS 的情况；

DNS 解析失败

```
Caused by: com.amazonaws.AmazonClientException: Unable to execute HTTP request: warehouse.s3.cn-north-1.amazonaws.com.cn
...
Caused by: java.net.UnknownHostException: warehouse.s3.cn-north-1.amazonaws.com.cn
	at java.net.InetAddress.getAllByName0(InetAddress.java:1280)
...
```

访问 RDS 失败

```
java.lang.RuntimeException: com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Could not create connection to database server. Attempted reconnect 3 times. Giving up.
...
Caused by: java.net.UnknownHostException: neo-mysql-prod-slave-data-0.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn
	at java.net.InetAddress.getAllByName0(InetAddress.java:1280)
...
```

- 将监测脚本同时部署到正常和出错节点上，每分钟执行一次

上线脚本到 172.31.31.154 和 172.31.17.30

```
#!/bin/bash

date +'%F %T'
dig warehouse.s3.cn-north-1.amazonaws.com.cn
dig warehouse.s3.cn-north-1.amazonaws.com.cn @114.114.114.114
dig neo-mysql-prod-slave-data-0.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn
dig neo-mysql-prod-slave-data-0.cyhqzoqszkzc.rds.cn-north-1.amazonaws.com.cn @114.114.114.114
echo
echo
echo
echo
```

- 检查 /var/log/message 确认是否有相关错误出现

根据与您在电话上沟通，JAVA 层面的报错是 java.net.UnknownHostException 看起来是域名解析的问题，网络层面我们首先查了 /var/log/message 里面的信息 那个时刻没有报错，且也没有 dhcp 更新失败，同时我们又检查了底层物理机，发现同一个物理机上 172.31.17.30 172.31.31.154 这两个机器都是集群的成员，172.31.17.30 有这个报错，但是 172.31.31.154 就没有问题，**由于 VPC 用的是默认 DNS 配置，dns 请求会先发到物理机，然后物理机转发**，172.31.31.154 的解析就没问题，看起来也不像物理机网络故障；

那么下一步建议排查一下 OS 层面是否能够正常解析 DNS ，根据咱们的讨论，在后台挂了 dig 的脚本，分别 dig VPC dns 和 114 dns ，然后监控对比一下这 2 个机器，看看是否能够抓到 OS 无法解析 dns 的问题；

- 抓包

从您的回复得知已经在实例 172.31.31.154 和 172.31.17.30 部署脚本来测试 dns 解析。

您也可以在这两个实例上使用 tcpdump 来抓包，观察一段时间，如何能抓到 dns 解析失败的包，对于分析这个问题非常有帮助。

```
sudo tcpdump -i eth0 udp and port 53 -w /tmp/dns20171121.pcap &
```

- 抓包脚本

```
#!/bin/bash

local_ip=`ifconfig |grep 172.|cut -d : -f 2|awk '{print $1}'`
log_file="/tmp/dns.${local_ip}.`date +'%F.%H%M%S'.log`"
echo $log_file
nohup tcpdump -i eth0 udp and port 53 -w "$log_file" &
```

- DNS 配置文件

线上服务器 /etc/resolv.conf 文件内容如下: 

```
[ec2-user@ip-172-31-16-61 ~]$ cat /etc/resolv.conf
options timeout:2 attempts:5
; generated by /sbin/dhclient-script
search cn-north-1.compute.internal
nameserver 172.31.0.2
```

所有节点内容都类似，没有修改过；

- 下一步

问题描述：

同一个 Autoscaling 组起来的机器，随机发生有机器报 DNS 无法解析问题；

后续步骤：

根据最近一次的电话沟通，由于**发生的机器不固定，发生时间也不固定**，目前建议采用的方案是把所有机器都挂后台跑脚本，同时进行 dig vpcdns 和 dig 114dns 并进行 tcpdump ；

希望确认一下发生故障的时候，dns 不能解析是只有 vpc dns 不能解析，还是 114 dns 也同时不能解析；然后再根据结果，制定下一步的排查方向；


## 相关链接

- [高峰期 Presto Autoscaling 节点 DNS 解析问题](https://phab.llsapp.com/T77831)
- [autoscaling group 节点访问 S3 出现域名错误](https://console.amazonaws.cn/support/home?region=cn-north-1#/case/?displayId=1422257394&language=zh)
- [Presto 访问 S3 报错 UnknownHostException: warehouse.s3.cn-north-1.amazonaws.com.cn](https://console.amazonaws.cn/support/home?region=cn-north-1#/case/?displayId=1397915224&language=zh)


