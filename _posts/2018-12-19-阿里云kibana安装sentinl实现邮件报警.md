---
layout:     post
title:      阿里云kibana安装sentinl实现邮件报警
subtitle:   sentinl
date:       2018-12-19
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - sentinl
    - ELK
    - kibana
---

## 阿里云kibana安装sentinl实现邮件报警
(1) sentinl github地址：https://github.com/sirensolutions/sentinl

(2) sentinl github个版本地址：https://github.com/sirensolutions/sentinl/releases

(3) 安装sentinl
```
1、 下载自己需要的版本，我这里kibana是6.5.2 我下载了sentinl-v6.5.2.zip
2、 安装sentinl /usr/share/kibana/bin/kibana-plugin install file:///`pwd`/sentinl-v6.5.2.zip
3、 重启kibana systemctl restart kibana
```
(4)访问kibana  http://192.168.100.15:5601

![Alt text](https://github.com/weiyanwei412/weiyanwei412.github.io/blob/master/img/kibana-sentl.png)

(5)配置邮件报警
```
1、配置kibana邮箱发送
修改kibana配置文件
cat >>/etc/kibana/kibana.yml<<-EOF
sentinl:
  settings:
    email:
      active: true
      user: XXX@163.com
      password: 327fbe5aca
      host: smtp.exmail.qq.com
      port: 465
      ssl: true
EOF
阿里云调用邮箱 重点是增加ssl 调用 465端口
重启kibana服务
systemctl restart kibana

2、配置setmail
yum install mail cyrus-sasl-*

cat >>/etc/mail.rc<<-EOF
set from=warn_info@company.com.cn
set smtp=smtps://smtp.exmail.qq.com:465
set smtp-auth-user=warn_info@company.com.cn
set smtp-auth-password=327fbe5aca
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb
EOF

3、配置业务日志邮件报警
sentinl 页面新增Watcher Advanced

{
  "actions": {
    "email_html_alarm_0050c91e-592f-4801-9fbe-9d9d910f8472": {
      "name": "email html alarm",
      "throttle_period": "1m",
      "email_html": {
        "to": "ops@conmpany.com.cn",
        "from": "warn_info@company.com.cn",
        "stateless": false,
        "subject": "生产环境nginx 5XX告警",
        "priority": "high",
        "html": "<p>各位好,</p>\n<p>本次日志扫描5分钟内发现{{payload.hits.total}}条5XX Error信息，请登录kibana查询具体错误信息。 <i>http://192.168.100.15:8449</i></p>\n<div style=\"color: grey\">\n  <hr>\n  <p>本次日志扫描采用以下策略:</p>\n  <ul><li>{{watcher.condition.script.script}}</li></ul>\n</div>"
      }
    }
  },
  "input": {
    "search": {
      "request": {
        "index": [
          "prod-nginx-*"
        ],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "http_code": {
                      "gt": "500"
                    }
                  }
                }
              ],
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m",
                      "lt": "now"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "script": "payload.hits.total > 1"
    }
  },
  "transform": {},
  "trigger": {
    "schedule": {
      "later": "every 5 minutes"
    }
  },
  "disable": false,
  "report": false,
  "title": "生产环境日志nginx 5XX告警",
  "save_payload": false,
  "spy": false,
  "impersonate": false
}

保存，在Alarms栏目 会看到相应的报警
```


## end










