# frontend Sentry 连接时断时续问题解决

## 测试命令

基于以下命令可以对 ft sentry 发起 HTTPS 访问测试

```
curl -i -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod-ft.llsops.com/api/0/organizations/lls/releases/ --verbose
```

## 主要变更

在 liang 的帮助下，完成了如下变更调整

- 变更了原来的 ip 地址，并绑定了 EIP ；
- 将域名从 prod.ft.llsops.com 变更为 prod-ft.llsops.com ；
- 将原来的私有证书变更为公司的付费证书；
- 将 `net.ipv4.tcp_tw_reuse` 和 `net.ipv4.tcp_tw_recycle` 恢复为 0 设置；

## 有价值的结论

- ping 不是万能的，至少在这次问题上，无法进行问题的诊断；换句话说，ping 只能提供 ip 协议层面的连通性证据；针对 tcp 层面的问题，没有办法；
- 使用浏览器的“无痕模式”有助于在排查问题的时候排除一些由于浏览器自身带来的干扰因素；
- 仅在 ft sentry 机器上放开禁 ping 相关的内核参数还不够，还需要修改作用于该机器的 sg 配置才行；
- 构成 sentry 服务的几个 docker 进程的启动顺序有先后要求，否则会在 web 页面看看到报错信息（不确定对 sentry 本身影响有多大）；
- 在外网访问正常，在公司办公网络访问就不正常：
    - 服务本身没问题；
    - 很大可能是公司网络问题，或者服务所在机器的系统配置有问题；
- 域名变更会对开产生较大影响，可能导致相关方需要紧急进行发布；
- 基于 https://www.ipip.net/ 可以查到很多有用的信息；
- 有用的命令 `sudo mtr 52.80.186.66 --port 443 --tcp --no-dns`



## 信息梳理


- liang 哥：建议通过抓包确认 mtu 变更的生效情况；
- liang 哥：对比 ft sentry 机器和 phab 服务机器的 aws 配置信息，发现仅有的差异在于是否绑定了 EIP ，遂邦之（同时变更了 ip 地址）；
- liang 哥：建议将 ft sentry 使用的私有证书换成公司的付费证书；
- liang 哥：怀疑办公室内不同区域的人通过不同 ap 访问网络时，网络状况表现可能有所不同；
- liang 哥：在 aws 控制台上放开 ft sentry 的禁 ping 功能；
- zly ：建议将 nginx 配置从 HTTPS 调整回 HTTP ，以排除 HTTPS 影响（yaowen 说关闭 HTTPS 会影响发布和线上的异常收集，不能关太久）；调整后，基于 HTTP 访问问题表现依旧；
- zly ：建议修改 nginx 的配置参数 `ssl_buffer_size 1400;`；然而基于配置说明，我觉得而修改这个没有啥用处；
- zly ：我这边测试下来非常奇怪，网络都可达，两条线路都有点问题。很难说是不是服务本身的问题，感觉是和我们路由什么设置冲突了。我**在外网都访问正常，只有在公司不正常**；
- zly ：建议再搭建一套新环境进行验证；
- zly ：要我“调整外网网卡 mtu 大小”，然而经 liang 哥确认 AWS 上无法调整该配置；liang 哥认为 AWS 内外网开的 mtu 调整应该是联动的；
- zly 一直强调 ft sentry 的“有时”恢复正常是由于“我这边就是改你们的 MTU 值才能正常”；即“把你们进出的 MTU 全部切到 1460 ”；然而，后续证实并不是这样；
- zly ：按照 zly 的要求将 ft sentry 服务器 eth0 接口的 mtu 从 9001 调整为 1500 ，调整为 1460 ，调整为 1365 进行测试；
- zly ：关于 mtu 意见有，“你们是发起端，最好是在改小一点，经过路由是要加包的。所以我觉得可以从 1460 开始试”，“我请求数据，你发起数据给我们，然后你的包大了，我们这边路由就全部丢了”，“我没有设置公司的 MTU ，默认是 1500 ，但是你发过来的包是大于 1500 的”；
- zly ：建议重启 nginx，sentry，还有网卡；
- moooofly ：在公司测试启用 VPN 和断开 VPN 时访问 ft sentry 的情况；均存在时断时续的问题；
- moooofly ：基于 https://www.ipip.net/ 梳理 mtr 每一跳的路径信息；
- moooofly ：怀疑访问 ft sentry 服务时，访问路径跳数过多；后确认访问 phab 时路径也差不多，但后者没有问题；
- moooofly ：发现基于 mtr 测试时，有时几乎没有丢包，有时丢包达到 90% ；
- moooofly ：确认通过 4G 网络可以成功发布；
- moooofly ：确认在家用电信网络可以正常访问；
- moooofly ：域名变更 `prod.ft.llsops.com -> prod-ft.llsops.com` 产生阵痛（三级域名->两级域名），导致相关使用方需要发布新版以适应该变更；
- moooofly ：确认一定概率能够成功打开 sentry 网站；同时一定概率网页上会出现部分数据或资源加载失败的情况；然而，zly 认为“你能打开是因为这个被缓存了”；然而，我之前操作时，已经多次清理过浏览器上的各种缓存信息；
- moooofly ：我们的 phab 服务和 sentry 部署情况是一样的，刚才抓包看了下，tcp 连接交互的 mtu 值也是一样的；但 zly 认为“因为我现在路由已经改了啊，所以你现在都是正常的”；
- moooofly ：确认在本次问题的场景下，ping 测试虽然是正常的，但通过 ping 发现不了问题，而基于 mtr 的 tcp 测试可以发现问题；
- moooofly ：怀疑过证书存在问题，因为之前的证书是 lxs 在 letsencrypt 上搞的私有证书（支持三级域名）；后经 liang 哥建议，替换成公司的付费证书（支持二级域名）后，问题依旧；
- moooofly ：怀疑过 sentry 所在机器 CPU 资源不足，无法处理大量 SSL 连接（其实没多大量），因为该机器上只有两个逻辑 CPU ，并且最高 CPU 使用率达到过 70%~80% 左右；然而，在公司内其他服务的机器上（k8s  跳板机、harbor prod 机器、k8s 随便一个 node 上），直接访问 Sentry 的 HTTP 接口，都能够立即获得应答；
- **症状**：我们在构建脚本中会将 source map 上传作为构建的一部分，在 sentry 服务不可用时，我的 CI 一直会挂；
- **症状**：向 sentry 上传 source map 的操作也比往常慢, 而且总失败；
- zly ：怀疑 Sentry 的 web 页面加载本身存在问题，因为会出现有些资源加载成功，有些资源加载失败的情况；
- zly 坚定认为问题和 mtu 有关：
    - @孙飞(fei) 你不要折腾了。我说过你的机器是 mtu 的问题
    - 现在没问题是因为 mtu 我这边还没改，我改了以后你就会出问题
    - 。。。。。
    - 除了配置。。还有其他原因导致你的包分片大了
    - 到我这边就是不正常的。这个需要反驳嘛。
