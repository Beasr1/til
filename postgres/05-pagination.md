# 5. Pagination that doesn't skip or repeat

## The problem

A user pages through a list and sees the same record on pages 3 and 4. Or a record
never appears on any page. Nobody can reproduce it, because it depends on timing and on
which plan the database picked that day. Every paging scheme has a way of doing this,
and you can't fix it by flipping `ORDER BY … DESC` to `ASC`. It comes from asking the
database for "rows 41–60" of an order that doesn't exist.

## `LIMIT`/`OFFSET` needs a total order

The manual is blunt ([SELECT, LIMIT clause](https://www.postgresql.org/docs/current/sql-select.html)):

> When using `LIMIT`, it is a good idea to use an `ORDER BY` clause that constrains the
> result rows into a unique order. Otherwise you will get an unpredictable subset of the
> query's rows — you might be asking for the tenth through twentieth rows, but tenth
> through twentieth in what ordering?

And the consequence:

> The query planner takes `LIMIT` into account when generating a query plan, so you are
> very likely to get different plans (yielding different row orders) depending on what
> you use for `LIMIT` and `OFFSET`. Thus, using different `LIMIT`/`OFFSET` values to
> select different subsets of a query result *will give inconsistent results* unless you
> enforce a predictable result ordering with `ORDER BY`.

Note the phrase "unique order". `ORDER BY created_at` is an order, but it isn't unique
when two rows share a timestamp, and bulk imports, batch jobs and coarse clocks all make
that common. Rows with equal keys can come out in any order, and the order is decided
per query. So page 3 (computed with one plan) and page 4 (computed with another, because
the `OFFSET` differs) can disagree about which of the tied rows came first. Same row on
both pages, or on neither.

The fix is one column: **add a unique tiebreaker**, usually the primary key.

```sql
ORDER BY created_at DESC, id DESC
```

Changing `DESC` to `ASC` doesn't help, because ties are just as tied in either direction.

## `OFFSET` also moves when the data moves

Even with a total order, `OFFSET` counts positions, and positions shift when rows are
inserted or deleted between two page requests:

```
Newest-first list, 20 per page.

Request page 1  →  rows at positions 1–20
                   (3 new rows are inserted at the top)
Request page 2  →  positions 21–40, which now begin with the
                   last 3 rows the user already saw on page 1
```

A delete does the opposite and a row is skipped. For an admin screen that's often
tolerable. For anything that processes every row ("export all", "sync everything since
yesterday") it's a data-loss bug.

And `OFFSET` gets slower the deeper you go: to return rows 10,001–10,020 the database
produces and throws away the first 10,000. The manual notes even row locks are taken on
rows "skipped over by `OFFSET`".

## Keyset pagination

Instead of "skip N rows", say "continue after the last row I saw". The client sends
back the sort key of the last row on the page, and the next query starts strictly after
it:

```sql
-- first page
SELECT id, created_at, ...
  FROM orders
 ORDER BY created_at DESC, id DESC
 LIMIT 20;

-- next page: pass the last row's (created_at, id)
SELECT id, created_at, ...
  FROM orders
 WHERE (created_at, id) < ($1, $2)
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

The `(a, b) < (x, y)` form is a **row comparison**. It compares lexicographically,
which is exactly "after this row in this order". With an index on
`(created_at DESC, id DESC)` the planner turns it into a single index range scan:

```
Limit
  ->  Index Only Scan using orders_created_id
        Index Cond: (ROW(created_at, id) < ROW($1, $2))
```

Every page costs the same no matter how deep it is, and inserts at the top can't push
rows across the boundary, because the boundary is a value, not a position.

> ⚠️ **The trap: keying on the timestamp alone.** `WHERE created_at < $1` with only the
> last row's timestamp looks like keyset pagination and loses data. If the page ended in
> the middle of a run of rows sharing one timestamp, every remaining row of that run is
> excluded from the next page, because none of them is strictly less than `$1`. On a
> table where ten rows share a timestamp, a page boundary that falls inside them drops
> up to nine of them for good. The tiebreaker has to be in the cursor, not just in the
> `ORDER BY`.

## What keyset gives up

| | `OFFSET` | Keyset |
|---|---|---|
| Jump to page 37 | Yes | No. Only next and previous |
| Cost of a deep page | Grows with the offset | Constant |
| Stable under concurrent inserts | No | Yes |
| Needs an index matching the order | Helps | Required to be fast |
| Cursor | A number | The last row's key, often encoded opaquely for the client |

"Jump to page 37" is mostly a feature of the UI, not of user needs. People search or
filter; they don't navigate to page 37. Where you truly need random access, keep
`OFFSET` but make the order total, and accept the drift.

## Check yourself

1. A list is ordered by `created_at DESC` with `LIMIT 20 OFFSET 40`. Someone changes it
   to `ORDER BY created_at ASC` "to make it consistent with limit and offset". Which
   problem does that fix, and which does it leave?
2. Why can two `OFFSET` queries, identical except for the offset value, disagree about
   the order of tied rows?
3. A nightly job pages through "everything created today" with `OFFSET`, while the
   application keeps inserting. Which rows can it miss or double-process?
4. Write the keyset condition for a list ordered by `priority ASC, created_at DESC,
   id DESC`. Why can't you use a single row comparison here, and what do you do instead?
5. What does a keyset cursor leak to the client, and why do some APIs encode it
   opaquely?
