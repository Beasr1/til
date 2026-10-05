# 7. Exercises

Worked answers to every **Check yourself** question, then things to try and questions
worth asking me.

---

## File 01 — What ClickHouse does with a table

**1. One row per insert, 200 a second, for an hour.**

Each insert writes a part, so an hour produces 720,000 parts before merging (200 × 3,600).
The background merger spends its time combining them, constantly rewriting the same data
into bigger parts, and the part count stays high because new ones arrive faster than merges
finish. Queries open many small parts instead of a few large ones. The fix is upstream:
batch rows in the application (or let the server buffer them, chapter 3 mentions
`async_insert`) so each insert carries thousands of rows.

**2. `created_at` filter under two sorting keys.**

With `ORDER BY (tenant_id, created_at)` and many tenants, `created_at` is sorted only within
each tenant's run. A time range matches some rows in nearly every tenant's run, so nearly
every granule might contain a match, and the generic exclusion search can't skip much. With
`ORDER BY (region, created_at)` and four regions, there are only four runs, each sorted by
time, so a time range is four contiguous stretches and everything else is skipped. The
cardinality of the leading column decides how sorted the second column is across granules.

**3. "`id` is the primary key, so no duplicates".**

ClickHouse doesn't enforce uniqueness: "ClickHouse does not require a unique primary key.
You can insert multiple rows with the same primary key." Show them: insert the same `id`
twice and `SELECT count() … WHERE id = …` returns 2. The primary key is a sort order and a
sparse index, nothing more.

**4. The index of a billion rows in memory.**

It has one entry per granule, not per row: a billion rows at 8,192 per granule is about
122,000 marks per key column. That fits in memory easily. The price is that the index can
only narrow a search to granules, and each read decompresses at least one granule.

**5. `MergeTree` created `ON CLUSTER`, inserts only into the first server.**

The second server returns no rows. `ON CLUSTER` ran the `CREATE` on both servers, but a plain
`MergeTree` isn't replicated, so each server's table is independent. Measured in chapter 5:
a row inserted on one server, zero on the other.

## File 02 — Replacing rows

**1. `ORDER BY (account_id, updated_at)`, updates with `updated_at = now()`.**

Every update has a different `updated_at`, so every update has a different sorting key and
none is a duplicate of another. After a week of merges there's one row per update, per
account. `FINAL` does read-time merging and finds nothing to merge, so you pay its cost for
no change. The version column doesn't help either: it only chooses between rows that already
share a sorting key.

**2. Two partitions, no version column, older state survives.**

Partition 0 holds "state v5" for account A, and partition 1 holds "state v3" for A, produced
earlier but on a different partition (a key change, or two producers). The consumer reads
partition 0 first and inserts v5, then catches up on partition 1 and inserts v3. Both rows
share the sorting key. Without `ver`, the merge keeps "the last in the selection", the most
recently inserted part, which is v3. With `ReplacingMergeTree(version)`, v5 would win.

**3. `count()` changes with no inserts.**

Between 10:00 and 10:05 a background merge combined parts and removed duplicate rows. A
plain `SELECT` counts every stored row, duplicates included, so it changes whenever merges
run. `SELECT count() FROM t FINAL` would give the same answer both times.

**4. Fast without `FINAL`, slow with it, on 24.8.**

On 24.8, `use_skip_indexes_if_final` defaults to 0, so under `FINAL` the bloom filter isn't
consulted and every granule is read, then merged. Measured: 10 of 379 granules without
`FINAL`, 379 of 379 with it. Options: upgrade (26.9 defaults the setting to 1 and keeps
results correct with `use_skip_indexes_if_final_exact_mode`), resolve duplicates yourself
with `argMax` or `LIMIT 1 BY` so you don't need `FINAL`, or enable the setting on 24.8 and
accept that, per its description, skipping can exclude "rows (granules) containing the latest
data".

**5. Latest row per key without depending on the engine.**

```sql
SELECT key, argMax(payload, version) AS payload
FROM t
WHERE <filters>
GROUP BY key
```

It reads every stored version of each matching key and keeps the highest, so its cost grows
with the number of unmerged versions, but its answer never depends on merges, the sorting
key or the engine's tie-break rule. If there's no version column, use a timestamp carried by
the message, not one set at write time.

## File 03 — Writing from a consumer

**1. Retry with `ingested_at = now()` set in the application.**

Twice. Block deduplication hashes the whole block, and the retried block has different
`ingested_at` values, so it isn't identical. Measured: the same two rows with `now64(3)`
re-evaluated were stored again. Fix: take timestamps from the message, or drop the column
from the insert and let the server fill it with `DEFAULT now64(3)`, which measured as still
deduplicated.

**2. One bad row in a batch of 1,000, error logged.**

Nothing from the batch is in the table. In clickhouse-go v2 the first failed `Append` stores
the error and releases the connection. Every later `Append` returns the same error, and
`Send` returns it without sending. The error surfaces at `Send` as a batch failure, far from
the logged `Append` error, and the 999 good rows go with it.

**3. Two consumers stamping `max(version) + 1`.**

Both read the same maximum and stamp the same next number, so most version numbers cover two
unrelated batches (measured: 40 claims, 20 distinct numbers, every number claimed twice). A
sync job asking "everything since version N" can still get every row if it filters with `>`,
but anything that treats a version as one unit of work, counts its rows, or resumes "after
version N" within a version, is wrong. And if the version table is a replacing table, the
history of which batch had which number merges away.

**4. Name-based ids and the check-then-insert race.**

The race makes two writers each create an id for the same new entity. With random ids, the
entity ends up with two identities and every later lookup picks one arbitrarily. With an id
computed from the entity's natural key, both writers produce the same id, so the race only
produces identical rows, which deduplication can collapse. The input has to be the same for
every writer: the same field, the same normalisation (case, whitespace), the same namespace
and the same separator between parts.

