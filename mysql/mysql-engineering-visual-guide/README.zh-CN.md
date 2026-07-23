# 学到 emo 之 MySQL：从索引、事务到分库分表

![MySQL 图解教程封面](imgs/cover.png)

> 若不是生活所迫，谁会想把索引、锁、事务和分库分表一次学完呢？

这篇文章整合了我以前写过的 5 篇 MySQL 笔记。以前的文章比较零散，这次我们顺着一条 SQL 的执行过程，从“怎么查得快”一直聊到“单库扛不住了怎么办”。

不用怕，全文只解决几个问题：

1. 索引为什么能让查询变快？
2. 为什么建了索引，MySQL 有时还是不用？
3. 事务、Undo、Redo、MVCC 和锁到底是什么关系？
4. 表结构应该怎么设计，线上又怎么修改？
5. 数据多了，是不是马上就要分库分表？

本文以生产中更稳妥的 **MySQL 8.4 LTS** 为基准。截至 2026 年 7 月，MySQL 最新的创新版本是 **9.7**。

## 0. 先看地图：一条 SQL 要过几关？

假设我们执行：

```sql
SELECT name
FROM user
WHERE phone = '17665395447';
```

MySQL 收到 SQL 后，并不是低头就开始一行一行找。

它会先分析 SQL，再决定走哪个索引；如果 SQL 在事务里，还要处理数据版本和锁；表太大时，我们才会继续考虑归档、分区或分库分表。

![一条 SQL 从查询到分片的完整路径](imgs/01-infographic-mysql-mental-model.png)

**闲狗言：先把 SQL 和索引搞明白，再聊分库分表。不要用分布式方案掩盖一条烂 SQL。**

## 1. 索引为什么会让查询变快？

先想象一本 500 页的书。

没有目录时，为了找到“事务隔离”，我们只能从第一页翻到第五百页。有了目录后，先查目录，再跳到对应页码。这本目录就是索引。

数据库也一样：

- 没有合适的索引：一行一行扫描；
- 有合适的索引：先在索引中找到位置，再取数据。

索引当然不是免费的。它要占磁盘空间，插入、更新和删除数据时也要跟着维护。

### 1.1 为什么不用普通二叉树？

InnoDB 从磁盘读取数据时，通常不是只读一个数字，而是一次读取一个数据页。默认情况下，一个页通常是 16 KiB。

如果树的每个节点只能放一个键，树会很高。树越高，找一条数据需要访问磁盘的次数就越多。

B+ 树会在一个页里放很多键，因此树长得又矮又胖。叶子节点还会按顺序连接起来，范围查询也很方便。

![为什么 InnoDB 使用 B+ 树](imgs/02-framework-bplus-tree.png)

**灵魂拷问：一棵三层 B+ 树到底能放多少行？**

答案是：没有固定数字。它取决于主键大小、每行数据大小、页利用率等因素。所以“固定能放两千万行”只能帮助理解，不能当成生产结论。

### 1.2 主键索引和普通索引有什么不同？

InnoDB 的主键索引也叫聚簇索引，它的叶子节点放的是完整数据。

普通索引也叫二级索引，它的叶子节点通常放：

```text
普通索引字段值 + 主键值
```

例如：

```sql
SELECT *
FROM user
WHERE phone = '17665395447';
```

如果 `phone` 有普通索引，MySQL 会：

1. 在 `phone` 索引树找到对应的主键；
2. 再拿主键去主键索引树找完整数据。

第二步就是我们常说的**回表**。

![普通索引回表与覆盖索引](imgs/03-flowchart-index-lookup.png)

如果查询改成：

```sql
SELECT id, phone
FROM user
WHERE phone = '17665395447';
```

而 `phone` 索引里已经有 `phone` 和主键 `id`，MySQL 不需要回表就能返回结果，这就是**覆盖索引**。

**闲狗言：不是看到 Using index 就开心，而是要看 MySQL 为这条 SQL 到底跑了多少路。**

