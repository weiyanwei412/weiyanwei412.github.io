---
layout:     post
title:      局域网DNS 之CoreDNS 集群部署
subtitle:   实战
date:       2020-02-19
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - DNS
    - linux 
---

## 局域网DNS 之CoreDNS 集群部署
# 基础环境及软件版本
- centos 7.6 
- etcd  3.3.18  https://github.com/etcd-io/etcd/releases 
- coredns 1.6.7  https://github.com/coredns/coredns/releases
- etcd1 coredns-node1 172.18.1.11
- etcd2 coredns-node2 172.18.1.12
- etcd3 coredns-node3 172.18.1.13
# 一、etcd 集群部署
- 下载好对应版本，并mv 到 /usr/bin/目录下
```
 (1)tar -xf etcd-v3.3.18-linux-amd64.tar.gz && cd etcd-v3.3.18-linux-amd64 && mv etcd* /usr/bin/
 (2)创建数据目录 与配置文件 三个节点分别执行 mkdir /var/lib/etcd
 (3)启动脚本
172.18.1.11上
 cat >/usr/lib/systemd/system/etcd.service<<-"EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name=etcd1 \
  --initial-advertise-peer-urls=http://172.18.1.11:2380 \
  --listen-peer-urls=http://172.18.1.11:2380 \
  --listen-client-urls=http://172.18.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=http://172.18.1.11:2379 \
  --initial-cluster-token=dns-etcd \
  --initial-cluster=etcd1=http://172.18.1.11:2380,etcd2=http://172.18.1.12:2380,etcd3=http://172.18.1.13:2380  \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

172.18.1.12上
 cat >/usr/lib/systemd/system/etcd.service<<-"EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name=etcd2 \
  --initial-advertise-peer-urls=http://172.18.1.12:2380 \
  --listen-peer-urls=http://172.18.1.12:2380 \
  --listen-client-urls=http://172.18.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=http://172.18.1.12:2379 \
  --initial-cluster-token=dns-etcd \
  --initial-cluster=etcd1=http://172.18.1.11:2380,etcd2=http://172.18.1.12:2380,etcd3=http://172.18.1.13:2380  \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

172.18.1.13上执行

 cat >/usr/lib/systemd/system/etcd.service<<-"EOF"
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name=etcd3 \
  --initial-advertise-peer-urls=http://172.18.1.13:2380 \
  --listen-peer-urls=http://172.18.1.13:2380 \
  --listen-client-urls=http://172.18.1.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=http://172.18.1.13:2379 \
  --initial-cluster-token=dns-etcd \
  --initial-cluster=etcd1=http://172.18.1.11:2380,etcd2=http://172.18.1.12:2380,etcd3=http://172.18.1.13:2380  \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
 
（4）etcd 启动
三台主机上分别执行
systemctl  daemon-reload
systemctl enable etcd
systemctl start etcd

（5）etcd  集群检查
ETCDCTL_API=3  etcdctl   --endpoints 172.18.1.11:2379,172.18.1.12:2379,172.18.1.13:2379 endpoint status  --write-out="table"
+------------------+------------------+---------+---------+-----------+-----------+------------+
|     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+------------------+------------------+---------+---------+-----------+-----------+------------+
| 172.18.1.11:2379 | a665d104822ee7c9 |  3.3.18 |   25 kB |      true |        81 |         25 |
| 172.18.1.12:2379 | f4f9dc017f438b07 |  3.3.18 |   25 kB |     false |        81 |         25 |
| 172.18.1.13:2379 | da1a094d31db677c |  3.3.18 |   25 kB |     false |        81 |         25 |
+------------------+------------------+---------+---------+-----------+-----------+------------+

ETCDCTL_API=3  etcdctl   --endpoints 172.18.1.11:2379,172.18.1.12:2379,172.18.1.13:2379 endpoint health
172.18.1.11:2379 is healthy: successfully committed proposal: took = 2.078969ms
172.18.1.12:2379 is healthy: successfully committed proposal: took = 2.209654ms
172.18.1.13:2379 is healthy: successfully committed proposal: took = 2.469605ms
```
# 二、CoreDNS 部署
- 三台机器上分别执行
```
mkdir /etc/coredns

(1)172.18.1.11上 
cat >/etc/coredns/Corefile<<-"EOF" 
.:53 {
  bind 172.18.1.11
      etcd {   # 配置启用etcd插件,后面可以指定域名,例如 etcd test.com {
        stubzones # 启用存根区域功能。 stubzone仅在位于指定的第一个区域下方的etcd树中完成
        path /coredns # etcd里面的路径 默认为/skydns，以后所有的dns记录就是存储在该存根路径底下
        endpoint http://172.18.1.11:2379 http://172.18.1.12:2379 http://172.18.1.13:2379# etcd访问地址，多个空格分开
        
        fallthrough # 如果区域匹配但不能生成记录，则将请求传递给下一个插件
    }
  hosts {
    172.18.1.14 dns.imysql.xyz
    172.18.1.15 test.imysql.xyz
    ttl 60
    reload 1m
    fallthrough
  }
  forward . /etc/resolv.conf
  cache 120
  reload 6s
  log
  errors
}
EOF

cat > /lib/systemd/system/coredns.service << EOF
[Unit]
Description=Coredns
After=network-online.target
[Service]
Type=simple
ExecStart=/usr/bin/coredns -conf /etc/coredns/Corefile
Restart=always
ExecStop=/bin/kill -9 $MAINPID
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=coredns
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable coredns
systemctl start coredns

（2）172.18.1.12
cat >/etc/coredns/Corefile<<-"EOF" 
.:53 {
  bind 172.18.1.12
      etcd {   # 配置启用etcd插件,后面可以指定域名,例如 etcd test.com {
        stubzones # 启用存根区域功能。 stubzone仅在位于指定的第一个区域下方的etcd树中完成
        path /coredns # etcd里面的路径 默认为/skydns，以后所有的dns记录就是存储在该存根路径底下
        endpoint http://172.18.1.11:2379 http://172.18.1.12:2379 http://172.18.1.13:2379# etcd访问地址，多个空格分开
        
        fallthrough # 如果区域匹配但不能生成记录，则将请求传递给下一个插件
    }
  hosts {
    172.18.1.14 dns.imysql.xyz
    172.18.1.15 test.imysql.xyz
    ttl 60
    reload 1m
    fallthrough
  }
  forward . /etc/resolv.conf
  cache 120
  reload 6s
  log
  errors
}
EOF

cat > /lib/systemd/system/coredns.service << EOF
[Unit]
Description=Coredns
After=network-online.target
[Service]
Type=simple
ExecStart=/usr/bin/coredns -conf /etc/coredns/Corefile
Restart=always
ExecStop=/bin/kill -9 $MAINPID
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=coredns
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable coredns
systemctl start coredns
(3)
cat >/etc/coredns/Corefile<<-"EOF" 
.:53 {
  bind 172.18.1.13
      etcd {   # 配置启用etcd插件,后面可以指定域名,例如 etcd test.com {
        stubzones # 启用存根区域功能。 stubzone仅在位于指定的第一个区域下方的etcd树中完成
        path /coredns # etcd里面的路径 默认为/skydns，以后所有的dns记录就是存储在该存根路径底下
        endpoint http://172.18.1.11:2379 http://172.18.1.12:2379 http://172.18.1.13:2379# etcd访问地址，多个空格分开
        
        fallthrough # 如果区域匹配但不能生成记录，则将请求传递给下一个插件
    }
  hosts {
    172.18.1.14 dns.imysql.xyz
    172.18.1.15 test.imysql.xyz
    ttl 60
    reload 1m
    fallthrough
  }
  forward . /etc/resolv.conf
  cache 120
  reload 6s
  log
  errors
}
EOF

cat > /lib/systemd/system/coredns.service << EOF
[Unit]
Description=Coredns
After=network-online.target
[Service]
Type=simple
ExecStart=/usr/bin/coredns -conf /etc/coredns/Corefile
Restart=always
ExecStop=/bin/kill -9 $MAINPID
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=coredns
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable coredns
systemctl start coredns


```

