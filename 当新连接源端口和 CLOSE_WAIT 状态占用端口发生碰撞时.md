# 当新连接源端口和 CLOSE_WAIT 状态占用端口发生碰撞时

## 抓包分析发生碰撞的场景

- Server 侧监听

```
[#157#root@ubuntu-1604 ~]$nc -l 12345
^Z
[1]+  Stopped                 nc -l 12345
[#158#root@ubuntu-1604 ~]$
```

- 监听状态确认

```
[#556#root@ubuntu-1604 ~]$while true; do ss -natp -o |grep 12345; sleep 1; echo "---"; done
---
...
---
LISTEN     0      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
---
LISTEN     0      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
---
...
```

- 启动抓包

```
[#308#root@ubuntu-1604 ~]$tcpdump -i lo tcp port 12345 -s 0 -w 12345.pcap
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes

```

- 发起 client 连接

```
[#158#root@ubuntu-1604 ~]$nc localhost 12345
^C
[#159#root@ubuntu-1604 ~]$
```

- 连接状态变化确认

```
...
---
LISTEN     0      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
---
LISTEN     0      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=12289,fd=3))
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=12289,fd=3))
---
...
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=12289,fd=3))
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,59sec,0)
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,58sec,0)
---
...
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,1.076ms,0)
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,052ms,0)
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
---
...
```

- 针对 CLOSE-WAIT 状态连接的端口发起新连接


```
[#159#root@ubuntu-1604 ~]$nc -v -p 39836 localhost 12345
Connection to localhost 12345 port [tcp/*] succeeded!
^C
[#160#root@ubuntu-1604 ~]$
```

- 新连接状态变化

```
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
SYN-SENT   0      1      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=14851,fd=3)) timer:(on,696ms,0)
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=14851,fd=3))
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=14851,fd=3))
---
...
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
ESTAB      0      0      127.0.0.1:12345              127.0.0.1:39836
ESTAB      0      0      127.0.0.1:39836              127.0.0.1:12345               users:(("nc",pid=14851,fd=3))
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,59sec,0)
---
...
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
FIN-WAIT-2 0      0      127.0.0.1:39836              127.0.0.1:12345               timer:(timewait,044ms,0)
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=11637,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39836
---
...
```

- 停止抓包

```
[#308#root@ubuntu-1604 ~]$tcpdump -i lo tcp port 12345 -s 0 -w 12345.pcap
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes

^C13 packets captured
26 packets received by filter
0 packets dropped by kernel
[#309#root@ubuntu-1604 ~]$
```

- 抓包分析

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E4%BD%BF%E7%94%A8%20CLOSE_WAIT%20%E8%BF%9E%E6%8E%A5%E5%8D%A0%E7%94%A8%E7%9A%84%20port%20%E4%BD%9C%E4%B8%BA%E6%BA%90%E7%AB%AF%E5%8F%A3%E5%BB%BA%E7%AB%8B%E6%96%B0%E8%BF%9E%E6%8E%A5%E6%97%B6%E7%9A%84%E6%83%85%E5%86%B5.png)

以下 frame 中的 Expert Info 信息应该关注

