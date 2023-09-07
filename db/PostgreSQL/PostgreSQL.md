# PostgreSQL



## 安装

### 使用 Docker

```bash
docker pull postgres
docker run --name some-postgres -p 5432:5432 -e POSTGRES_PASSWORD=<your-pw> -d postgres
```

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



## 数据库操作

```bash
create database <dbname>
drop database <dbname> # can not drop your current db

psql <dbname> # same as \c, connect databaes
```



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



## 执行分析

> * 来自 [polardb 数据库内核月报](http://mysql.taobao.org/monthly/)：http://mysql.taobao.org/monthly/2018/11/06/
> * https://www.modb.pro/db/101529



## 参考

* http://postgres.cn/docs
* https://www.sjkjc.com/postgresql
* https://www.cnblogs.com/flying-tiger/p/6704931.html

**数组类型对应**

* https://access.crunchydata.com/documentation/pgjdbc/42.3.2/arrays.html
