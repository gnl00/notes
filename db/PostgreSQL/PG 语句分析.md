---
description: 记录 PostgreSQL 使用过程中 SQL 语句分析的历程
tag: 
  - PostgreSQL
  - 数据库
---



# PG 语句分析



## 为何不走索引

```postgresql
explain select * from public.truck_rt tk
    join public.truck_gps_rt gps on tk.time = gps.time
         where tk.time >= '2023-10-22' and tk.time < '2023-11-23';
```

返回结果

```shell
Nested Loop  (cost=0.71..1322545.47 rows=1759459022 width=154) # 最外层是一个大循环
# 先查询 truck_gps_rt
	->  Append  (cost=0.29..6061.02 rows=187898 width=46)
	  # 从 _hyper_3_42_chunk 开始
	  # Index Scan 使用索引扫描 truck_gps_rt_time_idx
    ->  Index Scan Backward using _hyper_3_42_chunk_truck_gps_rt_time_idx on _hyper_3_42_chunk gps_1  (cost=0.29..1192.88 rows=33455 width=46)
    	Index Cond: (("time" >= '2023-10-22 00:00:00+00'::timestamp with time zone) AND ("time" < '2023-11-23 00:00:00+00'::timestamp with time zone))
    -> ...
    # 到 _hyper_3_243_chunk 结束
    # Seq Scan 顺序扫描
    ->  Seq Scan on _hyper_3_243_chunk gps_20  (cost=0.00..217.12 rows=8475 width=46)
    	Filter: (("time" >= '2023-10-22 00:00:00+00'::timestamp with time zone) AND ("time" < '2023-11-23 00:00:00+00'::timestamp with time zone))
# 下面开始查询 truck_rt 
	->  Append  (cost=0.42..6.81 rows=20 width=108)
    ->  Index Scan using _hyper_2_49_chunk_truck_rt_time_idx on _hyper_2_49_chunk tk_1  (cost=0.42..0.51 rows=1 width=108)
    -> ...
```

可以看到，最外层是一个大循环，分别扫描语句中涉及到的表。多表查询时，先根据条件分别从各个表中筛选出合适的数据， 最后再将所有表的结果合并。

> 目前使用的是 TimescaleDB 的超表，查询表中的数据时会根据超表的 chunk 来查询。当前的查询条件中 2023-10-22 被压缩在 _hyper_3_42_chunk 这块 chunk 中，所以从它开始，一直到 2023-11-23 的压缩 chunk _hyper_3_243_chunk 结束。

从 explain 信息中可以看到 truck_gps_rt 表的时候没有用到索引，查询一下

```postgresql
my_db=# select * from pg_indexes where tablename='truck_gps_rt';
 schemaname |  tablename   |       indexname       | tablespace |                                      indexdef
------------+--------------+-----------------------+------------+-------------------------------------------------------------------------------------
 public     | truck_gps_rt | truck_gps_rt_time_idx |            | CREATE INDEX truck_gps_rt_time_idx ON public.truck_gps_rt USING btree ("time" DESC)
```

truck_rt 和 truck_gps_rt  都有着单列索引 time。有一个索引，但是没用上。为什么？回头看 SQL 语句

```postgresql
select * from public.truck_rt tk
    join public.truck_gps_rt gps on tk.time = gps.time
         where tk.time >= '2023-10-22' and tk.time < '2023-11-23';
```

可能原因：

1、~~使用了不等号 `>=` 和 `<`~~

2、使用索引扫描的代价比顺序扫描的代价大

先看看第 1 点，为什么同样都有着不等号，如果是因为不等号的存在所以不走索引，那么 truck_rt 应该也不会走索引才对。但是 truck_rt 走索引，这里就矛盾了，排除掉 1。

接下来看第 2 点，因为 PG 没有像 MySQL 一样展示所有的 explain 情况给我们进行对比，第 2 点原因只是**根据经验判断**。

查询一下这两个表设计的数据量

```postgresql
select count(*) from truck_rt where time >= '2023-10-22' and time < '2023-10-23'; -- 84118

select count(*) from truck_gps_rt where time >= '2023-10-22' and time < '2023-10-23'; -- 8413
```

truck_rt 是 truck_gps_rt 的 10 倍。

所以 truck_gps_rt  不走索引其中一个可能的原因是数据量较少，使用全表扫描的代价反而比使用索引的代价低，所以没有走索引。