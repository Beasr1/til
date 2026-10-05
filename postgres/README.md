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
  [12-exercises.md](12-exercises.md).
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
| [Explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html) | Lock modes and what conflicts with what |
| [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html) / [DROP INDEX](https://www.postgresql.org/docs/current/sql-dropindex.html) | `CONCURRENTLY`, its restrictions, INVALID indexes |
| [Client connection defaults](https://www.postgresql.org/docs/current/runtime-config-client.html) | `statement_timeout`, `lock_timeout` |
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
| 4 | [The latest row per group](04-the-latest-row-per-group.md) | Explain why the planner won't use your index above a `DISTINCT ON`, and decide the semantics that let it |
| 5 | [Pagination that doesn't skip or repeat](05-pagination.md) | Write keyset pagination with a correct tiebreaker |

### Part 2 — Writing data

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Upserts and duplicate events](06-upserts-and-duplicate-events.md) | Make an event handler idempotent without turning duplicates into errors |
| 7 | [History rows and derived state](07-history-rows-and-derived-state.md) | Choose where derived state like "stale" lives, and say which timestamp is safe to compare |
| 8 | [Parameters, plans and types](08-parameters-plans-and-types.md) | Explain why a query changes plan on its sixth run, and read a type error that points at the wrong operator |

### Part 3 — Changing the schema

| # | File | After this you can… |
|---|------|---------------------|
| 9 | [Changing a live table](09-changing-a-live-table.md) | ⭐ Say which lock a schema change takes, prevent a lock-queue outage, and recover from a failed concurrent build |
| 10 | [Migration runners and what they remember](10-migration-runners.md) | Explain what your runner can and can't detect, switch runners without re-running history, and know which downs are lossy |

### Reference

| # | File | |
|---|------|---|
| 11 | [Glossary](11-glossary.md) | Terms, each linked to its chapter |
| 12 | [Exercises & answers](12-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. After that, Parts 1 and 3 are independent of each other. Read
chapter 9 before chapter 10, because 10 relies on INVALID indexes and transaction blocks.
Chapter 7 builds on chapters 4 and 6. Not yet written, and belonging here: vacuum and bloat
in depth, partitioning, replication and failover, and connection pooling. The questions at
the end of file 12 are the honest list.

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
where it belongs. Paging needs a unique order: add the primary key as a tiebreaker and
prefer keyset pagination, keyed on the tiebreaker too. `ON CONFLICT DO NOTHING RETURNING`
returns zero rows on a duplicate, which single-row driver helpers report as an error;
`NULL` keys never conflict; and a partial unique index must be named with its predicate.
Derived state such as "stale" belongs neither as a flag on history rows nor recomputed by
every reader. Materialise it, resolved by a writer that compares event times rather than
arrival order and backed by a recompute job, and never trust a trigger-maintained `updated_at`
you haven't checked exists.
Drivers that cache prepared statements switch to generic plans after five executions,
and generic plans can't use a partial index for a bound value. An untyped parameter takes
its type from the first operator it meets, which is why `$1 + interval` fails at the next
comparison. Every schema change takes a lock. Plain `CREATE INDEX` blocks writes, plain
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
