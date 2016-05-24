title: Java单例的实现方式及问题(转载 + 整理)
date: 2016-01-17 20:07:35
tags: [java, 设计模式, 单例]
categories: 
- 设计模式
---

### 7种主要的实现方式

1. 懒汉，线程不安全
2. 懒汉，synchronized
3. 饿汉，static final域中直接new
4. 饿汉，static块中直接new （类似3）
5. Double-check
6. 静态内部类
7. Enum

实现代码及关键解释参见我的[GitHub Repo](https://github.com/paranoidq/JavaHackUtils/tree/master/src/main/java/me/util/singleton).


### 常见单例实现的几个额外考虑的问题

1. 可序列化 - 需要重载readResolve()函数，返回同样的实例
 修复办法：(这个我不是很理解, why这么做？)
 ```java
  private static Class<?> getClass(String className) throws ClassNotFoundException {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if(classLoader == null)
            classLoader = SingletonWithStaticNestedClass.class.getClassLoader();
        return (classLoader.loadClass(className));

 }
 ```
2. 不同类加载器加载导致对象不同的问题
 修复办法：
 ```java
 private Object readResolve() {
        return SingletonHolder.instance;
 }
 ```


### 参考
[7种实现方式](http://www.blogjava.net/kenzhh/archive/2013/03/15/357824.html)
[双重检查的陷阱和解决方法及原理](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)
[Effective way to implement singleton in java](http://stackoverflow.com/questions/70689/what-is-an-efficient-way-to-implement-a-singleton-pattern-in-java)

