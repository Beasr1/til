# Vector Search from the Application Side — A Course

A course on what a vector database does with your embeddings, which of its settings change the
answers you build decisions on, and how to keep it consistent with the store it copies.

**This is reference learning material.** The ideas are general to approximate nearest-neighbour
search; the engine used for examples and measurements is Qdrant 1.19, run in a throwaway
container with the Go client on synthetic data whose true answers are known. Standards are
quoted from RFC 9562 and the HNSW paper; engine behaviour from Qdrant's documentation. The
motivating failures are ordinary: a deduplication job whose matches changed when quantisation
was switched on, a match that passed every threshold and was dropped by a top-k intersection,
two points for one item after an id format changed, and index writes that failed with no
lasting record.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the rule that prevents it.
- Every chapter ends with **Check yourself** questions. Answers are in
  [05-exercises.md](05-exercises.md).
- Measurements are labelled as measured, with the setup.

## The one thing to understand first

> **A vector search answers "what are the k closest items, approximately?", and nothing else.
> Every decision built on it, "is there a match above 0.8?", "is this close under both
> models?", "is this item already indexed?", is a second question you're answering with the
> first one's result, and the gap between the two is where the bugs live.**

## Reference implementations

| Source | What it settles |
|---|---|
| [Malkov & Yashunin, HNSW](https://arxiv.org/abs/1603.09320) | The layered graph and why search scales logarithmically |
| [Qdrant: search](https://qdrant.tech/documentation/search/search/) | `hnsw_ef`, `exact`, `score_threshold` direction |
| [Qdrant: quantization](https://qdrant.tech/documentation/guides/quantization/) | Scalar quantisation, `quantile`, `ignore`, `rescore`, `oversampling` |
| [Qdrant: points](https://qdrant.tech/documentation/manage-data/points/) | Id types, overwrite on same id, idempotent APIs, `wait` |
| [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html) | §5.5 UUIDv5, §6.6 well-known namespaces, §8 UUIDs aren't secrets |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What vector search is](01-what-vector-search-is.md) | ⭐ Explain embeddings, similarity measures, exact versus approximate search, HNSW and `ef`, recall, and points, payloads and filters. **Prerequisite for everything else** |

### Part 1 — Using it

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Search parameters that change the answer](02-parameters-that-change-the-answer.md) | ⭐ Predict how quantisation and rescoring move a threshold decision, and combine two models without losing matches |
| 3 | [Keeping an index in step](03-keeping-an-index-in-step.md) | Design deterministic point ids, change them safely, and make a dual write recoverable |

### Reference

| # | File | |
|---|------|---|
| 4 | [Glossary](04-glossary.md) | Terms, each linked to its chapter |
| 5 | [Exercises & answers](05-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. Chapters 2 and 3 are independent of each other. Not yet written, and
belonging here: choosing a threshold from labelled data, filtered search and payload indexes,
other quantisation methods, sharding and replication of a collection, and score fusion across
models. The questions at the end of file 05 are the honest list.

## If you're short on time

- **New to vector search:** chapter 1, all of it.
- **10 minutes:** chapter 2's run 2 table and the trap after it.
- **Matches changed after a configuration change:** chapter 2, quantisation section.
- **Combining two embedding models:** chapter 2, the intersection section.
- **Designing point ids or a backfill:** chapter 3 top to bottom.

## The one-paragraph summary of everything

A model turns each item into an embedding, and only distances between embeddings from the same
model mean anything; a score is a similarity, not a probability, and a threshold belongs to one
model. Exact search scores every vector and is the ground truth for testing; production uses
approximate search, usually HNSW, a layered graph walked greedily from the top, whose
query-time `ef` trades speed for recall, and which can miss a true neighbour silently. Scalar
quantisation stores a one-byte-per-dimension copy, four times smaller; without rescoring, the
returned scores are the quantised approximations, and a threshold applied to them moves
decisions near the boundary (measured: 108 of 1,000 true matches missed and 70 false ones
returned when every candidate sat within 0.01 of the threshold, none with rescoring on).
Intersecting two models' top-k lists isn't "close under both": look-alikes under one model fill
its k slots and push a true match out (measured: empty intersection at k = 10, found at 11).
Point ids should be a deterministic function of the item, a UUIDv5 of its natural key with a
fixed namespace, an unambiguous separator and fixed normalisation, so retries and backfills
overwrite instead of adding; such ids aren't secret, and changing how they're derived gives
every item a second, orphaned point. Writing the primary store and the index is a dual write
with no transaction: write the source of truth first, record index failures durably before
acknowledging, wait for writes to apply, and reconcile from the source.

## How to use me

Ask me things like:

- "Here's our search request. Which parameters can change who counts as a match?"
- "Is our point id scheme stable across every writer and environment?"
- "We're turning on quantisation to save memory. What should we measure first?"
- "How do we know whether the index is missing items the primary store has?"
- "Here's how we combine two models' results. What can it miss?"
