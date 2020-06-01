---
layout:  post
title:   记一次DNS异常导致的 `ECONNREFUSED`  问题
date:  2019-10-21 19:04 +08:00
categories:  [values]
tags:  [DNS,TCP/IP,NODEJS]
excerpt:  记一次DNS异常导致的 `ECONNREFUSED`  问题
---


* content
{:toc}


### 问题描述
`node.js` 程序出现 ` Error: connect ECONNREFUSED`  和 `getaddrinfo ENOTFOUND` 。
`ENOTFOUND` 官方定义 :  (DNS lookup failed): Indicates a DNS failure of either EAI_NODATA or EAI_NONAME. This is not a standard POSIX error.
这个问题一般是DNS 查询失败导致的。

`ECONNREFUSED`: (Connection refused): No connection could be made because the target machine actively refused it. This usually results from trying to connect to a service that is inactive on the foreign host.
无法建立连接，因为目标计算机主动拒绝了该连接。 这通常是由于尝试连接到外部主机上处于非活动状态的服务而导致的。 (这个也可能是DNS解析的 IP地址不对，导致远程主机无法连接)


详细日志如下：
```
2020-04-21 11:26:19 [PicGo ERROR] RequestError: Error: connect ECONNREFUSED 13.250.168.23:443
------Error Stack Begin------
RequestError: Error: connect ECONNREFUSED 13.250.168.23:443
    at new RequestError (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/errors.js:14:15)
    at Request.plumbing.callback (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/plumbing.js:87:29)
    at Request.RP$callback [as _callback] (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/plumbing.js:46:31)
    at self.callback (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request/request.js:185:22)
    at Request.emit (events.js:182:13)
    at Request.onRequestError (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request/request.js:881:8)
    at ClientRequest.emit (events.js:182:13)
    at TLSSocket.socketErrorListener (_http_client.js:391:9)
    at TLSSocket.emit (events.js:182:13)
    at emitErrorNT (internal/streams/destroy.js:82:8)
-------Error Stack End-------
```

```
2020-05-18 18:34:22 [PicGo WARN] failed
2020-05-18 18:34:22 [PicGo ERROR] RequestError: Error: getaddrinfo ENOTFOUND api.github.com
------Error Stack Begin------
RequestError: Error: getaddrinfo ENOTFOUND api.github.com
    at new RequestError (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/errors.js:14:15)
    at Request.plumbing.callback (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/plumbing.js:87:29)
    at Request.RP$callback [as _callback] (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request-promise-core/lib/plumbing.js:46:31)
    at self.callback (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request/request.js:185:22)
    at Request.emit (events.js:200:13)
    at Request.onRequestError (/Applications/PicGo.app/Contents/Resources/app.asar/node_modules/request/request.js:881:8)
    at ClientRequest.emit (events.js:200:13)
    at TLSSocket.socketErrorListener (_http_client.js:402:9)
    at TLSSocket.emit (events.js:200:13)
    at emitErrorNT (internal/streams/destroy.js:91:8)
-------Error Stack End-------
```


####  排查问题
对于`RequestError: Error: connect ECONNREFUSED 13.250.168.23:443` ERROR, 可能是某个域名的IP地址，使用dig反向解析IP地址对应的域名：
```bash
$ dig -x 13.250.168.23 +short
api.github.com

# 使用梯子代理进行解析
$ proxychain4 dig -x 13.250.168.23 +short
ec2-13-250-168-23.ap-southeast-1.compute.amazonaws.com.
```

```
#公司 设置公网DNS是： 114.114.115.115，可以从 lan 的状态上看到
dig @114.114.115.115 api.github.com +trace
```

![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_img_20200519094710.png)


解析到的`13.250.168.23` 这个IP，实际是不可访问的,`ping `和`telnet `都不通。
指定DNS`119.29.29.29`进行排查:

![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_img_20200519095335.png)


修改本地`hosts` 文件，进行测试，问题修复。
```
$ sudo echo "54.169.195.247 api.github.com" >> /etc/hosts
#或者在 /etc/hosts 添加一行:54.169.195.247 api.github.com

$ curl https://api.github.com
HTTP/1.1 200 OK
date: Tue, 19 May 2020 01:57:31 GMT
content-type: application/json; charset=utf-8
```


### 后续
DNS有问题，排查起来比较耗时间，Mianland CN , 建议使用 119.29.29.29 （ 腾讯） ， 海外用google DNS：8.8.8.8很稳妥。
另外家中的路由器也建议不要使用ISP提供的默认DNS，减少ISP广告的也更稳定。


### 参考
[node.jd Document Error List]([https://nodejs.org/api/fs.html)
[dig man](https://linux.die.net/man/1/dig)
[linux dig 命令](https://www.cnblogs.com/sparkdev/p/7777871.html)
