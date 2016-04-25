title: GC专家系列-笔记
date: 2016-01-24 22:20:29
tags: [Java, JVM, GC]
categories: 
- Java
- JVM
---

1. [理解Java垃圾回收](http://segmentfault.com/a/1190000004233812)
2. [Java垃圾回收的监控](http://segmentfault.com/a/1190000004255118)
3. [GC调优](http://segmentfault.com/a/1190000004303843)
4. [Apache的MaxClients设置及其对Tomcat Full GC的影响](http://segmentfault.com/a/1190000004316645)

<!--more-->

### 基于分代的垃圾回收
#### 弱分代假设
- 大多数对象很快不可达
- 极少数对象出现旧对象持有新对象的引用

#### 基本分代
![java-gc-areas](/img/2016-1-22-1.png)

- 新生代(minor gc)
- 老年代(major gc/full gc)
- 永久代（方法区）存储着类和接口的元信息以及interned的字符串信息。(major gc/full gc)

__注1：JDK1.7+的变化__
- String#intern移出perm区到heap区。[http://tech.meituan.com/in_depth_understanding_string_intern.html](http://tech.meituan.com/in_depth_understanding_string_intern.html)
- JDK1.8移出perm区，改用native memory实现meta区。[http://www.infoq.com/cn/articles/Java-PERMGEN-Removed](http://www.infoq.com/cn/articles/Java-PERMGEN-Removed)

__注2：老年代的对象需要持有新生代对象的引用__
- 为了处理这种场景，在老年代中设计了"索引表(card table)"，是一个512字节的数据块。不管何时老年代需要持有新生代对象的引用时，都会记录到此表中。当新生代中需要执行GC时，通过搜索此表决定新生代的对象是否为GC的目标对象，从而降低遍历所有老年代对象进行检查的代价。该索引表使用写栅栏(write barrier)进行管理。write barrier是一个允许高性能执行minor GC的设备。尽管它会引入一个数据位的开销，却能带来总体GC时间的大幅降低

#### 新生代
![java-eden-areas](/img/2016-1-22-2.png)

1个Eden区 + 2个Survivor区

- 大多数新生对象都被分配在Eden区。
- 第一次GC过后Eden中还存活的对象被移到其中一个Survivor区。
- 再次GC过程中，Eden中还存活的对象会被移到之前已移入对象的Survivor区。
- 一旦该Survivor区域无空间可用时，还存活的对象会从当前Survivor区移到另一个空的Survivor区。而当前Survivor区就会再次置空。
- 经过数次在两个Survivor区域移动后还存活的对象最后会被移动到老年代。


__注：如何实现Eden区高效的内存分配：__
指针碰撞(bump-the-pointer)"和"TLABs(Thread-Local Allocation Buffers)
- Bump-the-pointer技术会跟踪在Eden上新创建的对象。由于新对象被分配在Eden空间的最上面，所以后续如果有新对象创建，只需要判断新创建对象的大小是否满足剩余的Eden空间。如果新对象满足要求，则其会被分配到Eden空间，同样位于Eden的最上面。所以当有新对象创建时，只需要判断此新对象的大小即可，因此具有更快的内存分配速度。
- 在多线程环境下，将会有别样的状况。为了满足多个线程在Eden空间上创建对象时的线程安全，不可避免的会引入锁，因此随着锁竞争的开销，创建对象的性能也大打折扣。在HotSpot中正是通过TLABs解决了多线程问题。TLABs允许每个线程在Eden上有自己的小片空间，线程只能访问其自己的TLAB区域，因此bump-the-pointer能通过TLAB在不加锁的情况下完成快速的内存分配。


#### 老年代
五种GC
- Serial GC
- Parallel GC
- Parallel Old GC(Parallel Compacting GC)
- Concurrent Mark & Sweep GC (or "CMS")
- Garbage First (G1) GC
（这里原文讲得不好，参见周志明老师的《深入理解JVM虚拟机》第三章）

[如何选择和设置虚拟机参数](http://www.cnblogs.com/redcreen/archive/2011/05/04/2037057.html)


### GC监控

#### jstat

```plain
jps # 找到运行的java程序的vid
jstat -gc [vid] 1000  # 每1s显示一次
```

其他选项：
- gc  输出堆空间上各分区当前的大小及使用量(Ede, Survivor, Old等)，GC执行的总次数以及累积消耗的执行时长。
- gccapacity  输出堆空间上各分区的最小和最大容量，当前大小，每个区上的GC执行次数(不输出当前使用量和累积的GC耗时)。
- gccause 除了输出 -gcutil提供的信息外，还会输出最后一次GC和当前GC的原因。
- gcnew   新生代上的GC性能数据。
- gcnewcapacity   新生代容量的统计信息。
- gcold   老年代的GC性能数据。
- gcoldcapacity   老年代容量的统计信息。
- gcpermcapacity  持久代(方法区)上的统计信息。
- gcutil  以%的格式输出每个分区的使用量。同时也会输出GC执行的总次数及累积耗时。

GC输出：
```plain
S0C: Current survivor space 0 capacity (kB).

S1C: Current survivor space 1 capacity (kB).

S0U: Survivor space 0 utilization (kB).

S1U: Survivor space 1 utilization (kB).

EC: Current eden space capacity (kB).

EU: Eden space utilization (kB).

OC: Current old space capacity (kB).

OU: Old space utilization (kB).

MC: Metaspace capacity (kB).

MU: Metacspace utilization (kB).

CCSC: Compressed class space capacity (kB).

CCSU: Compressed class space used (kB).

YGC: Number of young generation garbage collection events.

YGCT: Young generation garbage collection time.

FGC: Number of full GC events.

FGCT: Full garbage collection time.

GCT: Total garbage collection time.
 ```

尤其需要关注YGC, YGCT, FGC, FGCT和GCT数量的变化。

#### --verbose:gc
- 运行Java应用时的一个JVM选项
- 输出每次GC前后新生代和老年代的容量变化及GC耗时

#### VisualVM + Visual GC插件
- jvisualvm

#### HPJMeter
- 分析--verbose:gc输出的结果的GUI


### GC调优

GC调优未必需要，如果你做了以下三个事情：
- 通过-Xms和-Xmx指定了JVM的堆内存大小
- 使用了-server选项
- 系统未产生太多的超市日志

**gc调优是不得已的选择！**


#### GC调优的目标分为两类：

- 降低移动到老年代的对象数量
- 缩短Full GC的执行时间

老年代的GC较新生代会耗时更长，因此减少移动到老年代的对象数量可以降低full GC的频率。减少对象转移到老年代可能会被误解为把对象保留在新生代，然而这是不可能的，相反你可以**调整新生代的空间大小。**
如果企图通过缩小老年代空间的方式来降低Full GC执行时间，可能会面临OutOfMemoryError或者带来更频繁的Full GC。如果通过增加老年代空间来减少Full GC执行次数，单次Full GC耗时将会增加。因此，__需要为老年代空间设置适当的大小__。

__GC调优的基本规则是对两台或更多的服务器设置不同的选项，并对比性能表现，然后把被证明能提升性能的选项添加到应用服务器上.__


#### 影响GC的选项：
1. 空间设置：
 - 堆空间：
        + -Xms(启动JVM时的初始堆空间大小)
        + -Xmx(堆空间最大值)

 - 新生代空间：
        + XX:NewRatio（新生代与老年代比例）
        + -XX:NewSize (新生代大小)
        + -XX:SurvivorRatio（Eden区与Survivor区的比例）
 
 -XX:PermSize和-XX:MaxPermSize可以设置永久代的大小，但是只有PermSize不足爆OutMemoryError的时候才需要。(JDK1.8采用了Meta之后，基本更加不再需要设置了)

2. GC类型：
 - Serial GC
 - Parallel New GC
 - Parallel Scavenge GC
 - CMS GC
 - G1
 
 通常Serial GC只有在client端用。


#### GC调优的过程
1. 监控
2. 分析数据并决定是否需要GC调优
 - 如果分析结果显示GC耗时在0.1-0.3秒以内的话，一般不需要花费额外的时间做GC调优。然而，如果GC耗时达到1-3秒甚至10秒以上，就需要立即对系统进行GC调优。
 - 如果你的应用分配了10GB的内存，且不能降低内存容量的话，其实是没办法进行GC调优的。这种情况下，你首先要去思考为什么需要分配这么大的内存
3. 设置GC类型和内存大小
4. 分析调优的结果
5. 如果结果可接受，则对所有服务应用调优选项并停止调优


如果GC执行时间满足以下判断条件，那么GC调优并没那么必须。
- Minor GC执行迅速(50毫秒以内)
- Minor GC执行不频繁(间隔10秒左右一次)
- Full GC执行迅速(1秒以内)
- Full GC执行不频繁(间隔10分钟左右一次)

内存大小与GC执行次数、每次GC耗时之间的关系：

- 大内存
    会降低GC执行次数
    相应的会增加GC执行耗时
- 小内存
    会缩知单次GC耗时
    相应的会增加GC执行次数

GC调优的两个案例[http://segmentfault.com/a/1190000004303843](http://segmentfault.com/a/1190000004303843)


原文出处: [http://www.cubrid.org/blog/tags/Garbage%20Collection/](http://www.cubrid.org/blog/tags/Garbage%20Collection/)
