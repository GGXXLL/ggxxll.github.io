---
title: Supervisor使用
tags: 
  - Supervisor
date: 2022-08-24
categories:
  - 后端
---

supervisor安装使用, 并配置服务崩溃邮件报警

## 环境
系统: Centos7, python3

## 安装依赖: 
```shell
yum install -y supervisor sendmail mailx && pip3 install superlance
```

## 配置邮件:

1. `vim /etc/mail.rc`, 然后添加如下内容:  
```yaml
# 发件人邮箱
set from=xxx@xxx.com  
# smtp服务
set smtp=smtps://smtp.xxx.com:465  
# 用户名
set smtp-auth-user=xxx@xxx.com  
# 密码
set smtp-auth-password=xxx  
set ssl-verify=ignore
```
2. 测试邮件: `echo 'this is test'| /usr/bin/mail -s 'xxxxx' xxx@xx.com`

## 配置项目:

编辑项目的配置文件, `vim /etc/supervisord.d/xxx.ini`

```ini
# 项目相关配置
[program:projectName]
# 设置命令在指定的目录内执行
directory=projectPath
# 这里为您要管理的项目的启动命令
command=projectRunCmd
# 以哪个用户来运行该进程
user=root
# supervisor 启动时自动该应用
autostart=true
# 进程退出后自动重启进程
autorestart=true
# 进程持续运行多久才认为是启动成功
startsecs=2
# 重试次数
startretries=3
# stderr 日志输出位置
stderr_logfile=path/stderr.log
# stdout 日志输出位置
stdout_logfile=path/stdout.log
```

```ini
# 报警邮件相关配置
[eventlistener:crashmail]
command=crashmail -p senddemo -s "echo 'projectName crashed!!'| /usr/bin/mail -s 'projectName' xxx@xx.com,xxx@xx.com"
events=PROCESS_STATE_EXITED
## stderr 日志输出位置
stderr_logfile=path/crashmail/stderr.log
## stdout 日志输出位置
stdout_logfile=path/crashmail/stdout.log
```


## 相关命令: 
```shell
supervisord -c /etc/supervisord.conf # 启动supervisor, 然后才可可以使用supervisorctl
supervisorctl stop program_name  # 停止某一个进程，program_name 为 [program:name] 里的 name
supervisorctl start program_name  # 启动某个进程
supervisorctl restart program_name  # 重启某个进程
supervisorctl stop groupworker:  # 结束所有属于名为 groupworker 这个分组的进程 (start，restart 同理)
supervisorctl stop groupworker:name1  # 结束 groupworker:name1 这个进程 (start，restart 同理)
supervisorctl stop all  # 停止全部进程，注：start、restartUnlinking stale socket /tmp/supervisor.sock
supervisorctl reload  # 载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程
supervisorctl update  # 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启
```