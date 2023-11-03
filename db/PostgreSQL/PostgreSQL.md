# PostgreSQL



## 启动

**使用 Docker**

```bash
docker pull postgres
docker run --name postgres-test -p 5432:5432 -e POSTGRES_PASSWORD=postgres -v /var/lib/postgresql/dump/2023-11-02:/var/lib/postgresql/data -d postgres
# 将 PostgreSQL 内部的数据目录挂载到外部时需要挂载 /var/lib/postgresql/data 这个路径
# 挂载 /var/lib/postgresql 路径的话数据目录是不会被映射出来的。
# ref：https://hub.docker.com/_/postgres # PGDATA 参数部分
```

…

---

<br>

## 数据库系统命令

```bash
psql -U <username> -h <host> -d <dbname>
\? # PostgreSQL 【系统级别】命令帮助文档
\h # PostgreSQL 中 SQL 命令帮助文档

\dg # list roles, role defualt is [postgres]

\l # list databases
\dt # list tables
\dx # list extensions

\db <tablename> # list tablespaces

\c <dbname> # connect databae
\conninfo # display information about current connection

# after connected to a database
SELECT version(); # Show postgreSQL's version
```

…

---

## 数据库操作

```bash
create database <dbname>
drop database <dbname> # can not drop your current db

psql <dbname> # same as \c, connect databaes
```

…

---

## 数据表操作

### 创建/删除

创建/删除表的语句和 MySQL 一致：

* create table
* drop table

从文件装载数据

```bash
# 使用 COPY 从文本文件中装载大量数据
COPY weather FROM '/path/weather.txt';
```

 表查询操作

```bash
\d+ <tablename> # 查看表的详情信息，表结构，字段等
```



**创建测试表**

```sql
CREATE TABLE employees (
  depname VARCHAR(20),
  empno INT,
  salary INT
);
```

```sql
INSERT INTO employees VALUES ('develop', 11, 5200);
INSERT INTO employees VALUES ('develop', 7, 4200); 
INSERT INTO employees VALUES ('develop', 9, 4500);
INSERT INTO employees VALUES ('develop', 8, 6000);
INSERT INTO employees VALUES ('develop', 10, 5200);
INSERT INTO employees VALUES ('personnel', 5, 3500);
INSERT INTO employees VALUES ('personnel', 2, 3900);  
INSERT INTO employees VALUES ('sales', 3, 4800);
INSERT INTO employees VALUES ('sales', 1, 5000);
INSERT INTO employees VALUES ('sales', 4, 4800);
```



### 修改表

- 增加列
- 移除列
- 增加约束
- 移除约束
- 修改默认值
- 修改列数据类型
- 重命名列
- 重命名表

…

---

## 数据类型

### 几何类型

| 类型    | 大小        | 描述                     | 备注                            |
| ------- | ----------- | ------------------------ | ------------------------------- |
| point   | 16 字节     | 平面上的点               | (x,y)                           |
| line    | 32 字节     | 无限长的线               | {A,B,C}                         |
| lseg    | 32 字节     | 有限线段                 | ((x1,y1),(x2,y2))，A 点到 B 点  |
| box     | 32 字节     | 矩形框                   | ((x1,y1),(x2,y2))，斜对角线端点 |
| path    | 16+16n 字节 | 封闭路径（类似于多边形） | ((x1,y1),...)                   |
| path    | 16+16n 字节 | 开放路径                 | [(x1,y1),...]                   |
| polygon | 40+16n 字节 | 多边形（类似于封闭路径） | ((x1,y1),...)                   |
| circle  | 24 字节     | 圆                       | <(x,y),r>（中心点和半径）       |



### 数组

#### 数组操作

```postgresql
CREATE TABLE user_hobbies (
  id serial not null,
  name VARCHAR(50),
  hobbies TEXT[]
);

-- 可以使用 array 关键字来表示数组
INSERT INTO user_hobbies (name, hobbies)
VALUES ('Tom', ARRAY['Football', 'Basketball']);

-- 也可以使用 {}
INSERT INTO user_hobbies (name, hobbies)
VALUES ('Tom', "{'Football', 'Basketball'}");

-- 查询操作
SELECT
    name,
    hobbies
FROM
    user_hobbies;
    
-- PostgreSQL 中的数组下标是从 1 开始的
SELECT
  name,
  hobbies[1]
FROM
  user_hobbies;
  
-- 使用 any() 过滤数组
SELECT
  name,
  hobbies
FROM
  user_hobbies
WHERE
  'Football' = ANY (hobbies);
  
-- 更新数组中的元素
UPDATE user_hobbies
SET hobbies[2] = 'Baseball'
WHERE ID = 1;

-- 更新整个数组
UPDATE user_hobbies
SET hobbies = '{"Baseball"}'
WHERE ID = 1*;
```



#### 数组类型对照

|         JAVA TYPE         | SUPPORTED BINARY POSTGRESQL TYPES | DEFAULT POSTGRESQL TYPE |
| :-----------------------: | :-------------------------------: | :---------------------: |
|   `short[] `, `Short[]`   |             `int2[]`              |        `int2[]`         |
|   `int[] `, `Integer[]`   |             `int4[]`              |        `int4[]`         |
|    `long[] `, `Long[]`    |             `int8[]`              |        `int8[]`         |
|   `float[] `, `Float[]`   |            `float4[]`             |       `float4[]`        |
|  `double[] `, `Double[]`  |            `float8[]`             |       `float8[]`        |
| `boolean[] `, `Boolean[]` |             `bool[]`              |        `bool[]`         |
|        `String[]`         |      `varchar[] `, `text[]`       |       `varchar[]`       |
|        `byte[][]`         |             `bytea[]`             |        `bytea[]`        |

…

---

## 数据操作

### 从修改行中返回数据

```postgresql
CREATE TABLE users (firstname text, lastname text, id serial primary key);

INSERT INTO users (firstname, lastname) VALUES ('Joe', 'Cool') RETURNING id;

UPDATE products SET price = price * 1.10
  WHERE price <= 99.99
  RETURNING name, price AS new_price; -- RETURNING 的数据是被修改行的新内容
  
DELETE FROM products
  WHERE obsoletion_date = 'today'
  RETURNING *; -- RETURNING 的数据是被删除行的内容
```

…

---

## 列特性



### 生成列

生成的列是一个特殊的列，生成列的数据总是从其他列计算而来。生成列有两种：存储列和虚拟列。 

* 存储生成列在写入(插入或更新)时计算，并且像普通列一样占用存储空间。
* 虚拟生成列不占用存储空间并且在读取时进行计算。

PostgreSQL 目前只实现了存储生成列。

```sql
CREATE TABLE people (
    ...,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED --必须指定关键字 STORED 以指定存储生成列
);
```

> 生成列不能在`INSERT` 或 `UPDATE`命令中被直接写入, 不能为生成列指定值, 但是可以指定关键字`DEFAULT`。



**使用限制**

* 生成表达式不能引用另一个生成列。
* 生成表达式不能引用系统表。
* 生成列不能是分区键的一部分。





### 约束

#### 使用

话不多说，上代码：

```postgresql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0)
);
```

为了清晰一点， 也可以加上 `CONSTRAINT` 关键字：

```postgresql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CONSTRAINT positive_price CHECK (price > 0)
);
```

一个检查约束也可以引用多个列

```postgresql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);

-- 也可以写成
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CHECK (price > discounted_price) --或者同一行 CHECK (discounted_price > 0 AND price > discounted_price)
);
```

<br>

#### 约束类型

* **非空约束**：NOT NULL（很常见，不展开）。

* **唯一约束**：UNIQUE。

  ```postgresql
  -- 同一行语法
  CREATE TABLE products (
      product_no integer UNIQUE
  );
  
  -- 表约束语法
  CREATE TABLE products (
      product_no integer,
      UNIQUE (product_no)
  );
  -- 定义一组唯一约束
  CREATE TABLE example (
      a integer,
      b integer,
      c integer,
      UNIQUE (a, c)
  );
  ```

* **主键约束**：PRIMARY KEY。

  ```postgresql
  -- 单列主键
  CREATE TABLE products (
      product_no integer PRIMARY KEY
  );
  
  -- 多列主键
  CREATE TABLE example (
      a integer,
      b integer,
      c integer,
      PRIMARY KEY (a, c)
  );
  ```

* **外键约束**：REFERENCES。

  ```postgresql
  -- 产品表
  CREATE TABLE products (
      product_no integer PRIMARY KEY,
      name text,
      price numeric
  );
  -- 订单表
  CREATE TABLE orders (
      order_id integer PRIMARY KEY,
      product_no integer REFERENCES products (product_no),
      quantity integer
  );
  
  -- 可以将外键约束的列省略，默认使用被引用表的主键作为外键。
  CREATE TABLE orders (
      order_id integer PRIMARY KEY,
      product_no integer REFERENCES products,
      quantity integer
  );
  
  -- 也可以使用 mysql-like 方式
  CREATE TABLE t1 (
    a integer PRIMARY KEY,
    b integer,
    c integer,
    FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
  );
  
  -- 外键约束相关操作
  CREATE TABLE products (
      product_no integer PRIMARY KEY,
      name text,
      price numeric
  );
  CREATE TABLE orders (
      order_id integer PRIMARY KEY,
      shipping_address text,
      ...
  );
  CREATE TABLE order_items (
      product_no integer REFERENCES products ON DELETE RESTRICT, -- 限制删除
      order_id integer REFERENCES orders ON DELETE CASCADE, -- 级联删除
      quantity integer,
      PRIMARY KEY (product_no, order_id)
  );
  ```

* **排他约束**：EXCLUDE。

  下面实例创建了一张 COMPANY7 表，添加 5 个字段，并且使用了 EXCLUDE 约束。

  ```postgresql
  CREATE TABLE COMPANY7(
     ID INT PRIMARY KEY     NOT NULL,
     NAME           TEXT,
     AGE            INT  ,
     ADDRESS        CHAR(50),
     SALARY         REAL,
     EXCLUDE USING gist
     (NAME WITH =,  -- 如果满足 NAME 相同，AGE 不相同则不允许插入，否则允许插入
     AGE WITH <>)   -- 其比较的结果是如果整个表达式返回 true，则不允许插入，否则允许
  );
  ```

  > 其中 `USING gist` 是用于构建和执行的索引一种类型。*需要为每个数据库执行一次 `CREATE EXTENSION btree_gist` 命令安装 btree_gist 扩展，它定义了对纯标量数据类型的 EXCLUDE 约束。*

  由于我们已经强制执行了年龄必须相同，让我们通过向表插入记录来查看这一点：

  ```postgresql
  INSERT INTO COMPANY7 VALUES(1, 'Paul', 32, 'California', 20000.00 );
  INSERT INTO COMPANY7 VALUES(2, 'Paul', 32, 'Texas', 20000.00 );
  -- 此条数据的 NAME 与第一条相同，且 AGE 与第一条也相同，故满足插入条件
  INSERT INTO COMPANY7 VALUES(3, 'Allen', 42, 'California', 20000.00 );
  -- 此数据与上面数据的 NAME 相同，但 AGE 不相同，故不允许插入
  ```

