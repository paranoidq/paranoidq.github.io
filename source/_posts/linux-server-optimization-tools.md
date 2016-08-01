title: Linux服务器性能调优常用工具及实例
date: 2016-08-02 00:37:17
tags: [linux, server]
categories: linux
---

### 查看进程情况: ps

#### 显示所有进程信息
```
$ ps -A
PID TTY      TIME   CMD
1 ?        00:00:00 init
2 ?        00:00:01 migration/0
3 ?        00:00:00 ksoftirqd/0
4 ?        00:00:01 migration/1
5 ?        00:00:00 ksoftirqd/1
6 ?        00:29:57 events/0
...
```

<!--more-->

#### 显示指定用户
```
$ ps -u root
PID TTY      TIME   CMD
1 ?        00:00:00 init
2 ?        00:00:01 migration/0
3 ?        00:00:00 ksoftirqd/0
4 ?        00:00:01 migration/1
5 ?        00:00:00 ksoftirqd/1
6 ?        00:29:57 events/0
7 ?        00:00:00 events/1
...
```
#### ps 与grep 组合使用，查找特定进程 (常用)
```
$ ps -ef|grep ssh
root      2720     1  0 Nov02 ?        00:00:00 /usr/sbin/sshd
root     17394  2720  0 14:58 ?        00:00:00 sshd: root@pts/0
root     17465 17398  0 15:57 pts/0    00:00:00 grep ssh
```

#### 列出目前所有的正在内存中的程序 (常用）
```
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  10368   676 ?        Ss   Nov02   0:00 init [3]
root         2  0.0  0.0      0     0 ?        S<   Nov02   0:01 [migration/0]
root         3  0.0  0.0      0     0 ?        SN   Nov02   0:00 [ksoftirqd/0]
root         4  0.0  0.0      0     0 ?        S<   Nov02   0:01 [migration/1]
root         5  0.0  0.0      0     0 ?        SN   Nov02   0:00 [ksoftirqd/1]
```
输出含义：
```
USER：该 process 属于那个使用者账号的
PID ：该 process 的号码
%CPU：该 process 使用掉的 CPU 资源百分比
%MEM：该 process 所占用的物理内存百分比
VSZ ：该 process 使用掉的虚拟内存量 (Kbytes)
RSS ：该 process 占用的固定的内存量 (Kbytes)
TTY ：该 process 是在那个终端机上面运作，若与终端机无关，则显示 ?，另外， tty1-tty6 是本机上面的登入者程序，若为 pts/0 等等的，则表示为由网络连接进主机的程序。
STAT：该程序目前的状态，主要的状态有
R ：该程序目前正在运作，或者是可被运作
S ：该程序目前正在睡眠当中 (可说是 idle 状态)，但可被某些讯号 (signal) 唤醒。
T ：该程序目前正在侦测或者是停止了
Z ：该程序应该已经终止，但是其父程序却无法正常的终止他，造成 zombie (疆尸) 程序的状态
START：该 process 被触发启动的时间
TIME ：该 process 实际使用 CPU 运作的时间
COMMAND：该程序的实际指令
```

#### ps -ef与 ps aux的区别
`ps aux`最初用到Unix Style中，而`ps -ef`被用在System V Style中，两者输出略有不同。现在的大部分Linux系统都是可以同时使用这两种方式的。

`ps aux`中与`ps -ef`不同的列有：
```
USER      //用户名
%CPU      //进程占用的CPU百分比
%MEM      //占用内存的百分比
VSZ       //该进程使用的虚拟內存量（KB）
RSS       //该进程占用的固定內存量（KB）（驻留中页的数量）
STAT      //进程的状态
START     //该进程被触发启动时间
TIME      //该进程实际使用CPU运行的时间
```
其中`STAT`状态为的常见字符有：
```
D      //无法中断的休眠状态（通常 IO 的进程）；
R      //正在运行可中在队列中可过行的；
S      //处于休眠状态；
T      //停止或被追踪；
W      //进入内存交换 （从内核2.6开始无效）；
X      //死掉的进程 （基本很少见）；
Z      //僵尸进程；
<      //优先级高的进程
N      //优先级较低的进程
L      //有些页被锁进内存；
s      //进程的领导者（在它之下有子进程）；
l      //多线程，克隆线程（使用 CLONE_THREAD, 类似 NPTL pthreads）；
+      //位于后台的进程组；
```

### 查看端口情况 netstat

#### 列出所有连接
```
$ netstat -a
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 enlightened:domain      *:*                     LISTEN     
tcp        0      0 localhost:ipp           *:*                     LISTEN     
tcp        0      0 enlightened.local:54750 li240-5.members.li:http ESTABLISHED
tcp        0      0 enlightened.local:49980 del01s07-in-f14.1:https ESTABLISHED
tcp6       0      0 ip6-localhost:ipp       [::]:*                  LISTEN 
...
```

#### 列出tcp/udp连接 `-u`和`-t`
```
$ netstat -at
$ netstat -au
```

#### 禁用反向域名解析，加快查询速度 `-n`
没有必要知道主机名，就使用 -n 选项禁用域名解析功能
```
$ netstat -ant
```

#### 只列出监听中的端口 `-l`
```
$ netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN
```
不要使用`-a`，否则linux会列出所有端口，而不只是监听（LISTEN）端口

#### 只列出active端口
```
$ netstat -atnp | grep ESTA
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 192.168.1.2:49156       173.255.230.5:80        ESTABLISHED 1691/chrome     
tcp        0      0 192.168.1.2:33324       173.194.36.117:443      ESTABLISHED 1691/chrome
```
active 状态的套接字连接用 "ESTABLISHED" 字段表示

#### 列出进程名，进程号和用户ID `-p`
```
~$ sudo netstat -nlpt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      1144/dnsmasq    
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      661/cupsd       
tcp6       0      0 ::1:631                 :::*                    LISTEN      661/cupsd
```
必须要root权限才能显示！如果没有root需要查看端口对应的进程，请参考`lsof`
`-ep`选项可以同时查看进程名和用户名

#### 实战

1. 查看服务是否运行
```
$ sudo netstat -aple | grep ntp
udp        0      0 enlightened.local:ntp   *:*     root       17430       1789/ntpd       
udp        0      0 localhost:ntp           *:*     root       17429       1789/ntpd       
udp        0      0 *:ntp                   *:*     root       17422       1789/ntpd       
udp6       0      0 fe80::216:36ff:fef8:ntp [::]    root       17432       1789/ntpd 
```
<!--more-->

2. 查看端口号的占用情况
```
$ netstat -an | grep 12000
```

3. 结合`watch`监控active状态的连接
```
$ watch -d -n0 "netstat -atnp | grep ESTA"
```


#### 附：watch命令
watch可以帮助使用者监测一个命令的运行结果，避免重复手动运行。watch命令会周期执行
参数：

- -n 时间间隔，缺省值为2s
- -d 高亮显示变化区域
- -t 关闭watch命令在顶部的时间间隔

实例：
```
每隔一秒高亮显示http链接数的变化情况
$ watch -n 1 -d 'pstree|grep http'

10秒一次输出系统的平均负载
$ watch -n 10 'cat /proc/loadavg'
```






### 查看使用CPU\MEM最多的进程






### 参考文献
[ps](http://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/ps.html)
[netstat](https://linux.cn/article-2434-1.html)
[watch](http://www.cnblogs.com/peida/archive/2012/12/31/2840241.html)

