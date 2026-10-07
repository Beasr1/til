# 15. Exercises & answers

Every "Check yourself" question, with worked answers. Commit to an answer before you read one.
The questions are built so that a half-formed model gives a confident wrong answer, and that
wrong answer is the useful part.

---

## File 1 — Foundations

**1. "My O(n) beats your O(n log n)." Two reasons it might not.**
Constants: log₂ n is only about 17 at n = 10^5, so if the O(n) code does 20 hash-map operations
per element and the O(n log n) code is one tight sort, the sort can win. Memory layout: an O(n)
walk over a linked list or a hash table's scattered entries misses the cache constantly, while a
sort over a contiguous array streams through it. (A third: "O(n)" might be an *expected* bound
that degrades on adversarial input, and the other a worst case.)

**2. Hash lookup is O(1): what kind of bound, and an input that makes it worse.**
Expected (average) time, assuming the hash spreads keys evenly; insert is also amortised over
resizes. Worst case is O(n) per lookup, when keys collide. Concretely: with an identity hash for
integers (what `std::hash<int>` did on the libc++ I tested), keys that are all multiples of the
bucket count land in one bucket, and every lookup scans them all.

**3. Growing by 1,000 slots each time.**
Resizes happen at sizes 1,000, 2,000, 3,000, …, copying that many elements each time. For n
appends there are about n/1,000 resizes, copying 1,000 × (1 + 2 + … + n/1,000) ≈ 1,000 ×
(n/1,000)²/2 = n²/2,000 elements in total. That's **O(n²) total, O(n) amortised per append**. The
constant 1/2,000 is small, which is why it looks fine in testing and falls over at scale.
Multiplying the capacity instead makes the copies a geometric series bounded by about 2n.

**4. n ≤ 2·10^5, values up to 10^9, an O(n²) idea.**
4·10^10 operations is out no matter how small the constant, so stop polishing it. Look for the
structure that gets you to O(n log n): sort, hash map, heap, window, binary search on the answer.
About the values: a sum of two fits easily in Go's 64-bit `int`; a product of two (10^18) is just
under the int64 limit of about 9.2·10^18, so one product is fine and a sum of several products
isn't. If you'd chosen `int32` anywhere, even a sum of three values overflows, silently, since Go
wraps.

**5. JavaScript recursive DFS on a path of 10^5 nodes.**
It throws `RangeError: Maximum call stack size exceeded`. Each call holds a frame until its
subtree finishes, and on a path the subtree is everything below, so depth reaches 10^5. Node
managed about 10,400 frames of a trivial function in my test, and real frames are bigger. The
options: rewrite the traversal with an explicit stack (an array you push and pop), or, if the
order doesn't matter, use BFS with a queue. Raising the stack limit is usually not an option on a
judge.

**6. Optimal idea in 15 minutes, bug found by the interviewer, marked down.**
Most of what the interviewer scores came from the steps you skipped. You didn't clarify, so they
couldn't see you check assumptions. You didn't state a brute force and its cost, so they couldn't
see where the optimal idea came from, and an unexplained jump to the answer reads the same as
recall. And you didn't test, so the bug was found by them rather than by you. That's the
verification signal, the one that matters more as AI writes more first drafts. The correct
algorithm earned the problem-solving marks; it couldn't earn the others.

---

## File 2 — Arrays and strings

**1. "What if `nums[lo]` pairs with something left of `hi`?"**
Everything still in play lies between `lo` and `hi`, and `nums[hi]` is the largest of them (the
array is sorted). `nums[lo] + nums[hi]` is already too small, so `nums[lo]` plus anything smaller
than `nums[hi]` is smaller still. No remaining element can complete `nums[lo]`. Elements beyond
`hi` were eliminated earlier by the mirror argument, so they're not candidates either.

**2. Why the sliding window is O(n) despite a nested `while`.**
Every iteration of the inner `while` increments `left`. `left` starts at 0, never decreases, and
never exceeds n, so across the *entire* run the inner loop executes at most n times in total, not
n times per outer iteration. The outer loop runs n times. Total: O(n). It's an amortised argument:
count the work by what moves, not by loop nesting.

**3. `"tmmzuxt"`.**

```
right  char  last seen          left   window   best
0      t     no                   0    t          1
1      m     no                   0    tm         2
2      m     at 1, 1 ≥ 0          2    m          2
3      z     no                   2    mz         2
4      u     no                   2    mzu        3
5      x     no                   2    mzux       4
6      t     at 0, 0 ≥ 2? no      2    mzuxt      5
```

