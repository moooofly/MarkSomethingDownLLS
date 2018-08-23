# 排查 Sentry 上传 sourcemap 时遇到的 500 错误

> https://phab.llsapp.com/T54237

## 背景信息

by jianhua.cheng

> - 跟文件数不知道有没有关系，我之前一个分支 40 个文件就能上传成功，换了构建脚本，现在有 60 个文件，就会出现 **Internal Server 500** 的错误；
> - 这回看到了更具体的错误信息：`StatusCodeError: 500 - "{"detail": "Internal Error", "errorId": "a9f76f2d3b774e02b5fc80cdfd86a259"}"`；还是上传 sourcemap 文件到 ft sentry 服务器上遇到的失败问题；
> - 我主要用库做的这件事情, 不是直接用的 `sentry-cli` ，要贴也就是贴个 `node scripts/build.js`，错误结果就是上面的了，没有更多；
> - 我只是在一个分支里改了个构建配置，同样的构建和上传 sourcemap 代码在另一个分支就是一直成功的；
> - 我的配置再有问题，生成的也只是 JavaScript 文件，`sentry-cli` 的输入也只是多个文件而已，而且错误是 500 ；如果是我参数有问题的话按理也不会是 5xx 的错误码吧；


by moooofly

> - 每次都失败，还是偶尔失败？
> - 请把完整执行命令和结果贴出来，我看下；
> - 能用等价的 sentry-cli 命令确认一下么？
> - 是否有可能是域名变更导致的？


## 问题表现

- jianhua.cheng 反馈的 500 上传错误

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20%E4%B8%8A%E4%BC%A0%E9%97%AE%E9%A2%98%20-%20%E5%BC%80%E5%8F%91%E4%BA%BA%E5%91%98%E5%8F%8D%E9%A6%88%E7%9A%84%20500%20%E4%B8%8A%E4%BC%A0%E9%94%99%E8%AF%AF.png)

- 自测复现的 500 上传错误

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20%E4%B8%8A%E4%BC%A0%E9%97%AE%E9%A2%98%20-%20%E8%87%AA%E6%B5%8B%E5%A4%8D%E7%8E%B0%E7%9A%84%20500%20%E4%B8%8A%E4%BC%A0%E9%94%99%E8%AF%AF.png)

## 测试脚本

```
#!/bin/sh
# VERSION=`sentry-cli releases propose-version`
VERSION="b513627c6215aadc86805975e441bf56dfd6a1b8."

export SENTRY_PROJECT="lms-mobile-staging"
export SENTRY_ORG="lls"
export SENTRY_AUTH_TOKEN="50682a06586c4f8e9133183a54972b5c9812cc44e6f14731b67d21849ef7f882"
export SENTRY_URL="https://prod-ft.llsops.com"

sentry-cli releases new "$VERSION"

sentry-cli \
releases files "$VERSION" \
upload-sourcemaps --url-prefix "https://cdn.llscdn.com/hybrid/lms-mobile/" "./dist" \
--validate

sentry-cli releases finalize "$VERSION"
```

测试文件

（略）


## 问题解决

- sentry_web 输出的触发 500 错误的异常日志

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20%E4%B8%8A%E4%BC%A0%E9%97%AE%E9%A2%98%20-%20sentry_web%20%E8%BE%93%E5%87%BA%E7%9A%84%E8%A7%A6%E5%8F%91%20500%20%E9%94%99%E8%AF%AF%E7%9A%84%E5%BC%82%E5%B8%B8%E6%97%A5%E5%BF%97.png)

- 导致 500 的几个名字过长的文件

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20%E4%B8%8A%E4%BC%A0%E9%97%AE%E9%A2%98%20-%20%E5%AF%BC%E8%87%B4%20500%20%E7%9A%84%E8%BF%87%E9%95%BF%E6%96%87%E4%BB%B6%E5%90%8D.png)

- 去掉超长名字文件后上传成功

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/sentry%20%E4%B8%8A%E4%BC%A0%E9%97%AE%E9%A2%98%20-%20%E5%8E%BB%E6%8E%89%E8%B6%85%E9%95%BF%E5%90%8D%E5%AD%97%E6%96%87%E4%BB%B6%E5%90%8E%E4%B8%8A%E4%BC%A0%E6%88%90%E5%8A%9F.png)

## 结论

- 需要 frontend 提供文件名长度的明确上限值；
- （我）从代码层面确认如何进行调整；
- 建议文件命名在不影响理解的情况下，应该尽量缩写；






