# Sentry 显示用户 URL 为内网地址问题排查

## 问题描述

> https://phab.llsapp.com/T44391

表现：当前所有异常都来自于同一个内部 IP
预期：异常 IP 应该是真实的用户 IP

相关链接（已经查不到数据了，因此已经进行了清理）：http://prod.ft.llsops.com/lls/cchybrid/?query=user%3A%22ip%3A172.17.0.1%22


## 问题排查

已知：所有异常地址均为 172.17.0.1 ；业务和 Sentry 之间至少存在一个 nginx ；
未知：访问路径；

### 调用链路相关

> 直接在问题机器上查看 sentry 相关的网络链接状态，并分析；

查看端口 80|9000|443 上的连接状态

```
# 第一次抽样
root@sentry-frontend-prod:/var/log/nginx# netstat -natp|grep -E "80|9000|443"
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 172.17.0.1:43078        172.17.0.4:9000         TIME_WAIT   -
tcp        0      0 127.0.0.1:60622         127.0.0.1:9000          TIME_WAIT   -
tcp        0      0 172.31.12.235:443       113.89.36.1:3790        ESTABLISHED 27530/nginx: worker
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
root@sentry-frontend-prod:/var/log/nginx#


# 第二次抽样
root@sentry-frontend-prod:/var/log/nginx# netstat -natp|grep -E "80|9000|443"
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 172.31.12.235:443       1.62.179.119:56696      ESTABLISHED 27530/nginx: worker
tcp        0      0 127.0.0.1:60788         127.0.0.1:9000          TIME_WAIT   -
tcp        0      0 172.31.12.235:443       113.89.36.1:3790        ESTABLISHED 27530/nginx: worker
tcp        0      0 172.17.0.1:43244        172.17.0.4:9000         TIME_WAIT   -
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
root@sentry-frontend-prod:/var/log/nginx#


# 第三次抽样
root@sentry-frontend-prod:/var/log/nginx# netstat -natp|grep -E "80|9000|443"
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 172.31.12.235:443       1.62.179.119:56696      ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       223.104.212.156:37185   ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       113.89.36.1:3790        FIN_WAIT2   -
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
root@sentry-frontend-prod:/var/log/nginx#


# 第 M 次抽样
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 172.31.12.235:443       49.112.236.29:49006     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       223.96.56.212:13006     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.79.120:43811    FIN_WAIT2   -
tcp        0      0 127.0.0.1:34858         127.0.0.1:9000          TIME_WAIT   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49010     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       112.224.67.47:49558     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       49.112.236.29:49008     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.17.0.1:45558        172.17.0.4:9000         TIME_WAIT   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49012     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       223.96.56.212:13005     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       219.159.255.165:37074   ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       112.224.67.47:49559     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.81.126:26410    FIN_WAIT2   -
tcp        0      0 127.0.0.1:34870         127.0.0.1:9000          TIME_WAIT   -
tcp        0      0 172.31.12.235:443       117.136.81.126:26409    FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49014     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       60.209.150.147:26585    FIN_WAIT2   -
tcp        0      0 172.17.0.1:45546        172.17.0.4:9000         TIME_WAIT   -
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
^C
root@sentry-frontend-prod:/var/log/nginx#


# 第 N 次抽样
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 127.0.0.1:34572         127.0.0.1:9000          TIME_WAIT   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49006     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       223.96.56.212:13006     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       61.158.149.233:5613     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.79.120:43811    FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       61.158.149.233:5603     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.17.0.1:45242        172.17.0.4:9000         TIME_WAIT   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49010     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       49.112.236.29:49008     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       49.112.236.29:49012     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       223.96.56.212:13005     ESTABLISHED 27530/nginx: worker
tcp        0      0 127.0.0.1:34580         127.0.0.1:9000          ESTABLISHED 27530/nginx: worker
tcp        0      0 172.17.0.1:45260        172.17.0.4:9000         TIME_WAIT   -
tcp        0      0 172.31.12.235:443       117.136.81.126:26410    FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       117.136.81.126:26409    FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       49.112.236.29:49014     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       60.209.150.147:26585    FIN_WAIT2   -
tcp        0      0 172.17.0.1:45268        172.17.0.4:9000         ESTABLISHED 3601/docker-proxy
tcp        0      0 127.0.0.1:34554         127.0.0.1:9000          TIME_WAIT   -
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
tcp6       0      0 127.0.0.1:9000          127.0.0.1:34580         ESTABLISHED 3601/docker-proxy
```

