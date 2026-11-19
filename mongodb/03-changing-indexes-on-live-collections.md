# 3. Changing indexes on live collections

## The problem

A team makes an identifier unique across all tenants instead of within each one. They change
the index declared on the model and deploy. On boot, the application's index sync drops the
old per-tenant index, because it's no longer declared, then tries to build the new global one.
That build fails: two tenants already share an identifier. The application logs the error and
carries on serving. The collection now has **no** uniqueness at all, and nothing says so.

Each step there was a documented feature doing its job. This chapter is about the order of
operations that avoids it: how an index build behaves on a busy collection, who should run
it, how to tighten a constraint on data that may already violate it, and how to order a
compound index's fields in the first place. It assumes [chapter 1](01-what-mongodb-does-with-a-query.md)
(compound indexes, replica sets) and [chapter 2](02-uniqueness.md) (unique and partial
indexes).

## What an index build does to a busy collection

Before MongoDB 4.2 you chose between a foreground build, which blocked the database, and a
`background: true` build, which was slower. Since 4.2 there's one process. The 4.2 release
notes (in the [mongodb/docs repository](https://github.com/mongodb/docs), branch `v4.2`,
`source/release-notes/4.2.txt`): "MongoDB uses an optimized build process that only holds the
exclusive lock at the beginning and end of the index build. The rest of the build process
yields to interleaving read and write operations." And: "MongoDB ignores the background index
build option". Code that still passes `background: true` is harmless and does nothing.

The current manual ([Index Builds on Populated Collections](https://www.mongodb.com/docs/manual/core/index-creation/))
gives the stages:

| Stage | Lock on the collection | Effect on traffic |
|---|---|---|
| Start | Exclusive (`X`), briefly | Reads and writes wait for a moment |
| Scan the collection, build keys | Intent exclusive (`IX`), yielded periodically | Reads and writes proceed; writes made meanwhile are recorded on the side |
| Drain the side writes | Shared (`S`) | Writes wait while the backlog is applied |
| Commit | Exclusive (`X`), briefly | Reads and writes wait for a moment |

"Briefly" isn't "free". The build reads the whole collection and competes for CPU, disk and
cache; the manual warns that building "when the target collection is under heavy write load
can result in reduced write performance and longer index builds". A server runs at most three
user index builds at once by default (`maxNumActiveUserIndexBuilds`), and self-hosted servers
give each `createIndexes` command 200 MB of memory before spilling to temporary files.

### Unique builds fail at the end, not the start

During the scan, writes continue, and they aren't checked against the index being built. The
manual: "Documents that violate the index constraints can exist during the index build. If
any documents violate the index constraints at the end of the build, the `mongod` terminates
the build and throws an error." Since 7.1 most index errors surface during the scan, but
duplicate keys are the stated exception. So a unique build on a large collection can run for
an hour and then fail, and the duplicate that kills it can be one written *during* the build.
The manual's advice: validate first, and stop writes from anything that can't guarantee it
won't create a violation.

Reproduced: a global unique index on data where two tenants shared `nid: "A"`:

```
Index build failed: ... :: caused by :: E11000 duplicate key error collection: lesson.scan
index: nid_global dup key: { nid: "A" }
```

### On a replica set the build runs everywhere

The primary starts the build and every data-bearing member builds at the same time. The
primary commits only when a **commit quorum** of members is ready, by default
`"votingMembers"`, all data-bearing voting members. The manual's warning: "If a data-bearing
voting node becomes unreachable and the commitQuorum is set to the default `votingMembers`,
index builds can hang until that node comes back online."

When the default build would hurt too much, the alternative is a **rolling build**
([Rolling Index Builds on Replica Sets](https://www.mongodb.com/docs/manual/tutorial/build-indexes-on-replica-sets/)):
take one secondary out, restart it as a standalone on another port, build the index there,
rejoin it and let it catch up, repeat for each secondary, then step the primary down and do
it last. The manual limits it to deployments under real pressure (average CPU above
`(N-1)/N - 10%` of `N` threads, or WiredTiger cache regularly above 90% full), says it
"lower[s] the resiliency of your cluster and increase[s] build duration", and for unique
indexes requires that "you must stop all writes to the collection during this procedure".
If you can't, build on the primary the normal way.

## Who runs the build

Declaring indexes next to the schema is good: the index definitions are reviewed with the
code that depends on them, and there's one place to look. The question is what turns the
declaration into a build.

**Mongoose's default is: every process, on startup.** "When your application starts up,
Mongoose automatically calls `createIndex` for each defined index in your schema." The same
page recommends "this behavior be disabled in production" because "index creation can cause a
significant performance impact", via `autoIndex: false` on the schema or connection
([Mongoose guide, Indexes](https://mongoosejs.com/docs/guide.html#indexes)).

**`Model.syncIndexes()` goes further: it drops.** It "will drop any indexes that are not
defined in the model's schema except the `_id` index" and then builds what's missing
([Mongoose Model API](https://mongoosejs.com/docs/api/model.html)). That includes an index a
colleague created by hand during an incident, and an index another service that shares the
collection relies on. `Model.diffIndexes()` is the dry run: it returns `{ toDrop, toCreate }`
without changing anything.

What goes wrong when that runs on every replica's boot:

- **Builds start during a rolling deploy**, at the moment capacity is already reduced, and on
  every deploy that changes a declaration.
- **Drops happen without review**, whenever the deployed schema and the database disagree,
  including the window where old and new versions of the service are both running.
- **Failures are easy to miss.** A failed build at boot is a log line in one of N processes.
  If the old index was already dropped, the constraint is gone until someone reads the logs.
- **Running it at import time is worse.** Calling sync as a side effect of loading the model
  file runs it in every process that imports the model (workers, CLIs, test runners), and
  possibly before the connection is ready.

The fix is to make index changes **a deliberate step with one runner**: a migration script
run once by the deploy pipeline, or a startup step enabled by a flag on exactly one job.
Either way it should run after the connection is up, be awaited, fail loudly, and be
reviewable. The declared indexes stay in the code, so `diffIndexes()` in CI can tell you when
the database and the code disagree.

> **Teacher's aside.** People hear "sync" and think "make the database match the code",
> which sounds safe. Read it as "**drop** whatever the code doesn't mention, then **try** to
> build the rest". The drop always succeeds; the build can fail. Any process that might run
> half of that operation is a process that can remove a constraint.

## Tightening a constraint without a gap

Here's the order that never leaves the collection unprotected, for moving from a per-tenant
unique index to a global one.

```mermaid
flowchart TD
    A[1. Scan for violators<br/>with the new index's filter] --> B{Any?}
    B -- yes --> C[2. Reconcile them<br/>a human decision, not a script default]
    C --> A
    B -- no --> D[3. Stop new violators<br/>app-level check, or pause writers]
    D --> E[4. Create the NEW index<br/>under a new name]
    E --> F{Built?}
    F -- no --> A
    F -- yes --> G[5. Drop the OLD index]
    G --> H[6. Declared schema matches the database<br/>so the next sync is a no-op]
    style E fill:#d4edda,stroke:#155724
    style G fill:#fff3cd,stroke:#856404
```

**1. Scan with the new index's exact filter.** Use the same `partialFilterExpression` as a
`$match`, or the scan will report duplicates the index would ignore, or miss ones it won't:

```js
db.people.aggregate([
  { $match: { deleted: false, nid: { $type: "string" } } },   // the new index's filter
  { $group: { _id: "$nid", n: { $sum: 1 }, ids: { $push: "$_id" } } },
  { $match: { n: { $gt: 1 } } }
], { allowDiskUse: true })
```

For a derived key array ([chapter 2](02-uniqueness.md)), unwind it and count distinct
documents per key, because one document may repeat a key legitimately:

```js
db.people.aggregate([
  { $match: { docKeys: { $type: "string" } } },
  { $unwind: "$docKeys" },
  { $group: { _id: "$docKeys", ids: { $addToSet: "$_id" } } },
  { $match: { "ids.1": { $exists: true } } }      // two or more distinct documents
])
```

Make the script's default mode report-only, with a non-zero exit when it finds anything, and
make the destructive mode refuse to run while violators exist.

**2. Reconcile.** Which of two people keeps a shared identifier is a business decision.
"Keep the oldest" is a reasonable default for a backfill; it isn't automatically right.

**3. Stop new violators** for the duration of the build, because the build only fails at the
end ([above](#unique-builds-fail-at-the-end-not-the-start)). An application-level check
narrows the window; pausing the writers closes it.

**4. Create the new index first, under a new name.** MongoDB will hold two indexes with the
same key and different options if the names differ ([chapter 2](02-uniqueness.md)). Until it
finishes, the old index is still enforcing the weaker rule.

**5. Then drop the old one.** Now there's no moment with neither.

**6. Match the declaration.** Use the same key, options and name in the model as in the
script, so the application's own sync, if it still runs, has nothing to do.

## Ordering a compound index: equality, sort, range

Most index changes are for speed, not constraints, and the most common mistake is field order.
MongoDB's guideline ([The ESR (Equality, Sort, Range) Guideline](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/)):

| Position | Field kind | Why |
|---|---|---|
| First | **Equality**: exact match on one value | "Placing equality fields first keeps the remaining index fields in sorted order" |
| Next | **Sort** | "To avoid in-memory sorts, put sort fields before range in the index" |
| Last | **Range**: `$gt`, `$lt`, and also `$ne`, `$nin` and `$regex` | Range bounds read a span of keys, which breaks the sort order of anything after them |

The guide allows one exception: "If your range predicate in the query is very selective,
then put it before sort fields (ERS)."

Reproduced on 20,000 orders, for `find({tenant: "t7", total: {$gt: 400}}).sort({createdAt: -1}).limit(10)`
(about 20% of each tenant's orders match the range):

```
{tenant, total, createdAt}   ERS   FETCH <- SORT <- IXSCAN   keys 200  docs 10
{tenant, createdAt, total}   ESR   LIMIT <- FETCH <- IXSCAN  keys 30   docs 10
```

With ERS, the server read every key in the range and sorted them in memory to find ten. With
ESR, it walked the tenant's keys newest first, checked `total` on each key without fetching,
and stopped after ten matches. The trade flips when the range matches very few documents:
then reading those few and sorting them is cheaper than walking the sort order hoping to meet
them.

## Keep connection strings out of logs

A last habit that belongs with deploy-time database code: startup paths like to log "connected
to …", and a MongoDB connection string usually carries a username and password
(`mongodb+srv://user:password@host/…`). The OWASP
[Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
lists "Database connection strings" among the data to exclude from logs. Log the host and
database name, or a redacted form: Go's `url.URL.Redacted()` "replaces any password with
`xxxxx`". Logs are copied to more places, and kept longer, than secrets stores.

## Check yourself

1. A service still passes `{ background: true }` to every `createIndex`. A colleague says
   removing it will make builds block the collection. Are they right, and what does block
   during a build?
2. A unique index build on a large collection runs for forty minutes, then fails with a
   duplicate key. The scan for duplicates beforehand found none. Give a sequence of events
   that explains it.
3. Ten replicas of a Node service run `Model.syncIndexes()` on boot. A colleague added an
   index by hand last week to fix a slow dashboard query, without declaring it. What happens
   on the next deploy, and why might nobody notice for a day?
4. You're moving `{ tenant: 1, nid: 1 }` unique to `{ nid: 1 }` unique. Your script drops the
   old index, then creates the new one. What state is the collection in if the create fails,
   and what's the corrected order?
5. A query filters `{ tenant: T, status: { $in: ["paid", "refunded"] } }` and sorts by
   `createdAt` descending. Is `$in` equality or range for ESR purposes here? Propose an index,
   and say what you'd check in `explain` to confirm it.
