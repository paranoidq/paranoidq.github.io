title: Git stash使用
tags: [git]
categories: [tools]
---

在配置alias的时候，偶然敲错了一个gsta的命令，用`alias | grep gsta`查看发现是git stash。网上查阅了资料之后发现git stash还蛮有用的，故记录一下。


### 用途
修改一个分支，然后想切换另一个分支去工作。git提醒你，需要提交当前分支的工作，否则之后就无法回到这个工作点了。但是我不想提交这个没有稳定的代码，怎么办？
git stash就是帮你做这个工作的。

git stash 可以获取你工作目录的中间状态（即你修改过的tracked文件和暂存的变更），并将它保存到一个未完结变更的堆栈中，随时可以重新应用。

<!--more-->

### 储藏
```
git stash
Saved working directory and index state \
 "WIP on master: 049d078 added the index file"
 HEAD is now at 049d078 added the index file
(To restore them type "git stash apply")
```
然后你的工作目录就是clean的了，可以切换到其他分支工作了
```
git status

# On branch master
nothing to commit, working directory clean
```

注意：只有当前branch的状态是clean的情况下，才能切换分支。

### 查看
```
git stash list
stash@{0}: WIP on master: 049d078 added the index file
stash@{1}: WIP on master: c264051 Revert "added file_size"
stash@{2}: WIP on master: 21d80a5 added number to log
```

### 恢复
```
# 恢复到最近的一个stash
git stash apply  

# 恢复到特定的stash
git stash apply stash@{2}
```

对文件的变更被重新应用，但是被暂存的文件没有重新被暂存。
想那样的话，你必须在运行`git stash apply`命令时带上一个`--index`的选项来告诉命令重新应用被暂存的变更。如果你是这么做的，你应该已经回到你原来的位置。
```
git stash apply --index
```


### 移除
apply 选项只是尝试应用储藏的工作，储藏的内容仍然在堆栈上，要移除它，需要drop
```
git stash drop stash@{0}
```
也可以运用`git stash pop` 来重新应用储藏的工作，并同时立刻移除

### 恢复储藏并建立到新的分支

恢复储藏可能出现的问题: 如果你储藏了一些工作，暂时不去理会，然后继续在你储藏工作的分支上工作。然后你希望重新apply stash上的工作，这时候可能会与你修改的文件发生归并冲突。

好的方案是，不在原来非branch上apply stash，而创建一个新的branch来恢复stash。如果需要合并，再去处理冲突的问题。
```
git stash branch stash_branch
```



### 参考
[git stash](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E5%82%A8%E8%97%8F%EF%BC%88Stashing%EF%BC%89)



