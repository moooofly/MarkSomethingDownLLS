# glide 使用


- Glide 是一种用于管理 Go 项目 dependencies 的命令行实用工具；
- 依赖管理的起点就是基于位于项目 root 目录下的 `glide.yaml` 开始的；
- Glide 将 dependencies 保存在 `vendor` 目录中；Glide 正是用于管理 Go package 内 vendor 的目录工具；
- Glide 通过扫描目标应用的源码或库文件来确定必要的 dependencies ；为了确定具体的 versions 和 locations （例如 aliases for forks），Glide 会读取按照一定规则编写的 `glide.yaml` 文件内容，并通过该信息获取相应的 dependencies ；
- 获取的 dependencies 被导出到 `vendor/` 目录下，之后 go tools 就能够找到并进行使用；与此同时，还会生成一个名为 `glide.lock` 的文件，其中包含了全部 dependencies 相关的信息，包括那些 **transitive ones** ；
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


## 参考

- [Masterminds/glide](https://github.com/Masterminds/glide)
- [godoc: Masterminds/glide](https://godoc.org/github.com/Masterminds/glide)
- [glide.sh](https://glide.sh/)
- [glide.readthedocs.io](https://glide.readthedocs.io/en/latest/)


