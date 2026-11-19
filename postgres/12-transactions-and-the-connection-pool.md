# 12. Transactions and the connection pool

## The problem

A service starts timing out on database calls. `pg_stat_activity` shows connections
`idle in transaction`, the pool reports no free connections, and locks are held by sessions
that aren't doing anything. The usual first response is to raise the pool size, add retries
and lengthen timeouts. None of it helps, because the pool isn't too small. The application
is using it wrongly, in one of four ways this chapter takes in turn: a transaction spread
across connections, a timeout that leaks what it was waiting for, a second connection
requested while holding the first, and pool sizes that add up to more than the server allows.

Chapter 11's locks make all four worse, since a lock is held by a connection for as long as
its transaction lasts.

## 12.1 A transaction belongs to one connection

`BEGIN` doesn't start a transaction "for your code". It changes the state of one server
session, the backend process at the other end of one connection. Every statement until
`COMMIT` or `ROLLBACK` has to go down that same connection, or it isn't part of the
transaction.

A pool hides connections behind a convenience method. In node-postgres that's `pool.query`,
which borrows whichever client is free, runs one statement, and returns the client. So this
looks like a transaction and isn't:

```js
await pool.query('BEGIN');
await pool.query('INSERT INTO t VALUES (1)');
await pool.query('ROLLBACK');
```

