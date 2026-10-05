# 6. Glossary

Reference, not reading. Each term links to where it's taught.

| Term | Meaning | Taught in |
|---|---|---|
| **argMax(x, v)** | Aggregate returning the `x` from the row with the largest `v`; a way to pick the latest version yourself | [02](02-replacing-rows.md) |
| **Block deduplication** | Dropping an insert whose whole block hashes the same as a recent one; on by default for replicated tables | [03](03-writing-from-a-consumer.md) |
| **Distributed DDL / `ON CLUSTER`** | Running a DDL statement on every server in a cluster definition | [01](01-what-clickhouse-does-with-a-table.md), [05](05-changing-schema-on-a-cluster.md) |
| **`EXCHANGE TABLES`** | Atomic swap of two table names (one pair; `Atomic` databases) | [05](05-changing-schema-on-a-cluster.md) |
| **`EXPLAIN indexes = 1`** | Shows how many parts and granules each index selected | [01](01-what-clickhouse-does-with-a-table.md) |
| **`FINAL`** | Query modifier that resolves duplicates at read time using the engine's rules | [02](02-replacing-rows.md) |
| **Generic exclusion search** | How the primary index handles a filter on a non-leading key column; effective only if earlier columns have low cardinality | [01](01-what-clickhouse-does-with-a-table.md) |
| **Granule** | Group of rows (8,192 by default) that the primary index addresses as a unit | [01](01-what-clickhouse-does-with-a-table.md) |
| **Keeper** | ClickHouse's coordination service (ZooKeeper-compatible) used by replicated tables | [01](01-what-clickhouse-does-with-a-table.md) |
| **Keeper path** | The path a replicated table registers under; same path means same table | [05](05-changing-schema-on-a-cluster.md) |
| **Mark** | One primary index entry, for one granule | [01](01-what-clickhouse-does-with-a-table.md) |
| **`MATERIALIZE INDEX`** | Builds a skip index for parts written before the index existed | [04](04-finding-rows-off-the-sorting-key.md) |
| **Merge** | Background combining of parts; when replacing engines drop duplicates | [01](01-what-clickhouse-does-with-a-table.md), [02](02-replacing-rows.md) |
| **Part** | Immutable sorted set of column files written by one insert, until merged | [01](01-what-clickhouse-does-with-a-table.md) |
| **Primary key** | A prefix of the sorting key, indexed sparsely; not a uniqueness constraint | [01](01-what-clickhouse-does-with-a-table.md) |
| **Projection** | A second, differently ordered or aggregated copy of the data inside a table | [04](04-finding-rows-off-the-sorting-key.md) |
| **ReplacingMergeTree** | Engine that keeps one row per sorting key when parts merge | [02](02-replacing-rows.md) |
| **Replica** | Another server's copy of the same shard of a replicated table | [01](01-what-clickhouse-does-with-a-table.md) |
| **Shard** | A slice of a table's data; replication is per shard | [01](01-what-clickhouse-does-with-a-table.md) |
| **Skip index** | Per-block summary (minmax, set, bloom filter) that lets reads skip granules | [04](04-finding-rows-off-the-sorting-key.md) |
| **Sorting key** | The `ORDER BY` of a MergeTree table: physical order, and the duplicate key for replacing engines | [01](01-what-clickhouse-does-with-a-table.md), [02](02-replacing-rows.md) |
| **`ver` column** | Optional version argument of ReplacingMergeTree; highest wins at merge | [02](02-replacing-rows.md) |
