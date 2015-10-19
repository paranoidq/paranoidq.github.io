---
layout: post
title: 操作系统进程的状态、转换及其调度策略
category: 技术
tags: OS Process
keywords: 
description: 
---


###操作系统进程的状态：

- 新建：process正在被创建
- 运行：process获得CPU的执行时间
- 等待：process等待某个事件发生
- 就绪：process进入就绪队列，等待分配CPU
- 终止：process完成执行

![process status](/public/img/posts/2015-10-19/process-schduling-1.JPG)

###进程控制

####创建 （fork()）
- 为new process分配唯一的Pid，并申请`PCB`（有限），若PCB申请失败，则创建失败
- 为new Process分配资源：分配程序、数据、用户栈等必要的空间（`PCB中体现`）。注意：这里内存不足时，处于`等待`状态或`阻塞`状态，等待内存资源
- init PCB：标志信息、状态、CPU控制信息、优先级
- 加入就绪队列（在可以容纳的情况下，**如果无法容纳呢？**）

列出进程列表

``` 
ps
```

列出系统所有当前活动进程的完整信息

```
ps -el
```

当进程创建新进程时，有两种可能：

1. 父进程与子进程并发执行
2. 父进程等待，直到某个或全部子进程执行完毕

新进程的地址空间也有两种可能：

1. 是父进程的复制品（内容拷贝，但是地址不同）
2. 子进程装入另一个程序（通过`exec()/execlp()`将二进制文件装入，替换原有的程序段）


####终止 (exit())
- 根据被终止进程的标识符，检索PCB，从中读出该进程的状态。
- 若被终止进程处于执行状态，立即终止该进程的执行，将处理机资源分配给其他进程。
- 若该进程还有子进程，则`应将其所有子进程终止`。
- 将该进程所拥有的全部资源，或归还给其父进程或归还给操作系统。
- 将该PCB从所在队列（链表）中删除。

####阻塞和唤醒

阻塞

- 找到将要被阻塞进程的标识号对应的PCB。
- 若该进程为运行状态，则保护其现场，将其状态转为阻塞状态，停止运行。
- 把该PCB插入到相应事件的等待队列中去。

唤醒

- 在该事件的等待队列中找到相应进程的PCB。
- 将其从等待队列中移出，并置其状态为就绪状态。
- 把该PCB插入就绪队列中，等待调度程序调度。

####进程的切换

- 保存CPU context：程序计数器、寄存器信息
- 更新PCB
- 把process PCB移入相应队列，如就绪队列、阻塞队列等
- 根据调度算法选择另一个process，更新PCB
- 更新内存管理数据结构
- 恢复CPU context


###进程构成

1. 进程控制块PCB
2. 程序段 （可以被多个进程共享）
3. 数据段 （原始数据或中间结果）：全局变量
4. 堆栈段 （临时数据）：函数参数、返回地址、局部变量等

####PCB

1. 进程描述信息：
    - PID，进程唯一标识
    - UID，进程的归属用户，用户标识主要为共享和保护服务

2. 进程控制和管理信息：
    - process状态，作为调度依据
    - process优先级
    - 代码运行入口地址
    - 程序外存地址
    - 进入内存时间
    - CPU占用时间
    - 信号量使用

3. 资源分配清单：
    - 代码段指针
    - 数据段指针
    - 堆栈段指针
    - 文件描述符
    - 键盘
    - 鼠标

4.  CPU相关信息：<u>当进程被切换时，处理机状态信息 都必须保存在相应的PCB中，以便在该进程重新执行时，能再从断点继续执行</u>。
    - 通用register值
    - 地址register值
    - 控制register值
    - 标志register值
    - 状态字

####Linux下的PCB
```c
// 就绪队列中，PCB通过双向链表链接，内核为当前正在运行的进程保存current指针
// 设备队列中，PCB也通过双向链表链接，每个设备都有自己的设备队列
task_struct:
    pid_t pid;
    long state; // process state
    unsined int time_slice;
    struct files_struct *files;
    struct mm_struct *mm;
```




###进程通信

####共享存储

通信进程间访问同一个内存区域，需要同步互斥工具。分两种：

- 低级方式：基于数据结构的共享
- 高级方式：基于存储区的共享

**注意:** 用户进程空间一般独立，需要特殊的系统调用实现（**how?**）。（线程则是自然共享进程空间）


POSIX共享内存

```c
#include<stdio.h>
#include<sys/shm.h>
#include<sys/stat.h>

int main()
{
    /* the identifier for the shared mem segment */
    int segment_id;
    /* a pointer to the shared mem segment */
    char *shared_memory;
    /* the size(in bytes) of the shared memory segment */
    const int size = 4096;

    /* allocate a shared memory segment */
    /*
     * IPC_PRIVATE:     共享内存段关键字，生成一个新的共享内存段
     * S_IRUSR|SIWUSR:  表示如何使用，读or写
    */
    segment_id = shmget(IPC_PRIVATE, size, S_IRUSR | S_IWUSR);

    /* attach the shared memory segment */
    /*
     * 参数1： 共享内存段的整数标识
     * 参数2： 指针，将要加入到共享内存的位置。NULL表示由OS选择位置
     * 参数3： 标识，指定加入到的内存共享区域是只读还是只写等模式，0表示读写均可
    */
    shared_memory = (char *)shmat(segment_id, NULL, 0);

    /* write a message to the shared memory segment */
    sprintf(shared_memory, "Hi, There!");

    /* now print out the string from the shared memory */
    printf("*%s\n", shared_memory);

    /* now detach the shared memory */
    shmdt(shared_memory);

    /* now remove the shared memory segment */
    shmctl(segment_id, IPC_RMID, NULL);

    return 0;
}

```


####消息传递

进程间发送格式化消息，操作系统提供发送消息和接受消息的premitive。

- 直接通信：直接发送到接受进程的消息缓冲队列，并取消息
- 间接通信：发送消息到某个中间实体（message box）

####管道通信

消息传递的一种特殊方式，管道：用于连接一个读进程和一个写进程以实现他们之间通信的共享文件-pipe文件。发送进程向管道提供字符流，接收进程则接收管道的输出

需要的能力：互斥、同步、确定对方存在。



###进程调度

- 长期调度程序（作业调度）
    + 从磁盘的缓冲池中选择进程，装入内存准备执行
    + 执行频率低，开销大
    + 控制多道程序设计的程度（内存中进程的数量）
    + 需要协调IO-intensive的进程和CPU-intensive的进程，如果IO-intensive过多，会导致短期调度程序无事可做；如果都是CPU-intensive进程，则系统设备几乎不被使用，CPU压力过大。
- 短期调度程序 （CPU调度）
    + 从内存中选择进程并为之分配CPU
    + 执行频率高，开销小

Windows/Unix没有长期调度程序，而只是将所有新进程放入内存调度。

中期调度程序：进程可以换出内存，swapping。



###参考

[1] [http://c.biancheng.net/cpp/html/2591.html](http://c.biancheng.net/cpp/html/2591.html)