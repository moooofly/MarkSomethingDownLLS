# 利用 Chrome 原生工具进行网页长截图

> Ref: https://sspai.com/post/42193

Chrome 开发者工具中其实自带了截图命令，需要首先确保 Chrome 已升级至 59 或更高版本；

快捷键如下：

- 召唤出调试界面

```
⌘Command + ⌥Option + I
```

- 自动截取整个网页内容并保存至本地

```
⌘Command + ⇧Shift + P
```

之后输入命令 "Capture full size screenshot"（只输前几个字母就能找到），敲下回车；


- 截取手机版网页长图

只需要按下如下命令以便模拟移动设备；在顶部的工具栏中，你可以选择要模拟的设备和分辨率等设置；

```
⌘Command + ⇧Shift + M
```

- 准确截取网页的某一部分

```
⌘Command + ⇧Shift + C
```

选中想要的部分后，再运行 Capture node screenshot 命令，一张完美的选区截图就诞生了。


- 截取当前网页的可视部分

```
⌘Command + ⇧Shift + P
```

之后输入命令 "Capture screenshot"
