# 2. Replacing rows: what ReplacingMergeTree does and doesn't promise

## The problem

You want "one current row per thing", the way an upsert gives you in PostgreSQL. ClickHouse
has no upsert, so you reach for `ReplacingMergeTree`: insert a new version of the row, and
the engine throws the old one away. Then the duplicates don't go away. Sometimes they go
away eventually, sometimes never, and sometimes the row that survives is the older one.
Each of those outcomes follows from three rules, and none of them is "the primary key is
unique".

Assumes [chapter 1](01-what-clickhouse-does-with-a-table.md): parts, merges and the sorting
key. Measured on ClickHouse 26.9, with 24.8 where it differs. Names are invented.

## Rule 1: the duplicate key is the whole `ORDER BY`

From the documentation
([ReplacingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replacingmergetree)):
"Uniqueness of rows is determined by the `ORDER BY` table section, not `PRIMARY KEY`."

That sentence matters when the two differ. A table keyed for "latest first" reads:

```sql
CREATE TABLE t_ts (kind String, item_id String, created_at DateTime64(3), payload String)
ENGINE = ReplacingMergeTree
ORDER BY (kind, item_id, created_at)
PRIMARY KEY (kind, item_id);
```

Insert the same item three times, twice with the same timestamp, then force a full merge:

```
after OPTIMIZE FINAL    2
a  x1  2026-01-01 00:00:00.000  first
a  x1  2026-01-02 00:00:00.000  second-retry
with FINAL              2
```

Only the two rows with an identical `(kind, item_id, created_at)` collapsed. The row from a
different day survived a forced merge and survives `FINAL`, because as far as the engine is
concerned it isn't a duplicate. Put `ORDER BY (kind, item_id)` instead and the same inserts
end as one row.

> ⚠️ **A timestamp in the sorting key turns a replacing table into an append-only one.** If
> each write sets `created_at = now()`, no two writes of the same item ever share a key. The
> engine will keep every version for ever, and `FINAL` will pay its cost (below) for
> nothing. Check the last column of `ORDER BY` before trusting any claim that "the engine
> takes care of duplicates".

## Rule 2: which row survives depends on `ver`

`ReplacingMergeTree(ver)` takes an optional version column. The documentation: when `ver`
is given, the survivor is the row "with the maximum version"; without it, "the last in the
selection", meaning "the most recently created part (the last insert)". Ties on `ver` fall
back to the last-inserted rule.

That difference decides what happens when messages arrive out of order. Insert version 5,
then a late version 3:

| Table | Survivor |
|---|---|
| `ReplacingMergeTree(ver)` | `newer` (version 5) |
| `ReplacingMergeTree` (no `ver`) | `older-arrives-late` (version 3) |

Without a version column, "latest" means "latest to arrive", which for a consumer reading
several partitions or replaying a dead-letter queue isn't the same as latest in time. Kafka
gives no order across partitions (see [kafka/02](../kafka/02-partitions-keys-and-order.md)).

## Rule 3: deduplication is eventual

"Data deduplication occurs only during a merge. Merging occurs in the background at an
unknown time, so you can't plan for it." And: "This, however, offers eventual correctness
only - it does not guarantee rows will be deduplicated."

Before a merge, both versions are on disk and an ordinary `SELECT` returns both:

```
before merge, no FINAL   2
before merge, FINAL      1
after merge              1
```

So every read has to resolve duplicates itself. There are several ways, and they aren't
equivalent:

| Approach | What it does | Notes |
|---|---|---|
| `SELECT … FROM t FINAL` | Merges matching rows at read time, using the engine's own rules | Respects `ver`. Cost and index use depend on version (below) |
| `argMax(col, ver) … GROUP BY key` | Picks the value with the highest version per key | Explicit. Measured: returns `newer` even on the table without a `ver` column |
| `ORDER BY ver DESC LIMIT 1 BY key` | Keeps the first row per key after sorting | Same result here; reads all versions |
| `ORDER BY ts DESC LIMIT 1` for one key | Fine for a single lookup | Not a general answer for many keys |

The `argMax` and `LIMIT 1 BY` rows show something useful: if you resolve duplicates
yourself, *you* choose what "latest" means, and the engine's choice stops mattering.

## What `FINAL` costs, and why it changed between versions

`FINAL` does merge work at query time, so it reads more than the same query without it.
How much more depends on whether skip indexes (chapter 4) still apply, and that changed
between releases. Same table, same query, a bloom filter index on the filtered column:

| Version | Without `FINAL` | With `FINAL` |
|---|---|---|
| 24.8 | skip index used: 10 / 379 granules | skip index not used: 379 / 379 |
| 26.9 | skip index used: 10 / 379 | skip index used, then expanded: 132 / 379 |

The setting responsible is `use_skip_indexes_if_final`: 0 in 24.8, 1 in 26.9. Its
description explains the risk it guards against: "Skip indexes may exclude rows (granules)
containing the latest data, which could lead to incorrect results from a query with the
`FINAL` modifier." The newer release pairs it with `use_skip_indexes_if_final_exact_mode`,
which "scans the additional parts that overlap the ranges returned by the skip index",
which is the expansion from 10 to 132 granules.

So "`FINAL` is slow" is true and version-dependent. On older releases it silently disables
every skip index on the table, and a lookup that's fast without `FINAL` becomes a full scan
with it.

> **Teacher's aside.** `ReplacingMergeTree` is a storage optimisation, not a constraint. It
> makes old versions *eventually cheaper to keep*, by deleting them when parts happen to
> merge. It never makes them invisible on its own. Design reads as if every version is still
> there, because at any given moment some of them are.

## Check yourself

1. A table is `ORDER BY (account_id, updated_at)` with `ReplacingMergeTree(updated_at)`.
   Every update writes a new row with `updated_at = now()`. After a week of merges, how many
   rows per account are there, and what does `FINAL` do for you?
2. A consumer reads two Kafka partitions into a `ReplacingMergeTree` with no version column.
   Construct a sequence that leaves the older state as the survivor after a merge.
3. Why can an ordinary `SELECT count()` on a replacing table return a different number at
   10:00 and 10:05 with no inserts in between?
4. A lookup by a bloom-filtered column takes milliseconds without `FINAL` and seconds with
   it on ClickHouse 24.8. Explain mechanically, and name the setting.
5. You need the latest row per key and can't rely on the table's `ORDER BY` being right.
   Write the query that doesn't depend on the engine at all, and say what it costs.
