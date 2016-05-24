title: Linux netstat解析
tags:
  - netstat
  - linux
  - server
categories:
  - linux
date: 2016-05-19 11:04:40
---

### 常见参数
```
-a 显示所有选项，默认不显示LISTEN相关
-t 显示TCP相关
-u 显示UDP相关
-n 不显示别名
-l 仅列出LISTEN状态的服务

-p 显示相关的程序名 （需要root权限才能显示http、ftp等服务）
-r 显示路由表
-e 显示扩展信息，如uid等
-s 按协议进行统计
-c 每隔固定时间，执行该命令
```

LISTEN和LISTENING的状态只有用-a或者-l才能看到

<!--more-->

### 列出所有连接
```
# netstat -a | more
 Active Internet connections (servers and established)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:30037         *:*                     LISTEN
 udp        0      0 *:bootpc                *:*
 
Active UNIX domain sockets (servers and established)
 Proto RefCnt Flags       Type       State         I-Node   Path
 unix  2      [ ACC ]     STREAM     LISTENING     6135     /tmp/.X11-unix/X0
 unix  2      [ ACC ]     STREAM     LISTENING     5140     /var/run/acpid.socket
```

### 列出TCP端口
```
netstat -at
 Active Internet connections (servers and established)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:30037         *:*                     LISTEN
 tcp        0      0 localhost:ipp           *:*                     LISTEN
 tcp        0      0 *:smtp                  *:*                     LISTEN
 tcp6       0      0 localhost:ipp           [::]:*                  LISTEN
```

### 列出所有处于监听状态的 Sockets
```
netstat -l
 Active Internet connections (only servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State
 tcp        0      0 localhost:ipp           *:*                     LISTEN
 tcp6       0      0 localhost:ipp           [::]:*                  LISTEN
 udp        0      0 *:49119                 *:*
```

### 在 netstat 输出中显示 PID 和进程名称 netstat -p
```
# netstat -pt
 Active Internet connections (w/o servers)
 Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
 tcp        1      0 ramesh-laptop.loc:47212 192.168.185.75:www        CLOSE_WAIT  2109/firefox
 tcp        0      0 ramesh-laptop.loc:52750 lax:www ESTABLISHED 2109/firefox
```

### 在 netstat 输出中不显示主机，端口和用户名 (host, port or user)
```
netstat -an
```


### 找出程序运行的端口
并不是所有的进程都能找到，没有权限的会不显示，使用 root 权限查看所有的信息。
```
# netstat -ap | grep ssh
 tcp        1      0 dev-db:ssh           101.174.100.22:39213        CLOSE_WAIT  -
 tcp        1      0 dev-db:ssh           101.174.100.22:57643        CLOSE_WAIT  -
```
找出运行在指定端口的进程
```
# netstat -an | grep ':80'
```

**注意：**
1. 使用 -p 选项时，netstat 必须运行在 root 权限之下，不然它就不能得到运行在 root 权限下的进程名，而很多服务包括 http 和 ftp 都运行在 root 权限之下。

2. 相比进程名和进程号而言，查看进程的拥有者会更有用。使用 -ep 选项可以同时查看进程名和用户名。

3. 假如你将 -n 和 -e 选项一起使用，User 列的属性就是用户的 ID 号，而不是用户名。

### 显示网络接口列表
```
# netstat -i
# netstat -ie  (显示详细信息)
```

### 注意
**Linux下和OSX下的netstat没有联系，基本用法也不相同！！！！**

在OSX下建议改用lsof
```
lsof -i -P | grep -i "3000"
```

### 原文参考
http://www.cnblogs.com/ggjucheng/archive/2012/01/08/2316661.html
https://linux.cn/article-2434-1.html

