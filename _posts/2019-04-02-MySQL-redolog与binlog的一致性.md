---
layout:     post
title:      "MySQL-redolog与binlog的一致性"
subtitle:   原理
date:       2019-04-02
author:     dbstack
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
  - MySQL
---

## MySQL-redolog与binlog的一致性
```
binlog与redo的一致性
使用 内部分布式事物 来保证一致性
在 commit 时（无论用户自己输入，或者系统自动添加），会有如下几个步骤：
1. InnoDB 层 写 prepare log
  写的还是 redo file （或者就是redo log，只是内容不一样，这里不是记录页的变化了）
  写入的是 xid （事物id，show binlog events in “bin.000056”）
  (准确的说， xid 是写在 undo 页上的)
2. MySQL 层 写 binlog
3. InnoDB 层 写 commit log （这里同样也是redo log file）
注意：这里的写入是指写入到磁盘（落盘成功）

用户或者系统commit  ===>redo file(prepare log)====>binlog====>redolog(commit log)

1. 假设，如果 没有 第一步的 prepare log ，而是直接写第二步的 MySQL binlog ，以及接着写第三步的 InnoDB commit log ：
此时假设出现 binlog写入成功 ，而 commit log(redo)写入失败 的情况（比如宕机），那随后机器重启时 恢复 时，就会对该事物 回滚 ；
万一此时的 binlog 已经传递到了 slave 机器上，且 slave上commit 了。那此时 主从就不一致 了（Master上回滚了）

2. 现在有 prepare log 了以后， prepare log写入成功 ，假设还是 binlog写入成功 ，而 commit log(redo)写入失败 的情况下；
此时事物恢复的时候， 检查到prepare log写入成功， binlog写入成功 ，那就直接 commit 了（可以理解成补了那次失败的commit），而 不管commit log是否写入成功 了。

3. 如果 prepare log写入成功 ， binlog写入失败 了，那恢复时，也会回滚

4. 如果 没开binlog ，就没有第一和第二步， 只写第三步的commit log ，恢复的时候没有commit log，就会回滚。
  一个事物在prepare log中写入成功，在binlog中写入成功，那就 必须要提交 （commit）
  用户/系统 commit ===> redo file（ prepare log） ===> binlog ===> redo file（ commit log）
  xid 即 写入prepare log 中，也会 写入到binlog 中，当恢复时，会 对比 一下某个 xid 在两个文件中是否都存在，如果都存在，该xid对应的事物才会提交

1. 在MySQL5.6以后， 写入binlog（步骤二） 和 写入commit log（步骤三） ，都是通过 组提交 的方式刷入（fsync）到磁盘的。
2. 在MySQL 5.7以后，写入 prepare log（步骤一） 也是通过 组提交 的方式刷入（fsync）到磁盘的（在写binlog之前执行一次fsync，就批量刷入prepare log）
注意：组提交中失败了，并 不会回滚 该组中的 所有事物 ，而是哪个失败了，就回滚哪个。
```
