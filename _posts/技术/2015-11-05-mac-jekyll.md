---
layout: post
title: Mac下配置jekyll和markdown环境
category: 技术
tags: Mac Jekyll
keywords: 
description: 
---

### 配置jekyll环境

1.安装rvm
    
```
ruby -v # 检测ruby环境，如果是>1.9.2以上的版本，直接跳过，否则需要自行安装rvm + ruby环境
```


2.安装 jekyll


更换被墙的rubygems源为淘宝的gem源

```
$ gem sources -l
$ gem sources --remove https://rubygems.org/
$ gem sources -a http://ruby.taobao.org/
```

安装jekyll

```
gem install jekyll
```

安装redcarpet包（如果你用的是jekyll的其他渲染包，如rediscount之类的，则同理安装即可）

```
gem install redcarpet
```

安装依赖包时，提示权限不足(`permission denined`)，用`sudo`解决。

3.启动jekyll

```
cd /Users/paranoidq/paranoidq.github.io/
jekyll server
```
然后就可以访问localhost:4000看到你的gitpages了



### 配置sublime的markdown环境

1.使用的`ctrl+ /` ` 调出console，然后输入如下命令

```
import urllib,os,hashlib; h = 'eb2297e1a458f27d836c04bb0cbaf282' + 'd0e7a3098092775ccb37ca9d6b2e4b7d'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

2.安装markdown Extended， Markdown preview, Markdown Editing和Monokai Extended主题

- shift+command+P调出command palette
- 寻找包管理工具package control: install packages
- 输入需要安装的包
- 重启sublime



### 参考

[1] [http://www.jianshu.com/p/07064eb79740](http://www.jianshu.com/p/07064eb79740)

[2] [http://frank19900731.github.io/blog/2015/04/13/zai-sublime-zhong-pei-zhi-markdown-huan-jing/](http://frank19900731.github.io/blog/2015/04/13/zai-sublime-zhong-pei-zhi-markdown-huan-jing/)













