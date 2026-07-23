# Learning MySQL the Hard Way: From Indexes to Sharding

![Visual MySQL guide cover](imgs/cover.png)

> If life had been kinder, perhaps we would never have needed to learn indexes, locks, transactions, and sharding in one sitting.

This guide combines five of my earlier MySQL notes. Instead of presenting a wall of rules, we will follow one query and answer a few practical questions:

1. Why does an index make a query faster?
2. Why does MySQL sometimes ignore an index?
3. How do Undo, Redo, MVCC, and locks fit together?
4. How should we design and change a table?
5. Does a large table automatically need sharding?

The examples use **MySQL 8.4 LTS** as a production baseline. As of July 2026, **MySQL 9.7** is the latest Innovation release.

## 0. First, the map

Suppose we run:

```sql
SELECT name
FROM user
WHERE phone = '17665395447';
```

MySQL does not immediately scan every row. It parses the SQL, chooses an access path, handles transaction visibility and locks when needed, and only much later—if one server is genuinely insufficient—do we consider sharding.

![The journey from a SQL query to sharding](imgs/01-infographic-mysql-mental-model.png)

**Takeaway: understand the query and its index before reaching for a distributed system.**

## 1. Why does an index make a query faster?

Think about a 500-page book.

Without a table of contents, finding “transaction isolation” means turning pages one by one. With a table of contents, we find the topic first and jump to its page. A database index plays the same role.

- No useful index: inspect rows one by one.
- Useful index: locate the relevant part of the data first.

The index is not free. It consumes storage and must be maintained whenever rows are inserted, updated, or deleted.

### 1.1 Why not use a simple binary tree?

InnoDB normally reads a page from disk, not one number at a time. A page is commonly 16 KiB.

If one tree node held only one key, the tree would be tall and each lookup could require more I/O. A B+ tree stores many keys per page, so it is short and wide. Its leaf pages are linked, which also makes range scans efficient.

![Why InnoDB uses a B+ tree](imgs/02-framework-bplus-tree.png)

**Question: how many rows fit in a three-level B+ tree?**

There is no universal number. It depends on key width, row width, page utilization, and other details. Capacity estimates are useful for learning, not a production limit.

### 1.2 Primary and secondary indexes

InnoDB's primary-key index is clustered: its leaf records contain complete rows.

A secondary-index leaf usually contains:

```text
secondary key + primary-key value
```

For this query:

```sql
SELECT *
FROM user
WHERE phone = '17665395447';
```

MySQL may first find the primary key in the `phone` index and then use that key to fetch the complete row from the clustered index. That second trip is the table lookup.

![A secondary lookup versus a covering index](imgs/03-flowchart-index-lookup.png)

If the query only needs `id` and `phone`, and both are already available from the secondary index, MySQL can return the result without the second trip. That is a covering index.

**Takeaway: do not celebrate merely because a query “uses an index.” Count the work it still performs.**

### 1.3 Why does a composite index have a leftmost prefix?

Consider:

```sql
CREATE INDEX idx_order_status_time
ON orders(status, created_at, id);
```

The index is ordered by `status`, then by `created_at` within the same status, and finally by `id`.

So this is easy to locate:

```sql
WHERE status = 'PAID'
```

This is also convenient:

```sql
WHERE status = 'PAID'
  AND created_at >= '2026-01-01'
```

But searching only by `created_at` skips the leading order. It is like a phone book sorted by city and then name: searching by name alone is less convenient.

![How composite-index order works](imgs/04-infographic-composite-index.png)

The written order of predicates is a different matter:

```sql
WHERE status = 'PAID' AND created_at >= '2026-01-01';
WHERE created_at >= '2026-01-01' AND status = 'PAID';
```

The optimizer can reorder these predicates. “Leftmost” describes the index's column order, not the order in which conditions appear in SQL.

### 1.4 LIKE, functions, and index surprises

```sql
WHERE phone LIKE '176%';   -- usually provides a navigable prefix
WHERE phone LIKE '%176%';  -- the beginning is unknown
```

This can also be troublesome:

```sql
WHERE MONTH(created_at) = 7;
```

