title: Hadoop(1) 单机安装（Ubuntu）
date: 2016-01-15 16:27:28
tags: [hadoop]
categories: [big data]
---


### 参考 
[hadoop_Install_on_ubuntu_single_node_cluster](http://www.bogotobogo.com/Hadoop/BigData_hadoop_Install_on_ubuntu_single_node_cluster.php)
[start hadoop时的文件夹权限问题](http://stackoverflow.com/questions/29059250/cant-start-namenode-daemon-and-datanode-daemon-in-hadoop)

<!--more-->

### 权限问题
```
sudo chown -R hadoop /usr/local/hadoop/  # 不推荐777
```

