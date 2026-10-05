# 9. Changing a live table

## The problem

A schema change that takes two seconds on your laptop takes forty minutes on
production. During those forty minutes, depending on which lock it took, either nothing
happens to your users, or nobody can save anything, or nobody can even read. The
statement is the same in every case. What changes is the lock, and whether anything is
queued behind it. So the first skill in changing a live database is knowing which lock
each statement takes and what that lock blocks.

## Locks, briefly

Every statement takes a table-level lock in some mode, and modes conflict with each
other according to a fixed table
([§13.3](https://www.postgresql.org/docs/current/explicit-locking.html)). Ordinary
`SELECT` takes `ACCESS SHARE`. Ordinary `INSERT`/`UPDATE`/`DELETE` take `ROW EXCLUSIVE`.
The ones that matter for schema changes:

| Lock mode | Taken by (examples) | Blocks reads? | Blocks writes? |
|---|---|---|---|
| `SHARE UPDATE EXCLUSIVE` | `CREATE INDEX CONCURRENTLY`, `VACUUM`, `ANALYZE`, `VALIDATE CONSTRAINT` | No | No |
| `SHARE` | `CREATE INDEX` (without `CONCURRENTLY`) | No | **Yes** |
| `ACCESS EXCLUSIVE` | `DROP INDEX`, `DROP TABLE`, `TRUNCATE`, most `ALTER TABLE` forms | **Yes** | **Yes** |

> **Teacher's aside.** A widely repeated claim is that a plain `CREATE INDEX` "locks the
> table" and blocks everything. It blocks **writes, not reads**. The manual
> ([CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)):
> "Other transactions can still read the table, but if they try to insert, update, or
> delete rows in the table they will block until the index build is finished." For a
> write-heavy table that's an outage anyway. But get the mechanism right, because the
> statement that blocks reads too, `ACCESS EXCLUSIVE`, is a different and worse one, and
> plain `DROP INDEX` is in that row.

## The lock queue: why a fast statement can stop everything

This is the mechanism that turns a one-millisecond `ALTER TABLE` into an outage.

Lock requests queue. If a statement wants `ACCESS EXCLUSIVE` and something already
holds a conflicting lock, it waits. While it waits, **every later request that conflicts
with it queues behind it**, including plain `SELECT`s that would have been perfectly
compatible with whatever was holding the table.

Reproduced on PostgreSQL 17 with three sessions:

```
session A   BEGIN; SELECT count(*) FROM s; SELECT pg_sleep(20);   -- holds ACCESS SHARE
session B   ALTER TABLE s ADD COLUMN z int;                       -- waits: Lock / relation
session C   SELECT count(*) FROM s;                               -- waits behind B
            ERROR:  canceling statement due to statement timeout
```

Session C's read didn't conflict with A. It conflicted with B's *queued* request. So one
long-running analytics query plus one "instant" migration equals every request to that
table hanging until the analytics query finishes.

The defence is `lock_timeout`, set on the migration's session only
([§19.11](https://www.postgresql.org/docs/current/runtime-config-client.html)):

> Abort any statement that waits longer than the specified amount of time while
> attempting to acquire a lock on a table, index, row, or other database object.

```sql
SET lock_timeout = '3s';
ALTER TABLE s ADD COLUMN z int;   -- fails fast instead of queueing everyone
```

A migration that fails because it couldn't get its lock is safe to retry. An outage is
not.

## Building an index without blocking writes

`CREATE INDEX CONCURRENTLY` takes `SHARE UPDATE EXCLUSIVE`, which blocks neither reads
nor writes. It pays for that in several ways, and each one is a trap.

### It can't run inside a transaction block

The manual: "a regular `CREATE INDEX` command can be performed within a transaction
block, but `CREATE INDEX CONCURRENTLY` cannot." The same goes for `DROP INDEX
CONCURRENTLY`.

That includes **implicit** transactions. When several statements are sent as one simple
query string, PostgreSQL runs them as a single transaction
([§54.2.2.1](https://www.postgresql.org/docs/current/protocol-flow.html)):

```
$ psql -c "SELECT 1; CREATE INDEX CONCURRENTLY t_cc ON t(org);"
ERROR:  CREATE INDEX CONCURRENTLY cannot run inside a transaction block

$ psql -c "CREATE INDEX CONCURRENTLY t_cc ON t(org);"
CREATE INDEX
```

So a migration tool that sends a whole file as one string, or wraps each file in
`BEGIN … COMMIT`, can't run a concurrent build unless that statement is alone in its
file or the tool is told not to wrap it. Chapter 10 covers how tools expose this.

### It waits for every transaction that has touched the table

The build happens in phases: "the index is actually entered as an "invalid" index into
the system catalogs in one transaction, then two table scans occur in two more
transactions. Before each table scan, the index build must wait for existing
transactions that have modified the table to terminate." A connection left idle inside a
transaction stalls the build indefinitely. `idle_in_transaction_session_timeout` exists
for exactly this.

### If it fails, it leaves an INVALID index behind

> If a problem arises while scanning the table, such as a deadlock or a uniqueness
> violation in a unique index, the `CREATE INDEX` command will fail but leave behind an
> "invalid" index. This index will be ignored for querying purposes because it might be
> incomplete; however it will still consume update overhead.

Cancellation counts as a failure. A server-side `statement_timeout` will cancel a long
build part-way:

```
$ PGOPTIONS='-c statement_timeout=300ms' psql -c "CREATE INDEX CONCURRENTLY t_slow ON t USING gin (...);"
ERROR:  canceling statement due to statement timeout

SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
 t_slow | f
```

So a migration connection for index builds needs `statement_timeout` disabled for that
session, for example `options=-c statement_timeout=0` in the connection string.

> ⚠️ **The trap: `IF NOT EXISTS` skips a broken index.** It's natural to write every
> build as `CREATE INDEX CONCURRENTLY IF NOT EXISTS` so the migration is safe to re-run.
> But the invalid index *exists*. Re-running after a failed build does this:
>
> ```
> CREATE INDEX CONCURRENTLY IF NOT EXISTS t_slow ON t USING gin (...);
> NOTICE:  relation "t_slow" already exists, skipping
> CREATE INDEX
> ```
>
> The command reports success, the migration tool records the migration as applied,
> and the index stays INVALID for good: never used by a query, still maintained on every
> write. After any interrupted build, find and drop invalid indexes *before* re-running:
>
> ```sql
> SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
> DROP INDEX CONCURRENTLY IF EXISTS <name>;
> ```

### One build per migration

Only one concurrent build can run on a table at a time, and a failure part-way through a
file of six builds leaves you working out which ones finished. One index per migration
file means the runner's own bookkeeping tells you where it stopped.

### Watch it

```sql
SELECT phase, blocks_done, blocks_total,
       round(100.0 * blocks_done / nullif(blocks_total, 0), 1) AS pct
  FROM pg_stat_progress_create_index;
```

## Other changes that don't need a long lock

| Change | Safe form | Why |
|---|---|---|
| Add a column with a default | `ADD COLUMN … DEFAULT <constant>` | A non-volatile default is stored in metadata, so the table isn't rewritten. A volatile default such as `clock_timestamp()` forces a rewrite |
| Add a constraint | `ADD CONSTRAINT … NOT VALID`, then `VALIDATE CONSTRAINT` | The first step skips the scan; validation takes only `SHARE UPDATE EXCLUSIVE` |
| Drop an index | `DROP INDEX CONCURRENTLY` | Plain `DROP INDEX` takes `ACCESS EXCLUSIVE` |
| Change a column's type | Add a new column, backfill in batches, switch reads, drop the old one | An in-place type change usually rewrites the table under `ACCESS EXCLUSIVE` |

Each of these still takes an `ACCESS EXCLUSIVE` lock briefly in most cases, so
`lock_timeout` still applies.

## Check yourself

1. A plain `CREATE INDEX` on a busy table runs for ten minutes. What can users do during
   it and what can't they? Now answer the same for a plain `DROP INDEX` that runs for one
   second.
2. A migration does a sub-millisecond `ALTER TABLE ADD COLUMN`. The site goes down for
   four minutes. Give the most likely mechanism and the one setting that would have
   prevented it.
3. Why does `psql -c "SET statement_timeout=0; CREATE INDEX CONCURRENTLY …"` fail, and
   what's the difference from passing the setting through `PGOPTIONS`?
4. A concurrent build fails at 80%. Someone re-runs the migration and it succeeds in
   0.1 seconds. What state is the database in, and how do you prove it?
5. A concurrent build has been at `phase = waiting for old snapshots` for an hour. What
   do you look for, and where?
