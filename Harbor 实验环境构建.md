# Harbor 实验环境构建

Harbor installer 版本：

- **Online installer**: The installer downloads Harbor's images from Docker hub. For this reason, the installer is very small in size.
- **Offline installer**: Use this installer when the host does not have an Internet connection. The installer contains pre-built images so its size is larger.

> 下载地址：https://github.com/vmware/harbor/releases

在 Kubernetes 中部署 Harbor ；

> In addition, the deployment instructions on Kubernetes has been created by the community. Refer to [Harbor on Kubernetes](https://github.com/vmware/harbor/blob/master/docs/kubernetes_deployment.md) for details.

Harbor 安装依赖；

> Harbor is deployed as several Docker containers, and, therefore, can be deployed on any Linux distribution that supports Docker. The target host requires Python, Docker, and Docker Compose to be installed.

Prerequisites

- Python 2.7+
- Docker 1.10+
- Docker Compose 1.6.0+


## Docker 安装

详见《[Docker 安装](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Docker%20%E5%AE%89%E8%A3%85.md)》

## Docker Compose 安装

详见《[Docker Compose 安装](https://github.com/moooofly/MarkSomethingDownLLS/blob/master/Docker%20Compose%20%E5%AE%89%E8%A3%85.md)》

## Harbor 安装

> 参考：[Installation and Configuration Guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)

The installation steps

- [Download the installer](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md#downloading-the-installer);
- [Configure `harbor.cfg`](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md#configuring-harbor)
    - required parameters
    - optional parameters
    - 主要是针对 HTTPS 的调整
- [Configuring storage backend (optional)](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md#configuring-storage-backend-optional)
    - By default, Harbor stores images on your local filesystem. 
    - In a production environment, you may consider using other storage backend instead of the local filesystem, like S3, OpenStack Swift, Ceph, etc. 
    - What you need to update is the section of storage in the file `common/templates/registry/config.yml`.
- Run `install.sh` to install and start Harbor;

在成功 `install.sh` 后，就可以通过浏览器或者 CLI 登录了；

> If everything worked properly, you should be able to open a browser to visit the admin portal at `http://reg.yourdomain.com` (change `reg.yourdomain.com` to the hostname configured in your `harbor.cfg`). Note that the default administrator username/password are `admin`/`Harbor12345` .

需要先创建 project 才能 push images ；

> Log in to the admin portal and create a new project, e.g. `myproject`. You can then use docker commands to login and push images (By default, the registry server listens on port 80):

通用示例（注意 push 中的 myproject 对应的是事先创建的 project）

```
$ docker login reg.yourdomain.com
$ docker push reg.yourdomain.com/myproject/myrepo:mytag
```

具体例子

```
[#8#root@ubuntu-1604 ~]$docker login 11.11.11.12
Username (admin):
Password:
Login Succeeded
[#9#root@ubuntu-1604 ~]$
[#22#root@ubuntu-1604 ~]$docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
hello-world                 latest              05a3bd381fc2        5 weeks ago         1.84kB
vmware/harbor-log           v1.2.0              c7887347f435        6 weeks ago         200MB
vmware/harbor-jobservice    v1.2.0              1fb18427db11        6 weeks ago         164MB
vmware/harbor-ui            v1.2.0              b7069ac3bd4b        6 weeks ago         178MB
vmware/harbor-adminserver   v1.2.0              a18331f0c1ae        6 weeks ago         142MB
vmware/harbor-db            v1.2.0              deb8033b1c86        6 weeks ago         329MB
vmware/registry             2.6.2-photon        5d9100e4350e        7 weeks ago         173MB
vmware/postgresql           9.6.4-photon        c562762cbd12        2 months ago        225MB
vmware/clair                v2.0.1-photon       f04966b4af6c        3 months ago        297MB
vmware/nginx-photon         1.11.13             285492ff20d6        3 months ago        147MB
vmware/harbor-notary-db     mariadb-10.1.10     64ed814665c6        6 months ago        324MB
vmware/notary-photon        signer-0.5.0        b1eda7d10640        6 months ago        156MB
vmware/notary-photon        server-0.5.0        6e2646682e3c        7 months ago        157MB
photon                      1.0                 e6e4e4a2ba1b        16 months ago       128MB
[#23#root@ubuntu-1604 ~]$
[#23#root@ubuntu-1604 ~]$
[#23#root@ubuntu-1604 ~]$docker tag hello-world:latest 11.11.11.12/prj1/hello-world:latest
[#24#root@ubuntu-1604 ~]$docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
11.11.11.12/prj1/hello-world   latest              05a3bd381fc2        5 weeks ago         1.84kB
hello-world                    latest              05a3bd381fc2        5 weeks ago         1.84kB
vmware/harbor-log              v1.2.0              c7887347f435        6 weeks ago         200MB
vmware/harbor-jobservice       v1.2.0              1fb18427db11        6 weeks ago         164MB
vmware/harbor-ui               v1.2.0              b7069ac3bd4b        6 weeks ago         178MB
vmware/harbor-adminserver      v1.2.0              a18331f0c1ae        6 weeks ago         142MB
vmware/harbor-db               v1.2.0              deb8033b1c86        6 weeks ago         329MB
vmware/registry                2.6.2-photon        5d9100e4350e        7 weeks ago         173MB
vmware/postgresql              9.6.4-photon        c562762cbd12        2 months ago        225MB
vmware/clair                   v2.0.1-photon       f04966b4af6c        3 months ago        297MB
vmware/nginx-photon            1.11.13             285492ff20d6        3 months ago        147MB
vmware/harbor-notary-db        mariadb-10.1.10     64ed814665c6        6 months ago        324MB
vmware/notary-photon           signer-0.5.0        b1eda7d10640        6 months ago        156MB
vmware/notary-photon           server-0.5.0        6e2646682e3c        7 months ago        157MB
photon                         1.0                 e6e4e4a2ba1b        16 months ago       128MB
[#25#root@ubuntu-1604 ~]$
[#25#root@ubuntu-1604 ~]$
[#25#root@ubuntu-1604 ~]$docker push 11.11.11.12/prj1/hello-world:latest
The push refers to a repository [11.11.11.12/prj1/hello-world]
3a36971a9f14: Pushed
latest: digest: sha256:a5074d61e1e0175fb3a46e0bab46b1f764380ad00cac0e71d53bd4917d196988 size: 524
[#26#root@ubuntu-1604 ~]$
```

tag 不符合要求的情况

```
[#26#root@ubuntu-1604 ~]$docker tag hello-world:latest 11.11.11.12/prjxxx/hello-world:latest
[#28#root@ubuntu-1604 ~]$docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
11.11.11.12/prj1/hello-world     latest              05a3bd381fc2        5 weeks ago         1.84kB
11.11.11.12/prjxxx/hello-world   latest              05a3bd381fc2        5 weeks ago         1.84kB
hello-world                      latest              05a3bd381fc2        5 weeks ago         1.84kB
[#29#root@ubuntu-1604 ~]$docker push 11.11.11.12/prjxxx/hello-world:latest
The push refers to a repository [11.11.11.12/prjxxx/hello-world]
3a36971a9f14: Preparing
denied: requested access to the resource is denied
[#30#root@ubuntu-1604 ~]$
```


## 基于 HTTPS 访问 Harbor

> 参考：[Configuring Harbor with HTTPS Access](https://github.com/vmware/harbor/blob/master/docs/configure_https.md)

- **Create your own CA certificate**

```
[#116#root@ubuntu-1604 /opt/apps]$openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 365 -out ca.crt
Generating a 4096 bit RSA private key
.............................................++
...............................................................................................................................++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:SH
Locality Name (eg, city) []:SH
Organization Name (eg, company) [Internet Widgits Pty Ltd]:LLS
Organizational Unit Name (eg, section) []:DEV
Common Name (e.g. server FQDN or YOUR name) []:11.11.11.12
Email Address []:fei.sun@liulishuo.com
[#117#root@ubuntu-1604 /opt/apps]$ll
total 478984
drwxr-xr-x 3 root root      4096 Oct 23 17:07 ./
drwxr-xr-x 3 root root      4096 Oct 23 10:43 ../
-rw-r--r-- 1 root root      2078 Oct 23 17:07 ca.crt
-rw-r--r-- 1 root root      3272 Oct 23 17:07 ca.key
drwxr-xr-x 3 root root      4096 Oct 23 17:05 harbor/
-rwxr-xr-x 1 root root 490451083 Oct 23 15:55 harbor-offline-installer-v1.2.0.tgz*
[#118#root@ubuntu-1604 /opt/apps]$
```

- **Generate a Certificate Signing Request**

If you use FQDN like `reg.yourdomain.com` to connect your registry host, then you must use `reg.yourdomain.com` as CN (Common Name). Otherwise, if you use IP address to connect your registry host, CN can be anything like your name and so on:

```
[#118#root@ubuntu-1604 /opt/apps]$openssl req -newkey rsa:4096 -nodes -sha256 -keyout harbor_test.key -out harbor_test.csr
Generating a 4096 bit RSA private key
................................................................................................++
.................................................................................................++
writing new private key to 'harbor_test.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:SH
Locality Name (eg, city) []:SH
Organization Name (eg, company) [Internet Widgits Pty Ltd]:LLS
Organizational Unit Name (eg, section) []:DEV
Common Name (e.g. server FQDN or YOUR name) []:11.11.11.12
Email Address []:fei.sun@liulishuo.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:123456
An optional company name []:LLS
[#119#root@ubuntu-1604 /opt/apps]$
[#119#root@ubuntu-1604 /opt/apps]$ll
total 478992
drwxr-xr-x 3 root root      4096 Oct 23 17:16 ./
drwxr-xr-x 3 root root      4096 Oct 23 10:43 ../
-rw-r--r-- 1 root root      2078 Oct 23 17:07 ca.crt
-rw-r--r-- 1 root root      3272 Oct 23 17:07 ca.key
drwxr-xr-x 3 root root      4096 Oct 23 17:05 harbor/
-rwxr-xr-x 1 root root 490451083 Oct 23 15:55 harbor-offline-installer-v1.2.0.tgz*
-rw-r--r-- 1 root root      1789 Oct 23 17:16 harbor_test.csr
-rw-r--r-- 1 root root      3272 Oct 23 17:16 harbor_test.key
```

- **Generate the certificate of your registry host**

If you're using FQDN like reg.yourdomain.com to connect your registry host, then run this command to generate the certificate of your registry host:

```
openssl x509 -req -days 365 -in yourdomain.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out yourdomain.com.crt
```

If you're using IP, say `11.11.11.12` to connect your registry host, you may instead run the command below:

```
[#120#root@ubuntu-1604 /opt/apps]$echo subjectAltName = IP:11.11.11.12 > extfile.cnf
[#122#root@ubuntu-1604 /opt/apps]$openssl x509 -req -days 365 -in harbor_test.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf -out harbor_test.crt
Signature ok
subject=/C=CN/ST=SH/L=SH/O=LLS/OU=DEV/CN=11.11.11.12/emailAddress=fei.sun@liulishuo.com
Getting CA Private Key
[#123#root@ubuntu-1604 /opt/apps]$
[#123#root@ubuntu-1604 /opt/apps]$ll
total 479004
drwxr-xr-x 3 root root      4096 Oct 23 17:23 ./
drwxr-xr-x 3 root root      4096 Oct 23 10:43 ../
-rw-r--r-- 1 root root      2078 Oct 23 17:07 ca.crt
-rw-r--r-- 1 root root      3272 Oct 23 17:07 ca.key
-rw-r--r-- 1 root root        17 Oct 23 17:23 ca.srl
-rw-r--r-- 1 root root        32 Oct 23 17:19 extfile.cnf
drwxr-xr-x 3 root root      4096 Oct 23 17:05 harbor/
-rwxr-xr-x 1 root root 490451083 Oct 23 15:55 harbor-offline-installer-v1.2.0.tgz*
-rw-r--r-- 1 root root      1996 Oct 23 17:23 harbor_test.crt
-rw-r--r-- 1 root root      1789 Oct 23 17:16 harbor_test.csr
-rw-r--r-- 1 root root      3272 Oct 23 17:16 harbor_test.key
[#124#root@ubuntu-1604 /opt/apps]$
```

调整证书位置

```
[#128#root@ubuntu-1604 /opt/apps]$mdkir /data/cert
[#132#root@ubuntu-1604 /opt/apps]$cp harbor_test.crt /data/cert/
[#133#root@ubuntu-1604 /opt/apps]$cp harbor_test.key /data/cert/
```

调整 harbor.cfg 配置内容

```
  hostname = 11.11.11.12
  ui_url_protocol = https
  ...
  #The path of cert and key files for nginx, they are applied only the protocol is set to https
  ssl_cert = /data/cert/harbor_test.crt
  ssl_cert_key = /data/cert/harbor_test.key
```

重新生成 Harbor 配置文件

```
[#136#root@ubuntu-1604 /opt/apps/harbor]$./prepare
Clearing the configuration file: ./common/config/registry/root.crt
Clearing the configuration file: ./common/config/registry/config.yml
Clearing the configuration file: ./common/config/db/env
Clearing the configuration file: ./common/config/ui/app.conf
Clearing the configuration file: ./common/config/ui/private_key.pem
Clearing the configuration file: ./common/config/ui/env
Clearing the configuration file: ./common/config/jobservice/app.conf
Clearing the configuration file: ./common/config/jobservice/env
Clearing the configuration file: ./common/config/adminserver/env
Clearing the configuration file: ./common/config/nginx/nginx.conf
loaded secret from file: /data/secretkey
Generated configuration file: ./common/config/nginx/nginx.conf
Generated configuration file: ./common/config/adminserver/env
Generated configuration file: ./common/config/ui/env
Generated configuration file: ./common/config/registry/config.yml
Generated configuration file: ./common/config/db/env
Generated configuration file: ./common/config/jobservice/env
Generated configuration file: ./common/config/jobservice/app.conf
Generated configuration file: ./common/config/ui/app.conf
Generated certificate, key file: ./common/config/ui/private_key.pem, cert file: ./common/config/registry/root.crt
The configuration files are ready, please use docker-compose to start the service.
[#137#root@ubuntu-1604 /opt/apps/harbor]$
```

重启 Harbor

```
[#137#root@ubuntu-1604 /opt/apps/harbor]$docker-compose down
Stopping nginx      ... done
Stopping harbor-ui  ... done
Stopping harbor-db  ... done
Stopping harbor-log ... done
Removing harbor-jobservice  ... done
Removing nginx              ... done
Removing harbor-ui          ... done
Removing registry           ... done
Removing harbor-adminserver ... done
Removing harbor-db          ... done
Removing harbor-log         ... done
Removing network harbor_harbor
[#138#root@ubuntu-1604 /opt/apps/harbor]$
[#138#root@ubuntu-1604 /opt/apps/harbor]$docker-compose up -d
Creating network "harbor_harbor" with the default driver
Creating harbor-log ...
Creating harbor-log ... done
Creating harbor-adminserver ...
Creating harbor-db ...
Creating registry ...
Creating harbor-adminserver
Creating harbor-db
Creating registry ... done
Creating harbor-ui ...
Creating harbor-ui ... done
Creating harbor-jobservice ...
Creating harbor-db ... done
Creating nginx ...
Creating nginx ... done
[#139#root@ubuntu-1604 /opt/apps/harbor]$
```

验证 HTTPS

- 浏览器中登录 `https://11.11.11.12` ；
- On a machine with Docker daemon, make sure the option "-insecure-registry" does not present, and you must copy `ca.crt` generated in the above step to `/etc/docker/certs.d/reg.yourdomain.com`(or your registry host IP), if the directory does not exist, create it. If you mapped nginx port 443 to another port, then you should instead create the directory `/etc/docker/certs.d/reg.yourdomain.com:port`(or your registry host IP:port). Then run any docker command to verify the setup.

```
[#143#root@ubuntu-1604 /opt/apps/harbor]$mkdir -p /etc/docker/certs.d/11.11.11.12
[#150#root@ubuntu-1604 /opt/apps]$cp ca.crt /etc/docker/certs.d/11.11.11.12/
[#152#root@ubuntu-1604 /opt/apps]$docker login 11.11.11.12
Username: admin
Password:
Login Succeeded
[#153#root@ubuntu-1604 /opt/apps]$
```


## **基于 `curl` 测试时的 HTTPS 问题**

```
[#41#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET --header 'Accept: application/json' 'https://11.11.11.12/api/search?q=prj'
curl: (60) server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
[#42#root@ubuntu-1604 /opt/apps/harbor]$
[#42#root@ubuntu-1604 /opt/apps/harbor]$
[#42#root@ubuntu-1604 /opt/apps/harbor]$curl -X GET -k --header 'Accept: application/json' 'https://11.11.11.12/api/search?q=prj'
{
  "project": [
    {
      "project_id": 3,
      "owner_id": 1,
      "name": "prj2",
      "creation_time": "2017-10-24T03:06:41Z",
      "creation_time_str": "",
      "deleted": 0,
      "owner_name": "",
      "public": 1,
      "Togglable": false,
      "update_time": "2017-10-24T03:06:41Z",
      "current_user_role_id": 0,
      "repo_count": 0,
      "enable_content_trust": false,
      "prevent_vulnerable_images_from_running": false,
      "prevent_vulnerable_images_from_running_severity": "",
      "automatically_scan_images_on_push": false
    }
  ],
  "repository": []
}[#43#root@ubuntu-1604 /opt/apps/harbor]$
```

对应 harbor 日志输出

```
[#93#root@ubuntu-1604 /var/log/harbor/2017-10-24]$tail -f ./*
...
==> ./ui.log <==
Oct 24 07:18:26 172.18.0.1 ui[1839]: 2017-10-24T07:18:26Z [DEBUG] [security.go:283]: user information is nil
Oct 24 07:18:26 172.18.0.1 ui[1839]: 2017-10-24T07:18:26Z [DEBUG] [security.go:296]: using local database project manager

==> ./adminserver.log <==
Oct 24 07:18:26 172.18.0.1 adminserver[1839]: 172.18.0.6 - - [24/Oct/2017:07:18:26 +0000] "GET /api/configurations HTTP/1.1" 200 950

==> ./ui.log <==
Oct 24 07:18:26 172.18.0.1 ui[1839]: 2017-10-24T07:18:26Z [DEBUG] [security.go:298]: creating local database security context...

==> ./proxy.log <==
Oct 24 07:18:26 172.18.0.1 proxy[1839]: 11.11.11.12 - "GET /api/search?q=prj HTTP/1.1" 200 590 "-" "curl/7.47.0" 0.005 0.005 .
```


##  使用 Docker Client 时的 HTTPS 问题

在 [docs/user_guide.md](https://github.com/vmware/harbor/blob/master/docs/user_guide.md) 中有

> **NOTE: Harbor only supports Registry V2 API. You need to use Docker client 1.6.0 or higher.**
>
> Harbor supports HTTP by default and Docker client tries to connect to Harbor using HTTPS first, so if you encounter an error as below when you pull or push images, you need to add '--insecure-registry' option to `/etc/default/docker` (ubuntu) or `/etc/sysconfig/docker` (centos) and restart Docker:
>
>
> ```
> Error response from daemon: Get https://myregistrydomain.com/v1/users/: dial tcp myregistrydomain.com:443 getsockopt: connection refused.
> ```
>
> If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add
`--insecure-registry myregistrydomain.com` to the daemon's start up arguments.
>
> In the case of HTTPS, if you have access to the registry's CA certificate, simply place the CA certificate at /etc/docker/certs.d/myregistrydomain.com/ca.crt .

隐含信息：

- **Harbor only supports Registry V2 API. You need to use Docker client 1.6.0 or higher** => 从 docker v1.3.2 版本开始，默认 docker registry 使用的是 HTTPS ，换句话说，Docker client 会首先使用 HTTPS 连接 docker registry ；而 Harbor 默认使用的是 HTTP ，因此遇到 "dial tcp xx.xx.xx.xx:443 getsockopt: connection refused" 就不奇怪了；
- **add '--insecure-registry' option to `/etc/default/docker` (ubuntu) or `/etc/sysconfig/docker` (centos) and restart Docker** => 这里没有说清楚前提条件，上述修改针对是基于 Upstart 和 SysVinit 的系统，对于使用 Systemd 的系统是没有效果的（[issues/3462](https://github.com/vmware/harbor/issues/3462)）；以 ubuntu 为例，需要修改配置文件 `/lib/systemd/system/docker.service` ，为 [Service] 下的 ExecStart 参数增加 `–insecure-registry xx.xx.xx.xx` 配置内容；另外，**restart Docker** 需要用到 `systemctl` 工具；
- 如果搭建的 harbor 服务属于 **private registry supports only HTTP** 或者 **HTTPS with an unknown CA certificate** 类型，则需要添加 `--insecure-registry myregistrydomain.com` 到 daemon 的 start up 参数中 => 这里说的和上面内容是一个意思，针对的是 Docker Daemon 配置；

在 [docs/installation_guide.md](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md) 中有

> **IMPORTANT:** The default installation of Harbor uses _HTTP_ - as such, you will need to add the option `--insecure-registry` to your client's Docker daemon and restart the Docker service.

说明针对 HTTP 情况，需要为 client 侧依赖的 Docker daemon 配置 `--insecure-registry` ；

综上，**client 和 server 两侧均需要配置 `--insecure-registry` 相关内容**；



实验验证：

在 Mac 上登录 Ubuntu 中的 Harbor (HTTPS)

```
➜  ~ docker login 11.11.11.12
Username (admin):
Password:
Error response from daemon: Get https://11.11.11.12/v2/: x509: certificate signed by unknown authority
➜  ~
```

调整 Docker for Mac 的配置（即 client 依赖的 docker daemon 配置），`Preferences -> Daemon -> Basic -> Insecure registries -> 添加 Harbor 服务对应的 ip 地址`，之后点击 `Apply & Restart` ；

```
➜  ~ docker login 11.11.11.12
Username (admin):
Password:
Login Succeeded
➜  ~
```




## Harbor 生命周期管理

```
[#12#root@ubuntu-1604 ~]$docker-compose -h
Define and run multi-container applications with Docker.
...
Commands:
  ...
  down               Stop and remove containers, networks, images, and volumes
  ps                 List containers
  start              Start services
  stop               Stop services
  up                 Create and start containers
  ...
```

可以使用 `docker-compose` 管理 Harbor 的生命周期（必须在 `docker-compose.yml` 所在目录中运行命令）；

- **Stopping** Harbor

```
$ sudo docker-compose stop
```

- **Restarting** Harbor after stopping

```
$ sudo docker-compose start
```


- To **change Harbor's configuration**, first **stop** existing Harbor instance and **update** `harbor.cfg`. Then run `prepare` script to **populate** the configuration. Finally **re-create** and **start** Harbor's instance (**re-deploy** Harbor) :

```
$ sudo docker-compose down -v
$ vim harbor.cfg
$ sudo prepare
$ sudo docker-compose up -d
```

- Removing Harbor's containers while keeping the image data and Harbor's database files on the file system:

```
$ sudo docker-compose down -v
```

- Removing Harbor's database and image data (for a clean re-installation):

```
$ rm -r /data/database
$ rm -r /data/registry
```

### 持久化数据和日志文件

By default, **registry data** is persisted in the host's `/data/` directory. This data remains unchanged even when Harbor's containers are removed and/or recreated.

```
[#163#root@ubuntu-1604 /opt/apps]$ll /data/
total 40
drw-------  9 root root 4096 Oct 23 17:33 ./
drwxr-xr-x 27 root root 4096 Oct 23 16:15 ../
drwxr-xr-x  2 root root 4096 Oct 23 16:16 ca_download/
drwxr-xr-x  2 root root 4096 Oct 23 17:33 cert/
drwxr-xr-x  2 root root 4096 Oct 23 16:16 config/
drwxr-xr-x  5  999  999 4096 Oct 23 17:38 database/
drwxr-xr-x  2 root root 4096 Oct 23 16:16 job_logs/
drwxr-xr-x  2 root root 4096 Oct 23 16:16 psc/
drwxr-xr-x  2 root root 4096 Oct 23 16:16 registry/
-rw-------  1 root root   16 Oct 23 16:15 secretkey
[#164#root@ubuntu-1604 /opt/apps]$
```

In addition, Harbor uses `rsyslog` to collect the logs of each container. By default, these log files are stored in the directory `/var/log/harbor/` on the target host for troubleshooting.

```
[#164#root@ubuntu-1604 /opt/apps]$ll /var/log/harbor/
total 12
drwxr-xr-x 3 root root   4096 Oct 23 16:16 ./
drwxrwxr-x 8 root syslog 4096 Oct 23 16:16 ../
drwxr-xr-x 2 root root   4096 Oct 23 17:01 2017-10-23/
[#165#root@ubuntu-1604 /opt/apps]$
[#165#root@ubuntu-1604 /opt/apps]$ll /var/log/harbor/2017-10-23/
total 568
drwxr-xr-x 2 root   root   4096 Oct 23 17:01 ./
drwxr-xr-x 3 root   root   4096 Oct 23 16:16 ../
-rw-r----- 1 ubuntu   16   9861 Oct 23 17:45 adminserver.log
-rw-r----- 1 ubuntu   16   1520 Oct 23 19:30 anacron.log
-rw-r----- 1 ubuntu   16    880 Oct 23 20:01 CROND.log
-rw-r----- 1 ubuntu   16  51218 Oct 23 17:38 jobservice.log
-rw-r----- 1 ubuntu   16  37825 Oct 23 17:38 mysql.log
-rw-r----- 1 ubuntu   16  56312 Oct 23 17:45 proxy.log
-rw-r----- 1 ubuntu   16  17212 Oct 23 17:45 registry.log
-rw-r----- 1 ubuntu   16    826 Oct 23 20:01 run-parts.log
-rw-r----- 1 ubuntu   16 359530 Oct 23 17:45 ui.log
[#166#root@ubuntu-1604 /opt/apps]$
```

### Troubleshooting

When Harbor does not work properly, run the below commands to find out if all containers of Harbor are in **UP** status:

```
[#56#root@ubuntu-1604 /opt/apps/harbor]$docker-compose ps
       Name                     Command               State                                Ports
------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/harbor_adminserver       Up
harbor-db            docker-entrypoint.sh mysqld      Up      3306/tcp
harbor-jobservice    /harbor/harbor_jobservice        Up
harbor-log           /bin/sh -c crond && rm -f  ...   Up      127.0.0.1:1514->514/tcp
harbor-ui            /harbor/harbor_ui                Up
nginx                nginx -g daemon off;             Up      0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:80->80/tcp
registry             /entrypoint.sh serve /etc/ ...   Up      5000/tcp
[#57#root@ubuntu-1604 /opt/apps/harbor]$
```

If a container is not in **UP** state, check the log file of that container in directory `/var/log/harbor`.


When setting up Harbor behind an nginx proxy or elastic load balancing, look for the line below, in `common/templates/nginx/nginx.http.conf` and remove it from the sections if the proxy already has similar settings: `location /`, `location /v2/` and `location /service/`.

```
proxy_set_header X-Forwarded-Proto $scheme;
```

and re-deploy Harbor refer to the previous section "Managing Harbor's lifecycle".


----------

## 用户手册信息梳理

> 参考：[User Guide](https://github.com/vmware/harbor/blob/master/docs/user_guide.md)

- Harbor manages **images** through **projects**.
- Users can be added into one project as a member with three different roles (`Guest`/`Developer`/`ProjectAdmin`).
- two system-wide roles: `SysAdmin` and `Anonymous`.
- A **project** in Harbor contains all repositories of an application.
- No images can be pushed to Harbor before the project is created.
- RBAC (Role Based Access Control) is applied to a project.
- There are two types of projects in Harbor: **Public** and **Private**.
- `Images replication` is used to replicate repositories from one Harbor instance to another.
- The function of `Images replication` is **project-oriented**, and once the system administrator set a rule to one project, all repositories under the project will be replicated to the remote registry.
- Each repository will start a **job** to run. If replication job fails due to the network issue, the job will be re-scheduled a few minutes later. A job represents the progress of replicating the repository to the remote instance.
- The member information will not be replicated.
- There may be a bit of **delay** during replication according to the situation of the network.
- Start replication by creating a rule.
- You can change authentication mode between **Database**(default) and **LDAP** before any user is added, when there is at least one user(besides admin) in Harbor, you cannot change the authentication mode.
- Harbor only supports **Registry V2 API**. You need to use Docker client 1.6.0 or higher.

----------

效果图如下：

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20v1.2.0%20Web%20-%201.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20v1.2.0%20Web%20-%202.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20v1.2.0%20Web%20-%203.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/Harbor%20v1.2.0%20Web%20-%204.png)

