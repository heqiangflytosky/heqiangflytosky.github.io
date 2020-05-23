---
title: Git 常用命令
categories: 开发工具
comments: true
keywords: Git
tags: [Git, 开发工具]
description: 列举一些常用的Git命令
date: 2015-3-7 10:00:00
---

## git log

 - `git log -p`：显示每一次的提交的差异。
 - `git log -p -n` （n为次数，整数）：指定了显示差异的次数，用 -2 则仅显示最近的两次更新.
 - `git log --stat`：显示简要的增改行数统计，后面跟次数比如 -2 则仅显示最近的两次更新。
 - `git log --author=寒江蓑笠`：显示某个作者的提交记录。
 - `git log --committer=寒江蓑笠`：显示某个提交者的提交记录。
 - `git log --since, --after`：仅显示指定时间之后的提交。
 - `git log --until, --before`：仅显示指定时间之前的提交。
 - `git log --pretty=oneline`：`--pretty` 选项可以指定使用完全不同于默认格式的方式展示提交历史。比如用 `oneline` 将 每个提交 放在一行显示，这在提交数很大时非常有用。另外还有 `short`，`full` 和 `fuller` 可以用。 `format` 可以定制要显示的记录格式，这样的输出便于后期编程提取分析。
 - `git log --shortstat`：只显示 --stat 中最后的行数修改添加移除统计。
 - `git log --name-only`：仅在提交信息后显示已修改的文件清单。
 - `git log --name-status`：显示新增、修改、删除的文件清单。
 - `git log --abbrev-commit`：仅显示 SHA-1 的前几个字符，而非所有的 40 个字符。
 - `git log --relative-date`：使用较短的相对时间显示（比如，"2 weeks ago"）。
 - `git log --graph`：显示 ASCII 图形表示的分支合并历史。

## git checkout

 - `git checkout -- **(file)`：还原对file的修改
 - `git checkout -b master remotes/master`：在本地建立一个与服务器对应的分支并切换过去

## git branch

 - `git branch test`：创建一个test的本地分支
 - `git branch -d`：删除本地分支
 - `git branch -D`：强行删除本地分支
 - `git branch --set-upstream-to=origin/V3.3 V3.3`：将本地分支和远程分支关联

## git stash

 - `git stash`：把当前修改保存到暂存区
 - `git stash pop`：把暂存区的内容恢复到本地
 - `git stash list`：显示stash 列表
 - `git stash apply stash@{1}`：取出指定版本号栈中的内容，栈里面的记录仍然保存
 - `git stash clear`：清楚stash列表
 - `git stash save "Test"` 为当前的入栈使用指定的说明Test
 - `git stash show` 查看最近的缓存的文件列表
 - `git stash show stash@{0}` 查看名为stash{0}缓存的文件列表
 - `git stash show -p stash@{0}`  查看名为stash{0}缓存的文件差异
 - `git stash drop stash@{0}` 丢弃名为stash{0}缓存

## git config

先说一下`--local`、`--global`和`--system`三个参数的区别：

 1. `--local`的作用域最小，仅对当前的git项目的配置
 2. `--global`的作用域是针对当前用户
 3. `--system`的作用域最大，是针对整个计算机系统

再来说一下他们的优先级，如果在三个级别都有配置，那么优先级为：
`--local` > `--global` > `--system`

 - 查看配置：`git config --local -l`
 - 配置用户名和邮箱：`git config --local user.name "xx"`  `git config --global user.email "xx"`

## git reset

 - `git reset --mixed`：此为默认方式，不带任何参数的`git reset`，即时这种方式，它回退到某个版本，只保留源码，回退commit和index信息
 - `git reset --soft`：回退到某个版本，只回退了commit的信息，不会恢复到index file一级。如果还要提交，直接commit即可
 - `git reset --hard`：彻底回退到某个版本，本地的源码也会变为上一个版本的内容
 - `git reset HEAD^` ：默认的`reset`方式，指向HEAD之前最近的一次commit，
 - `git reset --hard <commit>`：自从<commit>以来在working directory中的任何改变都被丢弃，并把HEAD指向<commit>
 - `git reset --hard HEAD~2`：丢弃最近两次的提交

### 删除远程仓库上最近的提交：
先`git reset --hard HEAD~2`，然后`git push -f`

## git rm

 - `git rm test`:删除test文件的跟踪，并且删除本地文件
 - `git rm --cached test`:删除test文件的跟踪，但是保留在本地

## git commit

 - `git commit --amend` ：对最后一次的 commit进行修改
 - `git commit --amend -m "Test"` 对最后一次的 commit进行修改，并修改提交信息
 - `git commit -s`：添加 Signed-off-by: 信息

