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
   report. Explain, and fix it so it can still use an index.
