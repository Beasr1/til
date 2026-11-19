# PostgreSQL in Production — A Course

A course on the part of PostgreSQL you meet once a table is big and busy: why an index
does or doesn't answer a query, how to change a schema without stopping traffic, and
what your migration tool does and doesn't remember.

**This is reference learning material.** Everything here is general PostgreSQL, checked
against the PostgreSQL manual and, where the manual is silent or simplified, reproduced
on PostgreSQL 17 in a throwaway container, with the result shown. The motivating
failures are ordinary ones: a search box backed by an index that got slower every month,
a duplicate event reported as an error, a migration re-run that "succeeded" over a broken
index, and a fix added to an old migration file that never reached production.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure that forced each rule comes before the rule.
- Every chapter ends with **Check yourself** questions. Answers are in
  [14-exercises.md](14-exercises.md).
- Every behavioural claim is quoted from the manual or reproduced, and I say which.

## The one thing to understand first

> **PostgreSQL only does what it can prove is correct, using only what it can see at
> the time.** An index is used only when the planner can prove it gives the same answer.
> A filter moves only when moving it can't change the result. A partial index serves
> only queries whose text implies its predicate. A migration runner applies only what
> its bookkeeping table says is missing.

Most surprises in this course are a case of that sentence: something you know is true
(the value is always `'FAILED'`, the latest row is the one you want, the old migration
was fixed) that the system can't see.

## Reference implementations

