---
layout: post
title: Linux学习总结-权限管理
category: Linux
tags: Linux
keywords: 
description: 
---


### 用户身份切换-su与sudo

#### 为什么要做多用户切换

- 使用一般账号，避免类似`rm -rf /`这样的危险操作
- 使用较低权限运行系统服务。例如用apache用户执行apache的软件，如果这个程序被破坏，则系统不至于损坏
- 默认不允许root登陆，如telnet。（ssh也可以拒绝root）登陆

#### 如何切换到root

- `su -` 直接变成root用户（需要root密码，需要一般用户也知道root密码）
- `sudo` 需要实现设置sudoers权限，只需要用户自己的密码即可执行root命令串

#### su

```
su # 单纯切换为root身份，但是读取的变量设置方式为non-login shell，原本的环境变量不会被改变为root环境

su - # login-shell的方式登陆，会读取root自己的环境变量（建议使用这种方式）

# 切换特定用户
su -username 
su -l username

# 仅执行一次命令
su - -c "head -n 3 /etc/shadow" # -c表示仅执行一次命令，执行完成后继续使用旧身份
```

默认情况下，ubuntu没有开启root密码和启用状态，需要设置后才能su：

```
$ sudo passwd root
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully
$ su -
# ...
```

#### sudo




