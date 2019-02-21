---
layout:     post
title:      percona工具pt-table-checksum

subtitle:   实战
date:       2019-2-21
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - MySQL
---
## percona工具pt-table-checksum

# 安装
````shell
官网地址
https://www.percona.com/software/database-tools/percona-toolkit
下载最新版本
安装依赖包
yum  install -y perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL

yum loaclinstall -y percona-toolkit*

````
# pt-table-checksum使用
````shell
1、在主库上创建percona库，并且创建一个slave表

create database percona;

use percona;

CREATE TABLE `slave` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`parent_id` int(11) DEFAULT NULL,
`dsn` varchar(255) NOT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;


2、创建检测账号
GRANT REPLICATION SLAVE,PROCESS,SUPER, SELECT ON *.* TO 'checksum_user'@'%' IDENTIFIED BY 'checksum_password';

GRANT ALL PRIVILEGES ON percona.* TO 'checksum_user'@'%';



3、insert记录到slave表，该记录表明要检测那个slave

insert into slave(`id`,`dsn`) values(1,'h=192.168.1.15,P=3306');


4、执行检测

pt-table-checksum --nocheck-replication-filter --no-check-binlog-format --databases="zx_sms"  --tables="sms_user_info"    --create-replicate-table --replicate=percona.checksums --recursion-method=dsn=D=percona,t=slave --host=localhost --port=3306 --user=checksum_user --password=checksum_password
````
````
注意报错
Character set 'utf8mb4' is not a compiled character set and is not specified in the '/usr/share/mysql/charsets/Index.xml' file
Character set 'utf8mb4' is not a compiled character set and is not specified in the '/usr/share/mysql/charsets/Index.xml' file
02-21T14:28:01 DBI connect(';host=localhost;port=3306;mysql_read_default_group=client','root',...) failed: Can't initialize character set utf8mb4 (path: /usr/share/mysql/charsets/) at /usr/bin/pt-table-checksum line 1581


修改  /usr/share/mysql/charsets/Index.xml
<charset name="utf8">
为
<charset name="utf8mb4">
````
````shell
5、同步数据(注意需要在从库上执行)  
./pt-table-sync --sync-to-master h=192.168.56.132,P=3306,u=root,p=123456 --databases=zx_sms --tables=sms_user_info --charset=utf8mb4 --print --execute    ----注意这里写从库的IP
````
#END
