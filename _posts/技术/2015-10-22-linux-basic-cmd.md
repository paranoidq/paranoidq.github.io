---
layout: post
title: Linux basic comands
category: 技术
tags: Linux
keywords: 
description: 
---
### ls

```
 ls
 ls -a # 列出所有，包括隐藏文件
 ls -l # 列出文件的属性
```

### 查找命令

#### find

#### locate

#### which

#### whereis



###grep家族

- grep
- egrep: extended grep, 支持更多的re元字符
- fgrep: fixed grep, 把所有字母都看做单词，正则表达式中的元字符也表示回原有字面含义

通过-G -E -F 命令行选项来选择试用哪一种grep


###文件传输

#### wget

#### curl

#### 区别

wget是个专职的下载利器，简单，专一，极致；而curl可以下载，但是长项不在于下载，而在于模拟提交web数据，POST/GET请求，调试网页，等等。在下载上，也各有所长，wget可以递归，支持断点；而curl支持URL中加入变量，因此可以批量下载。





###参考
[1] [http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2856896.html](http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2856896.html)