# MongoDB in Production — A Course

A course on the part of MongoDB you meet once a collection is big, shared and busy: what a
unique index really enforces when fields can be missing or `null`, how to change an index
without a window of no protection, how a query can leak across tenants, why a sort key that
isn't on every document breaks paging, and why a join in an aggregation can read the whole
history.

**This is reference learning material.** Everything here is general MongoDB, checked against
the [MongoDB manual](https://www.mongodb.com/docs/manual/) and, where the manual is silent or
simplified, reproduced on MongoDB 8.0 in a throwaway container, with the output shown. The
motivating failures are ordinary ones: a "unique when present" index that collided on stored
nulls, an index sync that dropped a constraint and failed to build its replacement, a lookup
by alternative keys that matched every record in a tenant, a cursor that billed some records
twice, and a dashboard funnel whose cost grew with all of history.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure that forced each rule comes before the rule.
- Every chapter ends with **Check yourself** questions. Answers are in
  [08-exercises.md](08-exercises.md).
- Every behavioural claim is quoted from the manual or reproduced, and I say which.
- Queries are in mongosh syntax. Application code is Go, with Rust, C++ and JavaScript
  versions in collapsible blocks, and each says what was compiled and run.

## The one thing to understand first

> **A field in MongoDB has three states (a value, `null`, or missing), and every feature
> treats them slightly differently.** An ordinary index stores missing as `null`. A sparse
> index skips missing but keeps `null`. `$exists: true` matches `null`. A sort puts missing
> and `null` together, below everything else. A comparison ignores both. Which state you
> store is decided by whichever driver or ODM wrote the document.

Most surprises in this course are a case of that, or of its sibling: something you know
(the field is always set, the tenant filter is always applied, the join only filters) that
the query, as written, doesn't say.

## Reference implementations

| Source | What it settles |
|---|---|
| [Unique Indexes](https://www.mongodb.com/docs/manual/core/index-unique/) | Missing stored as `null`; compound uniqueness is per combination; multikey uniqueness is between documents |
| [Partial Indexes](https://www.mongodb.com/docs/manual/core/index-partial/) | Allowed filter operators; unique partial constraints; when the planner may use one |
| [Sparse Indexes](https://www.mongodb.com/docs/manual/core/index-sparse/) | Sparse keeps `null`; compound sparse includes any document with one key |
| [Multikey Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/) | One key per array element |
| [$exists](https://www.mongodb.com/docs/manual/reference/operator/query/exists/) / [$type](https://www.mongodb.com/docs/manual/reference/operator/query/type/) / [$eq](https://www.mongodb.com/docs/manual/reference/operator/query/eq/) | What each matches, including `null` and arrays; `$eq` for user input |
| [Index Builds on Populated Collections](https://www.mongodb.com/docs/manual/core/index-creation/) | Build locks, unique violations at commit, commit quorum |
| [Rolling Index Builds on Replica Sets](https://www.mongodb.com/docs/manual/tutorial/build-indexes-on-replica-sets/) | When and how to build one member at a time |
| [The ESR Guideline](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/) | Compound index field order |
| [$or](https://www.mongodb.com/docs/manual/reference/operator/query/or/) / [$and](https://www.mongodb.com/docs/manual/reference/operator/query/and/) | Implicit AND of top-level conditions; index use per `$or` clause |
| [Comparison/Sort Order](https://www.mongodb.com/docs/manual/reference/bson-type-comparison-order/) | BSON type order; missing sorts as `null`; arrays in sorts |
| [cursor.sort()](https://www.mongodb.com/docs/manual/reference/method/cursor.sort/) / [$skip](https://www.mongodb.com/docs/manual/reference/operator/aggregation/skip/) | Sort consistency and skipping |
| [Aggregation Pipeline Optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/) | Which stages use indexes; what the optimiser reorders and coalesces |
| [$lookup](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/) / [$unwind](https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/) | Join forms, index use in sub-pipelines, row multiplication |
| [Mongoose: guide](https://mongoosejs.com/docs/guide.html), [Model API](https://mongoosejs.com/docs/api/model.html), [middleware](https://mongoosejs.com/docs/middleware.html), [`sanitizeFilter`](https://mongoosejs.com/docs/api/mongoose.html) | `autoIndex`, `syncIndexes()` drops, which writes skip save hooks, filter sanitising |
| [OWASP WSTG: Testing for NoSQL Injection](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection) / [Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) | Injection testing; connection strings out of logs |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What MongoDB does with a query](01-what-mongodb-does-with-a-query.md) | ⭐ Explain documents, BSON types, missing versus `null`, filters and implicit AND, what an index holds, per-document atomicity, and read `explain("executionStats")`. **Prerequisite for everything else** |

### Part 1 — Indexes and constraints

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Uniqueness](02-uniqueness.md) | ⭐ Write a "unique when present" index that survives stored nulls, enforce uniqueness across several fields with a derived key array, and replace a check-then-insert with an index |
| 3 | [Changing indexes on live collections](03-changing-indexes-on-live-collections.md) | ⭐ Say what an index build does to traffic, keep index changes out of every replica's boot, tighten a unique constraint with no unprotected window, and order a compound index by ESR |

### Part 2 — Reading data

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Queries that leak](04-queries-that-leak.md) | Keep tenant and soft-delete scopes outside `$or`, and stop untrusted input from turning into query operators without banning ordinary characters |
| 5 | [Sorting and paging](05-sorting-and-paging.md) | Choose a keyset sort key that's on every document, write limit+1 paging with a tiebreaker, and put an offset fallback after the sort |
| 6 | [Aggregations and joins](06-aggregations-and-joins.md) | Drive a pipeline from an indexed `$match`, and check existence through a join without multiplying rows |

### Reference

| # | File | |
|---|------|---|
| 7 | [Glossary](07-glossary.md) | Terms, each linked to its chapter |
| 8 | [Exercises & answers](08-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. Chapter 3 builds on chapter 2. Chapters 4, 5 and 6 are independent of
Part 1 and of each other, though chapter 6 reuses chapter 5's point about stage order. Not
yet written, and belonging here: schema design and embedding versus referencing,
transactions and read/write concerns, change streams, sharding and shard keys, and TTL
indexes. The questions at the end of file 08 are the honest list.

## If you're short on time

- **New to MongoDB:** chapter 1, all of it. Everything else assumes the three states of a
  field.
- **10 minutes before adding a unique index:** chapter 2, the sparse/`$exists`/`$type` table,
  then chapter 3, "Tightening a constraint without a gap".
- **Your service manages indexes on startup:** chapter 3, "Who runs the build".
- **Reviewing a multi-tenant query:** chapter 4, both halves.
- **A cursor skips, repeats or double-processes records:** chapter 5, "Rule 1" and "Rule 2".
- **A dashboard aggregation is slow or its counts are too high:** chapter 6.

## The one-paragraph summary of everything

A document's field can hold a value, hold `null`, or be missing, and the difference is decided
by whichever driver wrote it. `{f: null}` matches both null and missing, `$exists: true`
matches null, comparisons only compare values of the same BSON type, and an ordinary index
stores missing as the key `null`. So a plain unique index allows one document without the
field. A sparse index leaves out missing but keeps stored `null`s, and a compound sparse
index includes any document with any one key. For "unique when present", use a partial index
filtered on `$type` and enforce the type. A compound unique index constrains combinations, so
for "unique whichever field it's in", store a normalised key array, put a unique multikey
index on it with the same `$type` filter, and unset it rather than writing `[]`. Keep it true
through one write path, because ODM save hooks don't run on update queries. A check-then-insert
races; a unique (partial) index doesn't, and an upsert is only safe with one. Index builds
since 4.2 lock only briefly at start and end, but unique builds fail at commit if a duplicate
exists, including one written during the build, and replicated builds wait for every voting
member. Run index changes as one reviewed step, never as every replica's boot: Mongoose's
`syncIndexes()` drops undeclared indexes and then tries to build. To tighten a constraint,
scan with the new filter, stop new violators, build the new index, then drop the old.
Order compound indexes equality, sort, range. Top-level query conditions are ANDed, so
tenant and soft-delete scopes go at the top level next to an `$or`, never inside it.
Untrusted input becomes an operator when an object reaches a filter, so enforce the type and
use `$eq`, rather than banning characters like `.` that ordinary data contains. Missing and
`null` sort below every other value, so a keyset sort key must exist on every document, paging
needs a unique tiebreaker and limit+1, and an offset must come after the sort. In a pipeline,
only the first `$match` uses an index: drive from the collection that holds the filter
fields, and when a join is only a filter, use a pipeline `$lookup` with the condition and
`$limit: 1` and test for a match, instead of `$unwind`, which multiplies rows.

## How to use me

Ask me things like:

- "Here are our indexes. Which of them actually enforce what the comments say?"
- "Is this unique index safe to build on production right now? What's the order of steps?"
- "Our service calls `syncIndexes()` on boot. Walk me through moving it into the deploy."
- "Review this query builder for tenant leaks and operator injection."
- "Why does this cursor return some records twice?"
- "Here's an aggregation and its `explain`. Why is it slow, and are the counts right?"