| Source | What it settles |
|---|---|
| [PostgreSQL manual — Index types](https://www.postgresql.org/docs/current/indexes-types.html) | When a btree serves `LIKE` and `ILIKE`; what GIN is |
| [Operator classes](https://www.postgresql.org/docs/current/indexes-opclass.html) | `text_pattern_ops`, collation, and what it can't serve |
| [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html) | How trigrams are extracted and which operators they serve |
| [Partial indexes](https://www.postgresql.org/docs/current/indexes-partial.html) | Predicate matching at plan time, and parameters |
| [SELECT](https://www.postgresql.org/docs/current/sql-select.html) | `DISTINCT ON`, `LIMIT`/`OFFSET` and unique ordering |
| [INSERT](https://www.postgresql.org/docs/current/sql-insert.html) | `ON CONFLICT`, inference, `RETURNING` |
| [PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html) | Custom versus generic plans, parameter type inference |
| [Operator type resolution](https://www.postgresql.org/docs/current/typeconv-oper.html) | How an `unknown` parameter gets its type |
| [Frontend/Backend protocol: message flow](https://www.postgresql.org/docs/current/protocol-flow.html) | Simple versus extended query; named and unnamed statements |
| [pgx `QueryExecMode`](https://pkg.go.dev/github.com/jackc/pgx/v5#QueryExecMode) | What each driver mode sends, and which survive a pooler |
| [Date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html) / [Aggregate functions](https://www.postgresql.org/docs/current/functions-aggregate.html) | Intervals as months, days and microseconds; aggregates over no rows |
| [Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html) | Lock modes and what conflicts with what |
| [Advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS) | `pg_advisory_xact_lock` and friends, and their two key spaces |
| [Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html) | What Read Committed re-checks on a concurrent update; when a Repeatable Read snapshot is taken |
| [Current date/time](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-CURRENT) | `now()` is the transaction start time |
| [Range constraints](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT) / [Enumerated types](https://www.postgresql.org/docs/current/datatype-enum.html) | Exclusion constraints; why each enum is its own type |
| [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html) / [DROP INDEX](https://www.postgresql.org/docs/current/sql-dropindex.html) | `CONCURRENTLY`, its restrictions, INVALID indexes |
| [Client connection defaults](https://www.postgresql.org/docs/current/runtime-config-client.html) | `statement_timeout`, `lock_timeout` |
| [node-postgres: Transactions](https://node-postgres.com/features/transactions) / [Pool](https://node-postgres.com/apis/pool) | One client per transaction; release; pool timeouts and defaults |
| [pgx `Tx`](https://pkg.go.dev/github.com/jackc/pgx/v5#Tx) / [pgxpool](https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool) | Deferred rollback; pool size default |
| [deadpool `RecyclingMethod`](https://docs.rs/deadpool-postgres/latest/deadpool_postgres/enum.RecyclingMethod.html) / [`Timeouts`](https://docs.rs/deadpool/latest/deadpool/managed/struct.Timeouts.html) | What a pool checks on recycle; timeouts that default to none |
| [PgBouncer features](https://www.pgbouncer.org/features.html) | Pooling modes and what breaks in each |
| [Cumulative statistics](https://www.postgresql.org/docs/current/monitoring-stats.html) | `pg_stat_user_indexes`, progress views |
| [MVCC introduction](https://www.postgresql.org/docs/current/mvcc-intro.html) / [WAL](https://www.postgresql.org/docs/current/wal-intro.html) / [HOT](https://www.postgresql.org/docs/current/storage-hot.html) | Row versions, logging, heap-only updates |
| [Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html) / [Planner statistics](https://www.postgresql.org/docs/current/planner-stats.html) | Reading plans; where estimates come from |
| [ALTER TYPE](https://www.postgresql.org/docs/current/sql-altertype.html) | Enum values can be added and renamed, never dropped |
| [golang-migrate FAQ](https://github.com/golang-migrate/migrate/blob/master/FAQ.md) / [goose README](https://github.com/pressly/goose) | Two runners' bookkeeping, failure and versioning models |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What PostgreSQL does with a query](01-what-postgres-does-with-a-query.md) | ⭐ Explain heap pages, TIDs, MVCC row versions, HOT, WAL and planner statistics, and read an `EXPLAIN (ANALYZE, BUFFERS)` plan node by node. **Prerequisite for everything else** |

### Part 1 — Reading data

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [How an index answers a query](02-how-an-index-answers-a-query.md) | ⭐ Pick an index type and operator class for a search, and explain why trigrams on identifiers get slower as the table grows |
| 3 | [Is anything using this index?](03-is-anything-using-this-index.md) | Collect evidence before dropping an index, and know the four ways the counter misleads |
| 4 | [The latest row per group](04-the-latest-row-per-group.md) | Explain why the planner won't use your index above a `DISTINCT ON`, decide the semantics that let it, and say what a count means and what it returns on no rows |
| 5 | [Pagination that doesn't skip or repeat](05-pagination.md) | Write keyset pagination with a correct tiebreaker |

### Part 2 — Writing data

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Upserts and duplicate events](06-upserts-and-duplicate-events.md) | Make an event handler idempotent without turning duplicates into errors |
| 7 | [History rows and derived state](07-history-rows-and-derived-state.md) | Choose where derived state like "stale" lives, and say which timestamp is safe to compare |
| 8 | [Parameters, plans and types](08-parameters-plans-and-types.md) | Explain why a query changes plan on its sixth run, read a type error that points at the wrong operator, pick a driver mode for a transaction-mode pooler, and write date ranges that survive time zones and short months |
| 11 | [Invariants a constraint can't hold](11-invariants-a-constraint-cant-hold.md) | ⭐ Say when a unique constraint actually arbitrates a race, serialise check-then-write per business key with an advisory lock, and write state transitions as compare-and-set updates |
| 12 | [Transactions and the connection pool](12-transactions-and-the-connection-pool.md) | Keep a transaction on one connection, put acquisition timeouts in the pool rather than around it, avoid the second-connection deadlock, and size pools from the server's connection budget |

### Part 3 — Changing the schema

| # | File | After this you can… |
|---|------|---------------------|
| 9 | [Changing a live table](09-changing-a-live-table.md) | ⭐ Say which lock a schema change takes, prevent a lock-queue outage, and recover from a failed concurrent build |
| 10 | [Migration runners and what they remember](10-migration-runners.md) | Explain what your runner can and can't detect, switch runners without re-running history, and know which downs are lossy |

### Reference

| # | File | |
|---|------|---|
| 13 | [Glossary](13-glossary.md) | Terms, each linked to its chapter |
| 14 | [Exercises & answers](14-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. After that, Parts 1 and 3 are independent of each other. Read
chapter 9 before chapter 10, because 10 relies on INVALID indexes and transaction blocks.
Chapter 7 builds on chapters 4 and 6. Chapters 11 and 12 belong with the writing chapters,
after chapter 8: 11 builds on chapter 6, and 12 on 11. They were written later, so they have
later numbers. Not yet written, and belonging here: vacuum and bloat in depth,
partitioning, replication and failover, and running a server-side pooler. The questions at
the end of file 14 are the honest list.

## If you're short on time

- **New to PostgreSQL internals:** chapter 1, all of it. Everything else assumes it.
- **10 minutes:** chapter 9, the lock table and "The lock queue". That's the one that
  takes sites down.
- **A search is slow:** chapter 2, "Why trigrams are bad at digits", then chapter 4's
  push-down section.
- **About to run a migration on production:** chapter 9 top to bottom, then chapter 10's
  rules 1 and 2.
- **Switching migration tools:** chapter 10, "Switching tools: baselining".
- **An event handler logs errors on duplicates:** chapter 6, edge 1.
- **Duplicate rows despite a unique index, or a terminal state overwritten:** chapter 11,
  §11.2, §11.3 and §11.5.
- **The pool is exhausted and connections sit `idle in transaction`:** chapter 12, §12.1,
  §12.3 and §12.5, in that order.
- **Designing a "current status" from event history:** chapter 7, Design 3 and the timestamp table.

## The one-paragraph summary of everything

A table is an unordered heap of 8 kB pages, and an index is a separate structure mapping
keys to row addresses, written on every insert. An update writes a new row version (MVCC),
so readers see snapshots without blocking writers, dead versions wait for vacuum, and every
index needs a new entry unless the update is HOT. Every change goes to the WAL first, which
is also what replicas replay. The planner picks a plan from statistics refreshed by `ANALYZE`,
and `EXPLAIN (ANALYZE, BUFFERS)` shows the plan and the pages it touched. A btree is a sorted list, so it answers equality, ranges and anchored prefixes. Under a
non-C collation it needs `text_pattern_ops` before it will serve `LIKE 'x%'`, and it
serves `ILIKE` too when the prefix has no letters. A trigram GIN index answers *contains*
queries, but each trigram's posting list grows with the table, and on a ten-symbol
alphabet every list is long. It also can't be scoped to a tenant, so for identifiers the
better move is often to offer prefix search on a tenant-led btree. Before dropping an
index, reset its counter by the index's own OID, wait a representative window, and check
every server that takes reads, because replicas count their own scans. The planner won't
push a filter below a `DISTINCT ON`, because doing so changes the question from "latest
row matches" to "any row matches". Choose the semantics deliberately and write the filter
where it belongs. Decide what a count means (events or distinct entities, point in time or
cumulative) before writing it, and remember that `sum` and `array_agg` over no rows return
`NULL`, not 0 or `{}`. Paging needs a unique order: add the primary key as a tiebreaker and
prefer keyset pagination, keyed on the tiebreaker too. `ON CONFLICT DO NOTHING RETURNING`
returns zero rows on a duplicate, which single-row driver helpers report as an error;
`NULL` keys never conflict; and a partial unique index must be named with its predicate.
Derived state such as "stale" belongs neither as a flag on history rows nor recomputed by
every reader. Materialise it, resolved by a writer that compares event times rather than
arrival order and backed by a recompute job, and never trust a trigger-maintained `updated_at`
you haven't checked exists.
A unique constraint arbitrates concurrent inserts only when every racer computes the same
key, so a key containing `now()`, the transaction's start time, never collides. A rule no
constraint can hold, such as "none in the last 30 days" or "at most K per key", needs its
check and write serialised per business key with a transaction-level advisory lock, at Read
Committed, because a Repeatable Read snapshot is taken before the lock is granted. The code
still tolerates `23505`, and the constraint stays the authority. A state transition belongs
in the `UPDATE`'s `WHERE`, acted on by its rows-affected count, because Read Committed
re-checks the `WHERE` against a concurrent winner.
A transaction is a property of one connection, so every statement in it goes through one
checked-out connection, with rollback deferred on every path; a pool's convenience method
scatters `BEGIN` and `COMMIT` and leaks open transactions back into the pool. Acquisition
timeouts belong to the pool, because a raced timer leaves the acquisition running, and a
transaction must never ask the pool for a second connection while holding the first. Pool
sizes add up across every process against one `max_connections`.
Drivers that cache prepared statements switch to generic plans after five executions,
and generic plans can't use a partial index for a bound value. Named prepared statements
live on one server connection, so behind a transaction-mode pooler use unnamed
extended-protocol execution (pgx's `Exec` mode) rather than the simple protocol, and move
services onto a pooler one at a time. An untyped parameter takes
its type from the first operator it meets, which is why `$1 + interval` fails at the next
comparison. Date ranges are half-open, with both boundaries computed in the user's zone, and
calendar units stay symbolic: `interval '1 month'` is not 30 days when added to a date, even
though the two compare equal. Every schema change takes a lock. Plain `CREATE INDEX` blocks writes, plain
`DROP INDEX` and most `ALTER TABLE` block everything, and a waiting lock queues every
later query behind it, so migrations need `lock_timeout`. `CREATE INDEX CONCURRENTLY`
avoids blocking but can't run in any transaction block, implicit ones included. A failure
or timeout leaves an INVALID index that `IF NOT EXISTS` will happily skip on re-run.
Finally, a migration runner compares its bookkeeping table with filenames and nothing
else. Editing an applied migration changes nothing on existing databases. A
non-transactional migration can fail half-way with no record. golang-migrate silently
skips a migration numbered below the current version, while goose refuses by default.
A down migration that drops or moves data is a best-effort copy, not an undo. Switching tools
needs a baseline derived from the old tool's records, not from a guess.

## How to use me

Ask me things like:

- "Here's an `EXPLAIN (ANALYZE, BUFFERS)`. Why isn't my index used?"
- "Which lock does this migration take, and what would it block on production?"
- "Is this upsert idempotent under redelivery and reordering?"
- "Our migrations run on startup. Walk me through moving them into the deploy."
- "Design the evidence-gathering to drop this index safely."
- "Should this be a partial index, given how our driver prepares statements?"
- "Two requests for the same customer both got through this check. Which lock, or which
  `WHERE`, would have stopped the second?"
- "Here are our services, their replica and worker counts, and pool settings. Do they fit
  the server's connection budget during a rolling deploy?"
