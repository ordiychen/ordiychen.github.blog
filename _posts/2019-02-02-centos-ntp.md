---
layout:  post
title:  Centos7 设置时间
date:  2019-02-02 19:04 +08:00
categories:  [tech]
tags:  [OS,Centos,NTP]
---

* content
{:toc}


### 设置时间
```
#查看时区设置 默认是 America/NewYork
$ timedatectl

# 查看所有 支持的时区名
$ timedatectl list-timezones

# 设置 将硬件时钟调整为与本地时钟一致, 0 为设置为 UTC 时间
$ timedatectl set-local-rtc 1

# 设置系统时区为HongKong
timedatectl set-timezone Asia/Hong_Kong

```

### 校准时间
ntp校准时间
```
#install ntp
$ sudo yum install ntp -y

#设置ntp server,NIST Internet Time Servers , NTP Pool Project
$ sudo ntpdate time.nist.gov

#或者 asia.pool.ntp.org 延迟 0.05ms 左右，推荐使用
$ sudo ntpdate asia.pool.ntp.org

```
推荐使用 NTP Pool Project,部分节点使用GPS信号来设置时间，准确度很高。
链接：https://www.ntppool.org

PS：
NTP 漏洞问题：
https://www.securityfocus.com/bid/94452






