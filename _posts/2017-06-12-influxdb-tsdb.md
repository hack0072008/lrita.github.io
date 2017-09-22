---
layout: post
title: influxdb 源码解析-tsdb
categories: [database, influxdb]
description: influxdb tsdb
keywords: influxdb tsdb
---

`tsdb`是`influxdb`的存储引擎，主要用于持久化时序数据。在分析`tsdb`之前，我们先要了解`influxdb`在使用上关于
存储的一些概念。

## concept
对于`influxdb`中涉及到的各种概念，官方已经提供了一个[词汇表](https://docs.influxdata.com/influxdb/v1.2/concepts/glossary)，
以供用户查阅。

`influxdb`将数据按不同的`database`和`measurements`来存储。`database`可以类比为`mysql`中的`database`概念。
`measurements`可以类比为`mysql`中`table`的概念。写入一条数据时，需要制定其`database`和`measurements`。`influxdb`
提供一套`influxql`的类似`sql`的语法来使用，具体的操作可以参考[官方文档](https://docs.influxdata.com/influxdb/v1.2/query_language/schema_exploration/)

然后`influxdb`存储的数据是schemaless的，可以包含任意维度(列)，每个维度为一个K-V结构，其中维度划分为`tag`和
`field`2种。

#### tag
`tag`是可以建立索引的，在查询时有更好的性能，是可选的字段。通常用来存储一些meta字段。`field`不能建立索引，在查
询某个`field`的值时，需要遍历该时间区间内的全部数据。`tag`跟`field`的区别主要在是否建立索引上，因此写入数据时，
选取`tag`还是`field`主要依据查询语句而定，官方有一个[指导文档](https://docs.influxdata.com/influxdb/v1.2/concepts/schema_and_data_layout/)
可以进行参考。

#### shard
`shard`是一个存储单元，用于存储一段时间范围内的数据，同时`shard`与`retention policy`直接相关，不同的过期策略
下不同的`shard`，每个`shard`底层对应一个`TSM`存储引擎。当过期策略检查到某一个`shard`过期后，会释放其对应的资源。

#### point
每一次写入的数据称之为`point`，其包含`timestamp`、`measurements`、`retention policy`、`tag`和`field`等信息，其中`point`的`key`[由
`measurement`和`tag`hash后构成](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/models/points.go#L1481-L1485)。
然后根据`point.HashID`的值和`retention policy`选取合适的`shard`，将该`point`写入该`shard`。

`tsdb`对外提供的2个最重要的接口是：
* `CreateShard(database, retentionPolicy string, shardID uint64, enabled bool) error` 创建`database`对应的的`shard`
* `WriteToShard(shardID uint64, points []models.Point) error` 向指定`shard`写入数据

#### store
`store`是`influxdb`的存储模块，全局只有一个该实例。主要负责将数据按一定格式写入磁盘，并且维护`influxdb`相关的
存储概念。例如：创建/删除`Shard`、创建/删除`retention policy`、调度`shard`的compaction、以及最重要的`WriteToShard`
等等。在`store`内部又包含`index`和`engine`2个抽象概念，`index`是对应`shard`的索引，`engine`是对应`shard`的存储实现，
不同的`engine`采用不同的存储格式和策略。后面要讲的`tsdb`其实就是一个`engine`的实现。在`influxdb`启动时，会创建
一个`store`实例，然后`Open`它，初始化时，它会加载已经存在的[`Shard`](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/store.go#L154-L276)
，并启动一个`Shard`[监控任务](https://github.com/influxdata/influxdb/blob/4957b3d8be5fff66b1150f3fe894da09092e923a/tsdb/store.go#L1047-L1116)，
监控任务负责调度每个`Shard`的`Compaction`和对使用`inmem`索引的`Shard`计算每种`Tag`拥有的数值的基数(与配置中`max-values-per-tag`有关)。

## 文件结构
在`influxdb`指定的存储目录下的文件结构为:
```
.
├── data  # 配置中[data]下dir配置的路径
│   ├── _internal
│   │   └── monitor
│   │       ├── 67
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 69
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 70
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 71
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 72
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 73
│   │       │   └── 000000003-000000002.tsm
│   │       ├── 74
│   │       │   └── 000000003-000000002.tsm
│   │       └── 75
│   │           └── 000000001-000000001.tsm
│   └── testing
│       └── autogen
│           └── 2
│               └── 000000002-000000002.tsm
├── meta
│   └── meta.db
└── wal	# 配置文件[data]下wal-dir配置
    ├── _internal
    │   └── monitor
    │       ├── 67
    │       │   └── _00012.wal
    │       ├── 69
    │       │   └── _00012.wal
    │       ├── 70
    │       │   └── _00012.wal
    │       ├── 71
    │       │   └── _00012.wal
    │       ├── 72
    │       │   └── _00012.wal
    │       ├── 73
    │       │   └── _00012.wal
    │       ├── 74
    │       │   └── _00012.wal
    │       └── 75
    │           ├── _00005.wal
    │           ├── _00006.wal
    │           └── _00007.wal
    └── testing
        └── autogen
            └── 2
                └── _00003.wal
```

# tsdb
我们可以先阅读以下对于`tsdb`的[官方文档](https://docs.influxdata.com/influxdb/v1.2/concepts/storage_engine/)。
其采用的存储模型是`LSM-Tree`模型，对其进行了一定的改造。将其称之为`Time-Structured Merge Tree (TSM)`

当一个`point`写入时，`influxdb`根据其所属的`database`、`measurements`和`timestamp`选取一个对应的`shard`。每个
`shard`对应一个`TSM`存储引擎。每个`shard`对应一段时间范围的存储。

一个`TSM`存储引擎包含：
* `In-Memory Index` 在`shard`之间共享，提供`measurements`，`tags`，和`series`的索引 
* `WAL` 同其他database的binlog一样，当`WAL`的大小达到一定大小后，会重启开启一个`WAL`文件。
* `Cache` 内存中缓存的`WAL`，加速查找
* `TSM Files` 压缩后的`series`数据
* `FileStore` `TSM Files`的封装
* `Compactor` 存储数据的比较器
* `Compaction Planner` 用来确定哪些`TSM`文件需要`compaction`，同时避免并发`compaction`之间的相互干扰
* `Compression` 用于压缩持久化文件
* `Writers/Readers` 用于访问文件
