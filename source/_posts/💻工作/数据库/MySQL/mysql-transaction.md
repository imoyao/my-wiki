---
title: MySQL 事务
toc: true
tags:
  - 面试
  - 事务
categories:
  - "\U0001F4BB 工作"
  - 数据库
  - MySQL
date: 2019-08-07 12:27:56
---


## 事务定义

*   事务（Transaction）：一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务(例如银行账户转账业务，该业务就是一个最小的工作单元)
*   一个完整的业务需要批量的 DML(insert、update、delete)语句共同联合完成；
*   事务只和 DML 语句有关，或者说 DML 语句才有事务。这个和业务逻辑有关，业务逻辑不同，DML 语句的个数不同
- 在 MySQL 中只有使用了 Innodb 数据库引擎的数据库或表才支持事务。

### 示例

 关于银行账户转账操作，账户转账是一个完整的业务，最小的单元，不可再分——也就是说银行账户转账是一个事务

以下是银行账户表 t_act(账号、余额)，进行转账操作：
```plain
    actno       balance
    1           500
    2           100
```
转账操作
```plain
    update t_act set balance=400 where actno=1;
    update t_act set balance=200 where actno=2;
```
以上两条 DML 语句必须同时成功或者同时失败。最小单元不可再分，当第一条 DML 语句执行成功后，并不能将底层数据库中的第一个账户的数据修改，只是将操作记录了一下；这个记录是在内存中完成的；当第二条 DML 语句执行成功后，和底层数据库文件中的数据完成同步。若第二条 DML 语句执行失败，则清空所有的历史操作记录，要完成以上的功能必须借助事务


## 四大特征(ACID)

*   原子性(A)：事务是最小单位，不可再分
*   一致性(C)：事务要求所有的 DML 语句操作的时候，必须保证同时成功或者同时失败
*   隔离性(I)：事务 A 和事务 B 之间具有隔离性
*   持久性(D)：是事务的保证，事务终结的标志(内存的数据持久到硬盘文件中)


## 控制语句

*   开启事务：Start Transaction/ BEGIN 
*   事务结束：End Transaction
*   提交事务：Commit Transaction
*   回滚事务：Rollback Transaction

---

*   BEGIN 或 START TRANSACTION 显式地开启一个事务；
*   COMMIT 也可以使用 COMMIT WORK，不过二者是等价的。COMMIT 会提交事务，并使已对数据库进行的所有修改成为永久性的；
*   ROLLBACK 也可以使用 ROLLBACK WORK，不过二者是等价的。回滚会结束用户的事务，并撤销正在进行的所有未提交的修改；  
*   SAVEPOINT identifier，SAVEPOINT 允许在事务中创建一个保存点，一个事务中可以有多个 SAVEPOINT；
*   RELEASE SAVEPOINT identifier 删除一个事务的保存点，当没有指定的保存点时，执行该语句会抛出一个异常；
*   ROLLBACK TO identifier 把事务回滚到标记点；
*   SET TRANSACTION 用来设置事务的隔离级别。InnoDB 存储引擎提供事务的隔离级别有 READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ 和 SERIALIZABLE。

## 代码示例
```sql
mysql> use RUNOOB;
Database changed
mysql> CREATE TABLE runoob_transaction_test( id int(5)) engine=innodb;  # 创建数据表
Query OK, 0 rows affected (0.04 sec)
 
mysql> select * from runoob_transaction_test;
Empty set (0.01 sec)
 
mysql> begin;  # 开始事务
Query OK, 0 rows affected (0.00 sec)
 
mysql> insert into runoob_transaction_test value(5);
Query OK, 1 rows affected (0.01 sec)
 
mysql> insert into runoob_transaction_test value(6);
Query OK, 1 rows affected (0.00 sec)
 
mysql> commit; # 提交事务
Query OK, 0 rows affected (0.01 sec)
 
mysql>  select * from runoob_transaction_test;
+------+
| id   |
+------+
| 5    |
| 6    |
+------+
2 rows in set (0.01 sec)
 
mysql> begin;    # 开始事务
Query OK, 0 rows affected (0.00 sec)
 
mysql>  insert into runoob_transaction_test values(7);
Query OK, 1 rows affected (0.00 sec)
 
mysql> rollback;   # 回滚
Query OK, 0 rows affected (0.00 sec)
 
mysql>   select * from runoob_transaction_test;   # 因为回滚所以数据没有插入
+------+
| id   |
+------+
| 5    |
| 6    |
+------+
2 rows in set (0.01 sec)
```

## 和事务相关的两条重要的 SQL 语句(TCL)

*   commit:提交
*   rollback：回滚


## 事务开启的标志？事务结束的标志

### 开启标志

- 任何一条 DML 语句(insert、update、delete)执行，标志事务的开启
    

### 结束标志(提交或者回滚)

-  提交：成功的结束，将所有的 DML 语句操作历史记录和底层硬盘数据来一次同步
-  回滚：失败的结束，将所有的 DML 语句操作历史记录全部清空
    


