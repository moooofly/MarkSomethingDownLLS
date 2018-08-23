# front-end Sentry 访问不稳定问题调查

> task ：https://phab.llsapp.com/T50379

## 背景信息

同样的 curl 命令，有些人调用能够成功返回应答，有些人调用总是超时；后来发现，能够成功调用 curl 的人，也会概率性调用超时，即和具体 ip 没有必然关系；

需要证明访问超时的主要原因在于网络状况；

## 问题排查


失败表现

```
➜  ~ curl -i -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod.ft.llsops.com/api/0/organizations/lls/releases/ --verbose
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 54.222.132.135...
* TCP_NODELAY set



* Connection failed
* connect to 54.222.132.135 port 443 failed: Operation timed out
* Failed to connect to prod.ft.llsops.com port 443: Operation timed out
* Closing connection 0
curl: (7) Failed to connect to prod.ft.llsops.com port 443: Operation timed out
➜  ~
```

成功表现

```
➜  ~ curl -i -H Authorization\:\ Bearer\ 8ebab761f1514fc194a443b9a289acce53e73374077b40a0b20dcdfdc9ae0df2 -H User-Agent\:\ Emacs\ Restclient -XGET https\://prod.ft.llsops.com/api/0/organizations/lls/releases/ --verbose
Note: Unnecessary use of -X or --request, GET is already inferred.
*   Trying 54.222.132.135...
* TCP_NODELAY set
* Connected to prod.ft.llsops.com (54.222.132.135) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/cert.pem
  CApath: none
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
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: CN=prod.ft.llsops.com
*  start date: Feb  9 11:41:13 2018 GMT
*  expire date: May 10 11:41:13 2018 GMT
*  subjectAltName: host "prod.ft.llsops.com" matched cert's "prod.ft.llsops.com"
*  issuer: C=US; O=Let's Encrypt; CN=Let's Encrypt Authority X3
*  SSL certificate verify ok.
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
< Date: Wed, 14 Mar 2018 08:35:22 GMT
Date: Wed, 14 Mar 2018 08:35:22 GMT
< Content-Type: application/json
Content-Type: application/json
< Content-Length: 77872
Content-Length: 77872
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
[{"dateCreated": "2018-03-14T07:38:35.131Z", "lastEvent": "2018-03-14T07:47:53Z", "shortVersion": "0758ad9", "authors": [], "owner": null, "newGroups": 2, "data": {}, "projects": [{"status": "active", "defaultEnvironment": "", "features": ["releases"], "color": "#3fbf8f", "isPublic": false, "dateCreated": "2017-11-18T08:42:59.797Z", "platforms": ["javascript"], "callSignReviewed": true, "id": "6", "slug": "cchybrid-staging", "name": "cchybrid-staging", "isBookmarked": false, "callSign": "CCHYBRID-STAGING", "firstEvent": "2017-11-18T08:53:13Z", "processingIssues": 0}],
...
```

小结：

- 实验了 N 多遍，就发生了一次 timeout ，目前没结论；
- 虽然主要怀疑点还是网络问题，但是 server 侧证据很难拿到，只能看到 client 侧的 timeout ；
- 目前只能在发生问题时，持续观察+搜集现场信息；


----------


经过一段时间的尝试，抓到了如下内容

```
➜  ~ tcpdump -i en0 host 54.222.132.135 and tcp port 443 -w ft_sentry.pcap
```

> NOTE: 以下 tcp stream 的序号对应了出现的先后顺序

- tcp stream 0

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/tcp_stream_0.png)

- tcp stream 1

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/tcp_stream_1.png)

- tcp stream 2

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/tcp_stream_2.png)

- tcp stream 3/4/5

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/tcp_stream_345.png)

- tcp stream 6/7

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/tcp_stream_67.png)


整个抓包过程中，在 server 侧（sentry 所在机器）上看到的网络链接情况下（以下输出信息为一秒采样输出一次），能看出网络状况的波动：

