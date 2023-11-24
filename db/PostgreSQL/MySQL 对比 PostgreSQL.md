# MySQL VS PostgreSQL

## 架构层面

…

---

## 索引

### 索引类型

* MySQL：B-tree，B+tree，Hash。
* PostgreSQL：B-tree，Hash，GIN 倒排索引，BRIN 块范围索引，GiST/SP-GiST 索引。

…

### 相同点

* 索引均遵循最左匹配原则
* …

…

### 不生效场景

在 MySQL 中如果查询语句包含不等号，则不会使用索引；**PostgreSQL 不受不等号的影响**，在有不等号的情况下，PG 会考虑使用 B-Tree 索引。

…