# 12. Exercises

Worked answers to every **Check yourself** question, then things to try and questions
worth asking me.

---

## File 01 — What PostgreSQL does with a query

**1. 10 million rows, no index, `id = 42`.**

A sequential scan reads every page of the heap, all of them, and checks each row. With
roughly 8 kB pages that's every page the table occupies, which `pg_class.relpages` will
tell you. It can't stop at the first match either, unless the query has a `LIMIT`, because
nothing says `id` is unique without a unique index. "The rows are stored in id order" isn't
a way out, because the heap has no order. Rows go wherever there's free space, and updates
move them (as the `ctid` example showed), so the planner can't assume any order without an
index.

**2. Everything an `UPDATE` of one column might write.**

A new row version in the heap. The old version's header is marked as replaced. WAL records
for both changes, written before the data files. And a new entry in **every** index on the
table, unless the update is HOT. If the changed column is indexed, HOT is impossible, so
every index gets a new entry, not just the one on that column. If it isn't indexed and the
page has room, HOT applies and no index is touched.

**3. An hour-long report transaction on a busy table.**

The table grows. Every update leaves a dead version behind, and vacuum can only remove dead
versions that no open transaction might still need to see. The report's snapshot is an hour
old, so every version replaced in the last hour might still be visible to it, and none of
them can be reclaimed. When the report ends, vacuum can clean up, but the table's files don't
shrink back on their own. The space is reused for new rows instead.

**4. Estimated `rows=1`, actual `rows=48000`.**

The planner's statistics are wrong for this condition: stale (no `ANALYZE` since the data
changed), too coarse for a skewed distribution, or the condition involves several columns
whose correlation the planner can't see. A plan chosen for one row (a nested loop, say) can be
terrible for 48,000. Run `ANALYZE` on the table and re-check. If it's still wrong, look at
`pg_stats` for the column, and consider a higher statistics target or extended statistics.

**5. 4 ms with 40 buffers versus 6 ms with 9,000 buffers.**

The 40-buffer plan. Buffers count pages touched, which is a property of the plan. Time also
depends on cache state and on whatever else the machine was doing. The 9,000-buffer plan may
have run with every page already in memory, so it looked nearly as fast. On a cold cache, or
under load, it reads up to 9,000 pages and the gap becomes enormous.

**6. Why "readers never block writers" doesn't mean a `SELECT` never waits.**

MVCC removes conflicts over *row data*. A `SELECT` still takes an `ACCESS SHARE` lock on the
table, which conflicts with `ACCESS EXCLUSIVE`. So a `SELECT` waits behind a `DROP INDEX`, an
`ALTER TABLE`, or a `TRUNCATE`, including one that's only *queued* for its lock (chapter 9's
lock queue).

---

## File 02 — How an index answers a query

**1. Prefix `LIKE` does a sequential scan despite a plain btree.**

The database uses a non-C collation, so the default btree is sorted by language rules,
and "everything starting with `abc`" isn't guaranteed to be one contiguous range in it.
PostgreSQL won't rewrite the `LIKE` as a range it can't prove correct. The two fixes:
build the index with `text_pattern_ops`, which sorts character by character, or use the
C collation for that column or index (`COLLATE "C"`), which does the same thing by
another route. If you also need `<`/`>` or `ORDER BY` under the language collation, keep
a default btree as well.

**2. Why `ILIKE '100%'` can use a btree and `ILIKE 'abc%'` can't.**

Case-insensitive matching means `a` could be `a` or `A`, and those sort to different
places, so a prefix starting with a letter doesn't map to one range. Digits have no case:
`100` can only be `100`, so the prefix is one range, exactly as for `LIKE`. The manual
states it as "only if the pattern starts with non-alphabetic characters". The planner
takes the longest prefix of case-insensitive-safe characters. For `ILIKE '100a%'` it
would use `100` as the range and recheck the rest.

**3. Which can use a `text_pattern_ops` index on `name`?**

`name = 'x'`: yes. Equality works under any total order. `name LIKE 'x%'`: yes, that's
what it's for. `name > 'x'`: no. The manual says ordinary `<`, `<=`, `>`, `>=` "cannot
use" these operator classes, because they mean the collation's order and the index is in
byte order. `ORDER BY name`: no, for the same reason. The index's order isn't the order
the query asks for.

