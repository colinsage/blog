---
title: influx_read
date: 2017-10-19 15:51:15
tags:
---
## 入口
当使用http协议时，在handler的serveQuery处理方法。
使用命令行时，insert语句也是转换成对应的http URL，Path是 "/query"

## 类型
+ 元数据修改
+ 元数据查询
+ 数据查询

## 数据查询
+ influxsql解析成statement
+ statement创建iterator
+ 聚合计算

### 非聚和
1. 先根据measurement创建AuxIterator。最小粒度单个timeseries，逐级merge而成。
interator的嵌入结构如下:
{% asset_img "mip-2017101919283058.png" "iterator层级" %}
2. 根据fields创建iterator，第1步的AuxIterator有新创建的FieldAuxIterator的引用
3. AuxIterato执行backgroud
4. 返回第2步创建的FieldAuxIterators

### 创建基本类型Iterator
基本类型Iterator包括FloatIterator，IntegerIterator，StringIterator,BooleanIterator
1. 从元数据索引(内存 or TSI)中读取measurements所有seriesKeys
2. 根据查询stmt里的Dimensions，对sereiesKeys进行分组形成tagset，然后组内seriesKeys排序，再对tagset排序。
3. 对tagset里的seriesKeys再进行分组，然后创建goroutine处理每个分组

### 读取的递进过程
两条线并行进行：
1. 返回结果的Iterator
2. 获取底层数据的Iterator。
    两类Cursor，cur (floatCursor)和aux(CursorAt, 对原生cursor的封装)。

### 创建iterator的层级
node_s -> source_s > shard_s -> tagset_s -> serieskey_s
> 1. 远程的iterator到node级别
> 2. 涉及merger的有: node, source, shard, tagset. 基本上每级都会进行合并
> 3. source 是对多个shard进行source划分, 分别创建再merge合并

## Questions
### Q1: emitor里有个iterator，是如何同步多个iterator的时间戳的？
A: 创建buildAuxIterators时，先创建1个AuxIterator。 然后再根据fields创建多个FieldAuxIterator
同时把这些fieldAuxIterator注册到顶级AuxIterator, 最后返回fieldAuxIterator.   
   对于不同shard的interator，如何同步的呢？  
创建了一个SortedMergeIterator，里面有个小根heap。堆里的元素是子Iterator与point的组合结构item。
初始化时，从每个子Iterator读1个point，然后构造heap。读取时，从顶部pop出item，读取里面的point用于返回，
同时再从子Iterator里读一个point，然后再把这个item push到heap里。

### Q2: 为何limit会比较慢
A: 由于iterator的装饰特性, 先执行内部的mergerInterator, 再执行limit.  mergeInterator会根据所有的时间线创建1个堆,
以便于排序. 如果维度很高,创建这个堆的过程也很耗时. 即使查询条件limit=1, 也需要一段时间来执行mergeInterator.
