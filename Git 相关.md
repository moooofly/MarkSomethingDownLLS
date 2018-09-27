# Git 相关

## 记不牢的 Git 操作

### develop via a single branch

```
git branch -m {{branch}}
git fetch origin
git rebase origin/master -i
git push origin {{branch}}
```

### create a new branch

```
git checkout -b {{branch}}
checkout remote branch
git checkout -b {{branch}} origin/{{branch}}
```

### merge branch to master

```
git checkout master
git merge {{branch}}
```

### delete branch

```
git branch -D {{localBranch}}
git push --delete origin {{remoteBranch}}
```

### rename repo

```
git remote -v
// View existing remotes
// origin  https://github.com/user/repo.git (fetch)
// origin  https://github.com/user/repo.git (push)

git remote set-url origin https://github.com/user/repo2.git
// Change the 'origin' remote's URL
```

### add tag

```
git tag {{tag}}
git push --tags
```

### add tag for a history commit

```
// Set the HEAD to the old commit that we want to tag
git checkout {{leading 7 chars of commit}}

// temporarily set the date to the date of the HEAD commit, and add the tag
GIT_COMMITTER_DATE="$(git show --format=%aD | head -1)" git tag -a {{tag}} -m "{{commit message}}"

// set HEAD back to whatever you want it to be
git checkout master

git push --tags
```

### delete tag

```
git tag --delete {{tag}}
git push --delete origin {{tag}}
```

### gh-pages

```
http://{{group}}.github.io/{{repo}}/
```

After rename the repo, you need to push at least a commit to activate it.

### npm add owner

```
npm owner add {{name}}
```

### modify commit author

```
$ git config user.name 'yourname'
$ git config user.email 'youremail'

$ git rebase -i -p <some HEAD before all of your bad commits>

# Then mark all of your bad commits as "edit" in the rebase file

# Then, repeat the two commands below
$ git commit --amend --reset-author
$ git rebase --continue

$ git push -f
```

### Splitting a subfolder out into a new repository

https://help.github.com/articles/splitting-a-subfolder-out-into-a-new-repository/

### moving files from one git repository to another preserving history

http://gbayer.com/development/moving-files-from-one-git-repository-to-another-preserving-history/

> NOTE: `mv * <directory 1>` won't move hidden files such as `.eslintrc`, `mv * .* <directory 1>` will move hidden files and directories with dot prefix including directory `.git`




## 合作开发中的 git 使用

Ref: https://github.com/fool2fish/blog/issues/16

### 做个受人欢迎的 pr 提交者

```
#1. 从源分支进行 rebase，确保基线是最新的
$ git rebase ${sourceBranch}

#2. 使用 eslint 进行代码风格检查
$ npm run lint

#3. 运行测试，切记删除 node_modules，确保测试结果和 ci 保持一致
$ rm -rf node_modules
$ npm install
$ npm run test
# 哪怕一行最简单的修改都必须运行测试确保通过

#4. 合并 commits，使 commit 更有意义，也更便于 rebase
$ git rebase -i HEAD~${commitCount}
# 在打开的编辑器中，将除第一条外的 commit 标记从 pick 改为 squash，保存后退出
# 在打开的编辑器中，更新 commit 信息，保存后退出
# 对于已经 push 过的 commit，可运行 `$ git push --force` 强制 push
```

### 利用 ci 工具把控代码提交质量

```
#1. lint
#2. test
#3. test-cov
# 最后使用 [WIP] mr 进行人工 review
```

### 发布一个 npm 模块

```
#0. 首先确保已经安装 git-extra

#1. 修改 History
$ git changelog

#2. 修改 package.json 的版本号

#3. 发布模块
$ npm publish

#4. 打 tag
$ git release ${version}
```

### cherry-pick

```
$ git checkout ${targetBranch}
$ git cherry-pick ${commit}

# 如有冲突，解决冲突后再 add, commit 即可
```
