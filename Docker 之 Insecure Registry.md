# Docker 之 Insecure Registry

> 以下内容取自：[dockerd](https://docs.docker.com/engine/reference/commandline/dockerd/)


----------

> `dockerd` is the persistent process that manages containers. Docker uses different binaries for the **daemon** and **client**. To run the daemon you type `dockerd`.

```
Usage:	dockerd COMMAND

A self-sufficient runtime for containers.

Options:
      ...
      --insecure-registry list                Enable insecure registry communication (default [])
      ...
```


> Docker considers a `private registry` either **secure** or **insecure**. In the rest of this section, registry is used for private registry, and `myregistry:5000` is a placeholder example for a private registry.

private registry 分为 **secure** 或 **insecure** 两种；

> A `secure registry` uses TLS and a copy of its CA certificate is placed on the Docker host at `/etc/docker/certs.d/myregistry:5000/ca.crt`. An `insecure registry` is either not using TLS (i.e., listening on plain text HTTP), or is using TLS with a CA certificate not known by the Docker daemon. The latter can happen when the certificate was not found under `/etc/docker/certs.d/myregistry:5000/`, or if the certificate verification failed (i.e., wrong CA).

**secure registry** 要求：

- 启用 TLS
- 将 CA certificate 放置到 /etc/docker/certs.d/myregistry:5000/ca.crt

insecure registry 特征：

- 未启用 TLS
- 启用了 TLS 但 Docker daemon 不知道 CA certificate 在何处（未放置到正确位置 or 证书验证失败）

> By default, Docker assumes all, but local, registries are secure. Communicating with an insecure registry is not possible if Docker assumes that registry is secure. In order to communicate with an insecure registry, the Docker daemon requires `--insecure-registry` in one of the following two forms:

Docker 默认会认为除了 local 以外的所有 registries 都是 secure 的；因此在和 insecure registry 进行通信时，在未设置 `--insecure-registry` 的情况下，会失败；可以通过如下两种方式进行设置：


> - `--insecure-registry myregistry:5000` tells the Docker daemon that `myregistry:5000` should be considered insecure.
> - `--insecure-registry 10.1.0.0/16` tells the Docker daemon that all registries whose domain resolve to an IP address is part of the subnet described by the **CIDR** syntax, should be considered insecure.

> The flag **can be used multiple times** to allow multiple registries to be marked as insecure.

可以基于上述选项设置多次；

> If an `insecure registry` is not marked as insecure, `docker pull`, `docker push`, and `docker searc`h will result in an error message prompting the user to either secure or pass the `--insecure-registry` flag to the Docker daemon as described above.

> **Local registries**, whose IP address falls in the 127.0.0.0/8 range, **are automatically marked as insecure** as of Docker 1.3.2. It is not recommended to rely on this, as it may change in the future.

Local registries 默认就被认为是 insecure 的；

> Enabling `--insecure-registry`, i.e., allowing un-encrypted and/or untrusted communication, can be useful when running a local registry. However, because its use creates security vulnerabilities it should ONLY be enabled for testing purposes. For increased security, users should add their CA to their system’s list of trusted CAs instead of enabling `--insecure-registry`.


----------

## 证书问题导致的登录失败

### 失败一

登录基于备份数据恢复出来的 Harbor

```
➜  ~ docker login 172.31.2.7
Username: fei.sun
Password:
Error response from daemon: Get https://172.31.2.7/v2/: x509: cannot validate certificate for 172.31.2.7 because it doesn't contain any IP SANs
➜  ~
```

- 基于备份恢复出来的环境，其证书配置仍旧为原来针对 `prod-reg.llsops.com` 生成的那个（花钱购买的），因此基于 IP 地址登录时，会触发上述错误；
- 解决办法：
    - 在 `prod-reg.llsops.com` 绑定的 IP 地址未进行变更前（变更 DNSPod 上的配置），可以先创建一个自签名的、基于 IP 的证书使用；
    - 在目标机器的 `/etc/hosts` 中进行临时配置；

### 失败二

在 Mac 上登录 vagrant 中的 Harbor

```
➜  ~ docker login 11.11.11.12
Username (admin):
Password:
Error response from daemon: Get https://11.11.11.12/v2/: x509: certificate signed by unknown authority
➜  ~
```

需要调整 Docker for Mac 的配置（即 client 访问的本地 docker daemon 配置），Preferences -> Daemon -> Basic -> Insecure registries -> 添加 Harbor 服务对应的 ip 地址，之后点击 Apply & Restart ；

```
➜  ~ docker login 11.11.11.12
Username (admin):
Password:
Login Succeeded
➜  ~
```

### 失败三

```
Error response from daemon: Get https://myregistrydomain.com/v1/users/: dial tcp myregistrydomain.com:443 getsockopt: connection refused.
```

> Harbor supports HTTP by default and Docker client tries to connect to Harbor using HTTPS first, so if you encounter an error as below when you pull or push images, you need to add '`--insecure-registry`' option to `/etc/default/docker` (ubuntu) or `/etc/sysconfig/docker` (centos) and restart Docker.

> If this private registry supports only HTTP or HTTPS with an unknown CA certificate, please add
`--insecure-registry myregistrydomain.com` to the daemon's start up arguments.

> In the case of HTTPS, if you have access to the registry's CA certificate, simply place the CA certificate at `/etc/docker/certs.d/myregistrydomain.com/ca.crt` .

总结：

- 若 private registry 只支持 HTTP ，或基于自签名证书提供 HTTPS 服务，但没有将 ca.crt 放到合适的目录中，此时通过 docker cli 访问时会触发上述错误；
- 若将 ca.crt 放到了合适的目录，且在 docker client 侧配置了 `--insecure-registry` 则能访问成功；

### 成功情况分析

```
# 登录原始 registry
➜  ~ docker login prod-reg.llsops.com
Username: fei.sun
Password:
Login Succeeded
➜  ~
```

- `prod-reg.llsops.com` 在 DNSPod 中进行了注册；
- `docker login prod-reg.llsops.com` 登录过程中，会
    - 基于 `prod-reg.llsops.com` 进行证书验证；
    - 将 `prod-reg.llsops.com` 转为 IP 地址用于登录；


## 信息补充

**SSL needs identification of the peer**, otherwise your connection might be against a man-in-the-middle which decrypts + sniffs/modifies the data and then forwards them encrypted again to the real target. **Identification is done with x509 certificates which need to be validated against a trusted CA** and which need to identify the target you want to connect to.

Usually the target is given as a `hostname` and this is checked against the `subject` and `subject alternative names` of the certificate. In this case your target is a **IP**. To validate the certifcate successfully, the IP must be given in the certificate inside the `subject alternative names` section, but not as an DNS entry (e.g. hostname) but instead as IP.

参考：[这里](https://serverfault.com/questions/611120/failed-tls-handshake-does-not-contain-any-ip-sans)


----------

DNSPod 变更域名绑定信息后，大约 1min 左右就能成功更新，可以通过 `dig xxxx` 进行确认