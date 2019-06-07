---
title: Git快速上手
date: 2019-04-18 14:54:47
categories: git学习
tags:
- git
- 版本控制
- 常用指令
---
## 前言

git作为一个先进的版本管理工具，已经被广泛应用在大量项目中。近来发现了一个非常不错的git学习网站，虽然比较基础，但是可视化的界面能够帮助新人快速理解git每项指令的功能，同时也可以一定程度上的查漏补缺。
网站地址：[https://learngitbranching.js.org/](https://learngitbranching.js.org/)
而本文也记录了一些常用的git指令和使用技巧

## 常用git指令

### 新建仓库

在当前目录初始化一个git仓库
`git init` 
新建一个目录，初始化一个git仓库
`git init [projectName]` 
用于从远程仓库进行克隆，一般可以选择通过https或者ssh方式
`git clone [url]` 

### 配置

对于刚安装git的新人，一般需要配置邮箱和用户名，建议使用全局方式配置
`git conifg [--global] user.name "[username]"`
`git conifg [--global] user.email "[email]"`
可以查看当前的git配置
`git config --list` 
可以直接编辑git配置文件，之前通过命令行配置的在此也可以看到
`git config -e [--global]` 

### 增加、删除文件

这里需要简单介绍一下git中工作区和暂存区的概念
- 工作区可以简单理解为你在当前仓库的种种改动，git可以检测到但是并未将之准备为下次提交的内容。需要用户将之添加到暂存区。
- 暂存区可以简单理解为，下次执行提交中会被提交上去的文件。
- 整个工作流为： 你修改了某个文件 -> 该文件变为工作区文件 -> 你添加该文件进入暂存区 -> 提交暂存区文件，该文件被提交

添加指定文件到暂存区
`git add [file1] [file2] ...`
添加指定目录到暂存区（包含该目录下所有文件）
`git add [dir]`
添加当前目录下所有文件到暂存区
`git add .`
删除工作区文件，并将“删除”这个操作放入暂存区
`git rm [file1] [file2] ...`
停止追踪文件，但是该文件会保留在工作区，类似gitignore的作用
`git rm --cached [file]`
改名文件，并将“改名”这个操作放入暂存区
`git mv [origin-name] [target-name]`

### 代码提交

把暂存区内容提交到仓库， 最常用的提交指令
`git commit -m "message"`
提交暂存区中的指定文件到仓库
`git commit [file1] [file2] ... -m "message"`
直接将工作区自从上次commit之后的变化，提交到仓库（跳过暂存区）
`git commit -a`
使用一次新的commit，替代上一次提交,常用于简单修复
如果代码没有任何新变化，则用来改写上一次commit的提交信息
`git commit --amend -m [message]`

### 分支操作

git中的分支是一个非常强大的功能，新建、删除、切换分支速度极快，可以多多使用

列出所有本地分支
`git branch`
列出所有远程分支
`git branch -r`
列出所有本地和远程分支
`git branch -a`
新建一个分支，并且留在当前分支
`git branch [branch-name]`
新建一个分支，并切换到新的分支上
`git checkout -b [branch-name]`
从某一个commit记录为起点，新建一个分支；其中commit中填入commit的hash或者tag（如果有标签）（下同）
`git branch [branch-name] [commit] `
新建一个分支，并于远程的一个分支建立追踪关系
`git branch --track [branch-name] [remote-branch]`
切换分支
`git checkout [branch-name]`
合并指定分支到当前分支
`git merge [branch-name]`
合并指定分支到当前分支，并生成线性的记录
`git rebase [branch-name]`
交互式的rebase
`git rebase [branch] -i`
选择某一次提交（任意分支上的），合并到当前分支
`git cherry-pick [commit]`
删除分支
`git branch -d [branch-name]`
删除远程分支
`git branch -dr [origin/branch]`
`git push origin --delete [branch-name]`

### 标签

标签可以用来给某一次提交添加一个可以追踪的标记，该标记不受分支影响，不会变化，可以在任何情况下被追踪。对于某一次重大提交，常常可以用标签予以标记（如某一次版本发布）

列出所有tag
`git tag`
在当前的commit上新建一个标签
`git tag [tag-name]`
给指定的commit上新建一个标签
`git tag [tag-name] [commit]`
删除本地的一个标签
`git tag -d [tag-name]`
删除远程的一个标签
`git push origin :refs/tags/[tag-name]`
查看某个标签对应的提交信息
`git show [tag-name]`
提交指定tag, remote指远程仓库的名字，一般为origin
`git push [remote] [tag]`
提交所有tag
`git push [remote] --tags`
以某个标签指定的commit为基点，新建一个分支
`git branch [branch-name] [tag-name]`

### 查看信息

显示有变更的文件
`git status`
显示当前分支的版本历史
`git log`
显示commit历史，以及每次commit发生变化的文件
`git log --stat`
显示指定文件的每一次改动
`git log -p [file]`
显示指定文件是什么时间被什么人修改的
`git blame [file]`
显示暂存区与工作区的差异
`git diff`
显示暂存区与上一次commit之间的差异 （可指定文件）
`git diff --cached [file]`
显示工作区与当前分支最新commit之间的差异
`git diff HEAD`
显示你今天写了多少行代码
`git diff --shortstat "@{0 day ago}"`
显示当前分支最近的几次提交记录（常用来进行恢复）
`git reflog`

### 远程同步

下载远程仓库的所有变动
`git fetch [remote]`
显示所有远程仓库
`git remote -v`
显示某个远程仓库的信息
`git remote show [remote]`
新增一个远程仓库，并命名
`git remote add [name] [url]`
拉取远程仓库的变化，并与本地分支合并
`git pull [remote] [branch]`
上传本地分支到远程仓库
`git push [remote] [branch]`
强行推送当前分支到远程仓库，即使有冲突
`git push [remote] --force`
推送所有分支到远程仓库
`git push [remote] --all`

### 撤销

恢复暂存区的指定文件到工作区
`git checkout [file]`
恢复某个commit的指定文件到暂存区与工作区
`git checkout [commit] [file]`
恢复暂存区的所有文件到工作区
`git checkout .`
重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
`git reset [file]`
重置工作区与暂存区，与上一次commit保持一致
`git reset --hard`
重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
`git reset [commit]`
重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
`git reset --hard [commit]`
新建一个commit，用来撤销指定commit
后者的所有变化都将被前者抵消，并且应用到当前分支
常用来对远程仓库进行恢复
`git revert [commit]`
暂时将未提交的变化移除，稍后再移入
`git stash`
`git stash pop`

关于git reset指令，其实有 --soft --hard --mixed三种参数，默认为 --mixed参数。
具体详细用法可以参考这篇文章 [git reset详解](https://segmentfault.com/a/1190000009658888)


### HEAD移动

HEAD在git中是一个非常重要的概念，因此在这里把这部分单独列出来。
HEAD是git中用来标记当前位置的一个指针。
形象的说法就是：你现在在哪，HEAD就指向哪，因为HEAD，git才知道你在哪。

一般情况下，HEAD指向当前分支（上最近的提交），但是在有些时候，我们可以让HEAD指向某一次具体的提交，这也叫做分离HEAD。比如创建分支时，如果不指定commit，那么会在当前HEAD的位置创建分支。

移动HEAD的方法是使用checkout指令，指定一个commid的hash值进行绝对定位
`git checkout [commit-id]`
我们也可以使用相对定位，以当前HEAD或分支名等可以追踪位置的标记为基准。 ^代表当前位置的前一个提交
`git checkout HEAD^`
`git checkout master^`
我们也可以用 ~[number] 来一次移动多次提交
`git checkout HEAD~3`
`git checkout master~5`

关于HEAD的更多用法可以进一步去搜集资料

## 后记

以上仅仅为git 入门常用的一些指令，熟练之后可以应对一般git的使用场景。
在这里依然十分推荐[https://learngitbranching.js.org/](https://learngitbranching.js.org/)进行实际操作一次，相信对git的使用有很大帮助。

