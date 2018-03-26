---
title: influxdb_monitor
date: 2018-02-27 19:49:45
tags:
---
## 1. 前言
  无论是开发还是运维，对于influxd(InfluxDb服务端)的运行状态信息的获取都是至关重要的。
在influxd内部把监控信息分成了两大类: Diagnostics 和 Statistics 。 这些信息默认是持久化到
influxdb内的，对应得database是 `_internal` ， 每隔10s写入一次。


## 2. Statistics
  Statistics信息是可以计数的数字，需要定期存储起来以便事后分析。例如写入点的个数，查询请求数等。  

### 2.1 查询statistics
1. 在console执行命令。 SHOW STATS
2. 读http服务。 curl http://localhost:8086/debug/vars

### 2.2 添加statistics
对于新创建的服务，添加Statistics信息。参考以下步骤:
1. 定义Statistics结构，并作为的服务的Field之一。
```
type ShardWriteStatistics struct {
	DirectWriteReq      int64
	FlushWriteReq int64
	FlushCount  int64
}
```
2. 在服务相关的地方执行原子计数增加。  
```
atomic.AddInt64(&w.stats.FlushCount, 1)
```
3. 实现接口 monitor.Reporter。
```
func (w *ShardWriter) Statistics(tags map[string]string) []models.Statistic {
	return []models.Statistic{{
		Name: "shard_write",
		Tags: tags,
		Values: map[string]interface{}{
			directWriteReq:    atomic.LoadInt64(&w.stats.DirectWriteReq),
			flushWriteReq:     atomic.LoadInt64(&w.stats.FlushWriteReq),
			flushCount:        atomic.LoadInt64(&w.stats.FlushCount),
		},
	}}
}
```
4. 透出statistics信息。如果新服务实现了service接口，默认会注册到server的;其他情况可以在Server的Statistics方法中调用，并append到
最终结果中。
```
statistics = append(statistics, s.ShardWriter.Statistics(tags)...)
```

## 3. Diagnostics
  Diagnostics信息可以不是数字，也不需要存储，一般在内存中。例如influxd版本信息，各个service
的配置信息等。

### 3.1 查询diagnostics
1. 在console执行命令。 SHOW DIAGNOSTICS

### 3.2 添加diagnostics
  对于新创建的服务的配置信息或者运行时参数，可以作为diagnostics信息透出。可按下面步骤添加
1. 对新服务的config结构，实现接口 diagnostics.Client
```
func (c Config) Diagnostics() (*diagnostics.Diagnostics, error) {
	return diagnostics.RowFromMap(map[string]interface{}{
		"write-timeout":          c.WriteTimeout,
		"max-concurrent-queries": c.MaxConcurrentQueries,
		"query-timeout":          c.QueryTimeout,
		"log-queries-after":      c.LogQueriesAfter,
		"max-select-point":       c.MaxSelectPointN,
		"max-select-series":      c.MaxSelectSeriesN,
		"max-select-buckets":     c.MaxSelectBucketsN,
	}), nil
}
```
2. 在run包的config.go中的diagnosticsClients添加
```
func (c *Config) diagnosticsClients() map[string]diagnostics.Client {
 	m := map[string]diagnostics.Client{
		"config": c,
		"config-data":        c.Data,

		"config-dataext":        c.DataExt,  // added config

		"config-meta":        c.Meta,
		"config-retention":   c.Retention,
		"config-precreator":  c.Precreator,
		"config-monitor":    c.Monitor,
		"config-httpd":      c.HTTPD,
		}
	return m
}
```


## 4. References
[Server monitoring](https://docs.influxdata.com/influxdb/v1.4/troubleshooting/statistics/)
[System Monitoring](https://github.com/influxdata/influxdb/blob/master/monitor/README.md)
[How to use the SHOW STATS command and the \_internal database to monitor InfluxDB](https://www.influxdata.com/blog/how-to-use-the-show-stats-command-and-the-_internal-database-to-monitor-influxdb/)
