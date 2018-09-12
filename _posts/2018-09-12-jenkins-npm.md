---
layout:     post
title:     jenkins node js构建
subtitle:   jenkins
date:       2018-09-12
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - nodejs
    - npm
    - jenkins
---
## npm太慢,淘宝npm镜像使用方法
```shell 
线上jenkins构建nodejs 项目，依赖下载比较慢，导致构建失败
解决方法
[root@jenkin-slave ~]# npm config set registry https://registry.npm.taobao.org
[root@jenkin-slave ~]# npm config get registry
http://registry.npm.taobao.org/
安装cnpm代替npm
[root@jenkin-slave ~]# npm install -g cnpm --registry=https://registry.npm.taobao.org
```
