title: SSH Principles（总结和笔记）
tags: ssh
categories: 
- Linux
- SSH
---

### 前提

服务器需要开启远程服务
``` 
# 针对fedora的检测方法
service sshd status
# 通用检测方法：
ps -e | grep ssh

# 开启 in fedora
service sshd start
# 开启 in ubuntu
/etc/init.d/ssh start[/stop/restart]
```

<!--more-->

### 原理

1. 基本流程：
    - client请求登陆remote server
    - server发送公钥给client，并告知client发送自己的密码
    - client用公钥加密密码，发送给server
    - server用私钥解密密码，如果成功，就允许用户登陆
2. 为何安全：
    - 全程不传输私钥，及时截获报文，只要私钥不泄露，就不能获取密码
    - 密码由随机的公钥加密，可换
3. 为何有风险：
    - 如果有人截获了登录请求，然后冒充远程主机，将伪造的公钥发给用户，那么用户很难辨别真伪。因为不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，也就是说，都是自己签发的
    - 中间人攻击（Man-in-the-middle attack）：如果攻击者插在用户与远程主机之间（比如在公共的wifi区域）用伪造的公钥，获取用户的登录密码。再用这个密码登录远程主机，那么SSH的安全机制就荡然无存了

### 两种登录方式

#### 口令登录

```
$ ssh user@host
The authenticity of host 'host (12.18.429.21)' can't be established.
RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
Are you sure you want to continue connecting (yes/no)?
```
假定经过风险衡量以后，用户决定接受这个远程主机的公钥。然后要求用户输入密码
    ```
Warning: Permanently added 'host,12.18.429.21' (RSA) to the list of known hosts.
Password: (enter password)
```
保存远程主机的公钥在`$HOME/.ssh/known_hosts`中，以后连接主机时client能识别公钥已经保存在本地，跳过警告部分。（但是仍然需要输入密码）

`/etc/ssh/ssh_known_hosts`保存对所有用户可信的远程主机公钥


#### 公钥登录

>原理：

1. 用户将自己的公钥储存在远程主机上。
2. 登录时，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回来。
3. 远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录shell，不再要求密码

>操作：

用户生成自己的公钥：在`$HOME/.ssh`下会生成`id_rsa.pub`和`id_rsa`
```
ssh-keygen
```

拷贝公钥到server：

```
$ ssh-copy-id user@host
# 或
$ scp /Users/paranoidq/.ssh/id_rsa.pub paranoidq@192.168.235.131:/home/paranoidq/.ssh/authorized_keys 
# 但是scp不能附加多个authorized_keys，所以貌似只能支持一个用户一个公钥！
```

> 关于ssh-copy-id的过程：

远程主机将用户的公钥，保存在登录后的用户主目录的`$HOME/.ssh/authorized_keys`文件中。公钥就是一段字符串，只要把它追加在authorized_keys文件的末尾就行了

```
$ ssh user@host 'mkdir -p .ssh && cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```
解释：
1. `$ ssh user@host`表示登录远程主机；
2. `mkdir -p .ssh && cat >> .ssh/authorized_keys`表示登录后在远程shell上执行的命令
3. `mkdir -p .ssh`的作用是，如果用户主目录中的.ssh目录不存在，就创建一个
4. `cat >> .ssh/authorized_keys < ~/.ssh/id_rsa.pub`的作用是，将本地的公钥文件`~/.ssh/id_rsa.pub`，重定向追加到远程文件`authorized_keys`的末尾

注：有必要学习一下bash和shell的知识了，推荐Mendel Cooper的[《Advanced Bash: Scrpiting Guide》](http://www.tldp.org/LDP/abs/html/)


### 参考
1. [廖雪峰的官方网站，ssh原理与运用](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)
