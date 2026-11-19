# 5. Sorting and paging

## The problem

A billing job reads records in completion order and keeps a cursor, so each run picks up where
the last one stopped. Some records were rejected and never completed, so they have no
`completedAt`. In an ascending sort they come out first, ahead of records completed weeks
earlier, and a date-range filter on `completedAt` doesn't return them at all. Depending on
which query a run used, the same record lands in a different billing window, or in none, and
some customers are charged twice. Separately, a paged table gains a "jump to page N" button
implemented with `$skip`, and page 5 shows rows that belong nowhere near page 5.

Both come from how MongoDB orders values and where in a pipeline the order is applied. It
assumes [chapter 1](01-what-mongodb-does-with-a-query.md): the three states of a field and
type bracketing.

## How MongoDB orders values

A sort compares values of any type, so there's a fixed order across types. From the manual
([Comparison/Sort Order](https://www.mongodb.com/docs/manual/reference/bson-type-comparison-order/)),
lowest first:

| Rank | Type |
|---|---|
| 1 | MinKey (internal) |
| 2 | **Null** |
| 3 | Numbers (ints, longs, doubles, decimals, compared as one type) |
| 4 | Symbol, String |
| 5 | Object |
| 6 | Array |
| 7 | BinData |
| 8 | ObjectId |
| 9 | Boolean |
| 10 | **Date** |
| 11 | Timestamp |
| 12–15 | Regular expression, JavaScript, JavaScript with scope, MaxKey |

Three rules from the same page decide most surprises:

- "The comparison treats a non-existent field as if it were null." **Missing sorts with
  `null`, at the bottom.**
- "A sort places an empty array (for example, `[ ]`) before a `null` value or a missing
  field value."
- An array field sorts by its smallest element ascending and its largest descending.

Reproduced: six documents whose `done` field is a later date, missing, `null`, an earlier
date, `[]` and the string `"x"`, sorted `{done: 1, _id: 1}`:

```
ascending:   [] , missing , null , "x" , 2026-01-01 , 2026-01-03
descending:  2026-01-03 , 2026-01-01 , "x" , missing , null , []
```

Missing and `null` sit together (ordered between themselves only by the `_id` tiebreaker),
below strings, which are below every date. Now put that next to type bracketing: a range
filter `{done: {$gte: <date>, $lt: <date>}}` returned only the two dates. So a document
without the sort key is **first** in an unfiltered ascending sort and **absent** from any
range or cursor filter on that key.

## Rule 1: the sort key must exist on every document

A **keyset** (or cursor) page asks for "the next N after the last one I saw", using the
sort key itself as the bookmark. It's the right way to page: it uses the index, costs the
same on page 1,000 as on page 1, and doesn't skip or repeat rows when other rows are
inserted. It also assumes every document has a comparable value in the sort key.

When some don't:

| Query | Documents without the key |
|---|---|
| First page, no cursor, ascending | All returned first, before any real value |
| Next page, `{key: {$gt: last}}` | Never returned, because `$gt` a date doesn't match null or missing |
| Date-range filter on the key | Never returned |
| Descending sort | All returned last, after everything else |

A job that mixes those modes (a first run with no cursor, then cursor runs, then a backfill
with a date range) sees the same documents at different points, or not at all. That's the
double charge: a record without `completedAt` came out at the very start of one run, opening
a window ahead of records completed before it, and was invisible to the range query that
should have placed it.

The fix is a sort key that is **always present** with one type. Usually that's a timestamp
every document gets on every write (`updatedAt`), or `createdAt`, or `_id`. If the meaningful
key is optional, derive a required one (`effectiveAt`, set to `completedAt` when there is one
and the rejection time otherwise) and sort on that. A `$jsonSchema` validator requiring the
field with `bsonType: "date"` keeps it that way.

> **Teacher's aside.** "This field is always set" is a claim about every code path that has
> ever written the collection, including ones that were removed. Before keying anything on a
> field, count the documents where it's missing or not the expected type:
> `countDocuments({ key: { $not: { $type: "date" } } })`. It takes a second and it's the
> whole lesson.

## Rule 2: break ties with a unique field

From [cursor.sort()](https://www.mongodb.com/docs/manual/reference/method/cursor.sort/):
"When sorting on a field which contains duplicate values, documents containing those values
may be returned in any order", and "If consistent sort order is desired, include at least one
field in your sort that contains unique values. The easiest way to guarantee this is to
include the `_id` field."

For keyset paging, the tiebreaker has to be in the bookmark too, or rows sharing a timestamp
at a page boundary are skipped or repeated:

```js
db.orders.find({
  tenant: "t1",                                          // scope, top level (chapter 4)
  $or: [
    { updatedAt: { $gt: lastUpdatedAt } },
    { updatedAt: lastUpdatedAt, _id: { $gt: lastId } }
  ]
}).sort({ updatedAt: 1, _id: 1 }).limit(pageSize + 1)
```

with an index `{ tenant: 1, updatedAt: 1, _id: 1 }`, which is equality then sort
([chapter 3](03-changing-indexes-on-live-collections.md#ordering-a-compound-index-equality-sort-range)).

**Fetch one extra row.** Asking for `pageSize + 1` tells you whether there's a next page
without a separate count: if you got more than `pageSize`, drop the extra one, and the last
row you keep is the next bookmark.

## Rule 3: an offset comes after the sort

Sometimes the UI needs "page 7 of 40". An offset can serve that, with two cautions.

**Cost.** "The `skip()` method requires the server to scan from the beginning of the input
results set before beginning to return results. As the offset increases, `skip()` will become
slower" ([cursor.skip()](https://www.mongodb.com/docs/manual/reference/method/cursor.skip/)).

**Order.** In a `find()`, the call order doesn't matter. "The server always applies the skip
operation based on the sort order before it applies the limit." In an **aggregation**, stages
run in the order written, and the optimiser won't move a `$skip` past a `$sort`, because that
would change the answer. The only `$skip` reordering the
[optimisation page](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)
describes is moving it ahead of a `$project`. So a pipeline that skips first, then sorts,
returns a sorted slice of an arbitrary set of documents. Reproduced on ten documents:

```
[$sort, $skip: 3, $limit: 3]   ->  8:6, 5:5, 2:4      (the 4th–6th highest scores)
[$skip: 3, $sort, $limit: 3]   ->  7:9, 4:8, 8:6      (the top 3 of whatever was left)
```

`explain` showed the skip-first pipeline pushing the skip into the initial collection read,
before any sort existed. This is easy to write by accident when a cursor-based pipeline gains
an offset fallback: the cursor's `$match` goes first, and someone adds the `$skip` next to it.
Write `$match`, `$sort`, `$skip`, `$limit`, in that order.

## The shape in code

Keyset paging with a tiebreaker and limit+1. Reproduced with the Go driver v2.5.0 on seven
orders sharing timestamps in pairs: pages of 3, 3 and 1, with "more" true, true, false.

```go
type Order struct {
	ID        bson.ObjectID `bson:"_id"`
	Tenant    string        `bson:"tenant"`
	UpdatedAt time.Time     `bson:"updatedAt"`
}

type Cursor struct {
	UpdatedAt time.Time
	ID        bson.ObjectID
}

// pageOrders returns up to limit orders after the cursor, and the cursor
// for the next page, or nil when this is the last page.
func pageOrders(ctx context.Context, orders *mongo.Collection, tenant string,
	after *Cursor, limit int64) ([]Order, *Cursor, error) {
	filter := bson.D{{Key: "tenant", Value: tenant}}
	if after != nil {
		filter = append(filter, bson.E{Key: "$or", Value: bson.A{
			bson.D{{Key: "updatedAt", Value: bson.D{{Key: "$gt", Value: after.UpdatedAt}}}},
			bson.D{
				{Key: "updatedAt", Value: after.UpdatedAt},
				{Key: "_id", Value: bson.D{{Key: "$gt", Value: after.ID}}},
			},
		}})
	}
	opts := options.Find().
		SetSort(bson.D{{Key: "updatedAt", Value: 1}, {Key: "_id", Value: 1}}).
		SetLimit(limit + 1) // one extra row says whether another page exists
	cur, err := orders.Find(ctx, filter, opts)
	if err != nil {
		return nil, nil, err
	}
	var out []Order
	if err := cur.All(ctx, &out); err != nil {
		return nil, nil, err
	}
	if int64(len(out)) <= limit {
		return out, nil, nil
	}
	out = out[:limit]
	last := out[limit-1]
	return out, &Cursor{UpdatedAt: last.UpdatedAt, ID: last.ID}, nil
}
```

<details><summary>Rust</summary>

Written against the `mongodb` crate 3.x API docs
([`Find`](https://docs.rs/mongodb/latest/mongodb/action/struct.Find.html): `sort(Document)`,
`limit(i64)`) with `futures::TryStreamExt` to collect the cursor; not compiled here.

```rust
use futures::TryStreamExt;
use mongodb::bson::{doc, oid::ObjectId, DateTime};
use mongodb::Collection;
use serde::Deserialize;

#[derive(Deserialize)]
pub struct Order {
    #[serde(rename = "_id")]
    pub id: ObjectId,
    pub tenant: String,
    #[serde(rename = "updatedAt")]
    pub updated_at: DateTime,
}

pub struct PageCursor {
    pub updated_at: DateTime,
    pub id: ObjectId,
}

pub async fn page_orders(
    orders: &Collection<Order>,
    tenant: &str,
    after: Option<&PageCursor>,
    limit: i64,
) -> mongodb::error::Result<(Vec<Order>, Option<PageCursor>)> {
    let mut filter = doc! { "tenant": tenant };
    if let Some(a) = after {
        filter.insert("$or", vec![
            doc! { "updatedAt": { "$gt": a.updated_at } },
            doc! { "updatedAt": a.updated_at, "_id": { "$gt": a.id } },
        ]);
    }
    let mut rows: Vec<Order> = orders
        .find(filter)
        .sort(doc! { "updatedAt": 1, "_id": 1 })
        .limit(limit + 1) // one extra row says whether another page exists
        .await?
        .try_collect()
        .await?;
    if rows.len() as i64 <= limit {
        return Ok((rows, None));
    }
    rows.truncate(limit as usize);
    let last = rows.last().expect("limit > 0");
    let next = PageCursor { updated_at: last.updated_at, id: last.id };
    Ok((rows, Some(next)))
}
```

</details>

<details><summary>C++</summary>

Written against the mongocxx/bsoncxx 4.x API reference; not compiled here. Rows are returned
as owned BSON documents.

```cpp
#include <cstdint>
#include <optional>
#include <string>
#include <utility>
#include <vector>

#include <bsoncxx/builder/basic/array.hpp>
#include <bsoncxx/builder/basic/document.hpp>
#include <bsoncxx/builder/basic/kvp.hpp>
#include <bsoncxx/types.hpp>
#include <mongocxx/collection.hpp>
#include <mongocxx/options/find.hpp>

using bsoncxx::builder::basic::kvp;
using bsoncxx::builder::basic::make_array;
using bsoncxx::builder::basic::make_document;

struct PageCursor {
  bsoncxx::types::b_date updated_at;
  bsoncxx::oid id;
};

std::pair<std::vector<bsoncxx::document::value>, std::optional<PageCursor>>
page_orders(mongocxx::collection& orders, const std::string& tenant,
            const std::optional<PageCursor>& after, std::int64_t limit) {
  bsoncxx::builder::basic::document filter;
  filter.append(kvp("tenant", tenant));
  if (after) {
    filter.append(kvp("$or", make_array(
        make_document(kvp("updatedAt", make_document(kvp("$gt", after->updated_at)))),
        make_document(kvp("updatedAt", after->updated_at),
                      kvp("_id", make_document(kvp("$gt", after->id)))))));
  }
  mongocxx::options::find opts;
  opts.sort(make_document(kvp("updatedAt", 1), kvp("_id", 1)));
  opts.limit(limit + 1);  // one extra row says whether another page exists

  std::vector<bsoncxx::document::value> rows;
  for (auto&& d : orders.find(filter.extract(), opts)) rows.emplace_back(d);
  if (static_cast<std::int64_t>(rows.size()) <= limit) return {std::move(rows), std::nullopt};
  rows.erase(rows.begin() + limit, rows.end());
  auto last = rows.back().view();
  PageCursor next{last["updatedAt"].get_date(), last["_id"].get_oid().value};
  return {std::move(rows), next};
}
```

</details>

<details><summary>JavaScript</summary>

An ES module, run with Node 26 and the `mongodb` driver 6.22; same pages as the Go version.

```js
// Up to `limit` orders after `after`, and the next cursor or null.
export async function pageOrders(orders, tenant, after, limit) {
  const filter = { tenant };
  if (after) {
    filter.$or = [
      { updatedAt: { $gt: after.updatedAt } },
      { updatedAt: after.updatedAt, _id: { $gt: after.id } },
    ];
  }
  const rows = await orders.find(filter)
    .sort({ updatedAt: 1, _id: 1 })
    .limit(limit + 1)          // one extra row says whether another page exists
    .toArray();
  if (rows.length <= limit) return { rows, next: null };
  rows.length = limit;
  const last = rows[limit - 1];
  return { rows, next: { updatedAt: last.updatedAt, id: last._id } };
}
```

</details>

## Check yourself

1. A collection of tasks is paged by `dueAt` ascending with a keyset cursor. 5% of tasks have
   no due date. Describe exactly when, if ever, a user paging from the start sees those
   tasks, and what changes if the sort is descending.
2. Why does `{completedAt: {$gt: last}}` never return a document whose `completedAt` is
   `null`, even though `null` sorts *below* every date rather than being excluded from the
   order?
3. Two orders share `updatedAt` to the millisecond and fall either side of a page boundary.
   With a cursor on `updatedAt` alone (`$gt: last`), what happens to the second one? And with
   `$gte: last`?
4. A pipeline is `[{$match: cursorFilter}, {$skip: (page - 1) * size}, {$sort: {_id: -1}},
   {$limit: size}]`. What does page 3 contain, and what's the corrected pipeline?
5. You must add page numbers to a table that's paged by keyset today. Give one design that
   keeps keyset for "next" and still supports "jump to page 40", and say what it costs.
