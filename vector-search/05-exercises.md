# 5. Exercises

Worked answers to every **Check yourself** question, then things to try and questions
worth asking me.

---

## File 01 — What vector search is

**1. Two models' embeddings in one vector field.**

Searches return results ranked by a number that means nothing across the two groups. A query
from model A compared with vectors from model B gives an arbitrary score, so B's items appear or
vanish from A's results at random, and thresholds stop meaning anything. Nothing errors because
the engine only checks dimension and distance: if both models output vectors of the same length,
it has no way to know they live in different spaces. Use named vectors (one per model) or
separate collections.

**2. "A 0.8 threshold" before the model is chosen.**

The threshold has no meaning without the model and the distance measure: each model spreads
scores differently, so 0.8 can be strict for one and loose for another. What's missing is
labelled pairs (same / different) scored with the chosen model, and an agreed trade-off between
false matches and missed matches. The threshold falls out of those; it can't come first.

**3. `exact: true` in a recall test, not in production.**

Recall is defined against the true top-k, which only a full scan gives you, so the test needs
it. In production, exact search scores every vector for every query: for a hundred million
512-dimensional vectors, about fifty billion multiplications per query, so latency grows with
the collection.

**4. `ef = 16` misses what `ef = 256` finds.**

At the bottom layer the search keeps only `ef` candidates while it walks the graph. With 16, it
follows the most promising few paths and stops when none of their neighbours improve the list,
so a true neighbour reachable only through a path that looked worse early on is never visited.
With 256 it keeps many more paths alive and reaches it. Nothing is wrong with the index; the
search was allowed to look at less of it.

**5. Filter inside the search, not afterwards.**

Searching for ten and then dropping the author's items returns fewer than ten whenever the
author's items were among the closest, possibly zero if the author has many near-duplicates.
With the exclusion in the query's filter, the engine returns the ten nearest items that pass it.

## File 02 — Search parameters that change the answer

**1. Quantisation on, rescoring off, different duplicates.**

The scores come from the quantised vectors, so each is off by a small amount, and items close
to 0.80 cross the threshold in both directions. Measured: with all candidates within 0.01 of the
threshold, 108 of 1,000 true matches were missed and 70 non-matches returned. Before deciding,
measure on real data how many scored pairs fall within the observed score error of the
threshold, and compare the duplicate sets from quantised-without-rescore and exact search on a
sample. If the difference matters, turn rescoring on.

**2. Run 1: no wrong decisions despite a 0.0172 error.**

The planted neighbours were about 0.01 apart, so the closest to the threshold sat roughly 0.005
either side of it, and none of their errors happened to be large enough, in the wrong direction,
to cross. The 0.0172 maximum was somewhere else; the run didn't record where. It tells you little
because production data isn't spaced for you. Real duplicates cluster near whatever threshold you chose, as run 2 shows.

**3. `oversampling = 3`, `limit = 10`.**

The search uses the quantised vectors to pick 30 candidates (3 × 10), recomputes those 30
scores with the original vectors, and returns the best 10 by the recomputed scores. It fixes
ranking and threshold errors caused by quantisation among the candidates. It can't recover a
true neighbour that the quantised search didn't put in the 30, so it narrows the error rather
than eliminating it in general (here it eliminated it).

**4. A dataset that defeats top-10 intersection.**

One target item is moderately close to the query under both models, above threshold in each.
Ten or more other items are very close under model A only, and far under model B. Model A's
top 10 is all look-alikes, so the target ranks below the cut, and the intersection, which needs
it in both lists, is empty. Measured exactly like that: an empty intersection at limit 10, the
target found at limit 11.

**5. `min(score_a, score_b)` initialised to 1.0.**

First, the two models' scores aren't on the same scale, so the minimum mostly reflects whichever
model scores lower in general, not which match is weaker. Second, starting the running minimum
at 1.0 caps every combined score at 1.0, which is wrong for a metric like dot product whose
scores can exceed 1, and squashes differences between strong matches.

**6. Detecting a cut list without exact search.**

If the list came back with exactly `limit` results and the last one is still above the
threshold, there may be more items above the threshold that were cut. Then either raise the
limit and repeat until the last result falls below the threshold, or switch to the
candidates-then-score approach.

## File 03 — Keeping an index in step

**1. `kind + ":" + key` versus `kind + key`.**

Different names give different UUIDs, so each item ends up with two points, one per service.
Searches return both copies, so every item appears as its own near-perfect duplicate, and any
"is this already in the index?" check finds a match for everything. Updates from each service
touch only its own copy, so the copies drift apart.

**2. Separator byte, or length prefix.**

A separator that can't occur inside any field makes the joined string uniquely splittable: each
separator marks a boundary and nothing else, so different tuples can't produce the same string.
If no such byte can be guaranteed, prefix each field with its length (`3:abc5:de:fg`), which is
unambiguous for any content.

**3. "Anonymised because SHA-1".**

Check whether the namespace is known (a constant in code, a published namespace such as the DNS
one, a default in a library) and how large the space of names is. If identifiers are guessable,
anyone with the namespace can compute ids for every candidate name and match them. RFC 9562 §8:
"Implementations SHOULD NOT assume that UUIDs are hard to guess. For example, they MUST NOT be
used as security capabilities."

**4. Shortened prefix in the id name.**

Items written before the change have points under the old ids. The next write for each creates a
second point under the new id; the old one stays, matches queries and is never updated or
deleted. New items get only new ids. The change has to ship with a migration: delete-old-on-write
using both formulas, a rebuild of the collection, or a scheme that keeps old ids for existing
items.

**5. Index write fails, dead-letter publish fails, input acknowledged.**

The primary store has the item. The index doesn't. The dead-letter topic doesn't. The input is
acknowledged, so it won't be redelivered. The only trace is a log line. Unless a reconciliation
job compares the primary store with the index, the item is unsearchable for ever. The
acknowledgement should have waited for a successful dead-letter write.

**6. `wait = false`.**

Without waiting, Qdrant acknowledges the request before applying it, so "no error" means
"accepted", not "searchable now". An immediate search may not find the point, and a failure
during processing isn't reported to the writer. It's acceptable for bulk loads followed by a
check, or when a short delay before items become searchable doesn't matter. It isn't acceptable
when the next step searches for what was just written, or when the write's success decides
whether anything is retried.

---

## Things to try

- Build a collection with scalar quantisation, plant neighbours around a threshold, and compare
  exact, quantised, rescored and ignored searches. Then vary the quantile and `ef`.
- Store two named vectors per point, create ten look-alikes under one of them, and watch a top-k
  intersection lose the true match as the look-alikes grow in number.
- Compute UUIDv5 ids for the same names in two languages' standard libraries and check they
  agree; then change the separator and watch every id change.
- Write points with `wait = false`, search immediately, and count how often the point is missing.

## Questions worth asking me

- "Here's our similarity pipeline. Which settings change which items count as a match?"
- "How should we choose a threshold for this model, and how often should we revisit it?"
- "How do filtering and HNSW interact when the filter is very selective?"
- "What does product quantisation or binary quantisation change compared with scalar?"
- "How do we fuse scores from two embedding models properly?"
- "What's the cheapest reconciliation job that would catch a missing point within a day?"
