# Zsh 和 iTerm2 的配置问题

常见的组合推荐：**agnoster 主题 + Solarized 配色方案 + Powerline 字体**


## 切换 zsh 主题

编辑 `~/.zshrc`

```
ZSH_THEME="robbyrussell"
改为
ZSH_THEME="agnoster"
```

重新打开 iTerm2 ；

> 注意：此时会有奇怪字符出现，因为尚未安装好相应的 Powerline-patched 字体；


根据 [agnoster/agnoster-zsh-theme](https://github.com/agnoster/agnoster-zsh-theme) 中的说明可知，该主题是专门为使用 **Solarized 配色方案** + **Git** + 
**Unicode-compatible 字体和终端**的人设计的；

对于 Mac 用户来说，一般会建议使用 iTerm 2 + Solarized Dark + 能够正确显示 `echo "\ue0b0 \u00b1 \ue0a0 \u27a6 \u2718 \u26a1 \u2699"` 命令输出结果的字体；


## [为 iTerm2 安装一个 Solarized 配色方案](https://github.com/altercation/solarized/tree/master/iterm2-colors-solarized)

特点：颜色柔和，看着不累；

问题：模拟 fish shell 提示功能的 zsh-autosuggestions 的提示的颜色被遮挡，看不到提示信息；

```
$ mkdir tmpfiles
$ cd tmpfiles
$ wget https://raw.githubusercontent.com/altercation/solarized/master/iterm2-colors-solarized/Solarized%20Dark.itermcolors
$ wget https://raw.githubusercontent.com/altercation/solarized/master/iterm2-colors-solarized/Solarized%20Light.itermcolors
$ open .

# 双击对应文件就可以安装到 iterm2 中
# 或
# 打开 iTerm2 并通过 iTerm2 -> Preferences -> Profiles -> Colors -> Color Presets... -> Import 导入上述下载好的 Solarized 配色方案
#
# 安装后，需要选中对应主题才会生效
```

## 安装 Powerline 字体

> 解决 agnoster 主题中出现的奇怪字符；

几点说明：

- 特殊的字符都是对字体 patch 后才有的；
- 别人已经 patch 好的字体：[powerline/fonts](https://github.com/powerline/fonts) （推荐使用）；
- 自行对字体进行 patch 的工具：[powerline/fontpatcher](https://github.com/powerline/fontpatcher) （别自找麻烦了）；


字体安装（参考：https://github.com/powerline/fonts）

```
$ cd tmpfiles
$ git clone https://github.com/powerline/fonts.git --depth=1
$ cd fonts/
$ ./install.sh
# 字体文件会被安装到 $HOME/Library/Fonts 目录下面（mac 上）或 $HOME/.local/share/fonts 目录下面（linux 上）
$ cd ..
# 成功安装后，清理之前下载的 github 项目文件
$ rm -rf fonts
# 安装好的字体自动会出现在 iTerm2 -> Preferences -> Profiles -> Text -> Font -> Change Font 下的列表中，自行选择相应的字体即可 
```


## 调整 iTerm2 设置

- 选择配色方案；
- 选择 Font 和 Non-ASCII Font ；
- 把 Text Rendering 里的 "Draw bold text in bright colors" 给去掉（否则可能导致 ls 时列表无着色）；


----------

## 我的选择

> (*) 为推荐项


ZSH Themes

- robbyrussell (*) (default)
- agnoster (*)
- miloshadzic
- pygmalion



Color Presets

- Dark Background
- Solarized Light (*)
- Tango Dark (*)

Font

- Ayuthaya
- Courier New
- D2Coding for Powerline (*)
- Menlo
- Meslo LG S DZ for Powerline
- Monaco (*)
- Roboto Mono Light for Powerline


Non-ASCII Font

- D2Coding for Powerline (*)
- Meslo LG S DZ for Powerline
- Roboto Mono Light for Powerline (*)



----------


## [powerline/powerline](https://github.com/powerline/powerline)


Powerline is a **statusline plugin** for `vim`, and provides statuslines and prompts for several other applications, including `zsh`, `bash`, `tmux`, IPython, Awesome and Qtile. 

- 安装配置文档：https://powerline.readthedocs.io/en/latest/
- 配套使用的 pre-patched 字体：[`powerline/fonts`](https://github.com/powerline/fonts)
- Powerline 衍生品：[`vim-airline`](https://github.com/vim-airline/vim-airline)

## [powerline/fonts](https://github.com/powerline/fonts)

Patched fonts for Powerline users.

快速安装：

```
# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```

详细安装文档：

- for **linux**: https://powerline.readthedocs.io/en/latest/installation/linux.html#fonts-installation
- for **OS X**: https://powerline.readthedocs.io/en/latest/installation/osx.html

iTerm2 users need to set both the **Regular font** and the **Non-ASCII Font** in "`iTerm > Preferences > Profiles > Text`" to use a patched font (per this issue).

> In some distributions, Terminess Powerline is ignored by default and must be explicitly allowed. A fontconfig file is provided which enables it. Copy this file from the fontconfig directory to your home folder under `~/.config/fontconfig/conf.d` (create it if it doesn't exist) and re-run `fc-cache -vf`.


## [vim-airline/vim-airline](https://github.com/vim-airline/vim-airline)


lean & mean status/tabline for vim that's light as air



