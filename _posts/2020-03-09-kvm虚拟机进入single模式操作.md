---
layout:     post
title:      kvm虚拟机进入single模式操作
subtitle:   实战
date:       2020-03-09
author:     dbstack
header-img: img/post-bg-mma-5.jpg
catalog: true
tags:
    - kvm
    - linux 
---
## kvm虚拟机进入single模式操作

- 一、首先保证目标虚拟机配置文件xml种含有如下字段，不然无法在开机时候选择内核进行编辑
```
<graphics type='spice' autoport='yes'>
  <listen type='address'/>
  <image compression='off'/>
</graphics>
```

![zabbix-teample](https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/kvm-1.png?raw=true)

- 二、然后在虚拟机开机到内核选择步骤，按e进入编辑，添加init=/bin/bash，并执行ctl z
![zabbix-teample](https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/kvm-2.png?raw=true)
![zabbix-teample](https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/kvm-3.png?raw=true)





- 三、进入单用户模式后执行如下动作
```
mount -o remount,rw /
```
- 四、进行修改配置、密码更改等操作

- 五、最后退出命令：
```
exec /sbin/init
```





