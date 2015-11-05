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

#### scp

security copy，多用于拷贝文件

基于SSH协议和SCP协议

```
本地拷贝到远程 scp /etc/eva.log user@sysB:/home/user
远程拷贝到本地 scp -p user@sysB:/home/uesr/eva.log /etc （保持属性不变）
拷贝远程目录到本地 scp -r user@sysB:/home/user /home/user/tmp
```

免密码登陆

```
$ ssh-keygen -t rsa 

#将公钥id_rsa.pub拷贝到目标机器上的对应位置
$ scp /Users/paranoidq/.ssh/id_rsa.pub paranoidq@192.168.235.131:/home/paranoidq/.ssh/authorized_keys 
```



#### 区别

wget是个专职的下载利器，简单，专一，极致；而curl可以下载，但是长项不在于下载，而在于模拟提交web数据，POST/GET请求，调试网页，等等。在下载上，也各有所长，；而curl支持URL中加入变量，因此可以批量下载。

- curl是libcurl这个库支持的，wget是一个纯粹的命令行命令
- wget可以递归，支持断点
- curl支持更多的协议，而wget斤支持http1.0
- 
- 
- 





###参考
[1] [http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2856896.html](http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2856896.html)