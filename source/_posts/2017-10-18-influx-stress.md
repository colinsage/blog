---
title: influx_stress造数据令人困惑的问题
date: 2017-10-18 20:47:59
tags:
- influxdb
- tsdb
---

## 1.简介
  influx_stress是influxdb社区版里的一个命令。主要功能是可以解析一个脚本文件，然后执行里面的动作，包括造数据，写db，读db。
从而对单机版进行读写压力测试。 目前有两个版本，其中v2性能更好一点。下面主要讨论v2版本存在的问题。

## 2.造数据  
### 2.1 问题    
以下是官方文档的节选

```
You can write points like this:

INSERT mockCpu
cpu,
host=server-[int inc(0) 100],location=[east|sourth|west|north]
value=[float rand(1000) 0]
100000 10s

Explained:

# INSERT keyword kicks off the statement, next to it is the name of the statement for reporting and templated query generation
INSERT mockCpu
# Measurement
cpu,
# Tags - separated by commas. Tag values can be templates, mixed template and fixed values
host=server-[float rand(100) 100],location=[int inc(0) 10],fixed=[fix|fid|dor|pom|another_tag_value]
# Fields - separated by commas either templates, mixed template and fixed values
value=[float inc(0) 0]
# 'Timestamp' - Number of points to insert into this measurement and the amount of time between points
100000 10s
```  
其中tags, fields支持模板，可以按照模板中的表达式生成对应的值。
最后一行的100000， 表示制造的总数据量。
看到文档， 想当然的以为插入的数据有100000行。实际执行select count后发现并不是这样， 上面这个例子最终只有25000行数据可以查询到。
和预期的差太多了， 好奇怪。  

翻了代码发现，在解析的过程中把每个模板表达式解析成1个模板函数。 例子有三个，两个tag和一个field。它是如何使用的呢？
1. 根据tags的配置， 理论上应该有400个timeseries，因此会把100000分成250时间点。 每个时间点的数据时间戳相同， 包含400条数据。
这个还是符合预期的。
2. 生成每个时间点单个timeserie的数据，可以理解成db里的一行数据。依次调用模板函数，每个函数内部有个计数。当超过上限后恢复成1.
例如location=[east|sourth|west|north]，计数器是4，循环依次输出里面字符串。
3. 对于1行数据，调用了2次模板函数，每个的计数器都加1. 前面100行的tag组合都不一样，但是第101行开始host的计数器又恢复到1，location的计数器也恢复到1. 这样按计划产生的400行数据，只有100行的tags组合是不一样的。
4. influxdb查询事会把相同时间戳，tags的数据进行去重。上例中，某个时间戳查询结果只有100行结果。
5. 总共会产生250个时间点。那么100*250=25000就是最后返回的查询结果。

### 2.2 解决
  理解了里面运行机制， 那么可以只用1个tag， 那么生成与查询的结果就符合预期了。但是这样使用场景就太局限了。
因此还是有必要修改一下代码，让生成的数据符合预期。应该是400个timeseries，那就生成400个不同的timeseries。
怎么做呢？
1. 加个计数器，每产生1行数据加1
2. 计数器与series_count取模，得到当前series编号s_n
3. 从左往右，依次根据s_n，(s_n - 1)计算出当前tag模板函数的计数.
4. 如果有变化，则调用模板函数，其他tag使用上次生成的值，退出循环，结束此次模板替换。
5. 如果无变化，继续根据s_n，(s_n - 1)计算右边下一个tag的计数，判断是否有变化。
6. s_n计算当前模板函数计数的方式：  
<img src="http://chart.googleapis.com/chart?cht=tx&chl=c=[s_n mod  (n_{i} * n_{i-1} * \cdots * n_1 )] / (n_{i-1} * n_{i-2} * \cdots * n_1 )" style="border:none;">

改动点：
+ ./stress/v2/statement/insert.go
+ ./influxdb/stress/v2/statement/function.go