**4. Trigrams on phone numbers versus free text.**

A posting list's length is the number of rows containing that trigram. Phone numbers
draw from ten symbols, so there are about a thousand possible trigrams and every one of
them shows up in a large, roughly fixed share of rows. As the table grows, every posting
list grows in proportion, and so does every query, which reads a dozen of them. Free text
draws from a much larger alphabet with a skewed distribution: most trigrams are rare and
have short lists, and a query for a distinctive word touches mostly rare ones. Both grow
with the table, but text queries mostly read short lists. Number queries only ever read
long ones.

**5. GIN trigram index plus a btree on `tenant_id`, query filters on both.**

Plan A: bitmap scan on the trigram index for the pattern across the whole table, then
recheck and apply the tenant filter on the heap. The cost is the posting-list reads for
every tenant plus heap visits for candidates in other tenants that get thrown away.
Plan B: scan the btree for the tenant's rows, fetch each one, and evaluate the pattern
on it. The cost is proportional to the tenant's size, and each evaluation extracts the
JSON value, possibly detoasting it. A third option, a BitmapAnd of both indexes, still
pays the full trigram read. None of them is "look only inside this tenant's slice of the
trigram index", because that slice doesn't exist. That's the structural problem, and it's
what a btree with the tenant as its leading column fixes.

**6. Two JSON paths, an index on only the new one.**

Queries written against the new path find only documents written by the new version.
Old documents have nothing at that path, so the expression is `NULL` and they never
match: no error, just missing results. You notice when someone searches for a record
they know exists, or, better, when a monitoring query compares "documents with a value
at either path" against "documents found by the search path" per tenant and sees tenants
with zero coverage. The fix is a query branch and an index per path until old documents
are rewritten, or promoting the field to a column filled from either path.

---

## File 03 — Is anything using this index?

**1. Three situations where `idx_scan = 82` and dropping is still safe.**

(a) All 82 scans happened before a code change months ago, and nothing has used it since.
The counter has never been reset. (b) The scans come from the feature you're migrating
to a replacement index, which doesn't exist yet, so the old index is the only one the
planner can choose. (c) The scans come from a benchmark or a one-off investigation, not
from production traffic. Each one is resolved the same way: reset, wait, re-check.

**2. Zero scans on the primary after a month. Why might the drop still break things?**

Read replicas count their own scans and the primary doesn't see them. A reporting query
that runs only on a replica can depend on the index while the primary shows zero.
Dropping on the primary drops it on every replica through replication. Check
`pg_stat_user_indexes` on every server that takes reads. Also confirm the month covered
everything that runs: quarterly jobs, year-end reports.

**3. One migration that creates B and drops A.**

The runner applies both in one deploy, so there's never a period where B exists, the code
has moved to B, and A's counter can be watched. If something other than the migrated
feature still needs A, you find out from an incident instead of a counter. There's also a
mechanical issue: if A is dropped without `CONCURRENTLY`, the drop takes `ACCESS
EXCLUSIVE` on the table.

**4. Reset one index rather than the whole database, and the table-name trap.**

A database-wide reset wipes every counter that everyone else uses for capacity planning,
vacuum tuning and their own investigations, to answer a question about one index.
Resetting by index OID touches only what you're measuring. If you pass the *table's*
name, the table's own counters are reset and the index's `idx_scan` is untouched. You
then wait a week watching a number that never reset, and draw the wrong conclusion.

**5. Statistics say used, code says not.**

First `pg_stat_statements`, filtered by text that the relevant query would contain, to
find the actual statements and their call counts. That tells you *what* is running.
Then `EXPLAIN` each candidate to confirm which index it chooses, because the statistics
only say that *some* query used the index. Usually the culprit is a caller you didn't
search (another service, a scheduled job, a BI tool) or a query that's assembled
dynamically, so its text never appears in the code in full.

---

## File 04 — The latest row per group

**1. Two rows tie on `created_at`.**

Either one may be returned, and which one can change between executions and plans. The
manual says the first row of each group is "unpredictable unless `ORDER BY` is used to
ensure that the desired row appears first", and a tie means `ORDER BY` didn't settle it.
Add a unique tiebreaker: `ORDER BY customer_id, created_at DESC, id DESC`.