Answer 5. Without the `i >= left` check, iteration 6 would set `left = 1`, reporting `"mmzuxt"`
(length 6) despite its repeated `m`.

**4. Sliding window with negatives.**
`nums = [4, −2, 1]`, `k = 3`. At `right = 0` the sum is 4 > 3, so the window shrinks past it and
`left = 1`. The window then grows to `[−2, 1]` (sum −1, length 2), and that's the answer it
reports. But the whole array sums to exactly 3, length 3. What broke is the monotonicity
assumption: the window abandoned index 0 because the sum was too big *at that moment*, but adding
the negative later would have brought it back under the limit. Prefix sums plus a sorted or
hashed structure handle this case.

**5. Interleaved updates.**
An update to `nums[i]` changes `prefix[j]` for every `j > i`: O(n) per update, so 10^5 updates cost
up to 10^10. Prefix sums assume a *static* array. The standard tools for both updates and range
queries in O(log n) are a **Fenwick tree** (binary indexed tree) or a **segment tree**. This course
doesn't cover them; they're worth learning next if you meet this pattern.

---

## File 3 — Hashing

**1. Storing before checking in Two Sum.**
An element can find itself as its own complement whenever `x = target − x`, that is,
`x = target/2`. `nums = [3, 2, 4]`, `target = 6`: at index 0 you store `3 → 0`, then look for
`6 − 3 = 3` and find index 0, returning `(0, 0)`. The right answer is `(1, 2)`.

**2. Why seed `{0: 1}`.**
The sum of `nums[0..j]` is `prefix[j+1] − prefix[0]`, and `prefix[0] = 0` is the empty prefix. It
never appears as a running sum inside the loop unless the elements happen to sum to 0, so without
the seed, a subarray *starting at index 0* with sum `k` is only counted by luck. In the trace,
`[3, 4]` and `[3, 4, −7, 1, 3, 3]` are both lost, and the function returns 2 instead of 4. (`[1, 3, 3]`
survives because the running sum returned to 0 after `−7`, putting a 0 in the map by coincidence.)

**3. Longest subarray summing to k.**
Store, for each prefix sum, the **first** index at which it occurred, with the seed `{0: −1}` (the
empty prefix "ends" before index 0). At index `j`, if `sum − k` is in the map at index `i`, the
subarray `i+1 … j` has sum `k`, length `j − i`. Don't overwrite on later occurrences: the earliest
start gives the longest subarray, and overwriting would replace it with a later, shorter one.

**4. Iterating over `nums` instead of the set.**
Take m copies of the run start followed by the rest of the run: `[1, 1, …, 1, 2, 3, …, m]`. Each
copy of 1 has no predecessor 0, so each one walks the whole run of length m. With n = 2m elements,
that's m × m = n²/4 steps. Iterating the *set* sees 1 only once.

**5. JavaScript to Go, groups in "iteration order".**
JavaScript's `Map` iterates in insertion order, which happened to be first-appearance order.
Go's map iteration order is unspecified and deliberately varies between runs, so sometimes the
order matches and sometimes it doesn't: an intermittent failure. Fix: keep a separate slice of
keys, appending a key the first time you create its group, and build the output by walking that
slice.

**6. `unordered_map<pair<int,int>, …>` won't compile.**
There's no default `std::hash` for `std::pair`. Fixes: (a) use `std::map<pair<int,int>, …>`, which
needs only `operator<` (pairs have it): O(log n) per operation instead of expected O(1); (b) write a
hash functor, for example pack both ints into one 64-bit value and hash that: O(1) expected, but the
hash quality is now your responsibility; (c) encode the pair into a single `long long` key
yourself, which is the same as (b) without the functor. In an interview, (a) is quickest to write
and the log factor rarely matters.

---

## File 4 — Linked lists

**1. Empty list.**
`head` is `nil`, so `cur` starts `nil`, the loop body never runs, and the function returns `prev`,
which is `nil`. The reverse of an empty list is an empty list, with no special case.

**2. No dummy, both pointers start at `head`.**
Removing the head itself breaks it. `[1]`, `n = 1`: the gap loop runs `n + 1 = 2` times; the first
step sets `fast = nil`, and the second dereferences `nil.Next`: a panic in Go, undefined behaviour in
C++. More generally, when the victim is the head there's no node before it for `slow` to stop on.
The dummy provides one.