数据分析：

- 存在 TCP 连接异常状态 `FIN_WAIT2` ，且本地地址均为 `172.31.12.235:443` ，可以断定访问服务的客户端存在某种问题（对端将处于 `CLOSE_WAIT` 状态）；


绘制完整调用流（基于某次信息全面的抽样数据）

```
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4791/nginx -g daemo
tcp        0      0 172.31.12.235:443       117.136.72.155:42937    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       61.158.149.233:5613     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.79.120:43811    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       61.158.149.233:5603     ESTABLISHED 27530/nginx: worker
tcp        0      0 172.17.0.1:44392        172.17.0.4:9000         ESTABLISHED 3601/docker-proxy
tcp        0      0 172.31.12.235:443       117.136.72.155:42938    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.72.155:42940    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.72.155:42939    ESTABLISHED 27530/nginx: worker
tcp        0      0 127.0.0.1:33704         127.0.0.1:9000          ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.72.155:42719    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.72.155:42941    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       112.224.67.47:49557     FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       112.224.67.47:49556     FIN_WAIT2   -
tcp        0      0 172.31.12.235:443       117.136.81.126:26410    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       117.136.81.126:26409    ESTABLISHED 27530/nginx: worker
tcp        0      0 172.31.12.235:443       60.209.150.147:26585    ESTABLISHED 27530/nginx: worker
tcp6       0      0 :::80                   :::*                    LISTEN      4791/nginx -g daemo
tcp6       0      0 :::9000                 :::*                    LISTEN      3601/docker-proxy
tcp6       0      0 127.0.0.1:9000          127.0.0.1:33704         ESTABLISHED 3601/docker-proxy
```

综合上下文信息，可知调用流为

```
                               nginx                                       docker-proxy
client <--> 172.31.12.235:443 <-----> 127.0.0.1:33704 <==> 127.0.0.1:9000 <------------> 172.17.0.1:44392 <==> 172.17.0.4:9000
```

![Sentry 调用链路](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Sentry%20%E8%B0%83%E7%94%A8%E9%93%BE%E8%B7%AF.png)


> 此处不详细展开另外三个对比试验：
> 
> - host-only 网络：raven-go => vboxnet0 (宿主机) => enp0s8 (虚拟机) => sentry
> - nat 网络：raven-go => vboxnet0 (宿主机) => sentry
> - 虚拟机中的本地：raven-go => sentry

当前结论：

- 172.17.0.1 地址为 docker0 的地址；
- User Ip 随着 raven-go 中代码的设置而变化；
- SDK ClientIP 似乎采用的是到 Sentry 最后一跳的地址；

### 代码研究

问题焦点：Sentry 的 Web UI 上显示的 **USER** 下的 **IP Address** 字段为 `172.17.0.1` ；同时观察到 **SDK** 下的 **Client IP** 也为 `172.17.0.1` ；


- 设置 USER 和 SDK 中 ip 地址的地方