- moooofly ：我的观点，“@张龙赢(Zly) 说实话，我不认你说的是对的，因为我看到过很多服务都是那个 mtu 值，我只是没当面反驳你而已：；

## 一些调整    

### 禁 ping 调整

> - https://www.cnblogs.com/chenshoubiao/p/4781016.html
> - https://unix.stackexchange.com/questions/412446/how-to-disable-ping-response-icmp-echo-in-linux-all-the-time

- 内核参数配置

```
root@sentry-frontend-prod:~# sysctl -a|grep icmp_echo_ignore_all
net.ipv4.icmp_echo_ignore_all = 0
```

- iptables 配置

未变更前

```
root@sentry-frontend-prod:~# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy DROP)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             ip-172-17-0-2.cn-north-1.compute.internal  tcp dpt:smtp
ACCEPT     tcp  --  anywhere             ip-172-17-0-4.cn-north-1.compute.internal  tcp dpt:9000

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
root@sentry-frontend-prod:~#
```

变更为

```
root@sentry-frontend-prod:~# iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPTroot@sentry-frontend-prod:~#
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~# iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     icmp --  anywhere             anywhere             icmp echo-request

Chain FORWARD (policy DROP)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     icmp --  anywhere             anywhere             icmp echo-reply

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             ip-172-17-0-2.cn-north-1.compute.internal  tcp dpt:smtp
ACCEPT     tcp  --  anywhere             ip-172-17-0-4.cn-north-1.compute.internal  tcp dpt:9000

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
root@sentry-frontend-prod:~#
```

最后

```
root@sentry-frontend-prod:~# iptables -A INPUT -p icmp --icmp-type 8 -s 0/0 -j DROP
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~#
root@sentry-frontend-prod:~# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     icmp --  anywhere             anywhere             icmp echo-request
DROP       icmp --  anywhere             anywhere             icmp echo-request

Chain FORWARD (policy DROP)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     icmp --  anywhere             anywhere             icmp echo-reply

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             ip-172-17-0-2.cn-north-1.compute.internal  tcp dpt:smtp
ACCEPT     tcp  --  anywhere             ip-172-17-0-4.cn-north-1.compute.internal  tcp dpt:9000

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
root@sentry-frontend-prod:~#
```


----------



## 排查细节

### 将 ft sentry 改成 http 访问

```
root@sentry-frontend-prod:/etc/nginx/sites-enabled# netstat -nltp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      27253/nginx -g daem
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      10019/sshd
tcp6       0      0 :::9100                 :::*                    LISTEN      1068/node_exporter
tcp6       0      0 :::22                   :::*                    LISTEN      10019/sshd
tcp6       0      0 :::25                   :::*                    LISTEN      14391/docker-proxy
tcp6       0      0 :::9000                 :::*                    LISTEN      4136/docker-proxy
root@sentry-frontend-prod:/etc/nginx/sites-enabled#
```

即调整 nginx 的配置，不使用 SSL ；

- 基于 **ip 地址**访问

在浏览器中访问 `http://54.222.132.135/` 会**立刻给出**登录页面，点击登录会出现“CSRF 验证失败”信息；但至少**表明基于 ip 地址直接进行网络交互没有问题**；

```
...

117.131.6.70 - - [12/Apr/2018:08:35:39 +0000] "GET / HTTP/1.1" 302 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:35:39 +0000] "GET /auth/login/ HTTP/1.1" 302 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:35:39 +0000] "GET /auth/login/lls/ HTTP/1.1" 200 3487 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:35:51 +0000] "POST /auth/login/lls/ HTTP/1.1" 403 3203 "http://54.222.132.135/auth/login/lls/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"

...
```

- 基于**域名地址**访问

在浏览器中访问 `http://prod.ft.llsops.com/` 会**立刻给出**登录页面，点击登录会出现“CSRF 验证失败”信息；但至少**表明基于域名地址直接进行网络交互没有问题**；

```
...

117.131.6.70 - - [12/Apr/2018:08:34:31 +0000] "GET / HTTP/1.1" 302 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:34:31 +0000] "GET /auth/login/ HTTP/1.1" 302 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:34:31 +0000] "GET /auth/login/lls/ HTTP/1.1" 200 3484 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"
117.131.6.70 - - [12/Apr/2018:08:34:52 +0000] "POST /auth/login/lls/ HTTP/1.1" 403 3203 "http://prod.ft.llsops.com/auth/login/lls/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"

...
```

### 将 ft sentry 改成 https 访问

```
root@sentry-frontend-prod:/etc/nginx# netstat -nltp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      15716/nginx -g daem
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      10019/sshd
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      15716/nginx -g daem
tcp6       0      0 :::9100                 :::*                    LISTEN      1068/node_exporter
tcp6       0      0 :::22                   :::*                    LISTEN      10019/sshd
tcp6       0      0 :::25                   :::*                    LISTEN      14391/docker-proxy
tcp6       0      0 :::9000                 :::*                    LISTEN      4136/docker-proxy
root@sentry-frontend-prod:/etc/nginx#
```

登录界面显示的非常卡，但确实刷出来了，同时仍有部分资源没有获取成功；

点击登录按钮，过了非常长的时间后，刷出了部分界面，但仍有很多请数据获取失败；


### CPU 使用情况

发现 ft sentry 的 CPU 使用率有点高；

