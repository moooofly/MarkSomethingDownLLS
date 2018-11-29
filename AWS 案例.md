# AWS 案例

## 目录

- [一些海外用户无法正常访问 ELB](#一些海外用户无法正常访问-elb)
- [用户请求不到 ELB](#用户请求不到-elb)
- [用户反馈手机通过 wifi 访问服务超时，切换到移动网络就可以](#用户反馈手机通过-wifi-访问服务超时切换到移动网络就可以)


## 一些海外用户无法正常访问 ELB

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1420393104&language=zh

### 0x01 提供的信息

时间：北京时间9月27日14:30左右
现象：86.41.119.4 这个 ip 的用户访问我们服务有问题（通过 ELB: neo-app-prod-elb），看了 ELB log，北京时间9月27日 14:30 左右没有这个 ip 的 log，但同时访问其他 ELB 能够收到请求。说明用户的网络没有问题，而是访问这个 ELB 出现了网络问题。
ELB FQDN(s): neo-app-prod-elb

### 0x02 分析

> [问题描述]
>
> 您案例里提到了来自**爱尔兰**的用户（ 86.41.119.4 ）在北京时间今天（9月27日）下午14:30分左右访问 ELB 出现了问题，但同时访问其他 ELB 是正常的，您想知道原因。如果我的理解有误，还请您指正。

> [分析过程及解决方案]
> 
> 我们**检查了您 neo-app-prod-elb 这个 ELB 以及对应后端实例的状态**，没有发现异常情况。同时，对于您的描述我们有几点不太清楚的情况，想跟您确认一下并且收集一些信息以便我们进一步的定位问题所在：
>
> 1. 您说访问出现了问题，能具体描述一下是什么问题吗？如果方便的话您可否在重现问题的时候打开浏览器开发者工具（Chrome 和 Firefox 按下 F12 即可打开），点击 Network 一栏，将具体发生错误的链接详细信息截图，以便我们确认访问具体问题所在；
> 2. 您能否做一个**从遇到问题客户端到 ELB 节点的（双向） `traceroute`** ，以便我们排查网络方面的问题；
> 3. 您在案例里提到从 86.41.119.4 这个 IP 访问其他的 ELB 是可以收到请求的，这里其他的 ELB 指的是中国 Region 的 ELB 吗？
>
> 由于**国际间的网络传输环境比较复杂**，可能有很多原因导致问题的出现，您提供的信息将对我们排查问题有非常大的帮助，我们会根据您提供的信息尽力为您排查。

要点：

- PC 端可以通过浏览器提供的开发者工具进行问题排查（手机上用不了）；
- 需要提供客户端到 ELB 节点的双向 `traceroute` 或抓包；

### 0x03 信息补充

1. 就是访问 API 一直连不上，因为是手机端，所以信息不是很多
2. 现在还不行
3. 是的。同一个账号里的其他 ELB

### 0x04 分析

> 您的这个用户（ 86.41.119.4 ）现在可以正常访问了吗？我们在北京做了 curl 访问 ELB：neo-app-prod-elb 是正常的。
> 
> 经过和后台网络团队沟通，我们得知，**在北京时间9月27日下午14:30分中国到爱尔兰的网络有拥塞的情况发生**，您能否告知我们连接失败的具体时间段，以便我们确认连接失败与拥塞的时间是否吻合。
>
> 由于您暂时无法提供**用户到服务器之间双向 traceroute 的信息**以及**客户端和服务端的抓包记录**，我们能够用来判断的信息有限，您能否提供给我们发生问题对应时间段 ELB：neo-app-prod-elb 以及您之前提到的另一个正常 ELB 的访问日志，以便我们进一步分析。

### 0x05 信息补充

9月23日出现，然后测试时间是 9.27日14:31。

能请求到的那个日志放在附件里了。（ELB: kong-llsapp-lb）

neo-api 日志太多了，这里放不了（全部 gzip 压缩后有 75.2MB），但我找了，确实没有：

```
$ ls
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.144.69_4zbbzph4.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.154.228_62lvltjp.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.248.43_5dvsmar6.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.157.95_386m2y6n.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.31.11_2xzjgqse.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.177.24_59fxsvda.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.49.63_5x1vn2w7.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.222.194.144_1a098i83.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.80.92.163_5vex76fe.log   009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.124.196_3l7s9vqm.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.81.4.171_53vnzd0w.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.184.231_56pkna9b.log
009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_52.81.8.215_wu8faezd.log    009943267516_elasticloadbalancing_cn-north-1_neo-app-prod-elb_20180927T0700Z_54.223.251.198_4ssg0b8j.log
$ ag 86.41.119.4
(blank)
```

### 0x06 分析

> 结合您提供日志中的时间段，我们检查了对应时间下 ELB 及后端实例的各项指标，没有发现异常。从北京到爱尔兰的网络拥塞时间段（北京时间9月27日下午2点半到3点）的这个时间段和您用户无法访问服务的时间段是吻合的，由此我们怀疑很可能是网络的原因导致了问题的发生。**对于分析这类网络问题，最有帮助的同时也是我们还没有得到的是问题客户端与服务端的抓包日志和双向的 mtr 记录**。
>
> 我们现在观察到北京到爱尔兰的网络拥塞情况已经有所缓解，不知道您的这个用户现在是否可以正常访问 ELB: neo-app-prod-elb ，如果依然不能正常访问的话，我们还是希望您能帮忙**提供客户端 ip 下的双向 mtr 记录**以**及客户端和服务端的抓包日志**，以便我们查找产生问题的原因。如果没有详细的日志，我们能做到的分析也会比较有限，还望您能谅解。
>
> 您的用户不能正常访问服务，我们明白您的感受，同时国际间的网络环境的确比较复杂，我们会尽可能的根据现有信息帮您分析，还望您能理解。如果有进一步问题，欢迎随时与我们联系。谢谢！

## 用户请求不到 ELB

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1420556564&language=zh

### 0x01 提供的信息

时间：9月27日 14:25 左右
现象：ip 为 183.238.79.10 的用户请求发不到 kong-llsapp-lb 这个 ELB ，一直超时，重试多次都没用；切 wifi 和 4G 都不行，并且 ELB log 中没有这个 ip，而同时这个用户却可以正常请求到 neo-app-prod-elb 这个 ELB 。


### 0x02 分析

> 我们检查了您 **ELB: kong-llsapp-lb** 以及对应后端实例（的状态），并且将  **ELB: kong-llsapp-lb** 与 **ELB: neo-app-prod-elb** 的 metrics 做了一个对比，没有发现异常。同时，我们对您的 ELB 做了一个 curl，显示结果如下，是正常的：

```
[ec2-user@ip-172-31-15-229 ~]$ curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn
* Rebuilt URL to: https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn/
*   Trying 54.223.53.86...
* TCP_NODELAY set
* Connected to kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn (54.223.53.86) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Client hello (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: OU=Domain Control Validated; OU=EssentialSSL Wildcard; CN=*.llsapp.com
*  start date: Nov 29 00:00:00 2017 GMT
*  expire date: Dec 10 23:59:59 2019 GMT
*  issuer: C=GB; ST=Greater Manchester; L=Salford; O=COMODO CA Limited; CN=COMODO RSA Domain Validation Secure Server CA
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x25d5e60)
> GET / HTTP/2
> Host: kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn
> User-Agent: curl/7.55.1
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
< HTTP/2 404
< date: Fri, 28 Sep 2018 10:31:53 GMT
< content-type: application/json; charset=utf-8
<
{"message":"no route and no API found with those values"}
* Connection #0 to host kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn left intact
```

> 令我们困惑的是，您说这个 **183.238.79.10** 用户通过 wifi 和 4G 都访问不了，这两种访问方式应该是有对应 2 个 IP ，我们想知道另一个 IP 以及它的访问方式（是 4G 还是 wifi），这样就我们就可以确认两种访问方式是否是来自同一个运营商。
>
> 您在案例里提到这个用户在9月7日下午 14:25 分左右访问不到 ELB，我们想知道这个现象大概持续了多久，现在是否已经可以访问了？您能否方便向我们提供一下 **ELB: neo-app-prod-elb** 以及 **ELB: kong-llsapp-lb** 在问题发生期间的 Log ，这将有助于我们去排查这是 ELB 的问题还是网络的问题，以便做进一步的分析。
>
> 您如果方便的话，能否在您的出问题用户的 wifi 环境下做个 `curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn` ，并将结果发给我们。

要点：

- 针对不同 ELB 进行 metrics 对比
- 通过 `curl -kv https://kong-llsapp-lb-1173809350.cn-north-1.elb.amazonaws.com.cn` 确认 ELB 是否能被正常访问；
- 在出问题的用户环境中执行上述 curl 命令（有困难）


## 用户反馈通过 wifi 访问服务超时切换到移动网络就可以

> ref: https://console.amazonaws.cn/support/v1#/case/?displayId=1419913324&language=zh

### 0x01 提供的信息

服务：apineo.llsapp.com
服务对应的 ELB FQDN(s)：neo-app-prod-elb
现象：最近主要集中在福州，已经收到了好几个福州的用户反馈有相似的问题

### 0x02 分析

> 我们理解您的用户在使用 wifi 的情况访问您的业务超时，但使用手机移动网络就是正常的。
>
> 客户使用的 wifi 和他的移动网络可能并不是同一个网络运营商，譬如客户的 wifi 可能是中国联通的，但手机移动网络用的是中国移动的。这种情况下有可能是运营商造成的问题。
>
> 我们检查您的 ELB 运行都是正常的。为了进一步确认问题原因，不知您是否可以收集客户端更多的信息，例如，客户端使用 wifi 时的 ISP 是什么，使用的 IP 地址是什么，如果能够收集 `mtr`/`traceroute` 等 log 那就更好。

思路：

- 重新描述一遍客户提供的信息，以便确认信息的准确性；
- 提出可能的怀疑方向；
- 提供自身进行的检测和结论；
- 提供需要用户配合完成的测试（指明工具和使用方法）；

要点：

- 客户端使用 wifi 时的 ISP 是什么？
- 使用的 IP 地址是什么？
- 收集 `mtr`/`traceroute` 等 log

### 0x03 信息补充

一些遇到问题的终端用户的 IP 地址：

> 可能是通过 https://www.ipip.net/ip.html 查得的

| 地点 | ISP | IP |
| -- | -- | -- |
| 福建省福州市 | 铁通 | 111.142.134.165|
| 福建省三明市 | 铁通 | 111.142.73.78 |
| 福建省福州市 | 铁通 | 111.146.32.113 |
| 福建省南平市 | 铁通 | 111.128.16.139 |

### 0x04 分析

> 我们的 network 团队确认此**问题是由于终端用户使用的铁通运营商有问题导致**。我们检测**从 AWS 到这些终端用户的 IP 地址都有比较严重的丢包现象**：

```
# mtr --tcp --port 22 -r -c 100 111.128.16.139
Start: Mon Sep 17 13:44:26 2018
HOST: ip-10-0-1-117.cn-north-1.co Loss% Snt Last Avg Best Wrst StDev
1.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
2.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
3.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0


4.|-- 100.65.0.129       0.0% 100 0.5 0.7 0.4 4.6 0.6
5.|-- 54.222.0.139       0.0% 100 3.2 2.1 1.4 6.5 0.5
6.|-- 54.222.24.167      0.0% 100 5.8 7.6 1.4 64.6 6.5
7.|-- 54.222.0.162       0.0% 100 3.9 2.2 1.4 5.3 0.6
8.|-- 124.65.226.133     0.0% 100 3.7 10.2 3.6 79.5 17.5
9.|-- 202.96.13.117      52.0% 100 3.8 5.6 3.5 7.6 1.0
10.|-- 124.65.194.81     0.0% 100 4.6 16.3 4.3 1021. 101.6
11.|-- 219.158.101.22    0.0% 100 4.9 5.1 4.2 37.8 3.3
12.|-- 202.97.10.236     0.0% 100 3.6 3.8 3.5 4.2 0.0
13.|-- 61.237.127.229    0.0% 100 5.4 7.0 4.8 9.8 1.1
14.|-- 61.237.121.70     0.0% 100 22.6 22.2 21.6 30.6 1.3
15.|-- 61.237.126.106    0.0% 100 36.4 37.1 36.0 53.8 2.7


16.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0
17.|-- ??? 100.0 100 0.0 0.0 0.0 0.0 0.0


18.|-- 218.207.222.54    86.0% 100 56.6 56.6 56.2 57.3 0.0
19.|-- 218.207.222.54    51.0% 100 57.0 325.0 50.3 7159. 1109.4
20.|-- 218.207.222.166   35.7% 98 7961. 1378. 50.0 7961. 2182.7
21.|-- 112.5.255.89      42.9% 98 1898. 2238. 49.9 8047. 2209.8
22.|-- 112.5.255.89      51.5% 97 8205. 2039. 49.8 8205. 2858.6
23.|-- 112.5.255.94      45.7% 94 8154. 2395. 49.8 8154. 2783.4
24.|-- 112.5.255.94      45.7% 94 3096. 1323. 49.4 8077. 2423.2
25.|-- 112.5.255.89      53.8% 93 1051. 2250. 49.9 7975. 2384.0
26.|-- 112.5.255.89      46.2% 93 3090. 1304. 50.0 8015. 2437.6
27.|-- 112.5.255.94      64.1% 92 7988. 2722. 49.8 8138. 2828.2
28.|-- 112.5.255.89      46.7% 92 7349. 1136. 49.6 8178. 2364.1
29.|-- 211.143.158.250   57.8% 90 3966. 2808. 49.8 8162. 3129.5
30.|-- 112.5.255.94      53.3% 90 1081. 812.9 49.7 8083. 1734.7
```

> 通过 https://bgp.he.net/ 可以查到，**111.142.134.165** 和 **61.237.126.106** 是**中国铁通**（`AS9394`）的地址，而 **218.207.222.54** 和下面的 **112.5.255.89** 等地址都是**中国移动**（`AS9808`）的地址。从这里看，AWS 发出的数据报已经到达了**目的地址**的网内（`AS9394`），但铁通又将数据报转发给了**中国移动**（`AS9808`），并最终路由不通。
> 
> 我们**怀疑是铁通内部的路由配置有误导致的这个问题**。由于我们与 `AS9394` 的网络运营商没有直连渠道，我们建议您让您的终端客户向 `AS9394` ，即这些客户使用的运营商“**福建铁通**”投诉，以修复这个问题。

思路：

- 将 mtr 的输出分为几个段（上面已经通过空行进行了拆分）
- 确定路由转发的走向（基于 IP 查 AS，结合 Loss% 信息）

要点：

- 通过 mtr 检测从 AWS 到终端用户的 IP 地址是否有比较严重的丢包现象；
- 要能读懂 mtr 输出信息的含义；
- 要能理解 mtr 测试参数的选择（--tcp --port 22）；
- 要了解如何使用 https://bgp.he.net/ 这个网站上提供的信息；
- 要能知道 ASxxxx 对应的 ISP 是谁，要能理解不同 ISP 互通时可能存在的问题；



----------



# AS 相关

> 以下信息通过 https://bgp.he.net/ 查得

| Origin AS (ASN 数据) | Announcement (CIDR | Description |
| -- | -- | -- |
| AS55960 | 54.222.0.0/19 | Beijing Guanghuan Xinwang Digital Technology co.Ltd. |
| AS4808 | 124.65.0.0/16 <br> 124.65.192.0/18 | China Unicom Beijing province network |
| AS4808 | 202.96.13.0/24 | China National Instruments Import & Export Corp |
| AS4837 | 219.158.96.0/19 | CNC group |
| AS4134 | 202.97.0.0/19 | CHINANET backbone network |
| AS9394 | 61.236.0.0/15 <br> 61.237.0.0/16	 <br> 61.237.0.0/17 | China TieTong Telecommunications Corporation |
| AS9808 | 218.200.0.0/13 <br> 218.204.0.0/14 <br> 218.206.0.0/15 <br> 218.207.192.0/19 <br><br> 112.0.0.0/10 <br> 112.5.0.0/16 <br><br> 211.136.0.0/13 <br> 211.140.0.0/14	<br> 211.142.0.0/15 | China Mobile Communications Corporation |
| AS9808 | 211.143.144.0/20 | China Mobile Communications Corporation - fujian |
| | xx通 | |
| AS9929 | | China Netcom Backbone |
| AS9800 | | CHINA UNICOM |
| AS4538 | | China Education and Research Network Center |
| AS9306 | | China International Electronic Commerce Center |
| AS4799 | | CNCGROUP Jitong IP network |
| | 城域网 | |
| AS17623 | | China Unicom Shenzen network |
| AS17816 | | China Unicom IP network China169 Guangdong province |
| | IDC | |
| AS4816 | | China Telecom (Group) |
| AS23724 | | IDC, China Telecommunications Corporation |
| AS4835 | | China Telecom (Group) |
| | ISP | |
| AS9812 | | Oriental Cable Network Co., Ltd. |
| | 中国电信北方九省 | |
| AS17785 | | asn for Henan Provincial Net of CT |
| AS17896 | | asn for Jilin Provincial Net of CT |
| AS17923 | | asn for Neimenggu Provincial Net of CT |
| AS17897 | | asn for Heilongjiang Provincial Net of CT |
| AS17883 | | asn for Shanxi Provincial Net of CT |
| AS17799 | | asn for Liaoning Provincial Net of CT |
| AS17672 | | asn for Hebei Provincial Net of CT |
| AS17638 | | ASN for TIANJIN Provincial Net of CT |
| AS17633 | | ASN for Shandong Provincial Net of CT |
| | IXP/NAP | |
| AS4847 | | China Networks Inter-Exchange |
| AS4839 | | NAP2 at CERNET located in Shanghai |
| AS4840 | | NAP3 at CERNET located in Guangzhou |
| | CN2 | |
| AS4809 | | China Telecom Next Generation Carrier Network |
| | 教育城域网 | |
| AS9806 | | Beijing Educational Information Network Service Center Co., Ltd |
| | IPv6 Test Network | |
| AS23912 | | China Japan Joint IPv6 Test Network |
| AS9808 | | Guangdong Mobile Communication Co.Ltd. |

## Origin AS

> ref: https://www.arin.net/resources/originas.html

The **Origin Autonomous System** (`AS`) field is an optional field collected by `ARIN` during all IPv4 and IPv6 block transactions (allocation and assignment requests, reallocation and reassignment actions, transfer and experimental requests). This additional field is used by IP address block holders (including legacy address holders) to record a list of the **Autonomous System Numbers** (`ASNs`), separated by commas or whitespace, from which the addresses in the address block(s) may originate.

Collecting and making Origin AS information available to the community is part of the implementation of Policy [ARIN-2006-3: Capturing Originations in Templates](https://www.arin.net/vault/policy/proposals/2006_3.html), included in the ARIN Number Resource Policy Manual (`NRPM`) Section 3.5: "[Autonomous System Originations](https://www.arin.net/policy/nrpm.html#three5)." This information is available using our [Bulk Whois](https://www.arin.net/resources/request/bulkwhois.html) service.

## AS 相关信息

> ref: http://www.cnblogs.com/webmedia/archive/2006/01/28/324031.html


- http://www.caida.org/home/ -- 一个研究 AS 级的拓扑结构的网站，在这个网站可以找到因特网 AS 级的拓扑资料和各种分析；其分析数据的三个来源是：
    - http://www.routeviews.org/ - BGP
    - http://www.caida.org/tools/measurement/skitter/ -- RouterTrace
    - Whois
- http://www.caida.org/analysis/topology/as_core_network/historical.xml -- 全球 AS 爆炸性增长的一个直观印象