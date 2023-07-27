# PostgreSQL



## 安装

### 使用 Docker

```bash
docker pull postgres
docker run --name some-postgres -p 5432:5432 -e POSTGRES_PASSWORD=pgauth -d postgres
```



## 操作

### 系统级命令

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



### 数据库操作

```bash
create database <dbname>
drop database <dbname> # can not drop your current db

psql <dbname> # same as \c, connect databaes
```



### 数据表操作

创建/删除表的语句和 MySQL 一致：

* create table
* drop table

从文件装载数据

```bash
# 使用 COPY 从文本文件中装载大量数据
COPY weather FROM '/home/user/weather.txt';
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
  rank() OVER (PARTITION BY depname ORDER BY salary DESC) FROM empsalary;
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
select ... from cities
```

会同时查询 cities 和 capitals 两个表。如果只想查询 cities 表，不查询它的子表，可以使用 ONLY 关键字。例如：

```sql
select ... from ONLY cities;
```

除了 Select，Update 和 Delete 关键字也都支持 ONLY。

## 参考

* http://postgres.cn/docs
