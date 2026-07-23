# MySQL Engineering, Visually: From One Query to Sharding

![MySQL Engineering, Visually cover](imgs/cover.png)

> A visual field guide for backend engineers: indexes, query plans, transactions, MVCC, locks, online DDL, and sharding. Examples use **MySQL 8.4 LTS** as the production baseline; as of July 2026, **MySQL 9.7** is the current Innovation release.

[中文版 / Chinese version](README.zh-CN.md)

## Start with the whole map

A query's performance and reliability are shaped by five layers:

![MySQL engineering mental model](imgs/01-infographic-mysql-mental-model.png)

Investigate from near to far: SQL and access path first, then transactions and schema, and sharding last. Distributed complexity should not conceal a single-node problem.

## 1. Indexes: optimize work, not the “uses index” badge

InnoDB primarily uses B+ trees. A page holds many keys, creating high fan-out and a shallow tree; linked leaf pages support efficient range scans. The default page size is commonly 16 KiB, but tree capacity depends on row width, key width, and page utilization—there is no universal row-count formula.

![B+ tree and disk pages](imgs/02-framework-bplus-tree.png)

### Clustered, secondary, and covering indexes

- Clustered-index leaves store full rows.
- Secondary-index leaves store secondary keys plus primary-key values.
- Following a secondary entry into the clustered tree is a table lookup.
- If one index contains every required column, it can cover the query.

![InnoDB index lookup paths](imgs/03-flowchart-index-lookup.png)

Keep primary keys short and stable because each secondary record carries the primary key. Auto-increment keys are often practical, but not doctrine: consider write hotspots, ID locality, index width, and business semantics.

### Order composite indexes around query patterns

![Composite index column order](imgs/04-infographic-composite-index.png)

For `(status, created_at, id)`, reason about the ordered prefix:

```sql
CREATE INDEX idx_order_status_time
    ON orders (status, created_at, id);
```

Equality columns usually precede a range column; ordering and coverage influence the remaining columns. Predicate order in the SQL text does not determine index usability—the optimizer can reorder predicates.

```sql
-- Navigable prefix range
WHERE phone LIKE '176%'

-- Usually cannot seek to a B+ tree start point
WHERE phone LIKE '%176%'
```

Functions and implicit conversions can make a predicate non-sargable. Functional indexes and indexed generated columns may help; verify the actual plan rather than memorizing absolutes.

## 2. Close the loop with EXPLAIN ANALYZE

![EXPLAIN ANALYZE tuning loop](imgs/05-flowchart-explain-analyze.png)

```sql
EXPLAIN FORMAT=TREE
SELECT id, status
FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC
LIMIT 50;

EXPLAIN ANALYZE
SELECT id, status
FROM orders
WHERE status = 'PAID'
ORDER BY created_at DESC
LIMIT 50;
```

Compare estimated versus actual rows, first-row and total time, loops, rows scanned, sorting, temporary tables, and table lookups. `EXPLAIN ANALYZE` executes the statement, so assess the impact before using it on production writes or expensive reads.

For deep pagination, do not assume contiguous IDs:

```sql
SELECT id, created_at, title
FROM posts
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

Keyset pagination resumes from an indexed position instead of paying for an ever-growing `OFFSET`.

## 3. Transactions: logs, versions, and locks have different jobs

![InnoDB transaction protection mechanisms](imgs/06-framework-transaction-engine.png)

- **Undo** supports rollback and reconstructs older versions for consistent reads.
- **Redo** records page changes for crash recovery.
- **MVCC** provides snapshot reads.
- **Locks** protect current reads and conflicting writes.

`innodb_flush_log_at_trx_commit=1` provides the strongest commit-durability behavior. A value of `2` can reduce flush cost but may lose recent commits after an operating-system or machine failure; it is not simply “lossless.”

### MVCC: which version is visible?

![MVCC visibility and undo chain](imgs/07-flowchart-mvcc-visibility.png)

InnoDB records carry a transaction identifier and a roll pointer into undo. A Read View decides whether a version is visible:

- `READ COMMITTED` normally takes a fresh snapshot for each consistent-read statement.
- `REPEATABLE READ` normally reuses the snapshot established by the transaction's first consistent read.
- `UPDATE`, `DELETE`, and `SELECT ... FOR UPDATE/FOR SHARE` perform current reads.

So “a snapshot is always created at transaction start” is inaccurate. Long transactions delay purge and hold locks longer; keep their boundaries small.

## 4. Locks: the access path shapes the footprint

![Access path and lock scope](imgs/08-comparison-lock-scope.png)

InnoDB locks index records and scanned ranges. Locking reads, `UPDATE`, and `DELETE` generally lock index records encountered by execution; under Repeatable Read, range access can also use next-key locks.

- A unique equality lookup usually has the narrowest footprint.
- A range scan can lock records and gaps.
- A broad scan caused by a poor index amplifies contention.

Ordinary snapshot `SELECT` statements normally set no record locks under Read Committed and Repeatable Read. The useful rule is not “InnoDB has row locks”; it is **the access path determines the lock footprint**.

## 5. Schema and online changes: rules should serve the workload

![Schema design and online DDL](imgs/09-infographic-schema-online-ddl.png)

Good defaults:

- InnoDB and `utf8mb4`;
- an explicit, compact, stable primary key;
- the smallest sufficient data types;
- indexes that support measured query patterns;
- `NULL` based on domain meaning—not fake zeroes or magic dates.

“Never use subqueries,” “indexed columns must never be NULL,” and “ALTER always blocks the whole table” are too absolute. MySQL can transform subqueries into semi-joins or derived-table plans. DDL behavior depends on the operation, version, and algorithm:

```sql
ALTER TABLE orders
  ADD COLUMN source VARCHAR(32) NULL,
  ALGORITHM=INSTANT;
```

`INSTANT`, `INPLACE`, and `COPY` have very different costs. Test with production-like volume and monitor metadata locks, replication lag, disk headroom, and rollback options.

## 6. Sharding: there is no universal row threshold

![MySQL sharding decision path](imgs/10-flowchart-sharding-decision.png)

“Five million rows” and “twenty million rows” are not MySQL limits. The decision depends on working-set fit, index quality, write throughput, storage and replication capacity, latency targets, and whether vertical scaling, archiving, partitioning, or read replicas have been exhausted.

Shard only after measurement proves that a single node still cannot meet the objective.

### Design the costs together with the shard key

Evaluate balance, query locality, hotspots, and future expansion. Sharding introduces:

- cross-shard transactions and consistency;
- distributed joins, aggregation, and pagination;
- global IDs;
- routing, rebalancing, migration, and hot shards;
- more complex backup, recovery, and observability.

Native MySQL table partitioning organizes data inside a MySQL deployment; it is not cross-node sharding. InnoDB partitioning in MySQL 8.4 also has limitations, including foreign-key restrictions.

## One-page checklist

1. Find the real bottleneck with slow logs, metrics, and `EXPLAIN ANALYZE`.
2. Match indexes to filtering, ordering, joins, and coverage.
3. Keep transactions short; distinguish snapshot reads from current reads.
4. Infer lock scope from the access path.
5. Make the DDL algorithm, locks, space, and rollback plan explicit.
6. Use sharding last, and own its distributed-systems costs.

## Sources and further reading

This guide consolidates and updates five earlier posts by the author:

- [MySQL indexes](https://xiangou.blog.csdn.net/article/details/106245641)
- [MySQL transactions and locks](https://xiangou.blog.csdn.net/article/details/115595598)
- [Database CRUD and online changes](https://xiangou.blog.csdn.net/article/details/115499603)
- [Database design conventions](https://xiangou.blog.csdn.net/article/details/121448905)
- [Database and table sharding](https://xiangou.blog.csdn.net/article/details/128910863)

Official references:

- [MySQL 8.4 Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/)
- [Optimization and Indexes](https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html)
- [InnoDB Multi-Versioning](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html)
- [Locks Set by Different SQL Statements](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- [Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
- [Partitioning Overview and Limitations](https://dev.mysql.com/doc/refman/8.4/en/partitioning-overview.html)
- [MySQL 9.7 Release Notes](https://dev.mysql.com/doc/relnotes/mysql/9.7/en/)
