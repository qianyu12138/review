# Limit的深度分页



```sql
select * from tb order by idx limit 10000,10;
```

在通常的情况都是需要循环分页，每次都需要扫O(n)条。

改为

```sql
select * from tb where id > #{lastPageId} limit 10;
```

直接定位到id=10000的数据，往后扫10条。

##### 关于回表

这样深度分页，结果10条回表肯定是需要的，但前10000条也需要。

存储引擎是一条一条数据的去找，直到找到第Y条，但是这些数据都会被server层的执行器抛弃，然后把Y+1到第N条的数据作为结果集返回。

可以通过SQL执行顺序解释：

**FROM**
<表名> # 选取表，将多个表数据通过笛卡尔积变成一个表。
**ON**
<筛选条件> # 对笛卡尔积的虚表进行筛选
**JOIN** 
\# 指定join，用于添加数据到on之后的虚表中，例如left join会将左表的剩余数据添加到虚表中
**WHERE**
\# 对上述虚表进行筛选
**GROUP BY**
<分组条件> # 分组
\# 用于having子句进行判断，在书写上这类聚合函数是写在having判断里面的
**HAVING**
<分组筛选> # 对分组后的结果进行聚合筛选
**SELECT**
<返回数据列表> # 返回的单列必须在group by子句中，聚合函数除外
**DISTINCT**
<数据除重>
**ORDER BY**
<排序条件> # 排序
**LIMIT**
<行数限制>

**limit是在select后执行的。**

一条语句的查询，是由逻辑算子组成。

逻辑算子介绍 在写具体的优化规则之前，先简单介绍查询计划里面的一些逻辑算子。

- DataSource 这个就是数据源，也就是表，select * from t 里面的 t。
- Selection 选择，例如 select xxx from t where xx = 5 里面的 where 过滤条件。
- Projection 投影， select c from t 里面的取 c 列是投影操作。
- Join 连接， select xx from t1, t2 where t1.c = t2.c 就是把 t1 t2 两个表做 Join。

选择，投影，连接（简称 SPJ） 是最基本的算子。其中 Join 有内连接，左外右外连接等多种连接方式。

select b from t1, t2 where t1.c = t2.c and t1.a > 5 变成逻辑查询计划之后，t1 t2 对应的 DataSource，负责将数据捞上来。上面接个 Join 算子，将两个表的结果按 t1.c = t2.c连接，再按 t1.a > 5 做一个 Selection 过滤，最后将 b 列投影。所以说不是mysql不想把limit, offset传递给引擎层，而是因为划分了逻辑算子，所以导致无法直到具体算子包含了多少符合条件的数据。