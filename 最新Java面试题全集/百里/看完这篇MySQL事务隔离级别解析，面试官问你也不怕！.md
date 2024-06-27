# 看完这篇MySQL事务隔离级别解析，面试官问你也不怕！


## 1.事务
MySQL事务是指一组数据库操作，这些操作要么全部执行成功，要么全部不执行。
如果其中任何一个操作失败，整个事务都会被回滚，即所有操作都会被撤销，数据库回到事务开始之前的状态。这样可以保证数据的一致性和完整性，避免了数据丢失或者不一致的情况。

## 2.事务特性
事务4大特性(ACID)：原子性、一致性、隔离性、持久性 

- 原子性（Atomicity）：事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么全不执行，不会出现部分执行的情况。
- 一致性（Consistency）：执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的。
- 隔离性（Isolation）：并发访问数据库时，事务的执行不会受到其他事务的干扰，即每个事务都应该像独立运行一样。
- 持久性（Durability）：事务一旦提交，其结果就应该永久保存在数据库中，即使系统崩溃也不应该丢失。 

## 3.事务隔离级别
MySQL事务隔离级别是指在多个事务同时访问数据库时，数据库如何保证数据的一致性和隔离性。常见的隔离级别如下：

- 读未提交：最低的隔离级别，允许读取尚未提交的数据变更。
- 读已提交：允许读取并发事务已经提交的数据。
- 可重复读：同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改。
- 串行化：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰。
| 隔离级别 | 并发问题 | 适用场景 | 隔离级别从上往下，由低到高。
隔离级别越高，事务的并发性能就越低。 |
| --- | --- | --- | --- |
| 读未提交（read-uncommitted） | 可能会导致脏读、幻读或不可重复读  | 并发性要求不高 |  |
| 读已提交（read-committed） | 可能会导致幻读或不可重复读  | 并发性要求较高 |  |
| 可重复读（repeatable-read） | 可能会导致幻读  | 数据一致性要求较高 |  |
| 可串行化（serializable） | 不会产生干扰 | 数据一致性要求非常高 |  |

在实际应用中，需要根据具体情况选择合适的隔离级别，平衡数据的一致性和并发性能。例如，在高并发的Web应用程序中，可以选择可重复读隔离级别，以保证数据的一致性和并发性能。

- 脏读（Dirty Reads）：事务A读取到了事务B已经修改但尚未提交的数据
- 不可重读（Non-Repeatable Reads）：事务A内部的相同查询语句在不同时刻读出的结果不一致
- 幻读（Phantom Reads）：事务A读取到了事务B提交的新增数据

### 3.1.数据准备
```plsql
DROP TABLE test;
CREATE TABLE test (id INT PRIMARY KEY, name VARCHAR(20), balance INT);
INSERT INTO test VALUES (1, 'Alice', 1000);
```

### 3.2.读未提交（read-uncommitted）

- 事务A修改balance并且不提交事务，事务B读取balance值为900；
- 如果此时事务A回滚数据，事务B读取balance值为1000（脏读）；
```plsql
# 事务A
set tx_isolation = 'read-uncommitted';
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM test WHERE id = 1;

# @1
rollback
COMMIT;

# 事务B
set tx_isolation = 'read-uncommitted';
BEGIN;
SELECT balance FROM test WHERE id = 1;

# @1:rollback后
SELECT balance FROM test WHERE id = 1;
commit;
```

### 3.3.读已提交（read-committed）

- 事务A修改balance并且不提交事务，事务B读取balance为1000；当事务A提交后，事务B读取balance值为900；
- 再重新开启事务A修改balance并提交事务，事务B中在读取balance值为800(整个过程事务B都不提交)（不可重复读）；
```plsql
update test set balance = 1000 where id = 1;
# 事务A
set tx_isolation = 'read-committed';
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM test WHERE id = 1;
COMMIT;

# @2：再次修改balance并提交事务
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM test WHERE id = 1;
COMMIT;

# 事务B
set tx_isolation = 'read-committed';
BEGIN;
# 事务A提交前
SELECT balance FROM test WHERE id = 1;

# 事务A提交后
SELECT balance FROM test WHERE id = 1;

# @2：再次查询balance
SELECT balance FROM test WHERE id = 1;
commit;
```

### 3.4.可重复读（repeatable-read）

- 事务A修改balance并且不提交事务，事务B读取balance为1000；当事务A提交后，事务B读取balance值为1000；
- 开启事务A修改balance并提交事务，事务B中在读取balance值为1000（可重复读）(整个过程事务B都不提交)；
- 开启事务A插入为2的记录，事务B无法读取到2的记录，此时修改id为2balance+1000，可以修改成功，重新读取为2的记录balance为3000（幻读）(整个过程事务B都不提交)
```plsql
update test set balance = 1000 where id = 1;
# 事务A
set tx_isolation = 'repeatable-read';
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM test WHERE id = 1;
COMMIT;

# @1:再次修改balance
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
SELECT balance FROM test WHERE id = 1;
COMMIT;

# @2:插入id:2记录
BEGIN;
INSERT INTO test VALUES (2, 'Alice2', 2000);
COMMIT;

# 事务B
set tx_isolation = 'repeatable-read';
BEGIN;
# 事务A提交前
SELECT balance FROM test WHERE id = 1;

# 事务A提交后
SELECT balance FROM test WHERE id = 1;

# @1:再次查询balance
SELECT balance FROM test WHERE id = 1;

# @2:查询id:2的记录
SELECT balance FROM test WHERE id = 2;

# 修改id:2的balance，修改成功
update test set balance = balance + 1000 where id = 2;

# 查询id:2的记录
SELECT balance FROM test WHERE id = 2;
commit;
```

### 3.5.可串行化（serializable）

- 事务A修改balance并且不提交事务，事务B无法读取balance值（阻塞中），当事务A提交后，事务B才能读取balance值为1000（修改与读取串行化，不能同时执行）
```plsql
update test set balance = 1000 where id = 1;
# 事务A
set tx_isolation = 'serializable';
BEGIN;
UPDATE test SET balance = balance - 100 WHERE id = 1;
COMMIT;

# 事务B
set tx_isolation = 'serializable';
BEGIN;
SELECT balance FROM test WHERE id = 1;
commit;
```



> 原文: <https://www.yuque.com/tulingzhouyu/sfx8p0/ychbh22quhepw2qg>