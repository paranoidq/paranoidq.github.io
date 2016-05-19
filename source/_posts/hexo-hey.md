title: Hexo博客管理神奇：hexo hey
tags:
  - 碎片
  - hexo
categories:
  - hexo
date: 2016-05-19 13:48:04
---

>有了 hexo hey，妈妈再也不用担心我的hexo博客无法管理了。

### 安装
1. install hexo-hey
```
npm install hexo-hey --save
```

2. add admin user to `_config.yaml`
```
Admin
admin:
    name: hexo
    password: hey
    secret: hey hexo
    expire: 60*1
    # cors: http://localhost:3000
```
3. hexo serve
然后访问 [http://localhost:4000/admin](http://localhost:4000/admin)

<!--more-->

### 注意问题

1.  更新npm，否则有可能出现`TypeError: Cannot read property '_id' of undefined]`错误
```
npm update -S
```

2. 关于添加多个tag或category的问题:
需要手动Enter才能添加


### 参考
[https://github.com/hexojs/hexo](https://github.com/hexojs/hexo)
[https://github.com/nihgwu/hexo-hey/issues/17](https://github.com/nihgwu/hexo-hey/issues/17)