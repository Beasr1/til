# 4. The latest row per group

## The problem

Many tables are histories: every change to a customer, an order or a device is a new
row. Most screens want the *current* state, which is the latest row per entity. That's
easy to write and easy to make slow. The slowness has one cause that is worth
understanding properly, because it isn't really a performance problem. It's a question
about what the query means, and you have to answer it before the planner can help you.

## `DISTINCT ON`

PostgreSQL has a direct way to say "one row per group, chosen by an order". From
[SELECT](https://www.postgresql.org/docs/current/sql-select.html):

> `SELECT DISTINCT ON ( expression [, ...] )` keeps only the first row of each set of
> rows where the given expressions evaluate to equal. […] Note that the "first row" of
> each set is unpredictable unless `ORDER BY` is used to ensure that the desired row
> appears first.

And the rule that trips everyone at least once:

> The `DISTINCT ON` expression(s) must match the leftmost `ORDER BY` expression(s).

```sql
SELECT DISTINCT ON (customer_id) customer_id, status, created_at
  FROM customer_events
 ORDER BY customer_id, created_at DESC;      -- latest event per customer
```

Two things follow from "first row of each set":

- **Ties are arbitrary.** If two events for one customer share a `created_at`, either
  one may come back, and it can change between runs. Add a tiebreaker that's unique,
  usually the primary key: `ORDER BY customer_id, created_at DESC, id DESC`.
- **It has to see every row of a group to know which is first.** That's either a sort,
  or an index already in that order. An index on `(customer_id, created_at DESC)` lets
  the planner read groups in order without sorting.

The portable equivalents are `row_number() OVER (PARTITION BY … ORDER BY …) = 1` and a
`LATERAL` subquery with `LIMIT 1`. They have the same semantics and the same trap,
which comes next.

## The trap: a filter above the dedup can't move below it

A search box on the current-state screen: "find customers whose latest record has this
phone number". The natural query puts the filter after the dedup:

```sql
WITH latest AS (
  SELECT DISTINCT ON (customer_id) customer_id, created_at, doc ->> 'phone' AS phone
    FROM customer_events
   WHERE tenant_id = $1
   ORDER BY customer_id, created_at DESC
)
SELECT customer_id FROM latest WHERE phone LIKE $2 || '%';
```

You have a perfect index on `phone`. The planner ignores it, and reads every row in
the tenant. Here is a run on a test table of 1.2 million rows, searching one tenant
with 24,000 rows (plans abridged here and below, with index names adapted to this
example):

```
Subquery Scan on latest
  Filter: (latest.phone ~~ '100000198030433%')
  Rows Removed by Filter: 3999
  ->  Unique  (rows=4000)
        ->  Sort  (rows=24000)
              ->  Bitmap Heap Scan on t  (rows=24000)
Buffers: 17,277    Execution: 60 ms
```

It isn't being stupid. **Moving the filter below the dedup would change the answer.**

Filter after dedup: "customers whose *latest* row matches". Filter before dedup:
"customers who have *any* row that matches, then show me their latest row". Someone
who changed their phone number last month is in the second set and not the first. The
planner is only allowed transformations that preserve results, so it can't push the
predicate down. The index on `phone` can only help once the filter sits right on the
base table.

So the first step isn't tuning. It's deciding which question the screen is asking.

| Semantics | Where the filter goes | Index on the searched column |
|---|---|---|
| Latest row matches | Above the dedup | Can't be used. Every row in scope is read and sorted |
| Any row matches | Below the dedup, on the base table | Used |

For identifier search, "any row matches" is usually what users want anyway: someone
searching an old reference number expects to find the customer. When that's
acceptable, write it explicitly as a candidate set:

```sql
WITH candidates AS (
  SELECT customer_id FROM customer_events
   WHERE tenant_id = $1 AND doc ->> 'phone' LIKE $2 || '%'
),
latest AS (
  SELECT DISTINCT ON (customer_id) customer_id, created_at
    FROM customer_events
   WHERE tenant_id = $1 AND customer_id IN (SELECT customer_id FROM candidates)
   ORDER BY customer_id, created_at DESC
)
SELECT customer_id FROM latest;
```

Same table, same tenant:

```
Index Scan using events_tenant_phone  (rows=4)      ← the search
Nested Loop → Index Scan using events_tenant_customer  (rows=12)
Buffers: 25    Execution: 0.07 ms
```

On this data the second query also returned **two** customers where the first returned
one. That's the semantic difference, visible in the output.

> **Teacher's aside.** When a query is slow because "the planner won't use my index",
> check whether using it would be *correct*. Filters above `DISTINCT ON`, window
> functions, `LIMIT`, aggregates, and the nullable side of an outer join all block
> push-down for the same reason: moving the filter would change the result. The fix is
> never a planner hint. You decide the semantics and write them down.

## Anything above the dedup is computed for every row below it

A corollary that costs real money with JSON columns. In the slow query above, `doc ->>
'phone'` is in the dedup's select list because the filter above needs it. So it gets
evaluated for all 24,000 input rows, not the 4,000 survivors. If `doc` is large it lives
in **TOAST** (out-of-line storage for big values), and extracting one key means fetching
and decompressing the whole document. That cost is paid even when nobody typed a
search term, if the query builder always includes the column.

The fix falls out of the push-down: the inner scans select only the columns they sort
on, and the display columns are fetched afterwards for the one page of results being
shown.

## Several search terms: `OR` or `UNION`

A search box often matches against several fields: phone *or* document number *or*
name. One query with `OR` across the fields is the obvious form.

On a single table, PostgreSQL can combine indexes for an `OR` with a **BitmapOr**,
which scans each index separately and merges the row sets. So `OR` is not automatically
a disaster. It goes wrong in two common situations:

- One arm isn't indexable, and the whole `OR` falls back to a scan, because a bitmap
  merge needs every arm indexed.
- The data lives in several tables combined with `UNION ALL`, and the `OR` sits above
  the union, where it's the push-down problem again.

Writing one `SELECT` per term per table, combined with `UNION`, makes every branch a
simple indexed lookup and lets each pick its own index. It's more SQL. It's also the
form whose plan you can predict.

## Counting the total

The other expensive thing on a paged list is "showing 1–20 of 48,213".

`COUNT(*) OVER ()` attached to the page query looks free, but it can only be computed
once the query has produced every row in scope, so a `LIMIT 20` stops being a reason to
stop early. On an unfiltered list over a large tenant, the count *is* the query.

The options:

| Approach | Cost | Accuracy |
|---|---|---|
| `COUNT(*) OVER ()` on the page query | Materialises the whole result | Exact |
| Separate `SELECT count(*)` | Same work, second round trip | Exact |
| The planner's estimate from `EXPLAIN (FORMAT JSON)`, field `Plan Rows` | Planning only | Approximate, and can be badly off for complex filters |
| Don't show a total; show "next page" | Nothing | n/a |

A reasonable hybrid: when a search term narrows the set to a handful of rows, count
exactly, because it's cheap. When the list is unfiltered, show the estimate and label
it as approximate. The PostgreSQL wiki's
[count estimate page](https://wiki.postgresql.org/wiki/Count_estimate) covers the
estimate technique and its limits.

### What is being counted

Before the cost question there's a meaning question, and it's the one that causes wrong
dashboards. "Verifications this month" on a history table can be any of these, and they
give different numbers on the same data:

| Question | SQL | On 5 events from 3 customers |
|---|---|---|
| How many events? | `count(*)` | 5 |
| How many events with an amount? | `count(amount)`, which skips `NULL`s | 3 |
| How many distinct customers? | `count(DISTINCT customer)` | 3 |
| How many failure events? | `count(*) FILTER (WHERE kind = 'fail')` | 2 |
| How many customers had a success? | `count(DISTINCT customer) FILTER (WHERE kind = 'verify')` | 2 |

(Reproduced on PostgreSQL 17.11 with a five-row table.) And there's a time question: is it
"as of now" (a point-in-time count of current state, chapter 7's territory) or "everything
that happened in the window" (cumulative)? Decide both, write the decision next to the
metric, and then write the SQL. `FILTER` lets one pass compute several of these side by side
instead of one query per number.

The empty case is the other trap. From
[Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html):

> It should be noted that except for `count`, these functions return a null value when no rows
> are selected. In particular, `sum` of no rows returns null, not zero as one might expect,
> and `array_agg` returns null rather than an empty array when there are no input rows.

So a month with no events gives `count(*) = 0` but `sum`, `avg` and `array_agg` all `NULL`,
and the same goes per group when a `FILTER` matches nothing in that group. Wrap them where
the consumer expects a value: `coalesce(sum(amount), 0)`, `coalesce(array_agg(id), '{}')`.
Then check the driver end. In Go, lib/pq scans an array through an adapter,
`Scan(pq.Array(&ids))`, and turns a `NULL` array into a nil slice; `encoding/json` then
writes a nil slice as `null`, not `[]`. A client that expected a list gets `null` for every
empty month.

Money aggregates have one more rule: keep full precision until presentation. Three line
items of 0.333333, 0.333333 and 0.333334 sum to 1.00 if you round the total, and to 0.99 if
you round each line first (both checked on 17.11). Store and aggregate at the higher scale
(`numeric` with enough digits, never `float`), and round once, where the number is shown or
invoiced. `avg` of an integer or `numeric` column returns `numeric` (the docs' function
table), so averages don't silently truncate, but the scale of the result is whatever the
division produced; round it explicitly for display.

## Check yourself

1. Two rows for the same customer have identical `created_at`. What does
   `DISTINCT ON (customer_id) … ORDER BY customer_id, created_at DESC` return, and how
   do you make it deterministic?
2. Explain to a colleague, without mentioning performance, why the planner can't move
   `WHERE phone LIKE 'x%'` from outside a `DISTINCT ON` subquery to inside it.
3. A customer changed their phone number. Under "latest matches" and "any matches"
   semantics, does a search for the *old* number find them? Which do support staff
   probably want?
4. A query's dedup step selects `doc ->> 'name'` for display, and the page shows 20
   rows out of 50,000 in scope. Where is the cost, and how would you restructure it?
5. Why does `COUNT(*) OVER ()` make `LIMIT 20` stop saving work? Then a product manager
   asks for "active customers this month" beside the total. What two things must you pin
   down before writing the SQL, and what do `count(*)`, `sum(amount)` and `array_agg(id)`
   return for a month with no rows?
6. When is a single `OR` across indexed columns fine, and when should you split it into
   a `UNION`?
