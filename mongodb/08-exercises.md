# 8. Exercises and answers

Worked answers to every **Check yourself** question, then things to try on a throwaway
server, then questions worth asking me. A local server is one command:
`docker run -d --name mongo-scratch -p 27017:27017 mongo:8.0`, then
`docker exec -it mongo-scratch mongosh`.

---

## Chapter 1 — What MongoDB does with a query

**1. `{phone: null}`, `{phone: {$exists: true}}`, `{phone: {$type: "string"}}`.**
`{phone: null}` matches both "missing" and "present, null": 900 + 50 = **950**.
`$exists: true` matches every document that has the field, whatever its value, so the 50
nulls and the 50 numbers: **100**. `$type: "string"` matches only real strings: **50**
(assuming the numbers are stored as strings, which phone numbers should be). Three queries
that sound similar, three different answers, which is the chapter's table in action.

**2. Entries in an ordinary index on `phone`.**
An ordinary index has an entry for every document, so **1,000**. The 900 missing and 50 null
documents all get the key `null`; the manual says the index stores "null values for those
documents that do not contain the indexed field". This is why a plain *unique* index on
`phone` can't be built on this collection at all: 950 documents share one key.

**3. Old documents with string dates missing from a `$lt` date query.**
Comparison operators are type-bracketed: `$lt: ISODate(...)` only compares against fields
whose BSON type is a date. A string `"2025-03-01"` is a different type, so it's not "less
than" the date; it's not compared at all, and the document isn't returned. The fix is to
migrate the strings to dates. Querying for both types is a stopgap, and sorting will still
put the strings in their own block (chapter 5).

**4. `IXSCAN`, 40,000 keys, 40,000 docs, 12 returned.**
The index was used but isn't selective for this query: the server walked 40,000 keys and
fetched every document to apply the rest of the filter, keeping 12. Look at the filter's
other fields. Usually there's an equality condition that isn't in the index, or is in it
after a range field, so the scan can't narrow on it. Adding that field to the index in the
right position (chapter 3, ESR) brings keys and docs down towards 12.

**5. Go with `omitempty`, Node with `undefined`.**
Go's nil `*time.Time` tagged `omitempty` is left out, so the document has **no** `closedAt`.
The Node driver writes `undefined` as `null` by default, so its documents have
`closedAt: null`. `{closedAt: null}` matches both. `{closedAt: {$exists: false}}` matches
only Go's, `{closedAt: {$type: "null"}}` only Node's, and a sparse index on `closedAt`
contains Node's documents but not Go's. Any "open orders" query written against one service's
habit silently misses the other's.

---

## Chapter 2 — Uniqueness

**1. A sparse unique index starts failing for a new service.**
Sparse skips documents that *don't have* the field. The old writers left `nid` out; the new
service writes `nid: null`. A field holding `null` is present, so the sparse index includes
it under key `null`, and the second such person collides. A partial index with
`partialFilterExpression: { nid: { $type: "string" } }` would have been immune: `null` isn't
a string, so those documents are never in the index, however a writer spells "no value".

**2. The compound sparse index rejects the second person in a tenant.**
A compound sparse index includes a document that has *any one* of its keys. Every person has
`tenant`, so every person is in the index, and a person with no `nid` gets the key
`(tenant, null)`. Two people in one tenant without a national id share that key. The
single-field sparse index on `nid` leaves out documents missing `nid`, so they never meet.
The fix is a partial filter on the optional field, which scopes by `nid` regardless of what
else is in the key.

**3. National ids stored as numbers under a `$type: "string"` filter.**
The constraint now guarantees uniqueness only among string ids. Numeric ids aren't in the
index at all, so two people can share `nid: 12345`, and a numeric `12345` and a string
`"12345"` never meet either. Add a type rule: a `$jsonSchema` validator requiring
`bsonType: "string"` for `nid`, or a single typed write path, plus a one-off migration
converting existing numbers to strings. Widening the filter to
`$type: ["string", "number"]` would admit both, but still wouldn't make `12345` and `"12345"`
collide; normalising to one type is the real fix.

**4. Compound unique versus a derived `docKeys` array.**
A compound unique index constrains the **combination**. Person A `(P1, I1)` and person B
`(I9, P1)` are two different pairs, so nothing objects, even though `P1` is on both people.
The derived array puts every document number of a person into one field. A multikey index
makes one key per element, so A contributes keys `P1` and `I1` and B contributes `I9` and
`P1`. Uniqueness applies between documents, so B's `P1` collides with A's. It works because
every value now lives in the same index, under one path.

