# MySQL锁（基于InnoDB）

### 原则

只有访问过的对象才会加锁，锁在索引上。

select for update除了锁索引，还会额外锁主键。



### 分类

InnoDB判断锁：X锁（排他锁，写锁），S锁（共享锁，读锁）

兼容模式：

|      | X      | S      |
| :--- | :----- | :----- |
| X    | 不兼容 | 不兼容 |
| S    | 不兼容 | 兼容   |

意向锁：对细粒度的对象上锁前，需要先对粗粒度的对象上锁。比如，对行r上锁X前，需要对r所属的页、表、数据库进行上锁IX。InnoDB对意向锁的实现比较简练，其意向锁即为表级锁。

兼容模式：

|      | IS     | IX     | S      | X      |
| :--- | :----- | :----- | :----- | :----- |
| IS   |        |        |        | 不兼容 |
| IX   |        |        | 不兼容 | 不兼容 |
| S    |        | 不兼容 |        | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

根据范围可细分为以下：

| 行锁 | GAP锁  | NEXT-KEY-LOCK | insert intention lock |
| :--- | :----- | :------------ | :-------------------- |
| RK   | GK     | NK            | IK                    |
| 锁行 | 区间锁 | RK+GK         | 意向锁                |

兼容模式：

| 请求\持有             | Row    | GAP        | NEXT-KEY-LOCK         | Insert Intention Lock |
| :-------------------- | :----- | :--------- | :-------------------- | :-------------------- |
| Row                   | 不兼容 |            | 不兼容                |                       |
| GAP                   |        |            |                       |                       |
| NEXT-KEY-LOCK         | 不兼容 |            | 不兼容                |                       |
| Insert Intention Lock |        | **不兼容** | **不兼容(解决幻读）** |                       |



gap锁的作用是为了阻止多个事务将记录插入到统一范围内，导致幻读。

当一个事务需要给自己需要的某个资源加锁的时候，如果遇到一个共享锁正锁定着自己需要的资源的时候，自己可以再加一个共享锁，不过不能加排他锁。但是，如果遇到自己需要锁定的资源已经被一个排他锁占有之后，则只能等待该锁定释放资源之后自己才能获取锁定资源并添加自己的锁定。而意向锁的作用就是当一个事务在需要获取资源锁定的时候，如果遇到自己需要的资源已经被排他锁占用的时候，该事务可以需要锁定行的表上面添加一个合适的意向锁。如果自己需要一个共享锁，那么就在表上面添加一个意向共享锁。而如果自己需要的是某行（或者某些行）上面添加一个排他锁的话，则先在表上面添加一个意向排他锁。意向共享锁可以同时并存多个，但是意向排他锁同时只能有一个存在。

### 锁释放

在一个事务中，只有commit/rollback后才会释放持有的锁。

（除自增涨锁，在SQL语句执行完释放。ps.自增涨列必须是索引，且是索引第一列）

### InnoDB行锁实现方式

InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁
在实际应用中，要特别注意InnoDB行锁的这一特性，不然的话，可能导致大量的锁冲突，从而影响并发性能。下面通过一些实际例子来加以说明。
（1）在不通过索引条件查询的时候，InnoDB确实使用的是表锁，而不是行锁。
（2）由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。
（3）当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。
（4）即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

select 锁应用

意向锁是InnoDB自动加的，不需用户干预。对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁（X)；对于普通SELECT语句，InnoDB不会加任何锁；事务可以通过以下语句显示给记录集加共享锁或排他锁。

> ```
> 共享锁（S）：SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE
> 排他锁（X)：SELECT * FROM table_name WHERE ... FOR UPDATE
> ```

比如业务需求，一个事务中select后紧跟着update，如果中间被其他事务update后，本事务仍然会执行update。加上select for update后，对结果集上锁，其他事务只有等待本事务提交释放锁后才能对数据进行操作。

事务隔离级别：REPEATABLE-READ

(1)select * from table1 where feedback_id = "a";

1 a fbf

(2)select * from table1 where feedback_id = "a";

1 a fbf

(1)update table1 set text = "111" where feedback_id = "a";

(2)update table1 set text = "222" where feedback_id = "a";

2阻塞

(1)commit

1 a 111

2执行

(2)commit

1 a 222

结果是，1的执行结果被2覆盖掉了。在加了select时加for update，在select语句时就持有排他锁，2事务select阻塞，在1事务执行之后，2才能读到1事务已经提交后的数据（而不是2事务开始时的数据）。

select for update通过索引查询记录，对于聚集索引，仅上行锁；对于非聚集索引，加的是Next-Key Lock。

在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁，更新时再申请排他锁，因为当用户申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁。



在READ-COMMITTED下，select采用Record Lock方式上锁，此种方式会导致幻读。

在REPEARTABLE-READ（默认）采用Next-Key Lock方式上锁。

例：

【1 2 6】

select n>5 in share mode

READ-COMMITTED锁【6】不阻塞，结果【6】，此时其他事务insert n = 7提交，select n = 7出现结果【6，7】

REPEARTABLE-READ锁【n>5】，select n = 7阻塞

### Insert锁

自增id，普通索引

以缺省自增ID方式插入，上insert intention lock

以指定id插入，或指定唯一索引值，上Row Lock。

例：

| id(primary key) | feedback_id(index) | text | dk(unique) |
| :-------------- | :----------------- | :--- | :--------- |
|                 |                    |      |            |

(1)insert into table1(feedback_id,text,dk) values ("af","f",5);

Query OK, 1 row affected (0.00 sec)

(2)insert into table1(feedback_id,text,dk) values ("af","f",5);

2阻塞

(3)insert into table1(feedback_id,text,dk) values ("af","f",4);

Query OK, 1 row affected (0.00 sec)

(4)insert into table1(feedback_id,text) values ("af","f");

Query OK, 1 row affected (0.00 sec)

### update锁

insert语句，如果该区间已经有了GAP锁，那么会申请意向锁，当其他线程持有GAP锁而不是自己持有时，当前线程会被阻塞。

update语句，如果where条件是索引，且目标行存在，那么会上行锁；如果目标行不存在，且where索引非主键，那么会上GAP锁。（select使用in share mode或for update，上锁范围同update）比如：

update table1 set text = "fea" where biz_id = 2;

（1）

| id（主键） | biz_id（索引） | text |
| :--------- | :------------- | :--- |
| 1          | 1              | af   |
| 2          | 4              | dd   |
| 3          | 6              | ad   |

当前biz_id=2不存在，锁1、2行[1,4]。此时如果insert biz_id 3会被阻塞。

（2）

| id（主键） | biz_id（索引） | text |
| :--------- | :------------- | :--- |
| 2          | 4              | dd   |

锁biz_id在（-oo,4]范围内。

同样如果表中无数据，相当于锁表。

（3）

如果where主键，且目标数据不存在，不会锁任何。

### 多版本并发控制（MVCC）

| 目标行版本       | v1   | v2               | v3   | v4   | v5             | v6   |
| :--------------- | :--- | :--------------- | :--- | :--- | :------------- | :--- |
| 本事务事件       |      | 事务开始         |      |      | select         |      |
| 对应事务隔离级别 |      | REPEARTABLE-READ |      |      | READ-COMMITTED |      |