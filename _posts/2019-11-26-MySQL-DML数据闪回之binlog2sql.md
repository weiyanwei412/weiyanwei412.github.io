---
layout:     post
title:     MySQL-DML数据闪回之binlog2sql
subtitle:   最佳实践
date:       2019-11-26
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - MySQL
---
## MySQL-DML数据闪回之binlog2sql
# 一、安装binlog2sql依赖
```
yum install git python-pip
git clone https://github.com/danfengcao/binlog2sql.git && cd binlog2sql
pip install -r requirements.txt
```
# 二、binlog2sql只支持MySQL数据库启动着
把需要解析的binlog复制到相应的logs目录下，对应好mysql-bin.index,赋权mysql权限
```
chown -R mysql. logs/
```
# 三、利用binlog2sql解析binlog
```
（1）解析zx_user_center库中uc_enterprise_user表 时间段内的dml  insert 语句
python /js/binlog2sql/binlog2sql/binlog2sql.py \
-hlocalhost -P3306 -uroot -p'XXXX' \
--start-file='mysql-bin.000004' \
--end-file='mysql-bin.000010' \
-d zx_user_center -t uc_enterprise_user \
--sql-type="INSERT" \
--start-datetime='2019-11-16 08:00:00' \
 --stop-datetime='2019-11-21 18:20:00' \
 >uc_user-iddle.sql

（2）解析zx_user_center库中uc_enterprise_user表 时间段内的dml  delete 语句
python /js/binlog2sql/binlog2sql/binlog2sql.py \
-hlocalhost -P3306 -uroot -p'XXXX' \
--start-file='mysql-bin.000004' \
--end-file='mysql-bin.000010' \
-d zx_user_center -t uc_enterprise_user \
--sql-type="DELETE" \
--start-datetime='2019-11-16 08:00:00' \
 --stop-datetime='2019-11-21 18:20:00' \
 >uc_user-iddle.sql

（3）生成zx_user_center库中uc_enterprise_user表 时间段内的dml  delete 语句的回滚语句

python /js/binlog2sql/binlog2sql/binlog2sql.py \
-hlocalhost -P3306 -uroot -p'XXXX' \
--start-file='mysql-bin.000004' \
--end-file='mysql-bin.000010' \
-d zx_user_center -t uc_enterprise_user \
--sql-type="DELETE" \
--start-datetime='2019-11-16 08:00:00' \
 --stop-datetime='2019-11-21 18:20:00' \
 -B >uc_user-iddle.sql
```
