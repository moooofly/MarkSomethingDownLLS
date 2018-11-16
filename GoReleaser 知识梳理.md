# GoReleaser 知识梳理

Ref:

- https://github.com/goreleaser/goreleaser
- https://goreleaser.com/
- https://github.com/goreleaser/old-go-releaser
- https://github.com/goreleaser/goreleaser/blob/master/Dockerfile
- https://raw.githubusercontent.com/caarlos0/go-releaser/master/release


----------

## 使用 tips

使用步骤：

```
# 1. 编写 go 程序
# 2. 生成 .goreleaser.yaml 文件
goreleaser init
# 3. 修改 .goreleaser.yaml 文件
# 4. 创建 GITHUB_TOKEN 并导出
export GITHUB_TOKEN='YOUR_TOKEN'
# 5. 执行 git add/commit
# 6. 为当前 commit 打 tag
git tag -a v0.1.0 -m "xxx"
# 7. 启用 GO111MODULE
export GO111MODULE=on
# 8. 以 dry run 模式运行
goreleaser release --rm-dist --skip-publish
# 9. 以 real 模式运行
goreleaser release --rm-dist
```

其他注意事项：

- GoReleaser enforces semantic versioning and will error on non compliant tags.
- Unfortunately, GoReleaser does not support CGO.
- GoReleaser requires a GitHub API token with the repo scope selected to deploy the artifacts to GitHub. This token should be added to the environment variables as `GITHUB_TOKEN`. Alternatively, you can provide the GitHub token in a file. GoReleaser will check `~/.config/goreleaser/github_token` by default, you can change that in the `.goreleaser.yml` file.

```
# .goreleaser.yml
env_files:
  github_token: ~/.path/to/my/token
```

- By default, GoReleaser will create its artifacts in the `./dist` folder. If you must, you can change it by setting it in the .goreleaser.yml file:

```
# .goreleaser.yml
dist: another-folder-that-is-not-dist
```