**3. Why fast can't jump over slow.**
Once both are in the cycle, measure the distance from fast forward to slow along the cycle. Each
step, slow moves 1 and fast moves 2, so the distance shrinks by exactly 1. A distance shrinking by
1 each step goes `…, 2, 1, 0`: it can't skip 0. So they land on the same node within λ steps of
both entering the cycle.

**4. μ = 3, λ = 4.**
Nodes 0, 1, 2 are the tail; 3, 4, 5, 6 form the cycle (6 → 3).

```
t   slow   fast
1    1      2
2    2      4
3    3      6
4    4      4     meet at node 4: a = 1 step into the cycle
```

Check: μ + a = 4 = 1 × λ. Phase 2: `p` from node 0 and `slow` from node 4, one step each: (1, 5),
(2, 6), (3, 3). They meet at node 3, the entry, after μ = 3 steps.

**5. Why nodes store the key.**
On eviction you get the oldest node from the list's back, and you must also delete its entry from
the map. The map is keyed by key. Without the key in the node, you'd have to search the whole map
for the entry pointing at this node: O(n), destroying the O(1) bound.

**6. Map plus a min-heap on last-used time.**
`Put` with eviction is O(log n) (pop the min). `Get` must *update* an existing element's priority,
and a binary heap can't find an arbitrary element cheaply: either O(n) to find it, or push a new
entry and leave the old one stale (lazy deletion), so the heap grows with every access and eviction
has to skip stale entries. At best O(log n) per operation with growing memory, against O(1) for the
linked-list design. The awkward part is precisely "move this existing item to most-recent", which is
a two-pointer splice in a list and has no cheap equivalent in a heap.

---

## File 5 — Stacks and queues

**1. The two emptiness checks.**
Without the final `len(stack) == 0`, `"(("` (or any string with unmatched openers) is accepted.
Without the check inside the loop, a closer with an empty stack, such as `")"`, evaluates
`stack[len(stack)-1]` with length 0: index −1, a panic in Go.

**2. Two-stack queue, O(n) total.**
Follow one element through its life: pushed onto `in` once, moved to `out` at most once, popped from
`out` once. That's at most three O(1) steps per element, however the operations interleave. n
operations involve at most n elements, so at most 3n steps in total: amortised O(1).

**3. The stack never increases.**
Induction on iterations. Before pushing `i`, the `while` loop pops every index whose temperature is
`< t`. When it stops, either the stack is empty or its top has temperature `≥ t`. Pushing `i` on top
of that preserves "never increasing bottom to top", and the rest of the stack was already in order.
Equal temperatures survive because `<` doesn't pop them. Changing `<` to `<=` (one character) pops
equal values too, making the stack *strictly* decreasing, and changes the meaning to "next day at
least as warm".

**4. `[70, 70, 71]`.**
With `<`: at index 1, `70 < 70` is false, so both 70s wait; at index 2, 71 pops both: answer
`[2, 1, 0]`. With `<=`: index 1 pops index 0 (`ans[0] = 1`), giving `[1, 1, 0]`. The first is right
for "strictly warmer"; the second computes "first day at least as warm".

**5. Expiring at most one element per iteration.**
The window moves right by exactly one position per iteration, so exactly one index (`i − k`) newly
falls out. Indices in the deque increase from front to back, so if anything has expired it's the
front. Could the front be *older* than `i − k`? Only if it should have expired on an earlier
iteration, when it was itself `i' − k`; but on that iteration the check examined the front and
removed it (or it had already been evicted from the back). So one check per iteration is enough.

**6. Max-heap with lazy discards.**
O(n log n): each element is pushed once and popped at most once. The heap grows largest on
**increasing** input: each new element is the new maximum, so the top is always in the window and
nothing is ever discarded. All n elements stay in the heap. The monotonic deque would hold just one.

---

## File 6 — Binary search

**1. `firstTrue` terminates and returns `lo`.**
While `lo ≤ hi`, `mid` lies in `[lo, hi]`, and each branch moves `hi` below `mid` or `lo` above it, so
the unknown range `[lo, hi]` loses at least one element per iteration and must empty. It empties
with `lo = hi + 1`. The invariant says everything left of `lo` is false and everything right of `hi`
is true; with `lo = hi + 1` there's nothing in between, so `lo` is the first true position and
`hi` the last false one. Returning `hi` would give you the last false instead.

**2. Last index with `a[i] <= x`.**
Pass `a[i] > x` (upper bound) and subtract one. If no element is `<= x`, the upper bound is 0 and the
result is −1, which is exactly "no such index": no special case needed.

