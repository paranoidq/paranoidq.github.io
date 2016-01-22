title: Docker学习笔记
date: 2016-01-20 18:26:21
tags: [docker]
categories: [docker]
---

### 安装（Mac）
[Docker Toolbox](https://docs.docker.com/engine/installation/mac/)


### 创建docker machine
```
$ docker-machine create --driver vmwarefusion demo
```
<!--more-->

之后需要进行一些设置和初始化工作
```
$ docker-machine env demo
// 按照提示运行 eval
$ eval $(docker-machine env demo)
// 现在docker-machine demo已经running了
$ docker-machine ls

NAME   ACTIVE   URL            STATE     URL                          SWARM   DOCKER   ERRORS
demo   *        vmwarefusion   Running   tcp://192.168.235.137:2376           v1.9.1
```

启动和停止：
```
docker-machine start demo
docker-machine stop demo
```

### ssh连接

```
docker-machine ssh demo
```

### 获取镜像
```
docker run -d nginx # -d表示后台运行
docker ps           # 查看正在运行的镜像
docker ps -a        # 查看所有镜像
docker kill <name>  # 停止镜像
docker rm <name>    # 移除镜像
docker images       # 查看本地已有的镜像
```


