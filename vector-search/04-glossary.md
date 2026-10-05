# 4. Glossary

Reference, not reading. Each term links to where it's taught.

| Term | Meaning | Taught in |
|---|---|---|
| **ANN** | Approximate nearest neighbour search: trades occasional misses for speed | [01](01-what-vector-search-is.md) |
| **Collection** | Set of points sharing a vector configuration | [01](01-what-vector-search-is.md) |
| **Cosine similarity** | Cosine of the angle between two vectors; higher is more similar | [01](01-what-vector-search-is.md) |
| **Dual write** | Writing the same change to two stores with no transaction between them | [03](03-keeping-an-index-in-step.md) |
| **`ef` / `hnsw_ef`** | Size of HNSW's candidate list at query time; the main recall/speed dial | [01](01-what-vector-search-is.md) |
| **Embedding** | Fixed-length vector a model produces for an item | [01](01-what-vector-search-is.md) |
| **Exact search** | Scoring every stored vector; ground truth, too slow for large collections | [01](01-what-vector-search-is.md) |
| **HNSW** | Hierarchical Navigable Small World graph, the common ANN index | [01](01-what-vector-search-is.md) |
| **Named vectors** | Several vectors per point, one per model | [01](01-what-vector-search-is.md) |
| **Name-based UUID (UUIDv5)** | UUID computed from a namespace and a name with SHA-1; deterministic, not secret | [03](03-keeping-an-index-in-step.md) |
| **Oversampling** | Fetching extra candidates with quantised vectors before rescoring | [02](02-parameters-that-change-the-answer.md) |
| **Payload** | Fields stored with a point, used for filters and returned with results | [01](01-what-vector-search-is.md) |
| **Point** | One stored item: id, vectors, payload | [01](01-what-vector-search-is.md) |
| **Quantisation (scalar)** | A one-byte-per-dimension copy of each vector, 4× smaller | [02](02-parameters-that-change-the-answer.md) |
| **Recall** | Fraction of the true top-k that a search returned | [01](01-what-vector-search-is.md) |
| **Reconciliation** | Comparing the source of truth with the index and repairing differences | [03](03-keeping-an-index-in-step.md) |
| **Rescoring** | Recomputing candidates' scores with the original vectors | [02](02-parameters-that-change-the-answer.md) |
| **Score threshold** | Minimum (or maximum, for distances) score a result must have | [01](01-what-vector-search-is.md), [02](02-parameters-that-change-the-answer.md) |