The B+ tree is ordered by the original datetime, not the result of `MONTH()`. A range is easier to navigate:

```sql
WHERE created_at >= '2026-07-01'
  AND created_at <  '2026-08-01';
```

MySQL also supports functional indexes and indexed generated columns. Therefore, “functions never use indexes” is too absolute. Check the actual plan.

## 2. How do we find the slow part of a query?

Do not guess. Start with `EXPLAIN`.

```sql
EXPLAIN FORMAT=TREE
SELECT id, status
FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC
LIMIT 50;
```

`EXPLAIN` describes what MySQL plans to do. `EXPLAIN ANALYZE` executes the query and adds actual rows and timings.

![A practical EXPLAIN ANALYZE loop](imgs/05-flowchart-explain-analyze.png)

Look for:

- estimated rows versus actual rows;
- time to the first row and total time;
- steps that loop too many times;
- sorting, temporary tables, and repeated table lookups.

Remember that `EXPLAIN ANALYZE` executes the statement. Do not casually use it with a dangerous write in production.

### 2.1 What about deep pagination?

```sql
SELECT *
FROM posts
ORDER BY id
LIMIT 1000000, 100;
```

MySQL still has to find and skip the first million rows. Later pages therefore cost more.

If the product can use “load more,” remember the last row from the previous page:

```sql
SELECT id, created_at, title
FROM posts
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

This is keyset pagination. It does not require contiguous IDs; it needs stable ordering and a suitable index.

## 3. Undo, Redo, MVCC, and locks

Consider a transfer:

```text
subtract 100 from account A
add 100 to account B
```

If the first operation succeeds and the second fails, money disappears. A transaction makes the two operations succeed or fail as a unit.

![Four mechanisms inside an InnoDB transaction](imgs/06-framework-transaction-engine.png)

A simple mental model:

- **Undo Log** is the undo button.
- **Redo Log** is the crash-recovery record.
- **MVCC** supplies historical versions to readers.
- **Locks** prevent conflicting changes.

### 3.1 Undo: the database's regret mechanism

If a balance changes from 1000 to 900, Undo retains enough information to restore the earlier value. It is used for rollback.

Undo has another job: a snapshot query can follow the version chain backward to reconstruct an older row.

### 3.2 Redo: what if the machine crashes?

A modified page in memory may not yet be written to its final data file. Redo records the changes so committed work can be recovered after a crash.

`innodb_flush_log_at_trx_commit=1` gives the strongest commit durability. A value of `2` can reduce flush work, but recent commits may be lost after an operating-system or machine failure. It is not simply “equally safe but faster.”

### 3.3 MVCC: how can readers avoid blocking writers?

Imagine three versions:

```text
V1 ← V2 ← V3
```

V3 is current. Undo retains information needed to reconstruct V2 and V1. A Read View decides which version a query may see.

![Read Views and the Undo version chain](imgs/07-flowchart-mvcc-visibility.png)

- `READ COMMITTED` normally gets a new snapshot for each consistent-read statement.
- `REPEATABLE READ` normally reuses the snapshot created by the transaction's first consistent read.
- `UPDATE`, `DELETE`, and `SELECT ... FOR UPDATE` need the current version.

So a snapshot is not always created the moment a transaction begins. It is normally created when a consistent read needs it.

Long transactions delay cleanup of old versions and can hold locks for too long. If a transaction can finish in one second, do not let it live for an hour.

## 4. “InnoDB uses row locks,” so why is everything waiting?

The phrase “row-level locking” is easy to misread as “only one row is ever locked.”

InnoDB locks index records and scanned ranges. The path used to find rows therefore influences the lock footprint.

![The access path determines lock scope](imgs/08-comparison-lock-scope.png)

- Unique equality lookup: usually a small footprint.
- Range scan: may lock records and gaps.
- Poor access path: scans more data and can create much more contention.

Ordinary `SELECT` statements under common Read Committed and Repeatable Read settings normally use snapshots. They do not lock records in the same way as `SELECT ... FOR UPDATE`.

**Takeaway: a better index can reduce both query time and lock contention.**

## 5. How should we design and change tables?

Database standards often use the words “must” and “never.” Experience teaches us that many rules are better written as “a good default unless the workload gives us a reason to differ.”

![Schema design and online DDL](imgs/09-infographic-schema-online-ddl.png)

### 5.1 Sensible defaults

- Use InnoDB.
- Use `utf8mb4`.
- Define an explicit primary key.
- Keep the primary key compact and stable.
- Choose data types that are large enough, but not wasteful.
- Create indexes for real query patterns.
- Document what each important field means.

### 5.2 Is NULL forbidden?

No. `NULL` can be indexed.

It represents an unknown or absent value. If that is the real business meaning, use `NULL`. Do not invent fake values such as `0`, an empty string, or `1970-01-01` merely to avoid it.

### 5.3 Are subqueries always bad?

No. The optimizer can transform some subqueries into semi-joins or derived-table plans.

A better rule is:

1. write the correct query;
2. inspect it with `EXPLAIN ANALYZE`;
3. compare the subquery and join forms when performance matters.

### 5.4 Does every ALTER lock the whole table?

Different changes have very different costs:

- `INSTANT`: mostly a metadata change;
- `INPLACE`: performs work on the existing table;
- `COPY`: copies and rebuilds the table.

```sql
ALTER TABLE orders
ADD COLUMN source VARCHAR(32) NULL,
ALGORITHM=INSTANT;
```

“Online” still does not mean “risk-free.” Watch metadata locks, free disk space, replication lag, and long-running transactions.

## 6. Must we shard after five million rows?

No. MySQL has no rule saying that five or twenty million rows automatically require sharding.

Ten million narrow rows queried by a good primary key can behave well. A much smaller table with poor indexes and wide rows can already be slow.

Measure:

- latency and rows examined;
- index quality;
- whether the working set fits in memory;
- CPU, storage, and write throughput;
- replication lag;
- expected growth.

![When sharding becomes reasonable](imgs/10-flowchart-sharding-decision.png)

Try simpler steps first:

1. improve the SQL;
2. improve indexes and schema;
3. archive cold data;
4. consider replicas or table partitioning where appropriate;
5. scale the single node;
6. shard only when one node still cannot meet the goal.

### 6.1 What problems does sharding create?

The table gets smaller, but the application gets harder:

- cross-shard transactions;
- joins, aggregation, and global pagination;
- globally unique IDs;
- routing and data migration;
- rebalancing and hot shards;
- harder backup, recovery, and observability.

A shard key must be evenly distributed, keep common queries local, resist hotspots, and allow future expansion.

Native MySQL table partitioning is not the same as sharding. Partitioning organizes one table inside a MySQL deployment; sharding normally distributes data across databases or nodes.

**Takeaway: sharding is not the beginning of optimization. It is what we consider after simpler single-node solutions reach a measured limit.**

## 7. Six sentences to remember

1. An index is a table of contents, and a table of contents also has maintenance cost.
2. Composite-index order matters; predicate writing order usually does not.
3. Use `EXPLAIN ANALYZE` instead of guessing.
4. Undo handles rollback and old versions; Redo handles recovery; MVCC handles snapshots; locks handle conflicts.
5. The scanned access path often determines the lock footprint.
6. Learn to use one MySQL node well before splitting it into many.

## Sources

This guide combines and updates five of my earlier articles:

- [Learning sharding the hard way](https://xiangou.blog.csdn.net/article/details/128910863)
- [Database design conventions](https://xiangou.blog.csdn.net/article/details/121448905)
- [MySQL transactions and locks](https://xiangou.blog.csdn.net/article/details/115595598)
- [CRUD optimization and online schema changes](https://xiangou.blog.csdn.net/article/details/115499603)
- [MySQL indexes](https://xiangou.blog.csdn.net/article/details/106245641)

Official documentation:

- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
- [Partitioning Overview](https://dev.mysql.com/doc/refman/8.4/en/partitioning-overview.html)
- [MySQL 9.7 Release Notes](https://dev.mysql.com/doc/relnotes/mysql/9.7/en/)
