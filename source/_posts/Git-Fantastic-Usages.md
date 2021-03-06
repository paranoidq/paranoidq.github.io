title: Git Fantastic Usages
tags: git
date: 2016-01-20 12:20:08
categories: 
- git 
---

### .gitignore

1. 如何忽略所有文件除了某些文件

```
# Ignore everything
*

# But not these files...
!.gitignore
!script.pl
!template.latex
# etc...

# ...even if they are in subdirectories
!*/
```

<!--more-->

### 分支管理

创建本地分支
`git checkout -b dev`

删除本地分支
`git branch -d dev (用-D强行删除没有merge到master的分支)`

查看本地分支
`git branch`

查看包括远程分支
`git branch -a`

删除远程分支
`git push origin --delete dev`






### 撤销删除

注： checkout、reset既可以用于commit级别的，也可以用于文件级别的。revert只能用于commit级别的。

#### Case1: 修改了工作区，没有add
```
git checkout -- <file> // 用版本库的版本替换工作区的版本
```

#### Case2: 修改了工作区，并且add了，但是还没有commit
```
git reset HEAD <file> // 撤销add，回退到工作区（--mixed）
git checkout -- <file> // 回退工作区到前一次的commit状态，此时工作区clean
```

#### Case3: 已经commit了
```
git reset --hard <commit_id> // 回退commit

// 查看commit_id
$ git log

$ git reflog // 记录每一次的命令，可以反悔某一次回退
$ git reset <commit_id>  // 反悔之前的回退操作
```

注：
- --soft： staged snapshot 和 working directory 都未被改变 (建议在命令行执行后，再输入 git status 查看状态)
- --mixed： staged snapshot 被更新， working directory 未被更改。【这是默认选项】（建议同上)
- --hard： staged snapshot 和 working directory 都将回退，彻底删除没有commit的更改（危险！）

### Case4：回退远程版本库

git rest 适合在自己的版本库工作，不能用于远端版本库。可以用`git push -f`命令强行推送回退后修改的内容，但是会影响他人的更新合并。更好的建议是使用revert，它不会删除原有的commit，而是添加了一个commit，内容就是你想要反悔到的那个版本。

[https://ruby-china.org/topics/11637](https://ruby-china.org/topics/11637)
```
$ git log
commit e7c8599d29b61579ef31789309b4e691d6d3a83f
Author: fsword <li.jianye@gmail.com>
Date:   Sat Jun 8 14:27:11 2013 +0800

    补充后续计划和调整方案

commit d501310d245fe50959e8bcc1f5465bb64d67d1c8
Author: fsword <li.jianye@gmail.com>
Date:   Fri Jun 7 14:36:49 2013 +0800

    完成基本的设计

...
```
决定放弃最近提交的 e7c8599d29b61579ef31789309b4e691d6d3a83f
```
$ git revert e7c8599d29b61579ef31789309b4e691d6d3a83f
```

现在查看log，发现多了一次commit，其内容就是回到了原来的那个阶段

```
commit 7752d450a91a4c9663f5cd03f7ef3ff6d4848a12
Author: fsword <li.jianye@gmail.com>
Date:   Tue Jun 11 01:35:58 2013 +0800

    Revert "补充后续计划和调整方案"

    This reverts commit e7c8599d29b61579ef31789309b4e691d6d3a83f.

commit e7c8599d29b61579ef31789309b4e691d6d3a83f
Author: fsword <li.jianye@gmail.com>
Date:   Sat Jun 8 14:27:11 2013 +0800

    补充后续计划和调整方案

commit d501310d245fe50959e8bcc1f5465bb64d67d1c8
Author: fsword <li.jianye@gmail.com>
Date:   Fri Jun 7 14:36:49 2013 +0800

    完成基本的设计

...
```

比较一下，发现已经和提交前一样了
```
$ git diff d501310d245fe50959e8bcc1f5465bb64d67d1c8
$ 
```

#### Case5: Review前面的某一个版本提交
```
$ git checkout <commit_id>
$ git checkout HEAD~n
```