```
root@sentry-frontend-prod:~# top
top - 09:29:30 up 150 days, 11 min,  2 users,  load average: 1.24, 1.03, 0.71
Tasks: 154 total,   3 running, 151 sleeping,   0 stopped,   0 zombie
%Cpu(s): 76.8 us,  1.2 sy,  0.0 ni, 15.9 id,  0.0 wa,  0.0 hi,  6.1 si,  0.0 st
KiB Mem :  7657592 total,  1567976 free,  1201068 used,  4888548 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6185032 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4301 999       20   0 1150292 234372  20124 S  66.7  3.1 196:11.26 uwsgi
26762 999       20   0  343012 133328  10456 R  52.4  1.7   1:39.26 [celeryd: celer
26082 999       20   0  349912 140424  10456 R  28.6  1.8   2:18.31 [celeryd: celer
 4302 999       20   0 1145724 225044  20508 S   9.5  2.9 196:59.78 uwsgi
 4326 999       20   0  310432 113012  18168 S   9.5  1.5 435:22.28 [celeryd: celer
 4064 999       20   0  305688  95652  19984 S   2.4  1.2  23:28.63 [celery beat] r
    1 root      20   0  185180   5928   4100 S   0.0  0.1   0:45.99 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.39 kthreadd
    3 root      20   0       0      0      0 S   0.0  0.0   3:07.85 ksoftirqd/0
    5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H


root@sentry-frontend-prod:~# top
top - 09:30:18 up 150 days, 12 min,  2 users,  load average: 1.11, 1.03, 0.73
Tasks: 154 total,   3 running, 151 sleeping,   0 stopped,   0 zombie
%Cpu(s): 78.9 us,  0.0 sy,  0.0 ni, 15.5 id,  0.0 wa,  0.0 hi,  5.6 si,  0.0 st
KiB Mem :  7657592 total,  1562720 free,  1206100 used,  4888772 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6179864 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 4302 999       20   0 1145724 225044  20508 S  66.7  2.9 197:02.90 uwsgi
26762 999       20   0  348476 138072  10456 R  51.3  1.8   1:55.65 [celeryd: celer
26082 999       20   0  350680 141192  10456 R  43.6  1.8   2:34.23 [celeryd: celer
 4326 999       20   0  310432 113012  18168 S   5.1  1.5 435:24.18 [celeryd: celer
    1 root      20   0  185180   5928   4100 S   0.0  0.1   0:45.99 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.39 kthreadd
```

### 多机访问测试

- 在 ft sentry 所在机器，100% 成功


```
root@sentry-frontend-prod:~# while [ 1 ]; do curl -i -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod.ft.llsops.com/api/0/organizations/lls/releases/ --verbose; sleep 2; echo "---"; done
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 54.222.132.135...
* Connected to prod.ft.llsops.com (54.222.132.135) port 443 (#0)
* found 148 certificates in /etc/ssl/certs/ca-certificates.crt
* found 592 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
* SSL connection using TLS1.2 / ECDHE_RSA_AES_128_GCM_SHA256
* 	 server certificate verification OK
* 	 server certificate status verification SKIPPED
* 	 common name: prod.ft.llsops.com (matched)
* 	 server certificate expiration date OK
* 	 server certificate activation date OK
* 	 certificate public key: RSA
* 	 certificate version: #3
* 	 subject: CN=prod.ft.llsops.com
* 	 start date: Tue, 10 Apr 2018 11:12:11 GMT
* 	 expire date: Mon, 09 Jul 2018 11:12:11 GMT
* 	 issuer: C=US,O=Let's Encrypt,CN=Let's Encrypt Authority X3
* 	 compression: NULL
* ALPN, server accepted to use http/1.1
> GET /api/0/organizations/lls/releases/ HTTP/1.1
> Host: prod.ft.llsops.com
> Accept: */*
> Authorization: Bearer 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2
> User-Agent: Emacs Restclient
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.10.3 (Ubuntu)
Server: nginx/1.10.3 (Ubuntu)
< Date: Thu, 12 Apr 2018 10:39:26 GMT
Date: Thu, 12 Apr 2018 10:39:26 GMT
< Content-Type: application/json
Content-Type: application/json
< Content-Length: 75133
Content-Length: 75133
< Connection: close
Connection: close
< X-XSS-Protection: 1; mode=block
X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
X-Content-Type-Options: nosniff
< Content-Language: en
Content-Language: en
< Vary: Accept-Language, Cookie
Vary: Accept-Language, Cookie
< Link: <http://prod.ft.llsops.com/api/0/organizations/lls/releases/?&cursor=100:-1:1>; rel="previous"; results="false"; cursor="100:-1:1", <http://prod.ft.llsops.com/api/0/organizations/lls/releases/?&cursor=100:1:0>; rel="next"; results="true"; cursor="100:1:0"
Link: <http://prod.ft.llsops.com/api/0/organizations/lls/releases/?&cursor=100:-1:1>; rel="previous"; results="false"; cursor="100:-1:1", <http://prod.ft.llsops.com/api/0/organizations/lls/releases/?&cursor=100:1:0>; rel="next"; results="true"; cursor="100:1:0"
< Allow: GET, POST, HEAD, OPTIONS
Allow: GET, POST, HEAD, OPTIONS
< X-Frame-Options: deny
X-Frame-Options: deny
< Strict-Transport-Security: max-age=31536000
Strict-Transport-Security: max-age=31536000

<
[{"dateCreated": "2018-04-12T10:16:27.702Z", "lastEvent": null, "shortVersion": "7e39096", "authors": [], "owner": null, "newGroups": 0, "data": {}, "projects": [{"status": "active", "defaultEnvironment": "", "features": ["releases"], "color": "#3fbf51", "isPublic": false, "dateCreated": "2018-04-09T07:48:10.951Z", "platforms": ["javascript"], "callSignReviewed": true, "id": "24", "slug": "sprout-web-middle-staging", "name": "sprout-web-middle-staging", "isBookmarked": false, "callSign": "SPROUT-WEB-MIDDLE-STAGING", "firstEvent": "2018-04-09T09:35:50Z", "processingIssues": 0}], "dateReleased": null, "url": null
...

```


- 在 k8sp 跳板机上访问，100% 成功

执行

```
[deployer@k8s-deploy-prod0]$ curl -v -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod.ft.llsops.com/api/0/organizations/lls/releases/
```

立刻成功得到应答；抓包信息如下