<br>

### 系统列

每一个表都拥有一些由系统隐式定义的系统列。事实上用户不需要关心这些列，只需要知道它们存在即可。

| 列名         | 备注            | 描述                                                         |
| ------------ | --------------- | ------------------------------------------------------------ |
| oid/tableiod | table object id | 包含这一行的表的 OID                                         |
| xmin         |                 | 插入该行版本的事务身份（事务ID）                             |
| xmax         |                 | 删除事务的身份（事务ID）                                     |
| cmin         |                 | 插入事务中的命令标识符（从0开始）                            |
| cmax         |                 | 删除事务中的命令标识符                                       |
| ctid         |                 | 行版本在其表中的物理位置。尽管 ctid 可以被用来非常快速地定位行版本，但是一个行的 ctid 会在被更新或者被 `VACUUM FULL` 移动时改变。因此，`ctid` 不能作为一个长期行标识符。 |

…

---

<br>

## 表特性

### 表继承

直接上 SQL 代码：

```sql
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```

上面这段代码中 capitals 表继承 cities，同时还新增了一个 state 字段。

如果执行：

```sql
select ... from cities;
```

会同时查询 cities 和 capitals 两个表。如果只想查询 cities 表，不查询它的子表，可以使用 ONLY 关键字。例如：

```sql
select ... from ONLY cities;
```

除了 Select，Update 和 Delete 关键字也都支持 ONLY。



### 表分区

表分区指的是将逻辑上的一个大表分成一些小的物理上的片。分区的好处：

* 在某些情况下查询性能能够显著提升。
* 当查询或更新访问单个分区中的某一部分数据时，可以通过使用该分区的顺序扫描来提高性能（索引是在整个表中的随机访问读取）。



**PostgreSQL 分区形式**

* 范围分区，表被根据一个关键列或一组列划分为“范围”，不同的分区的范围之间没有重叠。相邻分区可以共享一个边界值，范围上限被视为不包含的边界。
* 列表分区，通过显式地列出每一个分区中出现的键值来划分表。
* 哈希分区，通过为每个分区指定模数和余数来对表进行分区。



**自定义分区创建**

```postgresql
CREATE TABLE measurement (
    id int not null,
    logdate date not null
) PARTITION BY RANGE (logdate);
```

…

---

<br>

## 高级特性

### 窗口函数

窗口函数是 SQL 函数，可以对当前行或当前行周围的若干行执行计算。一个窗口函数调用总是包含一个 OVER 子句，OVER 关键字可以指定窗口函数的范围，包括分区和排序顺序。

* `PARTITION BY` 对具有相同值的字段进行分区。对于每一行，窗口函数都会在当前行同一分区的行上进行计算。

  看下面的例子：

  ```sql
  SELECT depname, empno, salary, 
  avg(salary) OVER (PARTITION BY depname) 
  FROM employees;
  ```

  将相同部门的员工分为一个区，每次执行 avg() 都只是针对一个区进行的。看一下输出结果：

  ```bash
    depname  | empno | salary |          avg
  -----------+-------+--------+-----------------------
   develop   |    11 |   5200 | 5020.0000000000000000
   develop   |     7 |   4200 | 5020.0000000000000000
   develop   |     9 |   4500 | 5020.0000000000000000
   develop   |     8 |   6000 | 5020.0000000000000000
   develop   |    10 |   5200 | 5020.0000000000000000
   personnel |     5 |   3500 | 3700.0000000000000000
   personnel |     2 |   3900 | 3700.0000000000000000
   sales     |     3 |   4800 | 4866.6666666666666667
   sales     |     1 |   5000 | 4866.6666666666666667
   sales     |     4 |   4800 | 4866.6666666666666667
  (10 rows)
  ```

  结果和预期一样，先将部门分区，然后求每个部门的平均薪资。

* `ORDER BY` 控制窗口函数处理行的顺序。

  将上面的语句稍作修改：

  ```sql
  # rank 不需要显式的参数，因为它的行为完全决定于 OVER 子句。
  SELECT depname, empno, salary,
  rank() OVER (PARTITION BY depname ORDER BY salary DESC) FROM employees;
  ```

  输出结果如下：

  ```bash
    depname  | empno | salary | rank
  -----------+-------+--------+------
   develop   |     8 |   6000 |    1
   develop   |    10 |   5200 |    2
   develop   |    11 |   5200 |    2
   develop   |     9 |   4500 |    4
   develop   |     7 |   4200 |    5
   personnel |     2 |   3900 |    1
   personnel |     5 |   3500 |    2
   sales     |     1 |   5000 |    1
   sales     |     4 |   4800 |    2
   sales     |     3 |   4800 |    2
  (10 rows)
  ```

  先将部门分区，再对部门中员工的薪水进行排序。

* …

---

## 索引

### 索引类型

#### B-tree 索引

涉及到以下操作可以考虑使用 B-tree 索引：

* `<` 、 `<=`  、 `=` 、 `>=`  、 `>`

* BETWEEN 和 IN

* 在索引列上的 IS NULL 或 IS NOT NULL 条件

* LIKE 和 `~`

  但是 B-tree 索引只使用下面的情况

  ```postgresql
  column_name LIKE 'foo%'
  column_name LKE 'bar%'
  column_name  ~ '^foo'
  ```

  而不适用于下面的情况：

  ```postgresql
  col LIKE '%bar'
  ```



#### 哈希索引

每当索引列使用 `=` 运算符进行比较时，查询计划器将考虑使用哈希索引。

创建哈希索引，需要指定 `USING HASH`：

```postgresql
CREATE INDEX index_name
ON table_name USING HASH (indexed_column);
```



#### GIN 索引

> 通用倒排索引，Generalized Inverted Index。

正排索引查找时扫描表中每个文档，直到找出所有包含查询关键字的文档。倒排表以*字*或*词*为关键字作为索引，对应记录表中出现这个字或词的所有文档。由于每个字或词对应的文档数量是动态变化的，所以倒排表的建立和维护都较为复杂，但是在查询的时候效率高于正排表。



#### BRIN 索引

> 块范围索引，Block Range Indexes。



#### GiST 索引

GiST 索引不是单独一种索引类型，而是一种架构，可以在这种架构上实现很多不同的索引策略。

GiST 代表广义搜索树。GiST 索引允许构建通用的树结构。GiST 索引可用于索引几何数据类型和全文搜索。



#### SP-GiST 索引

> SP-GiST 代表空间分区的 GiST。

…

---

### 创建索引

```postgresql
CREATE [ UNIQUE ] INDEX [ [ IF NOT EXISTS ] index_name ] -- 自定义索引名称
    ON table_name [ USING method ] -- method 是索引方法名称，包括 btree, hash, gist, spgist, gin, 和 brin。默认使用 btree。
(
    column_name [ ASC | DESC ] [ NULLS { FIRST | LAST } ] -- column_name 表示需要创建索引的列名
    [, ...]
);

create index on table_name(colum_name);
create index my_hash_idx on tb_test using hash(colum_name); -- 创建 hash 索引

-- 多列索引，和 MySQL 一样遵循最左匹配原则
CREATE INDEX index_name
ON table_name(a, b, c);
```



### 部分索引

> 按照条件创建索引

```postgresql
-- 对 customer 表中 active = 0 的列创建索引
CREATE INDEX idx_customer_inactive
ON customer(active)
WHERE active = 0;
```



### 重建索引

* 重建单个索引

  ```postgresql
  REINDEX INDEX index_name;
  ```

* 重建表中的所有索引

  ```postgresql
  REINDEX TABLE table_name;
  ```

* 重建数据库中的所有索引

  ```postgresql
  REINDEX DATABASE database_name;
  ```

* …

---

## 执行分析