> 代码位置：[这里](https://github.com/getsentry/sentry/blob/1757bc84056faef7f6b7826dfa731ae128456db8/src/sentry/event_manager.py#L337)

```
class EventManager(object):
...
    def normalize(self, request_env=None):
        request_env = request_env or {}
        ...
		# 从 request_env 中获取 client_ip 的值
		# 若 client_ip 设置了值，则全部配置设置为该值（主要考虑 USER 和 SDK 中 ip）
        # Fill in ip addresses marked as {{auto}}
        client_ip = request_env.get('client_ip')
        if client_ip:
            if get_path(data, ['sentry.interfaces.Http', 'env', 'REMOTE_ADDR']) == '{{auto}}':
                data['sentry.interfaces.Http']['env']['REMOTE_ADDR'] = client_ip

            if get_path(data, ['request', 'env', 'REMOTE_ADDR']) == '{{auto}}':
                data['request']['env']['REMOTE_ADDR'] = client_ip

            if get_path(data, ['sentry.interfaces.User', 'ip_address']) == '{{auto}}':
                data['sentry.interfaces.User']['ip_address'] = client_ip

            if get_path(data, ['user', 'ip_address']) == '{{auto}}':
                data['user']['ip_address'] = client_ip
            ...
        # If there is no User ip_addres, update it either from the Http interface
        # or the client_ip of the request.
        auth = request_env.get('auth')
        is_public = auth and auth.is_public
        add_ip_platforms = ('javascript', 'cocoa', 'objc')

        # 从 HTTP 接口中获取 REMOTE_ADDR 地址
        http_ip = data.get('sentry.interfaces.Http', {}).get('env', {}).get('REMOTE_ADDR')
        if http_ip:
        	# 若存在，则将其设置为 User 的 ip_address 的值
            data.setdefault('sentry.interfaces.User', {}).setdefault('ip_address', http_ip)
        elif client_ip and (is_public or data.get('platform') in add_ip_platforms):
        	# 若不存在，则设置 User 的 ip_address 的值为 client_ip
            data.setdefault('sentry.interfaces.User', {}).setdefault('ip_address', client_ip)

        # 将 client_ip 设置为 sdk 的 client_ip 的值
        if client_ip and data.get('sdk'):
            data['sdk']['client_ip'] = client_ip
        ...
```

可以看到 client_ip 是从 request_env 中获取的；

- 设置 client_ip 到 request_env 的位置

> 代码位置：[这里](https://github.com/getsentry/sentry/blob/7dda44461489599d38f6cfbb47de719620d01cc3/src/sentry/coreapi.py#L450)

```
class LazyData(MutableMapping):
    def __init__(self, data, content_encoding, helper, project, key, auth, client_ip):
        self._data = data
        self._content_encoding = content_encoding
        self._helper = helper
        self._project = project
        self._key = key
        self._auth = auth
        self._client_ip = client_ip
        self._decoded = False

    def _decode(self):
    	...	
        # mutates data
        manager = EventManager(data, version=auth.version)
        # 调用 normalize 并设置 client_ip 的地方
        manager.normalize(request_env={
            'client_ip': self._client_ip,
            'auth': self._auth,
        })

        self._data = data
        self._decoded = True
        ...
```

可以看到，LazyData 在初始化时，会设置 client_ip 值，之后在 _decode 调用中设置给 request_env 

- LazyData 被调用的位置

> 代码位置：[这里](https://github.com/getsentry/sentry/blob/c046b91818c08290abeb6202667553e9775f2279/src/sentry/web/api.py#L277)

```
class StoreView(APIView):
    ...
    def post(self, request, **kwargs):
        try:
            data = request.body
        except Exception as e:
            logger.exception(e)
            # We were unable to read the body.
            # This would happen if a request were submitted
            # as a multipart form for example, where reading
            # body yields an Exception. There's also not a more
            # sane exception to catch here. This will ultimately
            # bubble up as an APIError.
            data = None

        if pubsub is not None and data is not None:
            pubsub.publish('requests', data)

        response_or_event_id = self.process(request, data=data, **kwargs)
        if isinstance(response_or_event_id, HttpResponse):
            return response_or_event_id
        return HttpResponse(
            json.dumps({
                'id': response_or_event_id,
            }), content_type='application/json'
        )
    ...
    def process(self, request, project, key, auth, helper, data, **kwargs):
        metrics.incr('events.total')

        if not data:
            raise APIError('No JSON data was found')

        # 从 REMOTE_ADDR 获取目标地址
        remote_addr = request.META['REMOTE_ADDR']

        data = LazyData(
            data=data,
            content_encoding=request.META.get('HTTP_CONTENT_ENCODING', ''),
            helper=helper,
            project=project,
            key=key,
            auth=auth,
            client_ip=remote_addr,  # 设置
        )
```

可以看到 client_ip 是从 request.META 中的 REMOTE_ADDR 获取的；

- request.META 中的 REMOTE_ADDR 的设置

> 代码位置：[这里](https://github.com/getsentry/sentry/blob/6e86a084879e8e24072d5c39df1c98efc696bbe2/src/sentry/middleware/proxy.py#L52)

```
class SetRemoteAddrFromForwardedFor(object):
    def __init__(self):
        if not getattr(settings, 'SENTRY_USE_X_FORWARDED_FOR', True):
            from django.core.exceptions import MiddlewareNotUsed
            raise MiddlewareNotUsed

    # 处理所有类型的 HTTP 请求
    def process_request(self, request):
        try:
            # 获取 X_FORWARDED_FOR 的内容
            real_ip = request.META['HTTP_X_FORWARDED_FOR']
        except KeyError:
            pass
        else:
            # HTTP_X_FORWARDED_FOR can be a comma-separated list of IPs.
            # Take just the first one.
            # 注意：这里直接取了第一个 ip ，即假定了不会恶意欺骗发生
            real_ip = real_ip.split(",")[0].strip()
            if ':' in real_ip and '.' in real_ip:
                # Strip the port number off of an IPv4 FORWARDED_FOR entry.
                real_ip = real_ip.split(':', 1)[0]
            request.META['REMOTE_ADDR'] = real_ip
```

和

> 代码位置：[这里](https://github.com/getsentry/sentry/blob/4a852d1c941e4927f39a1896bdfd9bd52c33b7f8/src/sentry/conf/server.py#L1213)

```
# Whether we should look at X-Forwarded-For header or not
# when checking REMOTE_ADDR ip addresses
SENTRY_USE_X_FORWARDED_FOR = True
```

到此可知，地址信息是从 HTTP Header `X-Forwarded-For` 中提取出来的；


### 配置研究

目标：

- 需要确保 `X-Forwarded-For` 这个 HTTP Header 被正确设置；
- 需要在 Nginx 启用了 SSL 功能下保证上述功能正确；


在《[Configuring Sentry](https://github.com/getsentry/sentry/blob/88ce93c1ec6cb0474c8ad124ef55831292b26aa4/docs/config.rst)》中有

> Additionally, if you're using SSL, you'll want to configure the following settings in `sentry.conf.py`:
>
> ```
> SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
> SESSION_COOKIE_SECURE = True
> CSRF_COOKIE_SECURE = True
> ```

在 `sentry.conf.py` 中有

```
##############
# Web Server #
##############

# If you're using a reverse SSL proxy, you should enable the X-Forwarded-Proto
# header and set `SENTRY_USE_SSL=1`

if Bool(env('SENTRY_USE_SSL', False)):
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SOCIAL_AUTH_REDIRECT_IS_HTTPS = True
```

在《[Installation with Python](https://github.com/getsentry/sentry/blob/a077fa94e837821dddf73d9a04c6eca1122a8405/docs/installation/python/index.rst)》中有


> If you are planning to use SSL, you will also need to ensure that you've enabled detection within the reverse proxy (see the instructions above), as well as within the Sentry configuration:
>
> ```
> SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
> SESSION_COOKIE_SECURE = True
> CSRF_COOKIE_SECURE = True
> ```

虽然诸多地方说了要配置这个那个，然而没有一个地方把问题彻底描述清楚；

经过梳理，结论如下：

- 若要 Sentry 能正确配合 reverse SSL proxy (nginx) ，则需要在启动 Sentry 服务时设置 `SENTRY_USE_SSL=1` 这个环境变量（启动 Sentry 容器时设置）；
- 需要确保 Sentry 中获取到的 REMOTE_ADDR 值是正确的 client 地址（调整 nginx 配置）；


最终确保功能正确的 nginx.conf 配置为

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
	worker_connections 768;
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

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
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
```