### 1.3 联合索引为什么讲“最左匹配”？

假设我们建立了这个索引：

```sql
CREATE INDEX idx_order_status_time
ON orders(status, created_at, id);
```

它会先按 `status` 排序；`status` 相同时，再按 `created_at` 排序；前两个都相同时，最后按 `id` 排序。

所以：

```sql
WHERE status = 'PAID'
```

可以快速找到一段连续数据。

```sql
WHERE status = 'PAID'
  AND created_at >= '2026-01-01'
```

也可以先定位 `status`，再在其中查时间范围。

但是只有：

```sql
WHERE created_at >= '2026-01-01'
```

就像一本电话簿先按城市、再按姓名排序，你却跳过城市直接找姓名，查起来就不那么方便了。

![联合索引的列顺序](imgs/04-infographic-composite-index.png)

注意，下面两条 SQL 的条件书写顺序不同：

```sql
WHERE status = 'PAID' AND created_at >= '2026-01-01';

WHERE created_at >= '2026-01-01' AND status = 'PAID';
```

通常不会因为书写顺序不同而影响索引使用。优化器会重新安排条件。最左匹配说的是**索引列的排列方式**，不是 `WHERE` 后面谁写在前面。

### 1.4 为什么 LIKE 和函数容易让索引失效？

```sql
WHERE phone LIKE '176%';   -- 通常可以利用前缀
WHERE phone LIKE '%176%';  -- 前面不确定，很难直接定位
```

对索引字段做函数也有类似问题：

```sql
WHERE MONTH(created_at) = 7;
```

B+ 树里排好序的是完整的 `created_at`，不是 `MONTH(created_at)` 的结果。可以改成时间范围：

```sql
WHERE created_at >= '2026-07-01'
  AND created_at <  '2026-08-01';
```

MySQL 也支持函数索引和生成列索引，所以不要背成“函数永远不能走索引”。最终还是要看执行计划。

## 2. 怎么判断一条 SQL 到底慢在哪里？

别猜，先 `EXPLAIN`。

```sql
EXPLAIN FORMAT=TREE
SELECT id, status
FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC
LIMIT 50;
```

普通 `EXPLAIN` 告诉我们 MySQL **准备怎么执行**。`EXPLAIN ANALYZE` 则会真的执行 SQL，并给出实际耗时和实际行数。

![使用 EXPLAIN ANALYZE 优化 SQL](imgs/05-flowchart-explain-analyze.png)

重点关注：

- 预计扫描多少行，实际扫描多少行；
- 返回第一行花了多久；
- 哪一步循环次数特别多；
- 是否发生排序、临时表或大量回表。

```sql
EXPLAIN ANALYZE
SELECT id, status
FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC
LIMIT 50;
```

注意：`EXPLAIN ANALYZE` 会真正执行 SQL。不要拿一条危险的更新语句在生产环境随手试。

### 2.1 千万级数据怎么分页？

很多人会这样写：

```sql
SELECT *
FROM posts
ORDER BY id
LIMIT 1000000, 100;
```

MySQL 仍然要先找到并跳过前面的一百万行，然后才拿后面 100 行。页码越靠后，工作越多。

如果业务允许“继续加载下一页”，可以记住上一页最后一条记录：