> * 来自 [polardb 数据库内核月报](http://mysql.taobao.org/monthly/)：http://mysql.taobao.org/monthly/2018/11/06/
> * https://www.modb.pro/db/101529

…

---

<br>

# PG 高可用架构

## PG 安装

> Docker/K8s 环境下部署不好控制，选择直接安装在服务器上。

1、[Ubuntu install](https://www.postgresql.org/download/linux/ubuntu/)

2、`psql -V` 查看版本

3、切换用户，并登录

```shell
sudo -u postgres psql
```

* 配置路径：`/etc/postgresql/16/main/`
* 数据路径：`/var/lib/postgresql/16/main/`
* 配置模板路径：`/usr/share/postgresql/16/main/`
* pg 命令行工具：`/usr/lib/postgresql/16/bin`

> 可能会遇到 `/var/lib/postgresql/` 目录下 *operation not permitted* 的问题，解决方法：
>
> * 一次性修改：`chown -R postgres:postgres /var/lib/postgresql/`
> * 先修改 owner：`chown -R postgres /var/lib/postgresql/`，再修改 group：`chown -R postgres /var/lib/postgresql/`

4、创建用户

```postgresql
CREATE USER root superuser login PASSWORD '123456';
```

5、开启远程访问

5.1、修改 pg_hba.conf

```text
# TYPE DATABASE USER ADDRESS METHOD
host all all 0.0.0.0/0 trust
```

> trust 表示不需要密码，如果需要密码则改为 md5

5.2、修改 postgresql.conf

```text
listen_addresses = '*'
```

5.3、重启服务

```shell
sudo service postgresql restart
# or
sudo /etc/init.d/postgresql restart
# or
sudo systemctl restart postgresql
```

> 现在就可以通过远程访问数据库了

6、创建测试数据

```postgresql
CREATE DATABASE test OWNER root;

create table myuser (
    id serial primary key,
    name varchar(20) not null,
    age int not null
);

insert into myuser (name, age) values ('zhangsan', 20);
insert into myuser (name, age) values ('lisi', 21);
insert into myuser (name, age) values ('wangwu', 22);
```

…

---

## 主从流复制

### 配置

7、主节点创建同步账号

```postgresql
CREATE USER repl REPLICATION LOGIN ENCRYPTED PASSWORD '123456';
```

8、修改 pg_hba.conf
```text
# TYPE DATABASE USER ADDRESS METHOD
host all repl 0.0.0.0/0 trust
```

9、修改 postgresql.conf
> [按照 Cc 的说法](https://www.net-cc.com/posts/linux-software/postgresql/02-postgresql-master_slave-replication/)：主从服务器都要设置, 万一那天从服务器提升为主了呢
```text
listen_addresses = '*'
wal_level = replica
archive_mode = on
#archive_command = '/bin/true'
archive_command = '{ sleep 5; true; }' # 9.5 后淘汰，或许可以尝试不填写？
max_wal_senders = 10
max_replication_slots = 10 # (修改) 设置支持的复制槽数量
max_slot_wal_keep_size = 1GB # (修改) 设置复制槽保留的 wal 最大值,默认单位是 M
hot_standby = on

#wal_sender_timeout = 60s
#wal_keep_segments = 64
#hot_standby_feedback = on # 如果有错误的数据复制向主进行反馈
#synchronous_commit = on  # 开启同步复制

#log_truncate_on_rotation = 'on' # 日志滚动
#log_rotation_age = '1d'
#log_rotation_size = 0
```

10、重启主节点

11、从节点同步主节点数据
11.1、停止从节点

```shell
sudo service postgresql stop
```

11.2、在从节点上备份主节点数据

```shell
pg_basebackup -h <master-host> -U repl -p 5432 -F p  -X stream -v -P -R -D /data/pgsql -C -S slave01 -l slave01

# -D 指定备份数据存放的目录
# -F, --format=p|t 输出格式 (纯文本 (缺省值), tar压缩格式)
# -X, --wal-method=none|fetch|stream 按指定的模式包含必需的WAL日志文件
# -v, --verbose 输出详细的消息
# -P, --progress 显示进度信息
# -R, --write-recovery-conf 为复制写配置文件
# -C, --create-slot 创建复制槽
# -S, --slot=SLOTNAME 用于复制的槽名
# -l, --label=LABEL 设置备份标签

# 备份从节点原先的数据
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.bak

# 使用主节点同步过来的数据代替从节点原先的数据
mv /data/pgsql /var/lib/postgresql/16/main

chown -R postgres:postgres -R /var/lib/postgresql/16/main
```

```postgresql
-- 复制槽状态检查
select * from pg_replication_slots;

-- 创建复制槽
select * from pg_create_physical_replication_slot('slave01');

-- 删除复制槽
select * from pg_drop_replication_slot('slave01');
```

11.2、启动从节点，并检查从节点状态

```shell
sudo service postgresql start
```

```postgresql
-- 节点状态检查
-- 查询结果为"f"表示主库, 't'表示从库
select pg_is_in_recovery();
```

```postgresql
-- 在主节点检查同步状态
select * from pg_stat_replication;
```

输出内容大概如下：
```shell
postgres=# select * from pg_stat_replication \g
-[ RECORD 1 ]----+------------------------------
pid              | 2299
usesysid         | 16388
usename          | repl
application_name | 16/main
client_addr      | 192.168.111.14
client_hostname  |
client_port      | 48412
backend_start    | 2023-10-13 03:11:24.319331+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000148
write_lsn        | 0/3000148
flush_lsn        | 0/3000148
replay_lsn       | 0/3000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2023-10-13 03:12:24.342696+00
```

12、尝试在主库插入数据，在从库查询。从库有记录表示主从复制配置完成。多配置几个从节点就是一主多从架构了。

---

**主节点宕机会发生什么？**

1、假设现在为一主二从的状态

```shell
postgres=# select * from pg_replication_slots;
-[ RECORD 1 ]-------+----------
slot_name           | slave01
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 2739
xmin                |
catalog_xmin        |
restart_lsn         | 0/7000330
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
conflicting         |
-[ RECORD 2 ]-------+----------
slot_name           | slave02
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 3553
xmin                |
catalog_xmin        |
restart_lsn         | 0/7000330
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
conflicting         |
```

2、让主节点下线

3、主节点下线后查看从节点状态

```shell
postgres=# select pg_is_in_recovery();
-[ RECORD 1 ]-----+--
pg_is_in_recovery | t
```

可以看到依然是从库，需要手动选择某一个从库提升为主库
```shell
cd /usr/lib/postgresql/16/bin
./pg_ctl promote -D /var/lib/postgresql/16/main
```

输出内容大概如下：
```shell
waiting for server to promote.... done
server promoted
```

再次检查该从节点状态
```shell
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 f
(1 row)
```

理论上来说可以这样子操作：

> 该从节点已经变成主节点了，接下来为对应的从节点创建复制槽
>
> ```postgresql
> select * from pg_create_physical_replication_slot('slave0X');
> 
> -- 查看复制槽状态
> select * from pg_replication_slots;
> ```
>
> 最后编辑其他从节点的 `postgresql.auto.conf` 文件，修改 `primary_conninfo` 为新的主节点地址，重启从节点即可。

但是我操作一番并未成功，可能需要再走一遍之前的步骤 11。

---

### 总结

主从流复制不需要借助其他第三方工具，直接配置即可。但是配置稍繁琐，而且如果主节点宕机了从节点不会自动切换，需要手动切换。

---

<br>

## 主从流复制 + repmgr

> *Replication Manager* 简称 repmgr。

### repmgr 配置

1、首先查看 repmgr 与 PostgreSQL 的[版本对应关系](https://www.repmgr.org/docs/current/install-requirements.html)，并[安装](https://www.repmgr.org/docs/current/installation-packages.html#INSTALLATION-PACKAGES-DEBIAN)。

此处以 pg-15 为例：

```shell
sudo apt install postgresql-15

curl https://dl.enterprisedb.com/default/release/get/deb | sudo bash
sudo apt install install postgresql-15-repmgr

# 注意：安装完成第一件事是取消 postgresql 开机自启
systemctl disable postgresql
```

2、修改 `/etc/postgresql/15/main/postgresql.conf` 和 `/etc/postgresql/15/main/pg_hba.conf`，允许外部 IP 访问。并重启服务。

3、每个节点  `/etc/postgresql/15/main/postgresql.conf` 配置修改

> 因为不确定故障转移的时候谁会是下一个主节点，全部修改比较保险。

```text
# Ensure WAL files contain enough information to enable read-only queries
# on the standby.
#
#  PostgreSQL 9.5 and earlier: one of 'hot_standby' or 'logical'
#  PostgreSQL 9.6 and later: one of 'replica' or 'logical'
#    ('hot_standby' will still be accepted as an alias for 'replica')
#
# See: https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-WAL-LEVEL

wal_level = 'replica'

# Enable WAL file archiving
#
# See: https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-ARCHIVE-MODE

archive_mode = on

# Set archive command to a dummy command; this can later be changed without
# needing to restart the PostgreSQL instance.
#
# See: https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-ARCHIVE-COMMAND

#archive_command = '/bin/true'
archive_command = '{ sleep 5; true; }'

# Enable replication connections; set this value to at least one more
# than the number of standbys which will connect to this server
# (note that repmgr will execute "pg_basebackup" in WAL streaming mode,
# which requires two free WAL senders).
#
# See: https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-WAL-SENDERS

max_wal_senders = 10

# If using replication slots, set this value to at least one more
# than the number of standbys which will connect to this server.
# Note that repmgr will only make use of replication slots if
# "use_replication_slots" is set to "true" in "repmgr.conf".
# (If you are not intending to use replication slots, this value
# can be set to "0").
#
# See: https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-MAX-REPLICATION-SLOTS

max_replication_slots = 10

# Enable read-only queries on a standby
# (Note: this will be ignored on a primary but we recommend including
# it anyway, in case the primary later becomes a standby)
#
# See: https://www.postgresql.org/docs/current/runtime-config-replication.html#GUC-HOT-STANDBY

hot_standby = on
```

4、主节点创建 repmgr 用户以及数据库来保存集群元数据

```shell
# 先切换到 postgres 用户
sudo su -- postgres
cd ~
# 再创建用户
createuser -s repmgr

createdb repmgr -O repmgr
```

5、编辑 `/etc/postgresql/15/main/pg_hba.conf`，为 repmgr 用户添加访问权限

> 主从节点都需要，因为从节点克隆主节点的数据会将 repmgr 用户和库一同克隆。

```text
local repmgr repmgr trust
host repmgr repmgr 127.0.0.1/32 trust
host repmgr repmgr 0.0.0.0/0 trust
```

> 上述配置是在测试环境下的，如果生产环境配置最好限制可远程访问的 IP 段，例如 `192.168.1.0/24`。

6、在主节点下创建 `repmgr.conf` 配置 `/etc/repmgr.conf`

```
node_id=1
node_name='node1'
conninfo='host=192.168.111.12 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/15/main'
```

> `repmgr.conf` should ***not*** be stored inside the PostgreSQL data directory, as it could be overwritten when setting up or reinitialising the PostgreSQL server. 

7、主节点注册到 repmgr

```shell
$ repmgr -f /etc/repmgr.conf primary register
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (ID: 1) registered

# 检查节点信息
$ repmgr -f /etc/repmgr.conf cluster show
```

9、从节点准备

9.1、安装 PG-15 和 postgresql-15-repmgr

9.2、停止 PG-15，编辑 `postgresql.conf` 开启远程监听，编辑 `pg_hba.conf` 允许用于远程访问

9.3、将数据目录 `/var/lib/postgresql/15/main` 备份到  `/var/lib/postgresql/15/backup` ，新建 main 文件夹，存放从主节点 clone 的数据。注意新建的 main 文件夹的用户和用户组设置为 `postgres`

> **Note**
>
> On the standby, do ***not*** create a PostgreSQL instance (i.e. do not execute initdb or any database creation scripts provided by packages), but do ensure the destination data directory (and any other directories which you want PostgreSQL to use) exist and are owned by the `postgres` system user. Permissions must be set to `0700` (`drwx------`).
>
> **Because**
>
> repmgr will place a copy of the primary's database files in this directory. It will however refuse to run if a PostgreSQL instance has already been created there.

使用 psql 命令检查从节点是否可访问到主节点

```shell
 psql 'host=<node1-host> user=repmgr dbname=repmgr connect_timeout=2'
```

9.4、创建配置 `/etc/repmgr.conf`

```
node_id=2
node_name='node2'
conninfo='host=192.168.111.12 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/15/main'
```

9.5、使用 `--dry-run` 参数检查是否可以从主节点复制

```shell
$ repmgr -h <node1-host> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone --dry-run

NOTICE: destination directory "/var/lib/postgresql/15/main" provided
INFO: connecting to source node
DETAIL: connection string is: host=192.168.111.12 user=repmgr dbname=repmgr
DETAIL: current installation size is 37 MB
INFO: "repmgr" extension is installed in database "repmgr"
INFO: replication slot usage not requested;  no replication slot will be set up for this standby
INFO: parameter "max_wal_senders" set to 10
NOTICE: checking for available walsenders on the source node (2 required)
INFO: sufficient walsenders available on the source node
DETAIL: 2 required, 10 available
NOTICE: checking replication connections can be made to the source server (2 required)
INFO: required number of replication connections could be made to the source server
DETAIL: 2 replication connections required
WARNING: data checksums are not enabled and "wal_log_hints" is "off"
DETAIL: pg_rewind requires "wal_log_hints" to be enabled
NOTICE: standby will attach to upstream node 1
HINT: consider using the -c/--fast-checkpoint option
INFO: would execute:
  pg_basebackup -l "repmgr base backup"  -D /var/lib/postgresql/15/main -h 192.168.111.12 -p 5432 -U repmgr -X stream
INFO: all prerequisites for "standby clone" are met
```

> 可能会提示：`no pg_hba.conf entry for replication connection from host ...`，配置 `pg_hba.conf`：
>
> ```text
> host replication all 0.0.0.0/0 trust
> ```
>
> 同样的，生产环境下注意缩小 IP 访问范围。

9.6、如果上述步骤无误，则直接从主节点克隆数据

```shell
$ repmgr -h <node1-host> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone

NOTICE: destination directory "/var/lib/postgresql/15/main" provided
INFO: connecting to source node
DETAIL: connection string is: host=192.168.111.12 user=repmgr dbname=repmgr
DETAIL: current installation size is 37 MB
INFO: replication slot usage not requested;  no replication slot will be set up for this standby
NOTICE: checking for available walsenders on the source node (2 required)
NOTICE: checking replication connections can be made to the source server (2 required)
WARNING: data checksums are not enabled and "wal_log_hints" is "off"
DETAIL: pg_rewind requires "wal_log_hints" to be enabled
INFO: creating directory "/var/lib/postgresql/15/main"...
NOTICE: starting backup (using pg_basebackup)...
HINT: this may take some time; consider using the -c/--fast-checkpoint option
INFO: executing:
  pg_basebackup -l "repmgr base backup"  -D /var/lib/postgresql/15/main -h 192.168.111.12 -p 5432 -U repmgr -X stream
could not change directory to "/home/gnl": Permission denied
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: pg_ctl -D /var/lib/postgresql/15/main start
HINT: after starting the server, you need to register this standby with "repmgr standby register"
```

> 1. 可以看到熟悉的 `pg_basebackup` 命令，实际上 repmgr 底层就是使用 `pg_basebackup` 命令来帮助从节点克隆数据的。
> 2. 从提示可以看到 `NOTICE: you can now start your PostgreSQL server`
> 3. 从提示可以看到 `HINT: after starting the server, you need to register this standby with "repmgr standby register"`

9.7、从节点编辑 repmgr 用户远程访问权限

> 主从节点切换会使用到，如果前面编辑过了可以跳过这一步。

```shell
local repmgr repmgr trust
host repmgr repmgr 127.0.0.1/32 trust
host repmgr repmgr 0.0.0.0/0 trust
```

9.8、启动从节点上的 PG

10、主从节点状态查询

启动完成后在主节点查询

```postgresql
postgres=# \x
Expanded display is on.
postgres=# SELECT * FROM pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 6011
usesysid         | 16388
usename          | repmgr
application_name | node2
client_addr      | 192.168.111.11
client_hostname  |
client_port      | 56912
backend_start    | 2023-10-16 14:02:30.04542+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/B000368
write_lsn        | 0/B000368
flush_lsn        | 0/B000368
replay_lsn       | 0/B000368
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2023-10-16 14:03:05.786353+08
```

可以看到有一个 node2 节点已经连接上来了。

在从库查询 wal receiver 状态：

```postgresql
postgres=# SELECT * FROM pg_stat_wal_receiver;

-[ RECORD 1 ]---------+
pid                   | 21444
status                | streaming
receive_start_lsn     | 0/B000000
receive_start_tli     | 1
written_lsn           | 0/B000368
flushed_lsn           | 0/B000368
received_tli          | 1
last_msg_send_time    | 2023-10-16 14:04:35.904542+08
last_msg_receipt_time | 2023-10-16 14:04:35.880378+08
latest_end_lsn        | 0/B000368
latest_end_time       | 2023-10-16 14:02:30.053889+08
slot_name             |
sender_host           | 192.168.111.12
sender_port           | 5432
conninfo              | user=repmgr passfile=/var/lib/postgresql/.pgpass 
```

11、将从节点注册到 repmgr 上

```shell
$ repmgr -f /etc/repmgr.conf standby register

INFO: connecting to local node "node2" (ID: 2)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID: 1)
INFO: standby registration complete
NOTICE: standby node "node2" (ID: 2) successfully registered
```

12、检查集群状态

主节点或者从节点

```shell
$ repmgr -f /etc/repmgr.conf cluster show
```

![image-20231017135533652](./assets/image-20231017135533652.png)

可以看到现在我们的集群中有两个节点了。

---

### 主从切换（手动）

> **repmgr 集群正确的启动方式**
>
> 如果使用 `syetmctl {start|stop|restart} postgresql` 或者 `service postgresql {start|stop|restart}` 在进行集群操作切换到 postrges 用户之后会无权限控制 PostgreSQL 的状态，需要改成使用 `pg_ctlcluster 15 main {start|stop|restart|status}` 来控制 PostgreSQL。

14、编辑 `/etc/repmgr.conf`，添加配置

```text
service_start_command='sudo pg_ctlcluster 15 main start'
service_stop_command='sudo pg_ctlcluster 15 main stop'
service_restart_command='sudo pg_ctlcluster 15 main restart'
service_reload_command  = 'sudo pg_ctlcluster 15 main reload'
```

为 postgres 用户添加权限，`sudo vim /etc/sudoers`

```text
Defaults:postgres !requiretty
postgres ALL= NOPASSWD: /usr/bin/pg_ctlcluster 15 main start
postgres ALL= NOPASSWD: /usr/bin/pg_ctlcluster 15 main stop
postgres ALL= NOPASSWD: /usr/bin/pg_ctlcluster 15 main restart
postgres ALL= NOPASSWD: /usr/bin/pg_ctlcluster 15 main reload
```

停止所有的 PG 实例，该用 `pg_ctlcluster` 来启动。

15、主从节点无密码 SSH 连接配置

> root 和 postgres 用户都需要设置，主从节点需要双向配置（主节点 SSH 无密码连接从节点，从节点 SSH 无密码连接主节点）

15.1、生成 SSH

```shell
# root
ssh-keygen -t rsa
cd /root/.ssh
cat id_dsa.pub >> root_authorized_keys

# postgres
ssh-keygen -t rsa
cd /var/lib/postgresql/.ssh
cat id_dsa.pub >> pg_authorized_keys
```



15.2、传输到服务器上（传输到需要无密码登录的机器上）

```shell
scp /root/.ssh/root_authorized_keys user@server_addr:/home/user

# 目标服务器
mv /home/user/root_authorized_keys /root/.ssh/authorized_keys
mv /home/user/pg_authorized_keys /var/lib/postgresql/.ssh/authorized_keys
```

> 如果知道登录用户的密码
>
> ```shell
> ssh-keygen -t rsa
> ssh-copy-id user@server_address 
> # 会传输保存到到对应账户的 `~/.ssh/authorized_keys` 中。
> ```



15.3、配置服务器 SSH 允许无密码登录

```shell
sudo vim /etc/ssh/sshd_config 
```

修改

```text
PasswordAuthentication no
```

15.4、重启 SSH 服务

```shell
sudo systemctl restart sshd
```

15.5、登录测试

```shell
ssh user@server_address 
```

如果有两台服务器192.168.111.11 和 192.168.111.12，两台服务器都需要能成功执行以下命令

```shell
# 192.168.111.11
root:~ ssh root@192.168.111.12
postgres:~ ssh postgres@192.168.111.12

# 192.168.111.12
root:~ ssh root@192.168.111.11
postgres:~ ssh postgres@192.168.111.11
```

无需密码证明配置成功。

16、停止 PG node1，模拟宕机

17、使用 `–dry-run` 尝试将 node2 迁移为 primary

```shell
$ repmgr -f /etc/repmgr.conf standby switchover --dry-run

NOTICE: checking switchover on node "node1" (ID: 1) in --dry-run mode
INFO: SSH connection to host "192.168.111.11" succeeded
INFO: able to execute "repmgr" on remote host "192.168.111.11"
INFO: 1 walsenders required, 10 available
INFO: demotion candidate is able to make replication connection to promotion candidate
INFO: 0 pending archive files
INFO: replication lag on this standby is 0 seconds
NOTICE: attempting to pause repmgrd on 2 nodes
NOTICE: local node "node1" (ID: 1) would be promoted to primary; current primary "node2" (ID: 2) would be demoted to standby
INFO: following shutdown command would be run on node "node2":
  "sudo pg_ctlcluster 15 main stop"
INFO: parameter "shutdown_check_timeout" is set to 60 seconds
INFO: prerequisites for executing STANDBY SWITCHOVER are met
```

18、standby 切换为 primary（需要在 standby 机器上操作）

```shell
$ repmgr -f /etc/repmgr.conf standby switchover
```

报提示 `NOTICE: waiting up to 30 seconds (parameter "wal_receive_check_timeout") for received WAL to flush to disk`

```
NOTICE: waiting up to 30 seconds (parameter "wal_receive_check_timeout") for received WAL to flush to disk
INFO: sleeping 1 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 2 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 3 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 4 of maximum 30 seconds waiting for standby to flush received WAL to disk
INFO: sleeping 5 of maximum 30 seconds waiting for standby to flush received WAL to disk
...
INFO: sleeping 30 of maximum 30 seconds waiting for standby to flush received WAL to disk
WARNING: local node "node2" is behind shutdown primary "node1"
DETAIL: local node last receive LSN is 0/5BA0000, primary shutdown checkpoint LSN is 0/6000028
NOTICE: aborting switchover
HINT: use --always-promote to force promotion of standby
```

主从节点修改 `postgresql.conf`

```
archive_command = '{ sleep 5; true; }'
```

`pg_ctlcluster` 重启主从节点，再次尝试

```shell
$ repmgr -f /etc/repmgr.conf standby switchover

NOTICE: executing switchover on node "node1" (ID: 1)
NOTICE: attempting to pause repmgrd on 2 nodes
NOTICE: local node "node1" (ID: 1) will be promoted to primary; current primary "node2" (ID: 2) will be demoted to standby
NOTICE: stopping current primary node "node2" (ID: 2)
NOTICE: issuing CHECKPOINT on node "node2" (ID: 2)
DETAIL: executing server command "sudo pg_ctlcluster 15 main stop"
INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
NOTICE: current primary has been cleanly shut down at location 0/13000028
NOTICE: promoting standby to primary
DETAIL: promoting server "node1" (ID: 1) using pg_promote()
NOTICE: waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
NOTICE: STANDBY PROMOTE successful
DETAIL: server "node1" (ID: 1) was successfully promoted to primary
NOTICE: node "node1" (ID: 1) promoted to primary, node "node2" (ID: 2) demoted to standby
NOTICE: switchover was successful
DETAIL: node "node1" is now primary and node "node2" is attached as standby
NOTICE: STANDBY SWITCHOVER has completed successfully
```

19、切换成功，查看集群状态

![image-20231017091905095](./assets/image-20231017091905095.png)

20、如果显示  `primary | ! running as standby` 

```shell
$ repmgr -f /etc/repmgr.conf cluster show
 ID | Name  | Role    | Status               | Upstream | Location | Priority | Timeline | Connection string
----+-------+---------+----------------------+----------+----------+----------+----------+-----------------------------------------------------------------
 1  | node1 | primary | ! running as standby |          | default  | 100      | 2        | host=192.168.111.12 user=repmgr dbname=repmgr connect_timeout=2
 2  | node2 | primary | * running            |          | default  | 100      | 2        | host=192.168.111.11 user=repmgr dbname=repmgr connect_timeout=2
```

则需要将 node1 以 standby 的身份重新加入集群

```shell
repmgr -f /etc/repmgr.conf standby register --force
```

如果加入失败，则说明可能 node1 和 node 2 的数据不同步

1. 停止 node2，将 node2 的数据备份，重新从 node1 克隆。
2. 启动 node2，以 standby 的身份重新加入集群。

…

---

### 自动切换 repmgrd

> 上面展示的主从切换需要人为干预，通常情况下我们不一定能在主节点宕机的第一时间就感知到，并完成迅速切换，此时就需要配置自动切换。

> repmgrd（*replication manager daemon*） 是一个管理和监控守护进程，运行在 PostgreSQL 集群的每个节点上,可以自动执行故障转移和更新备用节点等操作以跟上新的主节点，并提供关于每个备用服务器状态的监视信息。

21、在每个节点上创建 repmgrd 的 systemd 配置文件 `/etc/systemd/system/repmgrd.service`

> 有两个位置需要注意 ExecStart 中定义的 repmgrd 命令的位置，repmgr 配置文件的位置，pid 文件的位置
>

```shell
[Unit]
Description=repmgrd.service
#After=syslog.target
After=network.target
#After=pgserver.service

[Service]
Type=forking

User=postgres
Group=postgres

# Where to send early-startup messages from the server
# This is normally controlled by the global default set by systemd
# StandardOutput=syslog
ExecStart=/usr/bin/repmgrd -f /etc/repmgr.conf -p /tmp/repmgrd.pid -d --verbose
ExecStop=/usr/bin/kill `cat /tmp/repmgrd.pid`
PrivateTmp=false
# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

22、修改 `postgresql.conf`

```
shared_preload_libraries = 'repmgr'
```

修改 `repmgr.conf`

```
failover='automatic'

promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

repmgrd_service_start_command = 'repmgrd --daemonize=true'
repmgrd_service_stop_command = 'kill `cat /tmp/repmgrd.pid`'
repmgrd_pid_file='/tmp/repmgrd.pid'
```



23、启动 repmgrd 服务

```shell
# root 用户
# enable 表示开启开机自启
systemctl enable repmgrd --now
# 启动 repmgrd
systemctl start repmgrd
# 检查启动状态
repmgrd -v
# 或者
ps f u postgres
```

24、查看当前集群状态

> 每个节点的 repmgrd 都要处于 running 状态。

```shell
postgres$ repmgr service status
 ID | Name  | Role    | Status    | Upstream | repmgrd | PID   | Paused? | Upstream last seen
----+-------+---------+-----------+----------+---------+-------+---------+--------------------
 1  | node1 | primary |   running |     | running | 15374 | no      | 0 second(s) ago
 2  | node2 | standby | * running | node1    | running | 12854 | no      | n/a
```

```shell
# 或者查看 postgres 用户启动的进程状态
ps f -u postgres
```

25、停止主节点模拟宕机

```shell
postgres@vm01:~/15$ pg_ctlcluster 15 main stop
```

一分钟后 node2 变成主节点

```shell
[2023-10-17 16:30:37] [INFO] 0 active sibling nodes registered
[2023-10-17 16:30:37] [INFO] 2 total nodes registered
[2023-10-17 16:30:37] [INFO] primary node  "node1" (ID: 1) and this node have the same location ("default")
[2023-10-17 16:30:37] [INFO] no other sibling nodes - we win by default
[2023-10-17 16:30:37] [NOTICE] this node is the only available candidate and will now promote itself
[2023-10-17 16:30:37] [INFO] promote_command is:
  "/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file"
[2023-10-17 16:30:37] [NOTICE] promoting standby to primary
[2023-10-17 16:30:37] [DETAIL] promoting server "node2" (ID: 2) using pg_promote()
[2023-10-17 16:30:37] [NOTICE] waiting up to 60 seconds (parameter "promote_check_timeout") for promotion to complete
[2023-10-17 16:30:39] [NOTICE] STANDBY PROMOTE successful
[2023-10-17 16:30:39] [DETAIL] server "node2" (ID: 2) was successfully promoted to primary
[2023-10-17 16:30:39] [INFO] checking state of node 2, 1 of 6 attempts
[2023-10-17 16:30:39] [NOTICE] node 2 has recovered, reconnecting
[2023-10-17 16:30:39] [INFO] connection to node 2 succeeded
[2023-10-17 16:30:39] [INFO] original connection is still available
[2023-10-17 16:30:39] [INFO] 0 followers to notify
[2023-10-17 16:30:39] [INFO] switching to primary monitoring mode
[2023-10-17 16:30:39] [NOTICE] monitoring cluster primary "node2" (ID: 2)
```

node1 的状态变成 failed

![image-20231017163424709](./assets/image-20231017163424709.png)

26、node1 重新加入集群（服务未启动的情况下）

```shell
repmgr -f /etc/repmgr.conf node rejoin -d'host=172.24.246.161 port=5432 user=repmgr dbname=repmgr connect_timeout=2'
```

27、重新启动 node1

![image-20231017163516599](./assets/image-20231017163516599.png)

> 如果在执行 rejoin 之前启动了 node1，那么可能会出现 node1 和 node2 时间线不一致的情况，导致无法 rejoin。因此注意要在故障的节点未启动的情况下先进行 rejoin，再启动节点。
>
> 如果不小心先于 rejoin 之前启动了 node1，就需要将 node1 的数据清空，重新从 node2 克隆，再重新以 standby 的身份加入集群。

…

---

### repmgrd + witness

**什么是 witness**

witness 服务是独立于集群的一个 PostgreSQL 实例，在出现故障转移的情况下，它可以当作目击者来证明是由于网络问题导致 primary 节点不可用还是 primary 本身出现故障导致的不可用。

> 作为 primary 出故障的目击证人

**使用场景**

一主一从部署，且主从节点位于不同的位置（数据中心）。在主节点相邻的位置创建 witness 节点，如果主节点变得不可用，是否应该让从节点提升未主节点？

* 如果从节点同时无法连接主节点和 witness，说明是网络原因导致的的故障，从节点不会被提升；
* 如果从节点无法连接主节点，但是可以连接 witness，说明是主节点自身原因导致的不可用，从节点会被提升为新的主节点。

在更为复杂的情况下，比如说有多个数据中心，应该确保只允许和主节点处于同一位置的节点被提升为新的主节点，[参考连接](https://www.repmgr.org/docs/current/repmgrd-network-split.html)。

> 只有在启动了 repmgrd 的情况下，witness 服务才有效。

**创建 witness 节点**

> witness 实例不能与主节点部署在同一台物理机上。

> 一个 PostgreSQL 实例只能作为一个集群的 witness 节点来使用，不能作为多个 witness 节点。

0、创建 repmgr 用户，创建 repmgr 数据库

1、配置 `repmgr.conf`

```
node_id=100
node_name='witness100'
conninfo='host=<witness-host> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/15/main'
```

3、修改 `pg_hba.conf`

修改 `postgresql.conf`，允许远程访问

6、启动 PostgreSQL 实例，以 primary 的身份加入**自己的集群**

```shell
repmgr primary register
```

![image-20231018103016642](./assets/image-20231018103016642.png)

7、启动 repmgrd

```shell
repmgrd -d
```

8、以 witness 身份注册到**主从复制集群**

```shell
repmgr witness register --force -h <primary-host>
```

![image-20231018103045843](./assets/image-20231018103045843.png)

部署完成。

…

---

### 完整 postgresql.conf

```
listen_addresses = '*'
wal_level = replica
archive_mode = on
#archive_command = '/bin/true'
archive_command = '{ sleep 5; true; }'
max_wal_senders = 10
max_replication_slots = 10 # (修改) 设置支持的复制槽数量
max_slot_wal_keep_size = 1GB # (修改) 设置复制槽保留的 wal 最大值,默认单位是 M
hot_standby = on

shared_preload_libraries = 'repmgr'
```

…

### 完整 pg_hba.conf

```
host replication all 0.0.0.0/0 trust
local repmgr repmgr trust
host repmgr repmgr 127.0.0.1/32 trust
host repmgr repmgr 0.0.0.0/0 trust
host all all 0.0.0.0/0 trust
```

…

### 完整 repmgr.conf

```
node_id=1
node_name='node1'
conninfo='host=<node-host> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/15/main'

failover='automatic'
promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

service_start_command='pg_ctlcluster 15 main start'
service_stop_command='pg_ctlcluster 15 main stop'
service_restart_command='pg_ctlcluster 15 main restart'
service_reload_command  = 'pg_ctlcluster 15 main reload'

repmgrd_service_start_command = 'repmgrd --daemonize=true'
repmgrd_service_stop_command = 'kill `cat /tmp/repmgrd.pid`'
repmgrd_pid_file='/tmp/repmgrd.pid'
```

…

---

### 总结

主从流复制 + repmgr 的方式带来了两点优化：0、借助 repmgr 命令能很直观的看到集群的状态；1、从节点配置更加方便，直接 clone 主节点的数据即可；2、借助 repmgrd 能实现自动故障转移。

需要注意：1、出故障的节点在恢复之后仍然需要人为的 rejoin 和 restart；2、检查好 repmgrd 的状态，否则无法实现自动故障转移。

此外自动故障转移可能会出现“脑裂”的情况，此时就需要人为干预，越早调整越好。

---

<br>

## 主从流复制 + repmgr + keepalive

> [可参考](https://vlambda.com/wz_7iAykuog2O0.html)

…

---

<br>

## Patroni

Patroni 是一个不同于 repmgr 的 PostgreSQL 高可用方案。关于 Patroni 和 repmgr 的对比，优缺点网络上已经存在很多资源了，此处不列举。

### 集群部署

> 测试机为 Ubuntu
>
> * PostgreSQL/Patroni: 192.168.111.21/192.168.111.22/192.168.111.23
> * etcd: 192.168.111.20

1、部署 etcd 用于存储集群元数据

> 目前仅部署一个服务用作测试，一般来说生产环境下需要部署三个 etcd 服务。

```shell
apt install etcd -y
```

2、修改配置 `/etc/default/etcd`

> 本机 IP 为 192.168.111.20

```shell
ETCD_NAME="etcd0"
ETCD_DATA_DIR="/var/lib/etcd/default"
ETCD_LISTEN_PEER_URLS="http://192.168.111.20:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.111.20:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.111.20:2380"
ETCD_INITIAL_CLUSTER="etcd0=http://192.168.111.20:2380"

ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

ETCD_ADVERTISE_CLIENT_URLS="http://192.168.111.20:2379"
```

启动 etcd

```shell
systemctl start etcd
# 允许开机自启
systemctl enable etcd --now # 允许开机自启并立即启动
```

3、安装 Patroni

> Ubuntu 自带 python3，如果没有需要先安装
>
> ```shell
> apt install python3
> ```

```shell
apt-get install -y python3-pip python3-psycopg2 # install python3 psycopg2 module on Debian/Ubuntu

# 最好开启代理，否则 pip 可能比较慢
pip install --upgrade pip
pip install --upgrade setuptools
pip install patroni[etcd]

# 注意：需要取消 postgresql 开机自启
systemctl disable postgresql
```



4、创建 `/etc/patroni.yml`

> 有几个需要修改的地方：
>
> * restapi.connect_address
> * etcd.host
> * postgresql.connect_address
> * postgresql.data_dir
> * postgresql.bin_dir

```yaml
scope: pgsql
namespace: /service/
name: pg1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.111.21:8008

etcd:
  host: 192.168.111.20:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10 # timeout for DCS and PostgreSQL operation retries (in seconds). 
    maximum_lag_on_failover: 1048576
    primary_start_timeout: 300
    synchronous_mode: false # turns on synchronous replication mode
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: "0.0.0.0"
        port: 5432
        wal_level: logical
        hot_standby: "on"
        wal_keep_segments: 100
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        # shared_preload_libraries: ['pg_stat_statements']
      pg_hba:
      - local all all 0.0.0.0/0 trust
      - host replication repl 0.0.0.0/0 trust
      - host all all 0.0.0.0/0 trust

  initdb:
  - encoding: UTF-8
  - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.111.21:5432
  data_dir: /var/lib/postgresql/15/main # main 目录需要为空，且用户和用户组为 postgres
  #config_dir: /etc/postgresql/15/main/ # defaults to the data directory
  bin_dir: /usr/lib/postgresql/15/bin

  authentication:
    superuser:
      username: postgres
      password: "postgres"
    replication:
      username: repl
      password: "123456"

  basebackup:
    max-rate: 100M
    checkpoint: fast

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

> 其他几个节点的 patroni 配置也是差不多的，需要修改几个地方：
>
> * name
> * restapi.connect_address
> * postgresql.connect_address

5、创建 `/etc/systemd/system/patroni.service`

```shell
[Unit]
Description=patroni.service
After=syslog.target network.target
 
[Service]
Type=simple
User=postgres
Group=postgres
#StandardOutput=syslog
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.target
```

6、设置 patroni 自启动

```shell
# root
systemctl enable patroni
```

配置 postgres 免密 sudo 权限

> 免密的前提是执行命令前要加上 `sudo` 前缀

```shell
vim /etc/sudoers
# or
sudo visudo
```

添加

```
postgres ALL=(ALL) NOPASSWD: ALL
```

7、启动 patroni

7.1、启动前的准备

```shell
root:~$ systemctl status postgresql # 确保 pg 处于停止状态下
# 备份原来的数据目录
root:~$ cd /var/lib/postgresql/15
root:~$ mv main backup/
root:~$ sudo su -- postgres
# postgres
postgres:~$ mkdir main
# 注意 postgres 用户对 main 文件夹的权限应该是 0700 或者 0750
postgres:~$ chmod 0750 main
```

7.2、启动 patroni

```shell
# postgres 用户
# 注意：需要加上 sudo
postgres@ubt1:~$ sudo systemctl start patroni
postgres@ubt1:~$ sudo systemctl status patroni
● patroni.service
     Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-10-18 16:14:53 CST; 3s ago
   Main PID: 6939 (patroni)
      Tasks: 12 (limit: 2231)
     Memory: 89.3M
        CPU: 911ms
     CGroup: /system.slice/patroni.service
             ├─6939 /usr/bin/python3 /usr/local/bin/patroni /etc/patroni.yml
             ├─6974 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main --config-file=/var/lib/postgresql/15/main/postgresql.conf --listen_addresses=0.0.0.0 --port=5432 --c>
             ├─6976 "postgres: pgsql: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─6977 "postgres: pgsql: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─6982 "postgres: pgsql: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─6983 "postgres: pgsql: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─6984 "postgres: pgsql: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             └─6987 "postgres: pgsql: postgres postgres 127.0.0.1(51908) idle" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">

Oct 18 16:14:54 ubt1 patroni[6978]: 2023-10-18 16:14:54.872 CST [6978] LOG:  database system was shut down at 2023-10-18 16:14:54 CST
Oct 18 16:14:54 ubt1 patroni[6975]: localhost:5432 - rejecting connections
Oct 18 16:14:54 ubt1 patroni[6981]: 2023-10-18 16:14:54.875 CST [6981] FATAL:  the database system is starting up
Oct 18 16:14:54 ubt1 patroni[6980]: localhost:5432 - rejecting connections
Oct 18 16:14:54 ubt1 patroni[6974]: 2023-10-18 16:14:54.878 CST [6974] LOG:  database system is ready to accept connections
Oct 18 16:14:55 ubt1 patroni[6985]: localhost:5432 - accepting connections
Oct 18 16:14:55 ubt1 patroni[6939]: 2023-10-18 16:14:55,887 INFO: establishing a new patroni connection to the postgres cluster
Oct 18 16:14:55 ubt1 patroni[6939]: 2023-10-18 16:14:55,891 INFO: running post_bootstrap
Oct 18 16:14:55 ubt1 patroni[6939]: 2023-10-18 16:14:55,905 WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watch>
Oct 18 16:14:55 ubt1 patroni[6939]: 2023-10-18 16:14:55,910 INFO: initialized a new cluster
```

8、查看集群状态

```shell
postgres@ubt1:~$ patronictl -c /etc/patroni.yml list
+ Cluster: pgsql (7291209041344875305) ------+----+-----------+
| Member | Host           | Role   | State   | TL | Lag in MB |
+--------+----------------+--------+---------+----+-----------+
| pg1    | 192.168.111.21 | Leader | running |  1 |           |
+--------+----------------+--------+---------+----+-----------+
```

9、将 patroni 配置设置到全局环境变量

```shell
# postgres
vim ~/.bash_profile
# 添加
export PATRONICTL_CONFIG_FILE=/etc/patroni.yml
source ~/.bash_profile
# 测试
patronictl list
```

10、重复前面的步骤，添加其他节点

> 其他几个节点的 `patroni.yml` 配置也是差不多的，需要修改几个地方：
>
> * name
> * restapi.connect_address
> * postgresql.connect_address

部署完成，查看集群状态

```shell
postgres@ubt1:~$ patronictl list
+ Cluster: pgsql (7291209041344875305) -------+----+-----------+
| Member | Host           | Role    | State   | TL | Lag in MB |
+--------+----------------+---------+---------+----+-----------+
| pg1    | 192.168.111.21 | Leader  | running |  1 |           |
| pg2    | 192.168.111.22 | Replica | running |  1 |        15 |
| pg3    | 192.168.111.23 | Replica | running |  1 |         0 |
+--------+----------------+---------+---------+----+-----------+
```

…

---

### 故障转移

1、关掉 Leader 前集群状态

```shell
postgres@ubt2:~/15$ patronictl list
+ Cluster: pgsql (7291209041344875305) -------+----+-----------+
| Member | Host           | Role    | State   | TL | Lag in MB |
+--------+----------------+---------+---------+----+-----------+
| pg1    | 192.168.111.21 | Leader  | running |  1 |           |
| pg2    | 192.168.111.22 | Replica | running |  1 |        15 |
| pg3    | 192.168.111.23 | Replica | running |  1 |         0 |
+--------+----------------+---------+---------+----+-----------+
```

2、手动停止 Leader 观察其他节点状态

```shell
postgres@ubt2:~/15$ patronictl list
+ Cluster: pgsql (7291209041344875305) -------+----+-----------+
| Member | Host           | Role    | State   | TL | Lag in MB |
+--------+----------------+---------+---------+----+-----------+
| pg1    | 192.168.111.21 | Replica | stopped |    |   unknown |
| pg2    | 192.168.111.22 | Replica | running |  1 |        31 |
| pg3    | 192.168.111.23 | Leader  | running |  2 |           |
+--------+----------------+---------+---------+----+-----------+
```

Patroni 自带故障转移，可以看到 Leader 节点从 pg1 变成了 pg3。

3、重新启动 pg1


```shell
postgres@ubt2:~/15$ patronictl list
+ Cluster: pgsql (7291209041344875305) -------+----+-----------+
| Member | Host           | Role    | State   | TL | Lag in MB |
+--------+----------------+---------+---------+----+-----------+
| pg1    | 192.168.111.21 | Replica | running |  1 |        16 |
| pg2    | 192.168.111.22 | Replica | running |  1 |        31 |
| pg3    | 192.168.111.23 | Leader  | running |  2 |           |
+--------+----------------+---------+---------+----+-----------+
```

pg1 作为 Replica 重新加入节点。

…

---

### 防止脑裂

**两种办法**

* watchdog：利用 Linux 下的 watchdog 来监控 patroni 进程的运行，当 watchdog 接收不到 patroni 进程的心跳时触发 Linux 重启
* 同步复制：如果启用了 PostgreSQL 集群同步复制模式，主库在没有收到同步备库的响应时会阻塞写入。同步复制的办法即保证了不会出现双主，也不会发生*双写*，采用这种方法最为安全可靠。代价是同步复制相对于异步复制会降低一点性能。

…

---

#### watchdog 配置

1、编辑 `/etc/systemd/system/patroni.service`

```shell
[Unit]
Description=patroni.service
After=syslog.target network.target
 
[Service]
Type=simple
User=postgres
Group=postgres
#StandardOutput=syslog
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog # 添加 1
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog # 添加 2
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.target
```

> 也可以不编辑该文件，[参考官方文档](https://patroni.readthedocs.io/en/latest/watchdog.html)：
>
> ```shell
> # root
> modprobe softdog
> # Replace postgres with the user you will be running patroni under
> chown postgres /dev/watchdog
> ```

2、编辑 `patroni.yml`，添加内容

```yaml
watchdog:
  mode: automatic # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5
```

> `safety_margin`：watchdog 持有一个 leader lock 且带有过期时间。如果 leader 节点异常，watchdog 超过 5s 未收到 leader patroni 的心跳，watchdog 会在 leader lock 过期前 5s 重启 leader 节点。重启如果在 5s 内完成，leader 节点有机会再次获得 leader 身份，否则备库会通过选举成为新的 leader。

3、重启 patroni

```shell
# root
systemctl enable patroni
# postgres
sudo systemctl restart patroni
```

…

> 这种方法比较依赖于 watchdog 本身的可靠性。从生产实践上来说应对绝大部分场景都是足够的，但重启机器的做法可能有点太过暴力了，很多时候它可能并不是我们所期望的行为。

…

---

#### 同步复制配置

1、编辑 `/etc/patroni.yml`

```yaml
synchronous_mode: true
```

对于正在运行中的 patroni 进程，可以通过命令修改

```shell
patronictl -c /etc/patroni.yml edit-config -s 'synchronous_mode=true'
```

> 在同步复制模式下，只有同步备库具有被提升为主库的资格。如果同步备库临时不可用，Patroni 会把主库的复制模式降级成了异步复制，确保服务不中断。效果类似于 MySQL 的半同步复制，但是相比 MySQL 使用固定的超时时间控制复制降级，这种方式更加智能，同时还能防止脑裂。
>
> 如果主库被降级为异步复制，由于没有同步备库作为候选主库 failover 不会被触发，也就不会出现“双主”。如果主库没有被降级为异步复制，那么即使同步备库被提升为主库出现“双主”，由于旧主处于同步复制模式，收不到同步备库的响应数据无法被写入，也不会出现“双写”。

Patroni 通过动态调整 PostgreSQL 参数 `synchronous_standby_names` 控制同步异步复制的切换。并且 Patroni 会把同步的状态记录到 etcd 中，确保同步状态在 Patroni 集群中的一致性。

可以在 etcd 中查看到同步模式下集群的元数据：

```shell
etcdctl get /service/pgsql/sync

{"leader":"pg1","sync_standby":"pg2"} # 正常的同步模式

{"leader":"pg1","sync_standby":null} # 备库故障导致主库临时降级为异步复制
```

> 注意：key 不一定是 `/service/pgsql/sync` ，psql 就是 `/etc/patroni.yml` 中的 scope 值。也可以通过 `patronictl -c /etc/patroni.yml list` 查看
>
> ```shell
> + Cluster: pgsql (7291209041344875305) -------+----+-----------+
> | Member | Host           | Role    | State   | TL | Lag in MB |
> +--------+----------------+---------+---------+----+-----------+
> | pg1    | 192.168.111.21 | Replica | running |  1 |        16 |
> | pg2    | 192.168.111.22 | Replica | running |  1 |        31 |
> | pg3    | 192.168.111.23 | Leader  | running |  2 |           |
> +--------+----------------+---------+---------+----+-----------+
> ```
>
> 即为集群的名字。

> 如果集群中包含 3 个以上的节点，可以考虑采取更严格的同步策略，禁止 Patroni 把同步模式降级为异步。这样可以确保任何写入的数据至少存在于 2 个以上的节点。对数据安全要求极高的业务可以采用这种方式。
>
> ```yaml
> synchronous_mode_strict: true
> ```
>
> 如果集群包含异地的灾备节点，可以根据需要配置该节点为不参与选主，不参与负载均衡，也不作为同步备库。
>
> ```yaml
> tags:
>     nofailover: true
>     noloadbalance: true
>     clonefrom: false
>     nosync: true
> ```

…

---

### etcd 高可用

> 由于 PostgreSQL 集群的元数据是保存在 etcd 中的，Patroni 需要通过访问 etcd 来确认自己的身份。当无法访问 etcd 的时候，如果本机的是主库，Patroni 会将本机降级为备库。如果集群中所有 Patroni 节点都无法访问 etcd，集群中将全部都是备库，业务无法写入数据。这需要保证 etcd 集群的高可用。

为了预防 etcd 集群故障带来的严重影响，可以考虑为 Patroni 连接 etcd 异常时设置一个比较大的 `retry_timeout` 参数，比如 10000 天

```yaml
retry_timeout: 864000000
```

…

---

### patronictl 命令

**Patroni 常用命令**

*  `patronictl -c /etc/patroni.yml list`

* `patronictl -c /etc/patroni.yml show-config`

* …

**修改 PostgreSQL 参数**

```shell
patronictl -c /etc/patroni.yml edit-config -p 'max_connections=300'
```

> 修改最大连接数后需要重启才能生效

…

---

### 客户端连接

PostgreSQL 集群的高可用是动态的，主节点会根据故障转移变化，因此客户端在连接时也需要能动态的访问到新的主节点上。常用的方法有：

* 多主机 URL
* VIP（*Virtual IP*）
* haproxy

…

---

#### 多主机 URL

JDBC 原生支持多主机 URL，并且功能较为全面，支持自动 failover，读写分离和负载均衡。可以通过连接参数配置不同的连接策略。

1、连接主节点（可写节点），当出现“双主”甚至“多主”的时候连接第一个发现的可用主节点

```
jdbc:postgresql://192.168.111.21:5432,192.168.111.22:5432,192.168.111.23:5432/postgres?targetServerType=primary
```

2、优先连接备节点，无可用备节点时连接主节点，有多个可用备节点时随机连接其中一个

```
jdbc:postgresql://192.168.111.21:5432,192.168.111.22:5432,192.168.111.23:5432/postgres?targetServerType=preferSecondary&loadBalanceHosts=true
```

3、随机连接任意一个可用的节点

```
jdbc:postgresql://192.168.111.21:5432,192.168.111.22:5432,192.168.111.23:5432/postgres?targetServerType=any&loadBalanceHosts=true
```

…

---

#### 回调脚本实现 VIP 漂移

> 可以通过 Patroni 的回调脚本实现 VIP 的漂移

多主机URL的方式部署简单，但是不是每种语言的驱动都支持。而且如果数据库出现意外的“双主”，配置多主机 URL 的客户端在多个主上同时写入的概率比较高。如果客户端通过 VIP 的方式访问则在 VIP 上又多了一层防护（这种风险一般在数据库的 HA 架构有缺陷时发生。如果我们配置的是 Patroni 的同步模式，基本上没有这个担忧）。

Patroni 支持用户配置在特定事件发生时触发回调脚本。因此我们可以配置一个回调脚本，在主备切换后动态加载 VIP。

1、准备 Patroni 回调脚本 `/var/lib/postgresql/loadvip.sh`

> 需要提前安装 arping，`apt install arping`

```shell
#!/bin/bash

VIP=192.168.111.30 # 固定 VIP
GATEWAY=192.168.111.2 # VIP 网关
DEV=ens1 # 不一定叫 eth1，绑定当前机器已存在的网卡，多添加一个 IP

action=$1
role=$2
cluster=$3

log()
{
  echo "loadvip: $*" | logger
}

load_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  log "vip exists, skip load vip"
else
  sudo ip addr add ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to add vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "added vip ${VIP} at dev ${DEV}"

  sudo arping -U -I ${DEV} -s ${VIP} ${GATEWAY} -c 5 >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to call arping to gateway ${GATEWAY} rc=$rc"
    exit 1
  fi
  
  log "called arping to gateway ${GATEWAY}"
fi
}

unload_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  sudo ip addr del ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to delete vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "deleted vip ${VIP} at dev ${DEV}"
else
  log "vip not exists, skip delete vip"
fi
}

log "loadvip start args:'$*'"

case $action in
  on_start|on_restart|on_role_change)
    case $role in
      master)
        load_vip
        ;;
      replica)
        unload_vip
        ;;
      *)
        log "wrong role '$role'"
        exit 1
        ;;
    esac
    ;;
  *)
    log "wrong action '$action'"
    exit 1
    ;;
esac
```

> 所有节点都添加

2、修改 `/etc/patroni.yml`

```yaml
postgresql:
...
  callbacks:
    on_start: /bin/bash /var/lib/postgresql/loadvip.sh
    on_restart: /bin/bash /var/lib/postgresql/loadvip.sh
    on_role_change: /bin/bash /var/lib/postgresql/loadvip.sh
```

> 所有节点都修改

3、重载 Patroni 配置

```shell
patronictl -c /etc/patroni.yml reload pgsql
```

> pgsql 表示集群名

4、执行 switchover

```shell
postgres@ubt3:~/15$ patronictl -c /etc/patroni.yml switchover pgsql
Current cluster topology
+ Cluster: pgsql (7291209041344875305) --+-----------+----+-----------+
| Member | Host           | Role         | State     | TL | Lag in MB |
+--------+----------------+--------------+-----------+----+-----------+
| pg1    | 192.168.111.21 | Sync Standby | streaming |  8 |         0 |
| pg2    | 192.168.111.22 | Leader       | running   |  8 |           |
| pg3    | 192.168.111.23 | Replica      | streaming |  8 |         0 |
+--------+----------------+--------------+-----------+----+-----------+
Primary [pg2]:
Candidate ['pg1', 'pg3'] []: pg1 # Candidate 必须是 Standby
When should the switchover take place (e.g. 2023-10-19T15:16 )  [now]: now
Are you sure you want to switchover cluster pgsql, demoting current leader pg2? [y/N]: y
2023-10-19 14:16:33.43225 Successfully switched over to "pg1"
+ Cluster: pgsql (7291209041344875305) ---------+----+-----------+
| Member | Host           | Role    | State     | TL | Lag in MB |
+--------+----------------+---------+-----------+----+-----------+
| pg1    | 192.168.111.21 | Leader  | running   |  8 |           |
| pg2    | 192.168.111.22 | Replica | stopped   |    |   unknown |
| pg3    | 192.168.111.23 | Replica | streaming |  8 |         0 |
+--------+----------------+---------+-----------+----+-----------+
```

5、查看 VIP 漂移情况

```shell
# root
tail -f -n 100 /var/log/syslog # 当前系统为 Ubuntu，其他系统日志文件名可能不太一样
```

```shell
Oct 19 06:24:42 ubt1 postgres: loadvip: loadvip start args:'on_role_change master pgsql'
Oct 19 06:24:42 ubt1 patroni[10287]: 2023-10-19 14:24:42.311 CST [10287] LOG:  selected new timeline ID: 10
Oct 19 06:24:42 ubt1 postgres: loadvip: added vip 192.168.111.30 at dev eth1
```

通过 Leader 节点的日志我们可以看到，在执行 switchover 之后，VIP 漂移到了Leader 节点。

…

---

#### [keepalived 实现 VIP 漂移](https://github.com/ChenHuajun/blog_xqhx/blob/main/2020/2020-09-07-%E5%9F%BA%E4%BA%8EPatroni%E7%9A%84PostgreSQL%E9%AB%98%E5%8F%AF%E7%94%A8%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2.md#73-vip%E9%80%9A%E8%BF%87keepalived%E5%AE%9E%E7%8E%B0vip%E6%BC%82%E7%A7%BB)

Patroni 提供了一系列 [REST API](https://patroni.readthedocs.io/en/latest/rest_api.html)，其中有可用于检查节点角色健康状态的 API。可使用 REST API 搭配 keepalived 动态的在主备库上绑定 VIP。

1、安装 keepalived

```shell
apt install -y keepalived
```

2、编辑 keepalived 配置文件 `/etc/keepalived/keepalived.conf`

> 下面的例子表示备节点故障时则将只读 VIP 绑在主节点上。

```
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_leader {
    script "/usr/bin/curl -s http://127.0.0.1:8008/leader -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 10
}
vrrp_script check_replica {
    script "/usr/bin/curl -s http://127.0.0.1:8008/replica -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 5
}
vrrp_script check_can_read {
    script "/usr/bin/curl -s http://127.0.0.1:8008/read-only -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 10
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 21
    priority 100
    advert_int 1
    track_script {
        check_can_read
        check_replica
    }
    virtual_ipaddress {
       192.168.111.31
    }
}
```

> 有两个地方需要注意：
>
> * `interface` 绑定自己机器上的网卡接口
> * `virtual_router_id` 虚拟路由 ID，最好在同一个子网下唯一
> * `virtual_ipaddress` 绑定的 VIP

3、启动 keepalived

```shell
systemctl start keepalived

# 如果有必要，开机自启
systemctl enable keepalived
```

…

> 在网络抖动或其它临时故障时 keepalived 管理的 VIP 容易飘，更推荐使用 Patroni 回调脚本动态绑定读写 VIP。

…

---

#### [haproxy](https://github.com/ChenHuajun/blog_xqhx/blob/main/2020/2020-09-07-%E5%9F%BA%E4%BA%8EPatroni%E7%9A%84PostgreSQL%E9%AB%98%E5%8F%AF%E7%94%A8%E7%8E%AF%E5%A2%83%E9%83%A8%E7%BD%B2.md#74-haproxy)

haproxy 本身会占用一定的资源，且需要配合 keepalived 来使用。

…

---

### REST API

在 Patroni 配置中设置的 [REST API](https://patroni.readthedocs.io/en/latest/rest_api.html) 可以通过 `curl -s http://<ip>:<port>/patroni` 等接口来访问。

…

---

### 级联复制

通常集群中所有的备库都从主库复制数据，但是特定的场景下我们可能需要部署级联复制。基于 Patroni 搭建的集群支持 2 种形式的级联复制。

* 节点之间级联复制
* 集群之间级联复制

…

#### 节点间级联复制

指定某个备库优先从指定成员而不是 Leader 节点复制数据

```yaml
tags:
    replicatefrom: pg2
```

> `replicatefrom` 只对节点处于 Replica 角色时有效，并不影响该节点参与 Leader 选举。当 `replicatefrom` 指定的复制源节点故障时，Patroni 会自动切换到 Leader 节点进行复制。

…

#### 集群间级联复制

还可以创建一个只读的备集群，从另一个指定的 PostgreSQL 实例复制数据。这可以用于创建跨数据中心的灾备集群。

1、初始创建一个备集群，可以在 Patroni 配置文件 `/etc/patroni.yml` 中加入以下配置

```yaml
bootstrap:
  dcs:
    standby_cluster:
      host: 192.168.111.30 # 上游复制源的主机号，可以使用 VIP 避免主集群主备切换时影响备集群
      port: 5432 # 上游复制源的端口号
      primary_slot_name: slot1 # 可选的
      create_replica_methods:
      - basebackup
```

如果配置了复制槽，需要同时在主集群上配置持久 slot。

```yaml
slots:
  slot1:
    type: physical
```

对于已配置好的级联集群，可以使用 `patronictl edit-config` 命令动态修改 `standby_cluster`

```yaml
standby_cluster:
  host: 192.168.111.21
  port: 5432
  primary_slot_name: slot1
  create_replica_methods:
  - basebackup
```

添加 standby_cluster 将主机群变成备集群；删除 standby_cluster 将备集群变成主机群。

…

---

### Patroni 配置管理

默认情况下 Patroni 管理的 PostgreSQL 配置是和数据目录在一起的

```yaml
postgresql:
...
  data_dir: /var/lib/postgresql/15/main
  #config_dir: /etc/postgresql/15/main/ # defaults to the data directory
  #custom_conf: /etc/postgresql/15/main/postgresql.custom.conf # 自定义配置文件
```

数据目录下的 `postgresql.conf` 配置文件会被 Patroni 重写，因此如果需要手动修改 PostgreSQL 配置可以编辑 `/var/lib/postgresql/15/main/postgrsql.base.conf`，`postgresql.conf` 引用了该文件。

当本地或者动态的修改 PostgreSQL 配置时，生效的情况如下：

1. 首先检查 `postgrsql.base.conf` 或者 custom_conf 有无配置；

2. 如果配置了 custom_conf 且有值，Patroni 会忽略 `postgrsql.base.conf` 和 `postgrsql.conf`；

3. 如果没有配置 custom_conf，但是  `postgrsql.base.conf` 存在且有值，则会使用 `postgrsql.base.conf` 中定义的配置；
4. 如果 custom_conf 和 `postgrsql.base.conf` 都不存在则会使用 `postgrsql.conf`，并将 `postgrsql.conf` 重命名为 `postgrsql.base.conf`，原来的 base 会被作为备份。
5. 动态设置的值会被直接添加到 `postgrsql.conf`，而且  `postgrsql.conf` 会包含 `postgrsql.base.conf` 和 custom_conf 指定的配置来作为基础值。

…

---

### 总结

|          | Patroni                                                      | repmgr                                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 安装     | 较复杂，需要 pip 辅助安装，还需要安装额外的 etcd 工具管理集群数据 | 简单，直接 apt 从源安装                                      |
| 部署     | 简单，配置文件只需要小修改即可用于其他节点，启动 Patroni 自动加入集群 | 相对 Patroni 来说复杂点，配置小修改即可复用，但还是需要手动加入集群 |
| 故障转移 | 自动切换，无需干预，方便                                     | 需要配置 repmgrd，主节点恢复之后还需要人为 rejoin，相对 Patroni 复杂。 |
| 脑裂     | 两种办法来避免 watchdog 或者同步复制， 同步复制基本上可以达到目的 | 可以配置 witness 节点来预防双主产生，如果脑裂需要手动 pg_rewind，越早干预越好 |
| …        | …                                                            | …                                                            |

…

---

## 全量备份

如果部署了一主一从一备库还不放心，可以再定期对数据库进行全量冷备份。

> 在这里使用 K3s/K8s 中的定时任务来进行备份。

1、使用 pg_basebackup 命令

```shell
pg_basebackup -U repl -h <source-db-ip> -p 5432 -F p -X stream -P -R -v -D /pgdata/dump/2023-11-02
```

命令执行无误，下一步。

2、编写 CronJob

pg-secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-secret
  namespace: test
type: Opaque
data:
  pg-password: cG9zdGdyZXM= # 经过 base64 加密后的密码 postgres
```

pg-backup

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pg-backup
  namespace: test
spec:
  schedule: "13 2 */5 * *" # m h date-of-month month date-of-week
  timeZone: "Asia/Shanghai"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: pg-backup
            image: postgres:15.4
            imagePullPolicy: IfNotPresent
            env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pg-secret
                  key: pg-password
            command:
            - /bin/sh
            - -c
            - |
              current_date=$(date +"%Y%m%d");
              echo "$current_date backup started"; 
              mkdir -p /opt/pgdata;
              su postgres;
              pg_basebackup -U repl -h 192.168.2.200 -p 5432 -F p -X stream -P -R -v -D /opt/pgdata/$(date +"%Y%m%d");
              if [ $? -eq 0 ]; then
                echo "$current_date pg_basebackup backup succeed.";
                find /opt/pgdata -maxdepth 1 -mtime +10 -type d -exec rm -rf {} +;
              else
                echo "$current_date pg_basebackup backup failed."
              fi
            volumeMounts:
            - name: dump-dir
              mountPath: /opt/pgdata
            - name: localtime
              readOnly: true
              mountPath: /etc/localtime
          volumes:
          - name: dump-dir
            hostPath:
              path: /opt/pgdata
          - name: localtime
            hostPath:
              type: File
              path: /etc/localtime
```

> 其中
>
> ```shell
> find /opt/pgdata -maxdepth 1 -mtime +10 -type d -exec rm -rf {} +;
> ```
>
> 表示删除 10 天前的备份

…

99、备份完成后可以测试全量备份的数据是否正常

```shell
# 在备份的机器上操作

# 使用备份目录启动
pg_ctl start -D /pgdata/full-dump/2023-11-02
# 检查启动状态
pg_ctl status -D /pgdata/full-dump/2023-11-02

# 进入 postgresql 再使用 \l 或者 \dt 等命令查询数据，查看是否完整
```

…

> 启动报错 `invalid value for parameter "lc_monetary"`：
>
> 编辑 postgresql.conf，注释 lc_messages、lc_monetary、lc_numeric、lc_time 字段。

…

---

# TimescaleDB

> 以 Ubuntu 为例，在已经安装了 PostgreSQL 的基础下安装 TimescaleDB。

1、安装

```shell
echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main" | sudo tee /etc/apt/sources.list.d/timescaledb.list

wget --quiet -O - https://packagecloud.io/timescale/timescaledb/gpgkey | sudo apt-key add -

apt update

apt install timescaledb-2-postgresql-15
```

> If you want to install a specific version of TimescaleDB, you can specify the version like this
>
> ```shell
> apt install timescaledb-2-postgresql-12='2.6.0*' timescaledb-2-loader-postgresql-12='2.6.0*'
> ```

2、配置调整

2.1、手动调整 PostgreSQL 配置，以让 TimescaleDB 达到最佳运行状态

```
max_worker_processes = 21
shared_preload_libraries = 'pg_stat_statements,timescaledb'

shared_buffers = 486001kB
effective_cache_size = 1423MB
maintenance_work_mem = 243000kB
work_mem = 9720kB
timescaledb.max_background_workers = 16
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
wal_buffers = 14579kB
min_wal_size = 512MB
max_wal_size = 1GB
default_statistics_target = 500
random_page_cost = 1.1
checkpoint_completion_target = 0.9
autovacuum_max_workers = 10
autovacuum_naptime = 10
effective_io_concurrency = 256
```

> [参考官方配置](https://docs.timescale.com/self-hosted/latest/configuration/about-configuration/)

2.2、自动配置调整

```shell
timescaledb-tune -conf-path /your/config/path/postgresql.conf
```

…

> **注意**
>
> 如果使用了 Patroni 来管理 PostgreSQL 集群，配置文件默认在 data_dir 内
>
> * `/var/lib/postgresql/15/main/postgresql.conf`
>
> 但是该文件会被 Patroni 重写，可以编辑
>
> * `/var/lib/postgresql/15/main/postgresql.base.conf`



3、查看安装情况

```postgresql
postgres=# SELECT * FROM pg_available_extensions;
-- or
postgres=# SELECT * FROM pg_available_extensions WHERE name like '%timescaledb%';
```

4、启用插件

```postgresql
postgres=# CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;
postgres=# CREATE EXTENSION IF NOT EXISTS timescaledb_toolkit CASCADE;
-- 查看插件启用情况
postgres=# \x
Expanded display is on.
postgres=# \dx
List of installed extensions
-[ RECORD 1 ]--------------------------------------------------------------------------------------
Name        | plpgsql
Version     | 1.0
Schema      | pg_catalog
Description | PL/pgSQL procedural language
-[ RECORD 2 ]--------------------------------------------------------------------------------------
Name        | timescaledb
Version     | 2.12.1
Schema      | public
Description | Enables scalable inserts and complex queries for time-series data (Community Edition)
-[ RECORD 3 ]--------------------------------------------------------------------------------------
Name        | timescaledb_toolkit
Version     | 1.17.0
Schema      | public
Description | Library of analytical hyperfunctions, time-series pipelining, and other SQL utilities
```

…

---

<br>

# 参考

* http://postgres.cn/docs
* https://www.sjkjc.com/postgresql
* https://www.cnblogs.com/flying-tiger/p/6704931.html

**数组类型对应**

* https://access.crunchydata.com/documentation/pgjdbc/42.3.2/arrays.html

**repmgr**

* https://www.repmgr.org/docs/current/repmgr-administration-manual.html
* https://blog.frognew.com/2021/11/repmgr-postgresql-ha.html
* https://blog.51cto.com/u_10930585/5805207
* https://www.modb.pro/db/15359

**etcd**

* https://docs.openstack.org/install-guide/environment-etcd-ubuntu.html

**Patroni**

* https://github.com/ChenHuajun/blog_xqhx/tree/main/2020
* https://patroni.readthedocs.io/en/latest/README.html#running-and-configuring
* patroni 配置：https://github.com/zalando/patroni/tree/master

**TimescaleDB**

* https://docs.timescale.com/self-hosted/latest/install/installation-linux/
* https://severalnines.com/blog/how-enable-timescaledb-existing-postgresql-database/