**3. `<` instead of `<=` in the rotated search.**
`nums = [3, 1]`, `target = 1`. `lo = 0`, `hi = 1`, `mid = 0`, `nums[0] = 3 ≠ 1`. With `<`, the test
`nums[lo] < nums[mid]` is `3 < 3`, false, so the code assumes the *right* half `[mid..hi] = [3, 1]` is
sorted, and checks `3 < 1 ≤ 1`: false, so `hi = −1`. Loop ends, returns −1. With `<=`, the
single-element left half is correctly treated as sorted, 1 isn't in `[3, 3)`, so `lo = 1`, and the
next iteration finds it.

**4. Ship capacity starting at 1.**
A capacity below the heaviest package can never ship it. The usual greedy check ("if this package
doesn't fit, start a new day, then add it") adds it to a fresh day anyway, overloading the ship, and
counts that as valid. So small capacities look feasible and the search returns a wrong, too-small
answer. Starting at `max(weights)` keeps every candidate physically possible and the predicate
honest. The upper end is `sum(weights)`: everything in one day.

**5. Koko's monotonicity.**
For every pile, `⌈p/(k+1)⌉ ≤ ⌈p/k⌉`, because eating faster never takes more hours on a pile. Summing
over piles, the total hours at `k + 1` are at most the total at `k`, which is ≤ h. So if `k` is
feasible, `k + 1` is.

**6. Integer square root of n ≤ 10^18.**
You want the largest `x` with `x·x ≤ n`, so search for the first `x` with `x·x > n` and subtract one.
The trap: `x` can be as large as n in the naive range `[0, n]`, and `x·x` for `x` near 10^18 overflows
int64 (max about 9.2·10^18). In Go that wraps silently to a wrong value, which can make the predicate
non-monotone and the search return garbage. Fixes: cap the range at `[0, 2^32)` (since √(10^18) =
10^9 < 2^32), or test `x > n/x` instead of `x·x > n`.

---

## File 7 — Recursion and backtracking

**1. The leap of faith for `maxDepth`.**
Assume `maxDepth` returns the correct depth for any tree smaller than this one, in particular both
children's subtrees. Then this tree's depth is one (for this node) plus the larger of the two. What
you must check separately: the base case (`nil` has depth 0, by definition) and progress (each call
is on a strictly smaller subtree, so recursion reaches `nil`).

**2. Starting the loop at `start`, and what happens at 0.**
Each recursive call only chooses indices larger than the last one chosen, so every path through the
tree lists indices in increasing order, and each subset is produced exactly once, in sorted index
order. `[2, 1]` would need index 0 after index 1. Starting at 0 instead, with everything else the
same, nothing stops the recursion: every call loops over all indices and recurses again, so `path`
grows without bound until the stack overflows.

**3. Why exactly three entries are right.**
`out` stores slice headers pointing at `path`'s backing array, not copies. `[]` was recorded when
`path` was nil. `[1]` was recorded in a backing array of capacity 1; the next `append` (adding 2)
needed capacity 2 and allocated a new array, so the old one, holding `[1]`, was never written again.
Likewise `[1 2]` was left behind when adding 3 forced a capacity-4 array. From then on the capacity
sufficed, every record pointed into that same capacity-4 array, and every later write changed what
they all show. Its final contents were `[3 3 3 _]`, so each later record shows that many 3s.

**4. Forgetting `used[i] = false`.**
One permutation, `[1, 2, 3]`. After it's recorded, every index is marked used and never unmarked,
so every later branch finds nothing available and returns without recording.

**5. `break` in Combination Sum.**
Because `cands` is sorted ascending, every candidate after `i` is at least `cands[i]`, which already
exceeds `remain`, so none of them can fit either. That precondition, sortedness, is what makes
`break` correct. Without the sort, a smaller candidate could come later; `break` would skip it and
lose combinations, so you'd have to use `continue` and lose the pruning.

**6. Generate Parentheses.**
The tree has depth 2n; at each level choose `(` or `)`. Two pruning rules: add `(` only while fewer
than n have been used; add `)` only while fewer `)` than `(` have been used so far. Every string
built under those rules has, at every prefix, no more closers than openers, and at the end exactly n
of each, which is the definition of balanced. Invalid strings are never *built*, so there's nothing
to filter at the leaves. The number of results is the n-th Catalan number.

---

## File 8 — Trees

**1. Depth of 10^5 nodes.**
Balanced: h ≈ log₂ 10^5 ≈ 17. Degenerate: h = 10^5 − 1. JavaScript fails that worst case (about
10^4 frames in my test). Rust can fail it in a spawned thread with the 2 MiB default: a trivial
recursive function overflowed at 10^5 depth in my debug build. C++ depends on the frame size and the
OS stack (commonly 8 MB on the main thread), so it may survive; don't count on it. Go grows
goroutine stacks up to 1 GB and handles it.

**2. Why clamp gains at 0.**
A negative branch lowers any path that includes it, so it's better to stop than to extend into it.
Tree: root `1` with a single left child `−5`. Without the clamp: `gain(−5)` records −5 and returns
−5; at the root, `best = max(−5, 1 + (−5) + 0) = −4`. The right answer is 1, the root alone.

**3. Why `best` starts at `MinInt`.**
A path must contain at least one node. If every value is negative, the best path is the single
largest value, which is negative. Initialised to 0, `best` would report 0, the sum of an empty path
that isn't allowed. Input: a single node `−3`.

**4. The local check, fooled.**
Three nodes: root `2`, its left child `1`, and `1`'s right child `3`. At node 1: no left child,
right child 3 > 1. At node 2: left child 1 < 2. Every local check passes, but 3 sits in 2's *left*
subtree and is larger than 2.

**5. `p` is an ancestor of `q`.**
The recursion reaches `p` before anything below it, and returns `p` at once because `root == p`.
Every other branch returns `nil`, so `p` is passed up unchanged at each ancestor and becomes the
answer. It never confirms that `q` is below `p`, and doesn't need to: the problem guarantees `q` is
in the tree, and if `q` weren't under `p`, some ancestor would receive non-nil from *both* sides and
return itself instead. Since that didn't happen, `q` is in `p`'s subtree, and the LCA of a node and
its descendant is the node.

**6. Repeated k-th smallest on a changing BST.**
Store each node's subtree size. To find the k-th smallest at a node with left-subtree size L: if
k ≤ L, go left; if k = L + 1, this node; otherwise go right looking for the (k − L − 1)-th. That's
O(h) per query. Insertions and deletions update the sizes along the path they walk, also O(h).
This is an **order-statistic tree**.

---

## File 9 — Heaps

**1. Push 0, then pop, on `[1, 3, 2, 7, 4, 5]`.**
Push: append at index 6 → `[1,3,2,7,4,5,0]`. Parent of 6 is 2 (value 2); 0 < 2, swap →
`[1,3,0,7,4,5,2]`. Parent of 2 is 0 (value 1); 0 < 1, swap → `[0,3,1,7,4,5,2]`. Stop at the root.
Pop: remove 0, move the last element (2) to the root → `[2,3,1,7,4,5]`. Children of 0 are 3 and 1;
the smaller is 1 at index 2; 2 > 1, swap → `[1,3,2,7,4,5]`. Index 2's only child is 5 at index 5;
2 < 5, stop. Result `[1,3,2,7,4,5]`.

**2. Heapify in O(n).**
Sift-down from a node costs at most its height. Half the nodes are leaves (height 0, no work), a
quarter have height 1, an eighth height 2, and so on. Most nodes are near the bottom where sifting is
cheap; only a handful are near the top where it's expensive. The total is n × (0/2 + 1/4 + 2/8 +
3/16 + …), and that series converges to a constant, so heapify is O(n).

