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
5. Why does `COUNT(*) OVER ()` make `LIMIT 20` stop saving work?
6. When is a single `OR` across indexed columns fine, and when should you split it into
   a `UNION`?