- Default wise GoReleaser sets three ldflags
    - `main.version`: Current Git tag (the `v` prefix is stripped) or the name of the snapshot, if you’re using the `--snapshot` flag
    - `main.commit`: Current git commit SHA
    - `main.date`: Date according [RFC3339](https://golang.org/pkg/time/#pkg-constants)

- GoReleaser was built from the very first commit with the idea of running it as part of the CI pipeline in mind.
    

## 简介

> GoReleaser is a **release automation tool** for Go projects, the goal is to simplify the build, release and publish steps while providing variant customization options for all steps.

GoReleaser 是一个针对 Go 项目的发布自动化工具；

> GoReleaser is **built for CI tools**; you only need to download and execute it in your build script. You can customize your release process by creating a `.goreleaser.yml` file.

- GoReleaser 是为 CI 工具而创造；
- 你可以通过创建一个 `.goreleaser.yml` 文件来定制化你的 release 过程；

> The idea started with a [simple shell script](https://github.com/goreleaser/old-go-releaser), but it quickly became more complex and I also wanted to publish binaries via Homebrew taps, which would have made the script even more hacky, so I let go of that and rewrote the whole thing in Go.

GoReleaser 最初的形式是一个 shell 脚本；但随着后续越来越多的需求，问题变得越来越复杂了，因此基于 Go 重写了该工具；

## 安装

### 基于 homebrew

```
brew install goreleaser/tap/goreleaser
```

### 基于 Docker

You can use Docker to do simple releases.

```
$ docker run --rm --privileged \
  -v $PWD:/go/src/github.com/user/repo \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -w /go/src/github.com/user/repo \
  -e GITHUB_TOKEN \
  -e DOCKER_USERNAME \
  -e DOCKER_PASSWORD \
  goreleaser/goreleaser release
```


> Note that the image will almost always have the last stable Go version.

> If you need more things, you are encouraged to have your own image. You can always use GoReleaser’s [own Dockerfile](https://github.com/goreleaser/goreleaser/blob/master/Dockerfile) as an example though.

如果需要定制化更多的东西，可以构建自己的镜像；

取自 Dockerfile 文件

```
FROM golang:1.11
RUN apt-get update && \
	apt-get install -y --no-install-recommends rpm git apt-transport-https curl gnupg2 software-properties-common && \
	curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - && \
	add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" && \
	apt-get update && \
	apt-get install -y --no-install-recommends docker-ce &&\
	rm -rf /var/lib/apt/lists/*
COPY goreleaser /bin/goreleaser
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD [ "-h" ]
```

取自 scripts/entrypoint.sh

```
#!/usr/bin/env bash

if [ -n "$DOCKER_USERNAME" ] && [ -n "$DOCKER_PASSWORD" ]; then
    echo "Login to the docker..."
    docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
fi

goreleaser $@
```


### 手动

Download your preferred flavor from the [releases page](https://github.com/goreleaser/goreleaser/releases/latest) and install manually.

### 基于 go get

> Note: this method requires **Go 1.10+**.

```
$ go get -d github.com/goreleaser/goreleaser
$ cd $GOPATH/src/github.com/goreleaser/goreleaser
$ dep ensure -vendor-only
$ make setup build
```

> It is recommended to also run `dep ensure` to make sure that the dependencies are in the correct versions.

dep 项目地址：https://github.com/golang/dep

安装命令

```
curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
```

运行效果

```
[#23#root@ubuntu-1604 /go/src/github.com/goreleaser/goreleaser]$dep ensure -vendor-only
could not find project Gopkg.toml, use dep init to initiate a manifest
[#24#root@ubuntu-1604 /go/src/github.com/goreleaser/goreleaser]$
```

放弃使用~~

## Quick Start

- 创建 go 程序

```
// main.go
package main

func main() {
  println("Ba dum, tss!")
}
```

- 运行 `goreleaser init` 创建 `.goreleaser.yaml` 文件

```
[#10#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser init

   • Generating .goreleaser.yml file
   • config created; please edit accordingly to your needs file=.goreleaser.yml

[#11#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

默认生成的文件内容如下

```
[#15#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$cat .goreleaser.yml
# This is an example goreleaser.yaml file with some sane defaults.
# Make sure to check the documentation at http://goreleaser.com
before:
  hooks:
    # you may remove this if you don't use vgo
    - go mod download
    # you may remove this if you don't need go generate
    - go generate ./...
builds:
- env:
  - CGO_ENABLED=0
archive:
  replacements:
    darwin: Darwin
    linux: Linux
    windows: Windows
    386: i386
    amd64: x86_64
checksum:
  name_template: 'checksums.txt'
snapshot:
  name_template: "{{ .Tag }}-next"
changelog:
  sort: asc
  filters:
    exclude:
    - '^docs:'
    - '^test:'
[#16#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

- 定制化修改上述配置文件

> 详见：https://goreleaser.com/build/


- 创建 GITHUB_TOKEN 并导出为环境变量

> You’ll need to export a `GITHUB_TOKEN` environment variable, which should contain a valid GitHub token with the `repo` scope. It will be used to deploy releases to your GitHub repository. You can create a token [here](https://github.com/settings/tokens/new).

```
[#16#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$export GITHUB_TOKEN=6b1c019a5708a0b55dff66260742a135f05e96fc
```

- 为你的项目打 tag

> 注意：这里需要说明一点，即在基于 tag 创建 release 的模式下，最终执行 goreleaser 时要求 tag 打在 last commit id 上，否则会报错

goreleaser 默认是基于 tag 创建 release 的

> GoReleaser will use the latest Git tag of your repository. Create a tag and push it to GitHub:

```
[#17#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git tag -a v0.1.0 -m "First release"
[#18#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git tag
v0.1.0
[#19#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#19#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git push origin v0.1.0
Counting objects: 1, done.
Writing objects: 100% (1/1), 163 bytes | 0 bytes/s, done.
Total 1 (delta 0), reused 0 (delta 0)
To git@github.com:moooofly/playground
 * [new tag]         v0.1.0 -> v0.1.0
[#20#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

> Attention: Check if your tag adheres to semantic versioning.

如果你想要基于特定的 commit 创建 release ，而不是基于 tag ，则可以使用 `--snapshot`

> If you don’t want to create a tag yet, you can also create a release based on the latest commit by using the `--snapshot` flag.


- 运行 goreleaser 命令

失败一：需要启用 GO111MODULE

> 因为默认生成的 `.goreleaser.yml` 文件中定义了 before -> hooks -> `go mod download` 

```
[#21#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
   ⨯ release failed after 0.03s error=hook failed: go mod download
go: modules disabled inside GOPATH/src by GO111MODULE=auto; see 'go help modules'

[#22#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

失败二：需要将相关文件都 git add 、 git commit 了

```
[#22#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$export GO111MODULE=on
[#23#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#23#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.1.0, commit c9243aff643730694ddc99f43095b513a864a009
   ⨯ release failed after 0.24s error=git is currently in a dirty state:
?? go.mod
?? gorelease_example/

[#24#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

失败三：基于 tag 的 release 必须对应 last commit id 的值

```
[#31#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$cd ..
[#37#root@ubuntu-1604 /go/src/github.com/moooofly/playground]$mv go.mod gorelease_example/
[#39#root@ubuntu-1604 /go/src/github.com/moooofly/playground]$cd gorelease_example/
[#40#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$ll
total 24
drwxr-xr-x  3 root root 4096 Nov 16 17:19 ./
drwxr-xr-x 25 root root 4096 Nov 16 17:19 ../
drwxr-xr-x  2 root root 4096 Nov 16 17:14 dist/
-rw-r--r--  1 root root   38 Nov 16 17:14 go.mod
-rw-r--r--  1 root root  609 Nov 16 16:14 .goreleaser.yml
-rw-r--r--  1 root root   73 Nov 16 16:05 main.go
[#41#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#44#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git add .
[#45#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git commit -m "add go module support"
[master afbccab] add go module support
 1 file changed, 1 insertion(+)
 create mode 100644 gorelease_example/go.mod
[#46#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#46#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.1.0, commit afbccab4db7e0a45369c2f6fcaef60774ba73c93
   ⨯ release failed after 0.10s error=git tag v0.1.0 was not made against commit afbccab4db7e0a45369c2f6fcaef60774ba73c93
[#47#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

调整 commit history

```
[#47#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git lg1 -5
* afbccab - (4 minutes ago) add go module support — moooofly (HEAD -> master)
* 440a2a1 - (7 minutes ago) add go module support — moooofly
* c9243af - (2 weeks ago) update comments — moooofly (tag: v0.1.0, origin/master, origin/HEAD)
* 35c2b13 - (2 weeks ago) add convert_int32_to_string_example — moooofly
* 1783947 - (3 weeks ago) add cobra_example — moooofly
[#48#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#48#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git rebase -i HEAD~3
[detached HEAD 0353f3c] update comments
 Date: Thu Nov 1 14:03:45 2018 +0800
 4 files changed, 37 insertions(+), 1 deletion(-)
 create mode 100644 gorelease_example/.goreleaser.yml
 create mode 100644 gorelease_example/go.mod
 create mode 100644 gorelease_example/main.go
Successfully rebased and updated refs/heads/master.
[#49#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#49#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#49#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git lg1 -5
* 0353f3c - (2 weeks ago) update comments — moooofly (HEAD -> master)
| * c9243af - (2 weeks ago) update comments — moooofly (tag: v0.1.0, origin/master, origin/HEAD)
|/
* 35c2b13 - (2 weeks ago) add convert_int32_to_string_example — moooofly
* 1783947 - (3 weeks ago) add cobra_example — moooofly
* 01860cc - (3 weeks ago) update by 'cobra add' — moooofly
[#50#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$

[#52#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git tag -d v0.1.0
Deleted tag 'v0.1.0' (was 27695e7)
[#53#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#55#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git lg1 -5
* 0353f3c - (2 weeks ago) update comments — moooofly (HEAD -> master)
| * c9243af - (2 weeks ago) update comments — moooofly (origin/master, origin/HEAD)
|/
* 35c2b13 - (2 weeks ago) add convert_int32_to_string_example — moooofly
* 1783947 - (3 weeks ago) add cobra_example — moooofly
* 01860cc - (3 weeks ago) update by 'cobra add' — moooofly
[#56#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$

[#58#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git push -f
Total 0 (delta 0), reused 0 (delta 0)
To git@github.com:moooofly/playground
 + c9243af...0353f3c master -> master (forced update)
[#58#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#59#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git push -f origin v0.1.0
Counting objects: 10, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (9/9), done.
Writing objects: 100% (10/10), 1.21 KiB | 0 bytes/s, done.
Total 10 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To git@github.com:moooofly/playground
 + 27695e7...38a389f v0.1.0 -> v0.1.0 (forced update)
[#60#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

成功

```
[#60#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.1.0, commit 0353f3ce73eeb5303093682f92a5c9088d9ad9e9
   • WRITING EFFECTIVE CONFIG FILE
      • writing                   config=dist/config.yaml
   • GENERATING CHANGELOG
      • writing                   changelog=dist/CHANGELOG.md
   • LOADING ENVIRONMENT VARIABLES
   • BUILDING BINARIES
      • building                  binary=dist/linux_386/playground
      • building                  binary=dist/darwin_amd64/playground
      • building                  binary=dist/linux_amd64/playground
      • building                  binary=dist/darwin_386/playground
   • ARCHIVES
      • creating                  archive=dist/playground_0.1.0_Linux_i386.tar.gz
      • creating                  archive=dist/playground_0.1.0_Darwin_i386.tar.gz
      • creating                  archive=dist/playground_0.1.0_Darwin_x86_64.tar.gz
      • creating                  archive=dist/playground_0.1.0_Linux_x86_64.tar.gz
   • LINUX PACKAGES WITH NFPM
      • skipped                   reason=no output formats configured
   • SNAPCRAFT PACKAGES
      • skipped                   reason=no summary nor description were provided
   • CALCULATING CHECKSUMS
      • checksumming              file=playground_0.1.0_Linux_x86_64.tar.gz
      • checksumming              file=playground_0.1.0_Darwin_i386.tar.gz
      • checksumming              file=playground_0.1.0_Darwin_x86_64.tar.gz
      • checksumming              file=playground_0.1.0_Linux_i386.tar.gz
   • SIGNING ARTIFACTS
      • skipped                   reason=artifact signing is disabled
   • DOCKER IMAGES
      • skipped                   reason=docker section is not configured
   • PUBLISHING
      • S3
      • skipped                   reason=s3 section is not configured
      • releasing with HTTP PUT
      • skipped                   reason=put section is not configured
      • Artifactory
      • skipped                   reason=artifactory section is not configured
      • Docker images
      • Snapcraft Packages
      • GitHub Releases
      • creating or updating release repo=moooofly/playground tag=v0.1.0
      • release updated           url=https://github.com/moooofly/playground/releases/tag/v0.1.0
      • uploading to release      file=dist/checksums.txt name=checksums.txt
      • uploading to release      file=dist/playground_0.1.0_Darwin_x86_64.tar.gz name=playground_0.1.0_Darwin_x86_64.tar.gz
      • uploading to release      file=dist/playground_0.1.0_Darwin_i386.tar.gz name=playground_0.1.0_Darwin_i386.tar.gz
      • uploading to release      file=dist/playground_0.1.0_Linux_i386.tar.gz name=playground_0.1.0_Linux_i386.tar.gz
      • uploading to release      file=dist/playground_0.1.0_Linux_x86_64.tar.gz name=playground_0.1.0_Linux_x86_64.tar.gz
      • homebrew tap formula
      • skipped                   reason=brew section is not configured
      • scoop manifest
      • skipped                   reason=scoop section is not configured
   • release succeeded after 15.91s

[#61#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```


----------


新改动需求（测试目的）：

- 在上述操作之后，由于生成了 dist/ 目录，所以打算添加 .gitignore 文件
- 打算去掉 .goreleaser.yml 文件中对 windows 平台和 i386 的支持

```
[#62#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	dist/

nothing added to commit but untracked files present (use "git add" to track)
[#63#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#63#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#63#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$ll
total 24
drwxr-xr-x  3 root root 4096 Nov 16 17:25 ./
drwxr-xr-x 25 root root 4096 Nov 16 17:19 ../
drwxr-xr-x  6 root root 4096 Nov 16 17:30 dist/
-rw-r--r--  1 root root   38 Nov 16 17:25 go.mod
-rw-r--r--  1 root root  609 Nov 16 17:25 .goreleaser.yml
-rw-r--r--  1 root root   73 Nov 16 17:25 main.go
[#64#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$ll dist/
total 2232
drwxr-xr-x 6 root root   4096 Nov 16 17:30 ./
drwxr-xr-x 3 root root   4096 Nov 16 17:25 ../
-rw-r--r-- 1 root root    929 Nov 16 17:30 CHANGELOG.md
-r--r--r-- 1 root root    410 Nov 16 17:30 checksums.txt
-rw-r--r-- 1 root root   1887 Nov 16 17:30 config.yaml
drwxr-xr-x 2 root root   4096 Nov 16 17:30 darwin_386/
drwxr-xr-x 2 root root   4096 Nov 16 17:30 darwin_amd64/
drwxr-xr-x 2 root root   4096 Nov 16 17:30 linux_386/
drwxr-xr-x 2 root root   4096 Nov 16 17:30 linux_amd64/
-rw-r--r-- 1 root root 583527 Nov 16 17:30 playground_0.1.0_Darwin_i386.tar.gz
-rw-r--r-- 1 root root 607240 Nov 16 17:30 playground_0.1.0_Darwin_x86_64.tar.gz
-rw-r--r-- 1 root root 513895 Nov 16 17:30 playground_0.1.0_Linux_i386.tar.gz
-rw-r--r-- 1 root root 534557 Nov 16 17:30 playground_0.1.0_Linux_x86_64.tar.gz
[#65#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

添加 .gitignore 文件

```
[#70#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$vi .gitignore
[#71#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	.gitignore

nothing added to commit but untracked files present (use "git add" to track)
[#72#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

变更 .goreleaser.yml 文件

```
[#73#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$vi .goreleaser.yml
[#74#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#74#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   .goreleaser.yml

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	.gitignore

no changes added to commit (use "git add" and/or "git commit -a")
[#75#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

失败一：认为存在 dist 目录，要求将其删除

```
[#76#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   ⨯ release failed after 0.09s error=dist is not empty, remove it before running goreleaser or use the --rm-dist flag
[#77#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

失败一：认为当前处于 dirty state（故意没有指定 git add/commit/push 之类的）

```
[#77#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser --rm-dist

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
      • --rm-dist is set, cleaning it up
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.1.0, commit 0353f3ce73eeb5303093682f92a5c9088d9ad9e9
   ⨯ release failed after 0.10s error=git is currently in a dirty state:
 M gorelease_example/.goreleaser.yml
?? gorelease_example/.gitignore

[#78#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

失败二：基于获取到的当前最新 tag 和最新 commit id ，发现不匹配

```
[#79#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git add .
[#80#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git commit -m "add .gitignore and update .goreleaser.yml"
[master 2844281] add .gitignore and update .goreleaser.yml
 2 files changed, 1 insertion(+), 2 deletions(-)
 create mode 100644 gorelease_example/.gitignore
[#81#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$

[#82#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.1.0, commit 28442819e29cfae066313cf3539cc229761a5503
   ⨯ release failed after 0.09s error=git tag v0.1.0 was not made against commit 28442819e29cfae066313cf3539cc229761a5503
[#83#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

本地更新 tag 信息（注意，并没有 push 到远端），执行成功

```
[#83#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git tag -a v0.2.0 -m "add .gitignore and update .goreleaser.yml"
[#84#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#84#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$git tag
v0.1.0
v0.2.0
[#85#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$

[#85#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.2.0, commit 28442819e29cfae066313cf3539cc229761a5503
   • WRITING EFFECTIVE CONFIG FILE
      • writing                   config=dist/config.yaml
   • GENERATING CHANGELOG
      • writing                   changelog=dist/CHANGELOG.md
   • LOADING ENVIRONMENT VARIABLES
   • BUILDING BINARIES
      • building                  binary=dist/darwin_386/playground
      • building                  binary=dist/linux_386/playground
      • building                  binary=dist/darwin_amd64/playground
      • building                  binary=dist/linux_amd64/playground
   • ARCHIVES
      • creating                  archive=dist/playground_0.2.0_Darwin_386.tar.gz
      • creating                  archive=dist/playground_0.2.0_Linux_386.tar.gz
      • creating                  archive=dist/playground_0.2.0_Darwin_x86_64.tar.gz
      • creating                  archive=dist/playground_0.2.0_Linux_x86_64.tar.gz
   • LINUX PACKAGES WITH NFPM
      • skipped                   reason=no output formats configured
   • SNAPCRAFT PACKAGES
      • skipped                   reason=no summary nor description were provided
   • CALCULATING CHECKSUMS
      • checksumming              file=playground_0.2.0_Darwin_x86_64.tar.gz
      • checksumming              file=playground_0.2.0_Darwin_386.tar.gz
      • checksumming              file=playground_0.2.0_Linux_386.tar.gz
      • checksumming              file=playground_0.2.0_Linux_x86_64.tar.gz
   • SIGNING ARTIFACTS
      • skipped                   reason=artifact signing is disabled
   • DOCKER IMAGES
      • skipped                   reason=docker section is not configured
   • PUBLISHING
      • S3
      • skipped                   reason=s3 section is not configured
      • releasing with HTTP PUT
      • skipped                   reason=put section is not configured
      • Artifactory
      • skipped                   reason=artifactory section is not configured
      • Docker images
      • Snapcraft Packages
      • GitHub Releases
      • creating or updating release repo=moooofly/playground tag=v0.2.0
      • release updated           url=https://github.com/moooofly/playground/releases/tag/v0.2.0
      • uploading to release      file=dist/checksums.txt name=checksums.txt
      • uploading to release      file=dist/playground_0.2.0_Darwin_386.tar.gz name=playground_0.2.0_Darwin_386.tar.gz
      • uploading to release      file=dist/playground_0.2.0_Linux_386.tar.gz name=playground_0.2.0_Linux_386.tar.gz
      • uploading to release      file=dist/playground_0.2.0_Linux_x86_64.tar.gz name=playground_0.2.0_Linux_x86_64.tar.gz
      • uploading to release      file=dist/playground_0.2.0_Darwin_x86_64.tar.gz name=playground_0.2.0_Darwin_x86_64.tar.gz
      • homebrew tap formula
      • skipped                   reason=brew section is not configured
      • scoop manifest
      • skipped                   reason=scoop section is not configured
   • release succeeded after 6.74s

[#86#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

### Dry Run 模式


> If you want to test everything before doing a release "for real", you can use the `--skip-publish` flag, which will only build and package things.


```
[#91#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser -h

usage: goreleaser [<flags>] <command> [<args> ...]

Deliver Go binaries as fast and easily as possible

Flags:
  -h, --help     Show context-sensitive help (also try --help-long and --help-man).
  -v, --version  Show application version.

Commands:
  help [<command>...]
    Show help.

  init
    Generates a .goreleaser.yml file

  release* [<flags>]
    Releases the current project


[#92#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#92#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser release -h

usage: goreleaser release [<flags>]

Releases the current project

Flags:
  -h, --help                    Show context-sensitive help (also try --help-long and --help-man).
  -v, --version                 Show application version.
  -f, --config=.goreleaser.yml  Load configuration from file
      --release-notes=notes.md  Load custom release notes from a markdown file
      --snapshot                Generate an unversioned snapshot release, skipping all validations and without publishing any artifacts
      --skip-publish            Generates all artifacts but does not publish them anywhere
      --skip-sign               Skips signing the artifacts
      --skip-validate           Skips all git sanity checks
      --rm-dist                 Remove the dist folder before building
  -p, --parallelism=4           Amount of slow tasks to do in concurrently
      --debug                   Enable debug mode
      --timeout=30m             Timeout to the entire release process

[#93#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

```
[#93#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser release --skip-publish

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
   ⨯ release failed after 0.07s error=dist is not empty, remove it before running goreleaser or use the --rm-dist flag
[#94#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#94#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#94#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$goreleaser release --skip-publish --rm-dist

   • releasing using goreleaser dev...
   • loading config file       file=.goreleaser.yml
   • SETTING DEFAULTS
   • RUNNING BEFORE HOOKS
      • running go mod download
      • running go generate ./...
   • CHECKING ./DIST
      • --rm-dist is set, cleaning it up
   • GETTING AND VALIDATING GIT STATE
      • releasing v0.2.0, commit 28442819e29cfae066313cf3539cc229761a5503
   • WRITING EFFECTIVE CONFIG FILE
      • writing                   config=dist/config.yaml
   • GENERATING CHANGELOG
      • writing                   changelog=dist/CHANGELOG.md
   • LOADING ENVIRONMENT VARIABLES
      • skipped                   reason=publishing is disabled
   • BUILDING BINARIES
      • building                  binary=dist/darwin_386/playground
      • building                  binary=dist/linux_386/playground
      • building                  binary=dist/linux_amd64/playground
      • building                  binary=dist/darwin_amd64/playground
   • ARCHIVES
      • creating                  archive=dist/playground_0.2.0_Linux_x86_64.tar.gz
      • creating                  archive=dist/playground_0.2.0_Linux_386.tar.gz
      • creating                  archive=dist/playground_0.2.0_Darwin_x86_64.tar.gz
      • creating                  archive=dist/playground_0.2.0_Darwin_386.tar.gz
   • LINUX PACKAGES WITH NFPM
      • skipped                   reason=no output formats configured
   • SNAPCRAFT PACKAGES
      • skipped                   reason=no summary nor description were provided
   • CALCULATING CHECKSUMS
      • checksumming              file=playground_0.2.0_Darwin_x86_64.tar.gz
      • checksumming              file=playground_0.2.0_Darwin_386.tar.gz
      • checksumming              file=playground_0.2.0_Linux_x86_64.tar.gz
      • checksumming              file=playground_0.2.0_Linux_386.tar.gz
   • SIGNING ARTIFACTS
      • skipped                   reason=artifact signing is disabled
   • DOCKER IMAGES
      • skipped                   reason=docker section is not configured
   • PUBLISHING
      • skipped                   reason=publishing is disabled
   • release succeeded after 0.89s

[#95#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```