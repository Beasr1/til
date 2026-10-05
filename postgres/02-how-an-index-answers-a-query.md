# 2. How an index answers a query

## The problem

You add an index, the query is still slow, and `EXPLAIN` shows a sequential scan. Or
worse, the index is used and the query is *still* slow. Both happen because an index
isn't a generic accelerator. It is a data structure that can answer some questions
cheaply and others not at all, and which questions it can answer depends on its shape.
That shape is set by three choices: the index type, the operator class, and the column
order.

This chapter works through one search box, the kind every admin dashboard has: a
user types part of an identifier and expects matching records. The obvious index fails
twice before the right one turns up, and each failure teaches one of those three
choices.

## A btree is a sorted list

A **btree** index keeps its keys in sorted order, in a tree shallow enough that finding
any key costs a handful of page reads. Everything a btree can do follows from "sorted":

| Question | Why a sorted list can answer it |
|---|---|
| `x = 'abc'` | Binary search to the key |
| `x < 'abc'`, `x BETWEEN …` | Find one end, walk until the other |
| `x LIKE 'abc%'` | Rewritten as a range: `x >= 'abc' AND x < 'abd'` |
| `x LIKE '%abc'` | Can't. Values ending in `abc` are scattered all through the sort order |
| `x LIKE '%abc%'` | Can't, for the same reason |