```
[deployer@k8s-deploy-prod0]$ sudo tcpdump -i eth0 tcp port 443 -n -s 0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
11:05:42.576154 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [S], seq 472831066, win 26883, options [mss 8961,sackOK,TS val 1738522931 ecr 0,nop,wscale 7], length 0
11:05:42.576476 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [S.], seq 515429890, ack 472831067, win 26847, options [mss 1460,sackOK,TS val 3241542262 ecr 1738522931,nop,wscale 7], length 0
11:05:42.576490 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 1, win 211, options [nop,nop,TS val 1738522931 ecr 3241542262], length 0
11:05:42.627030 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [P.], seq 1:275, ack 1, win 211, options [nop,nop,TS val 1738522944 ecr 3241542262], length 274
11:05:42.627301 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], ack 275, win 219, options [nop,nop,TS val 3241542275 ecr 1738522944], length 0
11:05:42.629395 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 1:2897, ack 275, win 219, options [nop,nop,TS val 3241542275 ecr 1738522944], length 2896
11:05:42.629407 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 2897, win 256, options [nop,nop,TS val 1738522945 ecr 3241542275], length 0
11:05:42.629523 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [P.], seq 2897:3175, ack 275, win 219, options [nop,nop,TS val 3241542275 ecr 1738522944], length 278
11:05:42.629538 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 3175, win 278, options [nop,nop,TS val 1738522945 ecr 3241542275], length 0
11:05:42.630524 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [P.], seq 275:350, ack 3175, win 278, options [nop,nop,TS val 1738522945 ecr 3241542275], length 75
11:05:42.636998 IP 172.1.55.130.443 > 172.31.9.80.37796: Flags [P.], seq 3450130881:3450130914, ack 967009490, win 301, options [nop,nop,TS val 1712439641 ecr 1738515446], length 33
11:05:42.637010 IP 172.31.9.80.37796 > 172.1.55.130.443: Flags [.], ack 33, win 852, options [nop,nop,TS val 1738522947 ecr 1712439641], length 0
11:05:42.637018 IP 172.1.55.130.443 > 172.31.9.80.37796: Flags [P.], seq 33:221, ack 1, win 301, options [nop,nop,TS val 1712439641 ecr 1738515446], length 188
11:05:42.637022 IP 172.31.9.80.37796 > 172.1.55.130.443: Flags [.], ack 221, win 852, options [nop,nop,TS val 1738522947 ecr 1712439641], length 0
11:05:42.667276 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], ack 350, win 219, options [nop,nop,TS val 3241542285 ecr 1738522945], length 0
11:05:42.667296 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [P.], seq 350:401, ack 3175, win 278, options [nop,nop,TS val 1738522954 ecr 3241542285], length 51
11:05:42.667548 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], ack 401, win 219, options [nop,nop,TS val 3241542285 ecr 1738522954], length 0
11:05:42.667723 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [P.], seq 3175:3433, ack 401, win 219, options [nop,nop,TS val 3241542285 ecr 1738522954], length 258
11:05:42.668631 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [P.], seq 401:638, ack 3433, win 301, options [nop,nop,TS val 1738522954 ecr 3241542285], length 237
11:05:42.707149 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], ack 638, win 227, options [nop,nop,TS val 3241542295 ecr 1738522954], length 0
11:05:43.840975 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 3433:17913, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738522954], length 14480
11:05:43.841019 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 17913, win 527, options [nop,nop,TS val 1738523248 ecr 3241542578], length 0
11:05:43.841584 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 17913:46873, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738523248], length 28960
11:05:43.841609 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 46873, win 549, options [nop,nop,TS val 1738523248 ecr 3241542578], length 0
11:05:43.841941 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 46873:51217, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738523248], length 4344
11:05:43.842007 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 51217, win 516, options [nop,nop,TS val 1738523248 ecr 3241542578], length 0
11:05:43.842230 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 51217:77281, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738523248], length 26064
11:05:43.842254 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 77281, win 587, options [nop,nop,TS val 1738523248 ecr 3241542578], length 0
11:05:43.842516 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [.], seq 77281:78729, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738523248], length 1448
11:05:43.842661 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [FP.], seq 78729:79386, ack 638, win 227, options [nop,nop,TS val 3241542578 ecr 1738523248], length 657
11:05:43.842670 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [.], ack 79387, win 852, options [nop,nop,TS val 1738523248 ecr 3241542578], length 0
11:05:43.842920 IP 172.31.9.80.35596 > 54.222.132.135.443: Flags [P.], seq 638:669, ack 79387, win 852, options [nop,nop,TS val 1738523248 ecr 3241542578], length 31
11:05:43.843185 IP 54.222.132.135.443 > 172.31.9.80.35596: Flags [R], seq 515509277, win 0, length 0
^C
33 packets captured
33 packets received by filter
0 packets dropped by kernel
[deployer@k8s-deploy-prod0]$
```


- 在 sentry-prod 上访问，100% 成功

执行

```
root@sentry-prod:~# curl -v -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod.ft.llsops.com/api/0/organizations/lls/releases/
```

立刻成功得到应答；抓包信息如下

```
root@sentry-prod:~# tcpdump -i ens3 tcp port 443 -n -s 0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens3, link-type EN10MB (Ethernet), capture size 262144 bytes



11:10:44.434482 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [S], seq 4232568364, win 26883, options [mss 8961,sackOK,TS val 3565601404 ecr 0,nop,wscale 7], length 0
11:10:44.434730 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [S.], seq 132519421, ack 4232568365, win 26847, options [mss 1460,sackOK,TS val 3241617728 ecr 3565601404,nop,wscale 7], length 0
11:10:44.434756 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 1, win 211, options [nop,nop,TS val 3565601405 ecr 3241617728], length 0
11:10:44.486966 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [P.], seq 1:275, ack 1, win 211, options [nop,nop,TS val 3565601418 ecr 3241617728], length 274
11:10:44.487207 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], ack 275, win 219, options [nop,nop,TS val 3241617741 ecr 3565601418], length 0
11:10:44.488672 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 1:2897, ack 275, win 219, options [nop,nop,TS val 3241617741 ecr 3565601418], length 2896
11:10:44.488699 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 2897, win 350, options [nop,nop,TS val 3565601418 ecr 3241617741], length 0
11:10:44.488805 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [P.], seq 2897:3175, ack 275, win 219, options [nop,nop,TS val 3241617741 ecr 3565601418], length 278
11:10:44.488824 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 3175, win 373, options [nop,nop,TS val 3565601418 ecr 3241617741], length 0
11:10:44.489885 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [P.], seq 275:350, ack 3175, win 373, options [nop,nop,TS val 3565601418 ecr 3241617741], length 75
11:10:44.528652 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], ack 350, win 219, options [nop,nop,TS val 3241617752 ecr 3565601418], length 0
11:10:44.528673 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [P.], seq 350:401, ack 3175, win 373, options [nop,nop,TS val 3565601428 ecr 3241617752], length 51
11:10:44.528901 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], ack 401, win 219, options [nop,nop,TS val 3241617752 ecr 3565601428], length 0
11:10:44.529021 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [P.], seq 3175:3433, ack 401, win 219, options [nop,nop,TS val 3241617752 ecr 3565601428], length 258
11:10:44.530100 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [P.], seq 401:638, ack 3433, win 396, options [nop,nop,TS val 3565601428 ecr 3241617752], length 237
11:10:44.568649 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], ack 638, win 227, options [nop,nop,TS val 3241617762 ecr 3565601428], length 0
11:10:45.600270 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 3433:7777, ack 638, win 227, options [nop,nop,TS val 3241618019 ecr 3565601428], length 4344
11:10:45.600300 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 7777, win 535, options [nop,nop,TS val 3565601696 ecr 3241618019], length 0
11:10:45.600335 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 7777:17913, ack 638, win 227, options [nop,nop,TS val 3241618019 ecr 3565601428], length 10136
11:10:45.600350 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 17913, win 694, options [nop,nop,TS val 3565601696 ecr 3241618019], length 0
11:10:45.600658 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 17913:23705, ack 638, win 227, options [nop,nop,TS val 3241618019 ecr 3565601696], length 5792
11:10:45.600671 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 23705, win 834, options [nop,nop,TS val 3565601696 ecr 3241618019], length 0
11:10:45.600710 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 23705:26601, ack 638, win 227, options [nop,nop,TS val 3241618019 ecr 3565601696], length 2896
11:10:45.600722 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 26601, win 852, options [nop,nop,TS val 3565601696 ecr 3241618019], length 0
11:10:45.600745 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 26601:45425, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 18824
11:10:45.600778 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 45425:46873, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 1448
11:10:45.601230 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 46873, win 755, options [nop,nop,TS val 3565601696 ecr 3241618020], length 0
11:10:45.601490 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 46873:54113, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 7240
11:10:45.601505 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 54113, win 763, options [nop,nop,TS val 3565601696 ecr 3241618020], length 0
11:10:45.601558 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 54113:72937, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 18824
11:10:45.601580 IP 172.31.2.99.51254 > 54.222.132.135.443: Flags [.], ack 72937, win 771, options [nop,nop,TS val 3565601696 ecr 3241618020], length 0
11:10:45.601727 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 72937:77281, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 4344
11:10:45.601793 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [.], seq 77281:78729, ack 638, win 227, options [nop,nop,TS val 3241618020 ecr 3565601696], length 1448
11:10:45.602494 IP 54.222.132.135.443 > 172.31.2.99.51254: Flags [R], seq 132598808, win 0, length 0



^C
34 packets captured
37 packets received by filter
3 packets dropped by kernel
root@sentry-prod:~#
```


