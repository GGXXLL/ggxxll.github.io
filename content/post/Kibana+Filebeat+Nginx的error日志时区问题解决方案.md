---
title: Kibana+Filebeat+Nginx的error日志时区问题解决方案
tags: 
  - Kibana
date: 2022-08-24
categories:
  - 后端
---

使用Kibana + Filebeat + Nginx 进行日志监控，Nginx的error日志的时区提前了8个小时，解决方案如下：

原博： `https://www.colabug.com/2018/0425/2780769/`

### 步骤一
将 `/usr/share/filebeat/module/nginx/error/ingest/pipeline.json` 里面的
```
"date": {
  "field": "nginx.error.time",
  "target_field": "@timestamp",
  "formats": ["yyyy/MM/dd H:m:s"],
  {< if .convert_timezone >}"timezone": "{{ beat.timezone }}",{< end >}
  "ignore_failure": true
}
```
改为：
```
"date": {
  "field": "nginx.error.time",
  "target_field": "@timestamp",
  "formats": ["yyyy/MM/dd H:m:s"],
  "timezone" : "Asia/Shanghai",
  "ignore_failure": true
}
```

### 步骤二
```
删除旧的pipeline:

curl -XDELETE "http://localhost:9200/_ingest/pipeline/filebeat-6.7.0-nginx-error-pipeline"

# 重新设置
filebeat setup -e

# 查看是否生效
curl -XGET "http://localhost:9200/_ingest/pipeline/filebeat-6.7.0-nginx-error-pipeline"
```
**注意：** 一般来说索引格式固定，修改版本号（6.7.0）为自己的安装版本
