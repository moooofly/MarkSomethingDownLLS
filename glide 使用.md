# glide 使用


- Glide 是一种用于管理 Go 项目 dependencies 的命令行实用工具；
- 依赖管理的起点就是基于位于项目 root 目录下的 `glide.yaml` 开始的；
- Glide 将 dependencies 保存在 `vendor` 目录中；Glide 正是用于管理 Go package 内 vendor 的目录工具；
- Glide 通过扫描目标应用的源码或库文件来确定必要的 dependencies ；为了确定具体的 versions 和 locations （例如 aliases for forks），Glide 会读取按照一定规则编写的 `glide.yaml` 文件内容，并通过该信息获取相应的 dependencies ；
- 获取的 dependencies 被导出到 `vendor/` 目录下，之后 go tools 就能够找到并进行使用；与此同时，还会生成一个名为 `glide.lock` 的文件，其中包含了全部 dependencies 相关的信息，包括那些 **transitive ones** ，即依赖的依赖所产生的内容；
- Go 社区现在正在开发名为 [`dep`](https://github.com/golang/dep) 的项目用于进行依赖管理；**Please consider trying to migrate from Glide to dep**.

----------

## glide 使用常规流程

> 以下内容参考：https://glide.sh/

### Get Glide

> Get the latest release of Glide. The script puts it with your Go binaries (`$GOPATH/bin` or `$GOBIN`).

```
curl https://glide.sh/get | sh
```

### Initialization -- `glide init`

> **Scan** a codebase and **create** a `glide.yaml` file containing the dependencies.

`glide init` 完成：

- 扫描源码确定 dependencies
- 生成 `glide.yaml` 文件

```
[#1309#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$glide init
[INFO]	Generating a YAML configuration file and guessing the dependencies
[INFO]	Attempting to import from other package managers (use --skip-import to skip)
[INFO]	Scanning code to look for dependencies
[INFO]	--> Found reference to github.com/jessevdk/go-flags
[INFO]	--> Found reference to github.com/parnurzeal/gorequest
[INFO]	--> Found reference to golang.org/x/sys/unix
[INFO]	--> Found reference to gopkg.in/yaml.v2
[INFO]	Writing configuration file (glide.yaml)
[INFO]	Would you like Glide to help you find ways to improve your glide.yaml configuration?
[INFO]	If you want to revisit this step you can use the config-wizard command at any time.
[INFO]	Yes (Y) or No (N)?
y
[INFO]	Looking for dependencies to make suggestions on
[INFO]	--> Scanning for dependencies not using version ranges
[INFO]	--> Scanning for dependencies using commit ids
[INFO]	Gathering information on each dependency
[INFO]	--> This may take a moment. Especially on a codebase with many dependencies
[INFO]	--> Gathering release information for dependencies
[INFO]	--> Looking for dependency imports where versions are commit ids
[INFO]	Here are some suggestions...
[INFO]	The package github.com/jessevdk/go-flags appears to have Semantic Version releases (http://semver.org).
[INFO]	The latest release is v1.3.0. You are currently not using a release. Would you like
[INFO]	to use this release? Yes (Y) or No (N)
y
[INFO]	Would you like to remember the previous decision and apply it to future
[INFO]	dependencies? Yes (Y) or No (N)
y
[INFO]	Updating github.com/jessevdk/go-flags to use the release v1.3.0 instead of no release
[INFO]	The package github.com/jessevdk/go-flags appears to use semantic versions (http://semver.org).
[INFO]	Would you like to track the latest minor or patch releases (major.minor.patch)?
[INFO]	The choices are:
[INFO]	 - Tracking minor version releases would use '>= 1.3.0, < 2.0.0' ('^1.3.0')
[INFO]	 - Tracking patch version releases would use '>= 1.3.0, < 1.4.0' ('~1.3.0')
[INFO]	 - Skip using ranges
[INFO]	For more information on Glide versions and ranges see https://glide.sh/docs/versions
[INFO]	Minor (M), Patch (P), or Skip Ranges (S)?
M
[INFO]	Would you like to remember the previous decision and apply it to future
[INFO]	dependencies? Yes (Y) or No (N)
y
[INFO]	Updating github.com/jessevdk/go-flags to use the range ^1.3.0 instead of commit id v1.3.0
[INFO]	Updating github.com/parnurzeal/gorequest to use the release v0.2.15 instead of no release
[INFO]	Updating github.com/parnurzeal/gorequest to use the range ^0.2.15 instead of commit id v0.2.15
[INFO]	Configuration changes have been made. Would you like to write these
[INFO]	changes to your configuration file? Yes (Y) or No (N)
y
[INFO]	Writing updates to configuration file (glide.yaml)
[INFO]	You can now edit the glide.yaml file.:
[INFO]	--> For more information on versions and ranges see https://glide.sh/docs/versions/
[INFO]	--> For details on additional metadata see https://glide.sh/docs/glide.yaml/
[#1310#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$
[#1310#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$ll
total 7976
drwxr-xr-x 6 root root    4096 Nov 13 10:30 ./
drwxr-xr-x 3 root root    4096 Nov  3 01:11 ../
drwxr-xr-x 2 root root    4096 Nov 12 19:36 api/
-rw-r--r-- 1 root root     871 Nov 12 19:36 config.yaml
drwxr-xr-x 2 root root    4096 Nov  3 15:07 docs/
drwxr-xr-x 9 root root    4096 Nov 13 10:26 .git/
-rw-r--r-- 1 root root      30 Nov 12 19:36 .gitignore
-rw-r--r-- 1 root root     205 Nov 13 10:36 glide.yaml
-rwxr-xr-x 1 root root 8102697 Nov 12 22:28 harbor-go-client*
-rw-r--r-- 1 root root    1065 Nov  3 14:23 LICENSE
-rw-r--r-- 1 root root     376 Nov 12 19:36 main.go
-rw-r--r-- 1 root root    2388 Nov 12 19:36 README.md
-rw-r--r-- 1 root root     761 Nov 12 19:36 rp.yaml
-rwxr-xr-x 1 root root    5877 Nov  3 01:11 test.sh*
drwxr-xr-x 2 root root    4096 Nov 12 19:36 utils/
[#1311#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$git status
On branch develop
Untracked files:
  (use "git add <file>..." to include in what will be committed)

	glide.yaml

nothing added to commit but untracked files present (use "git add" to track)
[#1312#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$cat glide.yaml
package: git.llsapp.com/fei.sun/harbor-go-client
import:
- package: github.com/jessevdk/go-flags
  version: ^1.3.0
- package: github.com/parnurzeal/gorequest
  version: ^0.2.15
- package: golang.org/x/sys
  subpackages:
  - unix
- package: gopkg.in/yaml.v2
[#1313#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$
```

### Additional Configuration -- `edit glide.yaml`

> Optionally, **edit** the `glide.yaml` file to **add** **versions** and other information.

- 建议手动按需调整 `glide.yaml` 文件内容（例如增加 license/owners 等信息）

调整后变为

```
package: git.llsapp.com/fei.sun/harbor-go-client
license: MIT
owners:
- name: moooofly
  email: centos.sf@gmail.com
  homepage: https://github.com/moooofly/
import:
- package: github.com/jessevdk/go-flags
  version: ^1.3.0
- package: github.com/parnurzeal/gorequest
  version: ^0.2.15
- package: golang.org/x/sys
  subpackages:
  - unix
- package: gopkg.in/yaml.v2
```

### Resolve The Dependency Tree -- `glide update`

> **Install** the latest dependencies into the `vendor` directory matching the version resolution information. The complete dependency tree is installed, **importing** Glide, Godep, GB, and GPM configuration along the way. A lock file (`glide.lock`) is **created** from the final output.

`glide update` 完成：

- 下载 dependencies
- 解析 imports 信息（主要针对 Glide, Godep, GB 和 GPM 等配置信息）
- 导出 resolved dependencies
- 替换 vendor 下已存在的 dependencies
- 创建或更新 `glide.lock` 文件

```
[#1337#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$glide update
[INFO]	Downloading dependencies. Please wait...
[INFO]	--> Fetching updates for github.com/jessevdk/go-flags
[INFO]	--> Fetching updates for golang.org/x/sys
[INFO]	--> Fetching updates for gopkg.in/yaml.v2
[INFO]	--> Fetching updates for github.com/parnurzeal/gorequest
[INFO]	--> Detected semantic version. Setting version for github.com/jessevdk/go-flags to v1.3.0
[INFO]	--> Detected semantic version. Setting version for github.com/parnurzeal/gorequest to v0.2.15
[INFO]	Resolving imports
[INFO]	--> Fetching updates for github.com/moul/http2curl
[INFO]	--> Fetching updates for github.com/pkg/errors
[INFO]	--> Fetching updates for golang.org/x/net
[INFO]	Found Godeps.json file in /root/.glide/cache/src/https-github.com-moul-http2curl
[INFO]	--> Parsing Godeps metadata...
[INFO]	Downloading dependencies. Please wait...
[INFO]	Setting references for remaining imports
[INFO]	Exporting resolved dependencies...
[INFO]	--> Exporting github.com/jessevdk/go-flags
[INFO]	--> Exporting github.com/parnurzeal/gorequest
[INFO]	--> Exporting github.com/pkg/errors
[INFO]	--> Exporting github.com/moul/http2curl
[INFO]	--> Exporting gopkg.in/yaml.v2
[INFO]	--> Exporting golang.org/x/net
[INFO]	--> Exporting golang.org/x/sys
[INFO]	Replacing existing vendor dependencies
([INFO]    Versions did not change. Skipping glide.lock update.)  -- 若无版本变化，则会出现该行信息
[INFO]	Project relies on 7 dependencies.
[#1338#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$
```

生成的 `glide.lock` 文件

```
[#1343#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$cat glide.lock
hash: f21e1a5857975664feac5d8596e293375c8383b80cbb5777b420fc357c873483
updated: 2017-12-05T11:30:55.956865266+08:00
imports:
- name: github.com/jessevdk/go-flags
  version: 96dc06278ce32a0e9d957d590bb987c81ee66407
- name: github.com/moul/http2curl
  version: 9ac6cf4d929b2fa8fd2d2e6dec5bb0feb4f4911d
- name: github.com/parnurzeal/gorequest
  version: a578a48e8d6ca8b01a3b18314c43c6716bb5f5a3
- name: github.com/pkg/errors
  version: f15c970de5b76fac0b59abb32d62c17cc7bed265
- name: golang.org/x/net
  version: a8b9294777976932365dabb6640cf1468d95c70f
  subpackages:
  - publicsuffix
- name: golang.org/x/sys
  version: 8b4580aae2a0dd0c231a45d3ccb8434ff533b840
  subpackages:
  - unix
- name: gopkg.in/yaml.v2
  version: 287cf08546ab5e7e37d55a84f7ed3fd1db036de5
testImports: []
[#1344#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$
```

### Reproducible Installations -- `glide install`

> **Install** the dependencies and revisions listed in the lock file (`glide.lock`) into the `vendor` directory. If no lock file exists an update is run.

`glide install` 完成（同 `glide update` 命令，但不进行 scanning）：

- Downloading dependencies
- Setting references
- Exporting resolved dependencies
- Replacing existing vendor dependencies

```
[#1344#root@ubuntu-1604 /go/src/git.llsapp.com/fei.sun/harbor-go-client]$glide install
[INFO]	Downloading dependencies. Please wait...
[INFO]	--> Found desired version locally github.com/jessevdk/go-flags 96dc06278ce32a0e9d957d590bb987c81ee66407!
[INFO]	--> Found desired version locally github.com/moul/http2curl 9ac6cf4d929b2fa8fd2d2e6dec5bb0feb4f4911d!
[INFO]	--> Found desired version locally github.com/parnurzeal/gorequest a578a48e8d6ca8b01a3b18314c43c6716bb5f5a3!
[INFO]	--> Found desired version locally github.com/pkg/errors f15c970de5b76fac0b59abb32d62c17cc7bed265!
[INFO]	--> Found desired version locally golang.org/x/net a8b9294777976932365dabb6640cf1468d95c70f!
[INFO]	--> Found desired version locally golang.org/x/sys 8b4580aae2a0dd0c231a45d3ccb8434ff533b840!
[INFO]	--> Found desired version locally gopkg.in/yaml.v2 287cf08546ab5e7e37d55a84f7ed3fd1db036de5!
[INFO]	Setting references.
[INFO]	--> Setting version for github.com/jessevdk/go-flags to 96dc06278ce32a0e9d957d590bb987c81ee66407.
[INFO]	--> Setting version for github.com/parnurzeal/gorequest to a578a48e8d6ca8b01a3b18314c43c6716bb5f5a3.
[INFO]	--> Setting version for github.com/pkg/errors to f15c970de5b76fac0b59abb32d62c17cc7bed265.
[INFO]	--> Setting version for github.com/moul/http2curl to 9ac6cf4d929b2fa8fd2d2e6dec5bb0feb4f4911d.
[INFO]	--> Setting version for golang.org/x/net to a8b9294777976932365dabb6640cf1468d95c70f.
[INFO]	--> Setting version for gopkg.in/yaml.v2 to 287cf08546ab5e7e37d55a84f7ed3fd1db036de5.
[INFO]	--> Setting version for golang.org/x/sys to 8b4580aae2a0dd0c231a45d3ccb8434ff533b840.
[INFO]	Exporting resolved dependencies...
[INFO]	--> Exporting github.com/jessevdk/go-flags
[INFO]	--> Exporting github.com/parnurzeal/gorequest
[INFO]	--> Exporting github.com/moul/http2curl
[INFO]	--> Exporting github.com/pkg/errors
[INFO]	--> Exporting golang.org/x/net
[INFO]	--> Exporting gopkg.in/yaml.v2
[INFO]	--> Exporting golang.org/x/sys
[INFO]	Replacing existing vendor dependencies
```

### Add More Dependencies

```
glide get github.com/foo/bar
or
glide get github.com/foo/bar#^1.2.3`
```

> **Add** a new dependency to the `glide.yaml`, **install** the dependency, and **re-resolve** the dependency tree. Optionally, put a version after an anchor.


----------


## glide 使用简化流程

```
$ glide create/init              # Start a new workspace
$ open glide.yaml                # and edit away!
$ glide get github.com/xxx/xxx   # Get a package and add to glide.yaml
$ glide install                  # Install packages and dependencies
# work, work, work
$ go build                       # Go tools work normally
$ glide up                       # Update to newest versions of the package
```

- `glide create/init` - can be use to setup a new project (workspace), creates a `glide.yaml` file while attempting to guess the packages and versions to put in it.
- `glide update/up` - regenerates the dependency versions using scanning and rules. Download or update all of the libraries listed in the `glide.yaml` file and put them in the `vendor` directory. It will also recursively walk through the dependency packages to fetch anything that's needed and read in any configuration. A `glide.lock` file will be created or updated with the dependencies pinned to specific versions. For example, if in the `glide.yaml` file a version was specified as a range (e.g., ^1.2.3) it will be set to a specific commit id in the `glide.lock` file. 
- `glide install` - install the versions listed in the `glide.lock` file, skipping scanning (That allows for reproducible installs), unless the `glide.lock` file is not found in which case it will perform an update. When you want to install the specific versions from the `glide.lock` file use `glide install`. This will read the `glide.lock` file and install the commit id specific versions there. When the `glide.lock` file doesn't tie to the `glide.yaml` file, such as there being a change, it will provide a warning. Running glide up will recreate the `glide.lock` file when updating the dependency tree. If no `glide.lock` file is present `glide install` will perform an update and generate a lock file.
- `glide config-wizard` - runs a wizard that scans your dependencies and retrieves information on them to offer up suggestions that you can interactively choose.
- `glide get [package name]` - You can download one or more packages to your `vendor` directory and have it added to your `glide.yaml` file with glide get.
- `glide novendor/nv` - When you run commands like `go test ./...` it will iterate over all the subdirectories including the `vendor` directory. When you are testing your application you may want to test your application files without running all the tests of your dependencies and their dependencies. This is where the `novendor` command comes in. It lists all of the directories except `vendor`. `go test $(glide novendor)` will run `go test` over all directories of your project except the `vendor` directory.
- `glide name` -  returns the name of the package listed in the `glide.yaml` file.
- `glide tree` -  inspect code and give you details about what is imported. This command is deprecated and will be removed in the near future.
- `glide list` - Glide's list command shows an alphabetized list of all the packages that a project imports.


## example-glide.yaml

Ref: https://github.com/Masterminds/glide/blob/master/docs/example-glide.yaml

```
# The name of this package.
# 被 vendor 的目标 package
package: github.com/Masterminds/glide

# External dependencies.
import:
  # Minimal definition
  # 最简定义
  # This will use "go get [-u]" to fetch and update the package, and it will
  # attempt to keep the release at the tip of master. It does this by looking
  # for telltale signs that this is a git, bzr, or hg repo, and then acting
  # accordingly.
  # 这种定义形式将使用 "go get [-u]" 命令来 fetch 和 update 包；
  - package: github.com/kylelemons/go-gypsy

  # Full definition
  # 完整定义
  # This will check out the given Git repo, set the version to master,
  # use "git" (not "go get") to manage it, and alias the package to the
  # import path github.com/Masterminds/cookoo
  # 指定外部引用该 package 时的 import path
  - package: github.com/Masterminds/cookoo
    # 指定使用 git 命令，非 go get 命令
    vcs: git
    # 指定要 checkout 的版本（可以 branch/tag/hash 等）
    version: master
    # 指定 Git repo 的实际位置
    repo: git@github.com:Masterminds/cookoo.git

  # Here's an example with a commit hash for a version. Since repo is not
  # specified, this will use git to to try to clone
  # 'http://github.com/aokoli/goutils' and then set the revision to the given
  # hash.
  - package: github.com/aokoli/goutils
    vcs: git
    # 通过 commit hash 指定版本
    version: 9c37978a95bd5c709a15883b6242714ea6709e64

  # MASKING: This takes my fork of goamz (technosophos/goamz) and clones it
  # as if it were the crowdmob/goamz package. This is incredibly useful for
  # masking packages and/or working with forks or clones.
  #
  # Note that absolutely no namespace munging happens on the code. If you want
  # that, you'll have to do it on your own. The intent of this masking was to
  # make it so you don't have to vendor imports.
  # 指定外部引用该 package 时的 import path
  # 主要用途：
  # 1. 对 package 进行 mask ，方便配合 fork 或 clone 使用
  # 2. 可以针对某些 import 的 package 不做 vendor
  - package: github.com/crowdmob/goamz
    vcs: git
    # 指定目标 Git repo 的实际位置，用于 mask 实际的 package 内容
    repo: git@github.com:technosophos/goamz.git

  - package: bzr.example.com/foo/bar/trunk
    vcs: bzr
    repo: bzr://bzr.example.com/foo/bar/trunk
    # The version can be a branch, tag, commit id, or a semantic version
    # constraint parsable by https://github.com/Masterminds/semver
    # 通过 tag 指定版本
    version: 1.0.0

  - package: hg.example.com/foo/bar
    vcs: hg
    repo: http://hg.example.com/foo/bar
    version: ae081cd1d6cc

  # For SVN, the only valid version is a commit number. Tags and branches go in
  # the repo URL.
  - package: svn.example.com/foo/bar/trunk
    vcs: svn
    repo: http://svn.example.com/foo/bar/trunk


  # If a package is dependent on OS, you can tell Glide to only
  # fetch for certain OS or architectures.
  #
  # os can be any valid GOOS.
  # arch can be any valid GOARCH.
  - package: github.com/unixy/package
    os:
      - linux
      - darwin
    arch:
      - amd64
```

glide.yaml 的主要作用：

- It names the current package
- It declares external dependencies

## 坑

### subpackage 的 parent 不是一个 go package 

问题：使用 glide 做 vendor 时，如果 subpackage 的 parent 不是一个 go package 时，glide update/install 无法正常工作

具体：

- 目录结构如下

在 GitLab 上存在 group 的概念，这里 ops 是 base group ，hunter 是 sub-group ；

```
[#169#root@ubuntu-1604 /go/src/git.llsapp.com]$tree ops -L 2
ops
└── hunter
    ├── hunter-agent
    ├── hunter-demo-golang
    ├── ocgrpc-wrapper
    └── opencensus-go-exporter-hunter

5 directories, 0 files
[#170#root@ubuntu-1604 /go/src/git.llsapp.com]$
```

- glide 初始化后得到 glide.yaml

```
[#157#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide init
[#158#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
import:
- package: git.llsapp.com/ops/hunter      # 产生问题的地方
  subpackages:
  - ocgrpc-wrapper
  - opencensus-go-exporter-hunter
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#159#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 获取依赖出错

出错的原因：应该是将 git.llsapp.com/ops/hunter 作为 <project>/<repository> 进行获取时，发现其并不存在；可以推断，glide 在多级目录的处理上存在一些问题（该问题主要出现在 GitLab 上的原因就在于 GitLab 上存在 group 的概念，进而导致多层级目录）；

issue related: [glide/issues#1006](https://github.com/Masterminds/glide/issues/1006), [glide/issues#946](https://github.com/Masterminds/glide/issues/946), [glide/issues#887](https://github.com/Masterminds/glide/issues/887)

```
[#174#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[INFO]  Downloading dependencies. Please wait...
[INFO]  --> Fetching git.llsapp.com/ops/hunter
[INFO]  --> Fetching updates for github.com/golang/protobuf
[INFO]  --> Fetching updates for go.opencensus.io
[INFO]  --> Fetching updates for google.golang.org/grpc
[INFO]  --> Fetching updates for golang.org/x/net
Username for 'https://git.llsapp.com': fei.sun
Password for 'https://fei.sun@git.llsapp.com':
[WARN]  Unable to checkout git.llsapp.com/ops/hunter
[ERROR] Update failed for git.llsapp.com/ops/hunter: Unable to get repository: Cloning into '/root/.glide/cache/src/https-git.llsapp.com-ops-hunter'...
remote: The project you were looking for could not be found.
fatal: repository 'https://git.llsapp.com/ops/hunter.git/' not found
: exit status 128
[ERROR] Failed to do initial checkout of config: Unable to get repository: Cloning into '/root/.glide/cache/src/https-git.llsapp.com-ops-hunter'...
remote: The project you were looking for could not be found.
fatal: repository 'https://git.llsapp.com/ops/hunter.git/' not found
: exit status 128
[#175#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 第一次调整 glide.yaml 的内容，错误依旧

主要调整内容为在 package 中直接指定完整 path 

```
[#185#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
import:
- package: git.llsapp.com/ops/hunter/ocgrpc-wrapper                 # 调整
- package: git.llsapp.com/ops/hunter/opencensus-go-exporter-hunter  # 调整
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#186#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$

[#186#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[INFO]  Downloading dependencies. Please wait...
[INFO]  --> Fetching git.llsapp.com/ops/hunter
[INFO]  --> Fetching updates for github.com/golang/protobuf
[INFO]  --> Fetching updates for go.opencensus.io
[INFO]  --> Fetching updates for golang.org/x/net
[INFO]  --> Fetching updates for google.golang.org/grpc
Username for 'https://git.llsapp.com': fei.sun
Password for 'https://fei.sun@git.llsapp.com':
[WARN]  Unable to checkout git.llsapp.com/ops/hunter
[ERROR] Update failed for git.llsapp.com/ops/hunter: Unable to get repository: Cloning into '/root/.glide/cache/src/https-git.llsapp.com-ops-hunter'...
remote: The project you were looking for could not be found.
fatal: repository 'https://git.llsapp.com/ops/hunter.git/' not found
: exit status 128
[ERROR] Failed to do initial checkout of config: Unable to get repository: Cloning into '/root/.glide/cache/src/https-git.llsapp.com-ops-hunter'...
remote: The project you were looking for could not be found.
fatal: repository 'https://git.llsapp.com/ops/hunter.git/' not found
: exit status 128
[#187#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 第二次调整 glide.yaml 的内容，错误依旧

主要调整内容为在 package 中直接指定完整 path ，并指定 repo

```
[#193#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
import:
- package: git.llsapp.com/ops/hunter/ocgrpc-wrapper                 # 调整
  repo: git@git.llsapp.com:ops/hunter/ocgrpc-wrapper.git
- package: git.llsapp.com/ops/hunter/opencensus-go-exporter-hunter  # 调整
  repo: git@git.llsapp.com:ops/hunter/opencensus-go-exporter-hunter.git
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#194#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$


[#194#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[ERROR] Failed to parse /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang/glide.yaml: Import git.llsapp.com/ops/hunter repeated with different Repository details
[#195#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 第三次调整 glide.yaml 的内容，错误依旧

主要调整内容为在 package 中直接指定完整 path ，但去掉了 ops 这段，并指定 repo

```
[#196#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
import:
- package: git.llsapp.com/hunter/ocgrpc-wrapper
  repo: git@git.llsapp.com:ops/hunter/ocgrpc-wrapper.git
- package: git.llsapp.com/hunter/opencensus-go-exporter-hunter
  repo: git@git.llsapp.com:ops/hunter/opencensus-go-exporter-hunter.git
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#197#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$


[#197#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[INFO]  Downloading dependencies. Please wait...
[INFO]  --> Fetching updates for git.llsapp.com/hunter/ocgrpc-wrapper
[INFO]  --> Fetching updates for git.llsapp.com/hunter/opencensus-go-exporter-hunter
[INFO]  --> Fetching updates for github.com/golang/protobuf
[INFO]  --> Fetching updates for go.opencensus.io
[INFO]  --> Fetching updates for golang.org/x/net
[INFO]  --> Fetching updates for google.golang.org/grpc
[INFO]  --> Detected semantic version. Setting version for github.com/golang/protobuf to v1.2.0
[INFO]  --> Detected semantic version. Setting version for go.opencensus.io to v0.17.0
[INFO]  --> Detected semantic version. Setting version for google.golang.org/grpc to v1.15.0
[INFO]  Resolving imports
[INFO]  --> Fetching git.llsapp.com/ops/hunter
Username for 'https://git.llsapp.com': fei.sun
Password for 'https://fei.sun@git.llsapp.com':
[WARN]  Unable to checkout git.llsapp.com/ops/hunter
[ERROR] Error looking for git.llsapp.com/ops/hunter/ocgrpc-wrapper: Unable to get repository: Cloning into '/root/.glide/cache/src/https-git.llsapp.com-ops-hunter'...
remote: The project you were looking for could not be found.
fatal: repository 'https://git.llsapp.com/ops/hunter.git/' not found
: exit status 128
[WARN]  Unable to set version on git.llsapp.com/ops/hunter to . Err: Unable to retrieve checked out version: chdir /root/.glide/cache/src/https-git.llsapp.com-ops-hunter: no such file or directory
[ERROR] Error scanning git.llsapp.com/ops/hunter/opencensus-go-exporter-hunter: cannot find package "." in:
  /root/.glide/cache/src/https-git.llsapp.com-ops-hunter/opencensus-go-exporter-hunter
[INFO]  --> Fetching updates for google.golang.org/genproto
[INFO]  --> Fetching updates for golang.org/x/sys
[INFO]  --> Fetching updates for golang.org/x/text
[ERROR] Failed to retrieve a list of dependencies: Error resolving imports
[#198#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 第四次调整 glide.yaml 的内容，成功！！！

主要调整内容为将 import path 中的 ops 去掉

```
[#213#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$grep -irn "/hunter/" ./*
./example/grpc_example/helloworld_client/main.go:9: wrapper "git.llsapp.com/hunter/ocgrpc-wrapper"
./example/grpc_example/helloworld_client/main.go:10:  agent "git.llsapp.com/hunter/opencensus-go-exporter-hunter"
./example/grpc_example/helloworld_server/main.go:14:  wrapper "git.llsapp.com/hunter/ocgrpc-wrapper"
./example/grpc_example/helloworld_server/main.go:15:  agent "git.llsapp.com/hunter/opencensus-go-exporter-hunter"
./example/local_example/main.go:10: agent "git.llsapp.com/hunter/opencensus-go-exporter-hunter"
./example/callchain_example/callchain_server/main.go:16:  wrapper "git.llsapp.com/hunter/ocgrpc-wrapper"
./example/callchain_example/callchain_server/main.go:17:  agent "git.llsapp.com/hunter/opencensus-go-exporter-hunter"
./example/callchain_example/callchain_client/main.go:11:  wrapper "git.llsapp.com/hunter/ocgrpc-wrapper"
./example/callchain_example/callchain_client/main.go:12:  agent "git.llsapp.com/hunter/opencensus-go-exporter-hunter"
[#214#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

其他内容没有变化

```
[#217#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
import:
- package: git.llsapp.com/hunter/ocgrpc-wrapper
  repo: git@git.llsapp.com:ops/hunter/ocgrpc-wrapper.git
- package: git.llsapp.com/hunter/opencensus-go-exporter-hunter
  repo: git@git.llsapp.com:ops/hunter/opencensus-go-exporter-hunter.git
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#218#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
[#218#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[INFO]  Downloading dependencies. Please wait...
[INFO]  --> Fetching updates for git.llsapp.com/hunter/ocgrpc-wrapper
[INFO]  --> Fetching updates for git.llsapp.com/hunter/opencensus-go-exporter-hunter
[INFO]  --> Fetching updates for github.com/golang/protobuf
[INFO]  --> Fetching updates for go.opencensus.io
[INFO]  --> Fetching updates for golang.org/x/net
[INFO]  --> Fetching updates for google.golang.org/grpc
[INFO]  --> Detected semantic version. Setting version for github.com/golang/protobuf to v1.2.0
[INFO]  --> Detected semantic version. Setting version for go.opencensus.io to v0.17.0
[INFO]  --> Detected semantic version. Setting version for google.golang.org/grpc to v1.15.0
[INFO]  Resolving imports
[INFO]  --> Fetching updates for github.com/census-instrumentation/opencensus-proto
[INFO]  --> Setting version for github.com/census-instrumentation/opencensus-proto to a7586552457eea9fa020ac37fab5f41d11e9270c.
[INFO]  --> Fetching updates for google.golang.org/api
[INFO]  --> Fetching updates for golang.org/x/sync
[INFO]  --> Fetching updates for google.golang.org/genproto
[INFO]  --> Fetching updates for golang.org/x/sys
[INFO]  --> Fetching updates for golang.org/x/text
[INFO]  Downloading dependencies. Please wait...
[INFO]  Setting references for remaining imports
[INFO]  Exporting resolved dependencies...
[INFO]  --> Exporting git.llsapp.com/hunter/ocgrpc-wrapper
[INFO]  --> Exporting git.llsapp.com/hunter/opencensus-go-exporter-hunter
[INFO]  --> Exporting github.com/golang/protobuf
[INFO]  --> Exporting github.com/census-instrumentation/opencensus-proto
[INFO]  --> Exporting go.opencensus.io
[INFO]  --> Exporting google.golang.org/grpc
[INFO]  --> Exporting golang.org/x/text
[INFO]  --> Exporting golang.org/x/sys
[INFO]  --> Exporting google.golang.org/genproto
[INFO]  --> Exporting google.golang.org/api
[INFO]  --> Exporting golang.org/x/net
[INFO]  --> Exporting golang.org/x/sync
[INFO]  Replacing existing vendor dependencies
[INFO]  Project relies on 12 dependencies.
[#219#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
[#219#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
[#219#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$ll vendor/
total 28
drwxr-xr-x  7 root root 4096 Sep 26 14:51 ./
drwxr-xr-x  5 root root 4096 Sep 26 14:51 ../
drwxr-xr-x  4 root root 4096 Sep 26 14:51 github.com/
drwxr-xr-x  3 root root 4096 Sep 26 14:51 git.llsapp.com/
drwxr-xr-x  3 root root 4096 Sep 26 14:51 golang.org/
drwxr-xr-x  5 root root 4096 Sep 26 14:51 google.golang.org/
drwxr-xr-x 13 root root 4096 Sep 26 14:51 go.opencensus.io/
[#220#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

- 第五次调整 glide.yaml 的内容，补充测试，失败；

失败原因在于：无论 git.llsapp.com/hunter 还是 git.llsapp.com/ops/hunter 都不是真正的 repo ，因此没有 VCS 的东西；因为无法使用 repo 指定包含 VCS 内容的位置；而 subpackages 下又无法使用 repo 这个选项，故这种方式无法解决问题；

```
[#236#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$cat glide.yaml
package: git.llsapp.com/ops/hunter/hunter-demo-golang
license: MIT
owners:
- name: fei.sun
  email: fei.sun@liulishuo.com
  homepage: https://github.com/moooofly/
import:
- package: git.llsapp.com/hunter     # 将这里的 ops 去掉了
  subpackages:
  - ocgrpc-wrapper
  - opencensus-go-exporter-hunter
- package: github.com/golang/protobuf
  version: ^1.2.0
  subpackages:
  - proto
- package: go.opencensus.io
  version: ^0.17.0
  subpackages:
  - plugin/ocgrpc
  - stats/view
  - trace
  - zpages
- package: golang.org/x/net
  subpackages:
  - context
- package: google.golang.org/grpc
  version: ^1.15.0
[#237#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
[#237#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$glide update
[INFO]  Downloading dependencies. Please wait...
[INFO]  --> Fetching git.llsapp.com/hunter
[INFO]  --> Fetching updates for github.com/golang/protobuf
[INFO]  --> Fetching updates for go.opencensus.io
[INFO]  --> Fetching updates for golang.org/x/net
[INFO]  --> Fetching updates for google.golang.org/grpc
[WARN]  Unable to checkout git.llsapp.com/hunter
[ERROR] Update failed for git.llsapp.com/hunter: Cannot detect VCS
[ERROR] Failed to do initial checkout of config: Cannot detect VCS
[#238#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```


## 参考

- [Masterminds/glide](https://github.com/Masterminds/glide)
- [godoc: Masterminds/glide](https://godoc.org/github.com/Masterminds/glide)
- [glide.sh](https://glide.sh/)
- [glide.readthedocs.io](https://glide.readthedocs.io/en/latest/)


