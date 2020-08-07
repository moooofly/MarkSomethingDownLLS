> 相信一定有朋友曾经好奇过，朋友圈中发的长图是怎么制作出来的，诚然方法很多，不一而足。​本文介绍一种基于 Chrome 开发者工具进行制作的办法。

---

相信很多朋友不知道，Chrome 开发者工具中其实自带了截图命令，但需要首先确保 Chrome 已升级至 59 或更高版本；

## 召唤出调试界面

mac 命令

```shell
⌘Command + ⌥Option + I
```

windows 命令

```shell
Ctrl + Shift + I
```

## 截取常规尺寸网页长图

mac 命令

```shell
⌘Command + ⇧Shift + P
```

windows 命令

```shell
Ctrl + Shift + P
```

之后输入命令 "Capture full size screenshot"（只输前几个字母就能找到），敲下回车；

## 截取手机版网页长图

只需要按下如下命令即可模拟移动设备；

mac 命令

```shell
⌘Command + ⇧Shift + M
```

windows 命令

```shell
Ctrl + Shift + M
```

> 注：已知该快捷键和某输入法“系统菜单”打开命令有冲突，需要自行关闭

图

可以看到，在顶部的工具栏中，可以设置想要模拟的设备和分辨率等内容；

## 准确截取网页的某一部分

mac 命令

```
⌘Command + ⇧Shift + C
```

> windows 上实测该命令不生效，可以在调用 `Ctrl + Shift + P` 之后，自己通过上下箭头手动选择

选中想要的部分后，再运行 Capture node screenshot 命令，一张完美的选区截图就诞生了。


