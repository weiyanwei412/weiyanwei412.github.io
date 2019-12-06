---
layout:     post
title:      ELK收集nginx日志并在grafanna展示

subtitle:   实战
date:       2019-12-06
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - ELK
    - elastsearcher 
---
## ELK收集nginx日志并在grafanna展示
# 一、官方网站下载
（1）安装JDK
```
yum install -y java-1.8.0-openjdk-devel  # 安装1.8或1.8以上版本
```
(2)下载elasticsearchhttps://www.elastic.co/cn/downloads/elasticsearch，是ELasticsearch的官方站点，如果需要下载最新的版本，进入官网下载即可。可以下载到本地电脑然后再导入CentOS中，也可以直接在CentOS中下载。
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.4.rpm
```
(3)安装elasticsearch
```
rpm -ivh elasticsearch-6.5.4.rpm
```
(4)配置目录
安装完毕后会生成很多文件，包括配置文件日志文件等等，下面几个是最主要的配置文件路径
```
/etc/elasticsearch/elasticsearch.yml                            # els的配置文件
/etc/elasticsearch/jvm.options                                  # JVM相关的配置，内存大小等等
/etc/elasticsearch/log4j2.properties                            # 日志系统定义

/usr/share/elasticsearch                                        # elasticsearch 默认安装目录
/var/lib/elasticsearch                                          # 数据的默认存放位置
```
(5)创建用于存放数据与日志的目录
```
mkdir -p /opt/elasticsearch/data
mkdir -p /opt/elasticsearch/log
chown -R elasticsearch.elasticsearch /opt/elasticsearch/*
```
(6)集群配置
```
vim /etc/elasticsearch/elasticsearch.yml

cluster.name: imysql-els                               # 集群名称
node.name: els-node1                               # 节点名称，仅仅是描述名称，用于在日志中区分

path.data: /data/elasticsearch/data                 # 数据的默认存放路径
path.logs: /data/elasticsearch/log                  # 日志的默认存放路径

network.host: 172.18.1.10                        # 当前节点的IP地址
http.port: 9200                                    # 对外提供服务的端口，9300为集群服务的端口
#添加如下内容
#culster transport port
transport.tcp.port: 9300
transport.tcp.compress: true

discovery.zen.ping.unicast.hosts: ["172.18.1.10", "172.18.1.11","172.18.1.12"]       
# 集群个节点IP地址，也可以使用els、els.shuaiguoxia.com等名称，需要各节点能够解析

discovery.zen.minimum_master_nodes: 2              # 为了避免脑裂，集群节点数最少为 半数+1
```
(7)授权并启动elasticsearch
```
chown -R   elasticsearch.     /data/elasticsearch
systemctl start  elasticsearch
systemctl enable  elasticsearch

curl -i "http://172.18.1.10:9200"
```

# 二、 配置收集nginx日志
（1）修改nginx日志格式
```
log_format  main  '{"@timestamp":"$time_iso8601",'
                  '"@source":"$server_addr",'
                  '"hostname":"$hostname",'
                  '"ip":"$remote_addr",'
                  '"client":"$remote_addr",'
                  '"request_method":"$request_method",'
                  '"scheme":"$scheme",'
                  '"domain":"$server_name",'
                  '"referer":"$http_referer",'
                  '"request":"$request_uri",'
                  '"args":"$args",'
                  '"size":$body_bytes_sent,'
                  '"status": $status,'
                  '"responsetime":$request_time,'
                  '"upstreamtime":"$upstream_response_time",'
                  '"upstreamaddr":"$upstream_addr",'
                  '"http_user_agent":"$http_user_agent",'
                  '"https":"$https"'
                  '}';


logstash  

cat logstash.conf
input {
    file {
        path => [ "/data/tengine/*.log" ]
        ignore_older => 0
    codec => json
    }
}

filter {
    mutate {
      convert => [ "status","integer" ]
      convert => [ "size","integer" ]
      convert => [ "upstreatime","float" ]
      remove_field => "message"
    }
    geoip {
        source => "ip"
    }


}
output {
    elasticsearch {
        hosts => ["http://172.18.1.10:9200"]
        index => "prod-nginx-172.18.1.66-%{+YYYY-MM-dd}"
    }
#    stdout {codec => rubydebug}
}




```
(2) 安装grafanna
```
wget https://dl.grafana.com/oss/release/grafana-6.5.1-1.x86_64.rpm
yum localinstall grafana-6.5.1-1.x86_64.rpm

systemcrl start grafana-server

systemcrl enable grafana-server
```
(2)安装grafanna  展示nginx日志插件
```
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install  grafana-worldmap-panel
systemcrl restart grafana-server

```
(3)修改地图展示js
```
cd /var/lib/grafana/plugins/grafana-worldmap-panel
grafana-worldmap-panel\src\worldmap.ts 
grafana-worldmap-panel\dist\module.js 
grafana-worldmap-panel\dist\module.js.map 
将：https://cartodb-basemaps-{s}.global.ssl.fastly.net/light_all/{z}/{x}/{y}.png
替换成：http://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}.png

将：https://cartodb-basemaps-{s}.global.ssl.fastly.net/dark_all/{z}/{x}/{y}.png 
替换成：http://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}.png 

systemcrl restart grafana-server
```
(4)grafanna增加nginx数据源
<p><img width="1300" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/elk01.png" height="600" alt=""></p>

<p><img width="1300" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/elk02.png" height="600" alt=""></p>

<p><img width="1300" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/elk03.png" height="600" alt=""></p>


<p><img width="1300" src="https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/elk04.png" height="600" alt=""></p>

# 完毕