- 在家里电信网访问，100% 成功


```
                                      My traceroute  [vUNKNOWN]
sunfeideMacBook-Pro.local (192.168.1.5)                                     2018-04-15T23:01:24+0800
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                                            Packets               Pings
 Host                                                     Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1                                            0.0%   127    1.9  53.9   1.3 1031. 217.1
 2. 180.156.200.1                                          0.0%   127   12.4  22.9   4.8 1044.  92.0
 3. 61.152.8.41                                           48.4%   127   11.7  12.5   5.7  36.7   5.4
    61.152.8.45
 4. 101.95.120.86                                         20.5%   127   36.9  34.9  27.3  79.2   6.8
    101.95.88.162
    101.95.120.114
    202.101.63.174
 5. ???
 6. 36.110.31.98                                          41.3%   127   49.9  34.4  27.9  53.7   6.5
    106.120.220.42
 7. 36.110.31.98                                           0.0%   127   35.0  47.7  28.5 1065.  91.5
    54.222.0.159
    54.222.0.155
    106.120.220.42
    54.222.0.145
    54.222.0.151
    54.222.24.123
    54.222.0.149
 8. 54.222.24.127                                          0.0%   127   49.8  38.2  29.8  87.0   9.9
    54.222.0.214
    54.222.0.220
    54.222.0.155
    54.222.24.125
    54.222.0.218
    54.222.0.159
    54.222.0.210
 9. 54.222.1.35                                            0.0%   127   36.5  34.2  29.9  74.8   5.5
    54.222.25.7
    54.222.24.194
    54.222.24.190
    54.222.0.129
    54.222.25.13
    54.222.24.184
    54.222.0.220
10. 54.222.0.129                                          62.2%   127   52.3  37.0  30.1  69.5   7.6
    54.222.25.15
    54.222.1.91
    54.222.25.7
    54.222.25.13
    54.222.25.9
    54.222.1.35
    54.222.0.135
11. ???
12. ???
13. 52.80.186.66                                          38.9%   126   40.3  34.7  29.3  46.8   4.1

```

可以看到，有一定的丢包，但不影响访问（说明此时 TCP 层面能够自行解决问题）；