**5. Dropping the partial filter from the `docKeys` index.**
Two kinds of document start colliding. People whose `docKeys` is missing all get the key
`null`, so the second one fails. People with `docKeys: []` all get the key `undefined` (what
the server reports for an empty array), so the second of those fails too. In practice the
deploy fails at index build time if the collection already holds two of either. If it's
empty, the failure arrives later, as the second person without documents being refused.

**6. A `pre('save')` hook and an endpoint using `findOneAndUpdate`.**
`findOneAndUpdate` doesn't run save middleware, so the endpoint changes `passport.number`
and leaves `docKeys` holding the **old** key. The index still enforces uniqueness of whatever
is in `docKeys`, which no longer describes the person. Concrete duplicate: person A's passport
is corrected from `P1` to `P2`, but A's `docKeys` still says `["GBR::P1"]`. Person B then
registers passport `P2`; B's key `GBR::P2` doesn't collide with anything, so two people hold
`P2`. Worse, a third person now can't register `P1`, which nobody holds. Fix it by routing
the endpoint through `save()`, or recomputing `docKeys` in the update and keeping that code
next to the hook.

---

## Chapter 3 — Changing indexes on live collections

**1. Removing `background: true`.**
They're wrong. Since 4.2 the server ignores the option and every build uses the same process:
an exclusive lock briefly at the start and at commit, an intent lock that yields to reads and
writes during the scan, and a shared lock while draining writes made during the build. What
"blocks" is those short exclusive windows and the drain. What *hurts* is resource use: the
build reads the whole collection and competes with traffic, which no option changes. Removing
the flag is a no-op; deleting dead options is fine.

