title: How to find process and kill it
tags:
  - linux
  - server
categories:
  - linux
date: 2016-05-18 21:38:10
---

### 知道端口号


1. 查看端口使用情况：
```
netstat -tln | grep 8080
```
<!--more-->

2. 查看端口属于哪个程序：
```
lsof -i :8080
```

3. kill
```
kill -9 [pid]
```

### 不知道端口号

1. 查找进程
```
ps aux | grep java
```

2. kill
```
kill -9 [pid]
```
