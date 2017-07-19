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

 - `git log -p`：显示每一次的提交的差异，用 -2 则仅显示最近的两次更新。
 - `git log -p -x` （x为次数，整数）：指定了显示差异的次数
 - `git log --stat`：显示简要的增改行数统计，用 -2 则仅显示最近的两次更新。
 - `git log --author=寒江蓑笠`：显示某个作者的提交记录

## git checkout

 - `git checkout -- **(file)`：还原对file的修改
 - `git checkout -b master remotes/master`：在本地建立一个与服务器对应的分支并切换过去

## git branch

 - `git branch test`：创建一个test的本地分支
 - `git branch -d`：删除本地分支
 - `git branch -D`：强行删除本地分支

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
 - git remote add: 添加远程仓库

## git merge