**2. A unique build fails after forty minutes although the scan found nothing.**
Writes continue during the scan phase and aren't checked against the index being built.
Sequence: the scan reports zero duplicates; the build starts; twenty minutes in, a writer
inserts a document whose `nid` matches an existing one (nothing stops it, since the unique
index doesn't exist yet); at commit the server finds the violation and terminates the build.
The fix is step 3 of the chapter: stop new violators during the build, by pausing writers or
at least checking in the application, and then scan again.

**3. Ten replicas running `syncIndexes()` and an undeclared manual index.**
On the next deploy, each booting replica compares the database with the schema, sees an index
the schema doesn't declare, and drops it. `syncIndexes()` drops everything undeclared except
`_id`. The first replica to boot does the drop; the rest find nothing to do. The dashboard
query silently falls back to a slower plan, possibly a collection scan. Nobody notices
because nothing failed. The drop is a successful operation logged, if at all, in one of ten
processes, and the symptom (a slow page) shows up later, under load, with no obvious link to
a deploy.

**4. Drop-then-create fails at the create.**
The collection has **no** uniqueness constraint on `nid`: the old index is gone and the new one
never built. Every write until someone notices can add duplicates, which then also make the
eventual build harder. Correct order: scan with the new index's filter and reconcile, stop new
violators, create the new index under a new name, confirm it exists, then drop the old one,
then make the declared schema match.

**5. `$in` with a sort.**
For a short list the planner treats `$in` as several equality scans and merges them in sort
order. Reproduced with `{tenant: 1, status: 1, createdAt: -1}` and
`status: {$in: ["paid", "open"]}`, sorted by `createdAt: -1`, limit 10: the plan was
`SORT_MERGE` over two `IXSCAN`s with 11 keys and 10 documents examined, and no in-memory
`SORT`. So here it behaves as equality and `{tenant, status, createdAt}` is the ESR index. In
`explain`, confirm a `SORT_MERGE` (not a `SORT`) and keys close to the limit. There is a limit on
how many scans the planner will split a query into (`explain` reports a
`maxScansToExplodeReached` flag); I haven't measured where it falls, so check `explain` for
long lists.

---

## Chapter 4 — Queries that leak

**1. Rewriting the leaking `$or`.**
`{ tenant: t, deleted: false, $or: [{ passport: p }, { idCard: c }] }`. The tenant and
soft-delete conditions are top-level, so they're ANDed with the alternatives. The test: put
two people in tenant `t`, look up a passport and id card that belong to **neither**, and
assert the result is empty. The original returns one of them; the fixed query returns
nothing. Tests that only look up existing people pass with both forms.

**2. Two assignments to `filter.$or`.**
A JavaScript object holds one value per key, so the second assignment replaces the first. The
final query only says "matches one of the status conditions"; the identifier alternatives are
gone, and every document in the scope with a matching status comes back. Write
`filter.$and = [{ $or: idConditions }, { $or: statusConditions }]`, or build the filter in one
place from named parts so a later feature can't overwrite an earlier one.

**3. Same indexes, different plans.**
In the scoped query, every branch of the `$or` is effectively `tenant = t AND passport = p`
or `tenant = t AND idCard = c`, and each has a compound index starting with `tenant`. The
planner ran one `IXSCAN` per branch and merged them: 2 keys, 2 documents. In the leaking
query, one branch is `tenant = t` on its own, and the others aren't scoped. MongoDB uses
indexes for an `$or` only if *every* clause is supported. Here the planner chose a collection
scan, and even a perfect plan would have to return every document in the tenant, because
that's what the query asks for.

**4. The login handler.**
`email` is injectable: if the body is parsed JSON, `{"email": {"$ne": null}, "password": "x"}`
makes the filter "any email, and this password hash". `passwordHash` isn't directly injectable,
because `hash(...)` returns a string (assuming it stringifies its input, or throws on an
object). So an attacker who knows or guesses one password, or a common one, can log in as
*whichever user has that password*, without knowing the email. With `$regex` they could also
probe emails letter by letter. Fix: check `typeof body.email === "string"` and query with
`{ email: { $eq: body.email } }`.

**5. The character-blocking sanitiser.**
It breaks emails (`a.b@example.com`), decimal amounts (`12.50`), names with initials or
abbreviations (`St. John`), URLs and domain names. It doesn't stop the object: `String({"$ne":
null})` is `"[object Object]"`, which contains neither `.` nor `$`, so the check passes and
the original object goes into `findOne`, where it's an operator. The check inspected a
different value from the one that reached the query. Check the type of the value you're
going to use, then use exactly that value.

---

## Chapter 5 — Sorting and paging

**1. Tasks without a due date, keyset by `dueAt` ascending.**
They're on the **first page only**, ahead of every dated task, because missing sorts with
`null`, below every date. If there are more of them than one page holds, the cursor saved
from page one has a `null` (or missing) bookmark. Whether the next query even works then
depends on how your code handles it. If the bookmark is a real date, every later query is
`dueAt > date`, which can't match them, so a user who starts paging *after* them, or who
filters by a date range, never sees them. Descending, they're all on the **last** page, after
every dated task, and a cursor arriving from the dated side can't reach them with
`$lt: date` either, for the same reason. Fix: a required sort key, or page the undated tasks
as a separate, explicit section.

**2. Why `$gt: last` never returns `null`.**
Sorting and filtering follow different rules. A sort has to place every value somewhere, so it
uses the cross-type order, where `null` ranks below dates. A comparison operator is
type-bracketed: `$gt: <date>` only compares against dates, and a `null` or missing field is a
different type, so it simply doesn't match. The sort says where `null` *would* go; the filter
never asks.

**3. A shared timestamp at a page boundary.**
With `$gt: last` alone, the second order has `updatedAt == last`, so it isn't greater and is
**skipped** forever. With `$gte: last`, the first order, already shown, matches again and is
**repeated**, and if more rows share the timestamp than fit on a page, the cursor never
advances. The tiebreaker fixes both: `updatedAt > last OR (updatedAt == last AND _id >
lastId)`, sorted by `{updatedAt, _id}`.

**4. `$skip` before `$sort`.**
Page 3 skips `2 * size` documents of the `$match` output in whatever order the server read
them (often index or insertion order), then sorts the remainder by `_id` descending and keeps
the first `size`. That's the newest documents among an arbitrary remainder. It overlaps
heavily with pages 1 and 2 and isn't the third page of anything. Corrected:
`[{$match: …}, {$sort: {_id: -1}}, {$skip: (page - 1) * size}, {$limit: size}]`. Better
still, don't combine a keyset `$match` with a skip at all: use one mode or the other.

**5. Keyset for "next", plus page numbers.**
One design: keep keyset for next/previous, and implement "jump to page 40" as an offset query
(`sort` then `skip(39 * size)` then `limit(size)`) that returns the page *and* the bookmark of
its last row, so "next" from there is keyset again. The jump costs a scan of `39 * size` index
entries, which the manual warns grows with the offset; the page count needs a
`countDocuments`, which also scans. Another design is to cap jumps (only the first N pages,
or search instead of deep paging). In each case, say in the UI that page boundaries can shift
when data changes.

---

## Chapter 6 — Aggregations and joins

**1. Why the `$match` can't move earlier in pipeline A.**
The filter is on `o.tenant` and `o.createdAt`, fields of the joined array. Before the
`$lookup` runs, a step document has no `o` field, so there's nothing to filter on; the
filter's meaning depends on the join. The coalescence moves the `$match` *into* the
`$lookup`, so each lookup only returns orders that match. But the pipeline still begins with
every step in the collection (a collection scan) and still performs one lookup per step. It
reduces the size of what each lookup returns, not the number of lookups. The fix is changing
the driving collection, not the stage order.

**2. Steps by kind, parent-driven, and the denormalised version.**
`db.orders.aggregate([{$match: {tenant: T, createdAt: {$gte: R0, $lt: R1}}}, {$lookup: {from:
"steps", localField: "_id", foreignField: "order", as: "s"}}, {$unwind: "$s"}, {$group: {_id:
"$s.step", n: {$sum: 1}}}])`, with indexes `{tenant: 1, createdAt: 1}` on orders and
`{order: 1}` on steps. The `$unwind` is right here because you *want* one row per step.
Denormalised: write `tenant` and `orderCreatedAt` onto each step when it's created, index
steps on `{tenant: 1, orderCreatedAt: 1}`, and run `$match` then `$group` on steps alone.
It's safe when the copied fields never change after the step is written. A tenant and a
creation time don't; a status would.

**3. "Orders with a settled payment" via `$unwind`.**
It over-counts any order with **two or more** settled payments (a split payment, a retried
capture recorded twice): each settled payment is its own row after `$unwind`, and `$count`
counts rows. It doesn't drop orders that *should* be counted. Orders with no payments vanish
at the `$unwind`, but they also have no settled payment, so for this question the inner-join
behaviour is harmless. It would drop them wrongly if the question were "orders, with their
settled payment if any". Fix: a pipeline `$lookup` with `{$match: {status: "settled"}}`,
`{$limit: 1}`, then `{"p.0": {$exists: true}}`.

**4. `$expr` against a partial index.**
The `$lookup` page lists, among the limits on index use for `$expr` comparisons, that
"Multikey, partial, or sparse indexes are not used". The sub-pipeline will scan the foreign
collection once per input document. Change it to the concise form,
`localField: "_id", foreignField: "order"` plus a `pipeline` for the remaining conditions, and
give the foreign collection an ordinary (non-partial) index on `order`. Or add one alongside
the partial index if the partial one exists for a uniqueness rule (chapter 2).

**5. Finding out whether a cached query is a problem.**
Run it with `explain("executionStats")` directly against the database, bypassing the cache,
for a narrow range and a wide one. The tell is the documents examined by the **first** stage
(`$cursor`'s `totalDocsExamined`), and the number of lookups. If they're the same for a
one-day and a one-year range, and close to the size of the whole collection, the cost is
proportional to all of history and will keep growing with it. A healthy pipeline's first-stage
count scales with the range.

---

## Things to try

1. **The null table, yourself.** Create four collections with the four `nid` index
   definitions from chapter 2 and insert two missing, then two `null`, then two `"A"`. Then
   two `5`s into the `$type` one.
2. **Empty arrays.** `createIndex({k: 1}, {unique: true})`, insert `{k: []}` twice and read
   the error's `dup key`. Repeat with `partialFilterExpression: {k: {$type: "string"}}`.
3. **Does the planner use the partial index?** On the `$type` index, compare
   `find({nid: "A"}).explain()` with `find({nid: {$eq: "A", $type: "string"}}).explain()`.
4. **A unique build that fails at the end.** Insert a million documents, start a unique index
   build from one shell, and insert a duplicate from another shell while it runs.
5. **ESR versus ERS.** Build both index orders from chapter 3 and compare
   `totalKeysExamined` and the presence of a `SORT` stage as you make the range more and less
   selective.
6. **Leak test.** Write the scoped and the leaking `$or` from chapter 4 and run both with
   identifiers that match nobody.
7. **Sort order.** Insert documents whose sort field is a date, a string, `null`, missing and
   `[]`, sort both ways, then query with a date range.
8. **Skip placement.** Run `[$skip, $sort, $limit]` and `[$sort, $skip, $limit]` on the same
   data and compare, then look at `explain()` for each.
9. **Driving side.** Rebuild chapter 6's orders and steps, run pipelines A and B with
   `explain("executionStats")`, then double the steps and run them again.

## Questions worth asking me

- "Here's our people schema and its indexes. Which constraints do they actually enforce?"
- "Design the uniqueness for an identifier that can live in any of four fields."
- "We store `null` in some documents and omit the field in others. What breaks, and what's
  the migration?"
- "Is this check-then-insert safe? If not, what index replaces it?"
- "What does `collMod` with `prepareUnique` give me when converting an index to unique on a
  live collection?" (Not covered yet; I haven't checked its behaviour.)
- "How do read and write concerns interact with a unique index on a replica set?"
- "When should this be a transaction instead of a single-document write?"
- "How would sharding the collection change the uniqueness story?"
- "Is this aggregation a candidate for a materialised view or `$merge` into a summary
  collection?"
