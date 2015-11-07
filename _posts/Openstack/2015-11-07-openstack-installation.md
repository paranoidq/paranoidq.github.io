---
layout: post
title: Openstack Learning(1)-Installation
category: Openstack
tags: Openstack
keywords: 
description: 
---

### Devstack方式安装

#### 基本安装

- __Configure VM and install host OS distribution__

    (Ubuntu 14.04 (Trusty), Fedora 21 (or Fedora 22) and CentOS/RHEL 7 will be perfered)

- __Config git__

    apt-get instal git

- __Clone devstack repo__

        git clone https://git.openstack.org/openstack-dev/devstack

- __Configure__
    + devstack/samples/local.conf
    + devstack/samples/local.sh

    需要将这两个文件copy到devstack的主目录下面，miminal config只需要修改几个密码即可

    （其中解释了，local.conf替换旧版的localrc，以[local|localrc] section的方式出现，会替换在stack.sh中出现的默认值）

- __注册no-root用户__

    1.devstack不再支持root直接安装，需要创建用户。方法：

        cd devstack/tools/
        ./create-stack-user.sh

    2.然后修改权限
        sudo passwd stack # 设置stack用户密码
        chown -R stack:stack . #设置stack用户为devstack文件夹的owner和group 


- __Install__
    + cd devstack
    + ./stack.sh
    + 然后等待安装即可


#### 个性化配置（customize）

通过修改local.conf的方式实现（不再有旧版本的localrc文件），新的localrc以section的方式定义在local.conf中

local.conf文件中的section定义方式

```
'[[' <phase> '|' <config-file-name> ']]'
```
