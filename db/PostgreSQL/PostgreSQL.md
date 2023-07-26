# PostgreSQL



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

从文件装载数据：

```bash
# 使用 COPY 从文本文件中装载大量数据
COPY weather FROM '/home/user/weather.txt';
```

