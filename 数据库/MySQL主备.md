### 主备原理

![](./img/主备原理.png)

![](./img/a66c154c1bc51e071dd2cc8c1d6ca6a3.png)

备库 B 跟主库 A 之间维持了一个长连接。主库 A 内部有一个线程，专门用于服务备库 B 的这个长连接。一个事务日志同步的完整过程是这样的：

1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog，发给 B。
4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）。
5. sql_thread 读取中转日志，解析出日志里的命令，并执行。

##### 循环复制

![](./img/DoubleM.png)

双M结构互为主备

规则：

1. 规定绑定主备关系的server id必须不同。
2. 一个备库收到binlog并重放的过程中，生成与原binlog的server id相同的新的binlog。
3. 如果收到binlog的server id与自己的相同，直接丢弃。

### 主备延迟

1. 主库 A 执行完成一个事务，写入 binlog，我们把这个时刻记为 T1;
2. 之后传给备库 B，我们把备库 B 接收完这个 binlog 的时刻记为 T2;
3. 备库 B 执行完成这个事务，我们把这个时刻记为 T3。

即T3-T1

show slave status命令返回seconds_behind_master

每个事务的 binlog 里面都有一个时间字段，seconds_behind_master = 当前时间-当前事务的时间。

SELECT UNIX_TIMESTAMP()查主库时间，扣掉系统时间差。

网络正常时，T2-T1很短，主备延迟主要来源于备库执行事务，所以说，主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产 binlog 的速度要慢

##### 主备延迟的来源

1.备库性能比主库性能低

更新请求对主备库的IOPS要求是无差别的。通常备库设置为非双1模式，但备库除了执行主库的更新请求，还肩负着大量查询的要求。

2.备库压力大

一些后台分析语句，查询大量数据占用CPU资源，影响同步速度，造成主备延迟。

解决：1.业务角度跑任务一天一次 2.一主多从，分担读压力 3.binlog输出外部系统，如Hadoop，让外部系统提供统计类查询能力。

3.大事务

如果一个事务执行10分钟，那这个事务很有可能导致从库延迟10分钟。（大量delete，大表DDL）

4.备库并行复制能力

##### 可靠性优先策略

1. 判断备库 B 现在的 seconds_behind_master，如果小于某个值（比如 5 秒）继续下一步，否则持续重试这一步；
2. 把主库 A 改成只读状态，即把 readonly 设置为 true；
3. 判断备库 B 的 seconds_behind_master 的值，直到这个值变成 0 为止；
4. 把备库 B 改成可读写状态，也就是把 readonly 设置为 false；
5. 把业务请求切到备库 B。

即先进行全切readonly，待主从同步再切流量。

##### 可用性优先策略

不等主备数据同步，直接把连接切到备库 B，并且让备库 B 可以读写，那么系统几乎就没有不可用时间了。

这个切换流程的代价，就是可能出现数据不一致的情况。

如：

如果binlog_format=mixed，业务新数据插入使用自增主键，同时主库传入insert语句，有可能造成数据不一致情况，且不易被发觉。

如果binlog_format=row，业务新数据插入使用自增主键，同时主库传入insert语句，结果是主键冲突。