**5. Lookup with an empty identifier.**

`WHERE ident = ''` matches every row that stored `''` for "missing", so `LIMIT 1` returns one
of them: a real record belonging to someone else. The caller can't tell it apart from a
genuine hit. Refuse empty keys before querying, and store absence as one representation.

## File 04 — Finding rows off the sorting key

**1. Bloom filter on a high-cardinality column doesn't help.**

Either the index exists only for new parts (no `MATERIALIZE INDEX`), or the values don't
cluster, so most granules contain some matching value and every filter says "maybe". Tell
them apart with `EXPLAIN indexes = 1`: if the skip index line shows `Granules: N/N` for old
parts it isn't built; after materialising, if it still selects most granules, the data isn't
clustered. A third possibility is `FINAL` on an older release.

**2. Three values evenly spread versus near-unique.**

A bloom filter answers "might this value be in this block?". With three common values in
every granule, the answer is yes for every granule, so nothing is skipped. With a near-unique
value, the value is in one or two granules and the rest answer no (apart from the configured
false-positive rate).

**3. `MATERIALIZE INDEX` without `mutations_sync`.**

It's a mutation running in the background. The statement returns before old parts have their
index built, so `EXPLAIN` straight afterwards still shows the old parts unindexed. Wait for
the mutation (`system.mutations`), or run it with `SETTINGS mutations_sync = 2`.

**4. Ten granules without `FINAL`, all with it.**

An older release where `use_skip_indexes_if_final` defaults to 0. Options are the same as
file 02 question 4: upgrade, avoid `FINAL` by resolving duplicates explicitly, or enable the
setting knowing the correctness caveat.

**5. Projections and replacing merges.**

A merge of a replacing table removes rows from the main data. The projection, a separate
sorted copy, still contains them unless it's dealt with, so queries routed to the projection
would see rows that no longer exist. The setting's description says `drop` and `rebuild` are
"the action when merge projections", which I read as: `drop` removes the projection from the
merged part (reads fall back to the main data for that part until it's materialised again),
and `rebuild` rebuilds it during the merge (correct, but every merge costs more). I haven't
measured either; treat the costs as reasoning, not data.

## File 05 — Changing schema on a cluster

**1. `ALTER … ADD COLUMN` on one server of a 2×2 replicated cluster.**

The server you ran it on and its fellow replica in the same shard: replicated tables
propagate `ALTER` within a shard through Keeper (measured on one shard). The two servers of
the other shard don't get it, because "each shard has its own independent replication". Use
`ON CLUSTER` to reach every shard.

**2. Same explicit path on two servers; then a path without the database.**

Same explicit path on two servers: yes, they're replicas, because the Keeper path is the
identity (measured: a row inserted on one appeared on the other). If the path omits the
database and each server hosts `db1.t` and `db2.t`, the second `CREATE` on each server fails
with `REPLICA_ALREADY_EXISTS`, because both tables claim the same replica entry. If it somehow
succeeded on different servers, the two databases' tables would replicate into each other.

**3. Copy-then-rename across two migration files.**

Rows written to the old table after the copy's `SELECT` started are lost when the names swap.
On a cluster of unreplicated tables, the copy took only the connected server's rows. Between
the two files, the runner may stop, be interrupted or fail, leaving a window, possibly hours,
where the expected table name doesn't exist and inserts fail. And the multi-table `RENAME` is
documented as not atomic.

**4. Migrations 10–29 collapsed into 10–14.**

On a database at version 29, `Up` checks that version 29 exists in the folder, doesn't find
it, and fails with `no migration found for version 29`. It does nothing until someone checks
the schema by hand and forces a version. A fresh database applies the five new files and ends
at 14, with whatever schema they describe, which may differ from what 10–29 produced (an
engine argument or a version column changed during the tidy-up). Same folder, two schemas.

**5. Two pods running golang-migrate at once.**

Nothing on ClickHouse. The driver's lock is an in-process compare-and-swap, so each process
holds its own lock. Both read version 14, both run 15, and both record it. Run migrations from
a single job in the deploy, not on startup of every replica.

**6. `ORDER BY (k, ts DESC)` tested on a recent server.**

On an older server, 24.8 measured, the statement is a syntax error, so the migration fails in
production and leaves the runner dirty. Pin development to production's server version, and
run migrations against that version in CI.

---

## Things to try

- Create a `ReplacingMergeTree` with a timestamp as the last `ORDER BY` column, insert the same
  key three times, run `OPTIMIZE … FINAL`, and count. Then move the timestamp to a `ver`
  argument and repeat.
- Add a bloom filter index to a three-million-row table and compare `EXPLAIN indexes = 1`
  before and after `MATERIALIZE INDEX`. Then sort the table so the indexed column clusters, and
  compare again.
- Run the same `FINAL` query under 24.8 and a current release and compare granules read.
- Start two ClickHouse servers with embedded Keeper, and work out by experiment which of
  `CREATE`, `ALTER`, `RENAME` and `INSERT` reach the other server for replicated and plain
  tables.
- Insert the same block twice into a replicated table, then the same rows with a fresh
  `now64()`, and count.

## Questions worth asking me

- "Here's our table definition. Which column is really the duplicate key, and does `FINAL` do
  anything for it?"
- "Should this lookup be a skip index, a projection or a second table?"
- "How should we convert these tables to replicated engines without losing writes?"
- "What does a `Distributed` table add on top of everything here, and when do we need one?"
- "How do mutations (`ALTER … UPDATE/DELETE`) and lightweight deletes work, and what do they
  cost?"
- "How does `async_insert` change batching and deduplication for a consumer?"