**2. Why the filter can't move inside, without mentioning performance.**

Outside, it means "keep customers whose latest row matches". Inside, it means "look only
at matching rows, then keep each customer's latest of those". A customer whose latest row
doesn't match but whose older row does is excluded by the first and included by the
second. The planner may only make changes that can't alter results, and this one can.

**3. Old phone number, both semantics.**

Under "latest matches": not found, because their latest row has the new number. Under
"any matches": found, and shown with their current details. Support staff almost always
want the second. A caller quoting an old reference expects to be found. That's worth
confirming with whoever owns the screen, because it's a behaviour change, not just a
performance fix.

**4. `doc ->> 'name'` in the dedup, 20 shown out of 50,000.**

The expression is evaluated for every input row of the dedup, 50,000 or more, and for a
large `doc` each evaluation fetches and decompresses the TOASTed value. 49,980 of those
results are thrown away. Restructure: the dedup selects only the grouping and ordering
columns, the page of 20 ids is chosen, and only then is `doc ->> 'name'` fetched, by
joining back to the table for those 20 rows.

**5. `COUNT(*) OVER ()` and `LIMIT`.**

A window function over the whole result can't produce its value until it has seen every
row of that result. So every row in scope is produced, sorted and counted, and the
`LIMIT` only trims the output at the end. Without the window, the executor could stop
after producing 20 rows, given a suitable index order.

**6. One `OR` versus a `UNION`.**

A single `OR` is fine on one table when every arm is indexable. The planner can BitmapOr
the indexes. Split it when an arm isn't indexable (one unindexable arm makes the whole
`OR` a scan), when the data spans several tables combined by `UNION ALL` (the `OR` above
the union has the push-down problem), or when you want each term's plan to be
predictable on its own.

---

## File 05 — Pagination

**1. `DESC` to `ASC` "for consistency".**

It fixes nothing about ties. Tied rows are just as unordered in either direction, so
pages can still repeat or skip them. What it does change is drift under inserts: newest
rows now land on the last page instead of the first, so early pages stop shifting while
someone pages through. The real fix is a unique tiebreaker, plus keyset pagination if
drift matters.

**2. Two `OFFSET` queries disagree about tied rows.**

The planner considers `LIMIT` and `OFFSET` when choosing a plan, and different plans
(an index scan versus a top-N sort, for instance) produce tied rows in different orders.
The manual warns that different `LIMIT`/`OFFSET` values "will give inconsistent results
unless you enforce a predictable result ordering".

**3. Nightly job with `OFFSET` while inserts continue.**

If it orders oldest-first and new rows sort at the end, it may simply process the new
rows too, which may or may not be wanted. If it orders newest-first, every insert pushes
rows down by one, so the next page starts with rows already processed (duplicates). If
rows are deleted or change so that they no longer match, positions shift up and rows are
skipped. "Processed exactly once" needs a stable cursor: keyset on a unique key.

**4. Keyset for `priority ASC, created_at DESC, id DESC`.**

A row comparison compares every component in the same direction, and here the directions
are mixed. Expand it:

```sql
WHERE priority > $p
   OR (priority = $p AND (created_at, id) < ($c, $i))
ORDER BY priority ASC, created_at DESC, id DESC
```

The last two columns share a direction, so they can still use a row comparison. For an
index to serve this well, build it in the same mixed order:
`(priority ASC, created_at DESC, id DESC)`.

**5. What a keyset cursor leaks.**

The last row's sort-key values: a timestamp, an internal id, maybe a score. Clients can
read them, and worse, construct cursors by hand, which couples them to your sort order
and column choice. Encoding the cursor opaquely (base64 of a small structure, sometimes
signed) keeps it an implementation detail you can change, and stops clients from
fabricating positions.

---

## File 06 — Upserts and duplicate events

**1. `DO NOTHING RETURNING id` with `QueryRow().Scan()`, second delivery.**

The insert conflicts and does nothing. `RETURNING` produces zero rows, because only
inserted or updated rows are returned. `Scan` finds no row and returns `pgx.ErrNoRows`.
The repository function returns that as an error. The caller sees a failure on what was
a harmless duplicate. Depending on the code it logs an error, returns a 5xx (inviting
the sender to retry), or sends the event to a dead-letter queue, where it sits looking
like a real failure. The fix is to treat `ErrNoRows` from this statement as "already
recorded".

