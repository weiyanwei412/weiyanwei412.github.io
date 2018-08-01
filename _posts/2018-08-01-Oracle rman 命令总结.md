---
layout:     post
title:     Oracle rman 命令总结
subtitle:   最佳实践
date:       2018-07-31
author:     dbstack
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - linux
    - oracle
---
## Oracle rman 命令总结
```shell
--登录rman
  rman target /
  rman target sys/passwork
  rman target sys/passwork nocatalog   (控制文件方式)
  rman target sys/passwork catalog     (恢复目录方式)


--查看参数
  show all


--修改保存天数
  用sqlplus修改备份信息在控制文件中保留的天数
  show parameter control_file_record_keep_time

  alter system set control_file_record_keep_time=30 scope=spfile
  shutdown immediate
  startup


--rman数据库冷备份
  shutdown immediate;
  startup mount;
  backup database format='/u01/backup/rman/%d_%T_%s.bak';
  alter database open;
  sql 'alter system archive log current';


--rman数据库热备份
  backup database format='/u01/backup/rman/%d_%T_%s.bak';
  sql 'alter system archive log current';


--备份表空间
  backup tablespace emp；


--备份数据文件
  backup datafile '/u01/mytest.dbf';
  backup datafile 5 format='/u01/backup/rman/%N_%s.dbf';


--备份归档日志
  backup archivelog all
  backup archivelog from time 'sysdate-1'
  backup archivelog from sequence 400
  backup archivelog all delete input
  backup archivelog from sequence 400 delete input

  format='/u01/backup/rman/ar%T_%s.arc'   --指定路径 %T 日期


--备份过去一天的归档文件
  backup format='/u01/backup/rman/ar%d_%s.arc'
  archivelog
  from time='sysdate-1' until time='sysdate';


--备份数据文件和归档日志
  backup format='/u01/backup/rman/t%d_%s.bak' tablespace emp plus archivelog;


--备份控制文件
  backup current controlfile format='/u01/backup/rman/%d_%s.ctl';


--备份spfile
  backup spfile format='/u01/backup/rman/spf%d_%s.par';


--压缩备份集
backup as compressed backupset tablespace emp;


--建立控制文件映像副本
copy current controlfile to '/u01/backup/rman/dbtest.ctl';
backup as copy format='/u01/backup/rman/dbtest01.ctl' current controlfile;


--建立数据文件映像副本
backup as copy format='/u01/backup/rman/8.dbf' datafile 8;


--rman维护命令
list backup                                    --列出全部的备份信息
list backup of database                        --列出数据库备份
list backup of tablespace emp                  --列出指定的表空间备份
list backup of datafile 5                      --列出指定的数据文件备份 
list backup of controlfile                     --列出控制文件备份
list backup of spfile                          --列出spfile备份 
list archivelog all                            --列出归档日志
list backup of archivelog all                  --列出归档日志的备份

list backup of database summary                --列出可用的备份
list backup of tablespace emp summary          --关于表空间的备份
list backup by file                            --按文件类型列出备份
list expired backup of archivelog all summary  --失效的备份

report obsolete                                --查看过期的
delete obsolete                                --删除过期的

list recoverable backup of database            --列出有效的备份
list expired backup                            --列出失效的备份

list expired backup of archivelog all          --列出失效的归档日志备份
list expired backup of archivelog 
     until sequence 5                          --列出指定序列号的失效归档日志备份
list expired backup of archivelog 
     until time "to_date('2012-6-30','yyyy-mm-dd')"    
                                               --列出指定时间的失效归档日志备份备份

list copy                                      --列出映像文件副本  
list copy of database
list copy of tablespace emp
list copy of datafile 6
list copy of archivelog all
list copy of controfile

report schema
report need backup                             --列出需要备份的
report need backup days 2 database             --列出超过2天没有备份的

mount状态下
list incarnation;
reset database to incarnation 980;


--删除失效文件
删除失效备份
crosscheck backup(copy,archivelog all);
delete expired backup(copy,archivelog all);


删除失效日志
crosscheck archivelog all;  
delete expired archivelog all;

crosscheck backup of tablespace sysaux        --检查表空间备份
crosscheck backup of datafile 2               --检查数据文件2备份
crosscheck backup of controlfile              --检查控制文件备份
crosscheck backup of spfile                   --检查spfile
crosscheck backup of copy                     --检查copy
crosscheck backup completed after 'sysdate-2'  --当前时间前2天的备份

crosscheck copy of database
crosscheck copy of tablespace emp
crosscheck copy of controlfile
crosscheck copy of spfile

list backup summary                 --获得主键
validate backupset 16               --验证备份集16的有效性
change                              --修改备份状态
change backupset 16 unavailable
change backupset 16 available
change archivelog '/u01/backup/rman/***.log' unavailable

change backupset 16 delete                  --删除备份集16(同步删除)
delete expired backupset(archivelog all);   --删除失效

delete expired   --删除失效备份
delete obsolete  --删除旧于备份策略日期（过期）的备份


--恢复检查
restore database validate;
validate backupset 218;

restore database preview;
restore tablespace users preview;
restore datafile 5 preview;


--命令块
run{
2> shutdown immediate;
3> startup mount;
4> allocate channel d1 type disk;
5> backup as backupset database
6> format='/u01/backup/rman/%d_%T.bak';
7> alter database open;
8> sql 'alter system archive log current';
9> }

select * from v$log;
select * from v$archived_log;
select * from v$backup_redolog;


--恢复顾问
list failure       --诊断错误
advise failure     --建议
repair failure     --修复(数据文件和控制文件)


--rman下对数据文件重命名
run{
2> sql 'alter tablespace yesorno offline';
3> set newname for datafile '/u01/app/oracle/oradata/yesorno.dbf'
4> to '/u01/app/oracle/oradata/yesorno01.dbf';
5> restore tablespace yesorno;
6> switch datafile all;
7> recover tablespace yesorno;
8> sql 'alter tablespace yesorno online';
}


--rman下对数据文件移动
run{
2> sql 'alter tablespace yesorno offline';
3> set newname for datafile '/u01/app/oracle/oradata/yesorno01.dbf'
4> to '/u01/app/oracle/oradata/dbtest/yesorno01.dbf';
5> restore tablespace yesorno;
6> switch datafile all;
7> recover tablespace yesorno;
8> sql 'alter tablespace yesorno online';
}
```
## END
