title: Linux服务器性能调优常用工具及实例
date: 2016-08-01 00:08:17
tags: [linux, server]
categories: linux
---

### 查看进程是否存在: ps

ps工具标识进程的5种状态码: 
- D 不可中断 uninterruptible sleep (usually IO) 
- R 运行 runnable (on run queue) 
- S 中断 sleeping 
- T 停止 traced or stopped 
- Z 僵死 a defunct (”zombie”) process 

#### 显示所有进程信息
```
[root@localhost test6]# ps -A
PID TTY          TIME CMD
1 ?        00:00:00 init
2 ?        00:00:01 migration/0
3 ?        00:00:00 ksoftirqd/0
4 ?        00:00:01 migration/1
5 ?        00:00:00 ksoftirqd/1
6 ?        00:29:57 events/0
7 ?        00:00:00 events/1
8 ?        00:00:00 khelper
49 ?        00:00:00 kthread
54 ?        00:00:00 kblockd/0
55 ?        00:00:00 kblockd/1
56 ?        00:00:00 kacpid
217 ?        00:00:00 cqueue/0
...
```

#### 显示指定用户
```
[root@localhost test6]# ps -u root
PID TTY          TIME CMD
1 ?        00:00:00 init
2 ?        00:00:01 migration/0
3 ?        00:00:00 ksoftirqd/0
4 ?        00:00:01 migration/1
5 ?        00:00:00 ksoftirqd/1
6 ?        00:29:57 events/0
7 ?        00:00:00 events/1
8 ?        00:00:00 khelper
49 ?        00:00:00 kthread
54 ?        00:00:00 kblockd/0
55 ?        00:00:00 kblockd/1
56 ?        00:00:00 kacpid
...
```
#### ps 与grep 组合使用，查找特定进程 (常用)
```
[root@localhost test6]# ps -ef|grep ssh
root      2720     1  0 Nov02 ?        00:00:00 /usr/sbin/sshd
root     17394  2720  0 14:58 ?        00:00:00 sshd: root@pts/0
root     17465 17398  0 15:57 pts/0    00:00:00 grep ssh
```

#### 列出目前所有的正在内存中的程序 (常用）
```
[root@localhost test6]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  10368   676 ?        Ss   Nov02   0:00 init [3]
root         2  0.0  0.0      0     0 ?        S<   Nov02   0:01 [migration/0]
root         3  0.0  0.0      0     0 ?        SN   Nov02   0:00 [ksoftirqd/0]
root         4  0.0  0.0      0     0 ?        S<   Nov02   0:01 [migration/1]
root         5  0.0  0.0      0     0 ?        SN   Nov02   0:00 [ksoftirqd/1]
root         6  0.0  0.0      0     0 ?        S<   Nov02  29:57 [events/0]
root         7  0.0  0.0      0     0 ?        S<   Nov02   0:00 [events/1]
root         8  0.0  0.0      0     0 ?        S<   Nov02   0:00 [khelper]
root        49  0.0  0.0      0     0 ?        S<   Nov02   0:00 [kthread]
root        54  0.0  0.0      0     0 ?        S<   Nov02   0:00 [kblockd/0]
root        55  0.0  0.0      0     0 ?        S<   Nov02   0:00 [kblockd/1]
root        56  0.0  0.0      0     0 ?        S<   Nov02   0:00 [kacpid]
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

#### 进程与端口号
分两种情况：
- 已知进程ID，想查看进程占用哪个端口号
- 已知端口号，想查看端口被哪个线程占用

#### 情况1

#### 情况2