The PostgreSQL manual states the pattern-matching rule exactly ([§11.2](https://www.postgresql.org/docs/current/indexes-types.html)):

> The optimizer can also use a B-tree index for queries involving the pattern matching
> operators `LIKE` and `~` *if* the pattern is a constant and is anchored to the
> beginning of the string — for example, `col LIKE 'foo%'` or `col ~ '^foo'`, but not
> `col LIKE '%bar'`.

You can watch the rewrite happen. With a suitable index, a prefix search becomes two
range conditions in the plan:

```
Index Cond: (... (ref ~>=~ '10000977') AND (ref ~<~ '10000978'))
Filter:     (ref ~~ '10000977%')
```

`~>=~` and `~<~` are the character-by-character comparison operators. The original
`LIKE` (`~~`) stays as a recheck filter, which is cheap because the range has already
narrowed things down to a few rows.

## Why the obvious btree didn't work: collation

Here is the trap. You create a plain btree on the column and a prefix `LIKE` still
does a sequential scan.

The reason is **collation**. A database created with a locale such as `en_US.UTF-8`
sorts text by language rules, not byte values. Under those rules, "everything that
starts with `abc`" is not guaranteed to be one contiguous run of the index, so the
range rewrite above would be wrong, and PostgreSQL refuses to do it.

The fix is an **operator class**: an index-time choice of which comparison the index
is sorted by. `text_pattern_ops` sorts strictly character by character. The manual
([§11.10](https://www.postgresql.org/docs/current/indexes-opclass.html)):

> The difference from the default operator classes is that the values are compared
> strictly character by character rather than according to the locale-specific
> collation rules. This makes these operator classes suitable for use by queries
> involving pattern matching expressions (`LIKE` or POSIX regular expressions) when the
> database does not use the standard "C" locale.

It comes with a price, stated in the same section:

> Note that you should also create an index with the default operator class if you
> want queries involving ordinary `<`, `<=`, `>`, or `>=` comparisons to use an index.
> Such queries cannot use the `xxx_pattern_ops` operator classes. (Ordinary equality
> comparisons can use these operator classes, however.)

So a `text_pattern_ops` index serves `=` and anchored `LIKE`, and nothing that needs
real ordering. If the database uses the C locale, none of this applies and the default
operator class already works.

> **Teacher's aside.** People say "`ILIKE` can't use a btree". The truth is narrower,
> and the manual spells it out: a btree can serve `ILIKE` "only if the pattern starts
> with non-alphabetic characters, i.e., characters that are not affected by upper/lower
> case conversion." A search on a numeric identifier is all digits, so
> `ref ILIKE '1000%'` produces the *same* range scan as `LIKE`. Planners extract the
> longest case-insensitive-safe prefix. Use `LIKE` anyway when case can't matter,
> because it states the intent and doesn't depend on what the user typed. But if a
> comment in your code says `ILIKE` makes the index unusable, it's only true for
> patterns that start with a letter.

## The second attempt: trigrams

Now the product asks for *contains* search: type any part of the identifier, find
the record. A btree can't do `'%abc%'`, so the standard answer is the `pg_trgm`
extension and a **GIN** index.

A **trigram** is a run of three consecutive characters. `pg_trgm` breaks each value
into its trigrams, and a GIN index (an **inverted index**: one entry per component
value, pointing at every row that contains it) maps each trigram to the rows that
contain it. The manual's example ([pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)):

> the set of trigrams in the string "`cat`" is "`  c`", "` ca`", "`cat`", and "`at `".

To answer `LIKE '%abcdef%'`, PostgreSQL extracts the pattern's trigrams, fetches each
one's **posting list** (the list of rows containing it), intersects them, then rechecks
the candidates against the real pattern. `gin_trgm_ops` supports `LIKE`, `ILIKE`,
regex and `=`. That flexibility is what makes it the default recommendation.

It worked in development. It does not scale for this data, and why it doesn't is the
point of the chapter.

### Why trigrams are bad at digits

Trigram indexes are good at natural-language text because the alphabet is large and
most trigrams are rare. A rare trigram has a short posting list, so intersecting a few
of them is cheap.

An identifier made only of digits has an alphabet of ten. There are only 1,000
possible all-digit trigrams. Every row contributes roughly one trigram per character, so
each trigram appears in a large and growing share of the table. Every posting list is
long, and a full-length search has to read about fifteen of them.

A back-of-envelope estimate, not a measurement: a table of *N* fifteen-digit values
holds about 15*N* trigram entries spread over ~1,000 distinct trigrams, so the average
posting list is about 15*N* / 1,000 rows long. Doubling the table doubles the work of
every search. A btree lookup, by contrast, grows with the logarithm of *N*.

Measured on a 300,000-row test table, looking up one complete 15-digit value:

| Index | Plan | Buffers read | Time |
|---|---|---|---|
| btree `(org, status, ref text_pattern_ops)`, `LIKE 'x%'` | Index Scan | 4 | 0.01 ms |
| GIN `gin_trgm_ops`, `ILIKE '%x%'` | Bitmap Index Scan + recheck | 320 | 4.7 ms |

At 300,000 rows the difference is invisible to a user. Two orders of magnitude, scaling
linearly, is visible at tens of millions.

### The second reason: an inverted index can't be scoped

Look at the btree in that table. Its leading columns are the tenant and the status, so
a search inside one tenant walks one small corner of the index. A multi-tenant
dashboard always filters by tenant.

The GIN index can't do that. It answers "which rows contain this trigram?" for the
whole table, and only then can the tenant filter throw most of them away. The plan
shows it plainly: the tenant condition appears as a `Filter` on the heap scan, applied
after the index has done all its work.

```
Bitmap Heap Scan on t
  Recheck Cond: (ref ~~* '%100000396005433%')
  Filter: ((org = 7) AND (status = 'COMPLETED'))      ← applied after the index
  ->  Bitmap Index Scan on t_trgm
        Index Cond: (ref ~~* '%100000396005433%')
```

(The `btree_gin` extension can put scalar columns into a GIN index, which helps in some
cases. It doesn't change the posting-list arithmetic above.)

## The third attempt: change the question

The fix that worked was not a better index. It was a narrower feature: **prefix**
search instead of contains search, on a btree whose leading columns match the
dashboard's mandatory filters.

```sql
CREATE INDEX ... ON transactions
    (tenant_id, status, (doc -> 'info' ->> 'ref') text_pattern_ops);
```

Nobody searching an identifier types its middle. They type the start, or paste the
whole thing. Contains search was a feature nobody used, and it was dictating the most
expensive index in the database.

> ⚠️ **The trap: the expression index's parentheses.** An index on an expression needs
> the expression in its own parentheses, and the operator class goes *outside* them.
> From the synopsis in [CREATE INDEX](https://www.postgresql.org/docs/current/sql-createindex.html):
> `( { column_name | ( expression ) } [ COLLATE collation ] [ opclass ] ... )`.
>
> ```sql
> (tenant_id, ((doc ->> 'ref') text_pattern_ops))   -- syntax error
> (tenant_id, (doc ->> 'ref') text_pattern_ops)     -- correct
> ```
>
> The first wraps the opclass inside the expression and fails with
> `syntax error at or near "text_pattern_ops"`. It's an easy slip when converting a
> single-column expression index, which needs double parentheses, into a multi-column one.

## Indexing inside a JSON document

All of the above was on a value extracted from a `jsonb` column. That works: an
**expression index** indexes the result of the expression, and the planner uses it
for any query containing the *identical* expression.

The cost is in the word "identical". If one writer stores the identifier at
`doc -> 'info' ->> 'ref'` and a newer writer stores it at
`doc -> 'detail' ->> 'ref'`, they are two different expressions. One index serves only
one of them, and a query reading only one path silently finds nothing in documents
written by the other. A schema that drifts inside a JSON document needs an index, and
a query branch, per path. That's usually the moment to promote the field to a real
column.

## Choosing, in one table

| You need | Index |
|---|---|
| Equality only | Default btree |
| Equality and anchored `LIKE`, non-C locale | btree with `text_pattern_ops`, plus a default btree if you also need `<`/`>` |
| Contains / fuzzy on natural-language text | GIN or GiST with `pg_trgm` |
| Contains on a small-alphabet identifier at scale | Reconsider the feature. Posting lists grow linearly |
| Any of the above inside one tenant | Put the tenant column first in a btree |

## Check yourself

1. A prefix `LIKE 'abc%'` does a sequential scan despite a plain btree on the column.
   What's the one-line diagnosis, and what two fixes are available?
2. Why can a btree serve `ILIKE '100%'` but not `ILIKE 'abc%'`?
3. You have a `text_pattern_ops` index on `name`. Which of these can use it:
   `name = 'x'`, `name LIKE 'x%'`, `name > 'x'`, `ORDER BY name`?
4. Explain, without reference to measurements, why a trigram index on a column of
   phone numbers gets slower per query as the table grows, while a trigram index on
   free-text comments degrades much more slowly.
5. A multi-tenant table has a GIN trigram index and a btree on `tenant_id`. A search
   filters on both. Describe the two plans the planner might choose, and what each costs.
6. Two application versions store the same field at two JSON paths. You add an
   expression index on the new path. What happens to search results for documents
   written by the old version, and how would you notice?
