---
layout:     post
title:     Redis _conf最优配置
subtitle:   最佳实践
date:       2019-05-09
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - redis
---

## Redis _conf最优配置
````shell
bind 0.0.0.0 127.0.0.1
#绑定IP
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize yes
#是否开启后台运行
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/opt/module/redis/logs/redis.log"
databases 16
#开启rbd持久化
save ""

save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /home/redisdata
# slaveof <masterip> <masterport>
#master 节点是什么
# masterauth <master-password>
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
#是否开启无盘复制
repl-diskless-sync-delay 5
#无盘复制延迟时长
# repl-timeout 60
#主从同步默认多少s 超时，超时自动断开同步
repl-disable-tcp-nodelay no

#同城机房主从复制建议开启 高容灾
repl-backlog-size 200mb
#主从同步offset记录大小，默认1mb，网络不稳定时容易导致从库全量同步，所以改为默认200mb
slave-priority 100
requirepass foobared
#redis 密码设置

rename-command CONFIG ""

maxmemory 3G
#redis最大内存限制
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
# appendfsync no
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 2gb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""

hash-max-ziplist-entries 512
hash-max-ziplist-value 64

list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512

zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000

activerehashing yes

client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 1024mb 256mb 60
#客户端写入缓冲区，当60秒内消耗持续大于60mb，或者直接超过256MB，则直接关闭复制客户端连接
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

````
