---
layout:  post
title:  在线数据服务网关实践经验总结
date:  2019-10-21 19:04 +08:00
categories:  [tech]
tags:  [Java,Gateway,HighPerformance,Concurrent]
toc: true
---

在线数据服务网关实践经验分享,一种NIO高性能的存储层网关,高性能统一在线数据平台实践
<!-- more -->

## 业务需求
实现对底层存储的隔离
高性能低延时
高可用，支持主备切换
支持故障转移
熔断
横向扩展

![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_img_ods_20200519105652.png)

##  技术方案设计

### 整体设计
按业务数据流及需求，将整个系统划分为网络层、业务接入层、业务服务层、数据存储层。除网络层外，其它层用微服务的方式开发各层的组件。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_20200519110709.png)


### RPC框架选择
综合对比`grpc` `thrift` `restful`在扩展性、跨语言兼容、性能、序列化大小等方面进行比对


|参考指标 |grpc|thrift|rest|
|:---|:---|:---|:---|
|开发语言|	跨语言	 |	跨语言 |	跨语言 |
|分布式(服务治理) | ×（自己集成service mesh,spring cloud) |× （自己集成service mesh,spring cloud)|  √ （spring cloud）|
|多序列化框架支持 | (只支持protobuf) |	× (thrift格式) |   × （json) |
|多种注册中心	| ×	  | × | √ |
|管理中心	| 	×	 |× | x |
|跨编程语言	|  √	 |√ | √ |
|性能（吞吐）|	√(官方banchmark c++ 单核平均 7w QPS) |	√(与grpc 相差不大，未找到官方资料) | x (	 restful使用的 http1.1协议，性能会比http2差1/2 )|
|底层通信协议(应用层)兼容性 | √(http2) |  x(socket)  | √ (通常http1.1) |

由于`restful`协议使用的 http1.1协议，性能会比http2差1/2，所以将其排除。

综合比对，选择grpc 作为RPC框架。

|参考指标 |	grpc	|thrift | 备注 |
|:---|:---|:---|:---|:---|
|性能指标        | XXXX  |  XXXX   | 在性能上thrift耗时<  grpc 耗时,差距在 0.84s/ 1wQPS,基本可以忽略  |
|成熟度和应用广度 |  XXXX | XXX | grpc 社区和版本更新上好于thrift,并且google facebook等内部都在用 |
|服务治理        |  XXXX | XXXX | 使用 haproxy 4层代理转发 |
|通信           |   XXXX | XXX | https2 全双工，由于thrift 通过socket + TFramedTransport的传输方式 |
| 异步/非阻塞    | XXXX | XXX |  grp 使用了基于channel 的异步非阻塞NIO ,thrift TFramedTransport python等未能支持异步|


###  确定软件模块划分
将整个系统划分为 `Gateway`,`Read Service`,`Write Service`以及周边服务等。
![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_20200519110047.png)


## 功能实现

###  接入层及gateway 功能实现

#### 接入层的高可用、横向扩展
使用DNS轮询的方式，绑定多个haproxy 节点的IP地址，并使用haproxy 4层代理机制。
![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200519151549.png)


#### 用户表级别限流
需要根据不同账号的对表的不同操作进行限流，因此根据username + request_table + request_optation 设计令牌桶，实现限流功能，还需要支持系统运行中的手动修改。


#### 服务发现


#### 服务负载均衡及熔断


//http://jira.jpushoa.com/browse/DP-18



## 数据业务层的功能实现

### 读写分离实现





### 参考
【grpc banchmark](https://grpc.io/docs/guides/benchmarking/)
 [开源RPC（gRPC/Thrift）框架性能评测](https://www.cnblogs.com/softidea/p/7232035.html)
 [分布式RPC框架性能大比拼](https://colobu.com/2016/09/05/benchmarks-of-popular-rpc-frameworks/)
