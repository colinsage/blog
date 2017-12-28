---
title: influx_write
date: 2017-10-19 14:02:11
tags:
---

## 1. 架构
{% asset_img "mip-20171019150351540.png" "架构" %}

* service是对外暴露各种读写数据的服务.
* coordinator&influxql 读写数据的抽象接口
* store是内部具体的存储服务执行者.

## 2. 存储格式
tsm文件包含4个部分: header, blocks, index和footer.

### 2.1 header
```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```
 头部前4个字节是0x16D116D1标识是tsm文件, 后1个字节是版本号.

```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```
### 2.2 blocks
#### 2.2.1 block结构  
时序数据的基本单位是block，一个block是一个timeseries在一段时间内的数据集合。   
```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Block是读取的最小单元.每个block包括crc和data两部分.
+ crc用来校验block的数据完整性
+ data是压缩后的时间戳和value. block格式有多种, 由类型和数据分布决定.


block格式:
```
 {len timestamp bytes}, {ts bytes}, {value bytes}
```

#### 2.2.2 压缩
influxdb其中一个最大的卖点是文件压缩率高。一方面可以节省成本，另一方面提升单机的存储能力。
其中tsm数据文件是主要的数据存储地方。这个文件主要存储了时序数据及在当前文件中快速查找的索引。
时序数据是最主要的数据，对其压缩方式的选择直接决定了最终数据文件的大小。

##### timestamp:
* delta相同: {type}, {first}, {delta}, {n} || RLE
* delta不同且max_delta < (2**60-1): {type}, {first}, {packs}  || simple8
* delta不同且delta>=(2**60-1) : {type}, {first}, {second} ... | raw
> 选择压缩方式有点小问题, 应该用max_delta/div的值与2**60-1比较

##### float:
* 第一个不压缩
* 与前个值XOR, head tail的0个数, 比之前大: {10},{有效位}
与前个值XOR, head tail的0个数, 比之前小: {11},{head_5},{len_有效位_6},{有效位}
* 如果第2个值的head,tail都比较小且后续都比第2个大, 那么会浪费不少空间


##### integer
*  先计算delta, 再用zigzag编码. 把负数转成包含有效1bit尽量少.
*  delta相同: {type},{first}, {delta}, {n} || RLE
*  simple8 or raw
> zigzag: uint64(uint64(x<<1)^uint64((int64(x) >> 63)))

##### boolean
* 每个值占用1bit  
* {type}, {len}, {value_bytes}

##### string  
* {type},{snappy_string}   


### 2.3 index
Index 存放的是前面 Blocks 里内容的索引。索引条目的顺序是先按照 key 的字典序排序，再按照 time 排序。InfluxDB 在做查询操作时，可以根据 Index 的信息快速定位到 tsm file 中要查询的 block 的位置。
参考这两个结构体,更容易理解
```
type KeyIndex struct {
    KeyLen      uint16  //下面一个字段 key 的长度。
    Key         string  //seriesKey + 分隔符 + fieldName
    Type        byte  // Block 中 Data 内的数据的类型
    Count       uint32 //后面紧跟着的 Blocks 索引的个数
    Blocks      []*BlockIndex
}

type BlockIndex struct {
    MinTime     int64  //block 中 value 的最小时间戳
    MaxTime     int64  //block 中 value 的最大时间戳
    Offset      int64  //block 在整个 tsm file 中的偏移量
    Size        uint32  //block 的大小。根据 Offset + Size 字段就可以快速读取出一个 block 中的内容
}

┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```
## 2.4 footer

footer有8个字节, 存储index部分在当前tsm文件里偏移值.
```
┌─────────┐
│ Footer  │
├─────────┤
│Index Ofs│
│ 8 bytes │
└─────────┘
```

## 3. 请求处理
1. 写请求处理链路
   + http服务接收请求再转给coordinator
   + coordinator的pointwriter包含store的引用
   + 根据写请求参数,选择合适的shard. 如没有则创建
   + 先写入cache,再写wal文件.
   + 同步的写请求处理完成
```
 http.serveWrite -> co.pointwriter -> shard -> engine -> cache -> wal
```
2. 压缩数据到tsm
  + 每秒检查是否需要执行compact. 超过cache_max_size和duration_max, 执行compact
  + 创建cache的snapshot. cache这里用了一个交换机制, 交换snapshot的store和正在提供服务的cache的store.
  + 创建cacheKeyIteator. 并发执行数据的encode
  + 遍历iteator,写入到tsm文件.
