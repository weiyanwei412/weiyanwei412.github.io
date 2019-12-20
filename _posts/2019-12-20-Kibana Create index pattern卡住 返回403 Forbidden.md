---
layout:     post
title:      Kibana Create index pattern 卡住 返回403 Forbidden
subtitle:   实战
date:       2019-12-20
author:     dbstack
header-img: img/post-bg-mma-4.jpg
catalog: true
tags:
    - ELK
    - kibana
    - Elastsearcher 
---
## Kibana Create index pattern 卡住 返回403 Forbidden
# 一、现象
（1）kibana添加index pattern卡住，通过浏览器查看请求返回状态为403 Forbidden

（2）通过curl模拟请求响应如下：
```
 curl -XPOST -H 'Content-Type: application/json' -H 'kbn-version: 6.5.2' http://127.0.0.1:5600/api/saved_objects/index-pattern
 -d '{"attributes": {"title": "index-prefix-*", "timeFieldName": "@timestamp"}}'
 
{"message":"blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];:
[cluster_block_exception] blocked by: [FORBIDDEN/12/index read-only / allow delete (api)];
","statusCode":403,"error":"Forbidden"}
```
# 二、解决办法 设置es索引参数
```
(1) 
curl -XPUT -H 'Content-Type: application/json' http://127.0.0.1:9200/_settings -d'
{
    "index": {
        "blocks": {
            "read_only_allow_delete": "false"
        }
    }
}'
(2)

 curl -XPUT -H 'Content-Type: application/json' http://127.0.0.1:9200/$index_name/_settings -d'
{
    "index": {
        "blocks": {
            "read_only_allow_delete": "false"
        }
    }
}'
```
# 问题解决
