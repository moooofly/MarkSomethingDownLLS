# Dockefile 解析

## prometheus/client_golang

- https://github.com/prometheus/client_golang
- https://github.com/gliderlabs/registrator/blob/master/Dockerfile  -- 另外一个例子

如下 Dockerfile 利用了 docker 的分阶段编译 (multi-stage builds) 功能，即先在一个镜像中构建，再在另一个镜像中执行；

> multi-stage builds 的起源和官方文档：
>
> - [Builder pattern vs. Multi-stage builds in Docker](https://blog.alexellis.io/mutli-stage-docker-builds/)
> - [Use multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)

A workaround which is informally called the `builder pattern` involves using two Docker images - one to perform a build and another to ship the results of the first build without the penalty of the build-chain and tooling in the first image. This normally meant having two separate Dockerfiles and a shell script to orchestrate all of the 7 steps above.

An example of the `builder pattern`:

1. Derive from a Golang base image with the whole runtime/SDK (Dockerfile.build)
2. Add source code
3. Produce a statically-linked binary
4. Copy the static binary from the image to the host (docker create, docker cp)
5. Derive from SCRATCH or some other light-weight image such as alpine (Dockerfile)
6. Add the binary back in
7. Push a tiny image to the Docker Hub

`Multi-stage builds` give the benefits of the `builder pattern` without the hassle of maintaining three separate files.

```
# This Dockerfile builds an image for a client_golang example.
#
# Use as (from the root for the client_golang repository):
#    docker build -f examples/$name/Dockerfile -t prometheus/golang-example-$name .

# Builder image, where we build the example.
# 远端拉取后，使用 builder 作为相应镜像的别名，便于后续使用
FROM golang:1.9.0 AS builder
WORKDIR /go/src/github.com/prometheus/client_golang
COPY . .
WORKDIR /go/src/github.com/prometheus/client_golang/prometheus
# 这里下载的是啥
RUN go get -d
WORKDIR /go/src/github.com/prometheus/client_golang/examples/simple
# 默认情况下 CGO_ENABLED=1 ，程序和预编译的标准库都采用了 C 的实现
# 设置为 CGO_ENABLED=0 可以确保 Go 中存在的那些、同时支持 Go 和 C 实现方式的 feature 采用 Go 实现方式
RUN CGO_ENABLED=0 GOOS=linux go build -a -tags netgo -ldflags '-w'

# Final image.
# 这里引用了上面的别名，第二次使用 FROM 表示开始第二个构建流程
FROM scratch
LABEL maintainer "The Prometheus Authors <prometheus-developers@googlegroups.com>"
# 在这里，我们不再直接从我们的宿主机器中直接拷贝二进制文件, 而是从一个叫做 builder 的容器中获取
# 它会从我们起先构建的镜像中获得已经编译好的文件并引入到这个容器里
COPY --from=builder /go/src/github.com/prometheus/client_golang/examples/simple .
EXPOSE 8080
# 这里表示 workdir 默认就是 "/" ？
ENTRYPOINT ["/simple"]
```

实际执行

```
➜  client_golang git:(master) ✗ docker build -f examples/simple/Dockerfile -t prometheus/golang-example-simple .
Sending build context to Docker daemon  1.737MB
Step 1/12 : FROM golang:1.9.0 AS builder
1.9.0: Pulling from library/golang
219d2e45b4af: Pull complete
ef9ce992ffe4: Pull complete
d0df8518230c: Pull complete
38ae21afde7b: Pull complete
94bf2214b363: Pull complete
b9cea41720ff: Pull complete
15f04fd4e5e3: Pull complete
3f1ba867568b: Pull complete
Digest: sha256:c292001fcef9024c25f7ba1fdb6f1b0b50af3beca518a530031db6732ef57288
Status: Downloaded newer image for golang:1.9.0
 ---> 1cdc81f11b10
Step 2/12 : WORKDIR /go/src/github.com/prometheus/client_golang
Removing intermediate container 6a23dab1c424
 ---> 85c6799e515e
Step 3/12 : COPY . .
 ---> 89d6ca93cd16
Step 4/12 : WORKDIR /go/src/github.com/prometheus/client_golang/prometheus
Removing intermediate container a7569aec78b1
 ---> af5aae5fba19
Step 5/12 : RUN go get -d
 ---> Running in 01720d220d45
Removing intermediate container 01720d220d45
 ---> 6c417057ffc8
Step 6/12 : WORKDIR /go/src/github.com/prometheus/client_golang/examples/simple
Removing intermediate container fe0a47862532
 ---> e2de36fd07a3
Step 7/12 : RUN CGO_ENABLED=0 GOOS=linux go build -a -tags netgo -ldflags '-w'
 ---> Running in b58825d4e28e
Removing intermediate container b58825d4e28e
 ---> 22836880ea94
Step 8/12 : FROM scratch
 --->
Step 9/12 : LABEL maintainer "The Prometheus Authors <prometheus-developers@googlegroups.com>"
 ---> Running in 68c0cc351c1a
Removing intermediate container 68c0cc351c1a
 ---> 6b4998a7fef9
Step 10/12 : COPY --from=builder /go/src/github.com/prometheus/client_golang/examples/simple .
 ---> 40b2e5d533be
Step 11/12 : EXPOSE 8080
 ---> Running in 58e06134d1ca
Removing intermediate container 58e06134d1ca
 ---> 6318dd089135
Step 12/12 : ENTRYPOINT ["/simple"]
 ---> Running in 9e89f1dca413
Removing intermediate container 9e89f1dca413
 ---> 39263375e857
Successfully built 39263375e857
Successfully tagged prometheus/golang-example-simple:latest
➜  client_golang git:(master) ✗
➜  client_golang git:(master) ✗
➜  client_golang git:(master) ✗ docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
prometheus/golang-example-simple   latest              39263375e857        10 minutes ago      7.2MB
<none>                             <none>              22836880ea94        10 minutes ago      745MB
...
➜  client_golang git:(master) docker run --rm -it -p 80:8080 prometheus/golang-example-simple

```

访问 http://localhost/metrics 可以看到输出


## prometheus-config

### .gitlab-ci.yml 解析

```
stages:
  - test

.test_tmpl: &test_tmpl
  stage: test
  image: stag-reg.llsops.com/library/debian:jessie
  # 该 ci 只干了下面一件事，即通过程序检查规则是否合法
  script:
    - find ./ -type f -name '*.rules.yml' | xargs ./bin/promtool-linux check rules
  tags:
    - docker

run_v2:
  <<: *test_tmpl
```

## 一个 go-http-server 示例工程

- https://git.llsapp.com/codelab/go-http-server

### Dockefile 解析

```
FROM alpine:3.7
COPY bazel-bin/server /server
```

### .gitlab-ci.yml 解析

```
# 看来 stage 的命名可以随意，顺序比较重要？
stages:
  - compile
  - build

variables:
  # 这个变量在这里定义，实际用于 .build-image.yml 里，实现了抽象
  HARBOR_TEAM: platform

# 可以包含其他的 yaml 文件
include: .build-image.yml

bazel_build:
  image: stag-reg.llsops.com/library/bazel:2018-03-05
  script:
    - export BAZEL_RUNID=$CI_JOB_ID
    - bazel build //...
  artifacts:
    paths:
      - bazel-bin/server
  only:
    - develop
    - master
    - tags
  tags:
    - docker
    - bazel
    - overseas
  stage: compile
  # 可以指定重试次数
  retry: 2
```

问题：

- artifacts 是个什么概念
    - 不同 stage 间通过 artifact 来共享编译好的 go binary
- CI_JOB_ID 哪里来的
- stages 中内容的顺序重要么？

### .build-image.yml 解析

```
variables:
  HARBOR: stag-reg.llsops.com

.docker_build_template: &docker_build_template
  script:
    - export IMAGE_TAG=${CI_BUILD_TAG:-${CI_BUILD_REF_NAME}-${CI_BUILD_REF:0:8}}
    - export IMAGE_NAME=${REPOSITORY}:${IMAGE_TAG}
    - docker build -t ${IMAGE_NAME} -f Dockerfile .
    - docker push ${IMAGE_NAME}
    - docker rmi ${IMAGE_NAME}
    - echo "${IMAGE_NAME}"
  tags:
    - shell-builder
  stage: build

build_dev:
  <<: *docker_build_template
  variables:
    REPOSITORY: ${HARBOR}/${HARBOR_TEAM}/${CI_PROJECT_NAME}
  only:
    - develop   # 只针对 develop 分支
  environment:
    name: development

build_g_dev:
  <<: *docker_build_template
  variables:
    REPOSITORY: ${HARBOR}/${HARBOR_TEAM}/${CI_PROJECT_NAME}
  only:
    - develop    # 只针对 develop 分支
  environment:
    name: development
  tags:
    - g-shell-builder    # 这里的定义是对 docker_build_template 中 tags 的累加还是覆盖？

build_stag:
  <<: *docker_build_template
  variables:
    REPOSITORY: ${HARBOR}/${HARBOR_TEAM}/${CI_PROJECT_NAME}
  environment:
    name: staging
  only:
    - master   # 只针对 master 分支

build_prod:
  <<: *docker_build_template
  variables:
    REPOSITORY: ${HARBOR}/${HARBOR_TEAM}-rls/${CI_PROJECT_NAME}
  only:
    - tags   # 只针对 tags 分支
  environment:
    name: production

```

问题：

- CI_BUILD_TAG/CI_BUILD_REF_NAME/CI_BUILD_REF/CI_PROJECT_NAME 都是什么（CI 默认变量？）
- environment 和 variables 的区别？


## 流利说基于 GitLab CI 构建 golang + grpc 的 Dockefile

> - https://git.llsapp.com/ops/docker-base-images/
> - https://git.llsapp.com/ops/docker-base-images/blob/master/golang/Dockerfile
> - https://git.llsapp.com/ops/docker-base-images/blob/master/.gitlab-ci.yml
> - https://git.llsapp.com/ops/docker-base-images/-/jobs/370790

### Dockefile 解析

```
# Format: FROM    repository[:version]
# 诸如 {{ xxx }} 这种格式，均在 .gitlab-ci.yml 中通过 sed 进行了替换，以便实现模板化版本变更
FROM golang:{{ version }}

# Format: MAINTAINER Name <email@addr.ess>
MAINTAINER Liang Name <liang@liulishuo.com>

# update apt sources
# 添加 阿里云 的源
COPY debian-mirror-sources/aliyun-jessie-sources.list /etc/apt/sources.list

# Install essential tools
RUN apt-get update \
    && apt-get install -y patch \
    && apt-get install -y autoconf \
    && apt-get install -y automake \
    && apt-get install -y libtool \
    && apt-get install -y bison \
    && apt-get install -y zlib1g \
    && apt-get install -y zlib1g-dev \
    && apt-get install -y libgoogle-glog-dev \
    && apt-get install -y vim

# grab gosu for easy step-down from root
# gosu 安装相关（很多开源项目都是这么用的，看来是抄的）
# 这里有个 https_proxy 的使用问题（*）
ENV GOSU_VERSION 1.7
RUN set -x \
	&& (export https_proxy={{ http_proxy }}; wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)") \
	&& (export https_proxy={{ http_proxy }}; wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc") \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true

# Change work dir
# 这里直接切目录（看来能够附带创建）
WORKDIR /go/src/github.com/Masterminds

# Install glide package manager for golang
# 这里通过 {{ glide_branch }} 可以指定获取的版本（branch or tag）
RUN git clone -b {{ glide_branch }} https://github.com/Masterminds/glide.git \
    && cd glide \
    && make build \
    && mv glide /usr/local/bin/

# Change work dir to install cache
WORKDIR /go/src/code.llsapp.com/common/

# Clone go-package-cache
# go-package-cache 只是一个通过 glide 维护的列表
# 这里通过免密的方式访问了 gitlab (*)
# 最后为何要删除 vendor 和 glide.lock (*)
RUN git clone -b {{ go_package_cache_branch }} https://{{ gitlab_account }}:{{ gitlab_passwd }}@git.llsapp.com/common/go-package-cache.git \
    && cd go-package-cache \
    && (export http_proxy={{ http_proxy }}; export https_proxy={{ http_proxy }}; glide update) \
    && rm -rf vendor \
    && rm -f glide.lock

# Change work dir to install protoc-go-gen
WORKDIR /go/src/github.com/golang

# Install protoc-go-gen with some branch
# 安装 golang/protobuf
# 如何选取的 {{ protoc_gen_go_branch }} 数值 (*)
RUN (export https_proxy={{ http_proxy }}; git clone https://github.com/golang/protobuf.git) \
    && cd protobuf/protoc-gen-go \
    && git checkout {{ protoc_gen_go_branch }} \
    && go install \
    && rm -rf /go/src/github.com/golang/protobuf

# Change work dir to install 3rdlibs
WORKDIR /opt/3rdlibs/

# Install protobuf && grpc
# k8s-deploy 我没有权限查看 (*)
RUN git clone https://{{ gitlab_account }}:{{ gitlab_passwd }}@git.llsapp.com/k8s-deploy/deb_pkgs.git
RUN dpkg -i /opt/3rdlibs/deb_pkgs/debs/protobuf_{{ protobuf_version }}_amd64.deb \
    && dpkg -i /opt/3rdlibs/deb_pkgs/debs/grpc_{{ grpc_version }}_amd64.deb \
    && rm -rf deb_pkgs

ENV LD_LIBRARY_PATH=/usr/local/lib/:$LD_LIBRARY_PATH

# Change to apps work dir
WORKDIR /opt/apps/

# Pull gRPC health checker repo
# 为何只有该命令执行 --single-branch (*)
RUN git clone -b {{ health_checker_branch }} --single-branch https://{{ gitlab_account }}:{{ gitlab_passwd }}@git.llsapp.com/k8s-deploy/health-checker.git

### Pull gRPC health checker repo
##RUN git clone -b {{ health_checker_branch }} --single-branch https://{{ gitlab_account }}:{{ gitlab_passwd }}@git.llsapp.com/k8s-deploy/health-checker.git \
##    && cd health-checker \
##    && ([ -f .gitmodules ] && cp .gitmodules .gitmodules.bak) || true \
##    && ([ -f .gitmodules ] && sed -r -i -e 's#url *= *git@git.llsapp.com:(.*)/(.*).git#url = https://{{ gitlab_account }}:{{ gitlab_passwd }}@git.llsapp.com/\1/\2.git#g' .gitmodules) || true \
##    && git submodule sync \
##    && git submodule update --init --recursive \
##    && ([ -f .gitmodules ] && mv .gitmodules.bak .gitmodules) || true
##
### Build health checker
##WORKDIR /opt/apps/health-checker/health-checker
##RUN make && make clean

# Change to apps work dir
WORKDIR /opt/apps/
```

问题：

- 什么情况下才需要使用 {{ http_proxy }}
    - 其实大部分命令的执行都不需要 proxy , 只有 `go get`/`glide install/update`/`pip install` 需要设置；
    - 非常不建议设置全局代理（因为会导致一些访问我们自己 git 的情况也走了 proxy 的翻墙功能）；如果需要执行某个命令但是需要设置代理，请使用: `(export https_proxy='xxxx'; export http_proxy='xxx'; YOUR_COMMAND)`，这样会限制 proxy 的 scope 在这个 `(...)` 里面，其中 `(...)` 里面可以包含任意多个命令；
- 什么情况下才需要使用 --single-branch
- gitlab 账号信息的使用：{{ gitlab_account }} 和 {{ gitlab_passwd }}


### .gitlab-ci.yml 解析

> 这里只截取了 golang 相关的内容

```
# 定义了多个 stage ，但我们主要关心 build 这个
stages:
    - test
    - build
    - deploy

# 这里定义的变量，只在当前文件中使用
variables:
    DOCKER_REGISTRY: stag-reg.llsops.com
    # 两者相同是错误 or 故意 (*)
    HTTP_PROXY: http://stag.s.llsops.com:8123
    HTTPS_PROXY: http://stag.s.llsops.com:8123

# 这里应该是定义了一个 锚点 便于复用
.builder_template_grpc_1_8_4: &builder_template_grpc_1_8_4
    # 由于复用的缘故，所以下面的定义段都是 build 这个 stage
    stage: build
    # script 下的每一行都对应了 gitlab ci 中的一个顺序执行步骤
    script:
        - export GITLAB_ACCOUNT=gitlab-ci-token
        - export GRPC_VERSION=1.8.4-1
        - export PROTOBUF_VERSION=3.5.1-1
        - export HEALTH_CHECKER_BRANCH=v0.0.2
        - export GO_PACKAGE_CACHE_BRANCH=v0.0.4
        - export GLIDE_BRANCH=v0.12.3
        - export PROTOC_GEN_GO_BRANCH=c9c7427a2a70d2eb3bafa0ab2dc163e45f143317
        - export IMAGE_NAME=${DOCKER_REGISTRY}/library/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}-grpc-${GRPC_VERSION}-${CI_BUILD_TAG}
        # 从这里可以看出，${DOCKER_IMAGE_NAME}/Dockerfile 对应的路径和文件，是 gitlab ci 能够直接访问到的当前仓库位置
        - sed -i -e 's#{{ http_proxy }}#'${HTTP_PROXY}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ gitlab_account }}#'${GITLAB_ACCOUNT}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ gitlab_passwd }}#'${CI_BUILD_TOKEN}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ version }}#'${DOCKER_IMAGE_TAG}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ grpc_version }}#'${GRPC_VERSION}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ protobuf_version }}#'${PROTOBUF_VERSION}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ health_checker_branch }}#'${HEALTH_CHECKER_BRANCH}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ go_package_cache_branch }}#'${GO_PACKAGE_CACHE_BRANCH}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ glide_branch }}#'${GLIDE_BRANCH}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - sed -i -e 's#{{ protoc_gen_go_branch }}#'${PROTOC_GEN_GO_BRANCH}'#g' ${DOCKER_IMAGE_NAME}/Dockerfile
        - docker build -t ${IMAGE_NAME} -f ${DOCKER_IMAGE_NAME}/Dockerfile .
        - docker push ${IMAGE_NAME}
        - docker rmi ${IMAGE_NAME}
        - echo "${IMAGE_NAME}"

build_golang_1.8.3_grpc_1.8.4:
    # 锚点 应用
    <<: *builder_template_grpc_1_8_4
    # before_script 表明如下定义发生在 script 之前
    before_script:
        - export DOCKER_IMAGE_NAME=golang
        - export DOCKER_IMAGE_TAG=1.8.3
    # 干啥的？
    only:
        - tags
    # 指定 gitlab ci 使用的环境？资源？工具？(*)
    tags:
        - staging
        - shell-builder
```

问题：

- 需要理解 gitlab ci 的 stages 
- 需要理解锚点的使用
- 需要理解 only 和 tags 的用途
    - only 应该用于限定作用于哪些分支的
    - tags 应该是用来选择环境信息的



### aliyun-jessie-sources.list

```
# aliyun mirror source
deb http://mirrors.aliyun.com/debian/ jessie main non-free contrib
deb http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ jessie main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ jessie-proposed-updates main non-free contrib

# official source
deb http://deb.debian.org/debian jessie main
deb http://deb.debian.org/debian jessie-updates main
deb http://security.debian.org jessie/updates main
```

### go-package-cache/glide.yaml

> https://git.llsapp.com/common/go-package-cache/blob/master/glide.yaml

通过 glide 维护了 lls 常用的包

```
package: code.llsapp.com/common/go-package-cache
import:
- package: github.com/golang/protobuf
  version: master
- package: google.golang.org/grpc
  version: master
- package: github.com/go-sql-driver/mysql
  version: master
- package: github.com/garyburd/redigo
  version: master
  subpackages:
  - redis
- package: github.com/satori/go.uuid
  version: master
- package: golang.org/x/crypto
  version: master
  subpackages:
  - pkcs12
- package: golang.org/x/net
  version: master
- package: github.com/golang/glog
  version: master
- package: github.com/spf13/cobra
  version: master
- package: github.com/spf13/viper
  version: master
- package: golang.org/x/net
  version: master
```


## Elasticsearch 2.3 官方 Dockerfile

### Dockefile 解析

```
# 使用 Dockerhub 的 java:8-jre 作为基础镜像，elashticsearch 依赖于 jdk7 以上版本
FROM java:8-jre

# grab gosu for easy step-down from root
# elashticsearch 不能用 root 用户运行，所以安装 gosu
# 用法: ./gosu user-spec command [args]
# 这样可以用指定的用户运行指定的程序
# gosu 版本通过 GOSU_VERSION 指定
# 
# - wget 下载
# - mktemp -d 创建临时目录
# - gpg 去公钥服务器下载公钥并校验
# - 增加 gosu 执行权限
# - gosu nobody true 切换到 nobody 用户，安全
ENV GOSU_VERSION 1.7
RUN set -x \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true

# apt-key 是 Debian 软件包的安全管理工具。每个发布的 deb 包，都是通过密钥认证的，apt-key 用来管理密钥
# https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-repositories.html
# https://packages.elasticsearch.org/GPG-KEY-elasticsearch
RUN apt-key adv --keyserver ha.pool.sks-keyservers.net --recv-keys 46095ACC8548582C1A2699A9D27D666CD88E42B4

# 大版本，小版本，版本库 URL
ENV ELASTICSEARCH_MAJOR 2.3
ENV ELASTICSEARCH_VERSION 2.3.3
ENV ELASTICSEARCH_REPO_BASE http://packages.elasticsearch.org/elasticsearch/2.x/debian

RUN echo "deb $ELASTICSEARCH_REPO_BASE stable main" > /etc/apt/sources.list.d/elasticsearch.list

# 安装 ELASTICSEARCH
RUN set -x \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends elasticsearch=$ELASTICSEARCH_VERSION \
	&& rm -rf /var/lib/apt/lists/*

# 将 ELASTICSEARCH的bin 目录加入环境变量目录
ENV PATH /usr/share/elasticsearch/bin:$PATH

# 切换工作目录
WORKDIR /usr/share/elasticsearch

# 工作目录下面新建四个目录，并修改拥有者为 elasticsearch
RUN set -ex \
	&& for path in \
		./data \
		./logs \
		./config \
		./config/scripts \
	; do \
		mkdir -p "$path"; \
		chown -R elasticsearch:elasticsearch "$path"; \
	done

#将 config 目录下配置文件 elasticsearch.yml 和 logging.yml 放到工作目录下，详见 github
COPY config ./config

# 数据卷映射
VOLUME /usr/share/elasticsearch/data

# 将入口执行文件放到 "/" 根目录下面
COPY docker-entrypoint.sh /

# 端口映射
EXPOSE 9200 9300

# 容器启动入口，/docker-entrypoint.sh 是入口文件 ，elasticsearch 是参数
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["elasticsearch"]
```

### docker-entrypoint.sh 解析

```
#!/bin/bash

# set -e 表示若 shell 中的指令不返回 0，立即退出 shell
set -e

# Add elasticsearch as command if needed
if [ "${1:0:1}" = '-' ]; then
	set -- elasticsearch "$@"
fi

# Drop root privileges if we are running elasticsearch
# allow the container to be started with `--user`
# 如果参数 1 是 elasticsearch，并且是 root 用户
if [ "$1" = 'elasticsearch' -a "$(id -u)" = '0' ]; then
    # Change the ownership of user-mutable directories to elasticsearch
	# 则变更 /usr/share/elasticsearch/data 的拥有者为 elasticsearch
	chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data
	
	# 重新设置参数列表 "$@" 的值，即变更为 gosu elasticsearch "$@"
	set -- gosu elasticsearch "$@"
	# 变更后，最后的 exec "$@" 实际为 exec gosu elasticsearch "$@"
fi

# As argument is not related to elasticsearch,
# then assume that user wants to run his own process,
# for example a `bash` shell to explore this image
# 如果参数中没有 elasticsearch ，表示用户希望运行自己的其他进程
# 例如通过 bash 进入容器内部
exec "$@"
```


