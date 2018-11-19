# old-go-releaser 项目研究

| key | value |
| -- | -- |
| 项目名称 | go-releaser (shell 版本) |
| 地址 | https://github.com/goreleaser/old-go-releaser  |
| 实现语言 | shell |
| 日期 | 2018-11-19 |
| 一句话描述 | 基于 shell 脚本 build 和 release 二进制程序到 github |
| 详细说明 | <br> 1. 基于 `gox` 进行多平台项目编译（其实并没有）；<br> 2. 支持将 `main.version` ldflag 设置为当前 `git tag` 值 <br> 3. 打包相关文件并命名为 `binary_$(uname -s)_$(uname -m).tar.gz` <br> 4. 基于当前 tag 创建一个 github release ，并使用当前 tag 和前一次 tag 之间的 commit log 作为当前 release 的描述信息 <br> 5. 上传所有 `tar.gz` 文件到所创建的 release 中 |
| 使用约束 | <br> 1. Go 1.5 or higher <br> 2. 不支持 Windows <br> 3. 需要设置 `main.version` ldflag <br> 4. 脚本要求 README 和 LICENSE 文件必须存在 <br> 5. `dist` 目录中不应该包含任何重要的内容（因为每次都会被删除） |
| 项目依赖 | github.com/aktau/github-release |


----------

## 脚本说明

```
#!/bin/sh
set -e

# enables go 1.5 vendor experiment
export GO15VENDOREXPERIMENT=1

# cleanup stuff
cleanup() {
  rm -rf dist
}

# normalize Golang's OS and Arch to uname compatibles
normalize() {
  echo "$1" | sed \
    -e 's/darwin/Darwin/' \
    -e 's/linux/Linux/' \
    -e 's/freebsd/FreeBSD/' \
    -e 's/openbsd/OpenBSD/' \
    -e 's/netbsd/NetBSD/' \
    -e 's/386/i386/' \
    -e 's/amd64/x86_64/'
}

# builds the binaries with gox
build() {
  echo "Building $CURRENT..."
  for os in linux darwin; do
    for arch in amd64 386; do
      echo "  -> Building for $os on $arch..."
      GOOS="$os" GOARCH="$arch" go build \
        -ldflags="-s -w -X main.version=$CURRENT" \
        -o "dist/${BINARY}_${os}_${arch}/$BINARY" "$MAINGO"
    done
  done
}

# package the binaries in .tar.gz files
package() {
  # shellcheck disable=SC2039
  local folder filename
  echo "Packaging $CURRENT..."

  for folder in ./dist/*; do
    filename="$(normalize "$folder").tar.gz"
    test -n "$EXTRA_FILES" && cp -rf "$EXTRA_FILES" "$folder"
    cp -rf README* "$folder"
    cp -rf LICENSE* "$folder"
    tar -cvzf "$filename" --directory="$folder" .
  done
}

# release it to github
release() {
  echo "Releasing $CURRENT..."
  # shellcheck disable=SC2039
  local log description
  log="$(git log --pretty=oneline --abbrev-commit "$PREVIOUS".."$CURRENT")"
  description="$(printf '%s\n\n%s' "$log" "Built with: $(go version) and caarlos0/go-releaser")"
  go get -u -v github.com/aktau/github-release
  echo "Creating release $CURRENT..."
  github-release release \
    --user "$OWNER" \
    --repo "$REPO" \
    --tag "$CURRENT" \
    --description "$description" ||
      github-release edit \
        --user "$OWNER" \
        --repo "$REPO" \
        --tag "$CURRENT" \
        --description "$description"
}

# upload all tar.gz files to the previously created release
upload() {
  # shellcheck disable=SC2039
  local file
  for file in ./dist/*.tar.gz; do
    echo "--> Uploading $file..."
    github-release upload \
      --user "$OWNER" \
      --repo "$REPO" \
      --tag "$CURRENT" \
      --name "$(echo "$file" | sed 's:\./dist/::')" \
      --file "$file"
  done
}

usage() {
  cat <<EOF
NAME:
  $0 - Builds and releases a go binary to github releases
USAGE:
  $0 [options]
OPTIONS:
  -h, --help                Shows this screen
  -u, --user, --owner       Repository owner (e.g.: caarlos0)
  -r, --repo                Repository name (e.g.: go-releaser)
  -b, --binary              Binary final name (e.g.: antibody)
  -m, --main                Path to the main golang file (e.g.: cmd/main.go)
  -e, --extra               Extra files to package: (e.g.: USAGE.md)
EOF
}

test $# -eq 0 && {
  usage
  exit 0
}

while test $# -gt 0; do
  case "$1" in
    -u|--user|--owner)
      shift
      if [ $# -gt 0 ]; then
        export OWNER="$1"
        shift
      fi
      ;;
    -r|--repo)
      shift
      if [ $# -gt 0 ]; then
        export REPO="$1"
        shift
      fi
      ;;
    -b|--binary)
      shift
      if [ $# -gt 0 ]; then
        export BINARY="$1"
        shift
      fi
      ;;
    -m|--main)
      shift
      if [ $# -gt 0 ]; then
        export MAINGO="$1"
        shift
      fi
      ;;
    -e|--extra)
      shift
      if [ $# -gt 0 ]; then
        export EXTRA_FILES="$1"
        shift
      fi
      ;;
    --help|-h)
      usage
      exit 0
      ;;
    *)
      echo "Invalid option $1. Run $0 -h for help" >&2
      exit 1
      ;;
  esac
done

test -z "$OWNER" && {
  echo "Missing owner" >&2
  exit 1
}
test -z "$REPO" && {
  echo "Missing repository" >&2
  exit 1
}
test -z "$BINARY" && {
  echo "Missing binary name" >&2
  exit 1
}
test -z "$MAINGO" && {
  echo "Missing main go file" >&2
  exit 1
}

cleanup
CURRENT="$(git describe --tags --abbrev=0)"
PREVIOUS=$(git describe --tags --abbrev=0 --always "$CURRENT"^)
build
package
release
upload
```

