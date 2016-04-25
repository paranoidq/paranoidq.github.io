title: Linux Server Commands
tags: [Linux, Server]
categories: 
- Linux
- 命令
---


### ssh远程登录

服务器需要开启远程服务

``` 
# 针对fedora
service sshd status
# 通用检测方法：
ps -e | grep ssh

# fedora
service sshd start
# ubuntu
/etc/init.d/ssh start[/stop/restart]

```

<!--more-->

客户端登陆：

```
ssh paranoidq@[ip]
```

免密码登陆

```
$ ssh-keygen -t rsa 

# 将公钥id_rsa.pub拷贝到目标机器上的对应位置
$ scp /Users/paranoidq/.ssh/id_rsa.pub paranoidq@192.168.235.131:/home/paranoidq/.ssh/authorized_keys 
```

### 查看Linux服务器状况

#### CPU

主要通过/proc文件夹下面的文件来获取

```
# 获取CPU个数
cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l

# 获取CPU core个数
cat /proc/cpuinfo | grep "cpu cores" | uniq

# 获取逻辑CPU个数 = CPU个数 * cores
cat /proc/cupinfo | grep "processor" | wc -l

```


#### Memory

```
# 查看内存
free -m # 以MB的方式显示

total   used    free    shared      buffers     cached
3949    1397    2551    0           268         917
-/+ buffers/cached      211        3737
Swap:   8001       0        8001

```

- total：总内存
- used：已使用
- free：空闲
- shared: 多个进程共享内存
- buffers：buffer cache 和 cached page cache: 磁盘缓存
- -buffers/cached：(已用内存) = used - buffers - cached
- +buffers/cached：(`程序`可用内存) = free + buffers + cached

注：

- buffer 和 cached 对操作系统来讲属于被使用的部分，但是程序可以挪用他们，因此程序可用内存需要算上这两个
- Linux系统内存是拿来用的，不满时不会使用到swap分区（这点和Windows不同，无论何时都会使用硬盘交换文件来读）。因此`如果swap没有被使用，那么就不用担心内存小。如果经常看到swap被使用，就需要考虑增加物理内存了——这是判断linux内存是否够用的标准`


#### Disk

```
# 查看硬盘分区
fdisk -l

# 查看fs的硬盘占用情况
df -h

# 查看硬盘IO的性能
iostat -d -x -k 1 10 ???

# 查看目录大小
du -sh /data

找出占用空间最多的文件或目录前十
du -cks * | sort -rn | head -n 10


```


#### Linux Load

```
# 动态反应系统负载状况
top

# 过去1min、5min、15min内进程队列的平均进程数量
uptime

# 可以查看系统当前有哪些用户，占用哪些终端
w

# 
vmstat

```

- uptime输出的三个load average值一般不能大于系统逻辑CPU的个数，如果长期大于，则表明CPU过于繁忙，系统负载较高


#### 整体性能

vmstat [采样时间间隔/s] [采样次数]

```
vmstat 1 4
```

- procs
    + r: 等待运行的进程数
    + b: 处于非中断睡眠状态的进程数
- memory: 
    + swpd: 虚拟内存使用情况（KB）
    + free: 空闲内存
    + buff: 被用来作为缓存的内存
- swap：
    + si: 从磁盘交换到内存的页数（KB/s）
    + so: 从内存交换到磁盘的页数（-）
- io:
    + bi: 发送到块设备的块数（块/s）
    + bo: 从块设备接收的块数
- system:
    + in: 每秒的中断数，包括时钟中断
    + cs: 每秒的上下文切换次数
- cpu: (按CPU使用的百分比时间显示)
    + us: cpu用户模式使用时间
    + sy: cpu系统模式使用时间
    + id: cpu闲置时间

标准状况：r < 5 和 b ·= 0

如果r经常大于4，id经常少于50, | usr%+sy% > 85% ，则表示系统的性能比较糟糕。


#### 其他查看命令

```
# 查看内核
uname -a

# 简化版
uname -r

# 查看发行版信息

lsb_release -a

# 查看载入的模块(用lsmod查看lvs模块是否已经载入)
lsmod | grep ip_vs


# 查看系统是否64位
# 1. 查找 /lib64目录
ls -lF / | grep /$ | grep lib64

# 2. 通过file判断文件是32或64位
file /sbin/init

```


### 查看网络配置

#### Linux下基本网络配置
```
# 修改hostname
vi /etc/sysconfig/network

# 修改hosts
vi /etc/

# DNS域名
vi /etc/resolv.conf

```

#### 服务器网络连接状况

1.ifconfig

```
ifconfig -a

# 只显示eth0的网络配置
ifconfig eth0

# 只显示ehto0的Ip地址
# awk语句以空格和:为分隔符，并且打印第四列
ifconfig eth0 | grep "inet addr" | awk -F[:" "] + '{print $4}'


```

2.ping 

使用ICMP协议中的ECHO_REQUEST数据报强制从特定主机返回相应，用于检测网络中某个主机是否活动或发生故障

```
# 发生5个数据包
ping -c 5 www.163.com

```

3.netstat

显示网络连接、路由表和网络接口等信息。常用参数：

- -a: 显示所有套接字状态
- -n: 打印实际地址，而不是显示对地址的解释或显示主机网络名之类的符号
- -r: 打印路由选择表

```
netstat -an | grep -v unix

netstat -rn
```

4.traceroute

跟踪网络数据包的路由途径，默认包大小40B。第一条是本机client的网关地址。

```
tracerout www.163.com
```

5.nslookup / dig

查询机器的IP地址和对应的域名，其中nslookup是基于交互的，dig则直接在参数中附带网址

```
nslookup
> www.163.com
> ...
> ...
> exit


dig www.163.com

# 从根服务器开始追踪域名的解析过程
dig www.163.com + trace

```


6.finger

查询用户信息，例如用户名、主目录、登录时间等。类似w


7.`lsof`

非常有用的命令，list open files。查看当前系统打开了哪些文件。

`利用lsof查看打开文件列表，对于系统检测和排错非常有帮助。`

```
# 查看某一个端口被哪些程序占用
lsof -i :22
```


8.Sockstat

查看打开的socket情况，只适用于FreeBSD和OpenBSD，CentOS没有此命令





### 查看服务器进程

#### ps

确定有哪些进程正在运行和运行状态、占用资源等

选项：参见man

常用：

```
# 显示所有
ps aux

# 结合grep，精确定位需要的进程号
# -v ：反向选择，亦即显示出没有 '搜寻字符串' 内容的那一行
ps aux | grep -v grep | grep nginx

```


#### top

动态查看进程信息

还可以通过交互命令完成功能：

- P: 根据CPU使用多少排序
- T: 根据时间、累计时间排序
- M: 根据使用内存大小排序
- q: 退出top命令
- m: 切换显示内存信息
- t：切换显示进程和CPU状态信息
- c: 切换显示命令名称和完整命令行
- W: 将当前的设置写入~/.toprc文件，这是写top配置文件的推荐方法

最好使用的`htop`，使用更方便快捷！！！

#### pgrep

查找当前运行的进程Id

```
pgrep nginx
```


#### kill 和 killall

向linux内核发送操作系统信号和某个程序的进程标识号，然后系统内核就可以对进程标识号指定的进程进行kill操作。

```
# 强行终止
kill -9 [pid]

# 根据名称删除程序下所有进程
killall nginx


```




### 参考
[1] 构建高可用Linux服务器
