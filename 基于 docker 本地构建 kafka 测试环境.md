# 基于 docker 本地构建 kafka 测试环境

- kafka 环境：https://github.com/wurstmeister/kafka-docker/wiki/Connectivity
- [edenhill/kafkacat](https://github.com/edenhill/kafkacat)


## kafkacat


> kafkacat is a generic non-JVM producer and consumer for Apache Kafka >=0.8, think of it as a netcat for Kafka.

- 通用的、基于命令行的、non-JVM 的，Kafka producer 和 consumer ；
- 适用于 Apache Kafka >=0.8
- 等价于 netcat 的作用

## 模式

> In **producer mode** kafkacat reads messages from `stdin`, delimited with a configurable delimiter (`-D`, defaults to newline), and produces them to the provided Kafka cluster (`-b`), topic (`-t`) and partition (`-p`).

- 在 producer 模式下，kafkacat 从 stdin 进行 messages 读取，默认的多 messages 分隔符号为换行符（可以通过 `-D` 进行改变）；
- 在 producer 模式下，kafkacat 可以将 message 发送到指定的 Kafka 集群（`-b`）中，指定的 topic 上（`-t`），以及指定的 partition 中（`-p`）；


> In **consumer mode** kafkacat reads messages from a topic and partition and prints them to stdout using the configured message delimiter.

- 在 consumer 模式下，kafkacat 将从指定的 topic 和 partition 中进行 messages 的读取，并将其从 stdout 上输出；
- 输出时同样可以控制 message 使用的分隔符号；

> kafkacat also features a Metadata list (`-L`) mode to display the current state of the Kafka cluster and its topics and partitions.

kafkacat 还支持 Metadata 模式（`-L`），用于查看 kafka 集群的元数据信息，包括 topics 和 partitions 信息；

> There's also support for the Kafka >=0.9 high-level balanced consumer, use the `-G <group>` switch and provide a list of topics to join the group.

kafkacat 还支持 balanced consumer ；


## 安装

- Ubuntu

```
apt-get install kafkacat
sudo apt-get install librdkafka-dev libyajl-dev
```

- Mac OS X

```
brew install kafkacat
```


## 使用

> 更多高级用法，详见 github 上的 README.md

- 获取 metadata

```
root@proxy-hangzhou:~# kafkacat -L -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778
Metadata for all topics (from broker 1001: 47.98.126.155:32776/1001):
 3 brokers:
  broker 1001 at 47.98.126.155:32776
  broker 1003 at 47.98.126.155:32778
  broker 1002 at 47.98.126.155:32777
 2 topics:
  topic "test_topic" with 1 partitions:
    partition 0, leader 1001, replicas: 1001, isrs: 1001
  topic "beats" with 1 partitions:
    partition 0, leader 1001, replicas: 1001, isrs: 1001
root@proxy-hangzhou:~#
```

- 创建 consumer

```
root@proxy-hangzhou:~# kafkacat -C -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_topic

% Reached end of topic test_topic [0] at offset 15


1:{"order_id":1,"order_ts":1534772501276,"total_amount":10.50,"customer_name":"Bob Smith"}
2:{"order_id":2,"order_ts":1534772605276,"total_amount":3.32,"customer_name":"Sarah Black"}
3:{"order_id":3,"order_ts":1534772742276,"total_amount":21.00,"customer_name":"Emma Turner"}
% Reached end of topic test_topic [0] at offset 18



topic
% Reached end of topic test_topic [0] at offset 19
topic
% Reached end of topic test_topic [0] at offset 20

```

- 创建 producer


```
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_topic <<EOF
> 1:{"order_id":1,"order_ts":1534772501276,"total_amount":10.50,"customer_name":"Bob Smith"}
> 2:{"order_id":2,"order_ts":1534772605276,"total_amount":3.32,"customer_name":"Sarah Black"}
> 3:{"order_id":3,"order_ts":1534772742276,"total_amount":21.00,"customer_name":"Emma Turner"}
> EOF
root@proxy-hangzhou:~#
root@proxy-hangzhou:~#
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_a <<EOF
> aaa
> EOF
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_a <<EOF
aaa
EOF
root@proxy-hangzhou:~#
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_topic <<EOF
> topic
> EOF
root@proxy-hangzhou:~#
root@proxy-hangzhou:~#
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_a <<EOF
aaa
EOF
root@proxy-hangzhou:~#
root@proxy-hangzhou:~# kafkacat -P -b 47.98.126.155:32776,47.98.126.155:32777,47.98.126.155:32778 -t test_topic <<EOF
topic
EOF
root@proxy-hangzhou:~#
```

## 一个具体的使用案例

> https://github.com/moooofly/MarkSomethingDownLLS/issues/78#issuecomment-479787685



