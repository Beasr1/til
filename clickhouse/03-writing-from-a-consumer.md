# 3. Writing from a consumer

## The problem

A queue consumer is at-least-once (see [kafka/03](../kafka/03-offsets-and-delivery-guarantees.md)):
after a crash or a rebalance it writes the same batch again. In PostgreSQL you'd make the
write idempotent with a unique constraint and `ON CONFLICT`. ClickHouse has neither. What it
has instead is narrower, and easy to defeat without noticing: deduplication of a retried
insert *block*, a replacing engine that resolves duplicates eventually (chapter 2), and no
way to read-and-increment a value atomically. This chapter is about writing so that a retry
is harmless.

Assumes chapters [1](01-what-clickhouse-does-with-a-table.md) and [2](02-replacing-rows.md).
Measured on ClickHouse 26.9 unless stated. Names are invented.

## Retried inserts are dropped only if they're identical

Replicated tables remember a hash of each recently inserted block. The replication
documentation: "Data blocks are deduplicated. For multiple writes of the same data block
… the block is only written once." The setting description adds the condition that
matters: "The hash sum covers the whole inserted block, so an insert is deduplicated only
when its entire data matches a previous insert (a retry), not per individual part."

Non-replicated tables can do the same if you ask. The table settings, read from
`system.merge_tree_settings`:

| Setting | Default (26.9) | Applies to |
|---|---|---|
| `replicated_deduplication_window` | 10000 blocks | Replicated tables, hashes kept in Keeper |
| `replicated_deduplication_window_seconds` | 3600 | Replicated tables |
| `non_replicated_deduplication_window` | 0 (off) | Plain MergeTree, hashes kept on local disk |

Measured on a plain table with `non_replicated_deduplication_window = 100`:

```
identical retry               2     (two rows inserted twice: second insert dropped)
retry with fresh timestamps   4     (same two rows, now64(3) re-evaluated: stored again)
```

On a two-replica cluster, a plain `ReplicatedMergeTree` (no replacing engine, so merges
can't be what removed anything) given the same two-row insert twice held two rows, not four.

One more default moved between releases. With `async_insert` on, the server buffers small
inserts and writes them as larger parts later. In the official images I ran, the session
default was `async_insert = 0` on 24.8 and `1` on 26.9. If your batching or retry logic
assumes one of these, check the setting on the server you actually run, not the
documentation's default.

The second line of the measurement is the trap. A consumer that builds its rows with
`time.Now()` for `created_at` or `last_updated`, or generates a random id per write, sends a
*different* block on every retry. Block deduplication never fires, and if those columns are
also in the sorting key (chapter 2), the replacing engine never fires either.

> ⚠️ **Make a retry byte-identical.** Derive every column from the message, not from the
> moment you wrote it: the event's own timestamp, an id derived from the message's key, a
> version taken from the message. Then both ClickHouse mechanisms work for you. If you want
> write time, let the server fill it with `DEFAULT now64(3)` and leave it out of the insert.
> Measured on the replicated table: a retried insert that omitted such a column was still
> deduplicated (two rows, not four), with a second's gap between the attempts.

## One bad row fails the whole batch

Go code often appends rows to a batch in a loop and logs any `Append` error before carrying
on, expecting the bad row to be skipped. In `clickhouse-go` v2 (read from `conn_batch.go`,
v2.40.1), the first failed `Append` marks the batch invalid and releases its connection:

```go
if err := b.block.Append(v...); err != nil {
    b.err = fmt.Errorf("%w: %w", ErrBatchInvalid, err)
    b.release(err)
    return err
}
```

Every later `Append` returns that stored error, and `Send` begins with `if b.err != nil {
return b.err }`. So "log and continue" doesn't skip the bad row. It sends nothing, and the
error surfaces at `Send`, far from the row that caused it. Validate rows before appending
them, and treat an `Append` error as a failed batch.

## There is no atomic increment

A common need: a per-table counter, "batch number" or "change version", stamped on each write
so a downstream reader can ask for everything since version N. The obvious implementation is
"read `max(current)`, add one, insert". Two writers doing that concurrently, twenty times each,
against a `ReplacingMergeTree(current)` table:

```
claims made:                        40
distinct versions claimed:          20
versions claimed by both writers:   20
rows after merge:                    1   max(current) = 20
```

Every version number was handed out twice, so two batches share each number, and a reader
asking "what changed in version 7?" gets two unrelated batches as if they were one. The
replacing engine then merges the counter's history down to one row, so the per-batch record
(a per-batch row count, say) is gone too.

ClickHouse has no `SELECT … FOR UPDATE`, no sequences and no transactions to make
read-then-write atomic. Options that work:

| Option | Why it's safe |
|---|---|
| Use a value the message already carries, such as partition and offset, or the event's own time | No coordination needed. Ordered within a partition |
| Allocate versions in a store that has atomic increment | The increment and the claim are one operation |
| Run exactly one writer for the counter | Removes the race, but scaling the consumer reintroduces it |

## Check-then-insert races, and keys you can compute

The same shape appears for identifiers. "Look up this entity's id; if there isn't one,
create a random one and insert it." Two messages for a new entity in the same batch, handled
concurrently, both miss the lookup and both create an id. Now one entity has two. A
transactional database would let a unique constraint reject the second; ClickHouse will
store both.

The fix that needs no coordination is to make the id a pure function of something the
messages share, for example a name-based UUID (UUIDv5) of the entity's natural key. Both
writers compute the same value, so the race produces two identical rows, which chapter 2's
engine and block deduplication can collapse. The design rules for such ids are in
[vector-search/03](../vector-search/03-keeping-an-index-in-step.md), where they matter even
more.

## Pick one way to say "absent"

A `Nullable(String)` column can hold `NULL` or `''`. If one code path writes `NULL` for a
missing value and another writes `''`, then `WHERE col = ''` and `WHERE col IS NULL` each
find half the absent rows. Worse, a lookup by `col = ?` called with an empty string, because
the caller didn't have the value, matches every row where someone stored `''`, and returns
an unrelated entity. Choose one representation, convert at the boundary, and refuse to look
up by an empty key.

> **Teacher's aside.** Idempotency in a transactional database is usually enforced at write
> time: the second write is rejected. In ClickHouse it's almost always achieved by
> *convergence*: the second write is accepted, and the system later recognises it as the same
> thing, at insert (identical block) or at merge (same sorting key). Convergence only works if
> the duplicate really is the same, byte for byte or key for key. That's why every rule in this
> chapter is about computing values from the message instead of from the moment.

## Check yourself

1. A consumer retries a failed insert of 500 rows. Each row has `ingested_at = now()` set in
   the application. Will the replicated table store the rows once or twice? What would you
   change?
2. A batch of 1,000 rows has one row with a value of the wrong type. The code logs the
   `Append` error and continues. What's in the table afterwards, and where does the error
   surface?
3. Two consumer instances each stamp batches with `max(version) + 1`. Describe what a
   downstream job that syncs "everything since version N" sees.
4. Why does a name-based id turn the check-then-insert race from a correctness bug into a
   harmless duplicate, and what has to be true of the input to the name?
5. A lookup function is called with an empty identifier because the message didn't carry
   one. On a table where some rows store `''` for missing identifiers, what does it return?