**3. k smallest of a stream.**
A **max**-heap of size k. For each new element ask: *"is this smaller than the largest on my
shortlist?"* Push it, and if the heap now has k + 1 elements, pop the max. At the end the heap holds
the k smallest, with the k-th smallest on top.

**4. `h.Push(x)` instead of `heap.Push(h, x)`.**
Your methods only append and remove the last element; the sifting lives in the `heap` package
functions. Calling them directly leaves an ordinary slice. In `findKthLargest`, each push appends and,
once the size exceeds k, "pop" removes the element just appended, so the slice keeps the first k
elements of the input unchanged, and `(*h)[0]` returns `nums[0]`. It's right only on inputs where
the first element happens to be the k-th largest.

**5. Why `lo` ≤ `hi` always holds.**
Suppose it held before. Push x into `lo`, then move `lo`'s maximum m to `hi`. m is the largest of
`lo ∪ {x}`, so everything left in `lo` is ≤ m. Everything left in `lo` was either in the old `lo` (≤
every old `hi` value) or is x but not the maximum (so ≤ m). Hence every `lo` value ≤ every `hi`
value, including m. The rebalance step moves `hi`'s minimum down, which is ≥ everything in `lo` and
≤ everything left in `hi`, so it preserves the property too.

**6. k up to n/2, tight memory.**
The size-k heap is O(n log k) time and O(k) = O(n) memory when k = n/2, so it no longer saves
memory. Quickselect is O(n) average time and works in place with O(1) extra memory, but needs the
whole array in memory and modifies it; with a random pivot, the O(n²) worst case is very unlikely. If
the input is a stream you can't store, an exact median in one pass with small memory isn't
possible in general; in practice you'd use an approximate quantile sketch and say that the answer is
approximate. That's an honest limit, not a gap in your preparation.

