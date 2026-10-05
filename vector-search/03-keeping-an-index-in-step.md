# 3. Keeping a vector index in step with its source of truth

## The problem

A vector database is rarely the only copy. The items live in a primary store, and the vector
index is a second, searchable copy, written by the same service, by a migration job, by a
backfill. Every one of those writers has to produce the *same point* for the same item, or the
index fills with duplicates and orphans: two points for one item, both matching a query, and
neither ever deleted. And the two stores can't be written atomically, so some writes will reach
one and not the other. This chapter is about point identity and about the dual write.

Assumes [chapter 1](01-what-vector-search-is.md) (points, ids, payloads). Standards are quoted
from RFC 9562; engine behaviour from Qdrant's documentation.

## Make the point id a function of the item

Qdrant's documentation: "points with the same id will be overwritten when re-uploaded", and its
APIs are idempotent, "executing the same method several times in a row is equivalent to a single
execution." So if every writer computes the same id for the same item, re-sending a point is
harmless: a retry, a redelivered message and a backfill all overwrite rather than add.

That rules out random ids for anything that can be written twice. The standard tool is a
**name-based UUID**. RFC 9562 §5.5: "UUIDv5 values are created by computing an SHA-1 hash over a
given Namespace ID value concatenated with the desired name value". Same namespace and same name
always give the same UUID, in any language.

The design decisions are all about the *name*:

| Decision | Rule | What goes wrong otherwise |
|---|---|---|
| Which fields | The item's natural key, and nothing that changes | A field that changes (a status, a category) gives the same item a new id each time it changes |
| How they're joined | A separator that can't occur in any field, or a length prefix | `"a" + ":" + "b:c"` and `"a:b" + ":" + "c"` are the same string, so two items share an id |
| Normalisation | Fixed case, trimming and encoding, written down | One writer upper-cases a hex digest and another doesn't: two ids per item |
| Namespace | One constant, identical in every writer and every language | Different namespaces give disjoint id spaces; nothing ever converges |

> ⚠️ **Don't let the namespace default silently.** If the namespace comes from configuration with
> a fallback ("use the DNS namespace if unset"), one deployment that misses the setting computes
> different ids for everything it writes, and nothing errors. Either hard-code the namespace or
> refuse to start without it.

## A name-based id isn't a secret

It's tempting to think a UUIDv5 hides the name it came from, so ids built from personal
identifiers are safe to expose. RFC 9562 §8: "Implementations SHOULD NOT assume that UUIDs are
hard to guess. For example, they MUST NOT be used as security capabilities." Anyone who knows the
namespace can compute the id of any name they can guess, and identifiers usually come from small,
guessable spaces. The well-known namespaces are published: §6.6 lists the DNS namespace as
`6ba7b810-9dad-11d1-80b4-00c04fd430c8`, and it's a common default in libraries. Treat a name-based
id as exactly as sensitive as the name.

## Changing how ids are derived forks identity

Every change to the name, a new prefix, a shorter format to fit a length limit, a different
separator, upper-case instead of lower-case, gives every existing item a new id. The next write
for each item creates a new point next to the old one. Both match queries. The old one is never
updated again and never deleted, because nothing knows its id any more.

So an id-format change is a data migration, and it needs one of these:

- Compute the old and new ids side by side, and delete the old point when writing the new one.
- Rebuild the collection from the primary store under the new scheme and switch over.
- Keep the old format for existing items (a version marker in the name) and use the new one only
  for new items.

The same applies to any other store keyed by a derived value: change how a lookup key is hashed
and old rows stop matching, so the next write for each entity creates a duplicate.

## The dual write

The service writes the primary store, then the index. There's no transaction across them, so
four outcomes are possible, and two of them are inconsistent:

| Primary | Index | Result |
|---|---|---|
| ok | ok | consistent |
| ok | failed | item exists but can't be found by search |
| failed | not attempted | consistent (nothing written) |
| failed after partial write | ok | depends on the primary store's guarantees |

The usual design accepts the second row and repairs it: write the primary store first, treat it
as the source of truth, and when the index write fails, record the item somewhere durable (a
dead-letter topic, a "needs indexing" table) for a **reconciliation** job to re-send. Because
ids are deterministic and upserts idempotent, re-sending is safe however many times it happens.

Three details decide whether that repair actually happens:

- **Record the failure with the item's key, durably, before acknowledging the input.** A
  dead-letter write that can itself fail, logged and ignored, turns "index failed" into "index
  failed and nobody will ever know" (the same rule as [kafka/03](../kafka/03-offsets-and-delivery-guarantees.md)).
- **Wait for the index write to be applied.** Qdrant's `wait` parameter makes an upsert return
  only after the change is applied; without it, the request is acknowledged and processed
  asynchronously, so "no error" doesn't mean "searchable".
- **Reconcile from the source, not only from the failure log.** A periodic job that compares the
  primary store with the index catches the failures nobody recorded, deletes that never
  propagated, and points orphaned by an id-format change.

Payload fields copied into the index for filtering (an owner, a status, a category) are part of
the dual write too. When they change in the primary store, the point has to be rewritten, or
filters and post-search rules act on stale values.

> **Teacher's aside.** People design the vector index as if it were a cache: fast, derived,
> safe to lose. Then they build decisions on it, "no match above 0.8 means this is a new
> person", and a missing point stops being a performance problem and becomes a wrong answer. If
> decisions depend on the index being complete, the reconciliation job isn't housekeeping, it's
> part of correctness. Ask what a missing point costs before deciding how much repair to build.

## Check yourself

1. Two services write points for the same items. One builds the UUIDv5 name as
   `kind + ":" + key`, the other as `kind + key`. What's in the collection after both have run
   for a week, and what do searches return?
2. Why does a separator byte that can't occur in any field remove the ambiguity, and what's the
   alternative when no such byte exists?
3. A team says the point ids are "anonymised" because they're SHA-1 based. What would you check,
   and what does RFC 9562 say about relying on it?
4. To fit a length limit in another system, the id name's prefix is shortened. What happens to
   items written before and after the change, and what has to ship with it?
5. The index write fails, and the code logs the error, tries to publish to a dead-letter topic,
   that fails too, and acknowledges the input. Trace what the system knows about this item
   afterwards.
6. Why does writing with `wait = false` make "the write returned without error" a weaker
   statement, and when is that acceptable?
