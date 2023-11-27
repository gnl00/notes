# MySQL VS PostgreSQL

## 架构层面

从架构层面来说，PostgreSQL 和 MySQL 都是 C/S 架构。

…

---

## 索引

### 索引结构

* MySQL：B-tree，B+tree，Hash。
* PostgreSQL：B-tree，Hash，GIN 倒排索引，BRIN 块范围索引，GiST/SP-GiST 索引。

…

### 最左匹配原则

PG 和 MySQL 均遵循最左匹配原则。

…

### 不生效场景

在 MySQL 中如果查询语句包含不等号，则不会使用索引；**PostgreSQL 不受不等号的影响**，在有不等号的情况下，PG 会考虑使用 B-Tree 索引。

…

### 部分/表达式索引

PG 支持**部分索引**和**表达式索引**，用来应对复杂的查询需求。

…

### 为什么 PG 使用 B-Tree

* B-tree 更加擅长范围查询
* B-tree 能更好的处理随机插入/删除的情况
* 不需要维护叶子节点之间的链表结构
* …

…