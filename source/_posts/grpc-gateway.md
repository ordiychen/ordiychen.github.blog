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

# 业务需求
实现一个统一在线数据服务，提供对在线数据(目前主要是保存在 HBase，后面再考虑其他存储)的读写功能。支持多语言接入、负载均衡、横向扩展、权限控制、流量(频率)控制、日志审计等功能。

![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_img_ods_20200519105652.png)

#  技术方案设计

## 整体设计
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
|:---|:---|:---|:---|
|性能指标        | XXXX  |  XXXX   | 在性能上thrift耗时<  grpc 耗时,差距在 0.84s/ 1wQPS,基本可以忽略  |
|成熟度和应用广度 |  XXXX | XXX | grpc 社区和版本更新上好于thrift,并且google facebook等内部都在用 |
|服务治理        |  XXXX | XXXX | 使用 haproxy 4层代理转发 |
|通信           |   XXXX | XXX | https2 全双工，由于thrift 通过socket + TFramedTransport的传输方式 |
| 异步/非阻塞    | XXXX | XXX |  grp 使用了基于channel 的异步非阻塞NIO ,thrift TFramedTransport python等未能支持异步|


###  确定软件模块划分
将整个系统划分为 `Gateway`,`Read Service`,`Write Service`以及周边服务等。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/img_20_20200519110047.png)

### 用户portal交互设计
给用户提供一个可以进行小量数据查询、下载的portal web，以替代使用不便的hbase shell
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713194645.png)


### iportal控制台的设计
iportal控制台需要实现对用户授权（表权限、API、频次限制)、停止授权、查看用户调用报表等功能。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713201819.png)

## 功能实现

###  接入层及gateway-service功能实现

#### 接入层的高可用、横向扩展
使用keepalived + haproxy 实现高可用和负载均衡。（后期由于运维规划，统一使用nginx stream做tcp4层代理)
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713192549.png)


#### 用户表级别限流
底层存储每个库以及表数据容量、机器配置等差一，为保证系统的可用性必须对用户的请求进行限流。
根据不同账号的对表的不同操作进行限流，以username + request_table + request_optation 作为key设计令牌桶，可实现按分钟/按秒进行平滑限流。

![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713192114.png)

#### 服务发现与负载均衡
综合比对了eureka,envovy方式后，基于易用性和团队技术储备选择了使用`eureka`。并结合极光自研发的微服务管理平台可以实现对服务的自动监控、告警、节点平滑上下线等功能。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713192905.png)

gateway进行请求转发的过程示意图：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713195602.png)

#### 服务熔断与降级
底层存储故障、后端服务节点遇到异常时无法立即恢复，此时将请求发给后端节点已经没有意义，对于这类场景使用Hystrix进行熔断降级处理，对于异常节点直接进行平滑下线从eureka注册中心下线。



## 数据业务层的功能实现

### 读写分离实现
对于大流量的业务，使用Kafka MQ读写分离提高性能、平滑峰值。 
读写分离设计示意图：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713210750.png)

 数据写入过程：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713205115.png)

## 存储层Hbase主备切换
Hbase异常会出现大量查询超时和相应的Error，当失败达到一定阈值需要进行Hbase主备库切换。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713214008.png)
同时也需要支持手动切换hbase，对于手动切换，采用通过修改apollo 配置中心的对应的配置实现集群的手动切换：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713214629.png)

## 日志审计和统计和报表功能实现
在`gateway-service`层增加`logInterceptor`实现对用户的请求的进站和出站数据的记录，利用ELK来实现整个系统的日志收集和搜索。
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713215643.png)

使用Kibana进行日志查询：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713220032.png)


## 总结
在接入层使用负载均衡，在服务层通过微服务的架构方式方式保证整个架构的高可用和可伸缩性，对于gateway-service、hbase-service、writer等微服务使用全异步处理的方式提高了单个节点的性能，减小各节点对于整体耗时的影响。
各节点的最大QPS负载：

|组件 |	单节点(8C16G)测试实际QPS  | GC状况 |
|:---|:---|:---|
|gateway  | 	15K |		 当流量上升到12K+ 每分钟GC次数>20 ,每分钟GC总耗时>5s 需要优化  |
|writer-pre | 	33K+ |		当流量上升到33K+ 每分钟GC次数>4,每分钟GC总耗时>0.05s 正常|
|writer	|	20K+	|当流量上升到20K+ 每分钟GC次数>14 ,每分钟GC总耗时>0.42s 正常 |


线上生产环境每日流量：
![image](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200713212836.png)


## 可以继续优化
 - grpc 框架自带的loadblance策略与eureka策略整合
 - grpc支持stream模式，可以探索基于stream的转发

# 参考
-[grpc banchmark](https://grpc.io/docs/guides/benchmarking/)
- [开源RPC（gRPC/Thrift）框架性能评测](https://www.cnblogs.com/softidea/p/7232035.html)
- [分布式RPC框架性能大比拼](https://colobu.com/2016/09/05/benchmarks-of-popular-rpc-frameworks/)