```
➜  ~ ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1): 56 data bytes
64 bytes from 192.168.1.1: icmp_seq=0 ttl=64 time=1.867 ms
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=5.133 ms
^C
--- 192.168.1.1 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 1.867/3.500/5.133/1.633 ms
➜  ~
➜  ~ ping 180.156.200.1
PING 180.156.200.1 (180.156.200.1): 56 data bytes
64 bytes from 180.156.200.1: icmp_seq=0 ttl=254 time=4.286 ms
64 bytes from 180.156.200.1: icmp_seq=1 ttl=254 time=5.736 ms
^C
--- 180.156.200.1 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 4.286/5.011/5.736/0.725 ms
➜  ~
➜  ~
➜  ~ ping 61.152.8.41
PING 61.152.8.41 (61.152.8.41): 56 data bytes
64 bytes from 61.152.8.41: icmp_seq=0 ttl=253 time=8.263 ms
64 bytes from 61.152.8.41: icmp_seq=1 ttl=253 time=10.052 ms
^C
--- 61.152.8.41 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 8.263/9.157/10.052/0.895 ms
➜  ~
➜  ~
➜  ~ ping 61.152.8.45
PING 61.152.8.45 (61.152.8.45): 56 data bytes
64 bytes from 61.152.8.45: icmp_seq=0 ttl=253 time=17.761 ms
64 bytes from 61.152.8.45: icmp_seq=1 ttl=253 time=13.109 ms
64 bytes from 61.152.8.45: icmp_seq=2 ttl=253 time=11.596 ms
^C
--- 61.152.8.45 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 11.596/14.155/17.761/2.623 ms
➜  ~
➜  ~
➜  ~ ping 101.95.120.86
PING 101.95.120.86 (101.95.120.86): 56 data bytes
64 bytes from 101.95.120.86: icmp_seq=0 ttl=252 time=33.050 ms
64 bytes from 101.95.120.86: icmp_seq=1 ttl=252 time=29.003 ms
^C
--- 101.95.120.86 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 29.003/31.026/33.050/2.023 ms
➜  ~
➜  ~
➜  ~ ping 101.95.88.162
PING 101.95.88.162 (101.95.88.162): 56 data bytes
64 bytes from 101.95.88.162: icmp_seq=0 ttl=252 time=33.131 ms
64 bytes from 101.95.88.162: icmp_seq=1 ttl=252 time=36.684 ms
^C
--- 101.95.88.162 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 33.131/34.907/36.684/1.776 ms
➜  ~
➜  ~
➜  ~ ping 101.95.120.114
PING 101.95.120.114 (101.95.120.114): 56 data bytes
64 bytes from 101.95.120.114: icmp_seq=0 ttl=252 time=38.102 ms
64 bytes from 101.95.120.114: icmp_seq=1 ttl=252 time=43.909 ms
64 bytes from 101.95.120.114: icmp_seq=2 ttl=252 time=38.558 ms
^C
--- 101.95.120.114 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 38.102/40.190/43.909/2.637 ms
➜  ~
➜  ~
➜  ~ ping 202.101.63.174
PING 202.101.63.174 (202.101.63.174): 56 data bytes
64 bytes from 202.101.63.174: icmp_seq=0 ttl=252 time=34.164 ms
64 bytes from 202.101.63.174: icmp_seq=1 ttl=252 time=36.650 ms
64 bytes from 202.101.63.174: icmp_seq=2 ttl=252 time=41.510 ms
^C
--- 202.101.63.174 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 34.164/37.441/41.510/3.051 ms
➜  ~
➜  ~
➜  ~
➜  ~ ping 36.110.31.98
PING 36.110.31.98 (36.110.31.98): 56 data bytes
64 bytes from 36.110.31.98: icmp_seq=0 ttl=59 time=32.764 ms
64 bytes from 36.110.31.98: icmp_seq=1 ttl=59 time=31.669 ms
^C
--- 36.110.31.98 ping statistics ---
3 packets transmitted, 2 packets received, 33.3% packet loss
round-trip min/avg/max/stddev = 31.669/32.217/32.764/0.547 ms
➜  ~
➜  ~
➜  ~ ping 106.120.220.42
PING 106.120.220.42 (106.120.220.42): 56 data bytes
64 bytes from 106.120.220.42: icmp_seq=0 ttl=59 time=57.828 ms
64 bytes from 106.120.220.42: icmp_seq=1 ttl=59 time=36.133 ms
^C
--- 106.120.220.42 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 36.133/46.981/57.828/10.847 ms
➜  ~
➜  ~
➜  ~ ping 36.110.31.98
PING 36.110.31.98 (36.110.31.98): 56 data bytes
64 bytes from 36.110.31.98: icmp_seq=0 ttl=59 time=29.341 ms
64 bytes from 36.110.31.98: icmp_seq=1 ttl=59 time=35.632 ms
^C
--- 36.110.31.98 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 29.341/32.486/35.632/3.145 ms
➜  ~
➜  ~
➜  ~ ping 54.222.0.159
PING 54.222.0.159 (54.222.0.159): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
^C
--- 54.222.0.159 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
➜  ~
➜  ~
➜  ~ ping 54.222.0.155
PING 54.222.0.155 (54.222.0.155): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
^C
--- 54.222.0.155 ping statistics ---
3 packets transmitted, 0 packets received, 100.0% packet loss
➜  ~
➜  ~
➜  ~ ping 106.120.220.42
PING 106.120.220.42 (106.120.220.42): 56 data bytes
64 bytes from 106.120.220.42: icmp_seq=0 ttl=59 time=34.939 ms
64 bytes from 106.120.220.42: icmp_seq=1 ttl=59 time=33.937 ms
64 bytes from 106.120.220.42: icmp_seq=2 ttl=59 time=41.699 ms
^C
--- 106.120.220.42 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 33.937/36.858/41.699/3.447 ms
➜  ~
➜  ~
➜  ~ ping 54.222.24.127
PING 54.222.24.127 (54.222.24.127): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
^C
--- 54.222.24.127 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
➜  ~
➜  ~
➜  ~ ping 54.222.1.35
PING 54.222.1.35 (54.222.1.35): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
^C
--- 54.222.1.35 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
➜  ~
➜  ~
➜  ~ ping 54.222.0.129
PING 54.222.0.129 (54.222.0.129): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
^C
--- 54.222.0.129 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss
➜  ~
➜  ~
➜  ~
➜  ~ ping 52.80.186.66
PING 52.80.186.66 (52.80.186.66): 56 data bytes
64 bytes from 52.80.186.66: icmp_seq=0 ttl=52 time=29.509 ms
64 bytes from 52.80.186.66: icmp_seq=1 ttl=52 time=38.831 ms
64 bytes from 52.80.186.66: icmp_seq=2 ttl=52 time=30.067 ms
64 bytes from 52.80.186.66: icmp_seq=3 ttl=52 time=30.488 ms
64 bytes from 52.80.186.66: icmp_seq=4 ttl=52 time=30.683 ms
^C
--- 52.80.186.66 ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 29.509/31.916/38.831/3.481 ms
➜  ~
➜  ~
```

小结：

- 通过 mtr 测试到目的 ip 的网络情况，可以得到路径上每一跳的 ip 信息；
- 此时通过 ping 测试每一跳上的 ip 地址，可以梳理 TTL 的变化规律；
    - 不是所有的网络设备都对 ping 或 tcp 测试有回应；
    - 特定地区内设备输出的 TTL 成逐级递减规律；


## 截图留存

### ft sentry 访问测试

- 最开始在公司网络中基于 mtr 进行网络情况监测的截图

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E6%9C%80%E5%BC%80%E5%A7%8B%E5%9F%BA%E4%BA%8E%20mtr%20%E8%BF%9B%E8%A1%8C%E7%BD%91%E7%BB%9C%E6%83%85%E5%86%B5%E7%9B%91%E6%B5%8B%E7%9A%84%E6%88%AA%E5%9B%BE.png)

- ft sentry 访问测试 - 多机器测试对比

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E5%A4%9A%E6%9C%BA%E5%99%A8%E6%B5%8B%E8%AF%95%E5%AF%B9%E6%AF%94.png)

- ft sentry 访问测试 - 变更 ip + 绑定 eip

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E5%8F%98%E6%9B%B4%20ip%20%E7%BB%91%E5%AE%9A%20eip.png)

- ft sentry 访问测试 - 证书调整

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E8%AF%81%E4%B9%A6%E8%B0%83%E6%95%B4.png)

- ft sentry 访问测试 -  变更 ip + 绑定 eip + 证书调整 - 一段时间后

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E7%BB%91%E5%AE%9A%20eip%20%E8%AF%81%E4%B9%A6%E8%B0%83%E6%95%B4%E4%B8%80%E6%AE%B5%E6%97%B6%E9%97%B4%E5%90%8E.png)

- ft sentry 访问测试 - 在家中通过电信网络访问情况 - 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E5%9C%A8%E5%AE%B6%E7%94%B5%E4%BF%A1%E7%BD%91%E7%BB%9C%E8%AE%BF%E9%97%AE%E6%83%85%E5%86%B5%20-%201.png)

