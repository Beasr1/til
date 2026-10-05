# ClickHouse from the Application Side — A Course

A course on what ClickHouse does with the rows an application sends it: why the primary key
isn't unique, why replacing rows is eventual, how a consumer writes so that retries are
harmless, why an added index can change nothing, and what a schema change does on a cluster.

**This is reference learning material.** Everything here is general ClickHouse, checked
against the ClickHouse documentation and reproduced on ClickHouse 26.9 (and 24.8 where the two
differ) in throwaway containers, including a two-replica cluster with embedded Keeper. Driver
and migration-runner behaviour is read from the source of `clickhouse-go` v2 and
golang-migrate v4. The motivating failures are ordinary: duplicates that never merged away, a
bloom filter index that did nothing for existing data, a lookup that went from milliseconds
to seconds when `FINAL` was added, a version counter that handed out every number twice, and
a migration runner that stopped after the migration folder was tidied.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the rule that prevents it.
- Every chapter ends with **Check yourself** questions. Answers are in
  [07-exercises.md](07-exercises.md).
- Measurements are labelled as measured, with the server version. Table and column names in
  outputs are invented.

## The one thing to understand first

> **ClickHouse never edits a row. Every insert writes a new immutable part, and anything that
> looks like an update, a deduplication or an index is resolved later, at merge time or at
> read time, using the sorting key.** Most surprises in this course are a case of expecting
> it to happen at write time.

## Reference implementations

| Source | What it settles |
|---|---|
| [MergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree) | Parts and merges, primary key as a prefix of the sorting key, non-unique primary keys |
| [Sparse primary indexes](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes) | Granules, marks, generic exclusion search |
| [ReplacingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replacingmergetree) | `ORDER BY` as the duplicate key, the `ver` rule, eventual deduplication, `FINAL` |
| [Data skipping indexes](https://clickhouse.com/docs/optimize/skipping-indexes) | New data only until `MATERIALIZE INDEX`; the correlation requirement |
| [Data replication](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication) | What replicates, block deduplication, per-shard replication, converting to replicated |
| [Distributed DDL](https://clickhouse.com/docs/sql-reference/distributed-ddl) | What `ON CLUSTER` does |
| [RENAME](https://clickhouse.com/docs/sql-reference/statements/rename) / [EXCHANGE](https://clickhouse.com/docs/sql-reference/statements/exchange) | Multi-table rename isn't atomic; single-pair exchange is |
| `system.settings`, `system.merge_tree_settings`, `system.server_settings` | The defaults of the server you're actually running, with descriptions |
| [clickhouse-go v2](https://github.com/ClickHouse/clickhouse-go) `conn_batch.go` | One failed `Append` invalidates the batch |
| [golang-migrate](https://github.com/golang-migrate/migrate) `database/clickhouse`, `source/iofs` | In-process lock, version table, missing and duplicate versions |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What ClickHouse does with a table](01-what-clickhouse-does-with-a-table.md) | ⭐ Explain parts, merges, the sorting key, granules and marks, replicas, shards and `ON CLUSTER`, and read `EXPLAIN indexes = 1`. **Prerequisite for everything else** |

### Part 1 — Rows that look like updates

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Replacing rows](02-replacing-rows.md) | ⭐ Say which rows a ReplacingMergeTree will collapse and which survive, and choose between `FINAL`, `argMax` and `LIMIT 1 BY` |
| 3 | [Writing from a consumer](03-writing-from-a-consumer.md) | Make retried inserts harmless, avoid poisoned batches, and see why read-then-increment can't work |

### Part 2 — Reading and changing

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Finding rows off the sorting key](04-finding-rows-off-the-sorting-key.md) | Decide whether a skip index will help, build it for existing data, and know when a projection is refused |
| 5 | [Changing schema on a cluster](05-changing-schema-on-a-cluster.md) | ⭐ Predict which servers a DDL statement reaches, keep Keeper paths straight, and run migrations without a runner revolt |

### Reference

| # | File | |
|---|------|---|
| 6 | [Glossary](06-glossary.md) | Terms, each linked to its chapter |
| 7 | [Exercises & answers](07-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first, then 2 before 3. Chapter 4 needs chapter 2's `FINAL` section. Chapter 5
stands on chapter 1 and, for migration runners in general, on
[postgres/10](../postgres/10-migration-runners.md). Not yet written, and belonging here:
partitioning and TTL, mutations and lightweight deletes, `Distributed` tables and sharding
keys, materialized views, and `async_insert` in depth. The questions at the end of file 07 are
the honest list.

## If you're short on time

- **New to ClickHouse:** chapter 1, all of it.
- **10 minutes:** chapter 2, "Rule 1" and the trap after it.
- **Duplicates won't go away:** chapter 2, then chapter 3's first section.
- **A query is slow, or an index did nothing:** chapter 4's first two sections, then
  chapter 2's `FINAL` table.
- **About to run DDL on a cluster:** chapter 5 top to bottom.

## The one-paragraph summary of everything

ClickHouse stores each column separately and writes every insert as an immutable sorted part,
which background merges combine later, so inserts belong in batches. The sorting key
(`ORDER BY`) is the physical order, and the primary key is a prefix of it indexed sparsely, one
mark per 8,192-row granule; it isn't unique. Filters on the leading key columns prune to a few
granules, filters on later columns prune only if earlier ones have few values, and filters off
the key read everything. ReplacingMergeTree keeps one row per *whole sorting key*, not per
primary key, so a timestamp in `ORDER BY` means nothing is ever replaced; with a `ver` column
the highest version wins, without one the last insert wins, which is wrong when messages arrive
out of order. Deduplication happens only when parts merge, at an unknown time, so reads must
resolve duplicates with `FINAL`, `argMax` or `LIMIT 1 BY`, and `FINAL` ignored skip indexes on
older releases (24.8) but not newer ones (26.9). Replicated tables drop a retried insert only
if the block is identical, so build rows from the message, never from `now()` in the
application. In clickhouse-go one failed `Append` invalidates the whole batch, and
read-max-plus-one counters hand out duplicates under concurrency because there's no atomic
increment. A skip index covers only new parts until `MATERIALIZE INDEX`, and helps only if
matching values cluster into few granules; projections are refused on replacing tables unless
you choose what merges do to them. On a cluster, replicated tables carry `ALTER` and inserts
between replicas of a shard, but `CREATE`, `DROP` and `RENAME` need `ON CLUSTER`; plain tables
carry nothing. The Keeper path is a replicated table's identity, the default path needs
`ON CLUSTER`, and multi-table `RENAME` isn't atomic. golang-migrate's ClickHouse driver has
only an in-process lock, fails on a recorded version that's no longer in the folder or on
duplicate numbers, and silently ignores files it can't parse.

## How to use me

Ask me things like:

- "Here's a `CREATE TABLE`. Which rows will merge away, and which will stay for ever?"
- "Here's `EXPLAIN indexes = 1` for a slow query. What would make it read fewer granules?"
- "Is this consumer's insert idempotent under retry and redelivery?"
- "Walk me through converting this table to a replicated engine on a live cluster."
- "Which of these migrations will reach every server, and which only the one we connect to?"
