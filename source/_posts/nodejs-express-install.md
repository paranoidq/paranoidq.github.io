title: node.js安装express框架时出现command not found问题
date: 2016-06-07 20:23:45
tags: [nodejs, express, 碎片]
categories: nodejs
---

在安装nodejs的web框架express时遇到的问题及解决方案。
<!--more-->

安装时在文件夹下输入：
```
npm install -g express
```

但是无法使用express命令，出现`express: command not found`。原因在于在express4.0中，cli需要单独安装才能使用，cli功能被包含在`express-generator` package中。

因此需要如下操作：
```
npm install -g express-generator
```

参考: [http://stackoverflow.com/questions/23002448/express-command-not-found](http://stackoverflow.com/questions/23002448/express-command-not-found)


