---
layout:     post
title:     Python 基础学习
subtitle:   学习
date:       2018-09-03
author:     dbstack
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - python    
---
# Python 学习
## Python 第三方库安装
```shell
yum install -y python-pip
1、pip install XXX
2、easy_install XXX
```
## Python 的组成
```shell
1、库、包、模块、类、函数
2、数据结构：列表[]、字典{}、元组()、集合
3、注释、内建函数、变量、运算符
4、循环、异常抛错、队列、线程
```
## Python 难点
```shell
可变长参数--->函数编程式--->嵌套函数--->闭包--->装饰器
```
## vim  中设置tab为4个空格的方法：
```shell
cat >>~/.vimrc<<-EOF
set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab
EOF
```
## Python 空行的使用参考准则：
```
1）模块中类和函数之间空两行
2）在类中的方法（类中的函数）之间空一行
3）在函数中的逻辑段落间加空行，即把相关的代码紧凑写在一起，作为一个逻辑段落，段落间以空行分隔；
4）在import 不同种类的模块间加空行；
```
## Python 模块顺序
```python
import os
import sys
import threading

import tornado

import test

1）系统库
2）第三方库
4）本地模块

搜索模块路径：
1）程序所在目录
2）python安装标准库目录
3）第三方库目录
```
## Python 第三方库哪里找
https://pypi.org/  中搜索

## Python 内建函数
```python
isinstance :判断对象是否属于某种类型
a = 123
isinstance(a,int)
True

1）小练习
_input = [1, 'I', 1234, 'love', 'python']
#要求大家把输入中的字符串提取出来，并用' '隔开，连成一句话
方法一
#!/usr/bin/env python
#cofing:utf8
_input = [1, 'I', 1234, 'love', 'python']
for item in _input:
    if isinstance(item,str):
        print(item,end=' ')
方法二
#!/usr/bin/env python
#cofing:utf8
_input = [1, 'I', 1234, 'love', 'python']
ret = []
for item in _input:
    if isinstance(item,str):
         ret.append(item)
      ' '.join(item)
2）range 内建函数





