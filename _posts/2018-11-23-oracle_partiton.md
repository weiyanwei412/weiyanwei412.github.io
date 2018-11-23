---
layout:     post
title:     oracle分区表增加
subtitle:   jenkins
date:       2018-09-12
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - oracle
    - 分区表
---
## oracle 分区表增加
# 1、查询当前用户所有的分区表
````shell
select * from USER_PART_TABLES
````
<p><img width="848" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/1.png?raw=true" height="303" alt=""></p>

# 2、查询分区表分区统计
 ````shell
 select table_name,partition_name,high_value,tablespace_name from all_tab_partitions where table_name='IM_SEND_TASK_SUMMARY_NEW' order by partition_position desc;
 ````
 <p><img width="848" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/2.png?raw=true" height="500" alt=""></p>


 
 
## 新增 SMS_SUBMIT_NEW_P分区表


## 新增带P_MAX分区表的语句
````shell
ALTER TABLE 【table_name 表名】 SPLIT PARTITION 【pmaxvalue -- 需要拆分的分区名】 at (to_date('2010-01-01','yyyy-mm-dd')) INTO (PARTITION p201001, PARTITION pmaxvalue) update indexes;

ALTER TABLE SMS_SUBMIT_NEW_P SPLIT PARTITION  P_MAX at (TO_DATE(' 2019-03-01 00:00:00', 'SYYYY-MM-DD HH24:MI:SS', 'NLS_CALENDAR=GREGORIAN')) INTO (PARTITION P_201902, PARTITION P_MAX) update indexes;   
````

# 1、查询全局索引（global）
````shell
select index_name,status from user_indexes where table_name='SMS_SUBMIT_NEW_P';
````
<p><img width="848" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/3.png?raw=true" height="303" alt=""></p>

# 2、查询指定 索引状态
````shell
select index_name,partition_name,status from user_ind_partitions where index_name='IND_SAN_TR2' order by partition_name;
````
<p><img width="848" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/4.png?raw=true" height="500" alt=""></p>
