- ft sentry 访问测试 - 在家中通过电信网络访问情况 - 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%AE%BF%E9%97%AE%E6%B5%8B%E8%AF%95%20-%20%E5%9C%A8%E5%AE%B6%E7%94%B5%E4%BF%A1%E7%BD%91%E7%BB%9C%E8%AE%BF%E9%97%AE%E6%83%85%E5%86%B5%20-%202.png)

### mtu 相关

- ldl 认为抓包中的 1514 是 mtu 值 - 呵呵1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ldl%20%E8%AE%A4%E4%B8%BA%E6%8A%93%E5%8C%85%E4%B8%AD%E7%9A%84%201514%20%E6%98%AF%20mtu%20%E5%80%BC%20-%20%E5%91%B5%E5%91%B51.png)

- ldl 认为抓包中的 1514 是 mtu 值 - 呵呵2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ldl%20%E8%AE%A4%E4%B8%BA%E6%8A%93%E5%8C%85%E4%B8%AD%E7%9A%84%201514%20%E6%98%AF%20mtu%20%E5%80%BC%20-%20%E5%91%B5%E5%91%B52.png)

- ldl 认为抓包中的 1514 是 mtu 值 - 呵呵3

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ldl%20%E8%AE%A4%E4%B8%BA%E6%8A%93%E5%8C%85%E4%B8%AD%E7%9A%84%201514%20%E6%98%AF%20mtu%20%E5%80%BC%20-%20%E5%91%B5%E5%91%B53.png)

- mtu 数值调整 - 最初为 9001 时的情况

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/mtu%20%E6%95%B0%E5%80%BC%E8%B0%83%E6%95%B4%20-%20%E6%9C%80%E5%88%9D%E4%B8%BA%209001%20%E6%97%B6%E7%9A%84%E6%83%85%E5%86%B5.png)

- mtu 数值调整 - 按 zly 要求调整为 1500

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/mtu%20%E6%95%B0%E5%80%BC%E8%B0%83%E6%95%B4%20-%20%E6%8C%89%20zly%20%E8%A6%81%E6%B1%82%E8%B0%83%E6%95%B4%E4%B8%BA%201500.png)

- mtu 数值调整 - 按 zly 要求调整为 1460

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/mtu%20%E6%95%B0%E5%80%BC%E8%B0%83%E6%95%B4%20-%20%E6%8C%89%20zly%20%E8%A6%81%E6%B1%82%E8%B0%83%E6%95%B4%E4%B8%BA%201460.png)

- mtu 数值调整 - 按 zly 要求调整为 1365

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/mtu%20%E6%95%B0%E5%80%BC%E8%B0%83%E6%95%B4%20-%20%E6%8C%89%20zly%20%E8%A6%81%E6%B1%82%E8%B0%83%E6%95%B4%E4%B8%BA%201365.png)

- 上午10点21一次mtr无丢包的截图

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E4%B8%8A%E5%8D%8810%E7%82%B921%E4%B8%80%E6%AC%A1mtr%E6%97%A0%E4%B8%A2%E5%8C%85%E7%9A%84%E6%88%AA%E5%9B%BE.png)

- 下午1点40一次mtr丢包百分之90的截图

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E4%B8%8B%E5%8D%881%E7%82%B940%E4%B8%80%E6%AC%A1mtr%E4%B8%A2%E5%8C%85%E7%99%BE%E5%88%86%E4%B9%8B90%E7%9A%84%E6%88%AA%E5%9B%BE.png)

### ping 测试

- zly ping 测试 - 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/zly%20ping%20%E6%B5%8B%E8%AF%95%20-%201.png)

- zly ping 测试 - 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/zly%20ping%20%E6%B5%8B%E8%AF%95%20-%202.png)

### 页面症状

#### HTTPS 相关

- 基于HTTPS协议访问 ft sentry 的域名地址 - 弹出登录界面有点慢且部分请求失败 - 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2%E6%9C%89%E7%82%B9%E6%85%A2%E4%B8%94%E9%83%A8%E5%88%86%E8%AF%B7%E6%B1%82%E5%A4%B1%E8%B4%A5%20-%201.png)

- 基于HTTPS协议访问 ft sentry 的域名地址 - 弹出登录界面有点慢且部分请求失败 - 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2%E6%9C%89%E7%82%B9%E6%85%A2%E4%B8%94%E9%83%A8%E5%88%86%E8%AF%B7%E6%B1%82%E5%A4%B1%E8%B4%A5%20-%202.png)

- 基于HTTPS协议访问 ft sentry 的域名地址 - 弹出登录界面有点慢且部分请求失败 - 3

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2%E6%9C%89%E7%82%B9%E6%85%A2%E4%B8%94%E9%83%A8%E5%88%86%E8%AF%B7%E6%B1%82%E5%A4%B1%E8%B4%A5%20-%203.png)

- 基于HTTPS协议访问 ft sentry 的域名地址 - 弹出登录界面有点慢且部分请求失败 - 4

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2%E6%9C%89%E7%82%B9%E6%85%A2%E4%B8%94%E9%83%A8%E5%88%86%E8%AF%B7%E6%B1%82%E5%A4%B1%E8%B4%A5%20-%204.png)

- 基于HTTPS协议访问 ft sentry 的域名地址 - 弹出登录界面有点慢且部分请求失败 - 5

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2%E6%9C%89%E7%82%B9%E6%85%A2%E4%B8%94%E9%83%A8%E5%88%86%E8%AF%B7%E6%B1%82%E5%A4%B1%E8%B4%A5%20-%205.png)

- 基于HTTPS协议访问 ft sentry 的域名地址 -进入主页面后部分资源加载失败 - 6

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%E8%BF%9B%E5%85%A5%E4%B8%BB%E9%A1%B5%E9%9D%A2%E5%90%8E%E9%83%A8%E5%88%86%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD%E5%A4%B1%E8%B4%A5%20-%206.png)

- 基于HTTPS访问 ft sentry 的ip时更容易表现为不正常

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTPS%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84ip%E6%97%B6%E6%9B%B4%E5%AE%B9%E6%98%93%E8%A1%A8%E7%8E%B0%E4%B8%BA%E4%B8%8D%E6%AD%A3%E5%B8%B8.png)

#### HTTP 相关

- 基于HTTP协议访问 ft sentry 的ip地址 - 立刻弹出登录界面

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTP%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84ip%E5%9C%B0%E5%9D%80%20-%20%E7%AB%8B%E5%88%BB%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2.png)

- 基于HTTP协议访问 ft sentry 的ip地址 - 点击登录后触发CSRF错误

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTP%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84ip%E5%9C%B0%E5%9D%80%20-%20%E7%82%B9%E5%87%BB%E7%99%BB%E5%BD%95%E5%90%8E%E8%A7%A6%E5%8F%91CSRF%E9%94%99%E8%AF%AF.png)

