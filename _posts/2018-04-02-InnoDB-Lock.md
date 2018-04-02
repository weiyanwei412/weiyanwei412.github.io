---
layout:     post
title:      InnoDB-Lock
subtitle:   搭建
date:       2018-04-02
author:     dbstack
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - MySQL
---


## InnoDB Lock
一、InnoDB锁的类型
- 1、标准的行级锁
- a、共享锁（S Lock） 
- b、排它锁 (X Lock)
- 解释：如果一个事务T1已经获得了行r的共享锁，那么另外的事务T2可以立即获得行r的共享锁，因为读取并没有改变行r的数据，这种情况为锁兼容（Lock Compatible）
- 但若有其他的事务T3想获的行r的排它锁，则必须等待事务T1、T2释放行r上的共享锁---这种情况成为锁不兼容。
''''
排它锁与共享锁的兼容性
      X       S
X   不兼容   不兼容
S   不兼容    兼容
上表为同一行数据的兼容性
''''
# InnoDB 支持多粒度锁定
''''
颗粒度锁定允许事务在行级上的锁和表级上的锁同时存在。为了支持在不同颗粒度上进行加锁操作，InnoDB存储因为支持一种额外的锁方式，称之为意向锁（Intention Lock）
意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度（fine grantularity）上进行加锁。
如下图
''''
- ![GitHub Logo](../img/lock-1.jpg "lock-1.jpg")