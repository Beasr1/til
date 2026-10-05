# 11. Glossary

Reference, not reading. Each entry points at the chapter that explains it.

| Term | Meaning | Where |
|---|---|---|
| **ACCESS EXCLUSIVE** | The strongest table lock. Conflicts with every other mode, including plain `SELECT`. Taken by `DROP INDEX`, `DROP TABLE`, `TRUNCATE` and most `ALTER TABLE` forms | [09](09-changing-a-live-table.md) |
| **Autovacuum** | The background process that runs `VACUUM` and `ANALYZE` automatically | [01](01-what-postgres-does-with-a-query.md) |
| **Baseline** | Recording already-applied migrations in a new runner's bookkeeping table so it doesn't try to re-run them | [10](10-migration-runners.md) |
| **BitmapOr** | A plan node that scans several indexes and merges their row sets, letting one table serve an `OR` across indexed columns | [04](04-the-latest-row-per-group.md) |
| **Bloat** | Space taken by dead tuples that haven't been reclaimed, so the same live data spans more pages | [01](01-what-postgres-does-with-a-query.md) |
| **btree** | The default index type. Keys kept in sorted order, so it answers equality, ranges and anchored prefixes | [02](02-how-an-index-answers-a-query.md) |
| **Collation** | The rules for ordering text. Non-C collations order by language rules, which stops a default btree from serving `LIKE 'x%'` | [02](02-how-an-index-answers-a-query.md) |
| **Custom plan** | A plan made for one execution of a prepared statement, using the actual parameter values | [08](08-parameters-plans-and-types.md) |
| **Dead tuple** | A row version no transaction can see any more, left behind by an `UPDATE` or `DELETE` until vacuum removes it | [01](01-what-postgres-does-with-a-query.md) |
| **Derived state** | A fact computed from history rows, such as "stale since the last verification", rather than recorded directly | [07](07-history-rows-and-derived-state.md) |
| **Dirty flag** | golang-migrate's marker that the last migration started and didn't finish | [10](10-migration-runners.md) |
| **`DISTINCT ON`** | PostgreSQL's "first row per group, in this order". Must match the leftmost `ORDER BY` expressions | [04](04-the-latest-row-per-group.md) |
| **Down migration** | The reverse script next to a migration. Exact for schema-only changes, lossy for anything that drops or moves data | [10](10-migration-runners.md) |
| **Expand/contract** | Changing a schema in separately deployed steps: add, dual-use, migrate data, switch, remove | [10](10-migration-runners.md) |
| **Expression index** | An index on the result of an expression, such as a JSON path. Used only by queries containing the identical expression | [02](02-how-an-index-answers-a-query.md) |
| **Generic plan** | One plan reused for every execution of a prepared statement, made without seeing parameter values | [08](08-parameters-plans-and-types.md) |
| **GIN** | Generalised Inverted Index. One entry per component value (array element, trigram, JSON key) pointing at every row containing it | [02](02-how-an-index-answers-a-query.md) |
| **Heap** | A table's main storage: unordered 8 kB pages of row versions | [01](01-what-postgres-does-with-a-query.md) |
| **HOT update** | Heap-only tuple update: an update that touches no indexed column and fits on the same page, so no index needs a new entry | [01](01-what-postgres-does-with-a-query.md) |
| **Idempotent** | Doing it twice has the same effect as doing it once | [06](06-upserts-and-duplicate-events.md) |
| **Inference (ON CONFLICT)** | How PostgreSQL works out which unique index a conflict target refers to. A partial index needs its predicate repeated | [06](06-upserts-and-duplicate-events.md) |
| **INVALID index** | An index left behind by a failed concurrent build. Ignored by queries, still updated on every write | [09](09-changing-a-live-table.md) |
| **Keyset pagination** | Paging by "after the last row's sort key" rather than by offset | [05](05-pagination.md) |
| **Lock queue** | Waiting lock requests are queued, and later requests that conflict with a *waiting* one queue behind it | [09](09-changing-a-live-table.md) |
| **`lock_timeout`** | Aborts a statement that waits too long for a lock. The guard rail for migrations | [09](09-changing-a-live-table.md) |
| **MVCC** | Multi-version concurrency control. Updates write new row versions so each statement reads a consistent snapshot without blocking writers | [01](01-what-postgres-does-with-a-query.md) |
| **`now()`** | The start time of the current transaction, the same for every call within it. `clock_timestamp()` is the actual time | [07](07-history-rows-and-derived-state.md) |
| **Operator class** | The comparison an index is built with, e.g. `text_pattern_ops` for character-by-character ordering | [02](02-how-an-index-answers-a-query.md) |
| **Partial index** | An index covering only rows that match a predicate. Usable only when the planner can prove the query implies it | [08](08-parameters-plans-and-types.md) |
| **Planner statistics** | Row counts, page counts and per-column distributions (`pg_class`, `pg_stats`) the planner uses to estimate costs. Refreshed by `ANALYZE` | [01](01-what-postgres-does-with-a-query.md) |
| **Posting list** | In an inverted index, the list of rows containing one component value | [02](02-how-an-index-answers-a-query.md) |
| **Push-down** | Moving a filter closer to the base table. Only allowed when it can't change the result | [04](04-the-latest-row-per-group.md) |
| **Row comparison** | `(a, b) < (x, y)`, compared lexicographically. The basis of keyset pagination | [05](05-pagination.md) |
| **Sequential scan** | Reading every page of a table | [01](01-what-postgres-does-with-a-query.md) |
| **SHARE** | The lock taken by plain `CREATE INDEX`. Blocks writes, allows reads | [09](09-changing-a-live-table.md) |
| **SHARE UPDATE EXCLUSIVE** | The lock taken by `CREATE INDEX CONCURRENTLY`, `VACUUM`, `ANALYZE`. Blocks neither reads nor writes | [09](09-changing-a-live-table.md) |
| **`statement_timeout`** | Aborts any statement that runs too long. Will cancel a concurrent index build and leave it INVALID | [09](09-changing-a-live-table.md) |
| **TID / `ctid`** | A row version's physical address: page number and slot on that page | [01](01-what-postgres-does-with-a-query.md) |
| **TOAST** | Out-of-line storage for large values. Reading one key of a TOASTed JSON value means fetching and decompressing the whole value | [04](04-the-latest-row-per-group.md) |
| **Trigram** | Three consecutive characters. `pg_trgm` indexes them to support contains and fuzzy search | [02](02-how-an-index-answers-a-query.md) |
| **VACUUM** | Reclaims space from dead tuples and updates visibility information | [01](01-what-postgres-does-with-a-query.md) |
| **WAL** | Write-ahead log. Every change is logged before it reaches the data files; crash recovery and replicas replay it | [01](01-what-postgres-does-with-a-query.md) |
