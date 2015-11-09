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

- __注：曾出现的问题__

    __Problem 1.__

        ./stack.sh: line 136: /root/devstack/lib/database: Permission denied
        ...
        sudo: >>> /etc/sudoers.d/50_stack_sh: syntax error near line 1

    分析：可能是由于使用了root用户进行上述一系列操作导致的问题，所以请用一般用户进行安装

    __Problem 2.__

        Running local.sh ... ERROR: openstack Could not determine a suitable URL for the plugin

    分析：应该是认证出现了错误，authv2和authv3的问题。解决办法(原因有待探究)：

    *This error is related to the version of Keystone API you're using. As noted in https://bugs.launchpad.net/python-openstackclient/+bug/1447704 (this open bug report), it does not work when using an OS_AUTH_URL ending in "v2.0" and the variables OS\_PROJECT\_DOMAIN\_ID and OS\_USER\_DOMAIN\_ID are set.*

    *The workaround is to remove the variables OS\_PROJECT\_DOMAIN_ID and OS\_USER\_DOMAIN\_ID if you wish to use v2 of the Keystone API.*



    - 在accrc的admin中comment掉OS\_PROJECT\_DOMAIN\_ID和OS\_USER\_DOMAIN\_ID
    - 修改stackrc中的IDENTITY\_API\_VERSION版本为3
    - 参考
        * [1 https://ask.openstack.org/en/question/67118/openstack-could-not-determine-a-suitable-url-for-the-plugin/](https://ask.openstack.org/en/question/67118/openstack-could-not-determine-a-suitable-url-for-the-plugin/)
        * [2 https://bugs.launchpad.net/python-openstackclient/+bug/1447704](https://bugs.launchpad.net/python-openstackclient/+bug/1447704)
        * [3 http://eavesdrop.openstack.org/irclogs/%23openstack-keystone/%23openstack-keystone.2015-02-10.log](http://eavesdrop.openstack.org/irclogs/%23openstack-keystone/%23openstack-keystone.2015-02-10.log)

    __Problem 3.__



#### 个性化配置（customize）

通过修改local.conf的方式实现（不再有旧版本的localrc文件），新的localrc以section的方式定义在local.conf中

local.conf文件中的section定义方式

```
'[[' <phase> '|' <config-file-name> ']]'
```
