---
title: influx_read
date: 2017-10-19 15:51:15
tags:
---
## 非聚和
1. 先根据measurement创建AuxIterator。最小粒度单个timeseries，逐级merge而成。
interator的嵌入结构如下:
{% asset_img "mip-2017101919283058.png" "iterator层级" %}
2. 根据fields创建iterator，第1步的AuxIterator有新创建的FieldAuxIterator的引用
3. AuxIterato执行backgroud
4. 返回第2步创建的FieldAuxIterators

## 创建基本类型Iterator
基本类型Iterator包括FloatIterator，IntegerIterator，StringIterator,BooleanIterator
1. 从元数据索引(内存 or TSI)中读取measurements所有seriesKeys
2. 根据查询stmt里的Dimensions，对sereiesKeys进行分组形成tagset，然后组内seriesKeys排序，再对tagset排序。
3. 对tagset里的seriesKeys再进行分组，然后创建goroutine处理每个分组

## 读取的递进过程
两条线并行进行：
1. 返回结果的Iterator
2. 获取底层数据的Iterator。
    两类Cursor，cur (floatCursor)和aux(CursorAt, 对原生cursor的封装)。


## Questions
### A: emitor里有个iterator，是如何同步多个iterator的时间戳的？
Q: 创建buildAuxIterators时，先创建1个AuxIterator。 然后再根据fields创建多个FieldAuxIterator
同时把这些fieldAuxIterator注册到顶级AuxIterator, 最后返回fieldAuxIterator.   
   对于不同shard的interator，如何同步的呢？  
创建了一个SortedMergeIterator，里面有个小根heap。堆里的元素是子Iterator与point的组合结构item。
初始化时，从每个子Iterator读1个point，然后构造heap。读取时，从顶部pop出item，读取里面的point用于返回，
同时再从子Iterator里读一个point，然后再把这个item push到heap里。
