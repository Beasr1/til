# 7. Glossary

Reference, not reading. Each term links to the chapter that introduces it.

| Term | Meaning |
|---|---|
| **BSON** | The binary document format MongoDB stores: JSON plus types such as dates, 64-bit integers, decimals, binary and ObjectId. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Document** | An ordered set of field/value pairs; the unit of storage and of atomic writes. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Collection** | A set of documents. No schema is enforced unless a validator is added. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Missing vs null** | A field can be absent, or present with value `null`. `{f: null}` matches both; `$exists: true` matches null; an ordinary index stores both as key `null`. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **ObjectId** | A 12-byte id, the default `_id`, that sorts roughly by creation time. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Filter document** | A query: top-level conditions are ANDed. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Type bracketing** | Comparison operators only compare values of the operand's BSON type. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Collection scan (`COLLSCAN`)** | Reading every document because no index serves the query. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **`IXSCAN`, `FETCH`, `SORT`** | Plan stages: walk an index, load the documents it points at, sort in memory. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Compound index** | An index on several fields, sorted by the first, then the next within it. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Multikey index** | An index on an array field: one key per distinct element. [Ch 1](01-what-mongodb-does-with-a-query.md), [Ch 2](02-uniqueness.md) |
| **Upsert** | Update the matching document, or insert one if none matches. [Ch 1](01-what-mongodb-does-with-a-query.md), [Ch 2](02-uniqueness.md) |
| **Aggregation pipeline** | An array of stages, each transforming the previous stage's output. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Replica set, primary, secondary, oplog** | A primary takes writes; secondaries replay its operation log. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **ODM** | A library mapping documents to application objects with schemas, hooks and index management; Mongoose in Node. [Ch 1](01-what-mongodb-does-with-a-query.md) |
| **Unique index** | No two documents in the index share a key. [Ch 2](02-uniqueness.md) |
| **Sparse index** | Leaves out documents *missing* the field; a stored `null` is still indexed. A compound sparse index includes any document with at least one of its keys. [Ch 2](02-uniqueness.md) |
| **Partial index** | Indexes only documents matching `partialFilterExpression`; a unique partial index constrains only those. [Ch 2](02-uniqueness.md) |
| **Derived key array** | A field the application writes, holding normalised copies of values from several fields, so one unique multikey index can constrain them all. [Ch 2](02-uniqueness.md) |
| **Duplicate-key error (`E11000`, code 11000)** | The server's refusal to write a key a unique index already holds. [Ch 2](02-uniqueness.md) |
| **Check-then-insert race** | Two requests both read "none exists" and both insert; closed by a unique index, not by application code. [Ch 2](02-uniqueness.md) |
| **Index build (since 4.2)** | One process for all builds: an exclusive lock briefly at start and commit, yielding during the scan; `background` is ignored. [Ch 3](03-changing-indexes-on-live-collections.md) |
| **Commit quorum** | How many data-bearing members must be ready before the primary commits a replicated index build; default all voting members. [Ch 3](03-changing-indexes-on-live-collections.md) |
| **Rolling index build** | Building on one replica-set member at a time as a standalone; for deployments under resource pressure. [Ch 3](03-changing-indexes-on-live-collections.md) |
| **`autoIndex` / `syncIndexes()` / `diffIndexes()`** | Mongoose: create declared indexes at startup / drop undeclared indexes and build missing ones / report what sync would do. [Ch 3](03-changing-indexes-on-live-collections.md) |
| **ESR** | Equality, Sort, Range: the recommended field order for a compound index. [Ch 3](03-changing-indexes-on-live-collections.md) |
| **Scope** | A condition every result must satisfy (tenant, owner, soft delete); always a top-level AND. [Ch 4](04-queries-that-leak.md) |
| **Implicit AND** | Top-level conditions of a filter are all required. [Ch 4](04-queries-that-leak.md) |
| **Operator injection** | Untrusted input arriving as an object with `$`-prefixed keys and being read as query operators. [Ch 4](04-queries-that-leak.md) |
| **`sanitizeFilter`** | Mongoose option wrapping `$`-keyed nested objects in `$eq`. [Ch 4](04-queries-that-leak.md) |
| **BSON comparison order** | The cross-type order sorts use: null (and missing) low, then numbers, strings, …, dates. [Ch 5](05-sorting-and-paging.md) |
| **Keyset (cursor) pagination** | Paging by "after the last sort key I saw", with a unique tiebreaker and limit+1. [Ch 5](05-sorting-and-paging.md) |
| **Tiebreaker** | A unique field (usually `_id`) appended to a sort so the order is total. [Ch 5](05-sorting-and-paging.md) |
| **Offset pagination** | `skip(n)` then `limit`; cost grows with the offset; must come after the sort. [Ch 5](05-sorting-and-paging.md) |
| **Driving collection** | The collection an aggregation starts from; the one whose first `$match` can use an index. [Ch 6](06-aggregations-and-joins.md) |
| **`$lookup`** | A join stage adding matching foreign documents as an array; equality, `let`/`pipeline`, or both. [Ch 6](06-aggregations-and-joins.md) |
| **`$unwind`** | One output document per array element; drops documents with empty, null or missing arrays by default. [Ch 6](06-aggregations-and-joins.md) |
| **Denormalising a filter field** | Copying a parent's immutable field onto children so they can be filtered without a join. [Ch 6](06-aggregations-and-joins.md) |
| **Existence join** | A pipeline `$lookup` with the condition and `$limit: 1`, tested with `"field.0": {$exists: true}`. [Ch 6](06-aggregations-and-joins.md) |