## 事务与数据库底层数据


> 在事务进行过程中，未结束之前，DML 语句是不会更改底层数据，只是将历史操作记录一下，在内存中完成记录。只有在事务结束的时候，而且是成功的结束的时候，才会修改底层硬盘文件中的数据


## 在 MySQL 中，事务提交与回滚

> 在 MySQL 中，默认情况下，事务是自动提交的，也就是说，只要执行一条 DML 语句就开启了事务，并且提交了事务

### 以上的自动提交机制是可以关闭的

### 对`t_user`进行提交和回滚操作

### 提交操作(事务成功)
- start transaction
- DML 语句
- commit

```sql   
mysql> start transaction; //手动开启事务
mysql> insert into t_user(name) values('pp');
mysql> commit; // commit之后即可改变底层数据库数据
mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | jay  |
|  2 | man  |
|  3 | pp   |
+----+------+
3 rows in set (0.00 sec)
```

### 回滚操作(事务失败)

- start transaction
- DML 语句
- rollback

```sql 
mysql> start transaction;
mysql> insert into t_user(name) values('yy');
mysql> rollback;
mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | jay  |
|  2 | man  |
|  3 | pp   |
+----+------+
3 rows in set (0.00 sec)
```
    

## 事务四大特性之一——隔离性(isolation)

1.  事务 A 和事务 B 之间具有一定的隔离性
2.  隔离性有隔离级别(4 个)  
*   读未提交：read uncommitted
*   读已提交：read committed
*   可重复读：repeatable read
*   串行化：serializable

1. read uncommitted

- 事务 A 和事务 B，事务 A 未提交的数据，事务 B 可以读取到
- 这里读取到的数据叫做“脏数据”
- 这种隔离级别最低，这种级别一般是在理论上存在，数据库隔离级别一般都高于该级别
    

2. read committed

- 事务 A 和事务 B，事务 A 提交的数据，事务 B 才能读取到
- 这种隔离级别高于读未提交
- 换句话说，对方事务提交之后的数据，我当前事务才能读取到
- 这种级别可以避免“脏数据”
- 这种隔离级别会导致“不可重复读取”
- Oracle 默认隔离级别

3. repeatable read

- 事务 A 和事务 B，事务 A 提交之后的数据，事务 B 读取不到
- 事务 B 是可重复读取数据
- 这种隔离级别高于读已提交
- 换句话说，对方提交之后的数据，我还是读取不到
- 这种隔离级别可以避免“不可重复读取”，达到可重复读取
- 比如 1 点和 2 点读到数据是同一个
- MySQL 默认级别
- 虽然可以达到可重复读取，但是会导致“幻像读”
    
4. serializable

- 事务 A 和事务 B，事务 A 在操作数据库时，事务 B 只能排队等待
- 这种隔离级别很少使用，吞吐量太低，用户体验差
- 这种级别可以避免“幻像读”，每一次读取的都是数据库中真实存在数据，事务 A 与事务 B 串行，而不并发
    

## 隔离级别与一致性关系

![隔离级别](https://img-blog.csdn.net/2018032313015577)

## 设置事务隔离级别

### 方式一

* 可以在 my.ini 文件中使用 transaction-isolation 选项来设置服务器的缺省事务隔离级别。
    
* 该选项值可以是：
    
– READ-UNCOMMITTED
– READ-COMMITTED
– REPEATABLE-READ
– SERIALIZABLE

例如

```plain
[mysqld]
transaction-isolation = READ-COMMITTED
```

### 方式二

*   通过命令动态设置隔离级别  
 隔离级别也可以在运行的服务器中动态设置，应使用 SET TRANSACTION ISOLATION LEVEL 语句。  
其语法模式为：

```sql
SET [GLOBAL | SESSION] TRANSACTION ISOLATION LEVEL <isolation-level>
```
其中的<isolation-level>可以是：
– READ UNCOMMITTED
– READ COMMITTED
– REPEATABLE READ
– SERIALIZABLE

例如： `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`
        
## 隔离级别的作用范围

事务隔离级别的作用范围分为两种：
–   全局级：对所有的会话有效 
–   会话级：只对当前的会话有效 
例如，设置会话级隔离级别为 `READ COMMITTED` ：
```sql
mysql> SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
或：
```sql
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
设置全局级隔离级别为`READ COMMITTED`： 
```sql
mysql> SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
## 查看隔离级别

事务隔离级别的作用范围分为两种：
– 全局级：对所有的会话有效 
– 会话级：只对当前的会话有效 
例如，设置会话级隔离级别为`READ COMMITTED`：
```sql
mysql> SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
或：
```sql
mysql> SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
设置全局级隔离级别为 READ COMMITTED ： 
```sql
mysql> SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

## 参考来源
[MySQL——事务(Transaction)详解_浅然的专栏-CSDN 博客](https://blog.csdn.net/w_linux/article/details/79666086)
[MySQL 事务 | 菜鸟教程](https://www.runoob.com/mysql/mysql-transaction.html)