---

## File 10 — Graphs

**1. Why BFS distances are final when first reached.**
The queue always holds vertices in non-decreasing order of distance, and at most two distinct
distances at once (d and d + 1), because a vertex at distance d only enqueues vertices at d + 1. So
vertices are dequeued in order of distance. If v is first reached from u, then `dist(u) + 1` is the
smallest distance at which any neighbour of v gets dequeued; any shorter path to v would go through
a neighbour dequeued earlier, which would have reached v first. The argument rests on the FIFO
property.

**2. Marking on pop in `numIslands`.**
The count stays correct: the outer loop only counts a cell that's still `'1'`, and the BFS still
eventually marks the whole island. What gets worse is the queue: a cell can be pushed once by each
land neighbour before it's popped, so up to four entries per cell, with each duplicate re-examining
its neighbours. Worst on large solid blocks of land, where almost every cell has four land
neighbours. On a grid it's a constant factor; on a dense general graph, one entry per edge.

**3. Why `len(order) < n` means a cycle.**
Every vertex on a cycle has an incoming edge from another vertex on the cycle, and none of them can
be output first, so none ever reaches in-degree 0. Conversely, in an acyclic graph there's always a
vertex with in-degree 0 among those remaining, so Kahn's algorithm outputs everything. And yes:
vertices *downstream* of a cycle (reachable from it, not on it) are also missing, because they wait
on cycle vertices that never finish. So "missing" means "on or after a cycle".

**4. Negative edge: textbook versus lazy.**
Graph `A→B 2, A→C 3, C→B −2`. Textbook: pop A (0), relax B = 2, C = 3. Pop B (2): settled, final.
Pop C (3): the edge to B would give 1, but B is settled and isn't touched. Result B = 2; the truth is
1. The lazy version has no settled set: popping C finds `3 − 2 = 1 < 2`, updates B and pushes it
again, then re-expands B. It returns 1. It's right here because it keeps correcting, and that's
exactly why it's not Dijkstra: vertices can be re-expanded many times (the time bound is gone), and
with a negative cycle it never terminates. Use Bellman–Ford.

**5. Heap size in Dijkstra.**
Each successful relaxation pushes one entry, and each edge can succeed at most once per time its
source is expanded, which in correct (non-negative) Dijkstra is once. So at most E + 1 entries. Each
push and pop costs O(log E), and since E ≤ V², log E ≤ 2 log V = O(log V). With E + 1 pushes and
pops, the total is O((V + E) log V).

**6. Union by rank, no compression.**
Height is at most log₂ n. Rank only increases when two trees of *equal* rank merge, and by
induction a tree of rank r has at least 2^r nodes (two trees of at least 2^(r−1) each). So rank, and
height, can't exceed log₂ n, and `Find` is O(log n).

---

## File 11 — Greedy and intervals

**1. `{1, 10, 25}`.**
30: greedy takes 25 then five 1s (six coins); three 10s use three. Below 25 greedy uses only 10s and
1s, which is optimal for those; from 25 to 29, the alternatives with 10s are worse. So 30 is the
smallest failure. The lesson: twenty-nine examples passing told you nothing about the thirtieth.
Testing finds counterexamples; it can't prove there are none.

**2. Overwriting the end in `merge`.**
`[[1, 10], [2, 3]]`: the second interval overlaps, and `out[n-1][1] = 3` shrinks the run to
`[1, 3]`. The right answer is `[[1, 10]]`. The interval was entirely inside the run, and only `max`
keeps the run's end.

**3. The exchange argument, and why shortest-first fails.**
Take any optimal selection, sorted by end. Greedy's first pick ends no later than the optimal
selection's first interval (greedy chose the earliest end overall), so swapping it in can't create a
clash with anything later in the optimal selection. Same size, still valid. Repeat on the rest:
greedy's picks replace the optimum's one by one, so greedy keeps as many.

