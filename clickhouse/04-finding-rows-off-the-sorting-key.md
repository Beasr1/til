# 4. Finding rows off the sorting key

## The problem

A table is sorted for one access pattern, and then a second one arrives: "fetch everything
for this owner", where the owner isn't in the sorting key. The query reads the whole table.
Someone adds a bloom filter index, deploys it, and the query is exactly as slow as before.
Someone else proposes a projection, and the migration fails. Each of these has a mechanical
reason, and knowing them saves a week of "we added an index and nothing happened".

Assumes [chapter 1](01-what-clickhouse-does-with-a-table.md) (granules, marks, `EXPLAIN
indexes = 1`) and [chapter 2](02-replacing-rows.md) (`FINAL`). Measured on ClickHouse 26.9,
and 24.8 where marked. Names are invented.

## A skip index is a summary per granule

A **data skipping index** stores, for each block of granules, a small summary of a column's
values: a min and max (`minmax`), a set of distinct values (`set`), or a bloom filter
(`bloom_filter`). At query time ClickHouse checks each summary and skips granules that can't
contain a match. Unlike a B-tree it doesn't point at rows. It can only say "definitely not
here" or "maybe here".

The documentation
([data skipping indexes](https://clickhouse.com/docs/optimize/skipping-indexes)) defines
the size of a block: "Each indexed block consists of GRANULARITY granules." With
`GRANULARITY 1`, there's one summary per 8,192 rows.

## Adding an index doesn't index what's already there

Three million rows, sorted by `(kind, item_id)`, filtering on `owner_id`, which isn't in the
key. Each owner appears in three rows scattered across the table.

```
after ADD INDEX only        Skip idx_owner   Granules: 367/367
after MATERIALIZE INDEX     Skip idx_owner   Granules: 6/367
```

The documentation: "Normally skip indexes are only applied on newly inserted data, so just
adding the index won't affect the above query." `ALTER TABLE … ADD INDEX` changes the table
definition and indexes new parts as they're written or merged. Existing parts have no
summary, so every one of their granules is a "maybe". Building the index for existing data
is a separate statement, `ALTER TABLE … MATERIALIZE INDEX`, which runs as a background
mutation (add `SETTINGS mutations_sync = 2` to wait for it).

> ⚠️ A migration that only runs `ADD INDEX` passes review, applies cleanly, shows the index
> in `SHOW CREATE TABLE`, and changes nothing for historical data. The only way to know is
> `EXPLAIN indexes = 1`, before and after.

## A skip index needs the values to cluster

Six granules out of 367 is good, and it's good here because each owner's rows sit in only a
few granules. The documentation is direct about when that fails: "In most cases a useful
skip index requires a strong correlation between the primary key and the targeted,
non-primary column/expression." If every granule contains a few rows of every owner, every
bloom filter says "maybe", and the index costs space and time for nothing.

So a skip index works when values are rare or clustered: an id that appears in a handful of
rows, or a value that tends to arrive together in time when the sorting key starts with
time. It doesn't work for a column with a few common values spread evenly.

Two smaller points:

- A skip index on a **column that's already the leading sorting key** adds little. The
  primary index already narrowed that filter to one granule in chapter 1's table.
- Under `FINAL`, older releases don't use skip indexes at all (chapter 2: 379 of 379
  granules on 24.8, because `use_skip_indexes_if_final` defaulted to 0).

## Projections: a second sort order, but not on every engine

A **projection** stores a copy of the data, or an aggregate of it, in a different order
inside the same table, and the optimiser can choose it for queries that match. For "filter
by owner" on a table sorted by item, a projection ordered by owner is the textbook answer.

On a `ReplacingMergeTree` it fails by default, on both versions tested:

```
26.9: ADD PROJECTION is not supported in ReplacingMergeTree with
      deduplicate_merge_projection_mode = throw. Please set setting
      'deduplicate_merge_projection_mode' to 'drop' or 'rebuild'.
24.8: Projection is fully supported in ReplacingMergeTree with
      deduplicate_merge_projection_mode = throw. Use 'drop' or 'rebuild' option of
      deduplicate_merge_projection_mode.
```

(The 24.8 message says "fully supported" in the same breath as refusing; read it as "not
supported".) The reason is that a merge which removes duplicate rows from the main data
would leave the projection holding rows that no longer exist. The setting's description
lists the choices, `ignore`, `throw`, `drop`, `rebuild`, and warns that `ignore` "might
result in incorrect answer". So a projection on a replacing table is a deliberate decision
about what happens at each merge, not a one-line migration.

## Choosing between them

| Want | Tool | Watch for |
|---|---|---|
| Fast lookups by the main access pattern | Put those columns first in `ORDER BY` | Only one sort order per table |
| Occasional lookups by a rare value (an id) | `bloom_filter` skip index, then `MATERIALIZE INDEX` | Useless if values don't cluster; ignored under `FINAL` on older releases |
| Frequent lookups by a second key | Projection, or a second table sorted that way | Replacing engines need `deduplicate_merge_projection_mode` chosen |
| Range filters on a value correlated with time | `minmax` skip index | Correlation again |

> **Teacher's aside.** In a row database, an index is a separate structure that finds rows,
> and adding one is almost always a win. In ClickHouse the sorting key does most of the
> finding, and a skip index only lets it *skip* blocks. The question to ask about any
> proposed index isn't "is the column filtered on?" but "are the matching rows bunched into
> few granules?" `EXPLAIN indexes = 1` answers it in one command.

## Check yourself

1. A migration adds a bloom filter index on a column with 50 million distinct values. The
   query using it doesn't speed up. Give two separate reasons that could be, and how you'd
   tell which.
2. Why is a bloom filter on a column with three distinct values, evenly spread, close to
   useless, while the same index on a near-unique column is useful?
3. You add a skip index and run `MATERIALIZE INDEX` without `mutations_sync`. Someone checks
   `EXPLAIN` immediately and sees no improvement. What's happening?
4. A query reads 10 granules without `FINAL` and all of them with `FINAL`. What release
   behaviour does that indicate, and what are your options?
5. Why does a projection conflict with a replacing engine's merges, and what does each of
   `drop` and `rebuild` cost?
