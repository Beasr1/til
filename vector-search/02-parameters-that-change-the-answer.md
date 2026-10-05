# 2. Search parameters that change the answer

## The problem

A vector search returns a list of matches with scores, and code then makes a decision from
them: "anything above 0.8 is a duplicate". Two settings that look like performance knobs
change which items cross that line: quantisation and its rescoring, and, when there are two
models, how their results are combined. A change to either can make a deduplication job report
fewer duplicates, or different ones, without anyone touching the threshold. This chapter
measures both.

Assumes [chapter 1](01-what-vector-search-is.md). Measured on Qdrant 1.19 with the Go client,
in a throwaway container. The data is synthetic: random 128-dimensional unit vectors with
planted neighbours at known cosine similarity, so the true answer is known exactly.

## Quantisation stores a cheaper copy of every vector

A float32 vector of 512 dimensions takes 2 KB. Held in memory for a hundred million points
it's 200 GB. **Scalar quantisation** stores a second copy of each vector at one byte per
dimension. From Qdrant's [quantization guide](https://qdrant.tech/documentation/guides/quantization/):
it performs "`float32 -> uint8` conversion, reducing memory requirements by a factor of 4", and
"The error introduced by scalar quantization is usually less than 1%". The `quantile` sets the
range the 256 levels cover: with 0.99, "1% of extreme values will be excluded from the
quantization bounds."

Search can then use the cheap copy. Three query-time parameters decide how:

| Parameter | Documentation |
|---|---|
| `ignore` | "Toggle whether to ignore quantized vectors during the search process." Default: use them |
| `rescore` | "re-evaluate top-k search results using the original vectors" |
| `oversampling` | "how many extra vectors should be pre-selected using quantized index, and then re-scored using original vectors" (2.4 × limit 100 = 240 candidates) |

So with quantisation on and rescoring off, the **scores you get back are computed from the
quantised vectors**. They're approximations, and a threshold is applied to the approximation.

## Measured: what happens at a threshold

Setup: 30,000 random vectors, plus 100 query vectors each with 20 planted neighbours. Threshold
0.80, limit 50. Ground truth by computing every cosine in Go. Four search modes.

**Run 1:** neighbours spread over cosine 0.70–0.90 (a gap of about 0.01 between neighbours).

| Mode | Truly ≥ 0.80 | Returned correctly | Missed | Returned but truly < 0.80 | Max score error |
|---|---|---|---|---|---|
| Exact | 1,000 | 1,000 | 0 | 0 | 0.0000 |
| HNSW, quantised, no rescore | 1,000 | 1,000 | 0 | 0 | 0.0172 |
| HNSW, quantised, rescore, oversampling 3 | 1,000 | 1,000 | 0 | 0 | 0.0000 |
| HNSW, quantisation ignored | 1,000 | 1,000 | 0 | 0 | 0.0000 |

**Run 2:** the same, but neighbours packed into cosine 0.79–0.81, all within 0.01 of the
threshold.

| Mode | Truly ≥ 0.80 | Returned correctly | Missed | Returned but truly < 0.80 | Max score error |
|---|---|---|---|---|---|
| Exact | 1,000 | 1,000 | 0 | 0 | 0.0000 |
| HNSW, quantised, no rescore | 1,000 | 892 | **108** | **70** | 0.0073 |
| HNSW, quantised, rescore, oversampling 3 | 1,000 | 1,000 | 0 | 0 | 0.0000 |
| HNSW, quantisation ignored | 1,000 | 1,000 | 0 | 0 | 0.0000 |

Three things to take from these tables:

1. **The score error was small and the effect wasn't.** Errors under two hundredths changed
   nothing in run 1 and moved 178 of 1,000 decisions in run 2. What matters is how much of your
   data sits near the threshold, and for a deduplication threshold that's exactly where the
   interesting cases are.
2. **"Usually less than 1%" and a 0.0172 error aren't in conflict, and aren't comparable.** The
   guide doesn't say what the percentage is of: recall, score, or something else. A maximum
   absolute score error of 0.0172 on scores near 0.8 is about 2% of the score. Measure the
   quantity your decision depends on.
3. **Rescoring removed the error entirely here**, because the final scores came from the
   original vectors. With HNSW on this small dataset, recall was perfect in every mode; on a
   large collection, a low `ef` can also lose true neighbours before rescoring ever sees them.
   That's a separate effect from quantisation, measured separately.

> ⚠️ **A threshold applied to unrescored quantised scores is a different threshold.** Turning
> rescoring off for speed, or turning quantisation on for memory, changes which items pass "≥
> 0.80", near the boundary, in both directions. Treat either change like a change of threshold:
> measure it against exact search on real data before shipping.

## Combining two models by intersecting their top-k lists

With two embedding models per item (two named vectors), a natural rule for "this is a match"
is "it's a match in both". The easy implementation: search each vector with `limit = k` and
the threshold, then keep the ids that appear in both lists. It's not the same rule.

Measured: one target point scores 0.85 against the query in *both* models, so it passes the
0.80 threshold in each. Ten other points score 0.95 in the first model and 0.10 in the second,
look-alikes under model 1 only.

```
limit=10: e1 returned 10, e2 returned 1, true match in e1 list=false, intersection=[]
limit=11: e1 returned 11, e2 returned 1, true match in e1 list=true,  intersection=[1]
```

With `limit = 10`, model 1's ten slots go to the look-alikes, the true match is ranked eleventh
and cut, and the intersection is empty. The point passes both thresholds and is reported as no
match at all. With one more slot it appears. In real data the "look-alikes" are just the many
items that are close under one model, and the more items in the collection, the more of them
crowd the top k.

What to do instead:

| Approach | Behaviour |
|---|---|
| Search one model with a generous limit, then score those candidates against the other model's vectors | Finds every item that passes both thresholds, if the first search's limit covers all items above its threshold |
| Search each with a limit large enough that no item above the threshold is cut (check: did the list fill to the limit?) | Correct, but the limit grows with the collection |
| Fuse the two scores into one ranking | A different rule; needs the scales calibrated first |

Two more traps in the combine step. Taking the **minimum** of two models' scores as "the" score
assumes the scales are comparable, which they aren't (chapter 1). And a code path that
initialises a running minimum to `1.0` silently caps every result at 1.0 for a metric, such as
dot product, whose scores can exceed it.

> **Teacher's aside.** A top-k list is an answer to "what are the k closest?", and nothing
> else. Every time you use it to answer a different question, "is anything above the
> threshold?", "is this item close under both models?", you're betting that the answer to the
> second question fits inside the first one's k slots. Make the bet explicit: check whether the
> list came back full, and if it did, the answer may have been cut.

## Check yourself

1. A team turns on scalar quantisation with rescoring off to save memory. Their deduplication
   job, threshold 0.80, reports a different set of duplicates. Explain why, and what you'd
   measure before deciding whether it matters.
2. In run 1, quantised search without rescoring had no wrong decisions despite a 0.0172 score
   error. Why, and why does that tell you little about production?
3. What does `oversampling = 3` with `limit = 10` do, step by step, and which error does it
   fix?
4. Construct, in words, a dataset where intersecting two top-10 lists misses a point that both
   models score above threshold.
5. A combined score is `min(score_model_a, score_model_b)`, initialised to `1.0`. Name two
   separate ways it can misrank results.
6. How would you detect, at query time and without exact search, that a top-k list may have cut
   items that pass the threshold?
