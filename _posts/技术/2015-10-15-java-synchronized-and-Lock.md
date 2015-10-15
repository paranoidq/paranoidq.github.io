---
layout: post
title: 深入Java synchronized
category: 技术
tags: Java MultiThreading
keywords: 
description: 
---


###synchronized锁获取与释放

####获取


####释放

两种情况下会（JVM自动）释放锁：

1. 获取锁的线程执行完了该代码块，然后线程释放对锁的占有；
2. 线程执行发生异常，此时JVM会让线程自动释放锁。

>产生的问题：

如果这个获取锁的线程由于要等待IO或者其他原因（比如调用sleep方法）被阻塞了，但是又没有释放锁

>解决方案：Lock


###synchronized的缺陷
当某个线程进入同步方法获得对象锁，那么其他线程访问这里对象的同步方法时，必须等待或者阻塞，这对高并发的系统是致命的，这很容易导致系统的崩溃。如果某个线程在同步方法里面发生了死循环，那么它就永远不会释放这个对象锁，那么其他线程就要永远的等待。这是一个致命的问题。


###参考
[1] [http://langgufu.iteye.com/blog/2152608](http://langgufu.iteye.com/blog/2152608)

[2] [http://www.cnblogs.com/dolphin0520/p/3923167.html] (http://www.cnblogs.com/dolphin0520/p/3923167.html)