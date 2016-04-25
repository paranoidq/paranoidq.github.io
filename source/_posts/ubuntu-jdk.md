title: Ubuntu如何安装JDK，并配置为默认
date: 2016-01-14 14:10:08
tags: [Linux, Ubuntu, Java, JDK, 碎片]
categories: 
- Java
- 配置
---

### Install Oracle JDK
```
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```
<!--more-->

### 设置JDK为默认环境
```
sudo apt-get install oracle-java8-set-default
```
或
```
paranoidq@ubuntu:~$ sudo update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                           Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-8-oracle/jre/bin/java         1072      auto mode
  1            /usr/lib/jvm/java-7-openjdk-i386/jre/bin/java   1071      manual mode
  2            /usr/lib/jvm/java-8-oracle/jre/bin/java         1072      manual mode

Press enter to keep the current choice[*], or type selection number: 2
```

### 配置JAVA_HOME等

1. 查看java路径
 - `which java`参考我的另一篇文章：[Mac下查找Java路径](http://paranoidq.github.io/2016/01/03/Mac-java-config/)
 - 或 sudo update-alternatives --config java
 - 或 readlink -f /usr/bin/java

2. 添加如下环境变量到~/.bashrc
 ```
 export JAVA_HOME=/usr/lib/jvm/java-8-oracle
 export JRE_HOME=${JAVA_HOME}/jre
 export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
 export PATH=${JAVA_HOME}/bin:$PATH
 ```

3. `source .bashrc`

 注：
 - ~~添加到`~./bashrc`只对本用户生效，添加到`/etc/profile`则对所有用户生效~~。
 - ~~网上直接添加/etc/environment的方法，实际上等同于对所有用户生效，不推荐~~。
 
 这里的解释不对，具体区别参见 [https://wido.me/sunteya/understand-bashrc-and-profile](https://wido.me/sunteya/understand-bashrc-and-profile)

### 参考资料

[Uninstall OpenJDK](http://askubuntu.com/questions/335457/how-to-uninstall-openjdk)
[Install JDK in Ubuntu](http://tecadmin.net/install-oracle-java-8-jdk-8-ubuntu-via-ppa/)
[http://www.cnblogs.com/myqiao/archive/2012/04/19/2457881.html](http://www.cnblogs.com/myqiao/archive/2012/04/19/2457881.html)
