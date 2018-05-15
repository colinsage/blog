---
title: influxdb-cluster
date: 2017-11-01 12:31:38
tags:
---
### 结构
1. meta node：元数据的写，读。
2. data node：数据的写，读
```
┌───────┐     ┌───────┐      
│       │     │       │      
│ Meta1 │◀───▶│ Meta2 │      
│       │     │       │      
└───────┘     └───────┘      
    ▲             ▲          
    │             │          
    │  ┌───────┐  │          
    │  │       │  │          
    └─▶│ Meta3 │◀─┘          
       │       │             
       └───────┘             

─────────────────────────────────
      ╲│╱    ╲│╱             
  ┌────┘      └──────┐       
  │                  │       
┌───────┐          ┌───────┐   
│       │          │       │   
│ Data1 │◀────────▶│ Data2 │   
│       │          │       │   
└───────┘          └───────┘   
```

### 写过程
数据的分布。 有两级划分，先时间段划分成shardgroup，每个里面包含一段时间内的数据。
时间段长度有创建的rp决定。再按照数据的measurement+tags进行hash后划分，分配到每个shard。
每个shard落在具体的datanode机器上。

shardgroup里shard个数由总机器数/复制因子算出来，取整数。每个shard根据配置的复制因子再确定出所属的
datanode机器。  

因此写入数据时，由datanode接收写请求，根据写请求参数查询meta信息（本地缓存或者meta node）。
结合meta信息，对于每个点，先根据timestamp路由到对应的shardgroup，再根据measurement+tags
路由到shard，最后在把数据分别发送给对应的datanode，剩下的就是单机的写入流程。

前述这些shardgorup，shard在创建shardgroup时创建好的。上述关联信息都保存在metanode中。

### 读过程
   读数据也是由datanode处理的， 请求可以落到任意一台datanode上，和Cassandra很类似。
根据读请求查询meta信息（本地缓存或者meta node），根据时间的start，end确定shardgroup，
然后再找到shardgroup下所有的shard，再从shard关联的datanode中选择其中一个放入请求列表，
然后并行把请求发送给实际存储数据的datanode，最后合并分布式查询结果，并返回给客户端。

## 如何提高查询能力
按照场景进行划分
+ 数据分析
验证想法，频率低。可以支持功能，但是限制查询频率
+ 数据应用
查询参数基本一样，仅仅时间range不一样。预聚合进入tsdb。
查询语句转换成预聚合查询结果的查询

因此可以抽象出三个组件。
+ store
kv的查询模式。

+ stream
聚合流式计算

+ proxy
查询入口，采集分析查询语句的频率，生成预聚合规则。


### 参考资料
1. [first:google gorup](https://groups.google.com/forum/#!msg/influxdb/3jQQMXmXd6Q/cGcmFjM-f8YJ)
2. [0.9.0]( https://www.influxdata.com/blog/clustering-tags-and-enhancements-to-come-in-0-9-0/#signup)
