# bazel

- [Installing Bazel on Ubuntu](#installing-bazel-on-ubuntu)
- [How to Use Bazel](#how-to-use-bazel)
- [Problem with Using Bazel in Liulishuo](#problem-with-using-bazel-in-liulishuo)
    - [Solution to the Dependency Problem: External](#solution-to-the-dependency-problem-external)
    - [Solution to the Dependency Problem: Internal](#solution-to-the-dependency-problem-internal)
    - [(Planned) Solution to the Downloading Problem](#planned-solution-to-the-downloading-problem)
- [错误解决](#错误解决)
- [一些说法](#一些说法)
- [Bazel Golang Hello World](#bazel-golang-hello-world)
- [相关文章](#相关文章)
    
## Installing Bazel on Ubuntu

> Ref: https://docs.bazel.build/versions/master/install-ubuntu.html

Recommend: Use the [binary installer](https://github.com/bazelbuild/bazel/releases)

- Step 1: Install required packages

```
$ sudo apt-get install pkg-config zip g++ zlib1g-dev unzip python
```

- Step 2: Download Bazel

```
$ wget https://github.com/bazelbuild/bazel/releases/download/0.17.2/bazel-0.17.2-installer-linux-x86_64.sh
$ chmod +x chmod +x bazel-0.17.2-installer-linux-x86_64.sh
```

- Step 3: Run the installer

```
$ ./bazel-0.17.2-installer-linux-x86_64.sh --user

Bazel installer
---------------

Bazel is bundled with software licensed under the GPLv2 with Classpath exception.
You can find the sources next to the installer on our release page:
   https://github.com/bazelbuild/bazel/releases

# Release 0.17.2 (2018-09-21)

Baseline: aa118ca818baf722aede0bc48d0a17584fa45b6e

Cherry picks:
   + 0e0462589528154cb5160411991075a2000b5452:
     Update checker framework dataflow and javacutil versions
   + 3987300d6651cf0e6e91b395696afac6913a7d66:
     Stop using --release in versioned java_toolchains
   + 438b2773b8c019afa46be470b90bcf70ede7f2ef:
     make_deb: Add new empty line in the end of conffiles file
   + 504401791e0a0e7e3263940e9e127f74956e7806:
     Properly mark configuration files in the Debian package.
   + 9ed9d8ac4347408d15c8fce7c9c07e5c8e658b30:
     Add flag
     --incompatible_symlinked_sandbox_expands_tree_artifacts_in_runfil
     es_tree.
   + 22d761ab42dfb1b131f1facbf490ccdb6c17b89c:
     Update protobuf to 3.6.1 -- add new files
   + 27303d79c38f2bfa3b64ee7cd7a6ef03a9a87842:
     Update protobuf to 3.6.1 -- update references
   + ddc97ed6b0367eb443e3e09a28d10e65179616ab:
     Update protobuf to 3.6.1 -- remove 3.6.0 sources
   + ead1002d3803fdfd4ac68b4b4872076b19d511a2:
     Fix protobuf in the WORKSPACE
   + 12dcd35ef7a26d690589b0fbefb1f20090cbfe15:
     Revert "Update to JDK 10 javac"
   + 7eb9ea150fb889a93908d96896db77d5658e5005:
     Automated rollback of
     https://github.com/bazelbuild/bazel/commit/808ec9ff9b5cec14f23a4b
     a106bc5249cacc8c54 and
     https://github.com/bazelbuild/bazel/commit/4c9149d558161e7d3e363f
     b697f5852bc5742a36 and some manual merging.
   + 4566a428c5317d87940aeacfd65f1018340e52b6:
     Fix tests on JDK 9 and 10
   + 1e9f0aa89dad38eeab0bd40e95e689be2ab6e5e5:
     Fix more tests on JDK 9 and 10
   + a572c1cbc8c26f625cab6716137e2d57d05cfdf3:
     Add ubuntu1804_nojava, ubuntu1804_java9, ubuntu1804_java10 to
     postsubmit.
   + 29f1de099e4f6f0f50986aaa4374fc5fb7744ee8:
     Disable Android shell tests on the "nojava" platform.
   + b495eafdc2ab380afe533514b3bcd7d5b30c9935:
     Update bazel_toolchains to latest release.
   + 9323c57607d37f9c949b60e293b573584906da46:
     Windows: fix writing java.log
   + 1aba9ac4b4f68b69f2d91e88cfa8e5dcc7cb98c2:
     Automated rollback of commit
     de22ab0582760dc95f33e217e82a7b822378f625.
   + 2579b791c023a78a577e8cb827890139d6fb7534:
     Fix toolchain_java9 on --host_javabase=<jdk9> after
     7eb9ea150fb889a93908d96896db77d5658e5005
   + 2834613f93f74e988c51cf27eac0e59c79ff3b8f:
     Include also ext jars in the bootclasspath jar.
   + fdb09a260dead1e1169f94584edc837349a4f4a5:
     Release 0.17.1 (2018-09-14)
   + 1d956c707e1c843896ac58a341c335c9c149073d:
     Do not fail the build when gcov is not installed
   + 2e677fb6b8f309b63558eb13294630a91ee0cd33:
     Ignore unrecognized VM options in desugar.sh, such as the JVM 9
     flags to silence warnings.

Important changes:

  - In the future, Bazel will expand tree artifacts in runfiles, too,
    which causes the sandbox to link each file individually into the
    sandbox directory, instead of symlinking the entire directory. In
    this release, the behavior is not enabled by default yet. Please
    try it out via
    --incompatible_symlinked_sandbox_expands_tree_artifacts_in_runfile
    s_tree and let us know if it causes issues. If everything looks
    good, this behavior will become the default in a following
    release.

## Build informations
   - [Commit](https://github.com/bazelbuild/bazel/commit/76d175c)
Uncompressing.......

Bazel is now installed!

Make sure you have "/root/bin" in your path. You can also activate bash
completion by adding the following line to your ~/.bashrc:
  source /root/.bazel/bin/bazel-complete.bash

See http://bazel.build/docs/getting-started.html to start a new project!
```

```
$ bazel --help
                                                           [bazel release 0.9.0]
Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  coverage            Generates code coverage report for specified test targets.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  license             Prints the license of this software.
  mobile-install      Installs targets to mobile devices.
  print_action        Prints the command line args for compiling a file.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.
[#27#root@ubuntu-1604 ~/workspace/tmp]$
```

The `--user` flag installs Bazel to the `$HOME/bin` directory on your system and sets the `.bazelrc` path to `$HOME/.bazelrc`. Use the `--help` command to see additional installation options.

- Step 4: Set up your environment

Add this command to your `~/.bashrc` file.

```
export PATH="$PATH:$HOME/bin"
```

## How to Use Bazel

- **Write a `WORKSPACE` file**
    - Indicating root of the repository
        - All paths are relative to the dir where `WORKSPACE` lives
        - No more “relative to what”
    - Specifying all external repository dependencies
- **Write a `BUILD` file in every directory**
    - Specifies build targets
    - For each target, specifies dependencies

> - The "**WORKSPACE**" file is mandatory for every Bazel project. This file typically contains references to external build rules and project dependencies.
> - By definition, every package contains a **BUILD** file, which is a short program written in the Build Language. Most BUILD files appear to be little more than a series of declarations of build rules; indeed, the declarative style is strongly encouraged when writing BUILD files.
    
## Problem with Using Bazel in Liulishuo

Like everything, Bazel has its downside too

Designed with single repository model in mind

- External repository dependency is an “afterthought”
- Diamond dependencies often cause build failure
    - A depends on B 1.0 and C 1.1
    - B 1.0 depends on D 1.0
    - C 1.1 depends on D 1.1
- No build in method to build from repository head
    - Same proto field change problem

Bazel assumes first world internet speed

- Downloading from `googlesource.com` is often slow
- Downloading from `github.com` is even slower

### Solution to the Dependency Problem: External

Register non-Liulishuo dependencies in a one place

- git.llsapp.com/common/bazel/build_rules/deps.bzl

### Solution to the Dependency Problem: Internal

Always pull Liulishuo repository from head

- Ensures internal dependencies up to date

> Caveat: You need to specify `BAZEL_RUNID=$RANDOM` in the CI script to ensure pulling the repository from head.

### (Planned) Solution to the Downloading Problem

Register all external dependencies in one place

- In progress, currently only done for GRPC dependencies

Build all external dependencies into Bazel docker image

- No more downloading
- Periodically updating the Bazel image to upgrade external repositories

## 错误解决

```
[#10#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$bazel build //...
ERROR: error loading package '': Encountered error while reading extension file 'build_rules/grpc/go/rules.bzl': no such package '@com_llsapp_git_common_bazel//build_rules/grpc/go': no such package '@io_bazel_rules_go_repository_tools//': no such package '@io_bazel_rules_go_toolchain//': no such package '@local_go_linux//': Expected directory at /usr/lib/go but it does not exist.
ERROR: error loading package '': Encountered error while reading extension file 'build_rules/grpc/go/rules.bzl': no such package '@com_llsapp_git_common_bazel//build_rules/grpc/go': no such package '@io_bazel_rules_go_repository_tools//': no such package '@io_bazel_rules_go_toolchain//': no such package '@local_go_linux//': Expected directory at /usr/lib/go but it does not exist.
INFO: Elapsed time: 0.725s
FAILED: Build did NOT complete successfully (0 packages loaded)
[#11#root@ubuntu-1604 /go/src/git.llsapp.com/ops/hunter/hunter-demo-golang]$
```

只需要

```
ln -s /usr/local/go /usr/lib/go
```

## 一些说法

- 目的：实现自动构建项目中所依赖的包。
- 使用方法：在每一级的 BUILD 中写明该类的依赖，由根目录下的 BUILD 文件，逐渐深入到各个子目录下，最后实现整个项目所依赖的包的构建。但需要在 `WORKSPACE` 和 `genereated_workspace.bzl` 中先声明，否则系统会认为依赖包不存在。
- bazel build ：一般我们的 `bazel build` 命令都会写到 docker-compose.sh 文件里面。但是，在 build 的时候，不能只写 `bazel build ***`，如果这样，build 每次都只会下载新的依赖放到指定文件夹中，而不会调用。这样造成的结果就是，每一次 CI 的 test 的时间都会很长，因此，在 build 的时候，一定要指明 output_user_root 、--output_base 等参数！！！！


## Bazel Golang Hello World

Ref: 

- https://chrislovecnm.com/golang/bazel/bazel-hello-world/
- https://github.com/chrislovecnm/go-bazel-hello-world/

```
[#18#root@ubuntu-1604 /go/src/github.com/moooofly/playground]$mkdir go-bazel_example
[#19#root@ubuntu-1604 /go/src/github.com/moooofly/playground]$cd go-bazel_example/

[#27#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$wget https://raw.githubusercontent.com/chrislovecnm/go-bazel-hello-world/master/hack/create-bazel-workspace.sh
[#28#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$chmod +x create-bazel-workspace.sh

[#31#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$./create-bazel-workspace.sh
please provide your projects base go path as an argument.
./create-bazel-workspace.sh: line 5: ehco: command not found
[#32#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$./create-bazel-workspace.sh github.com/moooofly/playground/go-bazel_example
project WORKSPACE and BUILD files created, run gazelle to create required BUILD.bazel files
[#33#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$ll
total 20
drwxr-xr-x  2 root root 4096 Oct 15 16:39 ./
drwxr-xr-x 19 root root 4096 Oct 15 16:34 ../
-rw-r--r--  1 root root  262 Oct 15 16:39 BUILD
-rwxr-xr-x  1 root root 1293 Oct 15 16:36 create-bazel-workspace.sh*
-rw-r--r--  1 root root  750 Oct 15 16:39 WORKSPACE
[#34#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$
（添加 go 文件）
[#106#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$tree .
.
├── BUILD
├── cmd
│   └── main.go
├── create-bazel-workspace.sh
├── Makefile
├── pkg
│   └── cmd
│       ├── hello.go
│       └── root.go
└── WORKSPACE

3 directories, 7 files
[#107#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$
```

- Running Gazelle to create various `BUILD.bazel` files

```
[#37#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$bazel run //:gazelle
...........
INFO: Analysed target //:gazelle (33 packages loaded).
INFO: Found 1 target...
Target //:gazelle up-to-date:
  bazel-bin/gazelle_script.bash
  bazel-bin/gazelle
INFO: Elapsed time: 230.696s, Critical Path: 3.78s
INFO: Build completed successfully, 24 total actions

INFO: Running command line: bazel-bin/gazelle
[#38#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$

（后续再重跑，由于缓存的存在，则会非常快）

[#111#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$tree -L 3
.
├── bazel-bin -> /root/.cache/bazel/_bazel_root/f9d68222aea57b473f3dc4bda6e12755/execroot/__main__/bazel-out/k8-fastbuild/bin
├── bazel-genfiles -> /root/.cache/bazel/_bazel_root/f9d68222aea57b473f3dc4bda6e12755/execroot/__main__/bazel-out/k8-fastbuild/genfiles
├── bazel-go-bazel_example -> /root/.cache/bazel/_bazel_root/f9d68222aea57b473f3dc4bda6e12755/execroot/__main__
├── bazel-out -> /root/.cache/bazel/_bazel_root/f9d68222aea57b473f3dc4bda6e12755/execroot/__main__/bazel-out
├── bazel-testlogs -> /root/.cache/bazel/_bazel_root/f9d68222aea57b473f3dc4bda6e12755/execroot/__main__/bazel-out/k8-fastbuild/testlogs
├── BUILD
├── cmd
│   ├── BUILD.bazel
│   └── main.go
├── create-bazel-workspace.sh
├── Makefile
├── pkg
│   └── cmd
│       ├── BUILD.bazel
│       ├── hello.go
│       └── root.go
└── WORKSPACE

8 directories, 9 files
[#112#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$
```

One thing that Gazelle did not do initially was define a rule to build a binary.

```
[#115#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$make all
bazel build //...
INFO: Analysed 4 targets (6 packages loaded).
INFO: Found 4 targets...
INFO: Elapsed time: 15.475s, Critical Path: 1.19s
INFO: Build completed successfully, 10 total actions
[#116#root@ubuntu-1604 /go/src/github.com/moooofly/playground/go-bazel_example]$
```

## 相关文章

- https://git.llsapp.com/codelab/bazel_grpc  -- bazel 实验项目
- https://git.llsapp.com/common/util -- examples of building go and C++ libraries/binaries
- https://git.llsapp.com/common/bazel -- skylark rules for building grpc targets
- https://git.llsapp.com/common/protos -- proto and grpc rules
- https://www.youtube.com/watch?v=6GCDfoAOKIY -- bazel tech talk
- https://zoom.us/recording/play/VSK8qE1mSk1Syl5KLXttXg8DnObd3nzbOtw0UyFyUpxoAMMqpv7ytUt8OItr-hMD -- codelab talk slide
- https://bazel.build/  -- tutorial and official document
- https://github.com/bazelbuild/bazel -- github 地址
- https://docs.bazel.build/versions/master/output_directories.html -- Output Directory Layout
