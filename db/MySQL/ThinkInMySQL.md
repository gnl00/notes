---
description: 在使用 MySQL 的过程中的一些思考
sticky: true
tag: 
  - MySQL
  - 数据库
---

# Think In MySQL

## SQL

### SQL 优化思考

首先导入[测试数据](https://github.com/datacharmer/test_db)。思考一下，开发中最常见的 SQL 优化场景基本上都是和 select 有关的。思考一下，**一个慢 select 应该如何优化？**

拿测试数据来说，比如我们有这么个 SQL，查询所有员工的 emp_no、first_name、last_name、gender、hire_date、dept_name、title、salary。

```sql
select emp_dept.emp_no emp_no, first_name, last_name, dept_name, gender, hire_date, title, salary from (
select emp.emp_no emp_no, first_name, last_name , hire_date, dept_emp.dept_no,  gender from employees as emp
join dept_emp on dept_emp.emp_no = emp.emp_no 
) as emp_dept
join departments on emp_dept.dept_no = departments.dept_no
join titles on emp_dept.emp_no = titles.emp_no
join salaries on emp_dept.emp_no = salaries.emp_no;
```

跨 employees、dept_emp、departments、titles、salaries 这 5 个表，查询员工的完整信息。在终端中执行以下看看：

```shell
5124191 rows in set (9.28 sec)
```

查出来的结果有五百万行，在我的机子上执行这条 SQL 花费时间 9.28 秒。显然，对于这个结果我们是不能接受的。**如何优化呢？**

<br>

这时最常见的做法是**分页**，

```
select <column_you_need> from employees limit (pageSize * pageNum) - 1, pageSize;
```

让我们来试试，分成 100 条数据一页，我们请求第 1 页：

```shell
100 rows in set (0.00 sec)
```

分成 200 条数据一页，我们请求第 500 页：

```shell
200 rows in set (0.11 sec)
```

分成 200 条数据一页，我们请求第 5000 页：

```shell
200 rows in set (0.87 sec)
```

分成 200 条数据一页，我们请求第 10000 页：

```sql
200 rows in set (1.77 sec)
```

跳过 25000 页的时候：

```shell
200 rows in set (4.59 sec)
```

耗费 4.59 秒，随着跳过的页数增加（或者分页大小的增加），耗费的时间也会越来越多。但是相比于一开始的 9 秒多，分页之后花费的时间已经少了一半。

如果对这个优化结果不满意，**接下来从哪里入手呢？**

<br>

**建立索引**，索引本质上是一种数据结构，能加快数据的查询速度。需要注意：**需要对哪个列创建索引？是否应该创建联合索引？**

答案是对不含有大量重复元素，且经常用作查询的列建立索引。是否创建联合索引需要根据实际的查询场景来定，如果某几个列经常被放在一起用作查询，可以在这几个列上建立联合索引，但是使用联合索引的时候需要注意遵循最左匹配原则。

还有一个需要重视的问题是：**索引是否一定生效？SQL 的执行是否会选择刚创建好的索引？**

索引确实能帮助我们加快查询速度，但不是所有的索引都可以，只有**花费代价最小的索引**才会被选用来进行查询操作。在 MySQL 中，一条 SQL 执行的以后会经过 SQL 解析和优化器，优化器会帮助我们选择正确的索引。

<br>

让我们来感受一下，首先打开优化器追踪，后续分析的时候会用到：

```sql
set optimizer_trace="enabled=on";
```

为 employees 表建立一个索引（类似 gender 这种大量重复数据的列就不需要建立索引了）：

```sql
create index <index_name> on <table_name> (column1, column2, ...);
```

```sql
create index idx_name_date on employees (first_name, last_name, hire_date);
```

执行查询操作：

```sql
select first_name, last_name, hire_date from employees where emp_no > 100000 and emp_no < 250000;
```

接着，explain 分析以下该 SQL 的执行情况：

```shell
mysql> explain select first_name, last_name, hire_date from employees where emp_no > 100000 and emp_no < 250000 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: employees
   partitions: NULL
         type: range
possible_keys: PRIMARY,idx_name_date
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 120736
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

可以发现，存在 2 个可能被选用的索引，PRIMARY 和 idx_name_date，但最终选择的是 PRIMARY，而不是 idx_name_date。**怎么回事呢？**

创建的联合索引 idx_name_date 看起来并没有问题，而且查询的时候也遵循了最左匹配原则，但 MySQL  最终执行的时候还是没有选用它。这就要说到 **SQL 优化器**了。还记得刚才开启的 optimizer_trace 吗？现在排上用场了。

<br>

让我们来看一下刚才那一条 SQL 的追踪结果：

```sql
select * from INFORMATION_SCHEMA.OPTIMIZER_TRACE\G;
```

在 TRACE 中出现的字段表示查询执行过程中的一个步骤或操作：

1. join_preparation：连接准备阶段，包括连接类型、连接条件等。
2. join_optimization：连接优化阶段，包括连接顺序、连接算法等。
3. join_execution：连接执行阶段，包括连接操作的具体执行。

我们主要关注 join_optimization 阶段，显示结果如下：

```json
TRACE: {    
  "steps": [
    {
      "join_preparation": {}
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "table_dependencies": [ // 查询所依赖的表
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "rows_estimation": [ // 估算此次查询涉及到的行数
              {
                "table": "`employees`",
                "range_analysis": {
                  "table_scan": { // 全表扫描
                    "rows": 299423,
                    "cost": 30176.7 // 全表扫描花费的代价
                  },
                  "potential_range_indexes": [ // 可能使用的索引
                    {
                      "index": "PRIMARY",
                      "usable": true,
                      "key_parts": [
                        "emp_no"
                      ]
                    },
                    {
                      "index": "idx_name_date",
                      "usable": true,
                      "key_parts": [
                        "first_name",
                        "last_name",
                        "hire_date",
                        "emp_no"
                      ]
                    }
                  ],
                  "best_covering_index_scan": { // 覆盖索引扫描 == 使用索引扫描
                    "index": "idx_name_date",
                    "cost": 32453, // 覆盖索引扫描花费的代价
                    "chosen": false, // 不使用覆盖索引扫描
                    "cause": "cost" // 因为花费的代价 32453 大于全表扫描 30176.7
                  },
                  "skip_scan_range": { // 跳过扫描的范围
                    "potential_skip_scan_indexes": [ // 跳过扫描可能会使用的索引
                      {
                        "index": "PRIMARY",
                        "usable": false, // 不使用 PRIMARY 索引来协助跳过扫描
                        "cause": "query_references_nonkey_column" // 因为 PRIMARY 索引不包含查询中的列（first_name、last_name、hire_date）
                      },
                      {
                        "index": "idx_name_date",
                        "tree_travel_cost": 0.95,
                        "num_groups": 294014,
                        "rows": 299423,
                        "cost": 606051 // 使用 idx_name_date 索引帮助跳过扫描花费的代价
                      }
                    ]
                  },
                  "best_skip_scan_summary": { // 最终使用的跳过扫描的总结
                    "type": "skip_scan",
                    "index": "idx_name_date",
                    "key_parts_used_for_access": [
                      "first_name",
                      "last_name",
                      "hire_date",
                      "emp_no"
                    ],
                    "range": [
                      "100000 < emp_no < 250000"
                    ],
                    "chosen": false, // 不使用 idx_name_date 索引
                    "cause": "cost" // 因为花费的代价太大
                  },
                  "analyzing_range_alternatives": { // 分析范围扫描的选择，即条件中的 emp_no>100000 and emp_no<250000
                    "range_scan_alternatives": [
                      {
                        "index": "PRIMARY", 
                        "ranges": [
                          "100000 < emp_no < 250000"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": false,
                        "index_only": false,
                        "in_memory": 1,
                        "rows": 120736, // 涉及到的行数 120736
                        "cost": 12103.8, // 花费的代价为 12103
                        "chosen": true // 使用 PRIMARY 索引 
                      },
                      {
                        "index": "idx_name_date",
                        "chosen": false, // 不使用 idx_name_date 索引，因为 idx_name_date 索引中没有包含 emp_no 字段
                        "cause": "no_valid_range_for_this_index"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    }
                  },
                  "chosen_range_access_summary": { // 最终选用的范围扫描
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "PRIMARY",
                      "rows": 120736,
                      "ranges": [
                        "100000 < emp_no < 250000"
                      ]
                    },
                    "rows_for_plan": 120736,
                    "cost_for_plan": 12103.8,
                    "chosen": true // 使用 PRIMARY 索引
                  }
                }
              }
            ]
          },
          {
            "considered_execution_plans": [ // 最终确定的执行计划
              {
                "plan_prefix": [
                ],
                "table": "`employees`", // 最终执行涉及到的表
                "best_access_path": { // 最好的执行路径
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 120736,
                      "access_type": "range",
                      "range_details": {
                        "used_index": "PRIMARY" // 最终确定范围扫描使用的索引是 PRIMARY
                      },
                      "resulting_rows": 120736, // 涉及到的总行数
                      "cost": 24177.4, // 总花费
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 120736,
                "cost_for_plan": 24177.4, // 总花费代价
                "chosen": true
              }
            ]
          }
        ]
      }
    },
    {
      "join_execution": {}
    }
  ]
}
```

可以看到，基于代价模型的优化，MySQL 最终并没有选用 idx_name_date 索引。

<br>

**建立索引之后？**

如果索引建立得很好，SQL 查询次次命中，查询速度嗖嗖快，并且你很满意，那你的优化之旅可以暂时告一段落了。**那如果我对使用了索引之后的结果还是不满意呢？**

再之后，**可以在 SQL 本身上做文章**。如果 SQL 语句中 join 了多个表，能拆分成子查询就尽量拆分，因为子查询的 SQL 可以利用索引。

> ...

**关于分库分表**

此外还有分表，分库等操作。

**分表**操作如下：如果一个表的数据量实在太大，可以将其分为多个表。如果一个表中的列数太多可以分为多个业务表，称为垂直分表；如果一个表中的数据太多，可以按照一定的范围，见其分为多个表，如将一个 2000 万数据的表分为每 500 万数据一个表，称为水平分表。还可以根据日期、哈希、求模等方式进行分表。

**分库**则是将单一的数据库分为多个数据库，分为垂直（业务）和水平分库。垂直（业务）分库和分表类似；水平分库也是可以按照一定范围或者哈希等方式进行。

**哈希、求模在分库分表中如何使用？**

求模：将某个唯一标识（如用户 ID、订单 ID 等）与数据库或表的数量进行求模运算，以确定数据应该存储在哪个数据库或表中；

哈希：将某个唯一标识（如用户 ID、订单 ID 等）通过哈希函数映射到一个固定范围的值，如哈希函数可以将用户 ID 映射为一个整数，该整数即为数据存放的库或者表；再进一步还可以对哈希结果进行求模，再存放数据。

**What's more?**

如果是查询为主的场景，还可以引入 Elasticsearch 搜索引擎，将从 MySQL 获取数据改成从 Elasticsearch 获取数据。



<br>

**总结**

> 总的来说，就是一步一步来。业务逻辑千变万化，我们可以预见其中一部分，但不是种种情况都能覆盖完全。在遇到 SQL 优化之类的问题时，不要求一步到位直接优化为 x ms，只需要分析好慢 SQL 造成的原因，再逐个击破就好。比如说，没分页，就分页；没索引，加索引，具体问题具体分析。
>
> 不要求一步到位的意思是，你此时的进一步优化在真正的业务路基中并不一定受用。比如我创建了表，直接连带着创建索引，咔咔疯狂创建索引。如果你的数据量太大，那么索引的维护对于数据库来说是一部分负担。而且你也不确定 MySQL 在执行 SQL 语句的以后 SQL 优化器就一定会选择你创建了的索引。
>
> 实事求是，具体问题具体分析。





<br>

## 参考

https://tech.meituan.com/2022/04/21/slow-query-optimized-advice-driven-by-cost-model.html

https://ost.51cto.com/posts/11797

https://developer.aliyun.com/article/51062