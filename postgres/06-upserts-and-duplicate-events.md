# 6. Upserts and duplicate events

## The problem

Anything that arrives over a network can arrive twice: a retried HTTP request, a Kafka
message redelivered after a rebalance, a webhook the sender wasn't sure you received.
If each arrival inserts a row, duplicates pile up and every count on top of them is
wrong. The standard defence is a **unique key on the event's identity** plus
`INSERT … ON CONFLICT`. It's a good defence. It also has three sharp edges, and all
three produce code that works in testing and misbehaves on the first real duplicate.

## The shape

```sql
CREATE UNIQUE INDEX uq_events_source
    ON events (source_event_key, tenant_id);

INSERT INTO events (source_event_key, tenant_id, payload)
VALUES ($1, $2, $3)
ON CONFLICT (source_event_key, tenant_id) DO NOTHING;
```

The first arrival inserts. Every later arrival with the same key does nothing, and the
statement succeeds. The handler doesn't need to know whether it has seen the event
before. That property, "doing it twice has the same effect as doing it once", is
**idempotency**, and it's what makes at-least-once delivery safe to build on.

## Edge 1: `RETURNING` returns nothing on a conflict

It's natural to want the new row's id back:

```sql
INSERT INTO events (...) VALUES (...)
ON CONFLICT (source_event_key, tenant_id) DO NOTHING
RETURNING id, created_at;
```

The manual ([INSERT](https://www.postgresql.org/docs/current/sql-insert.html)):

> Only rows that were successfully inserted or updated will be returned.

On a conflict, nothing was inserted, so **zero rows** come back. Run it twice:

```
INSERT 0 1     id = 1
INSERT 0 0     (0 rows)
```

Now look at how most drivers read "the one row I inserted": a single-row helper like Go's
`QueryRow(...).Scan(...)`, Python's `fetchone()` or Java's `executeQuery().next()`.
On zero rows the Go version returns a "no rows" error. So the duplicate, which
`DO NOTHING` was written to make harmless, comes back to the caller as a failure. The
caller logs it as an error, retries, or sends it to a dead-letter queue, all of which
is wrong.

Handle zero rows explicitly. In Go with pgx that means checking for `pgx.ErrNoRows`
and treating it as "already recorded". If you need the existing row's id too, either
select it in a second statement or use the `DO UPDATE` trick below.

> **Teacher's aside.** A related habit, `DO UPDATE SET col = EXCLUDED.col` "so
> RETURNING always gives a row", is not free. It turns a no-op into a write: a new row
> version, index updates, a trigger firing, and an `updated_at` bump that makes the row
> look as if something changed. Some teams switch from `DO UPDATE` to `DO NOTHING`
> precisely because a duplicate event was overwriting good data with a replay of old
> data. Pick on meaning: does a second arrival carry newer information
> (`DO UPDATE`, ideally with a `WHERE` guard comparing versions or timestamps), or is it
> the same event again (`DO NOTHING`)?

## Edge 2: `NULL` never conflicts

Unique indexes treat `NULL`s as distinct from each other by default. The manual
([CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html)): "The
default is that they are distinct, so that a unique index could contain multiple null
values in a column."

So if `source_event_key` is `NULL`, every insert succeeds, and `ON CONFLICT` never
fires:

```
INSERT ... VALUES (NULL, 1), (NULL, 1) ON CONFLICT ... DO NOTHING
INSERT 0 2
```

That can be deliberate. A common migration path is: old callers don't send an event key,
so their rows are stored with `NULL` and bypass deduplication, while new callers send
one and are deduplicated. If that's the intent, write it down next to the index. If it
isn't, `NULLS NOT DISTINCT` (PostgreSQL 15 and later) makes `NULL`s collide, or the
column should simply be `NOT NULL`.

Watch for the in-between case: the application converts an empty string to `NULL`
before inserting, "so the old callers bypass the index". Then any new caller that sends
an empty key by mistake silently loses deduplication too.

## Edge 3: a partial unique index must be named by its predicate

If the unique index only covers rows that have a key:

```sql
CREATE UNIQUE INDEX uq_events_source
    ON events (source_event_key, tenant_id)
 WHERE source_event_key IS NOT NULL;
```

then `ON CONFLICT (source_event_key, tenant_id)` alone fails:

```
ERROR:  there is no unique or exclusion constraint matching the ON CONFLICT specification
```

The conflict target has to repeat the predicate, so PostgreSQL can **infer** which index
you mean. From the manual: "`index_predicate`: Used to allow inference of partial
unique indexes."

```sql
ON CONFLICT (source_event_key, tenant_id) WHERE source_event_key IS NOT NULL
DO NOTHING
```

The good news is that this one fails loudly, on the first execution.

## What `ON CONFLICT DO UPDATE` guarantees, and what it doesn't

The manual: "`ON CONFLICT DO UPDATE` guarantees an atomic `INSERT` or `UPDATE` outcome;
provided there is no independent error, one of those two outcomes is guaranteed, even
under high concurrency." That's the reason to use it over "select, then insert or
update", which races.

Two limits worth knowing:

- **One statement can't hit the same row twice.** A multi-row `VALUES` list containing
  two rows with the same key fails with a cardinality violation under `DO UPDATE`.
  Deduplicate the batch before sending it.
- **Atomic isn't ordered.** If two different versions of the same entity arrive out of
  order, plain `DO UPDATE` keeps whichever arrived *last*, not whichever is *newest*.
  Guard it:

  ```sql
  ON CONFLICT (entity_id) DO UPDATE
     SET payload = EXCLUDED.payload, version = EXCLUDED.version
   WHERE entities.version < EXCLUDED.version;
  ```

## Projection tables kept by triggers

A related pattern: a small table derived from a big one ("which tenants has this
customer ever been seen in?") maintained by an `AFTER INSERT` trigger that upserts into
it. It keeps reads cheap. Two things to plan for:

- **Backfill.** A trigger only sees rows written after it exists. The history has to be
  copied in separately, and that copy is a long-running data migration with its own
  locking and timeout concerns (chapter 9). If the backfill is dropped from the
  migration because it's too slow, the projection is silently incomplete for old data.
- **Write amplification.** Every write to the hot table now also writes to the
  projection, inside the same transaction. A trigger that upserts with `DO UPDATE SET
  last_seen = now()` writes a new row version on every single insert, and concurrent
  inserts for the same key queue on that row's lock.

## Check yourself

1. A handler runs `INSERT … ON CONFLICT DO NOTHING RETURNING id` and scans the result
   into a variable. Describe exactly what happens on the second delivery of the same
   event, step by step, as far as the caller's error handling.
2. Why might switching an upsert from `DO UPDATE` to `DO NOTHING` *fix* a data bug?
   Give a concrete sequence of events.
3. A unique index exists on `(event_key, tenant_id)`. Old clients send no event key.
   What happens to their duplicates, and is that necessarily wrong?
4. You add `WHERE event_key IS NOT NULL` to make the unique index partial. Which
   existing statements break, and how will you find out?
5. Two updates for the same entity, versions 7 and 8, arrive in the order 8, 7. What
   does a plain `DO UPDATE` leave in the table, and what's the one-line fix?
