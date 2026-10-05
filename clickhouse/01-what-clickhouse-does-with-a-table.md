# 1. What ClickHouse does with a table

## The problem

ClickHouse speaks SQL, so people bring their row-database instincts to it: the primary key
identifies a row, an insert is cheap, an update edits a row in place, and an index lists
every row. All four are wrong here, and each wrong instinct turns into a specific bug:
duplicates that never go away, a table with thousands of tiny pieces, a lookup that reads
the whole table. This chapter builds the model the rest of the course relies on.

Skip it if you can explain what a part, a granule and a mark are, and why a filter on the
second column of the sorting key behaves differently from a filter on the first. Everyone
else should read it, because every later chapter assumes it.

All outputs below are measured on ClickHouse 26.9 in a throwaway container. Table and
column names are invented for the examples.

## Columns, not rows

A row database stores each row's fields together, which suits "fetch this one record". An
analytical database usually wants the opposite: "average this one field over a hundred
million records". ClickHouse stores **each column separately**, so a query reads only the
columns it names. Selecting three narrow columns from a wide table is cheap. `SELECT *` on
a table with large array columns is not, because it reads every one of them.

## Every insert writes a part

The MergeTree documentation
([MergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree))
describes the core mechanism: "Insert operations create table parts which are merged by a
background process with other table parts."

A **part** is an immutable directory of column files, sorted. Nothing is ever edited in
place. Three inserts, with background merges paused so we can see them:

```
active parts after 3 inserts        3
active parts after OPTIMIZE FINAL   1
```

`OPTIMIZE TABLE … FINAL` forces a merge. Normally merges happen in the background, when
ClickHouse decides to, which matters a great deal in chapter 2.

Two consequences follow straight away:

- **Insert in batches.** One row per `INSERT` means one part per row, and the merge process
  then spends its life combining tiny parts. Batching is part of the data model, not a
  performance nicety.
- **"Changing" a row means writing another one.** Every engine in this family that appears
  to update or deduplicate rows does it by writing new rows and resolving them later.

## The sorting key and the sparse primary index

Each part is sorted by the table's `ORDER BY` expression, the **sorting key**. The
**primary key** must be a prefix of it, and defaults to all of it.

The primary index isn't an index of rows. From the guide to
[sparse primary indexes](https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes):
"The primary index for a part has one index entry (known as a 'mark') per group of rows
(called 'granule')". A **granule** is 8,192 rows by default (`index_granularity`). So the
index for three million rows has about 367 entries, small enough to live in memory, and a
lookup narrows the search to granules, never to single rows.

```
one part, sorted by (kind, item_id)          (shape only; keys are illustrative)

granule 0    rows 0..8191        mark 0: first key in granule, e.g. ('a', '100…')
granule 1    rows 8192..16383    mark 1: first key in granule, e.g. ('a', '103…')
…
granule 366                      mark 366
```

How much a filter prunes depends on where its column sits in the key. Three million rows,
`ORDER BY (kind, item_id)`, where `kind` has three values:

| Filter | Granules read |
|---|---|
| `kind = 'b' AND item_id = '42'` (leading columns) | 1 / 367 |
| `item_id = '42'` (second column only) | 5 / 367 |
| `owner_id = '42'` (not in the key) | 367 / 367 |

The middle row is the interesting one. Without the leading column, ClickHouse can't
binary-search, and uses what the guide calls a generic exclusion search. It works here only
because `kind` has three values, so `item_id` is sorted within three long runs. Had the
leading column been high-cardinality, the second column would be close to unsorted across
granules, and the filter would read nearly everything.

You can see this for any query with `EXPLAIN indexes = 1`, which prints the granules
selected by each index. It's the first thing to run when a query is slow.

> **Teacher's aside.** In PostgreSQL, "primary key" means identity: unique, enforced, one
> row per value. In ClickHouse it means "the physical sort order, and a coarse index on
> it". The MergeTree documentation says it plainly: "ClickHouse does not require a unique
> primary key. You can insert multiple rows with the same primary key." If you read
> `PRIMARY KEY` in a ClickHouse schema as a uniqueness promise, you'll write code that
> assumes duplicates can't exist, and they will.

## Replicas, shards and Keeper

A **replica** is a full copy of one table on another server. A **shard** is a slice of the
data. The replication documentation
([Data replication](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication))
is explicit that the two are independent: "Each shard has its own independent
replication." Replicas of the same shard hold the same data; different shards hold
different data.

Replication is a property of the **table engine**. `ReplicatedMergeTree` and its siblings
(`ReplicatedReplacingMergeTree` and so on) coordinate through **ClickHouse Keeper** (or
ZooKeeper), where each replicated table has a path that identifies it. A plain `MergeTree`
on a cluster isn't replicated at all: each server's copy is independent. Chapter 5 shows
what does and doesn't travel between servers, measured.

`ON CLUSTER` is a different mechanism with a confusingly similar purpose. It sends a DDL
statement to every server named in a cluster definition. It replicates the *statement*,
not the data.

| Term | What it is |
|---|---|
| Part | An immutable, sorted directory of column files, one per insert until merged |
| Merge | Background process combining parts; the only time many engines resolve duplicates |
| Sorting key (`ORDER BY`) | The order rows are stored in within each part |
| Primary key | A prefix of the sorting key, indexed sparsely; not unique |
| Granule / mark | 8,192 rows by default / the index entry for one granule |
| Replica / shard | A copy of the same data / a slice of the data |
| Keeper | The coordination service replicated tables use |
| `ON CLUSTER` | Runs a DDL statement on every server in a cluster definition |

## Check yourself

1. A service inserts one row per incoming event, 200 events a second. Describe what the
   table looks like on disk after an hour, and what ClickHouse spends its time doing.
2. A table is `ORDER BY (tenant_id, created_at)`. Why might a filter on `created_at` alone
   read almost every granule, while the same filter on a table with `ORDER BY (region,
   created_at)` and four regions reads few?
3. A colleague says "we don't need to check for duplicates, `id` is the primary key". What
   do you tell them, and how would you show them?
4. Why can the primary index of a billion-row table stay in memory?
5. Two servers each run `CREATE TABLE t … ENGINE = MergeTree` through `ON CLUSTER`. A
   service inserts only into the first. What does a query on the second return, and why?
