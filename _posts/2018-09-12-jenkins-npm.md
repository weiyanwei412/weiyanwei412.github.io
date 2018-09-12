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
## npm太慢,淘宝cnpm镜像使用方法
```shell 
线上jenkins构建nodejs 项目，依赖下载比较慢，导致构建失败
解决方法
[root@jenkin-slave ~]# npm config set registry https://registry.npm.taobao.org
[root@jenkin-slave ~]# npm config get registry
http://registry.npm.taobao.org/
安装cnpm代替npm
[root@jenkin-slave ~]# npm install -g cnpm --registry=https://registry.npm.taobao.org
[root@jenkin-slave bin]# ll
total 29164
lrwxrwxrwx 1 jenkins jenkins      33 Sep 11 17:32 cnpm -> ../lib/node_modules/cnpm/bin/cnpm
-rwxrwxr-x 1 jenkins jenkins 29862509 Aug  1  2017 node
lrwxrwxrwx 1 jenkins jenkins     38 Jul 25 11:57 npm -> ../lib/node_modules/npm/bin/npm-cli.js

```