```
Frame 6: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 39836, Dst Port: 12345, Seq: 0, Len: 0
     ...
    [SEQ/ACK analysis]
        [iRTT: 0.000008000 seconds]
        [TCP Analysis Flags]
            [Expert Info (Note/Sequence): A new tcp session is started with the same ports as an earlier session in this trace]


Frame 7: 66 bytes on wire (528 bits), 66 bytes captured (528 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 12345, Dst Port: 39836, Seq: 1, Ack: 750470055, Len: 0
     ...
    [SEQ/ACK analysis]
        [iRTT: 0.000008000 seconds]
        [TCP Analysis Flags]
            [Expert Info (Warning/Sequence): ACKed segment that wasn't captured (common at capture start)]


Frame 8: 54 bytes on wire (432 bits), 54 bytes captured (432 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 39836, Dst Port: 12345, Seq: 750470055, Len: 0
    Source Port: 39836
    Destination Port: 12345
    [Stream index: 1]
    [TCP Segment Len: 0]
    Sequence number: 750470055    (relative sequence number)
    Acknowledgment number: 0
    0101 .... = Header Length: 20 bytes (5)
    Flags: 0x004 (RST)
    Window size value: 0
    [Calculated window size: 0]
    [Window size scaling factor: 128]
    Checksum: 0xa3a9 [unverified]
    [Checksum Status: Unverified]
    Urgent pointer: 0



Frame 9: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 39836, Dst Port: 12345, Seq: 0, Len: 0
    ...
    [SEQ/ACK analysis]
        [iRTT: 0.000008000 seconds]
        [TCP Analysis Flags]
            [Expert Info (Note/Sequence): This frame is a (suspected) retransmission]
            [The RTO for this segment was: 0.996552000 seconds]
            [RTO based on delta from frame: 8]


Frame 10: 74 bytes on wire (592 bits), 74 bytes captured (592 bits)
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
Transmission Control Protocol, Src Port: 12345, Dst Port: 39836, Seq: 3560068409, Ack: 1, Len: 0
    ...
    [SEQ/ACK analysis]
        [This is an ACK to the segment in frame: 9]
        [The RTT to ACK the segment was: 0.000019000 seconds]
        [iRTT: 0.000008000 seconds]
        [TCP Analysis Flags]
            [Expert Info (Note/Sequence): A new tcp session is started with the same ports as an earlier session in this trace]
            [Expert Info (Note/Sequence): This frame is a (suspected) retransmission]
            [The RTO for this segment was: 0.996579000 seconds]
            [RTO based on delta from frame: 7]

```



## Recv-Q 数值和 CLOSE_WAIT + ESTABLISHED 总量不等的原因


之前一直有一个疑问：为何 Recv-Q 的数值为 129 ，而看到的处于 CLOSE_WAIT 和 ESTAB 状态的连接总数却低于这个值；


实验如下：

- 监听后挂起

```
[#162#root@ubuntu-1604 ~]$nc -l 12345
^Z
[1]+  Stopped                 nc -l 12345
[#163#root@ubuntu-1604 ~]$
```

- 发起两次客户端连接并主动断开

```
[#163#root@ubuntu-1604 ~]$nc localhost 12345
^C
[#164#root@ubuntu-1604 ~]$nc localhost 12345
^C
[#165#root@ubuntu-1604 ~]$
```

- 连接状态确认

可以看到，这种情况下 Recv-Q 为 2 ，而 CLOSE_WAIT 连接数量也为 2 ；两个 CLOSE_WAIT 状态连接的 client 侧端口是不同的；

```
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=31458,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39958
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39968
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=31458,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39958
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39968
---
```

- 重新创建监听服务器并挂起

```
[#170#root@ubuntu-1604 ~]$nc -l 12345
^Z
[1]+  Stopped                 nc -l 12345
[#171#root@ubuntu-1604 ~]$
```

- 先建立一个 client 连接并断开

```
[#171#root@ubuntu-1604 ~]$nc localhost 12345
^C
[#172#root@ubuntu-1604 ~]$
```

- 此时连接状态为

```
---
LISTEN     1      1            *:12345                    *:*                   users:(("nc",pid=3098,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39994
---
```

- 以 CLOSE_WAIT 占用的 port 作为源端口建立一个新 client 连接后再断开

```
[#172#root@ubuntu-1604 ~]$nc -p 39994 localhost 12345
^C
[#173#root@ubuntu-1604 ~]$
```

- 连接状态稳定后为

```
---
LISTEN     2      1            *:12345                    *:*                   users:(("nc",pid=3098,fd=3))
CLOSE-WAIT 1      0      127.0.0.1:12345              127.0.0.1:39994
---
```

**这就是为何 Recv-Q 和 CLOSE_WAIT 状态连接数量不对应的原因**！！




