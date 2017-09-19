title: 如何用jstack命令分析java应用的性能问题
date: 2017-01-08 23:09:53
tags: [java, jstack]
categories: [java]
---

### 前言
在进行WCG项目优化的时候，性能测试的效果一直不理想。于是乎进行性能分析，除了查找内存、CPU等基本的参数之外，还从业务角度入手，查找了日志打印出来的时间，按照报文的时间进行分析对比。其中一项就是用jstack分析线程的情况。这里记录一下整个分析的过程以及收获。

### 整体过程
1. 首先用jps查看java应用的进程号
2. 然后根据进程号查找java中各个线程的情况，命令如下：
```
top -Hp [pid]
```
3. 然后会看到哪个线程占用了较高的CPU，记住这个线程号。
4. 用jstack命令dump下来应用的线程栈，命令如下：
```
jstack -l [pid] > jstack.log
```
 -l选项表示长列表. 打印关于锁的附加信息,例如属于java.util.concurrent的ownable synchronizers列表

5. 将第3步中的线程号转换为16进制，然后在dump下来的线程栈中查找到对应的线程调用栈，然后对其进行分析和判断，就有可能找到问题的所在，例如发生了死锁、大量线程等待某一个condition等。从而能够分析出问题的原因。

6. 如果需要的话，还可以统计一些线程状态的数量，这里需要用到shell的一些命令：
```
cat thread.tmp | grep "java.lang.Thread.State" | sort | uniq -c
```

### Java线程的状态
![java-thread-states](http://images.cnblogs.com/cnblogs_com/zhengyun_ustc/255879/o_clipboard%20-%20%E5%89%AF%E6%9C%AC039.png)

1. **NEW** 状态是指线程刚创建, 尚未启动

2. **RUNNABLE** 状态是线程正在正常运行中, 当然可能会有某种耗时计算/IO等待的操作/CPU时间片切换等, 这个状态下发生的等待一般是其他系统资源, 而不是锁, Sleep等。**所以一般IO操作的线程会进入这个状态**。

3. **BLOCKED**  这个状态下, 是在多个线程有同步操作的场景, 比如正在等待另一个线程的synchronized 块的执行释放, 或者可重入的 synchronized块里别人调用wait() 方法, 也就是这里是线程在等待进入临界区

4. **WAITING**  这个状态下是指线程拥有了某个锁之后, 调用了他的wait方法, 等待其他线程/锁拥有者调用 notify / notifyAll 一遍该线程可以继续下一步操作, 这里要区分 BLOCKED 和 WATING 的区别, **一个是在临界点外面等待进入, 一个是在临界点里面wait等待别人notify, 线程调用了join方法 join了另外的线程的时候, 也会进入WAITING状态, 等待被他join的线程执行结束**

5. **TIMED\_WAITING**  这个状态就是有限的(时间限制)的WAITING, 一般出现在调用wait(long), join(long)等情况下, 另外一个线程sleep后, 也会进入TIMED\_WAITING状态

6. **TERMINATED** 这个状态下表示 该线程的run方法已经执行完毕了, 基本上就等于死亡了(当时如果线程被持久持有, 可能不会被回收)

### 线程dump文件的分析
分析如下的一段线程栈：
```
"RMI TCP Connection(267865)-172.16.5.25" daemon prio=10 tid=0x00007fd508371000 nid=0x55ae waiting for monitor entry [0x00007fd4f8684000]
   java.lang.Thread.State: BLOCKED (on object monitor)
at org.apache.log4j.Category.callAppenders(Category.java:201)
- waiting to lock <0x00000000acf4d0c0> (a org.apache.log4j.Logger)
at org.apache.log4j.Category.forcedLog(Category.java:388)
at org.apache.log4j.Category.log(Category.java:853)
at org.apache.commons.logging.impl.Log4JLogger.warn(Log4JLogger.java:234)
at com.tuan.core.common.lang.cache.remote.SpyMemcachedClient.get(SpyMemcachedClient.java:110)
```

1）线程状态是 Blocked，阻塞状态。说明线程等待资源超时！
2）“ waiting to lock <0x00000000acf4d0c0>”指，线程在等待给这个 0x00000000acf4d0c0 地址上锁（英文可描述为：trying to obtain  0x00000000acf4d0c0 lock）。
3）在 dump 日志里查找字符串 0x00000000acf4d0c0，发现有大量线程都在等待给这个地址上锁。如果能在日志里找到谁获得了这个锁（如locked < 0x00000000acf4d0c0 >），就可以顺藤摸瓜了。
4）“waiting for monitor entry”说明此线程通过 synchronized(obj) {……} 申请进入了临界区，从而进入了上图中的“Entry Set”队列，但该 obj 对应的 monitor 被其他线程拥有，所以本线程在 Entry Set 队列中等待。
5）第一行里，"RMI TCP Connection(267865)-172.16.5.25"是 Thread Name 。tid指Java Thread id。nid指native线程的id。prio是线程优先级。[0x00007fd4f8684000]是线程栈起始地址。


当java系统运行慢的时候, 我们想到的应该先找到性能的瓶颈, 而jstack等工具, 通过jvm当前的stack可以看到当前整个vm所有线程的状态, 当我们看到一个线程状态经常处于
WAITING 或者 BLOCKED的时候, 要小心了, 他可能在等待资源经常没有得到释放(当然, 线程池的调度用的也是各种队列各种锁, 要区分一下, 比如下图)
![pool-waiting-demo](http://www.jiacheo.org/wp-content/uploads/2013/04/6db341bbd7680bbc2e6ae37a66329397.png)
这是个经典的并发包里面的线程池, 其调度队列用的是LinkedBlockingQueue, 执行take的时候会block住, 等待下一个任务进入队列中, 然后进入执行, **这种理论上不是系统的性能瓶颈, 找瓶颈一般先找自己的代码stack,再去排查那些开源的组件/JDK的问题**

### 如何排查
1. 发现有线程进入BLOCK, 而且持续好久, 这说明性能瓶颈存在于synchronized块中, 因为他一直block住, 进不去, 说明另一个线程一直没有处理好, 也就这个synchronized块中处理速度比较慢, 然后再深入查看. 
 **当然也有可能同时block的线程太多, 排队太久造成.**
 
2. 发现有线程进入WAITING, 而且持续好久, 说明**性能瓶颈存在于触发notify的那段逻辑**. **当然还有就是同时WAITING的线程过多, 老是等不到释放**.
 
3. 线程进入TIME_WAITING 状态且持续好久的, 跟2的排查方式一样.


### 参考资料
1. [http://www.jointforce.com/jfperiodical/article/3729](http://www.jointforce.com/jfperiodical/article/3729) 
2. [http://www.jiacheo.org/blog/338](http://www.jiacheo.org/blog/338)
3. [http://blog.csdn.net/yaowj2/article/details/48735247](http://blog.csdn.net/yaowj2/article/details/48735247)



