---
layout:     post
title:     repcached+keepalived的高可用memcached集群
subtitle:   最佳实践
date:       2019-07-17
author:     dbstack
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - memcached
---
## repcached+keepalived的高可用memcached集群
# 一 简介
- Repcached是用来实现 Memcached 复制功能的一个工具。它所构建的主从方案是一个单主单从的方案，不支持多主多从。但是，它的特点是主从两个节点可以互相读写，从而达到互相同步的效果

- 注意事项：如果主节点坏掉，从节点会很快侦测到连接断开，然后它会自动切换到监听状态( listen)从而成为主节点，并且等待新的从节点加入
如果原来挂掉的主节点恢复之后，我们只能人工手动以从节点的方式去启动。原来的主节点并不能抢占成为新的主节点，除非新的主节点挂掉。这也就意味着，基于 Repcached 实现的 Memcached 主从，针对主节点并不具备抢占功能
# 二 安装Repcached
（1）安装libevent：
- 下载地址：http://libevent.org/
```
[root@dep-test-s03 ~]# cd /usr/local/src
[root@dep-test-s03 src]# tar -zxvf libevent-2.1.8-stable.tar.gz
[root@dep-test-s03 src]# cd libevent-2.1.8-stable
[root@dep-test-s03 libevent-2.1.8-stable]# ./configure --prefix=/usr/local/
[root@dep-test-s03 libevent-2.1.8-stable]# make && make install
```
- 查看是否安装成功：
``` 
[root@dep-test-s03 libevent-2.1.8-stable]# ls -lh /usr/local/lib/libeven*
```
![Alt text](https://raw.githubusercontent.com/weiyanwei412/weiyanwei412.github.io/master/img/memcache1.png)
（2）安装memcached：
- 下载地址：http://memcached.org
```
[root@dep-test-s03 ~]# cd /usr/local/src
[root@dep-test-s03 src]# tar -zxvf memcached-1.4.20.tar.gz
[root@dep-test-s03 src]# cd memcached-1.4.20
[root@dep-test-s03 memcached-1.4.20]# ./configure --with-libevent=/usr/local
[root@dep-test-s03 memcached-1.4.20]# make && make install
```
（3）安装repcached：
- 下载地址：https://sourceforge.net/projects/repcached/
```
[root@dep-test-s03 ~]# cd /usr/local/src
[root@dep-test-s03 src]# tar -zxvf memcached-1.2.8-repcached-2.2.1.tar.gz
[root@dep-test-s03 src]# cd memcached-1.2.8-repcached-2.2.1
[root@dep-test-s03 memcached-1.2.8-repcached-2.2.1]# ./configure --enable-replication --program-transform-name=s/memcached/repcached/
[root@dep-test-s03 memcached-1.2.8-repcached-2.2.1]# make && make install
```
- make的时候发现爆了以下错误：
![Alt text](https://raw.githubusercontent.com/weiyanwei412/weiyanwei412.github.io/master/img/memcache2.png)
- 解决方案是修改/usr/local/src/memcached-1.2.8-repcached-2.2.1/memcached.c文件的以下内容：
```
/* FreeBSD 4.x doesn’t have IOV_MAX exposed. */
#ifndef IOV_MAX
#if defined(__FreeBSD__) || defined(__APPLE__)
# define IOV_MAX 1024
#endif
#endif
```
更改为：
```
/* FreeBSD 4.x doesn't have IOV_MAX exposed. */
#ifndef IOV_MAX
# define IOV_MAX 1024
#endif
```
- 改完之后再次make && make install即可
# 三 启动配置
```
[root@dep-test-s03 init.d]# useradd memcached -s /bin/false
[root@dep-test-s03 init.d]# mkdir -p /var/run/memcached
[root@dep-test-s03 init.d]# cd /var/run/
[root@dep-test-s03 init.d]# chown -R memcached:memcached memcached
```
- 启动命令
```
/usr/local/bin/repcached -d -p 11211 -u memcached -m 1024 -c 4096 -t 4 -P /var/run/memcached/memcached.pid -v >> /tmp/memcached.log 2>&1
查看日志：
tail -f /tmp/memcached.log
```
（2）启动salve节点memcached：
```
/usr/local/bin/repcached -d -x 10.101.100.3 -p 11211 -u memcached -m 1024 -c 4096 -t 4 -P /var/run/memcached/memcached.pid -v >> /tmp/memcached.log 2>&1
查看日志：
tail -f /tmp/memcached.log
```
注：-x参数表示master节点地址
附：关于memcached启动命令的介绍：

-d：启动一个守护（daemon）进程
-m：分配给 memcache 使用的内存数量，单位是MB
-u：运行 memcache 的用户
-l：监听的服务器IP地址
-p：设置 memcache 监听的端口，端口最好大于1024
-c：设置 memcache 最大运行的并发连接数，默认是1024
-t：设置最大线程数
-v：输出日志，-vv -vvv也是输出日志，不过其详细程度依次递增
-P：设置保存 memcache 的pid文件地址。设置pid文件的好处在于方便结束memcache 进程，如果存在pid文件，那么我们就可以通过如下命令快速结束 memcached 进程：kill `cat /var/run/memcached/memcached.pid`
（3）测试master节点和slave节点数据是否同步：
```
[root@dep-test-s03 ~]# telnet 10.101.100.3 11211
Trying 10.101.100.3…
Connected to 10.101.100.3.
Escape character is ‘^]’.
set hello 0 0 5
world
STORED

注：这里的“set hello 0 0 5”表示设置键值对，其KEY为“hello”，存储时间为永久，VALUE的字节数为5（world的长度为5）
```
ii）在slave节点（10.101.100.4）执行：
获取master节点设置的参数“hello”，并设置新参数“ping”
```
[root@dep-test-s04 lib]# telnet 10.101.100.4 11211
Trying 10.101.100.4…
Connected to 10.101.100.4.
Escape character is ‘^]’.
get hello
VALUE hello 0 5
world
END
set ping 0 0 4
pong
STORED
```
注意事项：
由于memcached的主/从没有抢占功能，因此如果主节点因为某种原因挂掉，再次恢复之后，只能作为现有主节点（也就是原来的从节点，在主节点挂掉的时候自动从从节点升级为主节点）的从节点启动，如：
```
/usr/local/bin/repcached -d -x 10.101.100.4 -p 11211 -u memcached -m 1024 -c 4096 -t 4 -P /var/run/memcached/memcached.pid -v >> /tmp/memcached.log 2>&1
```
# 四 安装keepalived 并设置监控
资源准备：
- 两台主机：10.101.100.3、10.101.100.4
- 待使用的虚IP（VIP）：10.101.100.5

（1）在两台服务器上面分别安装keepalived
```
yum install -y keepalived
```
(2) 10.101.100.3 keepalived.conf 配置文件
```
 global_defs {
     router_id Load-Balancer-1
}
 vrrp_script check_mem {
     script "/etc/keepalived/check_mem.sh"
     interval 2
     fall 3
 }
vrrp_instance haproxy {
        state BACKUP
        interface ens160
        virtual_router_id 110
        priority 150
        advert_int 1
        nopreempt
        garp_master_delay 10
    track_interface {
     ens160
     }
 authentication {
     auth_type PASS
     auth_pass flaginfo
 }
 virtual_ipaddress {
    10.101.100.5
 }
 track_script {
      check_mem
 }
          notify_master "/bin/bash /etc/keepalived/notify.sh Master"
          notify_backup "/bin/bash /etc/keepalived/notify.sh Backup"
          notify_fault "/bin/bash /etc/keepalived/notify.sh Fault"
          notify_stop "/bin/bash /etc/keepalived/notify.sh Stop"
}
EOF

```
(2) 10.101.100.4 keepalived.conf 配置文件
```
cat >/etc/keepalived/keepalived.conf<<-EOF
global_defs {
   router_id Load-Balancer-2
}
 vrrp_script check_mem {
    script "/etc/keepalived/check_mem.sh"
    interval 2
    fall 3
 }
vrrp_instance haproxy {
    state BACKUP
    interface ens160
    virtual_router_id 110
    priority 100
    advert_int 1
    nopreempt
    garp_master_delay 10
 track_interface {
  ens160
 }
 authentication {
  auth_type PASS
  auth_pass flaginfo
 }
 virtual_ipaddress {
    10.101.100.5
 }
 track_script {
    check_mem
 }
      notify_master "/bin/bash /etc/keepalived/notify.sh Master"
      notify_backup "/bin/bash /etc/keepalived/notify.sh Backup"
      notify_fault "/bin/bash /etc/keepalived/notify.sh Fault"
      notify_stop "/bin/bash /etc/keepalived/notify.sh Stop"
}
EOF
```

```
yxtuser@dep-test-s03:~$ cat /etc/keepalived/check_mem.sh 
#!/bin/bash
status=`echo "stats" |nc 127.0.0.1 11211|grep pid|awk '{print $3}'`
if [ "$status" > 0 ]; then 
        exit 0
else
        exit -1
fi
```

```
yxtuser@dep-test-s03:~$ cat /etc/keepalived/notify.sh 
#!/bin/bash
echo $(date +%F-%T)" Change to $1" >> /etc/keepalived/notify.log
```

(4)分别在 10.101.100.3， 10.101.100.4启动keepalived
```
 在10.101.100.3上
root@dep-test-s03:~#systemctl start keepalived

 在10.101.100.4上
root@dep-test-s04:~#systemctl start keepalived


```
（5）可以看到在10.101.100.3 挂载上了vip 
```
root@dep-test-s03:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:85:ed:b6 brd ff:ff:ff:ff:ff:ff
    inet 10.101.100.3/16 brd 10.101.255.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet 10.101.100.5/32 scope global ens160
       valid_lft forever preferred_lft forever
3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:85:4f:9f brd ff:ff:ff:ff:ff:ff
root@dep-test-s03:~# 
```
（5）应用程序直接调用vip 即可：10.101.100.5:11211

