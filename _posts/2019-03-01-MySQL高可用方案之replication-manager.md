---
layout:     post
title:      "MySQL高可用方案之replication-manager"
subtitle:   实践出真知
date:       2019-03-01
author:     dbstack
header-img: img/post-bg-cook.jpg
catalog: true
tags:
  - replication manager
  - MySQL
---
## MySQL高可用方案之Replication-Manager

- Replication-Manager 主要用于mysql主从结构的监控和主从切换.
- 代码仓库: https://github.com/signal18/replication-manager
- 官方文档：https://docs.signal18.io/installation/setup-instructions
- 配置yum  源
````shell
cat >/etc/yum.repos.d/signal18.repo<<-EOF
[signal18]
name=Signal18 repositories
baseurl=http://repo.signal18.io/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
EOF

1、安装
yum install replication-manager-osc

2.配置好一主双从的 MySQL主从结构（配置省略）
192.168.56.11:3306  master 
192.168.56.12:3306  slave
192.168.56.13:3306  slave


3、配置replication-manager
cat >/etc/replication-manager/config.toml<<-EOF
[db3306]
title = "db3306"
db-servers-hosts = "192.168.56.11:3306,192.168.56.12:3306,192.168.56.13:3306"
db-servers-prefered-master = "192.168.56.11:3306"
db-servers-credential = "root:db123"
replication-credential = "repl:123456"
failover-mode = "manual"
proxysql=true
proxysql-server="192.168.56.11"
proxysql-port=6033
proxysql-admin-port=6032
proxysql-writer-hostgroup=10
proxysql-reader-hostgroup=11
proxysql-user="admin"
proxysql-password="admin"
proxysql-bootstrap=false
[Default]
monitoring-datadir = "/var/lib/replication-manager"
monitoring-sharedir = "/usr/share/replication-manager"
log-level=7
log-file = "/var/log/replication-manager.log"
replication-multi-master = false
replication-multi-tier-slave = false
failover-readonly-state = true
http-server = true
http-bind-address = "0.0.0.0"
http-port = "10001"
EOF

3、启动
cat >start-replication-manager.sh<<-EOF
#!/bin/sh
nohup /usr/bin/replication-manager-osc monitor >/dev/null 2>&1 &
EOF

sh start-replication-manager.sh

在浏览器中打开：http://192.168.56.11:10001/

即可页面管理MySQL集群
````

# END