# 三、CoreDNS 测试
```
在机器172.18.1.17 上
cat >/etcd/resolv.conf<<-"EOF"
nameserver 172.18.1.11
nameserver 172.18.1.12
nameserver 172.18.1.13
EOF


nslookup dns.imysql.xyz
Server:		172.18.1.11
Address:	172.18.1.11#53

Name:	dns.imysql.xyz
Address: 172.18.1.14
```
# 四、接口操作CoreDNS
```
coredns只能使用etcd v3版本api添加的数据，etcdctl命令默认使用v2版本api，设置v3 api方法
export ETCDCTL_API=3
或者添加以下内容到环境变量 vim ~/.bash_profile:
export ETCDCTL_API=3
(1) A记录
etcdctl put /coredns/xyz/imysql/www '{"host":"172.18.1.14","ttl":10}'
OK
etcd的目录结构和域名是相反的，即上面表示域名：www.imysql.xyz
ttl值设置10s后，coredns每10s才会到etcd读取这个域名的记录一次

如果想添加多条记录，让coredns轮询，方法如下：
etcdctl put /coredns/xyz/imysql/www/1 '{"host":"172.18.1.14","ttl":10}'
etcdctl put /coredns/xyz/imysql/www/2 '{"host":"172.18.1.15","ttl":10}'

1和2可以自定义，比如a、b、c等
设置多个AAAA、CNAME等方法类似
添加/coredns/xyz/imysql/www/1、2后，请求www.imysql.xyz就不会再读取/coredns/xyz/imysql/www，
可以使用etcdctl del /coredns/xyz/imysql/www 删除值

    nslookup www.imysql.xyz
Server:		172.18.1.11
Address:	172.18.1.11#53

Name:	www.imysql.xyz
Address: 172.18.1.14
Name:	www.imysql.xyz
Address: 172.18.1.15
**注意：**如果想让取消设置的轮询值，需要删除/coredns/xyz/imysql/www/1与/coredns/xyz/imysql/www/2

（2） AAAA记录
% etcdctl put /coredns/xyz/imysql/www '{"host":"1002::4:2","ttl":10}'
OK
查询结果：

% dig -t AAAA @localhost +short www.imysql.xyz   
1002::4:2

(3) CNAME记录
% etcdctl put /coredns/xyz/imysql/www '{"host":"www.taobao.com","ttl":10}'
OK
查询结果：

% dig -t CNAME @localhost +short www.imysql.xyz
www.taobao.com.
这里cname设置成外部百度域名，按理说coredns应该也把这个cname记录继续解析成www.taobao.com的IP地址，但是经过测试发现请求www.imysql.xyz只能解析到CNAME：www.taobao.com，无法继续解析，原因未知，以后研究

(4)SRV记录
% etcdctl put /coredns/xyz/imysql/www '{"host":"www.taobao.com","port":80,"ttl":10}'
OK
SRV记录和CNAME记录类似，只是多了port，它们的添加方法其实可以通用
查询结果：

% dig -t SRV @localhost +short www.imysql.xyz
10 100 80 www.taobao.com.


(5)TXT记录
% etcdctl put /coredns/xyz/imysql/www '{"text":"Hello World!","ttl":10}'  
OK
查询结果：

% dig -t TXT @localhost +short www.imysql.xyz
"Hello World!"

```



# 五、CoreDNS 在etcd 存储查看
```
ETCDCTL_API=3  etcdctl   --endpoints 172.18.1.11:2379,172.18.1.12:2379,172.18.1.13:2379 get / --prefix
```















 
 
 
 
 
 
 
 
 
 
 
 
