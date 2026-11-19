# 8. Parameters, plans and types

## The problem

You test a query in `psql` with literal values and it's fast. The application runs the
same query with `$1` and `$2` and it's slow, or it errors with a type message that makes
no sense for the code you wrote. The difference is that a parameterised query is
planned and typed *without seeing the values*, at least some of the time, and both the
planner and the type system behave differently when they can't see them.

## Who prepares your statements

You may not think you use prepared statements. Your driver might. pgx, the common Go
driver, uses the extended protocol and caches prepared statements by default. Its
source says: "By default pgx uses the extended protocol and automatically prepares and
caches prepared statements." Many other drivers and ORMs do the same, and connection
poolers such as PgBouncer in transaction mode are a known reason to switch it off.

So the rules for prepared statements apply to ordinary application queries.

### Behind a transaction-mode pooler

A driver has two ways to send a query, defined by the
[protocol](https://www.postgresql.org/docs/current/protocol-flow.html):

| | Simple query | Extended query |
|---|---|---|
| Messages | One `Query` message holding the SQL text | `Parse`, `Bind`, `Execute` (and `Describe`), then `Sync` |
| Parameters | None. A driver that "supports" them interpolates values into the text, quoted and escaped | Sent separately from the SQL, never spliced into it |
| Several statements in one string | Allowed, run as one transaction | Not allowed: "The query string contained in a Parse message cannot include more than one SQL statement" |
| Prepared statement | None | A named one, or the unnamed one |

The last row is where poolers come in. From the same page: "a named prepared-statement object
lasts till the end of the current session", while "an unnamed prepared statement lasts only
until the next Parse statement specifying the unnamed statement as destination is issued".
A named statement is state on one server connection.

A transaction-mode pooler such as PgBouncer breaks that assumption on purpose. Its
[features page](https://www.pgbouncer.org/features.html): "A server connection is assigned to
a client only during a transaction." Your next transaction may run on a different server
connection, one that never saw your `PREPARE`. The same page's compatibility table lists
what can't survive that in transaction mode, among them `SET`/`RESET`, `LISTEN`, `WITH HOLD`
cursors, SQL-level `PREPARE`/`DEALLOCATE` and session-level advisory locks (one more reason
chapter 11 uses the transaction-level kind). Protocol-level prepared statements are the
exception: PgBouncer supports them in transaction mode once its `max_prepared_statements`
setting is non-zero.

So pgx's default, which names and caches a statement per connection, fails behind a pooler
that doesn't track prepared statements, with errors like `prepared statement "…" does not
exist` (SQLSTATE `26000`) or `… already exists`. pgx's
[`QueryExecMode`](https://pkg.go.dev/github.com/jackc/pgx/v5#QueryExecMode) options, from its
source comments (v5.7.6):

| Mode | Protocol | Named statements? | From the docs |
|---|---|---|---|
| `CacheStatement` | Extended | Yes, cached per connection | "This is the default" |
| `CacheDescribe` | Extended | No, but caches parameter and result types | The first run after a schema change "may fail" |
| `DescribeExec` | Extended, two round trips | No | "May cause problems with connection poolers that switch the underlying connection between round trips" |
| `Exec` | Extended, one round trip | No, unnamed only | Text-format parameters, types assumed from the Go types |
| `SimpleProtocol` | Simple | No | "Client side parameter interpolation. All values are quoted and escaped" |

The tempting fix is the simple protocol, because it obviously has no prepared statements.
pgx's own guidance is the opposite: "Prefer QueryExecModeExec over QueryExecModeSimpleProtocol
whenever possible. In general QueryExecModeSimpleProtocol should only be used if connecting
to a proxy server, connection pool server, or non-PostgreSQL server that does not support the
extended protocol." Exec mode keeps values out of the SQL text and still works through a
transaction-mode pooler, because the unnamed statement is created and used within one round
trip. What you give up is plan caching, and the generic-plan switch below goes with it.

Two smaller things worth knowing:

- **No parameters can mean no prepare.** pgx sends a query with zero arguments over the
  simple protocol whatever the mode ("Always use simple protocol when there are no
  arguments", a comment in its `conn.go`). If you write your own driver layer, treat an
  empty parameter list the same as a missing one: it saves a round trip, and the simple
  protocol is the only way to send several statements in one string.
- **Roll a pooler out one service at a time.** It's a behaviour change for every client
  that relies on session state, and you won't know which ones do until they break. Pointing
  a shared connection setting at it moves every service at once, so one incompatible client
  blocks everyone. Give each service its own override, move them one by one, and size each
  service's pool below the pooler's own limit (chapter 12 §12.6).

## Custom plans and generic plans

From [PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html):

> A generic plan is the same across all executions, while a custom plan is generated for
> a specific execution using the parameter values given in that call.

And how the server picks between them:

> The current rule for this is that the first five executions are done with custom plans
> and the average estimated cost of those plans is calculated. Then a generic plan is
> created and its estimated cost is compared to the average custom-plan cost. Subsequent
> executions use the generic plan if its cost is not so much higher than the average
> custom-plan cost as to make repeated replanning seem preferable.

So a query can be planned with real values for its first five runs on a connection and
then switch to a plan that sees only `$1`. A plan that depends on the *value* can
change on the sixth run.

## Partial indexes need the value

A partial index covers only rows matching its predicate:

```sql
CREATE INDEX orders_failed ON orders (tenant_id) WHERE status = 'FAILED';
```

The manual on when it can be used
([§11.8](https://www.postgresql.org/docs/current/indexes-partial.html)):

> Matching takes place at query planning time, not at run time. As a result,
> parameterized query clauses do not work with a partial index. For example a prepared
> query with a parameter might specify "x < ?" which will never imply "x < 2" for all
> possible values of the parameter.

That's slightly stronger than what you see in practice, because of custom plans. On
PostgreSQL 17, a prepared `WHERE tenant_id = 7 AND status = $1` executed with
`'FAILED'`:

| `plan_cache_mode` | Plan |
|---|---|
| `force_custom_plan` | Index Only Scan using `orders_failed` |
| `force_generic_plan` | A different index, with `status = $1` as a condition |
| Generic, with the literal `'FAILED'` written into the SQL | Index Only Scan using `orders_failed` |

So the partial index works for the first few executions and stops working once the
connection settles on a generic plan. That's the worst kind of performance bug: fine in
every test, slow in a long-running service.

Two ways out:

- Write the predicate as a **literal** in the SQL when it's always the same value. The
  query string is then provably within the index's predicate.
- Make the column an ordinary **index column** instead of a predicate:
  `(tenant_id, status)`. It's bigger but works for any bound value.

## Parameter types are inferred, and sometimes wrongly

A parameter with no declared type is typed from context. PREPARE again:

> When a parameter's data type is not specified or is declared as `unknown`, the type is
> inferred from the context in which the parameter is first referenced (if possible).

The rule for operators ([§10.2](https://www.postgresql.org/docs/current/typeconv-oper.html)):

> If one argument of a binary operator invocation is of the `unknown` type, then assume
> it is the same type as the other argument for this check.

Follow that rule through a common date filter, "created before the end of the day the
user picked":

```sql
WHERE created_at < $1 + interval '1 day'
```

`$1` is the left operand of `+`, the other operand is an `interval`, so `$1` is typed
as an **interval**. `interval + interval` is an interval. Then the comparison is
`timestamptz < interval`, which doesn't exist:

```
ERROR:  operator does not exist: timestamp with time zone < interval
HINT:  No operator matches the given name and argument types.
       You might need to add explicit type casts.
```

The error points at the `<`, but the cause is the `+`, one step earlier, where the
parameter got its type. The fix is to say what `$1` is:

```sql
WHERE created_at < $1::timestamptz + interval '1 day'
```

> **Teacher's aside.** "Add a cast" can look like noise, a workaround for a fussy
> database. It isn't. An untyped parameter is a hole in the query, and PostgreSQL fills
> it by local rules that don't know what you meant. Casting parameters at the point of
> first use makes the query mean one thing regardless of driver. Some drivers send typed
> parameters and hide the problem, and switching drivers or execution modes uncovers it.

## The end-of-day filter itself

The same line holds a second lesson. A user picks an end *date*. Is it inclusive?

| Condition | Includes 23:59:59 on the end date? |
|---|---|
| `created_at <= $end` with `$end` a date (midnight) | No. Only rows at exactly 00:00:00 |
| `created_at < $end + interval '1 day'` | Yes |
| `created_at::date <= $end` | Yes, but defeats an index on `created_at` |

The half-open form, `>= start AND < end + 1 day`, is the one that's both correct and
indexable. And "midnight" is in a time zone: a `timestamptz` parameter built from a bare
date gets the session's `TimeZone`, which may not be the user's.

### Whose midnight

A report for a user in New York covering 1 June 2024 means `[1 June 00:00 New York, 2 June
00:00 New York)`, which is `[1 June 04:00 UTC, 2 June 04:00 UTC)` while daylight saving is
in force. Compute the two boundaries in the
user's zone and only then convert to UTC. In SQL, turn the date into a wall-clock
`timestamp` first, then say which zone that wall clock is in:

```sql
WHERE created_at >= ($start::date)::timestamp AT TIME ZONE $tz
  AND created_at <  ($end::date + 1)::timestamp AT TIME ZONE $tz
```

The cast to `timestamp` matters. On PostgreSQL 17.11 with the session in `Europe/London`,
`'2024-06-01'::date AT TIME ZONE 'America/New_York'` gave `2024-05-31 19:00:00`: the date was
first read as London midnight, then shown as New York wall-clock time, the opposite
direction. `'2024-06-01'::date::timestamp AT TIME ZONE 'America/New_York'` gave 1 June 04:00
UTC, the right boundary. Pass the zone by name (`'America/New_York'`), not as a fixed offset,
so dates either side of a daylight-saving change each get their own offset.

The client side has its own trap. In JavaScript, `new Date('2024-06-01')` is UTC midnight,
but `new Date('2024-06-01T00:00')` is local midnight. MDN
([Date time string format](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#date_time_string_format)):
"date-only forms are interpreted as a UTC time and date-time forms are interpreted as a local
time", which it calls "a historical spec error". With `TZ=America/New_York`, Node 26 printed
`2024-06-01T00:00:00.000Z` for the first and `2024-06-01T04:00:00.000Z` for the second. A
date picker's value sent through `new Date(…)` is a different instant depending on which form
it happens to be in. Send the date as a date string plus the user's zone name, and let one
place, ideally the query, do the conversion.

### Months aren't 30 days

Billing windows and subscriptions add "one month". Converting that to days first, because
days are easy to count, is wrong twice a year at least. PostgreSQL keeps the units apart. From
[§8.5.4](https://www.postgresql.org/docs/current/datatype-datetime.html#DATATYPE-INTERVAL-INPUT):

> Internally, `interval` values are stored as three integral fields: months, days, and
> microseconds. These fields are kept separate because the number of days in a month varies,
> while a day can have 23 or 25 hours if a daylight savings time transition is involved.

What that buys, on PostgreSQL 17.11:

| Expression | Result |
|---|---|
| `'2024-01-31 10:00' + interval '1 month'` | 2024-02-29 10:00 (clamped to the month's last day) |
| `'2024-01-31 10:00' + interval '30 days'` | 2024-03-01 10:00 |
| `'2025-01-31 10:00' + interval '1 month'` | 2025-02-28 10:00 |
| `'2024-02-29 10:00' + interval '1 year'` | 2025-02-28 10:00 |
| `('2024-01-31' + interval '1 month') + interval '1 month'` | 2024-03-29: the clamp carries forward |
| `'2024-01-31' + interval '2 months'` | 2024-03-31 |
| `'2024-03-30 12:00' + interval '1 day'`, London | 2024-03-31 12:00 (23 hours later, across the DST change) |
| `'2024-03-30 12:00' + interval '24 hours'`, London | 2024-03-31 13:00 |

Two consequences. Compute the nth billing date from the anchor, `anchor + n * interval '1
month'`, not by adding a month to the previous date, or a 31st-of-the-month subscription
drifts to the 28th and stays there. And store a plan's period as an `interval` (`'1 month'`,
`'1 year'`) rather than a number of days, so the arithmetic stays symbolic until it meets a
real date.

> ⚠️ **Equal isn't the same.** `interval '1 month' = interval '30 days'` is **true** (checked
> on 17.11), because comparison treats a month as 30 days, the same assumption the manual
> describes for fractional input. Yet adding them to 31 January gives different dates. Never
> use interval equality to decide whether two periods are "the same plan".

## Check yourself

1. A query using a partial index is fast in tests and becomes slow after the service
   has been up a few minutes. Give the mechanism, citing what changes between the fifth
   and sixth execution.
2. Why does writing the predicate value as a literal let the generic plan use the
   partial index, while binding the same value as `$1` doesn't?
3. In `WHERE created_at < $1 + interval '1 day'`, which operator gives `$1` its type,
   and why does the error message point somewhere else?
4. Would `WHERE created_at - interval '1 day' < $1` have the same problem? Work it out
   from the rule.
5. A filter is `created_at BETWEEN $start AND $end` with both bound as dates. A
   customer says yesterday's records are missing from a "yesterday to yesterday"
   report. Explain, and fix it so it can still use an index. Then two follow-ups: the
   customer is in New York in June (UTC−4) and the session runs in UTC, so what's still
   wrong with your fix?
   And the monthly version of the report ends its window at `$start + interval '30 days'`;
   for which months is that wrong, and what should it say?
6. A service moves behind a transaction-mode pooler and starts failing with `prepared
   statement "…" does not exist`. One engineer proposes the simple protocol, another pgx's
   `Exec` mode. Explain the error from what a named statement is attached to, and argue
   which mode to pick and what each costs.
