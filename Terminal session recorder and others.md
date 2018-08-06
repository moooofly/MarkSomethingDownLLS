# Terminal Session Recorder and Others

## asciinema/asciinema

> github: [asciinema/asciinema](https://github.com/asciinema/asciinema)

一句话说明：Terminal session recorder

官网地址：https://asciinema.org

[asciinema](https://asciinema.org) is a default asciinema-server instance, and prints a secret link you can use to watch your recording in a web browser.

### 原理

将终端的操作记录成 JSON 格式，然后使用 JavaScript 解析，配合 CSS 展示，看起来像是视频播放器。实际上就是文本，相比 GIF 和视频文件体积非常之小（时长 2 分 50 秒的录屏只有 325KB），无需缓冲播放，也可以方便的分享给别人或嵌入到网页中。

经确认，存在两种格式：

- [asciicast file format (version 1)](https://github.com/asciinema/asciinema/blob/master/doc/asciicast-v1.md) -- asciicast file is JSON file containing meta-data like duration or title of the recording, and the actual content printed to terminal's stdout during recording.
- [asciicast file format (version 2)](https://github.com/asciinema/asciinema/blob/master/doc/asciicast-v2.md)  -- asciicast v2 file is newline-delimited JSON file.

> NOTE:
>
> - [asciicast](https://github.com/asciinema/asciinema/blob/master/doc/asciicast-v1.md) is asciinema recording.
> - Suggested file extension of asciicast v2 is `.cast`.
> - Suggested file extension of asciicast v1 is `.json`.

### 存在的问题

使用时通过嵌入一个链接到 asciinema.org 上静态图片，点击后，跳转到目标网站再播放；用户体验没有本地直接播放好；

例如：
[![demo](https://asciinema.org/a/113463.png)](https://asciinema.org/a/113463?autoplay=1)

### 安装

- Mac

```
brew install asciinema
```

- Ubuntu

```
sudo apt-add-repository ppa:zanchey/asciinema
sudo apt-get update
sudo apt-get install asciinema
```

可能会报如下错误信息，不过不影响使用；

```
...
Fetched 3,832 kB in 11s (341 kB/s)
Reading package lists... Done
W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: http://storage.googleapis.com/bazel-apt stable InRelease: The following signatures were invalid: KEYEXPIRED 1527185977  KEYEXPIRED 1527185977  KEYEXPIRED 1527185977
W: Failed to fetch http://storage.googleapis.com/bazel-apt/dists/stable/InRelease  The following signatures were invalid: KEYEXPIRED 1527185977  KEYEXPIRED 1527185977  KEYEXPIRED 1527185977
W: Some index files failed to download. They have been ignored, or old ones used instead.
[#3#root@ubuntu-1604 ~]$
```


### Examples

- Record your first session

```
asciinema rec first.cast
```

> first.cast is a JSON file

- Now replay it with double speed:

```
asciinema play -s 2 first.cast
```

- Or with normal speed but with idle time limited to 2 seconds:

```
asciinema play -i 2 first.cast
```

- Print full output of recorded asciicast to a terminal immediately.

```
asciinema cat first.cast
```

- If you want to watch and share it on the asciinema.org, upload it:

```
asciinema upload first.cast
```

- Replay the recorded session from asciinema.org URL.

```
asciinema play https://asciinema.org/a/Rv4XsAFMRdbr8SFZVRgEgE8oM
```

- You can record and upload in one step by omitting the filename:

```
asciinema rec
```

This spawns a new shell instance and records all terminal output. When you're ready to finish simply exit the shell either by typing `exit` or hitting `Ctrl-D`.

- If you want to manage your recordings on asciinema.org, you need to authenticate. Run the following command and open displayed URL in your web browser:

```
asciinema auth
```

### 配置文件

- [`$HOME/.config/asciinema/config`](https://github.com/asciinema/asciinema#configuration-file) -- 需要自行创建
- [`$HOME/.config/asciinema/install-id`](https://github.com/asciinema/asciinema#auth) -- Install ID is a random ID (UUID v4) generated locally when you run asciinema for the first time. It's purpose is to connect local machine with uploaded recordings, so they can later be associated with asciinema.org account.


## marionebl/svg-term-cli

> github: [marionebl/svg-term-cli](https://github.com/marionebl/svg-term-cli)

一句话说明：Share terminal sessions via **SVG** and **CSS**

### 功能

- Render `asciicast` to animated SVG
- Share `asciicasts` everywhere (sans JS)
- Style with common color profiles

存在的价值：

- 替代 GIF 格式的 `asciicast` recordings 为 SVG 格式，解决某些情况下无法正常使用 `asciinema` player 的问题；

> Replace **GIF** `asciicast` recordings where you can not use the [`asciinema` player](https://asciinema.org/), e.g. `README.md` files on GitHub and the `npm` registry.
>
> The image at the top of this README is an example. See how sharp the text looks, even when you zoom in? That’s because it’s an **SVG**!

### 安装

```
# Install asciinema via: https://asciinema.org/docs/installation
npm install -g svg-term-cli
```

### Examples

- 基于上传到 asciinema 上的 [asciicast](https://asciinema.org/a/113643) 生成 `parrot.svg`

```
svg-term --cast 113643
svg-term --cast 113643 --out examples/parrot.svg
svg-term --cast=113643 --out examples/parrot.svg --window
svg-term --cast 113643 --out examples/parrot.svg --window --no-cursor --from=4500
```

- 基于预生成的 asciicast v1/v2 数据创建 SVG

```
# asciicast v1
cat rec.json | svg-term
# asciicase v2
cat first.cast | svg-term --out first_window.svg --window
cat first.cast | svg-term --out first_window_frame_profile_hw2.svg --window --profile=Seti --term=iterm2 --height=40 --width=100
```

- [commitlint](https://github.com/marionebl/commitlint) 提供的功能展示图。基于 `svg-term-cli` 生成

```
cat docs/assets/commitlint.json | svg-term --out docs/assets/commitlint.svg --frame --profile=Seti --height=20 --width=80
```

- harbor 使用

```
cat 03_statistics.cast | svg-term --out 03_statistics.svg --window --profile=Seti --term=iterm2 --height=35 --width=120
```

## 组合技

- 基于 asciinema 录制 asciicast 文件


```
asciinema rec 00_harbor_setup.cast
```

- 基于 svg-term 将 asciicast 转换成 SVG

```
cat 00_harbor_setup.cast | svg-term --out 00_harbor_setup.svg --window --profile=Seti --term=iterm2 --height=35 --width=120
```


## 其他

- [Making Your Code Beautiful](https://hackernoon.com/presenting-your-code-beautifully-fdbab9e6fb68) -- (*)
- [sindresorhus/gifski-app](https://github.com/sindresorhus/gifski-app) -- Convert videos to high-quality GIFs on your Mac.
- [giphycapture](https://giphy.com/apps/giphycapture) -- GIPHY CAPTURE is the best way to create GIFs on your Mac.
- [kap](https://getkap.co/) -- An open-source screen recorder built with web technology.