```sql
SELECT id, created_at, title
FROM posts
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

这叫游标分页或 Keyset Pagination。它不要求主键连续，只要求排序条件稳定并有合适的索引。

## 3. 事务、Undo、Redo、MVCC 和锁是什么关系？

先看一个转账：

```text
A 账户扣 100 元
B 账户加 100 元
```

如果第一句成功，第二句失败，钱就凭空消失了。所以这两步要放在一个事务里：要么都成功，要么都失败。

![InnoDB 事务内部的四套机制](imgs/06-framework-transaction-engine.png)

可以先简单记住：

- **Undo Log**：后悔药，出错时把数据改回去；
- **Redo Log**：恢复记录，机器崩溃后把已提交修改重放出来；
- **MVCC**：为读操作准备历史版本；
- **锁**：避免大家同时修改同一份数据。

### 3.1 Undo Log 是怎么工作的？

假设：

```sql
UPDATE account SET balance = 900 WHERE id = 1;
```

修改前余额是 1000，Undo 会保存恢复旧值所需的信息。事务回滚时，就能把数据恢复。

Undo 还有第二份工作：普通查询需要看旧版本时，可以沿着 Undo 版本链往前找。

### 3.2 Redo Log 又是干什么的？

修改数据时，内存中的数据页已经变了，但数据页不一定马上写回磁盘。要是机器突然断电怎么办？

InnoDB 会先记录 Redo。崩溃重启后，可以用 Redo 恢复已经提交的修改。

`innodb_flush_log_at_trx_commit=1` 的提交持久性最强。设置为 `2` 可能减少磁盘刷新次数，但操作系统或整台机器故障时，最近一小段提交可能丢失。因此不能简单理解成“设为 2 也绝对不丢数据”。

### 3.3 MVCC 为什么能做到读写不互相堵住？

假设一行数据被改了三次：

```text
V1 ← V2 ← V3
```

当前数据是 V3，Undo 中还保留了构造 V2、V1 所需的信息。普通查询会根据 Read View 判断自己能看到哪个版本。

![MVCC 的 Read View 与 Undo 版本链](imgs/07-flowchart-mvcc-visibility.png)

- `READ COMMITTED`：每次普通查询通常都会拿一个新快照；
- `REPEATABLE READ`：事务内的普通查询通常复用第一次一致性读建立的快照；
- `UPDATE`、`DELETE`、`SELECT ... FOR UPDATE`：要读取当前最新数据，属于当前读。

所以“事务一开始就一定创建快照”并不准确。更准确地说，快照通常在需要一致性读时创建。

长事务会让旧版本迟迟不能清理，还会长时间占用锁。能 1 秒完成的事务，就别让它活 1 小时。

## 4. InnoDB 是行锁，为什么还是会锁很多数据？

先看一句最容易误解的话：

> InnoDB 使用行级锁，所以只会锁一行。

这句话不完整。

InnoDB 的锁加在索引记录和扫描范围上。SQL 为了找到目标数据扫描了多少范围，通常就可能影响多少范围。

![索引访问路径决定锁范围](imgs/08-comparison-lock-scope.png)

例如：

- 唯一索引等值查询：通常很快定位一条，锁的范围小；
- 范围查询：可能锁多条记录和间隙；
- 没有合适索引：可能扫描大量数据，锁冲突也会跟着变大。

普通 `SELECT` 在常用的 `READ COMMITTED` 和 `REPEATABLE READ` 下通常是快照读，不会像 `SELECT ... FOR UPDATE` 那样给记录加锁。

**闲狗言：优化索引不只是让查询更快，有时也能让锁得更少。**

## 5. 表结构到底应该怎么设计？

以前我们喜欢把规范写成“必须”和“禁止”。工作久了会发现，很多规范其实应该写成“默认建议，特殊情况请说明理由”。

![表结构设计与在线 DDL](imgs/09-infographic-schema-online-ddl.png)

### 5.1 比较靠谱的默认选择

- 存储引擎使用 InnoDB；
- 字符集使用 `utf8mb4`；
- 明确定义主键；
- 主键尽量短、稳定；
- 数据类型够用就好；
- 只给实际查询需要的字段建索引；
- 每个字段写清楚含义。

### 5.2 NULL 到底能不能用？

可以用，也可以建索引。

`NULL` 表示“未知”或“没有值”。如果业务上真的需要表达未知，就应该使用 `NULL`。不要为了所谓的性能，随便用 `0`、空字符串或者 `1970-01-01` 冒充真实数据。

是否使用 `NULL`，首先是业务语义问题，然后才是存储和查询问题。

### 5.3 子查询是不是一律不能用？

也不是。

MySQL 优化器可能把一些子查询转换成 semi-join、派生表等执行方式。有些子查询执行得很好，有些确实很差。

与其制定“不准写子查询”的死规则，不如：

1. 先把 SQL 写正确；
2. 看 `EXPLAIN ANALYZE`；
3. 对比子查询与 JOIN 的实际执行结果。

### 5.4 线上修改表结构一定会锁表吗？

不同的修改，代价完全不同：

- `INSTANT`：主要修改元数据，通常最快；
- `INPLACE`：可能需要在原表上做较多工作；
- `COPY`：复制并重建整张表，代价最大。

```sql
ALTER TABLE orders
ADD COLUMN source VARCHAR(32) NULL,
ALGORITHM=INSTANT;
```

即使支持在线 DDL，也要关注 metadata lock、磁盘空间、主从延迟和长事务。在线不等于没有风险。

## 6. 五百万行以后就必须分库分表吗？

这是一个经典的灵魂拷问。

答案是：**MySQL 没有“五百万行就必须分表”的规定。**

同样是一千万行：

- 一张只按主键查询的小表，可能跑得很好；
- 一张没有合适索引、字段又很宽的表，可能早就慢了。

行数只是一个现象，真正需要观察的是：

- SQL 延迟和扫描行数；
- 索引是否合理；
- 工作集能否放进内存；
- CPU、磁盘和写入吞吐；
- 主从复制延迟；
- 未来几年的数据增长速度。

![什么时候才应该分库分表](imgs/10-flowchart-sharding-decision.png)

推荐顺序：

1. 优化 SQL；
2. 调整索引和表结构；
3. 归档冷数据；
4. 根据场景使用读副本或表分区；
5. 扩容单机资源；
6. 仍然扛不住，再考虑分库分表。

### 6.1 分库分表会带来什么新麻烦？

数据分散以后，单张表是轻松了，应用却更累了：

- 跨库事务怎么保证一致性？
- JOIN 怎么做？
- 全局排序和分页怎么做？
- 全局 ID 从哪里来？
- 扩容时怎么迁移数据？
- 某个热门用户全落在一个分片怎么办？

所以 shard key 不能只考虑“能不能取模”，还要看是否均匀、常用查询能否只访问一个分片，以及以后怎么扩容。

MySQL 表分区也不等于分库分表。表分区仍然发生在一个 MySQL 部署中；分库分表则通常把数据放到不同数据库或节点。

**闲狗言：分库分表不是优化的起点，而是单机方案真的走到尽头后的选择。**

## 7. 最后总结

如果只记住 6 句话：

1. 索引像目录，但目录也要占空间和维护成本。
2. 联合索引看列的排列，不看 `WHERE` 的书写顺序。
3. 不要凭感觉优化，用 `EXPLAIN ANALYZE` 看实际执行。
4. Undo 管回滚和旧版本，Redo 管崩溃恢复，MVCC 管快照，锁管冲突。
5. 锁多少数据，往往由 SQL 扫描多少索引范围决定。
6. 先把单机 MySQL 用明白，再决定要不要分库分表。

## 参考资料

本文整合自我以前写的 5 篇文章：

- [学到 emo 之分库分表](https://xiangou.blog.csdn.net/article/details/128910863)
- [数据库设计规范](https://xiangou.blog.csdn.net/article/details/121448905)
- [使我郁郁寡欢的 MySQL 事务和锁](https://xiangou.blog.csdn.net/article/details/115595598)
- [阿里新零售数据库实战：CRUD 优化与在线修改](https://xiangou.blog.csdn.net/article/details/115499603)
- [使我日渐消瘦的 MySQL 索引](https://xiangou.blog.csdn.net/article/details/106245641)

官方文档：

- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
- [Partitioning Overview](https://dev.mysql.com/doc/refman/8.4/en/partitioning-overview.html)
- [MySQL 9.7 Release Notes](https://dev.mysql.com/doc/relnotes/mysql/9.7/en/)