要点

- 每次执行前，先删除 `dist/` ；
- 获取当前 tag 和前一次 tag 以便后续获取 commit log 作为 release 的 description ；
- build 中仅覆盖了 linux/darwin 以及 amd64/i386 的组合；
- 要求必须指定 owner/repository/binary/main 设置；
- release 行为依赖于 `github.com/aktau/github-release` ，并且通过 `-u` 选项强制更新；
- release 若不存在则创建，若存在则更新；
- release 能够成功的前提是必须提前设置好 GITHUB_TOKEN ，否则会看到 "error: empty token" ；

可能的改进点：

- 并没有使用 gox 进行多平台编译；
- 每次对 `github.com/aktau/github-release` 强制更新似乎不太好；
- build 的组合需要按需调整，并非所有；
- 对仓库目录层级的隐式要求，表现为最终的 release 包名是固定的，不能随意指定（使用 https://github.com/goreleaser/goreleaser 时没有这个问题）；
- release 的 description 段被强制增加了 "Built with: go version go1.11.1 linux/amd64 and caarlos0/go-releaser"；另外，整体格式也不是特别友好；
- 存在 "go get: warning: modules disabled by GO111MODULE=auto in GOPATH/src" 相关告警；


## 试用

- 帮助

```
[#32#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh
NAME:
  ./gen_github_release.sh - Builds and releases a go binary to github releases
USAGE:
  ./gen_github_release.sh [options]
OPTIONS:
  -h, --help                Shows this screen
  -u, --user, --owner       Repository owner (e.g.: caarlos0)
  -r, --repo                Repository name (e.g.: go-releaser)
  -b, --binary              Binary final name (e.g.: antibody)
  -m, --main                Path to the main golang file (e.g.: cmd/main.go)
  -e, --extra               Extra files to package: (e.g.: USAGE.md)
[#33#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

- 失败一：必须有 README

```
[#34#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh -u moooofly -r playground/gorelease_example -b gorelease_example -m ./main.go
Building v0.2.0...
  -> Building for linux on amd64...
  -> Building for linux on 386...
  -> Building for darwin on amd64...
  -> Building for darwin on 386...
Packaging v0.2.0...
cp: cannot stat 'README*': No such file or directory
[#35#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

- 失败二：必须有 LICENSE

```
[#35#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$touch README
[#36#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
[#36#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh -u moooofly -r playground/gorelease_example -b gorelease_example -m ./main.go
Building v0.2.0...
  -> Building for linux on amd64...
  -> Building for linux on 386...
  -> Building for darwin on amd64...
  -> Building for darwin on 386...
Packaging v0.2.0...
cp: cannot stat 'LICENSE*': No such file or directory
[#37#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

- 失败三：必须设置 GITHUB_TOKEN

```
[#38#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh -u moooofly -r playground/gorelease_example -b gorelease_example -m ./main.go
Building v0.2.0...
  -> Building for linux on amd64...
  -> Building for linux on 386...
  -> Building for darwin on amd64...
  -> Building for darwin on 386...
Packaging v0.2.0...
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
Releasing v0.2.0...
go get: warning: modules disabled by GO111MODULE=auto in GOPATH/src;
	ignoring go.mod;
	see 'go help modules'
github.com/aktau/github-release (download)
github.com/tomnomnom/linkheader (download)
github.com/dustin/go-humanize (download)
github.com/voxelbrain/goptions (download)
Creating release v0.2.0...
error: empty token
error: empty token
[#39#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

上面出现两次 "error: empty token" 的原因是在于 `github-release release` 失败一次，`github-release edit` 又失败一次

```
  github-release release \
    --user "$OWNER" \
    --repo "$REPO" \
    --tag "$CURRENT" \
    --description "$description" ||
      github-release edit \
        --user "$OWNER" \
        --repo "$REPO" \
        --tag "$CURRENT" \
        --description "$description"
```

- 失败四：貌似由于项目层级过多导致（其实是默认认为 github.com/moooofly/playground 为一个 repo ）

```
[#46#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$export GITHUB_TOKEN=6b1c019a5708a0b55dff66260742a135f05e96fc

[#48#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh -u moooofly -r playground/gorelease_example -b gorelease_example -m ./main.go
Building v0.2.0...
  -> Building for linux on amd64...
  -> Building for linux on 386...
  -> Building for darwin on amd64...
  -> Building for darwin on 386...
Packaging v0.2.0...
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
Releasing v0.2.0...
go get: warning: modules disabled by GO111MODULE=auto in GOPATH/src;
	ignoring go.mod;
	see 'go help modules'
github.com/aktau/github-release (download)
github.com/tomnomnom/linkheader (download)
github.com/dustin/go-humanize (download)
github.com/voxelbrain/goptions (download)
Creating release v0.2.0...
error: github returned 404 Not Found
error: expected '200 OK' but received '404 Not Found' (url: https://api.github.com/repos/moooofly/playground/gorelease_example/releases?access_token=6b1c019a5708a0b55dff66260742a135f05e96fc&per_page=100)
[#49#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

对比一个[成功获取 harbor-go-client release 的 case](https://api.github.com/repos/moooofly/harbor-go-client/releases?access_token=6b1c019a5708a0b55dff66260742a135f05e96fc&per_page=100) ；

- 成功

```
[#51#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$./gen_github_release.sh -u moooofly -r playground -b gorelease_example -m ./main.go
Building v0.2.0...
  -> Building for linux on amd64...
  -> Building for linux on 386...
  -> Building for darwin on amd64...
  -> Building for darwin on 386...
Packaging v0.2.0...
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
./
./gorelease_example
./README
./LICENSE
Releasing v0.2.0...
go get: warning: modules disabled by GO111MODULE=auto in GOPATH/src;
	ignoring go.mod;
	see 'go help modules'
github.com/aktau/github-release (download)
github.com/tomnomnom/linkheader (download)
github.com/dustin/go-humanize (download)
github.com/voxelbrain/goptions (download)
Creating release v0.2.0...
error: github returned 422 Unprocessable Entity (this is probably because the release already exists)
--> Uploading ./dist/gorelease_example_Darwin_i386.tar.gz...
--> Uploading ./dist/gorelease_example_Darwin_x86_64.tar.gz...
--> Uploading ./dist/gorelease_example_Linux_i386.tar.gz...
--> Uploading ./dist/gorelease_example_Linux_x86_64.tar.gz...
[#52#root@ubuntu-1604 /go/src/github.com/moooofly/playground/gorelease_example]$
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/%E5%9F%BA%E4%BA%8Egoreleaser%E5%92%8Cgithub-release%E4%B8%A4%E7%A7%8D%E5%8F%91%E5%B8%83%E7%9A%84%E5%AF%B9%E6%AF%94.png)


