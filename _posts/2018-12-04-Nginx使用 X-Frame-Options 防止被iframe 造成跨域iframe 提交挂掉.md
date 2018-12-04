---
layout:     post
title:      Nginx使用 X-Frame-Options 防止被iframe 造成跨域iframe 提交挂掉
subtitle:   nginx
date:       2018-12-04
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - nginx
    - linux 
---

## Nginx使用 X-Frame-Options 防止被iframe 造成跨域iframe 提交挂掉
````shell
Refused to display 'http://www.***.com/org/***' in a frame because it set 'X-Frame-Options' to 'SAMEORIGIN'. 
触发原因：页面的返回头被设置 X-Frame-Options SAMEORIGIN ，只能被同源的iframe 引用。跨域名的iframe 没法显示了。

nginx 在 http://www.***.com/org/ location下增加
      proxy_hide_header X-Frame-Options;
      add_header X-Frame-Options "ALLOW-FROM https://www.***.com/" always;
即可
````

