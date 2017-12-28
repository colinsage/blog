---
title: tsdb_in_my_eye
date: 2017-11-17 10:17:28
tags: tsdb
---

## 功能需求
### 1. 写
1. 支持单条，批量写接口。
2. 支持生命周期，到期自动删除
3. 支持other tsdb的写协议 (可选)
4. 支持主动消费消息队列里的点数据 (可选)

### 2. 读
1. 原始数据。单metric，单时间线/多时间线，原始数据查询。
2. 聚合。基于tag多时间线的聚合查询；基于tag + timeinterval的多时间线聚合查询。
3. 选择。first,last，min, max， sample
4. 转换。累计求和，滑动平均等
5. 预测。预测未来时间的点。
6. join。多个metric的点，join之后产生的新时间序列。
