title: Git 分支管理与实践（hexo博客源文件管理）
date: 2015-12-18 14:51:34
tags: git
categories: tools
---

### 查看

查看本地分支
`git branch`

查看包括远程分支
`git branch -a`

查看本地分支和远程分支的push情况
`git log <local-branch> ^origin/<remote-branch>`   # 可以查看本地有远程没有的提交。
`git log <remote-branch> ^<local-branch>`       # 可以查看远程有本地没有的提交。


### 创建

创建本地分支
`git checkout -b dev`


### 关联
目的：避免每次都需要制定push的具体分支，可以关联后在本地分支下直接push即可

两种方案：
1. 在本地branch第一次push的时候就指定：
    `git push -u origin <remote-branch>`
2.  以后指定：
    `git branch --set-upstream <local-branch> origin/<remote-branch>`
    或
    `git push --set-upstream origin <remote-branch>`
你的repo下的git/config会多出类似这样的配置：
```
[branch "src"]
remote = origin
merge = refs/heads/src
```

### 推送
`git push origin <local-branch> `  # 推送本地分支到对应的远程分支

`git push` # 如果指定了远程关联，可以直接push

两种配置： (注意，这个配置是写在~/.gitconfig中的，对所有的repo生效)
`git config --global push.default simple`(更好, 只push当前分支到你使用git pull拉取的远程代码)
`git config --global push.default matching` （如果没有指定具体分支，会push所有名字对应的分支）


### 删除
删除本地分支
`git branch -d dev (用-D强行删除)`

删除远程分支
`git push origin --delete dev`


### 实例： 管理hexo的src分支
说明：hexo的deployer本身在部署的时候只会生成static文件，并上传到github的master分支，而hexo的一些source和\_config.yaml等配置文件则只在本地。因此需要将这些文件也管理到git中去，方便备份和多终端同步。

基本思路是在本地利用src分支，然后上传源文件到src分支，并push到远程的src分支，即可管理。

master分支由于是hexo的页面展示部分，所以其实是不能与origin/master保持同步的，也千万不能push，否则结果就是源文件覆盖了hexo-deployer push到master分支的静态文件，从而访问的时候404了。

```
$ cd <repo>
$ git init (optional)
$ git checkout -b src
$ git add .
$ git commit -m "first commit for src branch"
$ git remote add origin git@github.com:<username>:<username>.github.io.git (optional)
$ git push origin src
```

1. optional部分，可能由于之前开始建立repo的时候已经做过了，所以不一定要在分支的过程中做了
2. checkout的时候，需要保证master分支全部commit。（这里其实我做的不够完善，一开始应该是整个本地的网站不要init，让master分支全部被hexo-deployer接管。然后在创建分支的时候，才init。这样可以保证本地只有一个src的分支需要我手动管理。
3.  如果你手贱之前已经建立了master分支，那么有两个办法：
 - 忽略与origin/master不同步的本地master分支
 - 删除本地的master分支 `git branch -D master`

**另外一台电脑上如何使用：**
安装node环境和hexo-cli环境
```
$ brew update 
$ brew install node
$ npm install hexo-cli -g
```

不要执行hexo init了，而是clone远程的src仓库
```
git clone -b src git@github.com:<username>:<username>.github.io.git
$ npm install //根据package.json来下载依赖包
```
然后就可以继续写博客了，整个过程，手动管理的只有src，master分支本地需要，由hexo-deployer负责push。

参考文献[1]中还提到了用git submodule解决第三方主题的同步问题，很不错。


### 参考：
1. [如何管理hexo的源文件](http://devtian.me/2015/03/17/blog-sync-solution/)
2. [为何以及如何删除master分支](https://gitcafe.com/GitCafe/Help/wiki/%E5%A6%82%E4%BD%95%E5%88%A0%E9%99%A4-Master-%E5%88%86%E6%94%AF?locale=zh-CN)
 

