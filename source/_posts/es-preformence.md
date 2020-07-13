---
layout:  post
title:  Elasticsearch5.2性能测试
date:  2019-10-21 19:04 +08:00
categories:  [tech]
tags:  [Elasticsearch,NoSQL,分布式]
---


Elasicseach是当前非常流行的分布式搜索引擎框架,是大数据处分析领域的一把利器。但是Elasticsearch使用传统的HDD存储的搜索性能并不出色,使用SSD存储数据可以大幅提升性能,搜索一个1亿文档(存储空间约40G)的索引在5k,1w QPS下50ms内完成查询的保证率分别可以达到99.75%和96.0%,相对于使用HDD存储来说性能提升了一个数据量级。
<!-- more -->

### 背景描述
测试Elasticsearch存储8亿文档(约350G)的情况下，Es的使用docId,routing和term search的性能。
硬件资源:

<div class="datatable-begin"></div>

|节点序号| CPU核心数据| 内存| 磁盘|
|:---|:---|:---|:---|
|0   |16   |48G| SSD 500G|
|1   |16   |48G| SSD 500G|
|2   |16   |48G| SSD 500G|

<div class="datatable-end"></div>

*注意： 同一个IDC内部, 1Gbps网络以上

软件版本:

|x| 版本| 备注|
|:---|:--- |:---|
|centos| 7.0| 操作系统|
|Elasticsearch | 5.2.2 | |
|java|bit64 1.8.111| jdk|


### Es数据存储设计
考虑到存储的数据不需要进行分词和评分查询,所以在Es数据结构和存储上设计了以下几点:
- 为充分利用Es的查询性能，在存储上将字段设计为[```keyword```类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html)。同时禁用了```_all```(请查阅[_all字段的用途和原理](http://cwiki.apachecn.org/display/Elasticsearch/_all+field)),设置了``` "dynamic": strict```;
- 在数据写入阶段不设置分片的副本，提升写入性能


索引的结构如下：
```
查询时
#f1 字段数据具有唯一性,将其作为为docId
#f2 substr(0,6) 作为routing
#f3 是被查询的数据,目前业务较少通过该字段进行查询

PUT /x_y_data
{
    "mappings": {
    "_all": {
        "enabled": false
      },
        "rbtype": {
            "dynamic": strict,
            "properties": {
                "f1": {
                    "type": "keyword",
                    "norms":false
                },
                "f2": {
                    "type": "keyword",
                    "norms":false
                },
                "f3": {
                    "type": "keyword",
                    "norms":false
                }
            }
        }
    },"settings" : {
      "index" : {
        "number_of_shards" : "5",
        "number_of_replicas" :"0",
      }
    }
}
```


### 基准测试工具
Elsticesearch使用了netty作为通信框架，很好的支持了非阻塞异步通信，所以我们选择vertx-java[参阅资料](http://vertx.io)作为异步框架,设计基准测试工具
![image](http://oybm9jmsf.bkt.clouddn.com/image.png)

*[项目源代码](http://oybm9jmsf.bkt.clouddn.com/file/src-data/es5-banchmarks.rar)

### 测试结果
接入promethesu监控请求耗时和QPS

* 1000 QPS 5min,50ms内完成的保证率99.76%
```bash
#原始记录
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.001",} 97577.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.002",} 293417.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.003",} 297628.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.004",} 298078.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.005",} 298257.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.01",} 298578.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.03",} 299277.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.05",} 299708.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.07",} 299874.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.1",} 299937.0

#分组统计
<= 0.003s: 97.8057%
<= 0.005s: 99.3593%
<= 0.010s: 99.4190%
<= 0.030s: 99.5260%
<= 0.050s: 99.7590%
<= 0.100s: 99.9580%
<= 1.00s: 100.0000%

50ms内完成reso: 99.7590%

```


  * 5000 QPS 5min,50ms内完成的保证率99.76%
```bash
#原始数据
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.001",} 507148.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.002",} 1452890.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.003",} 1469155.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.004",} 1473658.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.005",} 1475649.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.01",} 1479310.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.03",} 1486384.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.05",} 1492447.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.07",} 1497304.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.1",} 1499464.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.2",} 1499868.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.3",} 1500000.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.5",} 1500000.0

<= 0.003s: 96.8593%
<= 0.005s: 98.2439%
<= 0.010s: 98.3766%
<= 0.030s: 98.6207%
<= 0.050s: 99.0923%
<= 0.100s: 99.8203%
<= 0.300s: 99.9912%
<= 1.00s: 100.0000%

50ms内获得99.09%
```

  * 10000QPS 压测5min,50ms内完成的保证率97.0307%

```shell script
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.001",} 634865.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.002",} 2409485.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.003",} 2645172.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.004",} 2682787.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.005",} 2702355.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.01",} 2756471.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.03",} 2910921.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.05",} 2964501.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.07",} 2984026.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.1",} 2992563.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.2",} 2996912.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.3",} 2998432.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="0.5",} 3000000.0
micro_service_latency_seconds_bucket{api="get_doc_id",le="1.0",} 3000000.0

<= 0.003s: 80.3162%
<= 0.005s: 89.4262%
<= 0.010s: 90.0785%
<= 0.030s: 91.8824%
<= 0.050s: 97.0307%
<= 0.100s: 99.4675%
<= 0.300s: 99.8971%
<= 1.00s: 100.0000%
```


#### 使用docId,routing,filed查询不同QPS下的测试结果汇总
![image](http://oybm9jmsf.bkt.clouddn.com/image/jpg/markdown-src/test-res.png)


### 通用的基准测试


### 根据业务定义的基准测试
1.根据业务设定索引
[ES 基准测试](https://www.microsofttranslator.com/bv.aspx?from=en&to=zh-CHS&a=https%3A%2F%2Fpeople.apache.org%2F~mikemccand%2Flucenebench%2F)
