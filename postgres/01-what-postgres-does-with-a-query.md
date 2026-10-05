# 1. What PostgreSQL does with a query

## The problem

Every other chapter in this course says things like "the planner can't push this filter
down", "the update writes a new row version", "a replica replays the log". Each of those
sentences is short only because there's a model of the database behind it. Without the
model they're rules to memorise. With it they're consequences you could have predicted.
This chapter builds that model: where rows live, what an index really is, why an update
isn't an edit in place, how a plan gets chosen, and how to read one.

Skip it if you can already read `EXPLAIN (ANALYZE, BUFFERS)` and explain MVCC to
someone. Skim it if you can do one of the two.

## Where rows live: the heap

A table is stored as a file of fixed-size **pages**. From the manual
([Database page layout](https://www.postgresql.org/docs/current/storage-page-layout.html)): "Every table
and index is stored as an array of *pages* of a fixed size (usually 8 kB …)." The table's
pages are called the **heap**, and the name is apt. Rows go wherever there's room. The
heap has **no order**: not by primary key, not by insertion time, not by anything.

Every row has an address, its **TID** or `ctid`: "a page number and the index of an item
identifier" within the page. You can see it:

```sql
SELECT ctid, id FROM orders WHERE id IN (1, 2, 3) ORDER BY ctid;
 ctid  | id
-------+----
 (0,1) |  1
 (0,2) |  2
 (0,3) |  3
```

(This and the other outputs in this chapter come from a 200,000-row test table on
PostgreSQL 17. The rows happen to be in id order only because they were inserted in id
order into an empty table. Nothing keeps them that way.)

Reading a table without an index means reading every page. That's a **sequential scan**,
and for a big table it's the cost every index exists to avoid.

### Big values are stored elsewhere: TOAST

A page is 8 kB, and a row has to fit in one. Large values (a long text, a big JSON
document) are handled by **TOAST**: "The TOAST management code is triggered only when a
row value to be stored in a table is wider than `TOAST_TUPLE_THRESHOLD` bytes (normally
2 kB)." It will "compress and/or move field values out-of-line". The row then holds a
pointer, and reading the value means fetching it from a separate table and decompressing
it. That's cheap for one row and expensive for a hundred thousand, which matters as soon
as a query pulls one key out of every row's JSON document ([chapter 4](04-the-latest-row-per-group.md)).

## What an index is

An index is a **separate structure** that maps keys to TIDs. A btree index on `email` is,
in effect, a sorted list of `(email, TID)` pairs, arranged as a shallow tree so any key
can be found in a few page reads. To use it, PostgreSQL finds the key in the index, then
goes to the heap page the TID names to fetch the row.

Three consequences you'll meet throughout the course:

- **Every index is written on every insert.** Five indexes on a table means six places to
  write for each new row.
- **An index lookup is two reads, index then heap**, unless the index alone has every
  column the query needs (an **index-only scan**).
- **An index serves only questions its order can answer.** A sorted list of emails finds
  `= 'x'` and `LIKE 'x%'`, not `LIKE '%x'`. [Chapter 2](02-how-an-index-answers-a-query.md)
  is entirely about this.

## An update writes a new row: MVCC

PostgreSQL lets readers and writers work at the same time without waiting for each other.
It does it with **MVCC** (multi-version concurrency control). From
[§13.1](https://www.postgresql.org/docs/current/mvcc-intro.html):

> each SQL statement sees a snapshot of data (a *database version*) as it was some time
> ago, regardless of the current state of the underlying data.

and the payoff:

> reading never blocks writing and writing never blocks reading.

To make that possible, an `UPDATE` doesn't change the row in place. It writes a **new row
version** and marks the old one as replaced. A transaction that started earlier can still
see the old version, so it reads a consistent snapshot without waiting. A `DELETE` marks
the version as deleted and removes nothing.

You can watch the row move:

```sql
UPDATE orders SET status = 'paid' WHERE id = 1;
SELECT ctid, id, status FROM orders WHERE id = 1;
   ctid    | id | status
-----------+----+--------
 (1081,16) |  1 | paid
```

Row 1 was at `(0,1)`. The new version is on page 1081, at the end of the table, because
page 0 was full. The old version is still on page 0, now dead.

Old versions that no transaction can see any more are **dead tuples**. `VACUUM` (usually
run automatically by **autovacuum**) reclaims their space. A table that's updated heavily
and vacuumed too little accumulates dead tuples. That's **bloat**: more pages to read for
the same live data.

And since the new version has a new TID, every index has to learn about it, unless the
update is **HOT** (heap-only tuple). From
[Heap-only tuples](https://www.postgresql.org/docs/current/storage-hot.html), HOT applies when "the
update does not modify any columns referenced by the table's indexes" and "there is
sufficient free space on the page containing the old row". Then no new index entries are
needed. The update above couldn't be HOT: page 0 had no free space, so the new version had
to go on another page and every index on the table needed a new entry. Indexing a column
that changes on every update rules HOT out completely, which is a cost beyond the index's
own size. Leaving free space on each page (a lower `fillfactor`) makes HOT more likely for
tables that are updated often.

> **Teacher's aside.** "Readers never block writers" is true and often misread as "nothing
> ever blocks". MVCC is about *row data*. Changes to a table's *structure* (`ALTER TABLE`,
> `DROP INDEX`) still take locks that conflict with plain reads, and two writers updating
> the same row still queue for that row. Chapter 9 is the other half of this story.

## Every change is logged first: the WAL

Before a change reaches the table's files, it's written to the **write-ahead log**. From
[§28.3](https://www.postgresql.org/docs/current/wal-intro.html): "changes to data files
(where tables and indexes reside) must be written only after those changes have been
logged". After a crash, the database replays the log to redo anything that hadn't reached
the data files.

The same log is how **replicas** work: a standby receives the primary's WAL and replays
it. So a replica runs the same changes as the primary, but its *queries* are its own. That
detail decides how you check index usage ([chapter 3](03-is-anything-using-this-index.md)).

## How a plan is chosen

SQL says *what* you want, not *how* to get it. The **planner** considers ways to run the
query (which index, which join order, which join method) and estimates each one's cost
from **statistics** about the data. From
[§14.2](https://www.postgresql.org/docs/current/planner-stats.html), the planner "needs to
estimate the number of rows retrieved by a query in order to make good choices of query
plans". The statistics live in `pg_class` (`reltuples`, `relpages`: rows and pages per
table and index) and `pg_stats` (per-column most common values, histograms, distinct
counts). They're refreshed by `ANALYZE`, which autovacuum also runs. They're **not**
updated on every write.

Two facts about the planner run through the whole course:

- **It works from estimates.** Stale statistics, or data that doesn't fit its model, give
  a wrong row estimate and then a wrong plan. A large gap between estimated and actual
  rows in `EXPLAIN ANALYZE` is the first thing to look for.
- **It only makes changes it can prove don't alter the result.** If using an index or
  moving a filter *could* give a different answer, it won't, however obvious it is to you
  that the answer would be the same. Chapters 4 and 8 are built on this.

## Reading a plan

`EXPLAIN` shows the chosen plan as a tree. `EXPLAIN ANALYZE` runs the query and adds what
actually happened. `BUFFERS` adds how many pages were touched. The manual's warning
([§14.1](https://www.postgresql.org/docs/current/using-explain.html)) matters: with
`ANALYZE`, an `UPDATE` or `DELETE` really runs, so wrap it in `BEGIN … ROLLBACK`.

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM orders WHERE customer_id = 42 AND status = 'paid' LIMIT 1;

Limit  (cost=0.29..12.97 rows=1 width=4) (actual time=0.006..0.007 rows=1 loops=1)
  Buffers: shared hit=2 read=1
  ->  Index Scan using orders_customer_idx on orders
        (cost=0.29..165.02 rows=13 width=4) (actual time=0.006..0.006 rows=1 loops=1)
        Index Cond: (customer_id = 42)
        Filter: (status = 'paid'::text)
        Buffers: shared hit=2 read=1
Planning Time: 0.028 ms
Execution Time: 0.011 ms
```

Notice the inner node expects 13 matching rows (`rows=13`, the planner's guess for
`customer_id = 42 AND status = 'paid'`), while the `Limit` above it stops after one.
The index answered `customer_id = 42`. The status was checked row by row as a filter.

Read it from the innermost node outwards. Each node feeds its parent.

| Part | Meaning |
|---|---|
| `cost=0.42..8.45` | Planner's estimate, in arbitrary units: cost to the first row, then to the last |
| `rows=1` (in the cost bracket) | Estimated rows this node produces |
| `actual time=…`, `rows=1`, `loops=1` | What really happened. Multiply rows and time by `loops` for the total |
| `Index Cond` | The condition the index itself answered: the cheap part |
| `Filter` / `Rows Removed by Filter` | Checked row by row after fetching: work the index didn't save |
| `Buffers: shared hit=2 read=1` | Pages found in memory (`hit`) and read from disk (`read`) |

`Buffers` is the most honest number in the plan. Timing depends on what's cached and what
else the machine is doing, while pages touched is a property of the plan. That's why later
chapters compare plans by buffers.

The scan nodes you'll meet:

| Node | What it does | Typical when |
|---|---|---|
| **Seq Scan** | Reads every heap page | No usable index, or most of the table is wanted |
| **Index Scan** | Walks the index, fetches each matching row from the heap | Few rows wanted, in index order |
| **Index Only Scan** | Answers from the index alone | The index holds every needed column and the pages are marked all-visible |
| **Bitmap Index Scan + Bitmap Heap Scan** | Collects matching TIDs from one or more indexes, sorts them by page, then visits each page once | A moderate number of rows, or several indexes combined (`BitmapAnd`, `BitmapOr`) |

And the joins: **Nested Loop** (for each outer row, look up inner rows, which is good when
the outer side is small and the inner side indexed), **Hash Join** (build a hash table of
one side, probe it with the other) and **Merge Join** (walk two sorted inputs together).

## Transactions, briefly

Every statement runs in a **transaction**: implicitly, one per statement, or explicitly
between `BEGIN` and `COMMIT`. Its changes become visible to others only at commit, and all
of them or none. Two details come back later:

- Several statements sent as **one query string** run as one implicit transaction. That
  matters for commands that refuse to run inside one ([chapter 9](09-changing-a-live-table.md)).
- A transaction left open, especially one **idle in transaction**, holds its snapshot.
  That stops vacuum from removing dead tuples anywhere it might still need them, and it
  stalls operations that wait for old transactions to finish.

## Check yourself

1. A table has no index and 10 million rows, and you want the row with `id = 42`. Describe
   what PostgreSQL reads, in terms of pages, and why "the rows are stored in id order" isn't
   a way out.
2. An `UPDATE` changes one column of one row. List everything that might be written:
   heap, indexes, WAL. Then say what changes if the column is indexed.
3. A long-running report holds a transaction open for an hour while a busy table is updated
   constantly. What happens to that table's size, and why can't vacuum help yet?
4. In a plan, an Index Scan shows `rows=1` estimated and `actual rows=48000`. What does the
   gap suggest, and what would you run?
5. Two plans for the same query take 4 ms and 6 ms. One shows `Buffers: shared hit=40`, the
   other `shared hit=9000`. Which is better, and why might timing have misled you?
6. Why does "readers never block writers" not mean that a `SELECT` can never wait?