**2. How `DO UPDATE` to `DO NOTHING` can fix a data bug.**

Event E is processed and stored. Later a newer event F updates the row. Then E is
redelivered, after a consumer restart for example. With `DO UPDATE SET … = EXCLUDED.…`,
the replayed E overwrites F's newer data with E's old data, and bumps `updated_at` so it
looks fresh. With `DO NOTHING`, the replay has no effect. (The more general fix is a
version guard on `DO UPDATE`, but if repeated arrivals are by definition the same event,
`DO NOTHING` is exactly right.)

**3. Old clients send no event key.**

Their key is `NULL`. `NULL`s are distinct in a unique index by default, so their
duplicates never conflict and every one is inserted. That isn't necessarily wrong. It's a
reasonable way to keep legacy callers working while new ones get deduplication, as long
as it's deliberate and documented, and nothing new can send an empty key that gets
converted to `NULL`.

**4. Making the unique index partial.**

Every `ON CONFLICT (event_key, tenant_id)` that doesn't repeat the predicate fails with
"there is no unique or exclusion constraint matching the ON CONFLICT specification". You
find out on the first execution of each such statement, which is good. Make sure that
happens in a test or staging environment before production by exercising every insert
path.

**5. Versions arrive 8 then 7.**

8 inserts (or updates). Then 7 arrives and plain `DO UPDATE` overwrites with 7, so the
table holds the older version. Fix: add `WHERE target.version < EXCLUDED.version` to the
`DO UPDATE`, so a stale arrival is a no-op.

---

## File 07 — History rows and derived state

**1. Three answers from one flag.**

"Is the latest row's flag true?": no, because the new verification's row says `false`. "Is
any row's flag true?": yes, the old rows still say `true`. "Is the flag true on the latest row
*of each verification type*?": it depends on which type the new verification was, so possibly
yes and possibly no. Three reasonable queries, three answers, because the flag records a
fact about now on rows that describe the past.

**2. Trigger-maintained `updated_at`, environments disagree.**

Check that the trigger exists, on both tables, in both environments: `\d table` lists
triggers, or query `pg_trigger` joined to `pg_class`. A trigger added by editing an
already-applied migration exists only on databases created after the edit (chapter 10, rule
1). Then compare `updated_at` with `created_at` for rows known to have been updated. On a
database without the trigger they'll be equal.

**3. Queue lag and `created_at`.**

09:00 upstream data changes. The change event waits in a lagging queue. 09:05 the customer
completes a verification, which reflects the new data, and the verification row gets
`created_at = 09:05`. 09:08 the change event is consumed and its row gets `created_at = 09:08`.
The read-time comparison sees a change at 09:08 after a verification at 09:05, and marks the
customer stale, though the verification already reflected the change. Using the event's
`happened_at = 09:00` gives the right answer.

**4. A resolver without a time guard.**

10:00 a verification completes. Its "verified" event is delayed in a queue. 10:02 upstream
data changes, and a new active change row is inserted. The customer is now genuinely stale.
10:05 the delayed "verified" event is consumed, and the resolver sets `is_active = false` on
every active change for the customer, including the 10:02 one, which happened *after* the
verification. The customer looks current and isn't. The fix is `AND happened_at <= $verified_at`.

**5. Why the recompute job isn't optional.**

Materialised state is only as good as the events that maintain it, and events get lost
(producer crashes, dead-lettered messages, bugs in a past version of the resolver). Nothing in
Design 3 notices on its own: a change that should have been resolved simply stays active, or
one resolved wrongly stays resolved. The recompute applies the read-time definition and
repairs disagreements. It should alert whenever it changes anything, because every repair
means an event was lost or mishandled, and you want to know why.

**6. Three inserts in one transaction.**

All three get the same `created_at`, the transaction's start time, because `now()` is fixed for
the transaction. `ORDER BY created_at` can't tell them apart, so their relative order is
arbitrary and can change between queries, the tie problem from chapter 5. Add a unique
tiebreaker to the order, or use `clock_timestamp()` if you genuinely need distinct insertion
times.

