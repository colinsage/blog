---
title: influx_http_service
date: 2017-12-04 11:03:14
tags:
---
  支持的不同handler
## serveOptions
直接返回204状态码

## servePing
先记录ping的次数，再返回204状态码

## serveStatus
已过期，用servePing代替

## serveWrite
+ 先解析请求，构造points对象
+ 调用PointsWriter的WritePoints
+ PointsWriter是个接口，默认赋值的是coordinator.PointsWriter
+ 0.11. 先创建shardgroup，接着mapShards得到ShardMapping ，
然后再每个shardinfo的owner创建1个goroutine调用rpc执行写操作。
+ 0.11. 目的节点的cluster模块监听连接，接收请求，并调用本地的TSDBStore写本地的tsdb。
+ 1.3.  也是先mapShards， 再writeToShard。少了多data_node的rpc
