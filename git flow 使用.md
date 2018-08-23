# git flow 使用

![](https://raw.githubusercontent.com/moooofly/ImageCache/master/Pictures/A%20successful%20Git%20branching%20model.png)

## git-flow 备忘清单

详见《[git-flow 备忘清单](https://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)》

## git flow 理念

采用 Git-Flow 的理念进行项目的开发和管理：

- **永远不在你本地的 `master` 分支上进行开发**，remote 上的 `master` 分支每一次更新都要保证是 release ready 的；
- 一般对于**新版本的开发都是在 `develop` 分支以及从它分出的 `feature` 分支上进行开发**；
- 对于已经发布的版本，如果有**紧急的问题和功能需要添加**，需要从 `master` 分支分出的 `hotfix` 分支上进行开发；
- 有时可能会有一些 `feature` 分支会存在比较长时间，比如一个新版本的代码不能很快被合到主分支去，这个分支其实是作为一些大功能的 base 分支。这时可以直接从这个 `featrue` 分支 checkout 出新的分支（分支名可以不按 gitflow 的命名来，但是一定要有意义！）
- 你可以直接在已存在的 projects 中使用 `git flow init` 命令（创建出相应的 git-flow 工作环境）；
-  如果你不再想要使用 git-flow ，则只要不再使用 git-flow 命令就可以了，没有任何需要额外改变或删除的；
- 在 git-flow 中 `develop` 分支作为 default branch 使用，因此绝大多数工作都发生在该分支上，而 `master` 分支只被用作 production-ready 代码的跟踪；
- If you’re not doing versioned releases, Vincent’s git workflow and the git-flow library might not be a right fit for you. However, if you work on a project that’s semantically versioned, like a Rubygem or a versioned API, git-flow will give you a couple of simple commands that will do a lot of work under the hood, making working on features, pushing new releases and hotfixing bugs a lot easier. 

## 最佳实践

有人整理出一套最佳实践惯例，即 [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) ；简单来说，他將 branch 分成两个主要分支，三种支援性分支：

### 主要分支

- `Master` 分支：永远处于 production-ready 状态的分支；
- `Develop` 分支：最新的、为下次发布做准备、处于开发状态的分支；

### 支援性分支

- `Feature` 分支：开发新功能时要基于 feature 分支，feature 分支都从 develop 分支出來，完成后将 feature 分支 merge 回 develop 分支；
- `Release` 分支：准备要 release 的版本，只修 bugs 。从 develop 分支出來，完成后 merge 回 master 分支和 develop 分支；
- `Hotfix` 分支：等不及 release 版本就必须马上修 master 分支、赶着上线的情況。会直接从 master 分支出來，完成后再将 hotfix 分支 merge 回 master 分支和 develop 分支；


### 关于 feature 分支的合并

如果是开发时间比较久的 `feature` 分支，很可能会因为

1. 不定时的 merge `develop` 分支上的内容，以同步 `develop` 分支上的最新内容
2. 实验性质的修改
3. 需求的变更

等因素，而让这个 `feature` 分支的 commit 记录变得脏脏的，这时候我们会用下面的方式来进行 merge 动作：

1. **先对 `feature` 分支做 `git rebase develop`**。会很苦，但是弄完会很有成就感，整个分支 commit history 会变得很干净。请学习 `git bebase -i` 交互模式命令，可以让你拿掉（`fixup` 和 `drop`）、合并（`squash`）或修改（`edit` 和 `reword`）一些 commit 信息，你也可以 `rebase` 多次直到满意为止。
2. 在从 develop 分支做 `git merge feature/some_awesome_feature --no-ff` 时，`--no-ff` 的意思是会强制留一个 `merge commit log` 记录，这可以让 commit tree 上清楚展示发生了 merge 动作（因为我们刚做了 `rebase` ，而 git 预设的合并模式是 fast-forward，所以如果不加 `--no-ff` 是不会有 merge commit 的）；**这个 merge commit 的另一个额外方便之处是，如果想要 reset/revert 整个 branch 只要 reset/revert 这个 commit 就可以了**。
3. 如果此 feature 分支有 remote 分支，则要先通过 `git push origin :feature/some_awesome_feature` 砍掉 remote 分支，再执行 `git push origin develop` （这是因为 rebase 一个已经 push 出去的 repository ，然后又把修改的 history push 出去，会造成超级大灾难啊~）


### git flow feature finish <xxx> 内部动作

从内部实现上来说，git-flow 使用 `git merge --no-ff feature/authentication` 命令以确保在 feature 分支被移除前，不会丢掉任何历史信息；

> 实际使用中发现：若在 feature branch 中只有一个 commit 则没有使用 `--no-ff` ，故在 git log 中不会看到 merge 信息；


## git-flow 和 git-flow-completion 安装

### Mac

`git-flow` 安装：

```
brew install git-flow
```

`git-flow-completion` 安装：

将文件 `.git-flow-completion.zsh` 拷贝到某个位置（例如 `~/.git-flow-completion.zsh`）并将如下内容保存到 `.zshrc` 中；

```
source ~/.git-flow-completion.zsh
```

### ubuntu

`git-flow` 安装：

```
$ apt-get install git-flow
```

`git-flow-completion` 安装：

- [Install `git-completion`](https://github.com/bobthecow/git-flow-completion/wiki/Install-Bash-git-completion) (git-flow completion requires git-completion to work).
- Install `git-flow-completion.bash`. 

将 git-flow-completion.bash 拷贝到某个位置（例如 `~/git-flow-completion.bash`），并将如下内容刚保存到 `.profile` 或 `.bashrc` 文件中；

```
source ~/git-flow-completion.bash
```

## 常用命令

### 初始化

```
git flow init [-d]
```

使用基本分支结构初始化一个新的 repo ；

该命令会交互式与你进行问答，以确定你所想要作为 development 和 production 使用的分支形式，以及你想要使用的命名前缀；你可以通过简单的连续按回车以接受交互问题的默认设置；

若指定 `-d` 则表明接受所有默认设置值；

### feature 分支操作

```
# To list/start/finish feature branches
git flow feature
git flow feature start <name> [<base>]
git flow feature finish <name>

# To push/pull a feature branch to the remote repository
git flow feature publish <name>
git flow feature pull <remote> <name>
```

对于 `feature` 分支来说，`<base>` 参数必须是 `develop` 分支上的某个  commit 值；

### release 分支操作

```
# To list/start/finish release branches
git flow release
git flow release start <release> [<base>]
git flow release finish <release>
```

对于 `release` 分支来说，`<base>` 参数必须是 `develop` 分支上的某个  commit 值；


### hotfix 分支操作

```
# To list/start/finish hotfix branches, use:
git flow hotfix
git flow hotfix start <release> [<base>]
git flow hotfix finish <release>
```

对于 `hotfix` 分支来说，`<base>` 参数必须是 `master` 分支上的某个  commit 值；

### support 分支操作

```
# To list/start support branches, use:
git flow support
git flow support start <release> <base>
```

对于 `support` 分支来说，`<base>` 参数必须是 `master` 分支上的某个  commit 值；


## 补充说明

> 学习曲线（建议顺序）

首先，建议反复阅读 [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) 原文，原文中将各种分支使用的前因后果交代的非常清晰，另外原文是基于 git 原生命令演示的；git flow 相关命令已经是对原文命令的抽象，方便使用而已；

其次，仔细阅读文章 [Using git-flow to automate your git branching workflow](https://jeffkreeftmeijer.com/git-flow/) 中是如何使用 git-flow 工具进行项目开发管理的，本文主要是在前文理念的基础上，讲述如何具体操作的；

最后，安装并熟悉 [nvie/gitflow](https://github.com/nvie/gitflow) 工具，能够大幅提高效率；

若想了解反对上述分支模型的观点，见[这里](http://scottchacon.com/2011/08/31/github-flow.html)；

### tag 相关

#### 基于 git flow 的操作

在初始化时，会提示是否指定 tag 前缀

```
$ git flow init
Initialized empty Git repository in ~/project/.git/
No branches exist yet. Base branches must be created now.
Branch name for production releases: [master]
Branch name for "next release" development: [develop]

How to name your supporting branch prefixes?
Feature branches? [feature/]
Release branches? [release/]
Hotfix branches? [hotfix/]
Support branches? [support/]
Version tag prefix? []
```

在使用 git flow 时，不需要手动打 tag ，而是在使用 `git flow release start|finish <a.b.c>` 命令的过程中自动生成；

> If you need **tagged** and **versioned** releases, you can use git-flow’s `release` branches to start a new branch when you’re ready to deploy a new version to production.
>
> Like everything else in git-flow, you don’t have to use `release` branches if you don’t want to. Prefer to manually `git merge --no-ff develop` into `master` without tagging? No problem. However, if you’re working on a versioned API or library, `release` branches might be really useful, and they work exactly like you’d expect:

```
$ git flow release start 0.1.0
Switched to a new branch 'release/0.1.0'

Summary of actions:
- A new branch 'release/0.1.0' was created, based on 'develop'
- You are now on branch 'release/0.1.0'

Follow-up actions:
- Bump the version number now!
- Start committing last-minute fixes in preparing your release
- When done, run:

     git flow release finish '0.1.0'
```

> Bump the version number and do everything that’s required to release your project in the `release` branch. I personally wouldn’t do any last minute fixes, but if you do, git-flow will make sure everything is correctly merged into both `master` and `develop`. Then, finish the release:

```
$ git flow release finish 0.1.0
Switched to branch 'master'
Merge made by the 'recursive' strategy.
 authentication.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 authentication.txt
Deleted branch release/0.1.0 (was 1b26f7c).

Summary of actions:
- Latest objects have been fetched from 'origin'
- Release branch has been merged into 'master'
- The release was tagged '0.1.0'
- Release branch has been back-merged into 'develop'
- Release branch 'release/0.1.0' has been deleted
```

> Boom. git-flow pulls from origin, merges the `release` branch into `master`, tags the release and back-merges everything back into `develop` before removing the `release` branch.
> 
> You’re still on `master`, so you can deploy before going back to your `develop` branch, which git-flow made sure to update with the release changes in master.

#### 基于 git 原生命令的操作

> When the state of the `release` branch is ready to become a real release, some actions need to be carried out. 
> 
> - First, the `release` branch is merged into `master` (since **every commit on master is a new release by definition**, remember). 
> - Next, that commit on master must be **tagged** for easy future reference to this historical version. 
> - Finally, the changes made on the `release` branch need to be merged back into `develop`, so that future releases also contain these bug fixes.

> The first two steps in Git:

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2
```

> The release is now done, and tagged for future reference.

> To keep the changes made in the `release` branch, we need to merge those back into `develop`, though. In Git:

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff release-1.2
Merge made by recursive.
(Summary of changes)
```

> This step may well lead to a **merge conflict** (probably even, since we have changed the version number). If so, fix it and commit.

> Now we are really done and the `release` branch may be removed, since we don’t need it anymore:

```
$ git branch -d release-1.2
Deleted branch release-1.2 (was ff452fe).
```


### hotfix 相关

#### 基于 git flow 的操作

> Because you **keep your master branch always in sync with the code that’s on production**, you’ll be able to quickly fix any issues on production.

> For example, if your assets aren’t loading on production, you’d roll back your deploy and start a `hotfix` branch:

```
$ git flow hotfix start assets
Switched to a new branch 'hotfix/assets'

Summary of actions:
- A new branch 'hotfix/assets' was created, based on 'master'
- You are now on branch 'hotfix/assets'

Follow-up actions:
- Bump the version number now!
- Start committing your hot fixes
- When done, run:

     git flow hotfix finish 'assets'
```

> `Hotfix` branches are a lot like `release` branches, except they’re based on `master` instead of `develop`. You’re automatically switched to the new `hotfix` branch so you can start fixing the issue and bumping the minor version number. When you’re done, hotfix finish:

```
$ git flow hotfix finish assets
Switched to branch 'master'
Merge made by the 'recursive' strategy.
 assets.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 assets.txt
Switched to branch 'develop'
Merge made by the 'recursive' strategy.
 assets.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 assets.txt
Deleted branch hotfix/assets (was 08edb94).

Summary of actions:
- Latest objects have been fetched from 'origin'
- Hotfix branch has been merged into 'master'
- The hotfix was tagged '0.1.1'                // 这里 0.1.1 应该改为 assets
- Hotfix branch has been back-merged into 'develop'
- Hotfix branch 'hotfix/assets' has been deleted
```

> Like when finishing a `release` branch, the `hotfix` branch gets merged into both `master` and `develop`. The release is **tagged** and the `hotfix` branch is removed.

此时需要手动推送 `hotfix` 对应的 tag ，例如

```
git push origin assets
```

#### 基于 git 原生命令的操作

> `Hotfix` branches are created from the `master` branch. For example, say version 1.2 is the current production release running live and causing troubles due to a severe bug. But changes on `develop` are yet unstable. We may then branch off a `hotfix` branch and start fixing the problem:

```
$ git checkout -b hotfix-1.2.1 master
Switched to a new branch "hotfix-1.2.1"
$ ./bump-version.sh 1.2.1
Files modified successfully, version bumped to 1.2.1.
$ git commit -a -m "Bumped version number to 1.2.1"
[hotfix-1.2.1 41e61bb] Bumped version number to 1.2.1
1 files changed, 1 insertions(+), 1 deletions(-)
```

> **Don’t forget to bump the version number after branching off!**

> Then, fix the bug and commit the fix in one or more separate commits.

```
$ git commit -m "Fixed severe production problem"
[hotfix-1.2.1 abbe5d6] Fixed severe production problem
5 files changed, 32 insertions(+), 17 deletions(-)
```

> When finished, the bugfix needs to be merged back into `master`, but also needs to be merged back into `develop`, in order to safeguard that the bugfix is included in the next release as well. This is completely similar to how `release` branches are finished.

> First, update `master` and **tag the release**.

```
$ git checkout master
Switched to branch 'master'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
$ git tag -a 1.2.1
```

> Next, include the bugfix in develop, too:

```
$ git checkout develop
Switched to branch 'develop'
$ git merge --no-ff hotfix-1.2.1
Merge made by recursive.
(Summary of changes)
```

> Finally, remove the temporary branch:

```
$ git branch -d hotfix-1.2.1
Deleted branch hotfix-1.2.1 (was abbe5d6).
```

#### 两种方式对比

- 手动操作方式中 hotfix 分支的建议命名风格为 `hotfix-*` ，而基于 git flow 的默认命名风格为 `hotfix/*` ；
- 手动操作方式中操作步骤很清楚：
    - **切 hotfix 分支 => 增加版本号并 commit => 修改 bug 并 commit**
    - **切 master 分支并 merge 分支 hotfix 上的变更 => 为当前 master 打 tag => 推送 master 分支和对应的 tag**
    - **切 develop 分支并 merge 分支 hotfix 上的变更 => 推送 develop 分支（也可以不推送，继续开发）**
    - **删除本地 hotfix 分支（可以看到整个过程中没有产生过远端 hotfix 分支）**
- 基于 git flow 的方式中上述过程没有那么明显：
    - `git flow hotfix start base_on_goreport`
        - A new branch 'hotfix/base_on_goreport' was created, based on 'master'
        - You are now on branch 'hotfix/base_on_goreport'
        - Start committing your hot fixes
        - Bump the version number now!
    - `git flow hotfix finish base_on_goreport`
        - Branches 'develop' and 'origin/develop' have diverged.
        - And local branch 'develop' is ahead of 'origin/develop'.
        - Switched to branch 'master'
        - Your branch is up-to-date with 'origin/master'.
        - Merge made by the 'recursive' strategy.
        - Switched to branch 'develop'
        - Merge made by the 'recursive' strategy.
        - Deleted branch hotfix/base_on_goreport.
    -  整体效果
        - Hotfix branch 'hotfix/base_on_goreport' has been merged into 'master'
        - The hotfix was tagged 'base_on_goreport'
        - Hotfix tag 'base_on_goreport' has been back-merged into 'develop'
        - Hotfix branch 'hotfix/base_on_goreport' has been locally deleted
        - You are now on branch 'develop'
- The one exception to the rule here is that, **when a `release` branch currently exists, the `hotfix` changes need to be merged into that `release` branch, instead of `develop`.** Back-merging the bugfix into the `release` branch will eventually result in the bugfix being merged into `develop` too, when the `release` branch is finished. 


## 视频资料

- [How to use a scalable Git branching model called git-flow](http://buildamodule.com/video/change-management-and-version-control-deploying-releases-features-and-fixes-with-git-how-to-use-a-scalable-git-branching-model-called-gitflow) (by Build a Module)
- [A short introduction to git-flow](http://vimeo.com/16018419) (by Mark Derricutt)
- [On the path with git-flow](http://codesherpas.com/screencasts/on_the_path_gitflow.mov) (by Dave Bock)


## 参考

- [nvie/gitflow](https://github.com/nvie/gitflow)
- [gitflow FAQ](https://github.com/nvie/gitflow/wiki/FAQ)
- [bobthecow/git-flow-completion](https://github.com/bobthecow/git-flow-completion)
- [A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/) - **Vincent Driessen's branching model**
- [Using git-flow to automate your git branching workflow](https://jeffkreeftmeijer.com/git-flow/) - **the best introduction to get started with git flow**
- [Git flow 開發流程](https://ihower.tw/blog/archives/5140)