Shortest-first on `[1,5]`, `[4,7]`, `[6,10]`: `[4,7]` is shortest (length 3) and is chosen first,
and it overlaps both others, so only one interval is kept. Keeping `[1,5]` and `[6,10]` gives two.
The swap step fails because the shortest interval can end *later* than the optimum's first
interval, and so can clash with its second.

**4. Sorting starts and ends independently.**
The number of rooms in use at time t is (starts ≤ t) − (ends ≤ t), for half-open intervals. That
formula only uses counts, not which end belongs to which start. Since every meeting's end comes
after its own start, the count never goes negative, and the maximum of the running count is the
answer.

**5. `[1,4]` and `[4,5]` in the three functions.**
`merge` uses `<=`: they overlap and merge into `[1,5]`. `eraseOverlapIntervals` keeps an interval if
`start >= end`: they don't overlap, so both are kept. `minMeetingRooms` releases rooms with
`ends[e] <= s`: they don't overlap, one room. That's inconsistent, deliberately: each matches its
problem. LeetCode 56 merges touching intervals; LeetCode 435 says intervals touching at a point are
non-overlapping; meetings are naturally half-open. It matters because the definition is part of the
problem, and the right move is to ask.

**6. Gas Station.**
Let `S_i` be the running sum of `gas − cost` up to station i, and let m be the station where it's
lowest. Start at m + 1. For any later station i, the tank holds `S_i − S_m ≥ 0`, since `S_m` is the
minimum. After wrapping round to some i ≤ m, the tank holds `(S_total − S_m) + S_i`; since `S_i ≥ S_m`
and `S_total ≥ 0`, that's ≥ 0 too. The tank is never negative, so m + 1 works.

---

## File 12 — Dynamic programming

**1. Merge sort.**
It lacks overlapping subproblems. Its two halves are disjoint, and no subarray is ever sorted twice.
Memoising would store answers that are never looked up again: extra memory, no saving. That's divide
and conquer, not DP.

**2. Unique paths with obstacles.**
State: `dp[r][c]` = the number of paths from the top-left to cell `(r, c)`. Transition:
`dp[r][c] = dp[r−1][c] + dp[r][c−1]`, treating out-of-grid cells as 0. Base: `dp[0][0] = 1` if it's
not an obstacle (otherwise 0, and the answer is 0). An obstacle cell has `dp = 0`: no path ends
there, so it contributes nothing to its neighbours.

**3. `rob` with sequential assignment.**
After `take = skip + x`, the next line computes `skip = max(take, skip)` with the *new* `take`, which
includes the current house. So "skip" can now mean "took this one", and the following house adds to
it, robbing adjacent houses.

```
x    take          skip
2    0 + 2 = 2     max(2, 0) = 2
7    2 + 7 = 9     max(9, 2) = 9
9    9 + 9 = 18    18
3    21            21
1    22            22
```

It returns 22, the sum of every house. (With non-negative values, the corrupted recurrence always
does this.)

**4. Upward loop in `canPartition`.**
`[1, 3]`: total 4, target 2. Upwards with item 1: `can[1] = can[0] = true`, then
`can[2] = can[1] = true`, using the 1 twice. It returns `true`, but `{1}` and `{3}` don't balance.
Nothing smaller fails: `[1, 1]` really is `true`, and single elements are caught by the parity
check or never enter the loop.

**5. Word Break complexity.**
There are O(n²) pairs `(j, i)`, and for each one building `s[j:i]` and hashing it costs O(i − j),
up to O(n). Worst case O(n³). If every dictionary word has length at most L, only try
`j ≥ i − L`: O(n · L) pairs at O(L) each, O(n · L²). Alternatively walk a trie of the dictionary
forwards from each `j` ([file 13](13-tries-and-bits.md)), stopping as soon as no word continues.

**6. Weighted interval scheduling.**
Greedy by end fails because profits differ: `[1, 10]` worth 100 against `[1, 2]` and `[3, 4]` worth 1
each. Greedy keeps the two small jobs (2), and the answer is 100. Sort jobs by end time. State:
`dp[i]` = the best profit using only the first i jobs. Transition: skip job i (`dp[i−1]`) or take it
(`profit_i + dp[p(i)]`, where `p(i)` is the number of jobs ending no later than job i starts). Binary
search over the sorted end times finds `p(i)` in O(log n), so the whole thing is O(n log n).

---

