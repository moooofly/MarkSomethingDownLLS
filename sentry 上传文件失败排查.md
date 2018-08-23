# sentry 上传文件失败排查

## 问题描述

> https://phab.llsapp.com/T42723

在使用 `sentry-cli` 进行上传大量（其实也就 57 个文件，4MB 左右大小）的 js 文件时会出现类似错误：

```
> Adding source map references
> Analyzing 57 sources
> Uploading source maps for release v1.1
error: http error: [55] Failed sending data to the peer (Send failure: Broken pipe)
```

导致上传失败，任务终止（从错误信息上直观判定，问题可能是由于 Sentry 服务端或 nginx 没能正确处理多文件接收，提前关闭了 socket 连接导致）；


上传命令示例如下：

```
sentry-cli releases files v1.1 upload-sourcemaps ~/static
```

## 问题研究

### [getsentry/sentry-cli](https://github.com/getsentry/sentry-cli) 工具

> This is a Sentry command line client **for some generic tasks**. Right now this is primarily used to **upload debug symbols to Sentry** if you are not using the fastlane tools.

推荐安装方式：

```
curl -sL https://sentry.io/get-cli/ | bash
```

homebrew 安装方式：

```
brew install getsentry/tools/sentry-cli
```

docker 安装方式：

> As of version **1.25.0**, there is an official Docker image that comes with `sentry-cli` preinstalled. If you prefer a specific version, specify it as tag. The latest development version is published under the `edge` tag. In production, we recommend you to use the `latest` tag. To use it, run:

```
docker pull getsentry/sentry-cli
docker run --rm -it -v $(pwd):/work getsentry/sentry-cli sentry-cli --help
```

`sentry-cli` 帮助

```
root@sentry-frontend-prod:~# sentry-cli --help

sentry-cli update to 1.27.1 is available!
run sentry-cli update to update
sentry-cli 1.24.1
USAGE:
    sentry-cli <SUBCOMMAND>

OPTIONS:
        --api-key <API_KEY>          The the given Sentry API key.
        --auth-token <AUTH_TOKEN>    Use the given Sentry auth token.
    -h, --help                       Print this help message.
        --log-level <LOG_LEVEL>      Set the log output verbosity.
                                     [valid levels: TRACE, DEBUG, INFO, WARN, ERROR]
        --url <URL>                  Fully qualified URL to the Sentry server.
                                     [defaults to https://sentry.io/]
    -V, --version                    Print version information.

SUBCOMMANDS:
    bash-hook          Prints out a bash script that does error handling.
    difutil            Locate or analyze debug information files.
    help               Prints this message or the help of the given subcommand(s)
    info               Print information about the Sentry server.
    issues             Manage issues in Sentry.
    login              Authenticate with the Sentry server.
    projects           Manage projects on Sentry.
    react-native       Upload build artifacts for react-native projects.
    releases           Manage releases on Sentry.
    repos              Manage repositories on Sentry.
    send-event         Send a manual event to Sentry.
    uninstall          Uninstall the sentry-cli executable.
    update             Update the sentry-cli executable.
    upload-dsym        Upload debug symbols to a project.
    upload-proguard    Upload ProGuard mapping files to a project.
root@sentry-frontend-prod:~#




root@sentry-frontend-prod:~# sentry-cli releases --help

sentry-cli update to 1.27.1 is available!
run sentry-cli update to update
sentry-cli-releases
Manage releases on Sentry.

USAGE:
    sentry-cli releases <SUBCOMMAND>

OPTIONS:
    -h, --help         Prints help information
    -o, --org <ORG>    The organization slug

SUBCOMMANDS:
    delete             Delete a release.
    deploys            Manage release deployments.
    files              Manage release artifacts.
    finalize           Mark a release as finalized and released.
    help               Prints this message or the help of the given subcommand(s)
    info               Print information about a release.
    list               List the most recent releases.
    new                Create a new release.
    propose-version    Propose a version name for a new release.
    set-commits        Set commits of a release.
root@sentry-frontend-prod:~#




root@sentry-frontend-prod:~# sentry-cli releases files --help

sentry-cli update to 1.27.1 is available!
run sentry-cli update to update
sentry-cli-releases-files
Manage release artifacts.

USAGE:
    sentry-cli releases files <VERSION> <SUBCOMMAND>

OPTIONS:
    -h, --help       Prints help information
    -V, --version    Prints version information

ARGS:
    <VERSION>    The version of the release

SUBCOMMANDS:
    delete               Delete a release file.
    help                 Prints this message or the help of the given subcommand(s)
    list                 List all release files.
    upload               Upload a file for a release.
    upload-sourcemaps    Upload sourcemaps for a release.
root@sentry-frontend-prod:~#




root@sentry-frontend-prod:~# sentry-cli releases files upload-sourcemaps --help

sentry-cli update to 1.27.1 is available!
run sentry-cli update to update
sentry-cli-releases-files-upload-sourcemaps
Upload sourcemaps for a release.

USAGE:
    sentry-cli releases files <VERSION> upload-sourcemaps [OPTIONS] <PATHS>...

OPTIONS:
    -d, --dist <DISTRIBUTION>          Optional distribution identifier for the sourcemaps.
    -x, --ext <EXT>...                 Add a file extension to the list of files to upload.
    -h, --help                         Prints help information
    -i, --ignore <IGNORE>...           Ignores all files and folders matching the given glob
    -I, --ignore-file <IGNORE_FILE>    Ignore all files and folders specified in the given ignore
                                       file, e.g. .gitignore.
        --no-sourcemap-reference       Disable emitting of automatic sourcemap references.
                                       By default the tool will store a 'Sourcemap' header with
                                       minified files so that sourcemaps are located automatically
                                       if the tool can detect a link. If this causes issues it can
                                       be disabled.
        --rewrite                      Enable rewriting of matching sourcemaps so that indexed maps
                                       are flattened and missing sources are inlined if possible.
                                       This fundamentally changes the upload process to be based on
                                       sourcemaps and minified files exclusively and comes in handy
                                       for setups like react-native that generate sourcemaps that
                                       would otherwise not work for sentry.
        --strip-common-prefix          Similar to --strip-prefix but strips the most common prefix
                                       on all sources.
        --strip-prefix <PREFIX>...     Strip the given prefix from all filenames.
                                       Only files that start with the given prefix will be
                                       stripped.This requires --rewrite to be enabled.
    -u, --url-prefix <PREFIX>          The URL prefix to prepend to all filenames.
        --validate                     Enable basic sourcemap validation.
    -V, --version                      Prints version information

ARGS:
    <PATHS>...    The files to upload.
root@sentry-frontend-prod:~#
```


