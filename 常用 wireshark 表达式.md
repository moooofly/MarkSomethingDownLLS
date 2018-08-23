# 常用 wireshark 过滤表达式

参考：http://www.lovemytool.com/blog/2010/04/top-10-wireshark-filters-by-chris-greer.html

## ip.addr == 10.0.0.1

Sets a filter for any packet with 10.0.0.1, as either the source or dest

## ip.addr==10.0.0.1  && ip.addr==10.0.0.2

sets a conversation filter between the two defined IP addresses

## http or dns

sets a filter to display all http and dns

## tcp.port==4000

sets a filter for any TCP packet with 4000 as a source or dest port

## tcp.flags.reset==1

displays all TCP resets

## http.request

displays all HTTP GET requests

## tcp contains traffic

displays all TCP packets that contain the word ‘traffic’. Excellent when searching on a specific string or user ID

## !(arp or icmp or dns)

masks out arp, icmp, dns, or whatever other protocols may be background noise. Allowing you to focus on the traffic of interest

## udp contains 33:27:58

sets a filter for the HEX values of 0x33 0x27 0x58 at any offset

## tcp.analysis.retransmission

displays all retransmissions in the trace. Helps when tracking down slow application performance and packet loss

## dns.qry.name contains xxx.com.cn

过滤 DNS Query string 中包含指定子串的 Query 和 Query Response

## dns.qry.type == 1 || dns.qry.type == 28

过滤类型为 A (Host Address) 和 AAAA (IPv6 Address) 的 Query 和 Query Response




