title: Git Fantastic Usages
date: 2015-12-09 15:20:18
tags: git
categories: tools
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

### 分支管理

创建本地分支
`git checkout -b dev`

删除本地分支
`git branch -d dev (用-D强行删除)`

查看本地分支
`git branch`

查看包括远程分支
`git branch -a`

删除远程分支
`git push origin --delete dev`



<!--more-->
