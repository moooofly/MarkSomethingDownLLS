# 常见 Dockerfile 问题

> Ref: [9 Common Dockerfile Mistakes](https://runnable.com/blog/9-common-dockerfile-mistakes)

- [Running apt-get](#running-apt-get)
- [Using ADD instead of COPY](#using-add-instead-of-copy)
- [Adding your entire application directory in one line](#adding-your-entire-application-directory-in-one-line)
- [Using :latest](#using-latest)
- [Using external services during the build](#using-external-services-during-the-build)
- [Adding EXPOSE and ENV at the top of your Dockerfile](#adding-expose-and-env-at-the-top-of-your-dockerfile)
- [Multiple FROM statements](#multiple-from-statements)
- [Multiple services running in the same container](#multiple-services-running-in-the-same-container)
- [Using VOLUME in your build process](#using-volume-in-your-build-process)


## Running apt-get

> The first is running `apt-get upgrade`. This will update all your packages to their latests versions — which is bad because **it prevents your Dockerfile from creating consistent, immutable builds**.

使用 `apt-get upgrade` 会破坏一致性；

> Another issue is with running `apt-get update` in a different line than running your `apt-get install` command. The reason why this is bad is because a line with only `apt-get update` will get cached by the build and won't actually run every time you need to run `apt-get install`. Instead, make sure you run `apt-get update` in the same line with all the packages to ensure all are updated correctly.

`apt-get update` 和 `apt-get install` 必须在一个 RUN 指令中使用，否则会存在缓存失效问题；

a good example

```
# From https://github.com/docker-library/golang
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  g++ \
  gcc \
  libc6-dev \
  make \
  && rm -rf /var/lib/apt/lists/*
```

## Using ADD instead of COPY

> While similar, `ADD` and `COPY` are actually different commands. `COPY` is the simplest of the two, since it just copies a file or a directory from your host to your image. `ADD` does this too, but also has some more magical features like extracting TAR files or fetching files from remote URLs. In order to reduce the complexity of your Dockerfile and prevent some unexpected behavior, it's usually best to always use `COPY` to copy your files.

只使用 COPY 就对了；

```
FROM busybox:1.24

ADD example.tar.gz /add # Will untar the file into the ADD directory
COPY example.tar.gz /copy # Will copy the file directly
```

## Adding your entire application directory in one line

> Being explicit about what part of your code should be included in your build, and at what time, might be the most important thing you can do to significantly speed up your builds.

加速构建的关键在于合理拆分被包含的代码，以便最大程度利用缓存；

bad one:

```
# !!! ANTIPATTERN !!!
COPY ./my-app/ /home/app/
RUN npm install # or RUN pip install or RUN bundle install
# !!! ANTIPATTERN !!!
```

good one:

```
COPY ./my-app/package.json /home/app/package.json # Node/npm packages
WORKDIR /home/app/
RUN npm install

# Maybe you have to install python packages too?
COPY ./my-app/requirements.txt /home/app/requirements.txt
RUN pip install -r requirements.txt
COPY ./my-app/ /home/app/
```

## Using :latest

> Many Dockerfiles use the `FROM node:latest` pattern at the top of their Dockerfiles to pull the latest image from a Docker registry. **While simple, using the latest tag for an image means that your build can suddenly break if that image gets updated**. Figuring this out might prove to be very difficult, since the maintainer of the Dockerfile didn’t actually make any changes. To prevent this, just make sure you use a specific tag of an image (example: `node:6.2.1`). This will ensure your Dockerfile remains immutable.

不要使用 `:latest` 就对了

## Using external services during the build

> Many people forget the difference between building a Docker image and running a Docker container. When building an image, Docker reads the commands in your Dockerfile and creates an image from it. **Your image should be immutable and reusable until any of your dependencies or your code changes**. This process should be completely independent of any other container. Anything that requires interaction with other containers or other services (like a database) should happen when you run the container.

镜像构建以 immutable 和 reusable 为首要考虑因素；

## Adding EXPOSE and ENV at the top of your Dockerfile

> `EXPOSE` and `ENV` are cheap commands to run. If you bust the cache for them, rebuilding them is almost instantaneous. Therefore, **it’s best to declare these commands as late as possible**. You should only ever declare  `ENV`s whenever you need them in your build process. If they’re not needed during build time, then they should be at the end of your Dockerfile, along with `EXPOSE`.

`EXPOSE` 和 `ENV` 的变更会破会 build cache ，因此最佳实践为尽可能晚的声明；

一个好的示例

```
ENV GOLANG_VERSION 1.7beta1
ENV GOLANG_DOWNLOAD_URL https://golang.org/dl/go$GOLANG_VERSION.linux-amd64.tar.gz
ENV GOLANG_DOWNLOAD_SHA256 a55e718935e2be1d5b920ed262fd06885d2d7fc4eab7722aa02c205d80532e3b

RUN curl -fsSL "$GOLANG_DOWNLOAD_URL" -o golang.tar.gz \
 && echo "$GOLANG_DOWNLOAD_SHA256  golang.tar.gz" | sha256sum -c - \
 && tar -C /usr/local -xzf golang.tar.gz \
 && rm golang.tar.gz

ENV GOPATH /go
ENV PATH $GOPATH/bin:/usr/local/go/bin:$PATH
```

## Multiple FROM statements

> It might be tempting to try to combine different images together by using multiple `FROM` statements; this won’t work. Instead, Docker will just use the last `FROM` specified and ignore everything before that.

在 Dockerfile 中想要通过使用多个 `FROM` 达到将多个不同的镜像合并的目的是行不通的；

## Multiple services running in the same container

> It's a well established best-practice that **every different service which composes your application should run in its own container**. It's tempting to add multiple services to one docker image, but this practice has some downsides.
> 
> - First, you'll make it more difficult to horizontally scale your application. 
> - Second, the additional dependencies and layers will make your build slower. 
> - Finally, it'll make your Dockerfile harder to write, maintain, and debug.

最佳实践为每个容器中只运行一个服务；

一个容器中运行多个服务的缺点：

- 难以水平扩展
- 由于存在额外的依赖，将导致 layer 数量的增加，进而导致构建速度变慢
- 难以编写、维护，以及调试 Dockerfile

> If you want to quickly setup a Django+Nginx application for development, it might make sense to just run them in the same container and have a different Dockerfile in production where they run separately.

## Using VOLUME in your build process

> **Volumes in your image are added when you run your container, not when you build it**. In a similar way to #5, **you should never interact with your declared volume in your build process**. Rather, you should only use it when you run the container.

在构建镜像时 VOLUME 是不能被使用的；


good one

```
FROM busybox:1.24
RUN echo "hello-world!!!!" > /myfile.txt

CMD ["cat", "/myfile.txt"]

$ docker run volume-in-build
hello-world!!!!
```

bad one

```
FROM busybox:1.24
VOLUME /data
RUN echo "hello-world!!!!" > /data/myfile.txt

CMD ["cat", "/data/myfile.txt"]

$ docker run volume-in-build
cat: can't open '/data/myfile.txt': No such file or directory
```

> An interesting gotcha for this is that if any of your previous layers has a `VOLUME` declaration (which might be several `FROM`s away) you will still run into the same issue. For that reason, it's a good idea to be aware of what volumes your parent images declare. Use `docker inspect` if you run into problems.

还存在一种情况，如果之前的某层 layer 进行了 `VOLUME` 声明，则也会导致上述问题；

