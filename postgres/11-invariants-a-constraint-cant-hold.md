# 11. Invariants a constraint can't hold

## The problem

A billing rule: the first event for a customer in a 30-day window is charged, and later
events in that window are free. The table has a unique index and the insert says
`ON CONFLICT DO NOTHING`, so duplicates look impossible. Then two events for the same
customer are processed at the same moment and the customer is charged twice.

Nothing is broken in PostgreSQL. The application asked "is there a charge?" in one
statement and wrote one in another, and two transactions did that at once. This chapter is
about rules like that, the ones a `UNIQUE` constraint can't express by itself, and the three
ways to make the database arbitrate them anyway: a lock per business key, a conditional
`UPDATE`, and a row lock.

It assumes chapter 6 (`ON CONFLICT`) and the isolation levels. If Read Committed, lost
updates and write skew are new to you, read
[interview/cs-fundamentals file 07 §7.3–7.6](../interview/cs-fundamentals/07-transactions-and-isolation.md#75-what-postgresql-actually-does-reproduced)
first. This chapter doesn't repeat it.

## 11.1 Why check-then-insert races

The handler looks reasonable:

```sql
BEGIN;
SELECT EXISTS (SELECT 1 FROM charges
                WHERE customer = $1 AND window_start > now() - interval '30 days');
-- if false:
INSERT INTO charges (customer, window_start) VALUES ($1, now())
ON CONFLICT (customer, window_start) DO NOTHING;
COMMIT;
```

Two transactions, A and B, for the same customer:

```
A: BEGIN                          B: BEGIN
A: SELECT EXISTS … → false        B: SELECT EXISTS … → false     ← neither has committed
A: INSERT (cust, A's now())       B: INSERT (cust, B's now())
A: COMMIT                         B: COMMIT                      → two charges
```

At Read Committed, the default, each statement sees what was committed when it started.
When B checks, A's row isn't committed, so B can't see it. That's correct behaviour: B is not
allowed to read uncommitted data. The check was simply never a guarantee.

So why didn't the unique index catch the second insert? Because of the key.

## 11.2 A unique constraint only serialises racers that compute the same key

When two transactions insert the same key, PostgreSQL makes the second wait for the first and
then either fail or, with `ON CONFLICT`, take the conflict path. That is the one place the
database arbitrates a race for you, and it only works when **both racers produce the same
key**.

Here the key includes `window_start = now()`, and `now()` is not "now". From
[Current Date/Time](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-CURRENT):

> Since these functions return the start time of the current transaction, their values do not
> change during the transaction.

Two transactions that started a few microseconds apart have different `now()` values, so they
insert different keys, and the unique index has nothing to say. `ON CONFLICT` still does a
job here, deduplicating a redelivery of the *same* transaction's work if it's retried with the
same timestamp, but it was never the race guard it looked like.

If the window is a calendar bucket ("one charge per customer per month"), the fix is to put
the bucket in the key, `(customer, date_trunc('month', …))`. Every racer then computes the
same key and `ON CONFLICT DO NOTHING` is the whole answer. A rolling window ("no charge in
the last 30 days") has no such key: whether a new row is allowed depends on rows that exist,
not on a value the new row carries.

| Rule | Can a constraint hold it? |
|---|---|
| One row per (customer, month) | Yes. A unique index on the bucket |
| No overlapping periods per customer | Yes, if periods are stored as ranges: an exclusion constraint, below |
| No row in the last 30 days, counted from each event | No |
| At most K rows per key | No |

Ranges deserve a mention because they're the constraint people forget. If each charge stores
its window as a `tstzrange`, an **exclusion constraint** forbids overlap per customer, using
the [`btree_gist`](https://www.postgresql.org/docs/current/btree-gist.html) extension for the
equality part. The manual's example
([§8.17.10](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT)) is
a room booking:

```sql
CREATE EXTENSION btree_gist;
CREATE TABLE room_reservation (
    room text,
    during tsrange,
    EXCLUDE USING GIST (room WITH =, during WITH &&)
);
```

A conflicting insert fails with SQLSTATE `23P01`, `exclusion_violation`. When your rule can be
phrased as "these ranges don't overlap", prefer this: it's a constraint, so nothing can bypass
it.

## 11.3 Serialising per business key: advisory locks

When no constraint fits, make the check and the write happen one transaction at a time *per
customer*. A row lock can't do it, because when the customer has no charge yet there is no
row to lock. What you need is a lock on the *idea* "charges for customer 42".

PostgreSQL has exactly that. From
[Explicit Locking §13.3.5](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS):

> There are two ways to acquire an advisory lock in PostgreSQL: at session level or at
> transaction level. Once acquired at session level, an advisory lock is held until explicitly
> released or the session ends. […] Transaction-level lock requests, on the other hand, behave
> more like regular lock requests: they are automatically released at the end of the
> transaction, and there is no explicit unlock operation.

Use the transaction level. A session-level lock outlives a rollback (the manual: "a lock
acquired during a transaction that is later rolled back will still be held following the
rollback"), and behind a connection pool the session is whatever connection you happened to
get, so a forgotten unlock stays held by a connection someone else will borrow next.

| Function | Waits? | Released |
|---|---|---|
| `pg_advisory_xact_lock(key)` | Yes | At commit or rollback |
| `pg_try_advisory_xact_lock(key)` | No, returns `false` | At commit or rollback |
| `pg_advisory_lock(key)` / `pg_advisory_unlock(key)` | Yes | Explicit unlock, or session end. Stacks: lock twice, unlock twice |

The key is a `bigint`, or two `integer`s. The manual
([§9.28.10](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS))
notes "these two key spaces do not overlap". Derive the key from the business identity with a
stable hash, computed in the application:

```sql
BEGIN;
SELECT pg_advisory_xact_lock($1);   -- $1 = hash("charge", customer) as a bigint
SELECT EXISTS (…);                  -- the check, now alone for this customer
INSERT …;                           -- the write
COMMIT;                             -- releases the lock
```

The second transaction for the same customer waits at the lock. When it gets the lock, the
first has committed, and at Read Committed its next statement takes a fresh snapshot, so the
check sees the first charge.

I ran the handler above with eight concurrent calls for one customer, a 5 ms pause between
check and insert, on PostgreSQL 17.11 (Go, pgx v5.7.6; the code is at the end of the chapter).
Six runs, except the last row, which I added later and ran twice:

| Variant | Rows after 8 concurrent calls |
|---|---|
| No lock, Read Committed | 7 on the first run (probably one caller started after the first commit), 8 on the other five |
| Advisory lock, Read Committed | 1, every run |
| Advisory lock, **Repeatable Read** | 8, every run |
| No lock, Serializable | 1, with the other 7 failing `40001`, both runs |

> ⚠️ **Under Repeatable Read the lock doesn't help.** A Repeatable Read transaction "sees a
> snapshot as of the start of the first non-transaction-control statement in the transaction"
> ([§13.2.2](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-REPEATABLE-READ)).
> The first statement is the lock call, so the snapshot is taken *before* it waits. The
> check then reads that old snapshot and still sees no charge. Take advisory locks for
> check-then-write in Read Committed transactions. Serializable is the other honest answer:
> it detects the conflict without any lock, at the price of a retry loop for `40001`.

Four things to get right with the key:

- **Every writer must take the same lock.** "Advisory" means the system "does not enforce
  their use". A backfill script or an admin endpoint that inserts charges without the lock
  bypasses the rule silently.
- **Every writer must compute the same number.** If two services lock the same resource,
  they need the same hash over the same bytes. I checked that the Go and JavaScript versions
  below agree: both give `-193799470852560564` for `("charge", "customer-42")`. Put a
  separator between parts, so `("ab", "c")` and `("a", "bc")` differ.
- **Collisions only cost waiting.** Two unrelated customers that hash to the same key
  serialise against each other. That's slower, never wrong. A 64-bit hash makes it rare.
- **Prefix the key with what it protects.** `"charge"` above. Another feature locking the
  same customer for a different reason shouldn't queue behind billing.

The two-integer form is handy when part of the identity is already a small integer. A rule
like "keep at most K snapshots per (device, sensor number)" can lock
`pg_advisory_xact_lock(hash32(device), sensor)`, with `hash32` a 32-bit hash computed in the
application, then count, evict the oldest and insert, all under the lock. Only the string
part is hashed, so two sensors on one device never collide. That rule was a unique index
when K was 1. Raising K means dropping the
unique index, and the down migration has to repair data first (keep the newest row per slot,
with `DISTINCT ON` from chapter 4) before it can put the unique index back.

> **Teacher's aside.** People reach for advisory locks as if they were a mutex in shared
> memory. The useful mental model is narrower: an advisory lock is a row lock on a row that
> doesn't exist. It has the same lifetime as a row lock, it holds the connection while it
> waits (which matters for pools, chapter 12), and it protects only code that asks for it.
> If you can create the row, for example a `customers` row you lock with `FOR UPDATE`
> before inserting a charge, that works just as well and is visible in the schema.

## 11.4 The constraint is still the authority

Two pieces of code that look redundant are both needed.

**The application check is a courtesy.** A pre-flight "is this email taken?" gives a good
error message in the common case. Two concurrent sign-ups both pass it, and only the unique
index stops the second. If nothing handles that error, the second user sees a 500. Translate
SQLSTATE `23505`, `unique_violation`
([Appendix A](https://www.postgresql.org/docs/current/errcodes-appendix.html)), into the
same response the pre-check gives, typically
[409 Conflict (RFC 9110 §15.5.10)](https://www.rfc-editor.org/rfc/rfc9110#section-15.5.10).

**The lock doesn't remove the need to tolerate `23505`.** A writer that skips the lock, a
redelivered message, or a second job that writes the same effect can still collide on the
unique key. Either keep `ON CONFLICT DO NOTHING` on the insert, or catch `23505` and treat it
as "already done".

> ⚠️ **Catching `23505` doesn't rescue the transaction.** After any error, the transaction is
> aborted. The next statement fails with `25P02`, "current transaction is aborted, commands
> ignored until end of transaction block" (reproduced). If the insert isn't the last thing
> the transaction does, use `ON CONFLICT DO NOTHING`, which isn't an error, or wrap the insert
> in a `SAVEPOINT` and `ROLLBACK TO SAVEPOINT` on `23505`.

A related trap is scoping the idempotency key too narrowly. If a migration runs in two phases
and the "already processed?" check filters on `phase = current`, phase 2 re-processes
everything phase 1 committed. The key has to cover every run that can write the same effect.

### Case-insensitive uniqueness, in the right order

A common change: let users log in with any casing of their email. The tempting version is
`WHERE lower(email) = lower($1) LIMIT 1` and a plain index on `lower(email)`. If the table
already holds `Ana@example.com` and `ana@example.com` as two accounts, `LIMIT 1` picks one of
them arbitrarily (chapter 4's tie problem), and someone logs into the wrong account.

The manual ([Indexes on Expressions](https://www.postgresql.org/docs/current/indexes-expressional.html))
gives the constraint you actually want:

> If we were to declare this index `UNIQUE`, it would prevent creation of rows whose `col1`
> values differ only in case, as well as rows whose `col1` values are actually identical.

So the order is:

1. Find the case-duplicates: `SELECT lower(email), array_agg(id) FROM users GROUP BY 1 HAVING count(*) > 1`.
2. Merge or rename them. That's a product decision, not a migration detail.
3. `CREATE UNIQUE INDEX CONCURRENTLY … ON users (lower(email))` (chapter 9). If step 2 missed
   one, it fails with `23505: could not create unique index`, `DETAIL: Key (lower(email))=(ana@example.com) is duplicated`
   (reproduced), and leaves an INVALID index to drop.
4. Only then switch the lookup.

The [`citext`](https://www.postgresql.org/docs/current/citext.html) type is the other route,
and its page lists the same problem as a motive: a plain `UNIQUE` on `text` "won't enforce
uniqueness case-insensitively". The page also says to "consider using nondeterministic
collations … instead of this module", because they handle more Unicode cases correctly.

## 11.5 Compare-and-set: preconditions in the `WHERE` clause

State machines have the same race in a different shape. A session moves from `in_progress` to
`completed` or `failed`, and those are terminal. A message consumer and an API call both try
to finish it:

```
read state → "in_progress" → not terminal → UPDATE … SET state = 'failed'
```

Between the read and the write, the other path can finish the session, and this one
overwrites a terminal state. Worse, both paths think they made the transition, so both fire
the side effect: a bill, a notification, an event.

Put the precondition in the `UPDATE` and look at how many rows it changed:

```sql
UPDATE sessions SET state = $2
 WHERE id = $1 AND state NOT IN ('completed', 'failed');
```

The command tag says how many rows it updated. From
[UPDATE](https://www.postgresql.org/docs/current/sql-update.html):

> The *count* is the number of rows updated, including matched rows whose values did not
> change. […] If *count* is 0, no rows were updated by the query (this is not considered an
> error).

One row means this call made the transition and owns its side effects. Zero means someone
else got there first. That's not an error, so the caller has to check.

Why does it hold under concurrency at Read Committed? Because the `WHERE` is re-checked
against the winner's version. From
[§13.2.1](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED):

> the would-be updater will wait for the first updating transaction to commit or roll back
> […] The search condition of the command (the `WHERE` clause) is re-evaluated to see if the
> updated version of the row still matches the search condition.

The second `UPDATE` waits, sees `state = 'completed'`, no longer matches, and updates zero
rows. Eight concurrent finishers on PostgreSQL 17.11, three runs each:

| Version | Callers that thought they won | Final state |
|---|---|---|
| Read, decide in Go, then `UPDATE … WHERE id = $1` | 8, 8, 8 | Whichever wrote last |
| `UPDATE … WHERE id = $1 AND state NOT IN (…)`, check rows affected | 1, 1, 1 | The first finisher's |

> ⚠️ "Including matched rows whose values did not change" is the trap in the trap. If the
> precondition doesn't exclude the target state, `SET state = 'completed' WHERE id = $1` on an
> already-completed row reports **1**, and the side effect fires again. The `WHERE` must say
> what the row has to look like *before*, not just which row.

At Repeatable Read or Serializable the second updater doesn't re-check. It fails with
`could not serialize access due to concurrent update` and must retry, as reproduced in
[interview/cs-fundamentals file 07 §7.5](../interview/cs-fundamentals/07-transactions-and-isolation.md#75-what-postgresql-actually-does-reproduced).
The same chapter's §7.6 has the version-column form of this pattern
(`WHERE version = $read`), for when there's no natural state to compare.

## 11.6 Locking the row first: `SELECT … FOR UPDATE`

Some decisions don't fit in one `WHERE`. "Allow this debit if the balance stays above the
account's configured floor, where the price comes from a pricing call" needs application code
between the read and the write. Lock the row as you read it:

```sql
BEGIN;
SELECT balance, floor FROM accounts WHERE id = $1 FOR UPDATE;
-- decide in the application
UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;
```

From [§13.3.2](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS):

> `FOR UPDATE` causes the rows retrieved by the `SELECT` statement to be locked as though for
> update. This prevents them from being locked, modified or deleted by other transactions
> until the current transaction ends.

A concurrent debit waits at its own `SELECT … FOR UPDATE`, then reads the new balance. Keep
the work between the lock and the commit short, and never make a network call while holding
it if you can avoid it: every other debit for that account queues behind it.

When the rule fails, return a typed business error ("floor exceeded") that the HTTP layer maps
to a 4xx, rather than a generic error it has to string-match.

Choosing between the tools:

| The invariant | Use |
|---|---|
| One row per key, and every writer computes the same key | Unique constraint, `ON CONFLICT` |
| No overlapping periods per key | Exclusion constraint on a range |
| A transition on one row, decidable in SQL | Conditional `UPDATE`, check rows affected |
| A decision on one existing row that needs application logic | `SELECT … FOR UPDATE`, then `UPDATE` |
| A rule over rows that may not exist yet (none in 30 days, at most K) | Advisory lock per business key, or `FOR UPDATE` on a parent row |
| You'd rather not enumerate the cases | Serializable, and retry every `40001` |

## 11.7 Each enum is its own type

One more transition bug, because it turns up exactly when mapping states between tables. Two
columns hold "the same" state as two enums, `status order_status` and
`shipment shipment_status`, whose value lists overlap but aren't identical (`'NOT STARTED'`
in one, `'NOT_DISPATCHED'` in the other, `'SHIPPED'` in both). A consumer sets both from one
parameter. From
[Enumerated Types §8.7.3](https://www.postgresql.org/docs/current/datatype-enum.html#DATATYPE-ENUM-TYPE-SAFETY):

> Each enumerated data type is separate and cannot be compared with other enumerated types.

Reproduced on PostgreSQL 17.11:

```
UPDATE o SET status = $1, shipment = $1 …
ERROR:  42P08: inconsistent types deduced for parameter $1
DETAIL:  order_status versus shipment_status

UPDATE o SET status = $1::order_status, shipment = $1::shipment_status …   -- 'NOT STARTED'
ERROR:  22P02: invalid input value for enum shipment_status: "NOT STARTED"
```

The first fails at prepare time, which is kind. The second fails at runtime for exactly the
values the two enums don't share, which is not. Map the states explicitly in code, one
function with a case per value, and remember that enum values can be added but never removed
(the README's [ALTER TYPE](https://www.postgresql.org/docs/current/sql-altertype.html) row).

## 11.8 The three moves in application code

The per-key lock, the compare-and-set, and recognising `23505`, in Go with pgx v5.7.6. These
are the functions behind the tables above, minus the 5 ms pause and the isolation-level
switch. As printed, they compile with Go 1.24 and I ran them against the same PostgreSQL 17.11.

```go
// lockKey turns a business identity into a bigint advisory-lock key:
// 64-bit FNV-1a, with a 0 byte after each part.
func lockKey(parts ...string) int64 {
	h := fnv.New64a()
	for _, p := range parts {
		h.Write([]byte(p))
		h.Write([]byte{0})
	}
	return int64(h.Sum64())
}

// chargeOnce records a charge unless one exists in the last 30 days.
func chargeOnce(ctx context.Context, pool *pgxpool.Pool, customer string) error {
	tx, err := pool.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.ReadCommitted})
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx) // does nothing once Commit has succeeded

	if _, err := tx.Exec(ctx, `SELECT pg_advisory_xact_lock($1)`,
		lockKey("charge", customer)); err != nil {
		return err
	}
	var exists bool
	if err := tx.QueryRow(ctx, `SELECT EXISTS (SELECT 1 FROM charges
	     WHERE customer = $1 AND window_start > now() - interval '30 days')`,
		customer).Scan(&exists); err != nil {
		return err
	}
	if !exists {
		if _, err := tx.Exec(ctx, `INSERT INTO charges (customer, window_start)
		     VALUES ($1, now()) ON CONFLICT (customer, window_start) DO NOTHING`,
			customer); err != nil {
			return err
		}
	}
	return tx.Commit(ctx)
}

// finish moves a session to a terminal state. It reports whether this call
// made the transition, so only the winner fires side effects.
func finish(ctx context.Context, pool *pgxpool.Pool, id int64, to string) (bool, error) {
	tag, err := pool.Exec(ctx, `UPDATE sessions SET state = $2
	     WHERE id = $1 AND state NOT IN ('completed', 'failed')`, id, to)
	if err != nil {
		return false, err
	}
	return tag.RowsAffected() == 1, nil
}

func isUniqueViolation(err error) bool {
	var pgErr *pgconn.PgError
	return errors.As(err, &pgErr) && pgErr.Code == "23505"
}
```

<details><summary>Rust</summary>

tokio-postgres 0.7.13, built with `cargo build --offline` and run against the same
PostgreSQL 17.11 with eight concurrent tasks: one charge row, one `finish` winner, and
`is_unique_violation` true for a duplicate primary key.

```rust
use tokio_postgres::{error::SqlState, Client, Error};

/// 64-bit FNV-1a over the parts, with a 0 byte after each, as in the Go version.
fn lock_key(parts: &[&str]) -> i64 {
    let mut h: u64 = 0xcbf29ce484222325;
    for p in parts {
        for b in p.bytes().chain(std::iter::once(0u8)) {
            h ^= b as u64;
            h = h.wrapping_mul(0x100000001b3);
        }
    }
    h as i64
}

/// Records a charge unless one exists in the last 30 days.
async fn charge_once(client: &mut Client, customer: &str) -> Result<(), Error> {
    let tx = client.transaction().await?; // rolled back on drop unless committed
    tx.execute("SELECT pg_advisory_xact_lock($1)", &[&lock_key(&["charge", customer])])
        .await?;
    let exists: bool = tx
        .query_one(
            "SELECT EXISTS (SELECT 1 FROM charges
              WHERE customer = $1 AND window_start > now() - interval '30 days')",
            &[&customer],
        )
        .await?
        .get(0);
    if !exists {
        tx.execute(
            "INSERT INTO charges (customer, window_start) VALUES ($1, now())
             ON CONFLICT (customer, window_start) DO NOTHING",
            &[&customer],
        )
        .await?;
    }
    tx.commit().await
}

/// Moves a session to a terminal state; true only for the caller that did it.
async fn finish(client: &Client, id: i64, to: &str) -> Result<bool, Error> {
    let n = client
        .execute(
            "UPDATE sessions SET state = $2
              WHERE id = $1 AND state NOT IN ('completed', 'failed')",
            &[&id, &to],
        )
        .await?;
    Ok(n == 1)
}

fn is_unique_violation(e: &Error) -> bool {
    e.code() == Some(&SqlState::UNIQUE_VIOLATION)
}
```

</details>

<details><summary>C++</summary>

Written against libpqxx 7's documented API. **Not compiled**: libpqxx wasn't installed on
the machine I wrote this on, so treat it as a sketch and expect to fix small things.

```cpp
#include <cstdint>
#include <initializer_list>
#include <pqxx/pqxx>
#include <string>
#include <string_view>

// 64-bit FNV-1a over the parts, with a 0 byte after each, as in the Go version.
std::int64_t lock_key(std::initializer_list<std::string_view> parts) {
    std::uint64_t h = 0xcbf29ce484222325ULL;
    auto mix = [&h](unsigned char b) { h ^= b; h *= 0x100000001b3ULL; };
    for (auto p : parts) {
        for (unsigned char b : p) mix(b);
        mix(0);
    }
    return static_cast<std::int64_t>(h);
}

// Records a charge unless one exists in the last 30 days.
void charge_once(pqxx::connection& conn, const std::string& customer) {
    pqxx::work tx{conn};  // aborts in its destructor unless committed
    tx.exec_params("SELECT pg_advisory_xact_lock($1)", lock_key({"charge", customer}));
    bool exists = tx.exec_params1(
        "SELECT EXISTS (SELECT 1 FROM charges "
        " WHERE customer = $1 AND window_start > now() - interval '30 days')",
        customer)[0].as<bool>();
    if (!exists) {
        tx.exec_params(
            "INSERT INTO charges (customer, window_start) VALUES ($1, now()) "
            "ON CONFLICT (customer, window_start) DO NOTHING",
            customer);
    }
    tx.commit();
}

// Moves a session to a terminal state; true only for the caller that did it.
bool finish(pqxx::connection& conn, std::int64_t id, const std::string& to) {
    pqxx::work tx{conn};
    auto r = tx.exec_params(
        "UPDATE sessions SET state = $2 "
        " WHERE id = $1 AND state NOT IN ('completed', 'failed')",
        id, to);
    tx.commit();
    return r.affected_rows() == 1;
}

bool is_unique_violation(const pqxx::sql_error& e) { return e.sqlstate() == "23505"; }
```

</details>

<details><summary>JavaScript</summary>

node-postgres (`pg`) 8.16.0 on Node 26.13.0, run against the same PostgreSQL 17.11 with eight
concurrent calls: one charge row, one `finish` winner, and `isUniqueViolation` true for a
duplicate primary key. Note `pool.connect()`: a transaction has to run on one checked-out
client, which chapter 12 is about.

```js
const { Pool } = require('pg');

// 64-bit FNV-1a over the parts, with a 0 byte after each, as a signed bigint string.
function lockKey(...parts) {
  let h = 0xcbf29ce484222325n;
  for (const p of parts) {
    for (const b of [...Buffer.from(p), 0]) {
      h ^= BigInt(b);
      h = (h * 0x100000001b3n) & 0xffffffffffffffffn;
    }
  }
  return BigInt.asIntN(64, h).toString();
}

// Records a charge unless one exists in the last 30 days.
async function chargeOnce(pool, customer) {
  const client = await pool.connect(); // every statement below runs on this connection
  try {
    await client.query('BEGIN');
    await client.query('SELECT pg_advisory_xact_lock($1)', [lockKey('charge', customer)]);
    const { rows } = await client.query(
      `SELECT EXISTS (SELECT 1 FROM charges
         WHERE customer = $1 AND window_start > now() - interval '30 days') AS exists`,
      [customer]);
    if (!rows[0].exists) {
      await client.query(
        `INSERT INTO charges (customer, window_start) VALUES ($1, now())
         ON CONFLICT (customer, window_start) DO NOTHING`, [customer]);
    }
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// Moves a session to a terminal state; true only for the caller that did it.
async function finish(pool, id, to) {
  const res = await pool.query(
    `UPDATE sessions SET state = $2
      WHERE id = $1 AND state NOT IN ('completed', 'failed')`, [id, to]);
  return res.rowCount === 1;
}

const isUniqueViolation = (err) => err.code === '23505';
```

</details>

## Check yourself

1. A unique index is on `(customer, window_start)` and the insert sets `window_start = now()`.
   Two transactions for one customer start 50 µs apart. Walk through why `ON CONFLICT DO
   NOTHING` never fires, and say what would change if the key used
   `date_trunc('month', now())` instead.
2. With `pg_advisory_xact_lock` before the check, eight racers leave one row at Read
   Committed and eight at Repeatable Read. Explain the difference mechanically, naming the
   moment each transaction's snapshot is taken.
3. A rule says "at most three snapshots per (device, sensor)". Why can't a constraint hold
   it, and why doesn't `SELECT … FOR UPDATE` on the existing snapshots serialise two
   inserts into an *empty* slot? What does?
4. Your insert catches `23505` and treats it as "already recorded", then writes an audit row
   in the same transaction. What happens to the audit row, and what are two fixes?
5. `UPDATE sessions SET state = 'completed' WHERE id = $1` reports one row affected, and the
   caller bills on that. Describe the sequence that bills twice, and fix the statement.
6. You're asked to make login case-insensitive on a table that has never enforced it. Put
   the steps in order, and say what goes wrong if you switch the lookup to `lower(email)`
   first.
