---
layout:     post
title:      局域网DNS 之CoreDNS 集群部署
subtitle:   实战
date:       2019-12-20
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
# etcd 集群部署
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
  --name=etcd1 \
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
  --name=etcd1 \
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
# CoreDNS 部署
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

nohup /usr/bin/coredns -conf /etc/coredns/Corefile > /tmp/coredns.log 2>&1 &

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

nohup /usr/bin/coredns -conf /etc/coredns/Corefile > /tmp/coredns.log 2>&1 &
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

nohup /usr/bin/coredns -conf /etc/coredns/Corefile > /tmp/coredns.log 2>&1 &


```

# CoreDNS 测试
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
















 
 
 
 
 
 
 
 
 
 
 
 
