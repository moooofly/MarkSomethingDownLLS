# git 命令之 rebase/reset/reflog 使用


## 主要命令

```
git rebase -i HEAD~4
git reset --hard <hash>
git reflog
```

## 实验过程

### 当前 git log 状态


```
git log --graph --all --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(bold white)— %an%C(reset)%C(bold yellow)%d%C(reset)' --abbrev-commit --date=relative
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%201.png)

### 调整最后 4 次 commit 的 message 内容

```
git rebase -i HEAD~4
```

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%202.png)

差别对比：

- 直接以 `:q` 退出时，得到

```
Successfully rebased and updated refs/heads/master.
```

并会在 `git reflog` 输出中增加

```
53775be (HEAD -> master) HEAD@{0}: rebase -i (finish): returning to refs/heads/master
53775be (HEAD -> master) HEAD@{1}: rebase -i (start): checkout HEAD~4
```

- 删除所有未被 `#` 注释的内容后退出，得到

```
Nothing to do
```

不会在 `git reflog` 输出中增加任何内容；

结论：后一种方式才是推荐使用的正确的 abort 当前 rebase 的方式；

### 尝试直接修改

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%203.png)

### 触发 reword 命令

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%204.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%205.png)

### 触发 edit 命令

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%206.png)

### 手动执行 git commit --amend 进行 commit 修改

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%207.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%208.png)

### 手动执行 git rebase --continue 恢复 rebase 执行

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%209.png)

### 观察对比 rebase 前后 hash 值的变化

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2010.png)

### 从 reflog 中查看整个 rebase 的执行过程

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2011.png)

### 再次进行 rebase 调整

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2012.png)

### 第一次新调整

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2013.png)

### 调整顺序后触发冲突

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2014.png)

### 操作尝试：直接 abort 当前 rebase 操作

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2015.png)

> 可以看出状态直接回到了未进行 rebase 前；

### 操作尝试：使用 skip 命令跳过冲突 commit

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2017.png)

> 可以看到能够完成其他 rebase 命令动作；

### 执行 skip 命令后的其他 rebase 命令操作

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2016.png)

### 从 git log 中可以看到 commit 被成功合并

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2018.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2019.png)

### 从目标文件内容可以看出被 skip 掉的 commit 对应的内容已丢失

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2020.png)

### 从 reflog 中查看再次 rebase 的执行过程

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2021.png)


### 在 rebase 过程中删除指定 commit

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2022.png)

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2023.png)

### 删除后的效果

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2024.png)

### 直接通过 reset 命令实现删除 commit 的效果

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/git%20rebase_reset_reflog%20usage%20-%2025.png)

差别对比：

- 基于 rebase 方式进行 commit 删除，粒度更加细致，能够在删除的过程中同时完成其他调整；
- 基于 reset 方式进行 commit 删除，适用于简单的回退操作；




