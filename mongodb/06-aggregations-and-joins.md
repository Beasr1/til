# 6. Aggregations and joins

## The problem

A dashboard shows funnel metrics: how many steps of each kind happened, for one tenant, in a
date range. Steps live in one collection and point at their parent order; the tenant and the
order's creation date live on the parent. The pipeline starts on steps, joins each to its
order with `$lookup`, unwinds, then filters on the joined tenant and date. It returns the
right numbers, slowly, and gets slower every month whatever range is selected, because every
step ever recorded is read and joined before the filter runs. A cache hides it until the
entry expires. Elsewhere, a stats query joins and unwinds to check "has a matching record",
and one duplicate in the joined collection quietly counts a row twice.

This chapter is about where a pipeline's filter can run, which side of a join should drive,
and how to ask "does a match exist?" without multiplying rows. It assumes
[chapter 1](01-what-mongodb-does-with-a-query.md) (aggregation, `explain`) and reuses the
stage-order point from [chapter 5](05-sorting-and-paging.md#rule-3-an-offset-comes-after-the-sort).

## Only the start of a pipeline sees the indexes

A pipeline is a stream. The first stages are executed by the query layer against the
collection, with its indexes. Everything after works on documents already in flight. The
manual ([Aggregation Pipeline Optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)):
"the server can use an index if `$match` is the first stage in the pipeline, after any
optimizations from the query planner", and "When possible, put `$match` at the beginning of
the pipeline".

A `$lookup` adds the joined documents as an array field (`as`). A `$match` on that field
**can't** go before the join, because the field doesn't exist yet. So a filter on joined
fields filters documents that have already been read and joined.

The optimiser does help a little. When `$unwind` follows `$lookup` on its `as` field, and a
`$match` on that field follows the `$unwind`, it folds both into the `$lookup`: "the optimizer
also coalesces the `$match`". Reproduced: `explain` showed the `$match` moved inside the
lookup's own pipeline. That makes each lookup cheaper. It doesn't change how many documents
the pipeline **starts** with.

## Drive from the side that holds the filter

Reproduced with 2,000 orders (20 tenants, hourly) and 20,000 steps (ten per order), indexes
`{tenant: 1, createdAt: 1}` on orders and `{order: 1}` on steps. Question: how many steps
belong to tenant `t3`'s orders created in one week? Both pipelines answer 80.

**A. Driven from the child**, filter after the join:

```js
db.steps.aggregate([
  { $lookup: { from: "orders", localField: "order", foreignField: "_id", as: "o" } },
  { $unwind: "$o" },
  { $match: { "o.tenant": "t3", "o.createdAt": { $gte: from, $lt: to } } },
  { $count: "n" }
])
```

```
$cursor  docs 20000  keys 0                     (every step, collection scan)
$lookup  docs 20001  keys 20001  index _id_     (one join per step)
```

**B. Driven from the parent**, filter first:

```js
db.orders.aggregate([
  { $match: { tenant: "t3", createdAt: { $gte: from, $lt: to } } },   // indexed
  { $lookup: { from: "steps", localField: "_id", foreignField: "order", as: "steps" } },
  { $group: { _id: null, n: { $sum: { $size: "$steps" } } } }
])
```

```
$cursor  docs 88  keys 88     (8 orders via the index, then their 80 steps via order_1)
```

About 40,000 documents read against 88, for the same answer. A grows with the size of the
steps collection; B grows with the number of orders in the range. That's the whole point:
**the driving collection should be the one whose `$match` the index can answer.**

When the question really is about the child (group steps by kind, say), there are two good
shapes:

| Shape | How | Cost |
|---|---|---|
| Parent-driven, then unwind the children | `$match` orders, `$lookup` steps, `$unwind`, `$group` by step kind | Reads only the orders in range and their steps |
| **Denormalise the filter fields** onto the child | Copy `tenant` and the parent's `createdAt` onto each step at write time; `$match` steps first on an index `{tenant, createdAt}` | No join at all; the copies must be kept true |

Denormalising is safe when the copied fields never change after creation (a tenant, a
creation time). It's a maintenance burden when they do, for the same reason as the derived
keys in [chapter 2](02-uniqueness.md#keeping-a-derived-field-true).

> **Teacher's aside.** A cache in front of a slow aggregation doesn't make it fast; it makes
> it rare. The cost is still paid on every miss, and it keeps growing with history, so the
> page gets worse each month while the cache hit rate hides it. Look at the uncached
> `explain` before deciding the query is fine.

### The join itself needs an index

The `$lookup` page is blunt: "If a supporting index on the `foreignField` does not exist, a
`$lookup` operation that performs an equality match with a single join likely has poor
performance" ([$lookup](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)).
Inside a `let`/`pipeline` sub-query, comparisons in `$expr` can use an index on the foreign
collection, with limits: only against values that resolve to constants, not when the value is
empty or missing, and "Multikey, partial, or sparse indexes are not used". The concise form
(since 5.0), `localField`/`foreignField` *plus* a `pipeline`, does the equality match first,
on the foreign field's index, then runs the pipeline on the matches. In the reproduction below,
`explain` reported the `order_1` index used and no collection scans.

## Checking existence without multiplying rows

`$unwind` "deconstructs an array field from the input documents to output a document for
*each* element" ([$unwind](https://www.mongodb.com/docs/manual/reference/operator/aggregation/unwind/)).
After a `$lookup`, that means one row per **match**. If you joined only to ask "is there a
matching record?", two matches make two rows, and every count downstream is inflated. It also
"does not output a document if the field value is null, missing, or an empty array", so it
silently behaves as an inner join, which is sometimes what you want and sometimes a lost row.

Put the condition and a `$limit: 1` inside the lookup, and test whether anything came back:

```js
db.orders.aggregate([
  { $match: { tenant: "t5" } },
  { $lookup: {
      from: "steps", localField: "_id", foreignField: "order",
      pipeline: [ { $match: { step: "s0" } }, { $limit: 1 }, { $project: { _id: 1 } } ],
      as: "hit" } },
  { $match: { "hit.0": { $exists: true } } },     // at least one match
  { $count: "n" }
])
```

Reproduced after inserting one duplicate `s0` step for one order: this pipeline counted
**100** orders; the `$unwind` version (`$lookup` all steps, `$unwind`, `$match` on
`hit.step`) counted **101**. The `$limit: 1` also stops each lookup at the first match, and
`$project` keeps the joined documents small.

Two habits go with it. Decide what you're counting (orders that reached a step, or step
events) and count that entity, not rows of a join. And for "has no match", the same shape
works with `{ "hit.0": { $exists: false } }`.

## Stage order, recapped

| Do | Because |
|---|---|
| Indexed `$match` first, on the driving collection | Only the start of the pipeline uses indexes |
| `$sort` before `$skip` and `$limit` | The optimiser keeps `$skip` after `$sort` and folds `$limit` into the sort as a top-k ([chapter 5](05-sorting-and-paging.md)) |
| Filters on joined data inside the `$lookup` pipeline | They run per lookup, on the foreign collection's index where possible |
| `$project` at the end | The manual: projecting early "is unlikely to improve performance, because the database performs this optimization automatically" |

## Check yourself

1. Pipeline A above reads every step even when the date range is one day. Explain
   mechanically why moving the `$match` earlier isn't an option *in that pipeline*, and what
   the optimiser's `$lookup`/`$unwind`/`$match` coalescence does and doesn't change.
2. Rewrite "count steps of each kind for tenant T in range R" so it reads only the orders in
   range and their steps. Then give the denormalised alternative, and the one condition that
   makes it safe.
3. A stats query does `$lookup` (orders → payments), `$unwind`, `$match: {"p.status": "settled"}`,
   then `$count`. Product says "orders with a settled payment". When does it over-count, and
   when does it drop orders it shouldn't?
4. A `$lookup` sub-pipeline uses `$expr: {$eq: ["$order", "$$orderId"]}`, and the foreign
   collection's only index on `order` is partial (`{ deleted: false }`). What does the
   `$lookup` documentation say about index use here, and what would you change?
5. A cached dashboard query has been "fine" for a year. Without changing the cache, how would
   you find out whether it's a problem, and what number in `explain` tells you it will get
   worse over time?