### [sentry-cli 文档](https://docs.sentry.io/hosted/learn/cli/) 梳理

> For certain actions you can use the `sentry-cli` command line executable. It can connect to the Sentry API and manage some data for your projects. It’s primarily used for **managing debug information files** for iOS, Android as well as **release and sourcemaps management** for other platforms.

由于案例中主要和 release + file upload 有关，因此只需要研究如下内容：

#### [Release Management](https://docs.sentry.io/learn/cli/releases/)

> The `sentry-cli` tool can be used for **release management** on Sentry. It allows you to **create**, **edit** and **delete** releases as well as **upload** release artifacts for them.

通过 `sentry-cli` 可以实现的 release 相关操作：

- create
- edit
- delete
- upload

> **Note**
> 
>> Because **releases work on projects** you will need to specify the **organization** and **project** you are working with. For more information about this refer to [Working with Projects](https://docs.sentry.io/learn/cli/configuration/#sentry-cli-working-with-projects).

完成 releases 相关操作的前提需要指定 organization 和 project ；

##### Managing Release Artifacts

> When you are working with JavaScript and other platforms you can **upload `release artifacts` to Sentry** which are then considered during processing. The most common release artifact are **source maps** for which `sentry-cli` has specific support.

- release artifacts 常用于基于 JavaScript 的实现；
- 最常见的 release artifact 是 JavaScript sourcemap ；

> To manage release artfacts the `sentry-cli releases files` command can be used which itself provides various sub commands.

###### Upload Files

> The most common use case is to upload files. For the **generic upload** the `sentry-cli releases files VERSION upload` command can be used. However since most release artifacts are JavaScript sourcemap related we have a **Upload Source Maps** convenience method for that.

这里提及的文件上传是指通用的文件上传，针对使用场景最为广泛的 JavaScript sourcemap 上传，下面提供了专门的命令；

> Files uploaded are typically named with a **full** (eg: `http://example.com/foo.js`) or **truncated** URL (eg: `~/foo.js`).

> **Release artifacts are only considered at time of event processing.** So while it’s possible to modify release artifacts after the fact they will only be considered for future events of that release.

- Release artifacts 仅用于 event 处理过程中；
- 可以对 Release artifacts 进行修改，以便在处理将来其他 events 时生效；

> The first argument to upload is the path to the file, the second is an optional URL we should associate it with. Note that if you want to use an **abbreviated URL** (eg: `~/foo.js`) make sure to use **single** quotes to avoid the expansion by the `shell` to your home folder.

通用文件上传的完整使用命令如下：

```
$ sentry-cli releases files VERSION upload /path/to/file '~/file.js'
```

###### Upload Source Maps

> For sourcemap upload a separate command is provided which assists you in **uploading** and **verifying** source maps:

针对 JavaScript sourcemap 的上传，可以直接使用如下专用命令（提供**上传**+**校验**功能）：

```
$ sentry-cli releases files VERSION upload-sourcemaps /path/to/sourcemaps
```

> This command provides a bunch of options and attempts as much **auto detection** as possible. By default it will **scan the provided path for files and upload them named by their path with a `~/` prefix**. It will also **attempt to figure out references between minified files and source maps based on the filename**. So if you have a file named `foo.min.js` which is a minified JavaScript file and a sourcemap named `foo.min.map` for example, it will send a long a `Sourcemap` header to associate them. This works for files the system can detect a relationship of.

该命令实现了一定程度的智能：

- auto detection
- figure out **references** between **minified files** and **source maps** based on the filename

> The following options exist to change the behavior of the upload command:
>
> (略)

Some example usages:

```
$ sentry-cli releases files 0.1 upload-sourcemaps /path/to/sourcemaps
$ sentry-cli releases files 0.1 upload-sourcemaps /path/to/sourcemaps \
    --url-prefix '~/static/js'
$ sentry-cli releases files 0.1 upload-sourcemaps /path/to/sourcemaps \
    --url-prefix '~/static/js' --rewrite --strip-common-prefix
$ sentry-cli releases files 0.1 upload-sourcemaps /path/to/sourcemaps \
    --ignore-file .sentryignore
```


----------



### [Source Maps](https://docs.sentry.io/clients/javascript/sourcemaps/#) 梳理

> Sentry supports un-minifying JavaScript via Source Maps. This lets you view source code context obtained from stack traces in their original untransformed form, which is particularly useful for debugging minified code (e.g. UglifyJS), or transpiled code from a higher-level language (e.g. TypeScript, ES6).

这里提及了几个知识点：

- JavaScript file (code) 可以被 minify 
- 通过 Source Maps 可以查看 stack traces 中的最原始的 code context

> 科普
>
>> 尽管术语 "**traspiling**" 从上世纪就已经出现了，但是似乎对于它到底是什么意思以及 "transpiling" 和 "compiling" 有什么区别仍然存在很多疑惑。
>>
>> 首先，**transpiling 是一种特殊的 compiling** 。这很有助于让我们了解到我们是在谈论差不多类似的东西，实际上它就是一种很特别的编译。但区别于通常意义的编译，我们该如何定义它呢？
>> 
>> **Compiling** 这个术语通常是将一种语言编写的源代码转换为另一种。
**Transpiling** 是一个特定的术语，**用于将一种语言编写的源代码转换为另一种具有相同抽象层次的语言**。
>> 
>> 因此（简单来说）当你编译 C# 时，编译器将函数体转换为中间语言（IL），这不能称为 transpiling ，因为这两种语言的抽象层次完全不同。当你编译 TypeScript 时，编译器将它转换为 JavaScript 。二者的抽象层次相同，所以你可以称之为 transpiling 。
>> 
>> **编译器和 transpiler 都可以在处理过程中优化代码**。
>>
>> 其他一些常见的可以称为 transpiling 的组合包括 C++ 到 C ，CoffeeScript 到 JavaScript ，Dart 到 JavaScript 以及 PHP 到 C++ 。


#### [Making Source Maps Available to Sentry](https://docs.sentry.io/clients/javascript/sourcemaps/#making-source-maps-available-to-sentry)

Source maps can be either:

- Served publicly over HTTP alongside your source files.
- Uploaded directly to Sentry (**recommended**).

##### [Uploading Source Maps to Sentry](https://docs.sentry.io/clients/javascript/sourcemaps/#uploading-source-maps-to-sentry)

> In many cases your application may **sit behind firewalls** or you simply can’t expose source code to the public. Sentry provides an abstraction called **Releases** which you can **attach source artifacts to**.

Sentry 通过 releases 概念对 source artifacts 进行关联和管理；

> The release API is intended to allow you to **store source files (and sourcemaps) within Sentry**. This removes the requirement for them to be web-accessible, and also removes any inconsistency that could come from network flakiness (on either your end, or Sentry’s end).

将 source files 和 sourcemaps 保存到 Sentry 的好处就是避免了由于网络等问题导致的不一致性；

> You can either **interact with the API directly**, **upload sourcemaps with the help of the Sentry CLI** ([Using Sentry CLI](https://docs.sentry.io/clients/javascript/sourcemaps/#upload-sourcemaps-with-cli)) or you can **use sentry-webpack-plugin**.

上传方法：

- 直接通过 Sentry API 进行交互
- 通过 `sentry-cli` 进行上传
- 使用 `sentry-webpack-plugin`

#### [Using Sentry CLI](https://docs.sentry.io/clients/javascript/sourcemaps/#using-sentry-cli)

> You can also use the Sentry [Command Line Interface](https://docs.sentry.io/learn/cli/#sentry-cli) to manage **releases** and **sourcemaps** on Sentry. If you have it installed you can **create** releases with the following command:

针对指定 organization (MY_ORG)和 project (MY_PROJECT) 创建 release ：

```
$ sentry-cli releases -o MY_ORG -p MY_PROJECT new 2da95dfb052f477380608d59d32b4ab9
```

> After you have run this, you can use the `files` command to automatically add all javascript files and sourcemaps below a folder. They are automatically prefixed with a URL or your choice:

上传指定路径下（`/path/to/assets`）的文件到目标 release 下（所有文件均添加 `https://mydomain.invalid/static` 前缀）：

```
$ sentry-cli releases -o MY_ORG -p MY_PROJECT files \
  2da95dfb052f477380608d59d32b4ab9 upload-sourcemaps --url-prefix \
  https://mydomain.invalid/static /path/to/assets
```

> Assets Accessible at Multiple Origins
> 
>> If you leave out the `--url-prefix` parameter the paths will be prefixed with `~/` automatically to support multi origin behavior.

若不指定 `--url-prefix` 参数，则自动使用 `~/` 作为路径前缀（为了支持多源）；

> All files that end with `.js` and `.map` below `/path/to/assets` are automatically uploaded to the **release** `2da95dfb052f477380608d59d32b4ab9` in this case. If you want to use other extensions you can provide it with the `--ext` parameter.


----------


### [Configuration and Authentication](https://docs.sentry.io/learn/cli/configuration/)

#### 登录鉴权

大部分 Sentry 功能都需要登录鉴权后才能使用；

- **交互式鉴权**

```
$ sentry-cli login
```

- **手动直接鉴权**

首先，访问 https://prod.ft.llsops.com/api/ 查看和你账户相关的 Authentication Tokens ，若显示 "You haven't created any authentication tokens yet." ，则需要创建一个新 token ；

其次，将新建 token 的值（例如我的超级 token `017c1fc110c94774b45811835fe85f72cc68bc638c2d4902aa9f1d1149099485`）设置到环境变量 `SENTRY_AUTH_TOKEN` 中：

```
export SENTRY_AUTH_TOKEN=your-auth-token
```

另外，你也可以在 `sentry-cli` 中通过 `--auth-token` 命令行参数指定 token 值，或者直接将 token 值设置到 `.sentryclirc` 配置配置文件中；

默认情况下，`sentry-cli` 将连接到 `sentry.io` ，但是对于 **on-premise** 来说，可以登录到任意指定地址：

```
$ sentry-cli --url https://myserver.invalid/ login
```

**真实登录操作**：

```
deployer@sentry-frontend-prod:~$ sentry-cli --url https://prod.ft.llsops.com/ login
This helps you signing in your sentry-cli with an authentication token.
If you do not yet have a token ready we can bring up a browser for you
to create a token now.

Sentry server: prod.ft.llsops.com
Open browser now? [y/n] n
Enter your token: 017c1fc110c94774b45811835fe85f72cc68bc638c2d4902aa9f1d1149099485
Valid token for user fei.sun@liulishuo.com

Stored token in /home/deployer/.sentryclirc
deployer@sentry-frontend-prod:~$


deployer@sentry-frontend-prod:~$ cat /home/deployer/.sentryclirc
[auth]
token=017c1fc110c94774b45811835fe85f72cc68bc638c2d4902aa9f1d1149099485

[defaults]
url=https://prod.ft.llsops.com/
deployer@sentry-frontend-prod:~$
```

可以看到，登录到指定 URL 后，对应内容直接被设置到 `.sentryclirc` 文件中，方便后续直接使用；


#### 配置文件

`sentry-cli` 可以基于 `.sentryclirc` 配置文件进行配置，也可以基于环境变量和 `.env` 文件进行配置；配置文件的查找方式为从当前路径向上查找，同时 `~/.sentryclirc` 中的默认配置总是进行加载；另外，可以通过命令行参数覆盖相应的配置内容；

配置文件遵循标准的 `INI` 语法；

针对 on-premise 来说，你可以 export 环境变量 `SENTRY_URL` 的值为目标地址：

```
export SENTRY_URL=https://mysentry.invalid/
```

也可以将其添加到 `~/.sentryclirc` 配置文件；`login` 命令就是这么做的：

```
[defaults]
url = https://mysentry.invalid/
```

> `.env` 文件加载
> 
>> `sentry-cli` 默认会加载 `.env` 文件；从 sentry-cli 1.24 开始，可以通过设置 `SENTRY_LOAD_DOTENV=0` 达到不加载该文件的效果；

##### 可配置内容

全部可配置值：

- 全大写部分为对应的环境变量；
- 小括号中的内容为配置文件中的 config key ；


| 环境变量 | config key in `~/.sentryclirc` | 说明 |
| -- | -- | -- |
| **SENTRY_AUTH_TOKEN** | auth.token | the authentication token to use for all communication with Sentry. |
| SENTRY_API_KEY | auth.api_key | the **legacy** API key for authentication if you have one. |
| **SENTRY_URL** | defaults.url | The URL to use to connect to sentry. This defaults to https://sentry.io/. |
| **SENTRY_ORG** | defaults.org | the slug of the **organization** to use for a command. |
| **SENTRY_PROJECT** | defaults.project | the slug of the **project** to use for a command. |
| | http.keepalive | This ini only setting is used to control the behavior of the SDK with regards to HTTP keepalives. **The default is `true`** but it can be set to `false` to disable keepalive support. |
| http_proxy | http.proxy_url | The URL that should be used for the HTTP proxy. The standard `http_proxy` environment variable is also honored. Note that it is lowercase. |
| | http.proxy_username | This ini only setting sets the proxy username in case proxy authentication is required. |
| | http.proxy_password | This ini only setting sets the proxy password in case proxy authentication is required. |
| | http.verify_ssl | This can be **used to disable SSL verification** when set to `false`. You should never do that unless you are working with a known self signed server locally. |
| | http.check_ssl_revoke | If this is set to `false` then SSL revocation checks are disabled on Windows. This can be useful when working with a corporate SSL MITM proxy that does not properly implement revocation checks. Do not use this unless absolutely necessary. |
| | ui.show_notifications | If this is set to `false` some operating system notifications are disabled. This currently primarily affects xcode builds which will not show notifications for background builds. |
| **SENTRY_LOG_LEVEL** | log.level | Configures the log level for the SDK. **The default is `warning`**. If you want to see what the library is doing you can set it to `info` which will spit out more information which might help to debug some issues with permissions. |
| | dsym.max_upload_size | **Sets the maximum upload size in bytes (before compression) of debug symbols into one batch.** The default is **35MB** or **100MB** (depending on the version of `sentry-cli`) which is suitable for `sentry.io` but if you are using a different sentry server you might want to change this limit if necessary. |
| SENTRY_DISABLE_UPDATE_CHECK | update.disable_check | If set to `true` then the automatic update check in `sentry-cli` is disabled. This was introduced in 1.17. Versions before that did not include an update check. The update check is also not enabled for `npm` based installations of `sentry-cli` at the moment. |
| DEVICE_FAMILY | device.family | Device family value reported to Sentry. |
| DEVICE_MODEL | device.model | Device model value reported to Sentry. |

##### 配置确认

> To make sure everything works you can run `sentry-cli info` and it should print out some basic information about the Sentry installation you connect to as well as some authentication information.

对应上面 login 后的情况

```
deployer@sentry-frontend-prod:~$ sentry-cli info
Sentry Server: https://prod.ft.llsops.com/
Default Organization: -
Default Project: -

Authentication Info:
  Method: Auth Token
  User: fei.sun@liulishuo.com
  Scopes:
    - event:admin
    - event:read
    - member:read
    - org:read
    - project:read
    - project:releases
    - team:read
    - project:write
    - team:write
    - member:admin
    - org:write
    - team:admin
    - project:admin
    - org:admin
deployer@sentry-frontend-prod:~$
```

##### 直接指定 Projects

鉴于许多命令（例如 `sentry-cli releases` 命令）需要指定 organization 和 project 的值才能工作，故提供以下方法进行设置：

- **配置默认值**

直接在 `.sentryclirc` 文件中设置（适用于总是针对特定 projects 工作的场景）：

```
[defaults]
project=my-project
org=my-org
```

- **基于环境变量设置**

可以在环境变量中设置相应的目标值：

```
export SENTRY_ORG=my-org
export SENTRY_PROJECT=my-project
```

- 基于 **Properties** 文件

> Additionally sentry-cli supports loading configuration values from .properties files (common in the Java environment). 

- 基于 **Explicit** 选项

最后，还可以针对不同的命令直接设置；设置 organization 的参数选项总是叫做 `--org` 或 `-o` ，设置 project 的参数选项总是叫做 `--project` 或 `-p` ；

需要注意的是，**不同命令指定参数的位置是不同的**，例如针对 releases 命令，organiation 的指定在 releases 后，而 project 的指定必须在 new 之后：

```
$ sentry-cli releases -o my-org new -p my-project 1.0
```

**实际测试情况**：

```
deployer@sentry-frontend-prod:~$ sentry-cli releases --org lls -p moooofly_prj_1 files v1.8 upload-sourcemaps ~/static
> Adding source map references
> Analyzing 57 sources
> Uploading source maps for release v1.8

Source Map Upload Report
  Scripts
    ~/data/flot-data.js
    ~/data/morris-data.js
    ~/dist/js/sb-admin-2.js
    ~/vendor/bootstrap/js/bootstrap.js
    ~/vendor/datatables-plugins/dataTables.bootstrap.js
    ~/vendor/datatables-responsive/dataTables.responsive.js
    ~/vendor/datatables/js/dataTables.bootstrap.js
    ~/vendor/datatables/js/dataTables.bootstrap4.js
    ~/vendor/datatables/js/dataTables.foundation.js
    ~/vendor/datatables/js/dataTables.jqueryui.js
    ~/vendor/datatables/js/dataTables.material.js
    ~/vendor/datatables/js/dataTables.semanticui.js
    ~/vendor/datatables/js/dataTables.uikit.js
    ~/vendor/datatables/js/jquery.dataTables.js
    ~/vendor/flot-tooltip/jquery.flot.tooltip.js
    ~/vendor/flot-tooltip/jquery.flot.tooltip.source.js
    ~/vendor/flot/excanvas.js
    ~/vendor/flot/jquery.colorhelpers.js
    ~/vendor/flot/jquery.flot.canvas.js
    ~/vendor/flot/jquery.flot.categories.js
    ~/vendor/flot/jquery.flot.crosshair.js
    ~/vendor/flot/jquery.flot.errorbars.js
    ~/vendor/flot/jquery.flot.fillbetween.js
    ~/vendor/flot/jquery.flot.image.js
    ~/vendor/flot/jquery.flot.js
    ~/vendor/flot/jquery.flot.pie.js
    ~/vendor/flot/jquery.flot.selection.js
    ~/vendor/flot/jquery.flot.stack.js
    ~/vendor/flot/jquery.flot.symbol.js
    ~/vendor/flot/jquery.flot.threshold.js
    ~/vendor/flot/jquery.flot.time.js
    ~/vendor/flot/jquery.js
    ~/vendor/jquery/jquery.js
    ~/vendor/metisMenu/metisMenu.js
    ~/vendor/morrisjs/morris.js
    ~/vendor/raphael/raphael.js
  Minified Scripts
    ~/dist/js/sb-admin-2.min.js (no sourcemap ref)
    ~/vendor/bootstrap/js/bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables-plugins/dataTables.bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.bootstrap4.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.foundation.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.jqueryui.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.material.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.semanticui.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.uikit.min.js (no sourcemap ref)
    ~/vendor/datatables/js/jquery.dataTables.min.js (no sourcemap ref)
    ~/vendor/datatables/js/jquery.js (no sourcemap ref)
    ~/vendor/flot-tooltip/jquery.flot.tooltip.min.js (no sourcemap ref)
    ~/vendor/flot/excanvas.min.js (no sourcemap ref)
    ~/vendor/flot/jquery.flot.navigate.js (no sourcemap ref)
    ~/vendor/flot/jquery.flot.resize.js (no sourcemap ref)
    ~/vendor/jquery/jquery.min.js (no sourcemap ref)
    ~/vendor/metisMenu/metisMenu.min.js (no sourcemap ref)
    ~/vendor/morrisjs/morris.min.js (no sourcemap ref)
    ~/vendor/raphael/raphael.min.js (no sourcemap ref)
  Source Maps
    ~/vendor/font-awesome/css/font-awesome.css.map
deployer@sentry-frontend-prod:~$
```

多次上传过程中的网络连接情况：

```
root@sentry-frontend-prod:~# while true; do netstat -anpt|grep sentry-cli; sleep 1; echo "----"; done
----
----
----
----
tcp        0      0 172.31.12.235:45468     54.222.132.135:443      ESTABLISHED 17845/sentry-cli
----
tcp        0      0 172.31.12.235:45596     54.222.132.135:443      ESTABLISHED 17845/sentry-cli
----
tcp        0      0 172.31.12.235:45724     54.222.132.135:443      ESTABLISHED 17845/sentry-cli
----
tcp        0      0 172.31.12.235:45860     54.222.132.135:443      ESTABLISHED 17845/sentry-cli
----
tcp        0    517 172.31.12.235:46866     192.30.255.116:443      ESTABLISHED 17845/sentry-cli
----
tcp        0      0 172.31.12.235:46866     192.30.255.116:443      ESTABLISHED 17845/sentry-cli
----
tcp        0    155 172.31.12.235:46866     192.30.255.116:443      ESTABLISHED 17845/sentry-cli
----
----
----
----
tcp        0      0 172.31.12.235:45958     54.222.132.135:443      ESTABLISHED 17897/sentry-cli
----
tcp        0      0 172.31.12.235:46094     54.222.132.135:443      ESTABLISHED 17897/sentry-cli
----
tcp        0   1035 172.31.12.235:46266     54.222.132.135:443      ESTABLISHED 17897/sentry-cli
----
tcp        0      0 172.31.12.235:46394     54.222.132.135:443      ESTABLISHED 17897/sentry-cli
----
tcp        0    517 172.31.12.235:55732     192.30.255.117:443      ESTABLISHED 17897/sentry-cli
----
----
----
----
tcp        0      0 172.31.12.235:46530     54.222.132.135:443      ESTABLISHED 17942/sentry-cli
----
tcp        0      0 172.31.12.235:46650     54.222.132.135:443      ESTABLISHED 17942/sentry-cli
----
tcp        0      0 172.31.12.235:46786     54.222.132.135:443      ESTABLISHED 17942/sentry-cli
----
tcp        0    140 172.31.12.235:46930     54.222.132.135:443      ESTABLISHED 17942/sentry-cli
----
tcp        0    155 172.31.12.235:47956     192.30.255.116:443      ESTABLISHED 17942/sentry-cli
----
----
```

结论：

- 上传 N 次，没有一次出现 Broken Pipe 错误（因此可以断定问题的出现原因是之前 nginx 的配置存在问题）；
- `sentry-cli` 命令的执行存在两部分网络交互（通过增加 `--log-level TRACE` 可以看到）；


用于说明问题的部分输出：

```
...
> ~/vendor/datatables/js/dataTables.uikit.js
██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████░░░ 56/57
[INFO] sentry_cli::api request DELETE https://prod.ft.llsops.com/api/0/projects/lls/moooofly_prj_1/releases/v1.8/files/16779/
[INFO] sentry_cli::api using token authentication
[INFO] sentry_cli::api > DELETE /api/0/projects/lls/moooofly_prj_1/releases/v1.8/files/16779/ HTTP/1.1
[INFO] sentry_cli::api > Host: prod.ft.llsops.com
[INFO] sentry_cli::api > Accept: */*
[INFO] sentry_cli::api > User-Agent: sentry-cli/1.24.1
[INFO] sentry_cli::api > Authorization: Bearer 017c1fc1***
[INFO] sentry_cli::api < HTTP/1.1 204 NO CONTENT
[INFO] sentry_cli::api < Server: nginx/1.10.3 (Ubuntu)
[INFO] sentry_cli::api < Date: Wed, 10 Jan 2018 05:52:03 GMT
[INFO] sentry_cli::api < Content-Length: 0
[INFO] sentry_cli::api < Connection: close
[INFO] sentry_cli::api < X-XSS-Protection: 1; mode=block
[INFO] sentry_cli::api < Content-Language: en
[INFO] sentry_cli::api < X-Content-Type-Options: nosniff
[INFO] sentry_cli::api < Vary: Accept-Language, Cookie
[INFO] sentry_cli::api < Allow: GET, PUT, DELETE, HEAD, OPTIONS
[INFO] sentry_cli::api < X-Frame-Options: deny
[INFO] sentry_cli::api < Strict-Transport-Security: max-age=31536000
[INFO] sentry_cli::api response: 204
[INFO] sentry_cli::api body:
[INFO] sentry_cli::api request POST https://prod.ft.llsops.com/api/0/projects/lls/moooofly_prj_1/releases/v1.8/files/
[INFO] sentry_cli::api using token authentication
[INFO] sentry_cli::api sending form data
[INFO] sentry_cli::api > POST /api/0/projects/lls/moooofly_prj_1/releases/v1.8/files/ HTTP/1.1
[INFO] sentry_cli::api > Host: prod.ft.llsops.com
[INFO] sentry_cli::api > Accept: */*
[INFO] sentry_cli::api > User-Agent: sentry-cli/1.24.1
[INFO] sentry_cli::api > Authorization: Bearer 017c1fc1***
[INFO] sentry_cli::api > Content-Length: 4970
[INFO] sentry_cli::api > Content-Type: multipart/form-data; boundary=------------------------5a456db97ce43007
[INFO] sentry_cli::api < HTTP/1.1 201 CREATED
[INFO] sentry_cli::api < Server: nginx/1.10.3 (Ubuntu)
[INFO] sentry_cli::api < Date: Wed, 10 Jan 2018 05:52:03 GMT
[INFO] sentry_cli::api < Content-Type: application/json
[INFO] sentry_cli::api < Content-Length: 235
[INFO] sentry_cli::api < Connection: close
[INFO] sentry_cli::api < X-XSS-Protection: 1; mode=block
[INFO] sentry_cli::api < Content-Language: en
[INFO] sentry_cli::api < X-Content-Type-Options: nosniff
[INFO] sentry_cli::api < Vary: Accept-Language, Cookie
[INFO] sentry_cli::api < Allow: GET, POST, HEAD, OPTIONS
[INFO] sentry_cli::api < X-Frame-Options: deny
[INFO] sentry_cli::api < Strict-Transport-Security: max-age=31536000
[INFO] sentry_cli::api response: 201

Source Map Upload Report
  Scripts
    ~/data/flot-data.js
    ~/data/morris-data.js
    ~/dist/js/sb-admin-2.js
    ~/vendor/bootstrap/js/bootstrap.js
    ~/vendor/datatables-plugins/dataTables.bootstrap.js
    ~/vendor/datatables-responsive/dataTables.responsive.js
    ~/vendor/datatables/js/dataTables.bootstrap.js
    ~/vendor/datatables/js/dataTables.bootstrap4.js
    ~/vendor/datatables/js/dataTables.foundation.js
    ~/vendor/datatables/js/dataTables.jqueryui.js
    ~/vendor/datatables/js/dataTables.material.js
    ~/vendor/datatables/js/dataTables.semanticui.js
    ~/vendor/datatables/js/dataTables.uikit.js
    ~/vendor/datatables/js/jquery.dataTables.js
    ~/vendor/flot-tooltip/jquery.flot.tooltip.js
    ~/vendor/flot-tooltip/jquery.flot.tooltip.source.js
    ~/vendor/flot/excanvas.js
    ~/vendor/flot/jquery.colorhelpers.js
    ~/vendor/flot/jquery.flot.canvas.js
    ~/vendor/flot/jquery.flot.categories.js
    ~/vendor/flot/jquery.flot.crosshair.js
    ~/vendor/flot/jquery.flot.errorbars.js
    ~/vendor/flot/jquery.flot.fillbetween.js
    ~/vendor/flot/jquery.flot.image.js
    ~/vendor/flot/jquery.flot.js
    ~/vendor/flot/jquery.flot.pie.js
    ~/vendor/flot/jquery.flot.selection.js
    ~/vendor/flot/jquery.flot.stack.js
    ~/vendor/flot/jquery.flot.symbol.js
    ~/vendor/flot/jquery.flot.threshold.js
    ~/vendor/flot/jquery.flot.time.js
    ~/vendor/flot/jquery.js
    ~/vendor/jquery/jquery.js
    ~/vendor/metisMenu/metisMenu.js
    ~/vendor/morrisjs/morris.js
    ~/vendor/raphael/raphael.js
  Minified Scripts
    ~/dist/js/sb-admin-2.min.js (no sourcemap ref)
    ~/vendor/bootstrap/js/bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables-plugins/dataTables.bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.bootstrap.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.bootstrap4.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.foundation.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.jqueryui.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.material.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.semanticui.min.js (no sourcemap ref)
    ~/vendor/datatables/js/dataTables.uikit.min.js (no sourcemap ref)
    ~/vendor/datatables/js/jquery.dataTables.min.js (no sourcemap ref)
    ~/vendor/datatables/js/jquery.js (no sourcemap ref)
    ~/vendor/flot-tooltip/jquery.flot.tooltip.min.js (no sourcemap ref)
    ~/vendor/flot/excanvas.min.js (no sourcemap ref)
    ~/vendor/flot/jquery.flot.navigate.js (no sourcemap ref)
    ~/vendor/flot/jquery.flot.resize.js (no sourcemap ref)
    ~/vendor/jquery/jquery.min.js (no sourcemap ref)
    ~/vendor/metisMenu/metisMenu.min.js (no sourcemap ref)
    ~/vendor/morrisjs/morris.min.js (no sourcemap ref)
    ~/vendor/raphael/raphael.min.js (no sourcemap ref)
  Source Maps
    ~/vendor/font-awesome/css/font-awesome.css.map
[INFO] sentry_cli::api request GET https://api.github.com/repos/getsentry/sentry-cli/releases/latest
[INFO] sentry_cli::api > GET /repos/getsentry/sentry-cli/releases/latest HTTP/1.1
[INFO] sentry_cli::api > Host: api.github.com
[INFO] sentry_cli::api > Accept: */*
[INFO] sentry_cli::api > User-Agent: sentry-cli/1.24.1
[INFO] sentry_cli::api < HTTP/1.1 200 OK
[INFO] sentry_cli::api < Date: Wed, 10 Jan 2018 05:52:04 GMT
[INFO] sentry_cli::api < Content-Type: application/json; charset=utf-8
[INFO] sentry_cli::api < Content-Length: 10869
[INFO] sentry_cli::api < Server: GitHub.com
[INFO] sentry_cli::api < Status: 200 OK
[INFO] sentry_cli::api < X-RateLimit-Limit: 60
[INFO] sentry_cli::api < X-RateLimit-Remaining: 51
[INFO] sentry_cli::api < X-RateLimit-Reset: 1515566277
[INFO] sentry_cli::api < Cache-Control: public, max-age=60, s-maxage=60
[INFO] sentry_cli::api < Vary: Accept
[INFO] sentry_cli::api < ETag: "c2c6954f31d08e7135ae8e78c5c2bfa2"
[INFO] sentry_cli::api < Last-Modified: Wed, 03 Jan 2018 22:30:56 GMT
[INFO] sentry_cli::api < X-GitHub-Media-Type: github.v3; format=json
[INFO] sentry_cli::api < Access-Control-Expose-Headers: ETag, Link, Retry-After, X-GitHub-OTP, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval
[INFO] sentry_cli::api < Access-Control-Allow-Origin: *
[INFO] sentry_cli::api < Content-Security-Policy: default-src 'none'
[INFO] sentry_cli::api < Strict-Transport-Security: max-age=31536000; includeSubdomains; preload
[INFO] sentry_cli::api < X-Content-Type-Options: nosniff
[INFO] sentry_cli::api < X-Frame-Options: deny
[INFO] sentry_cli::api < X-XSS-Protection: 1; mode=block
[INFO] sentry_cli::api < X-Runtime-rack: 0.040956
[INFO] sentry_cli::api < Vary: Accept-Encoding
[INFO] sentry_cli::api < X-GitHub-Request-Id: B14A:CDA7:3B0FF3:50171E:5A55AA04
[INFO] sentry_cli::api response: 200
...
```


----------



## 问题梳理

- 命令 `sentry-cli releases files v1.1 upload-sourcemaps ~/static` 可重复操作么？

> 可以；通过 `--log-level TRACE` 可以看到其处理方式为：若文件存在，则先删除再上传，若文件不存在，则直接上传；

- 上述命令执行到一半后失败，会产生什么影响？

> 应该就是缺少部分文件，进而导致进行 event 处理时无法提供完整信息；（由于无法再次复现之前的 broken pipe 错误，故以上结论为推测）

- 上述命令执行成功后，如何确定功能正常？

> 可以在类似 https://prod.ft.llsops.com/lls/cchybrid/releases/a66fd95/artifacts/ 的链接上查看数量是否正确；另外就要在进行 event 的实际处理过程中看是否有信息缺失了；

- 在 sentry 上自建一个 project 能够验证这个问题么？

> 可以；针对上传文件导致 broken pipe 这个问题本身是可以的，针对 event 进行分析，则还是需要真实环境；

- `sentry-cli` 命令未指定 `--url <URL>` 参数，连接的目标地址是哪里？

> `sentry-cli` 默认连接到 https://sentry.io/ ，可以通过配置文件或环境变量设置目标 URL ，之后就无需再次指定了；

- `sentry-cli` 是基于 HTTP 还是 HTTPS ？

> 基于 HTTPS ；实际上是基于 authentication token 的方式解决了鉴权问题；


----------


## 只言片语

- Sentry provides an abstraction called **Releases** which you can **attach source artifacts to**.
- **Release artifacts** 可以理解为 release 衍生物，用于在后续 event 处理过程中提供更多的信息；
- Release artifacts are only considered at time of event processing.
- Releases are shared across the organization.
- Releases work on projects.
- **Authentication tokens** allow you to perform actions against the Sentry API on behalf of your account. They're the easiest way to get started using the API.
