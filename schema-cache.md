---
title: Schema 缓存
summary: TiDB 对于 schema 信息采用基于 LRU 的缓存机制，在大量数据库和表的场景下能够显著减少 schema 信息的内存占用以及提高性能。
---

# Schema 缓存

在一些多租户的场景下，可能会存在几十万甚至上百万个数据库和表。这些数据库和表的 schema 信息如果全部加载到内存中，一方面会占用大量的内存，另一方面会导致相关的访问性能变差。为了解决这个问题，TiDB 引入了类似于 LRU 的 schema 缓存机制。只将最近用到的数据库和表的 schema 信息缓存到内存中。

## 配置 schema 缓存

可以通过配置系统变量 [`tidb_schema_cache_size`](/system-variables.md#tidb_schema_cache_size-从-v800-版本开始引入) 来打开 schema 缓存特性。

## 最佳实践

在大量数据库和表的场景下（例如 10 万以上的数据库和表数量）或者当数据库和表的数量大到影响系统性能时，建议打开 schema 缓存特性。以下的最佳实践主要针对该场景。在数据库和表数量较少时，可以忽略以下设置。

- 可以通过观测 TiDB 监控中 **Schema load** 下的子面板 **Infoschema v2 Cache Operation** 来查看 schema 缓存的命中率。如果命中率较低（一般低于 70% 时对 QPS 和延迟会有明显的影响），如果业务对 QPS 和延迟比较敏感，建议调大 [`tidb_schema_cache_size`](/system-variables.md#tidb_schema_cache_size-从-v800-版本开始引入)，并保持命中率在 85% 以上。
- 可以通过观测 TiDB 监控中 **Schema load** 下的子面板 **Infoschema v2 Cache Size** 来查看当前使用的 schema 缓存的大小。
- 建议关闭 [`performance.force-init-stats`](/tidb-configuration-file.md#force-init-stats-从-v657-和-v710-版本开始引入) 以减少 TiDB 的启动时间。
- 如果需要创建大量的表（例如 10 万张以上），建议将参数 [`split-table`](/tidb-configuration-file.md#split-table) 设置为 `false` 以减少 Region 数量，从而降低 TiKV 的内存。
- 在使用 stale read 功能时，如果使用的 schema 太老，可能会导致全量加载历史的 schema，对性能有很大影响。可以通过调大 [`tidb_schema_version_cache_limit`](/system-variables.md#tidb_schema_version_cache_limit-从-v740-版本开始引入) 来缓解这一问题。
- 在访问 INFORMATION_SCHEMA 中的系统视图时，如果视图包含库名或表名字段，建议在查询条件中明确指定数据库名和表名，以避免读取过多元数据，进而影响查询性能，甚至导致 TiDB 内存溢出 (OOM) 问题。

## 已知限制

在大量数据库和表的场景下，有以下已知问题：

- 单个集群的表的数量不能超过 300 万张。
- 当单个集群的表的数量超过 30 万张时，请不要将 [`tidb_schema_cache_size`](/system-variables.md#tidb_schema_cache_size-从-v800-版本开始引入) 取值设置为 `0`，否则可能导致 TiDB OOM。
- 当使用外键时，可能会增加集群 DDL 的执行时长。
- 当对表的访问没有规律，如 time1 访问一批表，time2 访问另外一批表，而且设置的 `tidb_schema_cache_size` 较小时，会导致这些 schema 信息被频繁地被逐出，频繁地被缓存，造成性能抖动。该特性比较适合被频繁访问的库和表是相对固定的场景。
- 统计信息不一定能够及时收集。
- 一些元数据信息的访问会变慢。
- 切换 schema 缓存开关需要等待一段时间。
- 全量列举所有元数据信息的相关操作会变慢，如：

    - `SHOW FULL TABLES`
    - `FLASHBACK`
    - `ALTER TABLE ... SET TIFLASH MODE ...`
- 对于设置了 [`AUTO_INCREMENT`](/auto-increment.md) 或 [`AUTO_RANDOM`](/auto-random.md) 属性的表，如果 schema 缓存设置过小，这些表的元信息可能会在内存中被频繁地缓存和淘汰。这种频繁的缓存变动可能导致未使用完的 ID 段失效，从而引发 ID 跳变。在写入量较大的场景下，甚至可能导致 ID 段耗尽。为减少此类问题并提升系统稳定性，建议采取以下措施：

    - 通过监控面板查看 schema 缓存的命中率和大小，以评估缓存设置是否合理。并适当调大 schema 缓存大小，以减少频繁的缓存淘汰。
    - 将 [`AUTO_ID_CACHE`](/auto-increment.md#auto_id_cache) 设置为 `1`，以防止 ID 跳变。
    - 合理设置 `AUTO_RANDOM` 的分片位和保留位，避免可分配 ID 范围过小。
