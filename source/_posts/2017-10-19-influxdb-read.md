---
title: Influxdb之read流程分析
date: 2017-10-19 15:51:15
tags: influxdb sql tsm
---
## 前言
   由于influxdb的更新很快，以下描述对应的是1.6.2版本的代码。
从使用角度看，查询主要涉及两类，元数据和时序数据。例如查询database，retention_policy,
measurement，tag_key, tag_value, field_name 等都属于查询元数据。
时序数据查询过程中也会进行元数据查询以便定位数据。

## 查询入口
   当前influxdb的查询主要入口是httpd服务，未来2.0的查询入口是storage服务，以下主要描述
httpd服务里是怎么处理查询的。
   程序入口是'service/httpd/handler.go'的serveQuery方法。这个方法有三百多行，
主要干三个事情: 解析request生成Query对象，执行query查询，查询结果序列化返回给client。

```
func (h *Handler) serveQuery(w http.ResponseWriter, r *http.Request, user meta.User) {
    ....
    var qr io.Reader
    if qp := strings.TrimSpace(r.FormValue("q")); qp != "" {
    		qr = strings.NewReader(qp)
    }
    ...
    p := influxql.NewParser(qr)  // 构建parser
    ...

    q, err := p.ParseQuery()   // 解析query
    ...

    results := h.QueryExecutor.ExecuteQuery(q, opts, closing)  //执行query

    resp := Response{Results: make([]*query.Result, 0)}
    ...

    if !chunked {
		   n, _ := rw.WriteResponse(resp)  // 序列化并吐出给client
	  }
}
```
   前两项后面详细说明，这里介绍一下结果序列化。

   influxdb支持result结果以多个chunk的形式吐出给client，
需要在请求里添加2个参数，chunked=true和chunk_size=n 这里和[官方文档](http://docs.influxdata.com/influxdb/v1.6/tools/api/#query)的描述不太一样。 文档里指出只需要chunked参数，值可以是true和具体的数值。当为true，取默认值10000作为chunk的点的个数。
对比一下，还是文档里的更方便一些。  

   对于delete之类的操作元数据的查询请求，不需要响应结果。可以在请求里添加async参数，可以快速返回。
服务器会另起一个goroutine来处理result。

    查询结果的格式是根据request里Accept Header来决定的，支持csv, msgpack, json格式。
如果没有设置Accept头部，默认json格式。

当然httpd服务对于查询请求处理还做了一些日志记录，鉴权等工作，但不影响主流程的理解。

## 解析Query  
   请求传递过来的参数q是一个类似sql的string。语法定义参考[doc](http://docs.influxdata.com/influxdb/v1.6/query_language/spec/)
解析的结果是Statement，名字是不是很类似jdbc里的对象，意义也差不多，都是封装了查询参数的对象，用来传递给执行引擎。当前一共有46种类型的Statement。

   解析过程类似以前编译原理课程学习的词法分析和语法分析。 对输入的string, 扫描出token，并根据token的顺序
定位到对应的statement类型。Token可以简单理解成用一个个独立的字符串单元，例如"show", "select"等。
空白字符串也是一种Token。在parser_tree.go的init方法里，Language这个parseTree对象预定义了不同顺序token的解析函数。

例如"create database mydb",  扫描这个字符串，当识别到"create" "database" 这两个token时，
就可以从Language对象里找到解析CreateDatabaseStatement的函数，并执行这个函数创建CreateDatabaseStatement对象，
继续扫描识别到"mydb"这个token， 这个token是IDENT类型，把它赋值给CreateDatabaseStatement的Name字段。
这个解析函数还会继续扫描是否有 DURATION，REPLICATION等token，并进行相应赋值。我们这个例子到mydb就结束了，直接返回。
返回的*Statement对象，就是抽象语法树，传输给执行引擎使用。

解析Query代码主要在influxql这个project里。里面主要有下面几个文件:
  + ast.go 多种类型Statement的定义。需要扩展语法，一定会修改这个文件
  + parse_tree.go 定义了Language的解析树。
  + parse.go 多种statement的具体解析方法实现。
  + scanner.go 多种Token的扫描方法
  + token.go  多种Token的定义

从源代码看，influxdb和现在其它开源项目不太一样，它的查询语言的解析实现完全是手工打造。
其它项目遇到这类涉及语法解析的场景，会选择词法和语法分析的工具，定义好语法规则，然后使用工具根据语法规则文件来生成解析代码。
不过，2.0版本的ifql语法解析已经是使用工具生成parse相关go代码。


## 执行Query
### TaskManager
每个查询都会分配一个查询id，并在taskmanager中注册。运行中的查询任务，可以被查询并中断掉。

### RewriteStatement
对于部分Show*Statement这一类查询元数据的statement，会被转换成SelectStatement，例如ShowFieldKeysStatement。


### NormalizeStatement
紧接上一步，对于statement缺少类似database等字段的，会赋值default值。

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

## 非聚和
1. 先根据measurement创建AuxIterator。最小粒度单个timeseries，逐级merge而成。
interator的嵌入结构如下:
{% asset_img "mip-2017101919283058.png" "iterator层级" %}
2. 根据fields创建iterator，第1步的AuxIterator有新创建的FieldAuxIterator的引用
3. AuxIterato执行backgroud
4. 返回第2步创建的FieldAuxIterators

### 创建基本类型Iterator
基本类型Iterator包括FloatIterator, IntegerIterator, StringIterator, BooleanIterator
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

## 聚合
   聚合类型的查询会创建CallIterator类型的iterator。创建之前会从index查询出符合匹配条件的seriesKey
然后根据groupby的条件，为每个group创建tagset。每个tagset里包含多个属于该group的seriesKey。
   然后再根据tagset创建最底层数据类型的iterator。如果为tagset创建的iterator数目多余cpu的个数，
则会创建并行读取的iterator，这是个支持异步读取的iterator。
### merge涉及的层次
共5层.
+ shard内serie.
+ shard
+ sharggroup
+ shardmapping
+ query


## Questions
### Q1: emitor里有个iterator，是如何同步多个iterator的时间戳？
A: 创建buildAuxIterators时，先创建1个AuxIterator。 然后再根据fields创建多个FieldAuxIterator
同时把这些fieldAuxIterator注册到顶级AuxIterator, 最后返回fieldAuxIterator.   
   对于不同shard的interator，如何同步的呢？  
创建了一个SortedMergeIterator，里面有个小根heap。堆里的元素是子Iterator与point的组合结构item。
初始化时，从每个子Iterator读1个point，然后构造heap。读取时，从顶部pop出item，读取里面的point用于返回，
同时再从子Iterator里读一个point，然后再把这个item push到heap里。

### Q2: 为何limit会比较慢
A: 由于iterator的装饰特性, 先执行内部的mergerInterator, 再执行limit.  mergeInterator会根据所有的时间线创建1个堆,
以便于排序. 如果维度很高,创建这个堆的过程也很耗时. 即使查询条件limit=1, 也需要一段时间来执行mergeInterator.