```
...
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,26sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:63374               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,32sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,25sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,31sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,24sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,30sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                180.169.229.242:63375               timer:(on,976ms,0)
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,23sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,29sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-1 0      33805  172.31.12.235:443                180.169.229.242:63375               timer:(on,076ms,0)
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,22sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,28sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,21sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,27sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,20sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,26sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,19sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,25sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,18sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,24sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,17sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,23sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,16sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,22sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,15sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,21sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:63376               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,20sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,13sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,19sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,12sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,18sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,11sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,17sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,10sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,16sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,9.068ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,15sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,8.060ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,14sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,7.048ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,13sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,6.040ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,12sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,5.032ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,11sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,4.020ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,10sec,0)
ESTAB      0      0      172.31.12.235:443                123.153.18.200:51900               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,3.012ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,9.232ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,59sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,2.004ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,8.224ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,58sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.30.209:27257               timer:(timewait,996ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,7.216ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,57sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,6.204ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,56sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,5.196ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,55sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,4.188ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,54sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,3.180ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,53sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,2.168ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,52sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,1.160ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,51sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                180.169.229.242:41602               timer:(timewait,152ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,50sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,49sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,48sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,47sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,46sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,45sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,44sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,43sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,42sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,41sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,40sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,39sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                58.221.133.91:15562               users:(("nginx",pid=31875,fd=9)) timer:(on,088ms,0)
TIME-WAIT  0      0      172.31.12.235:443                114.242.250.217:20158               timer:(timewait,332ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,38sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                58.221.133.91:15562               timer:(timewait,012ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,37sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,36sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,35sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,34sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,33sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,32sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,31sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-1 0      815    172.31.12.235:443                180.169.229.242:63972               users:(("nginx",pid=31875,fd=9)) timer:(on,044ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,30sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,29sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                119.134.176.151:50295               timer:(on,984ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,28sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                119.134.176.151:50295               timer:(timewait,240ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,27sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,016ms,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,020ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,26sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,232ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,1.584ms,2)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,1.588ms,2)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,25sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,1.628ms,2)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,572ms,2)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,576ms,2)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,24sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,616ms,2)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,3.560ms,3)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,3.568ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,23sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,3.604ms,3)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,3.276ms,4)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,3.288ms,4)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,22sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,3.324ms,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,3.800ms,5)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,3.804ms,5)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,21sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,3.852ms,5)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35719               timer:(on,2.792ms,5)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35720               timer:(on,2.796ms,5)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,20sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,2.844ms,5)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,14sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,19sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,1.832ms,5)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,13sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,13sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,18sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,812ms,5)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,15sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,15sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,17sec,0)
SYN-RECV   0      0      172.31.12.235:443                39.182.84.105:35721               timer:(on,7.796ms,6)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,14sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,16sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,14sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,15sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,14sec,0)
ESTAB      0      2911   172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,14sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9)) timer:(on,11sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,13sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,10sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,12sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,9.548ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,11sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,8.536ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,10sec,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,7.524ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,9.004ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,6.508ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,7.988ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,5.492ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,6.972ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,4.480ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,5.960ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,3.468ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,4.948ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                61.148.244.123:38927               timer:(timewait,772ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,2.452ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,3.932ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.126.1.172:46579               timer:(timewait,688ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,1.436ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,2.916ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10)) timer:(on,420ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,1.900ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                123.153.18.200:51900               timer:(timewait,888ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:63974               timer:(timewait,104ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35719               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35720               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-1 0      32     172.31.12.235:443                39.182.84.105:35719               timer:(on,10sec,0)
FIN-WAIT-1 0      32     172.31.12.235:443                39.182.84.105:35720               timer:(on,9.888ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-1 0      32     172.31.12.235:443                39.182.84.105:35719               timer:(on,8.984ms,0)
FIN-WAIT-1 0      32     172.31.12.235:443                39.182.84.105:35720               timer:(on,8.856ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(keepalive,4.936ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(keepalive,5.032ms,0)
ESTAB      0      0      172.31.12.235:443                39.182.84.105:35721               users:(("nginx",pid=31875,fd=11))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(keepalive,3.924ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(keepalive,4.020ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,41sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(keepalive,2.908ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(keepalive,3.004ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,40sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(keepalive,1.900ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(keepalive,1.996ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,39sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(keepalive,888ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(keepalive,984ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,38sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min5sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min5sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,37sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min4sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min4sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,36sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min3sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min3sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,35sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min2sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min2sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,34sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min1sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min1sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,33sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1min,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1min,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,32sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,59sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,59sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,31sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,58sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,58sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,30sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,57sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,57sec,0)
TIME-WAIT  0      0      172.31.12.235:443                218.17.37.226:36370               timer:(timewait,288ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,29sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,56sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,56sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,28sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,55sec,0)
ESTAB      0      258    172.31.12.235:443                117.158.83.66:57399               users:(("nginx",pid=31875,fd=9)) timer:(on,244ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,55sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,27sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,54sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,54sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,26sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,53sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,53sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,25sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,52sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,52sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,24sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,51sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,51sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,23sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,50sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,50sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,22sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,49sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,49sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,21sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,48sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,48sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,20sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,47sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,47sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,19sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      258    172.31.12.235:443                117.136.68.217:22015               users:(("nginx",pid=31875,fd=9)) timer:(on,268ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,46sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,46sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,18sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                117.136.68.217:22015               timer:(timewait,096ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,45sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,45sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,17sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,44sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,44sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,16sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,43sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,43sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,15sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,42sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,42sec,0)
TIME-WAIT  0      0      172.31.12.235:443                111.41.180.244:20808               timer:(timewait,404ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,14sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,41sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,41sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,13sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,40sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,40sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,12sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,39sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,39sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,11sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,38sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,38sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,10sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,37sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,37sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,9.416ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,36sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,36sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,8.404ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,35sec,0)
SYN-RECV   0      0      172.31.12.235:443                122.191.119.19:30816               timer:(on,976ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,35sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,7.392ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49868               timer:(timewait,428ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49815               timer:(timewait,160ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,34sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,34sec,0)
TIME-WAIT  0      0      172.31.12.235:443                122.191.119.19:30817               timer:(timewait,132ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,6.380ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,33sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,33sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,5.364ms,0)
TIME-WAIT  0      0      172.31.12.235:443                110.185.151.111:7824                timer:(timewait,432ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,32sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,32sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,4.352ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,31sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,31sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,3.332ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,30sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,30sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,2.316ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,29sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,29sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,1.304ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,28sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,28sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(keepalive,292ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,27sec,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,27sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min41sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,26sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,920ms,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,920ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,26sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min40sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,25sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,1.908ms,1)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,1.908ms,1)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,25sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min39sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,24sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,896ms,1)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,896ms,1)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,24sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min38sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,23sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,3.884ms,2)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,3.884ms,2)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,23sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min37sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,22sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,3.648ms,3)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,3.644ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,22sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min36sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18542               timer:(on,840ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,20sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,3.840ms,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,3.828ms,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,21sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min35sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.828ms,1)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,19sec,0)
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18537               timer:(on,2.828ms,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
SYN-RECV   0      0      172.31.12.235:443                222.93.138.66:18536               timer:(on,2.816ms,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,20sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min34sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,18sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:49791               users:(("nginx",pid=31875,fd=9))
ESTAB      0      258    172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10)) timer:(on,8.584ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,19sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min33sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
SYN-RECV   0      0      172.31.12.235:443                180.169.229.242:49942               timer:(on,968ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,17sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,18sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min32sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49942               timer:(timewait,012ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49948               timer:(timewait,736ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,16sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49949               timer:(timewait,708ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49944               timer:(timewait,308ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49947               timer:(timewait,664ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,17sec,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49945               timer:(timewait,196ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49946               timer:(timewait,708ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min31sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,15sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,16sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min30sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,14sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,15sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min29sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,13sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,14sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min28sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,12sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,13sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                180.126.1.172:46586               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min27sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,11sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,12sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min26sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,10sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,11sec,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min25sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,9.864ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,10sec,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:49983               timer:(timewait,696ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min24sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,8.852ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,9.044ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min23sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,7.840ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,8.032ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min22sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18542               users:(("nginx",pid=31875,fd=12))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,6.828ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18537               users:(("nginx",pid=31875,fd=11))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,7.020ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.126.1.172:46587               timer:(timewait,156ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min21sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,2.572ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,5.816ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,6.008ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min20sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.556ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,4.800ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,4.992ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.126.1.172:46588               timer:(timewait,228ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min19sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,544ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,3.788ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,3.980ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min18sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,3.636ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,2.772ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,2.964ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min17sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,2.624ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50019               timer:(timewait,720ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,1.760ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,1.952ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min16sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.608ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35719               timer:(timewait,744ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35720               timer:(timewait,936ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min15sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,596ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min14sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,7.796ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min13sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,6.784ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min12sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,5.772ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min11sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,4.760ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min10sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,3.748ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min9sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,2.736ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min8sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.720ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min7sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,708ms,1)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min6sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,16sec,2)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min5sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,15sec,2)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min4sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,14sec,2)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min3sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,13sec,2)
ESTAB      0      51     172.31.12.235:443                117.136.49.104:9371                users:(("nginx",pid=31875,fd=12)) timer:(on,268ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:50085               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min2sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50093               timer:(timewait,700ms,0)
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,12sec,2)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50092               timer:(timewait,532ms,0)
TIME-WAIT  0      0      172.31.12.235:443                117.136.49.104:9371                timer:(timewait,388ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50085               timer:(timewait,272ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50087               timer:(timewait,380ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min1sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50096               timer:(timewait,076ms,0)
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,11sec,2)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50101               timer:(timewait,356ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50094               timer:(timewait,084ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50098               timer:(timewait,228ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50097               timer:(timewait,184ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50099               timer:(timewait,284ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50102               timer:(timewait,312ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50095               timer:(timewait,140ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50104               timer:(timewait,328ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50100               timer:(timewait,300ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50103               timer:(timewait,336ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50106               timer:(timewait,452ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50105               timer:(timewait,356ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1min,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,10sec,2)
SYN-RECV   0      0      172.31.12.235:443                211.94.248.166:33107               timer:(on,792ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,59sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,9.040ms,2)
SYN-RECV   0      0      172.31.12.235:443                211.94.248.166:33107               timer:(on,1.808ms,2)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,58sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,8.028ms,2)
SYN-RECV   0      0      172.31.12.235:443                211.94.248.166:33107               timer:(on,1.816ms,3)
TIME-WAIT  0      0      172.31.12.235:443                36.62.165.191:43970               timer:(timewait,100ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,57sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,7.012ms,2)
ESTAB      0      0      172.31.12.235:443                211.94.248.166:33107               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,56sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,6sec,2)
SYN-RECV   0      0      172.31.12.235:443                211.94.248.166:33076               timer:(on,900ms,0)
ESTAB      0      0      172.31.12.235:443                211.94.248.166:33107               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,55sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,4.984ms,2)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,5.168ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,22sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,54sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,3.956ms,2)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,4.140ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,21sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,53sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,2.940ms,2)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,3.124ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,20sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,51sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.924ms,2)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,2.108ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,19sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,50sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,912ms,2)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,1.096ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,18sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,49sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,32sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33076               timer:(timewait,080ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,17sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,48sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50162               timer:(timewait,316ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50167               timer:(timewait,612ms,0)
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,31sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,16sec,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50161               timer:(timewait,476ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,47sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50176               timer:(timewait,396ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50175               timer:(timewait,400ms,0)
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,30sec,3)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50171               timer:(timewait,240ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50169               timer:(timewait,120ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50173               timer:(timewait,252ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50178               timer:(timewait,356ms,0)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,15sec,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50172               timer:(timewait,220ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50179               timer:(timewait,436ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50168               timer:(timewait,076ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50180               timer:(timewait,432ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50174               timer:(timewait,292ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50170               timer:(timewait,064ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50177               timer:(timewait,544ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,46sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,29sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,14sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,45sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,28sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,13sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,44sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,27sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,12sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,43sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,26sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,11sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,42sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,25sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,10sec,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,41sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,24sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,9.752ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,40sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,23sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,8.744ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,39sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,22sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,7.728ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,38sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,21sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,6.716ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,37sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,20sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,5.704ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,36sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,19sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,4.692ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,35sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,18sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,3.680ms,0)
ESTAB      0      0      172.31.12.235:443                222.93.138.66:18536               users:(("nginx",pid=31875,fd=10))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,34sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,17sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,2.668ms,0)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,6.984ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,33sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,16sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,1.656ms,0)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,5.972ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,32sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,15sec,3)
TIME-WAIT  0      0      172.31.12.235:443                211.94.248.166:33107               timer:(timewait,648ms,0)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,4.964ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,31sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,14sec,3)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,3.948ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,30sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,13sec,3)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,2.940ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,29sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,12sec,3)
FIN-WAIT-1 0      32     172.31.12.235:443                222.93.138.66:18536               timer:(on,1.928ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,28sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,11sec,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,27sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,10sec,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,26sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,9.496ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,25sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,8.480ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,24sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,7.468ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,23sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,6.456ms,3)
ESTAB      0      2911   172.31.12.235:443                14.108.21.198:62623               users:(("nginx",pid=31875,fd=9)) timer:(on,060ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,22sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,5.440ms,3)
TIME-WAIT  0      0      172.31.12.235:443                14.108.21.198:62623               timer:(timewait,124ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,21sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,4.428ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,20sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,3.412ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,19sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,2.400ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,18sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1.384ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,17sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,372ms,3)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,16sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min5sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,15sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min4sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,14sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min3sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,13sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min2sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,12sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min1sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,11sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,1min,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,10sec,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,59sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,9.440ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,57sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,8.428ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,56sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,7.416ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,55sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,6.400ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,54sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,5.384ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,53sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,4.372ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,52sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,3.360ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,51sec,4)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,2.348ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,50sec,4)
ESTAB      0      0      172.31.12.235:443                180.154.11.147:56529               users:(("nginx",pid=31875,fd=9))
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,1.340ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,49sec,4)
TIME-WAIT  0      0      172.31.12.235:443                180.154.11.147:56529               timer:(timewait,128ms,0)
FIN-WAIT-2 0      0      172.31.12.235:443                39.182.84.105:35721               timer:(timewait,324ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,48sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,47sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,46sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,45sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,44sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,43sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,42sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,41sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,40sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,39sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,38sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,37sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,36sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,35sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,34sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,33sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,32sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,31sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,30sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,29sec,4)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,28sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,27sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,26sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:63397               users:(("nginx",pid=31875,fd=10))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,25sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,24sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,23sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,22sec,4)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50559               timer:(timewait,196ms,0)
ESTAB      0      2909   172.31.12.235:443                117.136.13.206:9824                users:(("nginx",pid=31875,fd=10)) timer:(on,500ms,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:63398               users:(("nginx",pid=31875,fd=18))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50563               timer:(timewait,280ms,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:50561               users:(("nginx",pid=31875,fd=13))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50558               timer:(timewait,140ms,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:50562               users:(("nginx",pid=31875,fd=17))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:50560               users:(("nginx",pid=31875,fd=14))
ESTAB      191    0      172.31.12.235:443                117.136.13.206:9825                users:(("nginx",pid=31875,fd=12))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,21sec,4)
ESTAB      0      258    172.31.12.235:443                117.136.13.206:9824                users:(("nginx",pid=31875,fd=10)) timer:(on,508ms,0)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50570               timer:(timewait,096ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50572               timer:(timewait,160ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50573               timer:(timewait,160ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50569               timer:(timewait,168ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50574               timer:(timewait,268ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50576               timer:(timewait,316ms,0)
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50575               timer:(timewait,284ms,0)
ESTAB      0      0      172.31.12.235:443                117.136.13.206:9825                users:(("nginx",pid=31875,fd=12))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50571               timer:(timewait,148ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,20sec,4)
ESTAB      0      0      172.31.12.235:443                117.136.13.206:9824                users:(("nginx",pid=31875,fd=10))
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
TIME-WAIT  0      0      172.31.12.235:443                180.169.229.242:50568               timer:(timewait,044ms,0)
TIME-WAIT  0      0      172.31.12.235:443                117.136.13.206:9825                timer:(timewait,2.188ms,0)
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,19sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,18sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
---
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=31876,fd=7),("nginx",pid=31875,fd=7),("nginx",pid=29492,fd=7))
LAST-ACK   1      1      172.31.12.235:443                222.93.138.66:18542               timer:(on,17sec,4)
ESTAB      0      0      172.31.12.235:443                180.169.229.242:59576               users:(("nginx",pid=31875,fd=9))
...
```

综上，可以认为是公司网络时好时坏导致的问题；