## File 13 — Tries and bits

**1. Why `isWord` is needed.**
Insert `app` and `apple`. The node for `app` has a child (`l`), so "no children" can't mark it as a
word, yet it is one. Insert only `apple`, and the same node exists with the same shape, but `app`
isn't a word. Same shape, different answers: only a flag can tell them apart.

**2. 10^5 random 10-letter words.**
I built it: **707,554 nodes**, each a 216-byte struct in Go (26 eight-byte pointers plus a bool,
padded), and the heap grew by about **159 MB**. The upper levels are shared (26 nodes, then 676, then
17,576); from about the fourth letter almost every word has its own path, so you pay roughly 6–7 new
nodes per word. A hash set of the same words stores about 10^5 × (10 bytes of text plus a 16-byte
string header plus table overhead), a few MB by estimate. (My attempt to measure the set gave a
number too low to trust, so treat "a few MB" as an estimate.) Random words are close to a worst case
for a trie, since they share so little; real vocabularies share far more prefixes.

**3. `12 & 11`.**

```
12 = 1100
11 = 1011
&  = 1000 = 8      the lowest set bit of 12 (value 4) is gone
```

Subtracting 1 turns the lowest set bit into 0 and every 0 below it into 1, and leaves everything above
it unchanged. ANDing with the original keeps the unchanged higher bits and zeros the region that
flipped, so exactly the lowest set bit disappears.

**4. Every element thrice except one.**
XOR cancels *pairs*: three copies give `a ^ a ^ a = a`, so the triples don't cancel and the result is
a mixture. Instead, for each of the 32 bit positions, count how many numbers have that bit set. The
triples contribute a multiple of 3, so `count mod 3` is the single number's bit. That's O(32n) time and
O(1) space. (In a fixed-width language, rebuild the sign from bit 31.)

**5. JavaScript masks for 40 items.**
JavaScript's bitwise operators work on 32-bit integers, and the shift count is taken mod 32, so
`1 << 32` is `1` and `1 << 33` is `2` (I checked): items 32–39 collide with items 0–7, and `1 << 31` is
negative. Small tests use few items and pass. Fixes: `BigInt` masks (`1n << BigInt(i)`), or split the
mask into two 32-bit numbers, or drop the bitmask and use an array of booleans or a `Set`.

---

## Things to try

1. **Feel the constraint table.** Write an O(n²) and an O(n log n) solution to the same problem in
   Go (say, counting pairs with a given sum), time both at n = 10^3, 10^4, 10^5, and find where the
   quadratic one crosses one second on your machine. Compare with the 10^8 rule of thumb.
2. **Break a hash map on purpose.** In C++, `reserve` an `unordered_map<long long, int>`, read its
   `bucket_count()`, and insert 10^5 keys that are all multiples of it. Time that against 10^5
   sequential keys, then repeat both with a custom hash that mixes the bits.
3. **Reproduce the aliasing bug.** Remove `slices.Clone` from `subsets` and predict the output before
   running it, for `[1, 2, 3]` and then `[1, 2, 3, 4]`. Then preallocate `path` with
   `make([]int, 0, 10)` and predict again.
4. **Watch `shift()` go quadratic.** Drain arrays of 10^4, 10^5 and 2·10^5 elements with `shift()` in
   Node, then with a head index. Plot the times.
5. **Port one chapter to all four languages without looking.** Take the monotonic deque from file 05,
   which is only written in Go here, and write it in Rust, C++ and JavaScript. Note which language
   made you think hardest, and about what.
6. **Prove a greedy, then break it.** For interval scheduling, write the exchange argument on paper;
   then change the problem to weighted intervals and find the smallest counterexample yourself before
   reading question 6 under File 12 above.

## Questions worth asking me

- "Give me a problem I haven't seen and make me do the six steps out loud; stop me when I skip one."
- "Which pattern is this, and how would I have recognised it?" (with a problem pasted in)
- "Here's my solution — what's its real complexity, and where would it break in Go/Rust/C++/JS?"
- "Explain Fenwick trees and segment trees as the next chapter after prefix sums."
- "Why exactly does Dijkstra fail with negative edges? Show me the smallest graph again, slower."
- "Walk me through designing the DP state for a problem from scratch, one question at a time."
- "What do interviewers at Indian product companies actually ask at my level, and how sure is that?"
- "Turn the language-traps table into flashcards I can drill."
- "I keep getting off-by-ones in binary search. Diagnose my code against the `firstTrue` invariant."
