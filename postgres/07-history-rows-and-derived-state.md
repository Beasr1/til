# 7. History rows and derived state

## The problem

A system records events as rows: a verification happened, a customer's details changed
upstream. Screens want a *state*: "is this customer's verification still current?". The
state isn't stored anywhere. It's derived from the events. Where you derive it (at read
time, in a column on the event rows, or in its own table) is one of the earliest design
decisions in a service. It's also the one most likely to be changed five or six times,
because each choice fails in a different and slightly delayed way. This chapter walks the
usual sequence so you can skip to the end of it.

The running example: a `verifications` table (one row per completed check) and a
`changes` feed (upstream data changed after a check, so that check may be out of date).
The derived state is **stale**: a customer is stale if a change arrived after their latest
verification.

## Design 1: a flag on the history rows

The first instinct is a column: `verifications.is_stale`. When a change arrives, set it on
the customer's rows.

```sql
UPDATE verifications SET is_stale = true, stale_reason = $2
 WHERE customer_id = $1;
```

It breaks in four ways, and they're worth naming because they recur:

| Problem | Why |
|---|---|
| It mutates history | A verification row describes something that happened. "Stale" is a fact about *now*, and writing it onto the past makes the table answer two questions at once |
| Which rows? | All of the customer's rows? Only the latest? In which of several tables, if verification types are stored separately? Each query answers differently |
| It's lost on the next event | A new verification row starts with `is_stale = false`, while the old ones still say `true`. Is the customer stale? It depends on which row you read |
| Write cost | Every change rewrites history rows: new row versions, index updates, no HOT if the flag is indexed ([chapter 1](01-what-postgres-does-with-a-query.md)) |

The tell is that queries start needing "the latest row's flag", which is the next design.

## Design 2: derive it at read time

Store changes in their own table and compute staleness when reading:

```sql
SELECT v.customer_id,
       (c.created_at IS NOT NULL AND c.created_at > v.created_at) AS is_stale
  FROM latest_verification v
  LEFT JOIN latest_change c USING (customer_id, tenant_id);
```

where `latest_verification` and `latest_change` are each a `DISTINCT ON` over their table
([chapter 4](04-the-latest-row-per-group.md)). History is immutable now, and there's exactly
one definition of stale, the comparison. That's a real improvement.

What goes wrong is subtler, and it's about **which timestamp**:

- **`updated_at` versus `created_at`.** The comparison uses one of them on each side. If
  `updated_at` is maintained by a trigger, the comparison is only as correct as the trigger.
  A trigger missing on some databases ([chapter 10](10-migration-runners.md), rule 1) makes
  the same query give different answers in different environments. If `updated_at` is set
  by the application, any code path that forgets to set it does the same. `created_at` with
  a column default is the only one of the four that's hard to get wrong. That's why teams
  that start with `updated_at` tend to end up on `created_at`.
- **Insert time versus event time.** `created_at` says when *your* row was inserted, not
  when the change happened upstream. If changes arrive through a queue with lag, a change
  that happened *before* a verification can be inserted *after* it and marks the customer
  stale wrongly. The comparison has to use the time the underlying event happened, carried
  in the event, stored in its own column.
- **Every reader repeats it.** The dashboard, the detail page, the count endpoint and the
  export each embed the comparison, the dedup and the join. Each copy drifts: one uses
  `>=`, one forgets the tenant, one compares against the latest row of the wrong table.
  When a fix lands in one copy, the screens disagree.
- **Cost.** Each read deduplicates two histories and joins them. That's fine per customer
  and expensive per page of a large tenant, the problem [chapter 4](04-the-latest-row-per-group.md)
  is about.

## Design 3: materialise the state, resolved by events

Keep the change rows immutable as facts, and add the derived state as explicit columns
that events maintain:

```sql
ALTER TABLE changes
  ADD COLUMN is_active          boolean NOT NULL DEFAULT true,
  ADD COLUMN resolved_at        timestamptz,
  ADD COLUMN resolved_by        uuid;

CREATE INDEX changes_active ON changes (tenant_id) WHERE is_active;
```

A new change is active. A completed verification *resolves* the changes it supersedes.
"Is the customer stale?" becomes "does the customer have an active change?", which is one
indexed lookup with no dedup and no timestamp comparison at read time.

This moves the hard part rather than removing it. Resolution now happens in a writer,
often a queue consumer, and the writer must be correct about three things:

- **Order.** Resolve only changes that happened *before* the verification:
  `WHERE is_active AND happened_at <= $verified_at`. Without that line, a late-arriving
  verification event resolves a change that came after it, and the customer looks current
  when they aren't. Arrival order isn't event order.
- **Idempotency.** The resolving event can be delivered twice. Setting `is_active = false`
  where it's already false is naturally idempotent. Make sure anything else the resolver does
  is as well.
- **Missed events.** If the event that should resolve a change is lost, the change stays
  active for ever. Keep a way to recompute: a job that applies Design 2's comparison and
  repairs any rows that disagree. Run it after incidents and on a schedule, and alert if it
  ever changes anything.

> **Teacher's aside.** People present "compute at read time" and "store derived state" as a
> trade-off between speed and correctness. It's really a choice of *where the ordering
> logic lives*. At read time, every query has to get the timestamp comparison right. In a
> materialised column, one writer has to get it right, and the reads become trivial. The
> second is usually better, because one place is easier to get right than six. But it only
> works if that writer compares event times, not arrival order, and if a recompute path
> exists for when it's wrong.

## Which timestamp, in one table

| Column | Set by | Means | Safe for ordering decisions? |
|---|---|---|---|
| `created_at DEFAULT now()` | The database, on insert | When this row was written | Within one table and one writer, mostly. Not across a queue |
| `updated_at` via trigger | A trigger, on update | When this row last changed | Only if the trigger exists on every database (check `\d table`) |
| `updated_at` set by code | Application | When some code path last remembered to set it | No |
| `happened_at` from the event | The producer | When the real-world thing happened | Yes, when producers' clocks are reasonable, and the only option across systems |

One more property of `now()` worth knowing: in PostgreSQL it returns the start time of the
current *transaction*, not the current instant, so every row inserted in one transaction
gets the same timestamp. That's another source of ties for the pagination problem in
[chapter 5](05-pagination.md). `clock_timestamp()` gives the actual time of the call.
Checked on PostgreSQL 17: two inserts 200 ms apart inside one transaction got identical
`now()` values and `clock_timestamp()` values 200 ms apart.

## Check yourself

1. A `verifications.is_stale` flag is set on a customer's rows when a change arrives. Then a
   new verification is inserted. Write down what three different reasonable-looking queries
   would report for "is this customer stale?".
2. A read-time comparison uses `change.updated_at > verification.updated_at`, where both
   columns are maintained by triggers. Production and staging disagree for some customers.
   What would you check first, and how?
3. Changes arrive through a queue with up to ten minutes of lag. Give a concrete timeline in
   which a `created_at` comparison marks a customer stale who isn't.
4. In Design 3, the resolver runs `UPDATE changes SET is_active = false WHERE customer_id =
   $1 AND is_active`. Construct the sequence of events that leaves a stale customer looking
   current.
5. Why is a periodic recompute job part of Design 3 rather than an optional extra? What
   should it alert on?
6. Three rows are inserted in one transaction with `created_at DEFAULT now()`. What are their
   timestamps, and what does that mean for `ORDER BY created_at`?