## git rebase
假设`mywork`是以远程分支`origin`创建的本地分支。
`rebase`个人理解就是重新在base分支提交的意思。
`git rebase`用于把一个分支的修改合并到当前分支。
```
$ git checkout mywork
$ git rebase origin
```
这些命令会把你的"mywork"分支里的每个提交(commit)取消掉，并且把它们临时 保存为补丁(patch)(这些补丁放到`.git/rebase`目录中),然后把`mywork`分支更新 为最新的`origin`分支，最后把保存的这些补丁应用到`mywork`分支上。相当于在`mywork`上更新了它的 “base” `origin`分支。
这个要注意和`git merge`的区别。

### 基本操作
`git rebase -i` 进入编辑模式，可以执行`edit`，`squash`等操作。
 - `pick`：`git`会应用这个补丁，以同样的提交信息（commit message）保存提交。
 - `squash`：`git`会把这个提交和前一个提交合并成为一个新的提交。
 - `edit`：`git`会完成同样的工作，但是在对`edit`的提交进行操作之前，它会返回到命令行让你对提交进行修正，或者对提交内容进行修改。可以在这个提交进行`amend`，分割提交等等修改操作。
 - 丢弃：`rebase`的最后一个作用是丢弃提交。如果把一行删除而不是指定`pick`、`squash`和`edit`中的任何一个，git会从历史中移除该提交。

### 合并提交
`git rebase -i HEAD~10`：将前10个提交合并为一个。
执行以后进入编辑模式，按提示操作，将第二个及以后的`pick`修改为`squash`或者`s`，然后再按提示操作保存退出。

## git pull
 - `git pull --rebase`：表示把你的本地当前分支里的每个提交(commit)取消掉，并且把它们临时 保存为补丁(patch)(这些补丁放到`.git/rebase`目录中),然后把本地当前分支更新 为最新的`origin`分支，最后把保存的这些补丁应用到本地当前分支上。

## git push

 - `git push origin --delete <branch_name>`：删除远程分支
 - `git push origin <branch_name>`：将本地分支推到远程，然后再调用 `git branch --set-upstream-to=origin/V3.3 V3.3` 将本地分支和远程分支关联

## patch
`git format-patch -n`：为前面的n次提交生成一个patch
应用patch：
先检查patch文件：`# git apply --stat newpatch.patch`
检查能否应用成功：`# git apply --check  newpatch.patch`
打补丁：`# git am --signoff < newpatch.patch`
如果有冲突将会合并不成功
可以下面方法：
http://blog.chinaunix.net/uid-27714502-id-3479018.html
先手工 `apply patch`中没有冲突的部分：
`git apply --reject 0001-test.patch`
这样，就把没有冲突的文件先合并了，剩下有冲突的作了标记。
可以看输出，同时，还会产生一个` *.rej`文件，里面也是上面这段因为冲突无法合并的代码片断。
根据apply的输出提示以及`mm/sparse.c.rej`文件中的描述，手动修正代码。
改好之后，用 `git add` 把文件添加到缓冲区，同时也要把其他没有冲突合并成功了的文件也加进来，因为在作 `apply` 操作的时候他们也发生了变化。
`git am --resolved`
然后合并提交即可。

## git remote
 - git remote: 不带参数，列出已经存在的远程分支
 - git remote -v | --verbose: 列出详细信息，在每一个名字后面列出其远程url
 - git remote add <name> <url>: 添加远程仓库
 - git remote show <origin>：查看本地和远程分支的状态，origin为远程分支的名称
 - git remote rename <原名字> <新名字>：重命名远程库
 - git remote remove <name>：删除添加的远程库

## git merge

### git merge -s ours

cherry pick 与 git merge

## git fetch

## 解 gerrit 冲突

<!--  https://www.cnblogs.com/xiaoerlang/p/3810206.html -->

### 解决本地有提交记录的远程冲突

```
git fetch origin
git rebase origin/develop
//修改冲突文件
...
git add .
git rebase --continue
git push origin HEAD:refs/changes/<changeID>
//不会产生新的changes记录，将原changes记录重新review提交即可
git pull
```

### 解决本地没有提交记录的远程冲突

```
// 下面的 url 指的是 pull request 对应的地址，比如： ssh://name@test.com:29418/app/ProjectName refs/changes/40/1000/3
git fetch <url> && git checkout FETCH_HEAD
git checkout -b new_branch_name        // 新建一个本地分支用来解冲突
git fetch origin                       //远程仓库的名称可以用  git remote -v 来查看
git rebase origin/develop              //rebase 你冲突的代码分支，develop为当前提交代码所在的分支
修改冲突文件
git rebase --continue
git push origin new_branch_name:refs/for/develop
git checkout develop
git branch -D new_branch_name
//不会产生新的 changes 记录，将原 changes 记录重新 review 提交即可，这时在原冲突机器上直接pull会本地冲突，需要
git reset --hard HEAD^
//否则会出现cannot do a partial commit during a merge.最后更新下代码
git pull
```
