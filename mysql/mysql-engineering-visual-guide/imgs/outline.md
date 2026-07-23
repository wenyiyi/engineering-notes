---
type: mixed
density: rich
style: sketch-notes
palette: cool-macaron
image_count: 10
---

## Illustration 1

**Position**: Overview
**Purpose**: Show the full path from one SQL statement to distributed storage.
**Visual Content**: Five-layer map: query, index, transaction, schema, sharding.
**Filename**: 01-infographic-mysql-mental-model.png

## Illustration 2

**Position**: Indexes
**Purpose**: Explain why InnoDB uses B+ trees.
**Visual Content**: Disk pages, shallow fan-out, linked leaf pages.
**Filename**: 02-framework-bplus-tree.png

## Illustration 3

**Position**: Clustered and secondary indexes
**Purpose**: Make lookup, table access, and covering indexes visible.
**Visual Content**: Secondary tree to primary tree versus covering path.
**Filename**: 03-flowchart-index-lookup.png

## Illustration 4

**Position**: Composite indexes
**Purpose**: Explain leftmost prefixes and column order.
**Visual Content**: Ordered phone-book analogy using real index terms.
**Filename**: 04-infographic-composite-index.png

## Illustration 5

**Position**: Query diagnosis
**Purpose**: Turn EXPLAIN ANALYZE into a repeatable workflow.
**Visual Content**: Observe, measure, change, verify loop.
**Filename**: 05-flowchart-explain-analyze.png

## Illustration 6

**Position**: Transactions and MVCC
**Purpose**: Connect redo, undo, snapshots, and locks.
**Visual Content**: ACID protection layers and read paths.
**Filename**: 06-framework-transaction-engine.png

## Illustration 7

**Position**: MVCC visibility
**Purpose**: Show version-chain visibility under RC and RR.
**Visual Content**: Read view, transaction IDs, undo chain.
**Filename**: 07-flowchart-mvcc-visibility.png

## Illustration 8

**Position**: Locks
**Purpose**: Show how access paths determine lock scope.
**Visual Content**: Unique lookup, range scan, full scan comparison.
**Filename**: 08-comparison-lock-scope.png

## Illustration 9

**Position**: Schema and online change
**Purpose**: Replace absolute schema rules with evidence-based choices.
**Visual Content**: Small types, short primary keys, online DDL algorithm ladder.
**Filename**: 09-infographic-schema-online-ddl.png

## Illustration 10

**Position**: Sharding
**Purpose**: Show when sharding is justified and its operational cost.
**Visual Content**: Scale-up decision path, shard routing, cross-shard trade-offs.
**Filename**: 10-flowchart-sharding-decision.png
