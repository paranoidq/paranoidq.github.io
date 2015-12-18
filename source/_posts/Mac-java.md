title: Mac中如何查找Java的路径
date: 2015-12-18 10:28:26
tags: [mac, java]
categories: mac
---


### 方法1：
最直接的方法，运行以下命令可以列出系统的default java版本和所有备选的java版本
```
/usr/libexec/java_home
```

<!--more-->

### 方法2：
step-wise的方法

```
which java
```
如果输出的时/usr/bin/java, 证明是链接，需要找到链接的source
```
ls -l `which java`
```
输出为实际的java安装路径: `rwxr-xr-x  1 root  wheel  74 11 25 13:36 /usr/bin/java -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/java`

如果/usr/bin/java指向的仍然是一个symbolic link, 那么继续执行以下命令，直到找到source为止 

[注]

ls -l \` which java\`

单引号把Linux命令视为字符集合。反引号会强迫执行Linux命令。和`$()`一样。在执行一条命令时，会先将其中或者是`$()` 中的语句当作命令执行一遍，再将结果加入到外层命令中执行



### 参考：
[http://stackoverflow.com/questions/18144660/what-is-path-of-jdk-on-mac](http://stackoverflow.com/questions/18144660/what-is-path-of-jdk-on-mac)