- 基于HTTP协议访问 ft sentry 的域名地址 - 立刻弹出登录界面

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTP%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E7%AB%8B%E5%88%BB%E5%BC%B9%E5%87%BA%E7%99%BB%E5%BD%95%E7%95%8C%E9%9D%A2.png)

- 基于HTTP协议访问 ft sentry 的域名地址 - 点击登录后触发CSRF错误

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTP%E5%8D%8F%E8%AE%AE%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84%E5%9F%9F%E5%90%8D%E5%9C%B0%E5%9D%80%20-%20%E7%82%B9%E5%87%BB%E7%99%BB%E5%BD%95%E5%90%8E%E8%A7%A6%E5%8F%91CSRF%E9%94%99%E8%AF%AF.png)

- 基于HTTP访问 ft sentry 的ip时更容易表现为正常

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8EHTTP%E8%AE%BF%E9%97%AE%20ft%20sentry%20%E7%9A%84ip%E6%97%B6%E6%9B%B4%E5%AE%B9%E6%98%93%E8%A1%A8%E7%8E%B0%E4%B8%BA%E6%AD%A3%E5%B8%B8.png)

### 调整 tw_reuse 和 tw_recycle 配置

- ft sentry 调整 tw_reuse 和 tw_recycle 配置前 - 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%B0%83%E6%95%B4%20tw_reuse%20%E5%92%8C%20tw_recycle%20%E9%85%8D%E7%BD%AE%E5%89%8D%20-%201.png)

- ft sentry 调整 tw_reuse 和 tw_recycle 配置前 - 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%B0%83%E6%95%B4%20tw_reuse%20%E5%92%8C%20tw_recycle%20%E9%85%8D%E7%BD%AE%E5%89%8D%20-%202.png)

- ft sentry 调整 tw_reuse 和 tw_recycle 配置前 - 3

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%B0%83%E6%95%B4%20tw_reuse%20%E5%92%8C%20tw_recycle%20%E9%85%8D%E7%BD%AE%E5%89%8D%20-%203.png)

- ft sentry 调整 tw_reuse 和 tw_recycle 配置后

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E8%B0%83%E6%95%B4%20tw_reuse%20%E5%92%8C%20tw_recycle%20%E9%85%8D%E7%BD%AE%E5%90%8E.png)


### 其他异常

- ft sentry 服务 CPU 使用情况高点留存

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E6%9C%8D%E5%8A%A1%20CPU%20%E4%BD%BF%E7%94%A8%E6%83%85%E5%86%B5%E9%AB%98%E7%82%B9%E7%95%99%E5%AD%98.png)

- ft sentry 服务能够正常访问后出现的新问题 - 服务启动顺序导致

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/ft%20sentry%20%E6%9C%8D%E5%8A%A1%E8%83%BD%E5%A4%9F%E6%AD%A3%E5%B8%B8%E8%AE%BF%E9%97%AE%E5%90%8E%E5%87%BA%E7%8E%B0%E7%9A%84%E6%96%B0%E9%97%AE%E9%A2%98%20-%20%E6%9C%8D%E5%8A%A1%E5%90%AF%E5%8A%A8%E9%A1%BA%E5%BA%8F%E5%AF%BC%E8%87%B4.png)

- sentry CSRF 验证失败问题 - 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20CSRF%20%E9%AA%8C%E8%AF%81%E5%A4%B1%E8%B4%A5%E9%97%AE%E9%A2%98%20-%201.png)

- sentry CSRF 验证失败问题 - 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20CSRF%20%E9%AA%8C%E8%AF%81%E5%A4%B1%E8%B4%A5%E9%97%AE%E9%A2%98%20-%202.png)

### 证据

- 证据留存 1 - zly 对 mtu 的说法以及傲慢态度

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%201%20-%20zly%20%E5%AF%B9%20mtu%20%E7%9A%84%E8%AF%B4%E6%B3%95%E4%BB%A5%E5%8F%8A%E5%82%B2%E6%85%A2%E6%80%81%E5%BA%A6.png)

- 证据留存 2 - zly 对 mtu 的说法

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%202%20-%20zly%20%E5%AF%B9%20mtu%20%E7%9A%84%E8%AF%B4%E6%B3%95.png)

- 证据留存 3 - zly 对 mtu 的说法以及傲慢态度

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%203%20-%20zly%20%E5%AF%B9%20mtu%20%E7%9A%84%E8%AF%B4%E6%B3%95%E4%BB%A5%E5%8F%8A%E5%82%B2%E6%85%A2%E6%80%81%E5%BA%A6.png)

- 证据留存 4 - zly 对 mtu 的说法以及傲慢态度

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%204%20-%20zly%20%E5%AF%B9%20mtu%20%E7%9A%84%E8%AF%B4%E6%B3%95%E4%BB%A5%E5%8F%8A%E5%82%B2%E6%85%A2%E6%80%81%E5%BA%A6.png)

- 证据留存 5 - zly 对 ssl 的说法

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%205%20-%20zly%20%E5%AF%B9%20ssl%20%E7%9A%84%E8%AF%B4%E6%B3%95.png)

- 证据留存 6 - zly 对 ssl 的说法

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%206%20-%20zly%20%E5%AF%B9%20ssl%20%E7%9A%84%E8%AF%B4%E6%B3%95.png)

- 证据留存 7 - zly 的傲慢态度

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%207%20-%20zly%20%E7%9A%84%E5%82%B2%E6%85%A2%E6%80%81%E5%BA%A6.png)

- 证据留存 8 - 有价值的讨论

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%208%20-%20%E6%9C%89%E4%BB%B7%E5%80%BC%E7%9A%84%E8%AE%A8%E8%AE%BA.png)

- 证据留存 9 - 有价值的讨论

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%209%20-%20%E6%9C%89%E4%BB%B7%E5%80%BC%E7%9A%84%E8%AE%A8%E8%AE%BA.png)

- 证据留存 10 - 有价值的讨论以及呵呵

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%2010%20-%20%E6%9C%89%E4%BB%B7%E5%80%BC%E7%9A%84%E8%AE%A8%E8%AE%BA%E4%BB%A5%E5%8F%8A%E5%91%B5%E5%91%B5.png)

- 证据留存 11 - zly 对 mtu 的说法

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E8%AF%81%E6%8D%AE%E7%95%99%E5%AD%98%2011%20-%20zly%20%E5%AF%B9%20mtu%20%E7%9A%84%E8%AF%B4%E6%B3%95.png)