---

## File 08 — Parameters, plans and types

**1. Fast in tests, slow after a few minutes.**

pgx (or another caching driver) prepares the statement once per connection and reuses it.
The first five executions on each connection use custom plans, made with the real value,
which can prove the partial index applies. After the fifth, the server builds a generic
plan. If its estimated cost is close to the custom average, it switches. The generic plan
can't see the value, so it can't prove the partial predicate, and it chooses another
index. Tests rarely run a statement more than five times on one connection. A
long-running service does it in seconds.

**2. Literal versus bound value.**

A partial index can be used when the planner proves that the query's `WHERE` implies the
index predicate. With the literal `status = 'FAILED'` in the SQL text, that's true for
every execution, so even a generic plan can use the index. With `status = $1` the generic
plan has to work for every possible `$1`, and most values don't imply the predicate.

**3. Which operator types `$1`?**

The `+`. Its other operand is an `interval`, so by the unknown-type rule `$1` is assumed
to be an `interval` too. The error appears at `<` because that's the first operator that
then has no match: `timestamptz < interval`. The cause is one step earlier than the
symptom.

**4. `created_at - interval '1 day' < $1`.**

No problem. The left side is `timestamptz - interval`, which is a `timestamptz`. Then
`$1` is compared with a `timestamptz`, so by the same rule it's typed `timestamptz`,
which is what you meant. (Checked on PostgreSQL 17: it prepares and runs.) Same rule, a
different first context, the right answer by luck. That's the argument for casting
explicitly.

**5. `BETWEEN` two dates misses yesterday.**

`BETWEEN '2026-01-01' AND '2026-01-01'` against a `timestamptz` means `>= 00:00 AND <=
00:00` on that day, so it matches only rows at exactly midnight. Use a half-open range:
`created_at >= $start AND created_at < $end + interval '1 day'`, with both cast to
`timestamptz` and an explicit decision about which time zone "midnight" means. It stays
indexable, because the column is compared directly.

---

## File 09 — Changing a live table

**1. Plain `CREATE INDEX` for ten minutes versus plain `DROP INDEX` for one second.**

During the build: reads continue, and inserts, updates and deletes block until it
finishes. That's a `SHARE` lock. During the drop: everything blocks, reads included,
because it takes `ACCESS EXCLUSIVE`. It holds that for only a second, but if it has to
wait for the lock first, every query arriving in the meantime queues behind it.

**2. A sub-millisecond `ALTER TABLE` and a four-minute outage.**

The `ALTER` needed `ACCESS EXCLUSIVE` and had to wait, because a long-running transaction
(a report, a backup, an idle-in-transaction connection) held a conflicting lock. While
it waited, every new query on the table queued behind it, reads included. When the long
transaction finished, the `ALTER` ran instantly and the queue drained. `SET lock_timeout =
'3s'` on the migration session would have made the `ALTER` give up after three seconds
and leave traffic alone.

**3. `psql -c "SET …; CREATE INDEX CONCURRENTLY …"` versus `PGOPTIONS`.**

Two statements in one `-c` string are sent as one simple query, which PostgreSQL runs as
a single implicit transaction, and `CREATE INDEX CONCURRENTLY` refuses to run inside a
transaction block. `PGOPTIONS='-c statement_timeout=0'` sets the parameter at connection
start, so the `-c` string holds only the one statement. Separate `-c` flags would also
work, because psql sends each as its own query.

**4. Failed at 80%, re-run succeeds in 0.1 seconds.**

The first run left an INVALID index. The re-run's `IF NOT EXISTS` saw a relation with
that name, skipped it with a notice, and reported success. The migration runner recorded
it as applied. The database now has an index that no query uses and every write
maintains. Prove it with `SELECT indexrelid::regclass FROM pg_index WHERE NOT
indisvalid`. Fix it by dropping it concurrently and building again.

**5. Stuck at "waiting for old snapshots".**

The build is waiting for transactions that started before a certain point to finish. Look
in `pg_stat_activity` for old `xact_start` values, especially `state = 'idle in
transaction'`, which are connections holding a transaction open while doing nothing.
Ending that transaction (or the session) lets the build continue.
`idle_in_transaction_session_timeout` prevents it from recurring.