The [node-postgres transactions page](https://node-postgres.com/features/transactions) says
it directly: "You must use the same client instance for all statements within a transaction.
This means if you initialize or use transactions with the `pool.query` method you will have
problems."

I ran those three lines with node-postgres 8.16.0 against PostgreSQL 17.11, a pool of three,
and one other query in flight at the same time:

| After the "rolled back" transaction | Result |
|---|---|
| Rows in `t`, seen from another session | **1**. The insert ran on a client with no `BEGIN`, so it autocommitted, and the `ROLLBACK` went elsewhere |
| Connections `idle in transaction` | **1**. The client that got `BEGIN` went back to the pool inside an open transaction |

The second line is the expensive one. The next caller to borrow that client runs inside a
transaction it didn't start: its writes aren't visible to anyone until something on that
client happens to commit, and any locks it takes stay held.

Two cheap defences on the server side, neither a fix:

- `idle_in_transaction_session_timeout` ends a session that sits idle inside a transaction
  longer than the setting. Its default is 0, disabled (`pg_settings.boot_val` on 17.11).
- `pg_stat_activity` with `state = 'idle in transaction'` and an old `xact_start` is how you
  find these.

A single statement doesn't need `BEGIN` at all. It's already atomic. Use the pool's
convenience method for single statements, and a checked-out connection for anything with
more than one.

## 12.2 Check out, begin, and roll back on every path

The shape is the same in every driver:

1. Check out one connection.
2. `BEGIN` on it.
3. Arrange for `ROLLBACK` on every exit path, *before* doing any work.
4. Do every statement through that connection.
5. `COMMIT`. Return the connection.

In Go, pgx makes step 3 one line, because rollback after commit is harmless. From the
[`Tx` interface](https://pkg.go.dev/github.com/jackc/pgx/v5#Tx):

> Rollback will return an error where errors.Is(ErrTxClosed) is true if the Tx is already
> closed, but is otherwise safe to call multiple times. Hence, a defer tx.Rollback() is safe
> even if tx.Commit() will be called first in a non-error condition. Any other failure of a
> real transaction will result in the connection being closed.

The last sentence is the pool's protection: a connection whose rollback failed is in an
unknown state, so pgx closes it rather than hand it to the next caller. Compiled with Go 1.24
and pgx v5.7.6, and run against PostgreSQL 17.11:

```go
// withTx runs fn inside one transaction on one pooled connection.
// fn must do all its database work through tx, never through pool.
func withTx(ctx context.Context, pool *pgxpool.Pool, fn func(pgx.Tx) error) error {
	tx, err := pool.Begin(ctx) // acquires a connection and sends BEGIN on it
	if err != nil {
		return err
	}
	// Runs on every return path. After a successful Commit it returns
	// ErrTxClosed and does nothing; after a failed Rollback pgx closes the
	// connection rather than return it to the pool in an unknown state.
	defer tx.Rollback(ctx)

	if err := fn(tx); err != nil {
		return err
	}
	return tx.Commit(ctx) // the connection goes back to the pool here
}

func newPool(ctx context.Context, url string, maxConns int32) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(url)
	if err != nil {
		return nil, err
	}
	cfg.MaxConns = maxConns // from the connection budget (§12.5), not a default
	cfg.MaxConnLifetime = 30 * time.Minute
	cfg.HealthCheckPeriod = time.Minute
	return pgxpool.NewWithConfig(ctx, cfg)
}
```

Inside `fn`, two `SELECT pg_backend_pid()` calls through `tx` returned the same process id,
and after `withTx` returned the pool had zero acquired connections.

<details><summary>Rust</summary>

deadpool-postgres over tokio-postgres. **Not compiled**: deadpool-postgres wasn't in my local
crate cache and I had no network for cargo, so treat it as a sketch. The `Transaction`
type is from tokio-postgres 0.7, which I did compile in chapter 11; it rolls back when
dropped without a commit, which covers step 3.

```rust
use deadpool_postgres::{Config, ManagerConfig, Pool, RecyclingMethod, Runtime, Timeouts};
use std::time::Duration;
use tokio_postgres::NoTls;

fn new_pool(max_size: usize) -> Pool {
    let mut cfg = Config::new();
    cfg.host = Some("127.0.0.1".into());
    cfg.user = Some("postgres".into());
    // The default, Fast, only checks Client::is_closed(); Verified also runs a test query.
    cfg.manager = Some(ManagerConfig { recycling_method: RecyclingMethod::Verified });
    let mut pool = deadpool_postgres::PoolConfig::new(max_size); // from the budget, §12.5
    // The default Timeouts has none set: a caller can wait for a slot for ever.
    pool.timeouts = Timeouts {
        wait: Some(Duration::from_secs(2)),
        create: Some(Duration::from_secs(2)),
        recycle: Some(Duration::from_secs(2)),
    };
    cfg.pool = Some(pool);
    cfg.create_pool(Some(Runtime::Tokio1), NoTls).expect("pool config")
}

async fn insert_two(pool: &Pool) -> Result<(), Box<dyn std::error::Error>> {
    let mut client = pool.get().await?; // one connection, returned to the pool on drop
    let tx = client.transaction().await?; // rolled back on drop unless committed
    tx.execute("INSERT INTO t VALUES (1)", &[]).await?;
    tx.execute("INSERT INTO t VALUES (2)", &[]).await?;
    tx.commit().await?;
    Ok(())
}
```

</details>

<details><summary>C++</summary>

libpqxx 7 has no pool of its own; a `pqxx::connection` is one session, and the transaction
object is tied to it. **Not compiled** (libpqxx wasn't installed where I wrote this).

```cpp
#include <pqxx/pqxx>

// The caller owns conn for the duration: check it out of whatever pool you use first,
// and give it back only after this returns.
void insert_two(pqxx::connection& conn) {
    pqxx::work tx{conn};      // BEGIN on this connection
    tx.exec("INSERT INTO t VALUES (1)");
    tx.exec("INSERT INTO t VALUES (2)");
    tx.commit();              // if anything above throws, ~work() rolls back
}
```

</details>

<details><summary>JavaScript</summary>

node-postgres 8.16.0 on Node 26.13.0, run against PostgreSQL 17.11. The pool docs
([Pool API](https://node-postgres.com/apis/pool)): "You must release a client when you are
finished with it", and `client.release(true)` destroys the client instead of returning it,
which is the right call when you don't know what state it's in.

```js
const { Pool } = require('pg');

const pool = new Pool({
  max: 10,                        // from the budget, §12.5 (10 is also the default)
  connectionTimeoutMillis: 2000,  // default 0: wait for ever
});

async function withTx(fn) {
  const client = await pool.connect();
  let broken = false;
  try {
    await client.query('BEGIN');
    const result = await fn(client); // every statement goes through this client
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try { await client.query('ROLLBACK'); } catch { broken = true; }
    throw err;
  } finally {
    client.release(broken); // true destroys a client whose state is unknown
  }
}
```

</details>

## 12.3 Timeouts belong to the pool

Waiting for a free connection needs a limit. A tempting way to add one is to race the
acquisition against a timer:

```js
const client = await Promise.race([pool.connect(), timeout(100)]);   // don't
```

When the timer wins, the caller gets an error and moves on. The `pool.connect()` is still
pending, though, and when a connection frees up it resolves with a client that nobody holds
and nobody will release. MDN's
[`Promise.race`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
is explicit: "Settling the returned promise does not cancel the losing operations".

Reproduced with a pool of one, its only client checked out, then a 100 ms race:

```
race: timed out
after release: total=1 idle=0 waiting=0          ← the slot is taken by a client nobody has
next connect: timeout exceeded when trying to connect
```

Each timeout leaks one slot for good. Add a retry wrapper that asks again straight away and
the pool drains faster under exactly the load that caused the first timeout.

The fix is to use the pool's own timeout, which removes the waiter from the queue when it
fires. In node-postgres that's `connectionTimeoutMillis` (its default, 0, means no timeout).
In pgx the acquisition takes a `context.Context`, so a deadline on the context bounds the wait.
In deadpool it's `Timeouts { wait, create, recycle }`, and the
[default](https://docs.rs/deadpool/latest/deadpool/managed/struct.Timeouts.html) is
"an empty Timeouts config (no timeouts set)".

> **Teacher's aside.** The general rule is wider than databases: a timeout that only stops
> *waiting for* an operation doesn't stop the operation. Whatever you were acquiring, a
> connection, a lock, a file handle, still arrives, and something has to own it. Either the
> thing you're waiting on supports cancellation, or the code that gives up has to arrange
> for the late result to be cleaned up.

## 12.4 What the pool checks when a connection comes back

A connection returned to the pool may be dead (the server restarted, a load balancer dropped
an idle socket) or dirty (a session setting changed, a temp table left behind). Pools differ
in how much they check before handing it out again. deadpool-postgres's
[`RecyclingMethod`](https://docs.rs/deadpool-postgres/latest/deadpool_postgres/enum.RecyclingMethod.html):

| Method | What it does on recycle |
|---|---|
| `Fast` (the default) | "Only run Client::is_closed() when recycling existing connections." The docs: it "does not check the connection health or perform any clean-up queries" |
| `Verified` | `is_closed()` and a test query. "Slower, but guarantees that the database connection is ready to be used" |
| `Clean` | Like `Verified`, but runs a reset sequence (`CLOSE ALL`, `DISCARD SEQUENCES` and more) for "a pristine connection" |
| `Custom(sql)` | Like `Verified`, with your query |

pgx's pool checks idle connections in the background every `HealthCheckPeriod` and retires
them after `MaxConnLifetime`. node-postgres discards idle clients after `idleTimeoutMillis`
(default 10 s) and has an optional `maxLifetimeSeconds` (default disabled). If your code
changes session state with `SET`, either undo it, use `SET LOCAL` inside a transaction, or
pick a recycling method that resets it.

## 12.5 Never ask for a second connection while holding the first

The fourth way is a deadlock inside the application. A transaction holds connection A and,
part-way through, calls a helper that does its own read through the pool, which needs
connection B. With N concurrent transactions on a pool of N, every connection is held by a
transaction waiting for a second one. Nobody can proceed until something times out.

It gets much more likely with chapter 11's locks. A transaction that waits for an advisory
lock still holds its connection while it waits, so the lock queue fills the pool, and the
lock holder then can't get the second connection it needs to finish.
[api-orchestration file 12 §12.5](../api-orchestration/12-retiring-what-others-still-use.md#125-a-lock-a-pool-and-a-jam)
measures exactly this: with a pool of four, three concurrent callers all finish, and at four
none do.

The fix is structural: pass the transaction (`pgx.Tx`, the checked-out client) into every
function that must read inside it, and make "takes a pool" and "takes a transaction"
different types, so a helper can't silently borrow a second connection.

## 12.6 Pool sizes add up

The server has one budget, `max_connections`, default 100, of which
`superuser_reserved_connections`, default 3, are kept back (both from `pg_settings.boot_val`
on PostgreSQL 17.11). Every pool in every process draws on it:

```
connections the server may see =
    replicas  ×  processes per replica  ×  pool max per process
  + migrations, cron jobs, consoles, monitoring
```

Defaults make this easy to overshoot without noticing:

| Driver | Default pool size |
|---|---|
| pgxpool | "The default is the greater of 4 or runtime.NumCPU()" ([pool.go](https://github.com/jackc/pgx/blob/v5.7.6/pgxpool/pool.go), the `MaxConns` comment) |
| node-postgres | 10 ([Pool API](https://node-postgres.com/apis/pool)) |
| deadpool | Set by you in `PoolConfig::new(max_size)` |

A Node service that forks one worker per CPU with a pool of 10 each, on a 16-CPU machine,
asks for 160 connections from one replica. A default derived from CPU count also changes
when the same image is scheduled on a bigger node. So set the pool size explicitly from the
budget: what's left after the reserved slots and the non-service users, divided by every
process that holds a pool at peak, including during a rolling deploy when old and new
replicas overlap.

When the arithmetic doesn't fit, a server-side pooler such as
[PgBouncer](https://www.pgbouncer.org/features.html) multiplexes many client connections onto
fewer server connections. In transaction mode "a server connection is assigned to a client
only during a transaction", which is why a transaction must be one connection end to end
(§12.1), and why session state doesn't survive between transactions. Chapter 8's
"Who prepares your statements" covers what that does to prepared statements.

## Check yourself

1. A helper runs `pool.query('BEGIN')`, two inserts and `pool.query('COMMIT')` through
   node-postgres. Under light load in tests it seems to work. Describe two different things
   that can happen under concurrent load, and what each leaves behind in the pool.
2. Why is `defer tx.Rollback(ctx)` placed immediately after `Begin` rather than in each error
   branch, and what does pgx do with the connection if that rollback itself fails?
3. A colleague adds a 100 ms timeout around `pool.connect()` with `Promise.race`, plus three
   immediate retries. Under a burst the pool stays exhausted after the burst ends. Explain
   the mechanism, slot by slot.
4. A publish handler holds `pg_advisory_xact_lock` on connection A and calls a validation
   helper that takes the pool. Pool size 8. At what concurrency can it deadlock, and why
   doesn't raising the pool to 16 fix it?
5. Four replicas, each running 4 worker processes with a pool of 10, plus a migration job
   and a monitoring agent, against the default `max_connections`. Do the arithmetic, say what
   fails first and how, and give two ways to make it fit.
