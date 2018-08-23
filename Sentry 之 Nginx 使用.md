# Sentry 之 Nginx 使用

## 用途

Sentry 环境构建使用了 nginx ，目的如下：

- **解决端口问题**：sentry 默认使用 HTTP 9000 端口，官方不建议直接修改 sentry 来变更监听端口，因此若想使用标准的 HTTP 80 端口，最好的方式就是在 sentry 前面放一个 nginx ；
- **解决 HTTPS 问题**：sentry 官方建议的 HTTPS 方案就是基于 nginx 实现的；
- **提供 rate limiting 功能**；


## 背景

在 onpremise 的项目文件中，可以看到没有 nginx 相关内容

```
[#532#root@ubuntu-1604 /opt/apps/onpremise]$tree . -L 2
.
├── config.yml
├── data
│   ├── postgres
│   └── sentry
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── README.md
├── requirements.txt
└── sentry.conf.py

3 directories, 7 files
[#533#root@ubuntu-1604 /opt/apps/onpremise]$
```

官方文档《[Deploying Sentry with Nginx](https://github.com/getsentry/sentry/blob/994c0db48de63f0541ab92ed7e42135cbf8ac05b/docs/nginx.rst)》描述

> If you're on Ubuntu, you can simply install the `nginx-full` package which will include the required **RealIP** module. 

并提供了一份 production ready 配置样例供参考

```
http {
  # set REMOTE_ADDR from any internal proxies
  # see http://nginx.org/en/docs/http/ngx_http_realip_module.html
  # 下面四行内容决定了 remote_addr 被设置为何值
  set_real_ip_from 127.0.0.1;
  set_real_ip_from 10.0.0.0/8;
  real_ip_header X-Forwarded-For;
  real_ip_recursive on;

  # SSL configuration -- change these certs to match yours
  ssl_certificate      /etc/ssl/sentry.example.com.crt;
  ssl_certificate_key  /etc/ssl/sentry.example.com.key;

  # NOTE: These settings may not be the most-current recommended
  # defaults
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:128m;
  ssl_session_timeout 10m;

  server {
    listen   80;
    server_name sentry.example.com;

    # 匹配所有请求
    # 如果是 HTTP GET 请求，则返回 405 ，并要求重写为对应的 HTTPS 请求
    location / {
      if ($request_method = GET) {
        rewrite  ^ https://$host$request_uri? permanent;
      }
      # 405 Method Not Allowed
      #
      # A request was made of a resource using a request method not 
      # supported by that resource; for example, using GET on a form 
      # which requires data to be presented via POST, or using PUT on 
      # a read-only resource.
      return 405;
    }
  }

  server {
    listen   443 ssl;
    server_name sentry.example.com;

    proxy_set_header   Host                 $http_host;
    proxy_set_header   X-Forwarded-Proto    $scheme;
    proxy_set_header   X-Forwarded-For      $remote_addr;
    proxy_redirect     off;

    # keepalive + raven.js is a disaster
    keepalive_timeout 0;

    # use very aggressive timeouts
    proxy_read_timeout 5s;
    proxy_send_timeout 5s;
    send_timeout 5s;
    resolver_timeout 5s;
    client_body_timeout 5s;

    # buffer larger messages
    client_max_body_size 5m;
    client_body_buffer_size 100k;

    location / {
      proxy_pass        http://localhost:9000;

      add_header Strict-Transport-Security "max-age=31536000";
    }
  }
}
```

针对当前的需求，以下功能是需要的：

- **REMOTE_ADDR**
- **SSL configuration**


## 线上 Nginx 安装方式

> 以下内容是基于本地虚拟机的测试

安装

```
apt install nginx-full
```

可以看到，安装之后立即启动了服务

```
[#504#root@ubuntu-1604 /opt/apps/harbor]$ps aux|grep nginx
root     32426  0.0  0.1 129484  1464 ?        Ss   13:38   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data 32427  0.0  0.3 129808  3196 ?        S    13:38   0:00 nginx: worker process
www-data 32428  0.0  0.3 129808  3196 ?        S    13:38   0:00 nginx: worker process
root     32494  0.0  0.0  12944   996 pts/0    S+   13:38   0:00 grep --color=auto --exclude-dir=.bzr} --exclude-dir=CVS} --exclude-dir=.git} --exclude-dir=.hg} --exclude-dir=.svn} nginx
[#505#root@ubuntu-1604 /opt/apps/harbor]$
```

并自动安装了如下配置文件

```
[#533#root@ubuntu-1604 /opt/apps/onpremise]$ll /etc/nginx/
total 64
drwxr-xr-x   6 root root 4096 Jan  4 13:43 ./
drwxr-xr-x 106 root root 4096 Jan  4 13:38 ../
drwxr-xr-x   2 root root 4096 Jul 12 18:34 conf.d/
-rw-r--r--   1 root root 1077 Feb 12  2017 fastcgi.conf
-rw-r--r--   1 root root 1007 Feb 12  2017 fastcgi_params
-rw-r--r--   1 root root 2837 Feb 12  2017 koi-utf
-rw-r--r--   1 root root 2223 Feb 12  2017 koi-win
-rw-r--r--   1 root root 3957 Feb 12  2017 mime.types
-rw-r--r--   1 root root 1462 Feb 12  2017 nginx.conf
-rw-r--r--   1 root root  180 Feb 12  2017 proxy_params
-rw-r--r--   1 root root  636 Feb 12  2017 scgi_params
drwxr-xr-x   2 root root 4096 Jan  4 13:38 sites-available/
drwxr-xr-x   2 root root 4096 Jan  4 13:38 sites-enabled/
drwxr-xr-x   2 root root 4096 Jan  4 13:38 snippets/
-rw-r--r--   1 root root  664 Feb 12  2017 uwsgi_params
-rw-r--r--   1 root root 3071 Feb 12  2017 win-utf
[#534#root@ubuntu-1604 /opt/apps/onpremise]$
```

此外，还可以发现新增了 `/etc/init/nginx.conf` 这个文件

```
[#535#root@ubuntu-1604 /opt/apps/onpremise]$cat /etc/init/nginx.conf
description "nginx - small, powerful, scalable web/proxy server"

start on filesystem and static-network-up
stop on runlevel [016]

expect fork
respawn

pre-start script
	[ -x /usr/sbin/nginx ] || { stop; exit 0; }
	/usr/sbin/nginx -q -t -g 'daemon on; master_process on;' || { stop; exit 0; }
end script

exec /usr/sbin/nginx -g 'daemon on; master_process on;'

pre-stop exec /usr/sbin/nginx -s quit
[#536#root@ubuntu-1604 /opt/apps/onpremise]$
```

从[这里](https://www.nginx.com/resources/wiki/start/topics/examples/ubuntuupstart/)可知，该文件是 Ubuntu 基于 **Upstart** 进行服务管理的配置文件；

> 在 Ubuntu 16.04 上已经通过 **systemd** 进行服务管理，因此该文件并没有被真正使用；因为可以发现 `initctl` 命令并不存在；

由于环境正是 Ubuntu 16.04 版本，因此

```
[#536#root@ubuntu-1604 /opt/apps/onpremise]$systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2018-01-04 13:38:45 CST; 37min ago
 Main PID: 32426 (nginx)
    Tasks: 3
   Memory: 4.6M
      CPU: 265ms
   CGroup: /system.slice/nginx.service
           ├─32426 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           ├─32427 nginx: worker process
           └─32428 nginx: worker process

Jan 04 13:38:45 ubuntu-1604 systemd[1]: Starting A high performance web server and a reverse proxy server...
Jan 04 13:38:45 ubuntu-1604 systemd[1]: Started A high performance web server and a reverse proxy server.
[#537#root@ubuntu-1604 /opt/apps/onpremise]$
```

其对应的配置文件内容如下

```
[#548#root@ubuntu-1604 /opt/apps/onpremise]$cat /lib/systemd/system/nginx.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
After=network.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
[#549#root@ubuntu-1604 /opt/apps/onpremise]$
```

可以看到两种服务管理方式下，操作内容基本相同；


至此，可以确定以下几点（通过 apt 安装 nginx-full 的情况）：

- nginx 服务基于 systemd 进行管理；
- nginx 的配置分为三部分：
    - `/etc/nginx/nginx.conf` 中提供基本配置内容；
    - `/lib/systemd/system/nginx.service` 中通过 `-g 'daemon on; master_process on;'` 设置了两个全局配置项；
    - （经和贤三确认）nginx 使用的证书是通过 certbot 创建的；
    

> 背景信息：发现 `/etc/nginx/nginx.conf` 这个配置文件中没有证书配置，经确认，实际是因为采用了 certbot 进行免费证书的创建和配置，因此就不用在 nginx 里面直接配置证书位置了；此证书和手动通过 openssl 命令创建的不同，详见 https://letsencrypt.org/ ；（贤三说）其实我们买了证书，但是配置到 elb 上比较方便。配置到 ec2 上比较麻烦，所以就用了这个；


## 线上 Sentry 配置

```
root@sentry-frontend-prod:~# cat /etc/nginx/nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
	worker_connections 768;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;


  	# set REMOTE_ADDR from any internal proxies
  	# see http://nginx.org/en/docs/http/ngx_http_realip_module.html
  	set_real_ip_from 127.0.0.1;
  	set_real_ip_from 10.0.0.0/8;
  	set_real_ip_from 172.0.0.0/8;
  	real_ip_header X-Forwarded-For;
  	real_ip_recursive on;

	##
	# SSL Settings
	##

        # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/prod.ft.llsops.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/prod.ft.llsops.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;
	gzip_disable "msie6";

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	# 这里被我直接注释掉了，相应配置内容全部放入了当前文件中
	#include /etc/nginx/sites-enabled/*;


	server {
        listen 80;
		server_name prod.ft.llsops.com;

		root /var/www/html;
		index index.html index.htm index.nginx-debian.html;

                location / {
                        if ($request_method = GET) {
                                rewrite  ^ https://$host$request_uri? permanent;
                        }
                        return 405;
                }
	}

	server {
        listen 443 ssl;
		server_name prod.ft.llsops.com;

        proxy_set_header   Host                 $http_host;
        proxy_set_header   X-Forwarded-Proto    $scheme;
        proxy_set_header   X-Forwarded-For      $remote_addr;
        proxy_redirect     off;

		keepalive_timeout 0;

		proxy_read_timeout 5s;
		proxy_send_timeout 5s;
		send_timeout 5s;
		resolver_timeout 5s;
		client_body_timeout 5s;

		# buffer larger messages
		client_max_body_size 5m;
		client_body_buffer_size 100k;

		root /var/www/html;
		index index.html index.htm index.nginx-debian.html;

		location / {
			proxy_pass http://localhost:9000;
			add_header Strict-Transport-Security "max-age=31536000";
		}
	}
}

root@sentry-frontend-prod:~#
```


## Nginx 配置说明

### 参数解析

在《[ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html)》中有

> The `ngx_http_realip_module` module is used to change the client address and optional port to those sent in the specified header field.
>
> This module is not built by default, it should be enabled with the `--with-http_realip_module` configuration parameter.

示例配置（取自上面的配置文件）

```
set_real_ip_from 127.0.0.1;
set_real_ip_from 10.0.0.0/8;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

- `set_real_ip_from`

> Syntax: `set_real_ip_from address | CIDR | unix:;`    
> Default: `—`
> 
> Defines **trusted addresses** that are known to send correct replacement addresses.
> 定义明确知道的、可信的 proxy 地址，用于帮助在 X-Forwarded-For 中确定 client 的 IP 地址；


- `real_ip_header`

> Syntax: `real_ip_header field | X-Real-IP | X-Forwarded-For | proxy_protocol;`    
> Default: `real_ip_header X-Real-IP;`
> 
> Defines the **request header** field whose value will be used to **replace** the client address. 
> 定义在哪个 Header 中确定出用于替换 client IP 地址的地址；


- `real_ip_recursive`

> Syntax: `real_ip_recursive on | off;`    
> Default: `real_ip_recursive off;`
> 
> If recursive search is **disabled**, the original client address that **matches** one of the trusted addresses is **replaced by the last address** sent in the request header field defined by the `real_ip_header` directive. If recursive search is **enabled**, the original client address that **matches** one of the trusted addresses is **replaced by the last non-trusted address** sent in the request header field.
> 用于确定使用 `real_ip_header` 指定的 Header 中的哪个值作为 client 的 IP 地址；若 disabled 则使用 **rightmost** ，若 enabled 则使用 **last non-trusted address** ；


- `$realip_remote_addr`

> keeps the original client address (1.9.7)

- `$realip_remote_port`

> keeps the original client port (1.11.0)


在《[ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)》中有

> The `ngx_http_proxy_module` module allows passing requests to another server.

示例配置（取自上面的配置文件）

```
server {
    ...
    proxy_set_header   Host                 $http_host;
    proxy_set_header   X-Forwarded-Proto    $scheme;
    proxy_set_header   X-Forwarded-For      $remote_addr;
    proxy_redirect     off;
    ...
}
```

或

```
location / {
    proxy_pass          http://localhost:8000;
    
    proxy_set_header    Host      $host;
    proxy_set_header    X-Real-IP $remote_addr;
}
```

- `proxy_set_header`

> Syntax: `proxy_set_header field value;`    
> Default: `proxy_set_header Host $proxy_host;` and `proxy_set_header Connection close;`
> 
> Allows **redefining** or **appending** fields to the request header passed to the proxied server. These directives are **inherited** from the previous level if and only if there are no `proxy_set_header` directives defined on the current level. 
> 
> - `proxy_set_header Host $http_host;` -- An unchanged “Host” request header field can be passed like this. 适用于 Host 头一定存在的情况，若不存在，则对应的头就不转发了；助解：http_host 表示 HTTP 中的 Host 头；
> - `proxy_set_header Host $host;` -- if this field (“Host” request header field) is not present in a client request header then nothing will be passed. In such a case it is better to use the `$host` variable - its value equals the server name in **the "Host" request header field** or the **primary server name** if this field is not present. 适用于 Host 头不一定存在的情况，若存在，则使用之，若不存在，则使用 nginx 中配置的 primary 虚拟主机名；
> - `proxy_set_header Host $host:$proxy_port;` -- the server name can be passed together with the port of the proxied server. 可以在 Host 头中增加 proxy 上使用的 port 信息；


- `proxy_redirect`

> Syntax: `proxy_redirect default|off|redirect replacement;`    
> Default: `proxy_redirect default`
>
> Sets the text that should be changed in the "`Location`" and "`Refresh`" header fields of a proxied server response.
> The `default` replacement: the two configurations below are equivalent
> ```
location /one/ {
    proxy_pass     http://upstream:port/two/;
    proxy_redirect default;
location /one/ {
    proxy_pass     http://upstream:port/two/;
    proxy_redirect http://upstream:port/two/ /one/;
> ```
> The `off` parameter cancels the effect of all proxy_redirect directives on the current level.


- `proxy_pass`

> Syntax: `proxy_pass URL;`    
> Default: `—`
>
> Sets the **protocol** and **address** of a proxied server and an optional URI to which a location should be mapped. 
>
> As a protocol, "`http`" or "`https`" can be specified. The address can be specified as a **domain name** or **IP address**, and an optional **port**:
> `proxy_pass http://localhost:8000/uri/`
> or as a UNIX-domain socket path specified after the word “unix” and enclosed in colons:
> `proxy_pass http://unix:/tmp/backend.socket:/uri/;`
>
> If a **domain name** resolves to several addresses, all of them will be used in a **round-robin** fashion. In addition, an address can be specified as a **server group**.


在《[ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)》中有

- `rewrite`

> Syntax: `rewrite regex replacement [flag];`    
> Default: `—`
>
> - If the specified regular expression matches a request URI, URI is changed as specified in the replacement string. 
> - The `rewrite` directives are **executed sequentially** in order of their appearance in the configuration file. 
> - It is possible to **terminate further processing** of the directives using `flags`. If a replacement string starts with "`http://`", "`https://`", or "`$scheme`", the processing **stops** and the redirect is **returned** to a client.
>
> An optional `flag` parameter can be one of:
> 
> - **last**
> **stops** processing the current set of `ngx_http_rewrite_module` directives and **starts a search for a new location** matching the changed URI;
> - **break**
> **stops** processing the current set of `ngx_http_rewrite_module` directives as with the `break` directive;
> - **redirect**
> **returns** a temporary redirect with the **302 code**; used if a replacement string does not start with "`http://`", "`https://`", or "`$scheme`";
> - **permanent**
> **returns** a permanent redirect with the **301 code**.




----------


### 网上资料

在《[Story behind X-Forwarded-For and X-Real-IP headers](https://distinctplace.com/2014/04/23/story-behind-x-forwarded-for-and-x-real-ip-headers/)》中有

使用场景

> `X-Forwarded-For` is usually **used by proxies** to carry original Client IP through intermediary hops. Otherwords each time request goes through proxy, it should add **current request IP** to the list. More details [here](https://en.wikipedia.org/wiki/X-Forwarded-For). The format should look like this pretty much:
> ```
> X-Forwarded-For: client, proxy1, proxy2
> ```

存在的问题和风险

> - There is one problem though it’s **not mandatory to use**, meaning some proxies will add the header and some will not.
> - And it’s quite **easy to spoof** as well. Let say you want to hide your real IP – to do that you can just send a request with `X-Forwarded-For: spoof` and proxy will gladly add request IP to the list. The result will look like this `X-Forwarded-For: spoof realip`. As you can see, you can’t just extract **leftmost IP**, because it **might be forged** (you also need to keep that in mind if you are using that `X-Forwarded-For` in the application for some kind of IP based logic).

知识点

> - by default Nginx will use `X-Real-IP` header (`real_ip_header`) if it’s present in the request. 
> - we have Client IP set by CDN to be `X-Real-IP`, and we should see correct IP in the logs after enabling the module.
> - `set_real_ip_from`: set addresses allowed to influence client IP change.
> - `real_ip_recursive`: 决定是否使用 rightmost 地址，或者使用除了 `set_real_ip_from` 之外的 last non-trust 地址；

[ServerFault post](https://serverfault.com/questions/314574/nginx-real-ip-header-and-x-forwarded-for-seems-wrong/414166#414166) 小结

> in order to protect yourself from IP spoof, and get REAL CLIENT IP (Morpheus: How do you define ‘real’?) you need to **enable `real_ip_recursive`** and **set known proxies using `set_real_ip_from`**. Nginx will remove IPs matching known proxies and then use **rightmost** IP which should be the IP you are looking for!
> 
> nginx was grabbing the **last IP address** in the chain by default because that was the only one that was assumed to be trusted. But with the new `real_ip_recursive` enabled and with multiple `set_real_ip_from` options, you can define multiple trusted proxies and it will fetch the **last non-trusted IP**.
> 
> For example, with this config:
>
> ```
set_real_ip_from 127.0.0.1;
set_real_ip_from 192.168.2.1;
real_ip_header X-Forwarded-For;
real_ip_recursive on;
> ```
> And an `X-Forwarded-For` header resulting in:
> 
> ```
X-Forwarded-For: 123.123.123.123, 192.168.2.1, 127.0.0.1
> ```
> nginx will now pick out 123.123.123.123 as the client's IP address.
>
> As for why nginx doesn't just pick the left-most IP address and requires you to explicitly define trusted proxies, it's to prevent easy IP spoofing.
> 
> Let's say a client's real IP address is 123.123.123.123. Let's also say the client is up to no good, and they're trying to spoof their IP address to be 11.11.11.11. They send a request to the server with this header already in place:
>
> ```
X-Forwarded-For: 11.11.11.11
> ```
> Since reverse proxies simply add IPs to this `X-Forwarded-For` chain, let's say it ends up looking like this when nginx gets to it:
>
> ```
X-Forwarded-For: 11.11.11.11, 123.123.123.123, 192.168.2.1, 127.0.0.1
> ```
> If you simply grabbed the left-most address, that would allow the client to easily spoof their IP address. But with the above example nginx config, nginx will only trust the last two addresses as proxies. This means nginx will correctly pick 123.123.123.123 as the IP address, despite that spoofed IP actually being the left-most.


----------


在[维基百科](https://en.wikipedia.org/wiki/X-Forwarded-For)上有

> The `X-Forwarded-For` (XFF) HTTP header field is a common method for identifying the originating IP address of a client connecting to a web server through an HTTP proxy or load balancer.
>
> **Without** the use of XFF or another similar technique, any connection through the proxy would **reveal only** the originating IP address of the proxy server, effectively turning the proxy server into an **anonymizing service**, thus making the detection and prevention of abusive accesses significantly harder than if the originating IP address was available. **The usefulness of XFF** depends on the proxy server truthfully reporting the original host's IP address; for this reason, effective use of XFF requires knowledge of which proxies are trustworthy, for instance by looking them up in a whitelist of servers whose maintainers can be trusted.
>
> The general format of the field is:
>
> ```
> X-Forwarded-For: client, proxy1, proxy2
> ```
> where the value is a comma+space separated list of IP addresses, the **left-most** being the **original client**, and each successive proxy that passed the request adding the IP address where it received the request from. In this example, the request passed through proxy1, proxy2, and then **proxy3** (not shown in the header). **proxy3 appears as remote address of the request**.
> 
> Since it is **easy to forge an `X-Forwarded-For` field** the given information should be used with care. The **last IP address** is always the IP address that connects to the last proxy, which means it is the most reliable source of information. `X-Forwarded-For` data can be used in a forward or reverse proxy scenario.
> 
> Just logging the `X-Forwarded-For` field is not always enough as the **last proxy IP address** in a chain is **not contained** within the `X-Forwarded-For` field, it is in the actual IP header. A web server should log BOTH the request's source IP address and the `X-Forwarded-For` field information for completeness.


----------


在《[三种在CDN环境下获取用户IP方法](http://www.ttlsa.com/nginx/nginx-get-user-real-ip/)》中有

> - CDN 自定义 header 头
> 优点：获取到最真实的用户 IP 地址，用户绝对不可能伪装 IP
> 缺点：需要 CDN 厂商提供
> 
> - 获取 forwarded-for 头
> 优点：可以获取到用户的 IP 地址
> 缺点：程序需要改动，以及用户 IP 有可能是伪装的
>
> - 使用 realip 获取
> 优点：程序不需要改动，直接使用 `remote_addr` 即可获取 IP 地址
> 缺点：IP 地址有可能被伪装，而且需要知道所有 CDN 节点的 IP 地址或者 IP 段


----------


## 问题排查

### 0x01 307 Internal Redirect

在 ROOT URL 中带有 9000 端口时，Web UI 上获取某些资源时会出现 307 ，用于从 HTTPS 降级为 HTTP 进行尝试；

> 以下内容取自[维基百科](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

- `301 Moved Permanently`

> This and all future requests should be directed to the given URI.

永久重定向；**可以变更请求方法**；

- `302 Found`

> This is an example of industry practice **contradicting** the standard. The **HTTP/1.0** specification (RFC 1945) required the client to perform a **temporary redirect** (the original describing phrase was "**Moved Temporarily**"), but popular browsers implemented 302 with the functionality of a `303 See Other`. Therefore, **HTTP/1.1** added status codes 303 and 307 to distinguish between the two behaviours. However, some Web applications and frameworks use the 302 status code as if it were the 303.

工业实现改变了标准语义的案例；
在 HTTP/1.0 中用于定义临时重定向（请求方法不变），但是一些流行浏览器将其实现成了 303 的行为（请求方法会变）；因此，在 HTTP/1.1 中增加了 303 和 307 用于区分上述请求，但是一些 web 应用和框架仍然将 302 当做 303 使用；

- `303 See Other` (since HTTP/1.1)

> The response to the request can be found under another URI using the `GET` method. When received in response to a `POST` (or `PUT`/`DELETE`), the client should presume that the server has received the data and should issue a new `GET` request to the given URI.

表示 server 已经通过 POST/PUT/DELETE 接收到了数据，但对应的应答需要 client 通过一个新 GET 请求获取；**关键在于请求的方法发生了变化**！！

- `307 Temporary Redirect` (since HTTP/1.1)

> In this case, the request should be repeated with another URI; however, future requests should still use the original URI. In contrast to how 302 was historically implemented, the request method is not allowed to be changed when reissuing the original request. For example, a POST request should be repeated using another POST request.

请求需要基于一个新 URL 重发，但后续其他请求仍旧使用之前的 URL 发送；
和最初版本的 302 实现相比，**当重发原始请求时，不允许变更请求方法**；

- `308 Permanent Redirect` (RFC 7538)

> The request and all future requests should be repeated using another URI. 307 and 308 parallel the behaviors of 302 and 301, but do not allow the HTTP method to change. So, for example, submitting a form to a permanently redirected resource may continue smoothly.

当前请求和后续所有请求都应该一个新的 URL 进行发送；
307 和 308 是 302 和 301 的等价存在，但**不允许发生 HTTP 方法的变更**；


> 以下内容取自 [HTTP 302](https://en.wikipedia.org/wiki/HTTP_302)

**这段问题描述清楚了 30x 的历史问题**！！

The HTTP response status code `302 Found` is a common way of performing **URL redirection**.

An HTTP response with this status code will additionally provide a URL in the header field `location`. The user agent (e.g. a web browser) is invited by a response with this code to make a second, otherwise identical, request to the new URL specified in the `location` field. The **HTTP/1.0** specification (RFC 1945) initially defined this code, and gives it the description phrase "**Moved Temporarily**".

Many web browsers implemented this code in a manner that **violated** this standard, changing the request type of the new request to `GET`, regardless of the type employed in the original request (e.g. `POST`).[1] For this reason, **HTTP/1.**1 (RFC 2616) added the new status codes **303** and **307** to disambiguate between the two behaviours, with 303 **mandating** the change of request type to `GET`, and 307 **preserving** the request type as originally sent. Despite the greater clarity provided by this disambiguation, the 302 code is still employed in web frameworks to preserve compatibility with browsers that do not implement the **HTTP/1.1** specification.

As a consequence, the update of RFC 2616 changes the definition to allow user agents to rewrite `POST` to `GET`.


### 0x02 HSTS

> http://www.ttlsa.com/web/hsts-for-nginx-apache-lighttpd/

HTTP 转 HTTPS 可以通过：

- 302
- HSTS


通常情况下，将用户的 HTTP 请求 302 跳转到 HTTPS 会存在两个问题：

- **不够安全**：302 跳转会暴露用户访问站点，也容易被劫持；
- **拖慢访问速度**：302 跳转需要一个 RTT ，浏览器执行跳转也需要时间；

由于 302 跳转是由浏览器触发的，服务器无法完全控制，由此导致了 **HSTS (HTTP Strict Transport Security)** 的诞生。

HTSP 的使用只需要添加 `add_header Strict-Transport-Security max-age=15768000;includeSubDomains` 这个 Header ，以便告诉浏览器针对目标网站需要使用 HTTPS 访问；而支持 HSTS 的浏览器就会在后面的请求中直接切换到 HTTPS 。

在 Chrome 浏览器中，会看到浏览器自己会有个 `307 Internal Redirect` 内部重定向。

在一段时间内，即 max-age 定义的时间内，不管用户输入 www.ttlsa.com 还是 http://www.ttlsa.com ，默认都会将请求内部跳转到 https://www.ttlsa.com 。

服务器端配置 HSTS 可以减少 302 跳转，其实 HSTS 的最大作用是防止 302 HTTP 劫持。HSTS 的缺点是浏览器支持率不高，另外配置 HSTS 后，HTTPS 很难实时降级成 HTTP 。

在 Sentry 使用的 Nginx 中进行了如下设置

```
add_header Strict-Transport-Security "max-age=31536000";
```

### 0x03 点击某些 Sentry 页面时 URL 会带 9000 端口问题

需要修改 https://prod.ft.llsops.com/manage/settings/ 中的 Root URL 内容（由 http://prod.ft.llsops.com:9000 改为 http://prod.ft.llsops.com）


### 0x04 nginx.service: Failed to read PID from file /run/nginx.pid: Invalid argument

背景信息：新搭建一个 Sentry 环境，使用 nginx 作为反向代理，但在启动 nginx 服务时，**时不时**会出现 "nginx.service: Failed to read PID from file /run/nginx.pid: Invalid argument" 这条报错信息；

```
root@sentry-staging:~# systemctl status nginx.service
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/nginx.service.d
           └─override.conf
   Active: active (running) since Tue 2018-01-30 05:55:56 UTC; 1h 30min ago
  Process: 27081 ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid (code=exited, status=0/SUCCESS)
  Process: 27096 ExecStartPost=/bin/sleep 0.1 (code=exited, status=0/SUCCESS)
  Process: 27091 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 27087 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 27095 (nginx)
    Tasks: 2
   Memory: 2.2M
      CPU: 233ms
   CGroup: /system.slice/nginx.service
           ├─27095 nginx: master process /usr/sbin/nginx -g daemon on; master_process on
           └─27098 nginx: worker process

Jan 30 05:55:55 sentry-staging systemd[1]: Starting A high performance web server and a reverse proxy server...
Jan 30 05:55:56 sentry-staging systemd[1]: Started A high performance web server and a reverse proxy server.
root@sentry-staging:~#
root@sentry-staging:~# journalctl -u nginx.service
-- Logs begin at Mon 2018-01-29 11:09:47 UTC, end at Tue 2018-01-30 07:20:10 UTC. --
Jan 30 02:39:21 sentry-staging systemd[1]: Starting A high performance web server and a reverse proxy server...
Jan 30 02:39:21 sentry-staging systemd[1]: nginx.service: Failed to read PID from file /run/nginx.pid: Invalid argument
Jan 30 02:39:21 sentry-staging systemd[1]: Started A high performance web server and a reverse proxy server.
Jan 30 03:00:22 sentry-staging systemd[1]: Reloading A high performance web server and a reverse proxy server.
Jan 30 03:00:22 sentry-staging systemd[1]: Reloaded A high performance web server and a reverse proxy server.
Jan 30 03:03:04 sentry-staging systemd[1]: Reloading A high performance web server and a reverse proxy server.
Jan 30 03:03:04 sentry-staging systemd[1]: Reloaded A high performance web server and a reverse proxy server.
Jan 30 03:04:31 sentry-staging systemd[1]: Stopping A high performance web server and a reverse proxy server...
Jan 30 03:04:31 sentry-staging systemd[1]: Stopped A high performance web server and a reverse proxy server.
Jan 30 03:04:35 sentry-staging systemd[1]: Starting A high performance web server and a reverse proxy server...
Jan 30 03:04:35 sentry-staging systemd[1]: nginx.service: Failed to read PID from file /run/nginx.pid: Invalid argument
Jan 30 03:04:35 sentry-staging systemd[1]: Started A high performance web server and a reverse proxy server.
Jan 30 03:45:42 sentry-staging systemd[1]: Stopping A high performance web server and a reverse proxy server...
Jan 30 03:45:42 sentry-staging systemd[1]: Stopped A high performance web server and a reverse proxy server.
...
```

网上查到相关资料如下：


----------


> https://www.v2ex.com/t/300986

因为 nginx 启动需要一点点时间，当发生了 systemd 在 nginx 完成启动前就去读取 pid 文件的情况，则会造成读取 pid 失败； 

解决方法：让 systemd 在执行 `ExecStart` 指令后等待一点点时间即可；如果你的 nginx 启动需要时间更长，可以把 sleep 时间改长一点；

```
mkdir -p /etc/systemd/system/nginx.service.d 
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" > /etc/systemd/system/nginx.service.d/override.conf 
```

然后 

```
systemctl daemon-reload 
systemctl restart nginx.service
```

个人观点：这种解决办法比较 tricky ，能解决问题，但不是很 nice ；


----------


> [How to fix the NGINX error “Failed to read PID from file”, quick and easy](https://www.cloudinsidr.com/content/heres-fix-nginx-error-failed-read-pid-file-linux/)

原因：This behavior is a known bug, caused by a **race condition** between `nginx` and `systemd`. **Systemd is expecting the PID file to be populated before nginx had the time to create it.** To fix the error, you have to create the file.

解决办法：

- Create the directory `/etc/systemd/system/nginx.service.d`
- Print data to file
- Reload the daemon
- Restart NGINX

```
mkdir /etc/systemd/system/nginx.service.d
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" > 
/etc/systemd/system/nginx.service.d/override.conf
systemctl daemon-reload
systemctl restart nginx
```

另一种解决办法：

- removing the `PIDFile` option and 
- adding the line: `ExecStopPost=/bin/rm -f /run/nginx.pid`


个人观点：

- 第一种办法和前文相同，问题也相同；
- 第二种办法相对来说更为合理，即停止 nginx 服务后顺便移除 pid 文件；


----------


> https://bugs.launchpad.net/ubuntu/+source/nginx/+bug/1581864

以下为 ubuntu 官方 bug 报告，关键信息点有：

- Nginx logs an error when started on a machine **with a single CPU**, Bumping the number of CPU available makes the error disappear. This is reproducible on VMs and containers.
- Confirmed on Xenial, with **1-CPU systems**, that there may be such a race condition.


完整且清晰的解释说明：

We are also affected, and I've looked into the source. In `main()`, the `ngx_create_pidfile()` gets called right after the `ngx_daemon()` function. `ngx_daemon()` of course forks, and the parent process exits immediately with `exit(0)`. This is a pure **race condition between nginx and systemd**.

This is not a problem (other than the scary log), because the `PIDFile` option is **not needed for** determining the main PID of nginx (systemd can correctly "guess" the main pid), it's **only for** removing the file when the service exits.

As far as I can tell, **this daemonization is not needed for the systemd service use-case**. The service should be `Type=simple`, and '`daemon off`'. The standard file handles get redirected by systemd anyway, and non-stop upgrade cannot be used in this case either. (See: http://nginx.org/en/docs/faq/daemon_master_process_off.html )

If this is too much of a change, **another workaround** is removing the `PIDFile` option, and adding the line: `ExecStopPost=/bin/rm -f /run/nginx.pid`


----------


## Certbot 使用

> Certbot 证书相关内容需要补全

On Ubuntu systems, the Certbot team maintains a **PPA**. Once you add it to your list of repositories all you'll need to do is `apt-get` the following packages.

```
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx 
```

安装后

```
[#576#root@ubuntu-1604 /opt/apps/onpremise]$which certbot
/usr/bin/certbot
[#577#root@ubuntu-1604 /opt/apps/onpremise]$
[#577#root@ubuntu-1604 /opt/apps/onpremise]$certbot -h

-------------------------------------------------------------------------------

  certbot [SUBCOMMAND] [options] [-d DOMAIN] [-d DOMAIN] ...

Certbot can obtain and install HTTPS/TLS/SSL certificates.  By default,
it will attempt to use a webserver both for obtaining and installing the
certificate. The most common SUBCOMMANDS and flags are:

obtain, install, and renew certificates:
    (default) run   Obtain & install a certificate in your current webserver
    certonly        Obtain or renew a certificate, but do not install it
    renew           Renew all previously obtained certificates that are near
expiry
   -d DOMAINS       Comma-separated list of domains to obtain a certificate for

  (the certbot apache plugin is not installed)
  --standalone      Run a standalone webserver for authentication
  --nginx           Use the Nginx plugin for authentication & installation
  --webroot         Place files in a server's webroot folder for authentication
  --manual          Obtain certificates interactively, or using shell script
hooks

   -n               Run non-interactively
  --test-cert       Obtain a test certificate from a staging server
  --dry-run         Test "renew" or "certonly" without saving any certificates
to disk

manage certificates:
    certificates    Display information about certificates you have from Certbot
    revoke          Revoke a certificate (supply --cert-path)
    delete          Delete a certificate

manage your account with Let's Encrypt:
    register        Create a Let's Encrypt ACME account
  --agree-tos       Agree to the ACME server's Subscriber Agreement
   -m EMAIL         Email address for important account notifications

More detailed help:

  -h, --help [TOPIC]    print this message, or detailed help on a topic;
                        the available TOPICS are:

   all, automation, commands, paths, security, testing, or any of the
   subcommands or plugins (certonly, renew, install, register, nginx,
   apache, standalone, webroot, etc.)
-------------------------------------------------------------------------------
[#578#root@ubuntu-1604 /opt/apps/onpremise]$
```