---

## File 10 — Migration runners

**1. Editing an applied migration to add an index.**

Fresh databases (new developers, CI, any environment created later) get the index.
Production and every other existing database don't, because the runner already marked
that version as applied and won't open the file again. Queries are fast in CI and slow in
production, and nobody suspects schema drift because the migrations "all ran". It can go
unnoticed for months, until someone compares `\d` output between environments.

**2. A migration numbered 18 lands on a database at 20.**

golang-migrate stores only "version 20", applies only files numbered above it, and so
never applies 18, silently, on that database. A fresh database applies it in order. That's
schema drift again. goose stores every applied version, sees that 18 is missing, and by
default refuses to run with an error. With `-allow-missing` it applies 18. Both of the
last two behaviours are fine. The silent one isn't.

**3. When does a failed goose migration leave partial changes?**

When it ran with `-- +goose NO TRANSACTION`. A transactional migration rolls back
completely on failure, so "not recorded" means "not applied". A non-transactional one
may have run some of its statements before failing, and goose records nothing. You must
make every statement in such a migration idempotent, keep them small (one index per file),
and after a failure check for leftovers, particularly INVALID indexes, before re-running.

**4. Forty-minute migration on startup, during a rolling deploy.**

The first new pod starts migrating and blocks. Its readiness probe fails, so it gets no
traffic and the rollout stalls. Its liveness probe fails after its threshold, so it's
killed mid-migration (and for a concurrent build, that leaves an INVALID index), then
restarted and tries again. Meanwhile the second new pod blocks on the migration lock
behind the first. If the deploy has a progress deadline, the rollout is marked failed and
may roll back the code, against a schema that's partly migrated.

**5. Why goose's hybrid fails with branch-deployed test environments.**

`goose fix` renumbers timestamped migrations into sequential numbers, on the assumption
that no shared database has applied them yet. If a test environment applies migrations
straight from feature branches, it has already recorded the timestamp versions. After
`fix`, the same migrations come back under new numbers. goose sees them as new and runs
them again, while the timestamp records sit there pointing at files that no longer exist.

**6. A data-moving migration's down, and a deleted migration.**

`legacy_orders` comes back containing every row in `orders`: the original legacy rows plus
everything written to `orders` in the week since, because nothing records which rows came
from where. Lost: the ability to tell them apart, and any legacy rows changed or deleted in
`orders` since, which come back in their new state or not at all. A down that moves data is
a best-effort copy, not an undo.

The deleted-file case: staging's `schema_migrations` records the deleted migration's version.
On its next run golang-migrate checks that the file for the current version exists, can't
find it, and refuses to migrate at all. The table it dropped is gone either way. The right
undo is a new migration that recreates the table, plus a restore of the data from backup.
Deleting or renumbering an applied migration never undoes anything.

---

## Things to try

- Start a throwaway PostgreSQL in Docker, build a few hundred thousand rows of
  digit-only identifiers, and compare a trigram `ILIKE '%x%'` with a `text_pattern_ops`
  `LIKE 'x%'`. Use `EXPLAIN (ANALYZE, BUFFERS)` and watch `Buffers` rather than time.
  Double the table and repeat.
- Reproduce the lock queue: one session holding a long transaction, one waiting
  `ALTER TABLE`, one plain `SELECT`. Then repeat with `lock_timeout` on the second.
- Cancel a `CREATE INDEX CONCURRENTLY` with a small `statement_timeout`, then re-run it
  with `IF NOT EXISTS` and inspect `pg_index.indisvalid`.
- `PREPARE` a query against a partial index and `EXPLAIN EXECUTE` it under
  `plan_cache_mode = force_custom_plan` and `force_generic_plan`.

## Questions worth asking me

- "Here's a slow query and its `EXPLAIN (ANALYZE, BUFFERS)`. Where is the time going,
  and is the fix an index or a rewrite?"
- "Should this field be a JSON path or a real column? What do I gain and pay?"
- "Is it safe to run this migration during business hours? Which locks does it take,
  and for how long?"
- "How would you partition this table, and which of these indexes survive the change?"
- "What does logical replication change about everything in chapter 9?"
- "How do Flyway's checksums and repeatable migrations change the rules in chapter 10?"
