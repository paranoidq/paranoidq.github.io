title: GC专家系列-笔记
date: 2016-01-22 14:47:29
tags: [java, jvm]
categories: [java]
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


原文出处: [http://www.cubrid.org/blog/tags/Garbage%20Collection/](http://www.cubrid.org/blog/tags/Garbage%20Collection/)
