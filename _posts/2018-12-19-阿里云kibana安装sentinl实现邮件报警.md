---
layout:     post
title:      阿里云kibana安装sentinl实现邮件报警
subtitle:   sentinl
date:       2018-12-19
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - sentinl
    - ELK
    - kibana
---

## 阿里云kibana安装sentinl实现邮件报警
- sentinl github地址：https://github.com/sirensolutions/sentinl
- sentinl github个版本地址：https://github.com/sirensolutions/sentinl/releases

# 安装sentinl
````shell
1、 下载自己需要的版本，我这里kibana是6.5.2 我下载了sentinl-v6.5.2.zip
2、 安装sentinl /usr/share/kibana/bin/kibana-plugin install file:///`pwd`/sentinl-v6.5.2.zip
3、 重启kibana systemctl restart kibana
````
访问kibana
http://192.168.100.15:5601
如图

