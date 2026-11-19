# 1. What MongoDB does with a query

## The problem

Every later chapter says things like "the index stores `null` for that document", "the
`$match` can't use an index after the join" or "missing sorts with `null`". Each sentence is
short only because there's a model of the database behind it. Without the model they're
rules to memorise; with it they're consequences you could have predicted. This chapter
builds the model: what a document is, how a filter is read, what an index holds, how a write
is made, and how to see what the server actually did.

Skip it if you can read `explain("executionStats")` and say what a multikey index is. Skim
it if you can do one of the two. Every behaviour shown as output in this course was run on
MongoDB 8.0 in a throwaway container; the rest is quoted from the
[MongoDB manual](https://www.mongodb.com/docs/manual/).

A note on code. Queries are written in **mongosh** syntax, MongoDB's own shell, because that
is the query language itself, the way the PostgreSQL course writes SQL. Application code is
Go, with Rust, C++ and JavaScript versions in collapsible blocks.

## Documents, and the difference between missing and null

A **document** is an ordered set of field/value pairs, stored as **BSON**, a binary form of
JSON with more types: 32- and 64-bit integers, doubles, decimals, dates, binary data,
**ObjectId** (a 12-byte id that sorts roughly by creation time), arrays, embedded documents,
and `null`.

```js
{ _id: ObjectId("..."), tenant: "acme", email: "a@example.com",
  tags: ["vip", "eu"], address: { city: "Lyon" }, closedAt: null }
```

Every document has an `_id`, and the collection always has a unique index on it.

A **collection** is a set of documents. By default it doesn't enforce a schema. The manual
([Schema Validation](https://www.mongodb.com/docs/manual/core/schema-validation/)): "By
default, documents in a collection don't need the same fields or data types." You can add a
validator, but a validator judges one document at a time. It has no way to ask "does another
document already have this value?", which is why uniqueness lives in indexes
([chapter 2](02-uniqueness.md)).

The flexible schema gives a field **three states**, not two:

| State | Example | `{ f: null }` matches? | `{ f: { $exists: true } }` matches? | `{ f: { $type: "null" } }` matches? |
|---|---|---|---|---|
| Present with a value | `{ f: "x" }` | no | yes | no |
| Present, `null` | `{ f: null }` | yes | yes | yes |
| Missing | `{ }` | yes | no | no |

Reproduced: on a collection with one `status: null` document and one with no `status`,
`{status: null}` counted 2, `{status: {$type: "null"}}` counted 1 and
`{status: {$exists: false}}` counted 1. The `$exists` page says it outright: `$exists: true`
"matches the documents that contain the field, including documents where the field value is
`null`" ([$exists](https://www.mongodb.com/docs/manual/reference/operator/query/exists/)).

> **Teacher's aside.** Which of "missing" and "null" you end up storing is often decided by
> your driver's defaults, not by you. Reproduced with the current drivers: a nil Go pointer
> field (`*time.Time`) is written as `null`, and the same field tagged `omitempty` is left
> out. The Node driver writes a property set to `undefined` as `null`, unless the client was
> created with `ignoreUndefined: true`, in which case it's left out. Two services writing
> the same collection can therefore store both states for the same meaning, and most of the
> traps in this course start there.

## Reading a filter

A query is a **filter document**. Each top-level field is a condition, and the conditions
are ANDed. The manual ([$and](https://www.mongodb.com/docs/manual/reference/operator/query/and/)):
"MongoDB provides an implicit `AND` operation when you specify a comma separated list of
expressions."

```js
db.orders.find({ tenant: "acme", status: "paid", total: { $gte: 100 } })
//              ^ equality       ^ equality       ^ operator expression
```

The parts you need:

| Form | Meaning |
|---|---|
| `{ f: v }` | `f` equals `v`. If `f` is an array, *any element* equals `v` |
| `{ f: { $eq: v } }` | The same, written explicitly. Matters for untrusted input ([chapter 4](04-queries-that-leak.md)) |
| `{ f: { $gt: v } }`, `$gte`, `$lt`, `$lte`, `$ne`, `$in`, `$nin` | Comparisons |
| `{ $or: [ {…}, {…} ] }` | At least one branch matches |
| `{ "a.b": v }` | **Dot notation**: field `b` inside embedded document `a` |
| `{ f: { $exists: true } }`, `{ f: { $type: "string" } }` | Presence, and BSON type |

Comparisons are **type-bracketed**. The manual
([Comparison/Sort Order](https://www.mongodb.com/docs/manual/reference/bson-type-comparison-order/)):
comparison operators apply "only on documents where the BSON type of the target field matches
the query operand type". So `{ closedAt: { $gt: ISODate("2025-01-01") } }` never matches a
document whose `closedAt` is missing, `null` or a string. Reproduced: of six documents with
two dates, one missing, one `null`, one empty array and one string, `$gt` a date returned
only the two dates.

## What an index holds

Without an index the server reads every document in the collection, a **collection scan**
(`COLLSCAN` in a plan). An index is a separate, sorted structure (a B-tree) of **keys**, each
pointing at a document. Finding a key takes a few page reads; then the server fetches the
document it points at (`IXSCAN` then `FETCH`).

The kinds that matter here:

| Kind | Key per document | Note |
|---|---|---|
| Single field `{ email: 1 }` | One | |
| Compound `{ tenant: 1, createdAt: -1 }` | One, a tuple | Sorted by the first field, then the second within it. Field order matters ([chapter 3](03-changing-indexes-on-live-collections.md)) |
| **Multikey** | One per distinct array element | Created automatically when an indexed field holds an array ([Multikey Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/)) |
| Unique | One, and no other document may have it | |
| Sparse or partial | Zero for documents left out | Chapter 2 |

The fact the whole of chapter 2 hangs on: an ordinary index has an entry for **every**
document, including ones without the field. The manual
([Sparse Indexes](https://www.mongodb.com/docs/manual/core/index-sparse/)): "non-sparse
indexes contain all documents in a collection, storing null values for those documents that
do not contain the indexed field." Missing and `null` become the same key.

Two consequences you'll meet repeatedly:

- **Every index is written on every insert**, and on every update that touches an indexed
  field. Indexes cost write throughput and memory.
- **An index answers only what its order can answer.** A compound index on
  `{ tenant, createdAt }` finds "this tenant, newest first" by walking one contiguous run of
  keys. It can't help a query on `createdAt` alone, because those keys are scattered across
  every tenant's run.

## Writes are atomic per document

The manual ([Atomicity](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/)):
"write operations are atomic on the single-document level, even if modifying multiple
values." An `updateOne` that sets three fields either sets all three or none. Across
documents there is no such promise without a transaction: "the modification of each document
is atomic, but the operation as a whole is not atomic."

Updates use operators rather than rewriting the document:

```js
db.people.updateOne(
  { _id: id },
  { $set: { email: "b@example.com" }, $unset: { phone: "" } },  // $unset removes the field
  { upsert: true })                                              // insert if nothing matched
```

An **upsert** updates the matching document, or inserts one if none matches. Whether two
concurrent upserts can both insert is a chapter 2 question.

## Aggregation: a pipeline of stages

For anything beyond "find documents matching this", MongoDB has the **aggregation
pipeline**: an array of stages, each consuming the previous stage's output.

```js
db.orders.aggregate([
  { $match: { tenant: "acme", status: "paid" } },     // filter (can use an index if first)
  { $group: { _id: "$customer", spent: { $sum: "$total" } } },
  { $sort: { spent: -1 } },
  { $limit: 10 }
])
```

Only the start of a pipeline can use the collection's indexes. Everything after works on
documents in memory. Chapters [5](05-sorting-and-paging.md) and [6](06-aggregations-and-joins.md)
are about what that implies for stage order.

## Seeing what the server did: `explain`

`explain("executionStats")` shows the winning plan and what it cost. The stages you'll read
most are `COLLSCAN`, `IXSCAN` (with the index name), `FETCH`, `SORT` (an in-memory sort) and
`LIMIT`, and the three counters `totalKeysExamined`, `totalDocsExamined` and `nReturned`.

From a 20,000-document `orders` collection, first without an index, then with
`{ tenant: 1, status: 1, createdAt: -1 }`:

```
find({tenant:"t7", status:"open"})
  COLLSCAN                                   keys 0   docs 20002  returned 333

find({tenant:"t7", status:"open"}).sort({createdAt:-1}).limit(10)
  LIMIT <- FETCH <- IXSCAN(tenant_1_status_1_createdAt_-1)
                                             keys 10  docs 10     returned 10
```

The healthy shape is keys ≈ docs ≈ returned. Docs examined far above returned means the
index isn't selective for this query. A `SORT` stage means the index order didn't match the
requested order and the server sorted in memory.

## Replica sets, briefly

Production MongoDB runs as a **replica set**: one **primary** that takes writes and some
**secondaries** that copy them by replaying the primary's **oplog**, a log of operations. An
index build is replicated too, and builds on every data-bearing member
([chapter 3](03-changing-indexes-on-live-collections.md)). Reads go to the primary unless you
ask otherwise.

## Drivers and ODMs

Applications talk to MongoDB through a **driver** (the official Go, Rust, C++ and Node
drivers appear in this course). Some teams add an **ODM** on top, most often
[Mongoose](https://mongoosejs.com/) in Node. An ODM adds a schema, defaults, middleware
hooks around saves, and automatic index creation. Several traps in this course are ODM
features doing exactly what they were documented to do, so the course names Mongoose where
it matters.

## Check yourself

1. A collection has 1,000 documents. 900 have no `phone` field, 50 have `phone: null` and 50
   have a phone number. How many documents does each of `{phone: null}`,
   `{phone: {$exists: true}}` and `{phone: {$type: "string"}}` match?
2. An ordinary (non-sparse) index on `phone` exists on that collection. How many index
   entries does it hold, and what key do the 950 documents without a number share?
3. `find({ createdAt: { $lt: ISODate("2026-01-01") } })` returns fewer documents than the
   team expects. Some old documents stored `createdAt` as a string. Explain mechanically why
   they're missing from the result.
4. A plan shows `IXSCAN` with `totalKeysExamined: 40000`, `totalDocsExamined: 40000` and
   `nReturned: 12`. What does that tell you about the index, and what would you look at
   next?
5. Two services write the same collection. One is Go, with `ClosedAt *time.Time` tagged
   `bson:"closedAt,omitempty"`. The other is Node, with a client created using
   `ignoreUndefined: false` (the default), and sets `closedAt: undefined` on open orders. What
   does each store for an open order, and name one query that treats the two differently.
