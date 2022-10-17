# 缓存表 Feature 增强

## 团队介绍
- Team: The Powerful Elephants
- [dbsid](https://github.com/dbsid): 测试工程师
- [zyguan](https://github.com/zyguan): 开发工程师
- [Lobshunter](https://github.com/Lobshunter): 开发工程师

## 项目介绍
提升缓存表在易用性、性能方面的用户体验

## 缓存表介绍
> TiDB 在 v6.0.0 版本中引入了缓存表功能。该功能适用于频繁被访问且很少被修改的热点小表，即把整张表的数据加载到 TiDB 服务器的内存中，直接从内存中获取表数据，避免从 TiKV 获取表数据，从而提升读性能。

[官网介绍](https://docs.pingcap.com/zh/tidb/dev/cached-tables)

## Table Cache 在实际应用遇到的痛点

### 缓存表存在 64MB 的限制
64MB 上限过于保守，限制了缓存表的使用场景

### 缓存表写延迟过高
默认 lease 机制导致导致写延迟默认最长 3 秒

ossinsight 上线过缓存表，用以缓存 AP 查询的结果集，在 ossinsight 网页查询时，会更新 `cached_table_cache` 表中的缓存内容。该写操作最长需要等待三秒，页面卡顿，导致缓存表在 ossinsight 下线。

![ossinsigt 应用例子](/images/write_latency.png)

![ossinsigt 慢 SQL 例子](/images/write_latency_slow_query.png)

### sysbench oltp_point_select 性能回退
sysbench `oltp_point_select` 场景，用上缓存表之后性能回退，因为 auto commit 下模式，缓存表没有 fastpath 优化，还需要取 tso。非缓存表模式，存在 fast path 优化，不需要获取 tso.

![](/images/point-get-fast-path.png)


## 本次 Hathathon 的目标

### 解除缓存表存在 64MB 的限制
增加一个系统变量控制缓存表上限

### 降低缓存表写操作的延迟
增加 TiDB 之间直接通信机制，参考 ddl notification 机制 https://github.com/pingcap/tidb/issues/32485

### 「挑战型」缓存表 point get 路径增加 fastpath 优化
参考非 cache 表，auto-commit 打开的情况下， Point get 路径特殊优化。探索缓存有效的情况下，不拿 tso 直接读缓存是否存在正确性问题已经性能提升幅度。

### 「挑战型」数据变更时实现 delta 更新，不需要全量更新
缓存表数据变更时，探索新的数据广播和通信机制，缓存表数据更新时，实现 tidb peer 节点缓存表数据 delta 更新

### 「挑战型」自动缓存功能

根据数据库负载，挑选合适的缓存对象，自动识别并缓存友好的表、region、行, 自动加载缓存，参考传统 buffer cache LRU 的思路

# TO-DO

## 降低缓存表写操作的延迟详细设计

### 保留方案

当前 table cache 采用 lease 方式实现，在 read lock 的 lease 内不允许写操作，因而写延迟通常较高。为了降低写延迟，我们需要能在确保正确性的前提下提前写入。为此我们在现有实现基础上引入如下策略：
1. 每次构建缓存数据时需要注册当前缓存数据元信息，并在内存中维护该 cache 最大的 read ts 。
2. 发起写入时，按原逻辑先写入 intend lock ，然后向各节点发起 invalidate cache 请求。
3. 各节点收到 invalidate 请求后，删除当前缓存数据，并将最终的 max read ts 更新到元信息。
4. 写入节点 watch 缓存数据元信息变更，当所有 cache 都更新了 max read ts 后，取得最大 max read ts 。
5. 写操作可以在最大 max read ts 之后完成提交，并清除所有缓存元数据。

该策略只是对原有实现的补充，如果我们未能在 lease 之前得到最大 max read ts ，将会回退到原来的逻辑，即 lease 过期时也可以写入。

## 激进方案

当前 table cache 采用 lease 方式实现，在 read lock 的 lease 内不允许写操作，因而写延迟通常较高。为了降低写延迟，我们需要能在确保正确性的前提下提前写入。为此我们在现有实现基础上引入如下策略：
1. 每次构建缓存数据时需要注册当前缓存数据元信息，并在内存中维护该 cache 最大的 read ts 。
2. 发起写入时，按原逻辑先写入 intend lock ，然后向各节点发起 invalidate cache 请求。
3. 各节点收到 invalidate 请求后，删除当前缓存数据，返回 response OK。
4. 写操作完成提交，并清除所有缓存元数据。

该策略只是对原有实现的补充，如果我们未能在 lease 之前得到收到所有 TiDB 节点的 response OK，将会回退到原来的逻辑，即 lease 过期时也可以写入。

## Benchmark
## FAQ
