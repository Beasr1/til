# 14. Glossary

Terms in the order you meet them, not alphabetically. Search with your editor. The file
reference after each group says where the term is taught properly.

## Analysis — [file 01](01-foundations.md)

**Big-O, O(g(n))** — an upper bound on growth: the cost grows no faster than g(n), up to a
constant, for large enough n. In interviews people say O when they mean Θ.

**Ω(g(n))** — a lower bound on growth. "Comparison sorting is Ω(n log n)" is a statement about
*every* possible comparison sort.

**Θ(g(n))** — a tight bound: both O and Ω.

**Worst / average / expected case** — which inputs a bound is about. Quicksort is O(n²) worst
case and O(n log n) on average; hash lookup is O(1) *expected*. Say which one you mean.

**Amortised cost** — total cost of any sequence of n operations, divided by n. A guarantee about
sequences, not an average over random inputs. Dynamic-array append is amortised O(1) and O(n) for
the occasional single call.

**Auxiliary space** — memory used beyond the input. What "O(1) space" usually means.

**Call stack / stack frame** — memory used by function calls still in progress. Recursion depth
counts as space, and its limit differs by language.

**Tail-call elimination** — reusing the current frame for a call in tail position. None of Go,
Rust, C++ or JavaScript guarantees it, so don't rely on it.

**Constraint-to-complexity table** — the rule of thumb (≈10^8 simple operations per second) that
turns "n ≤ 10^5" into "O(n log n) or better". Order of magnitude only.

## Arrays and strings — [file 02](02-arrays-and-strings.md)

**Two pointers** — two indices whose movement is justified by an elimination argument: each step
proves a candidate useless.

**Sliding window** — a contiguous range `[left, right]` that grows on the right and shrinks on the
left, each end moving only forward. Needs a monotone validity condition.

**Prefix sum** — `prefix[i]` = sum of the first i elements, with `prefix[0] = 0`; any range sum is
one subtraction.

**Half-open interval `[l, r)`** — includes l, excludes r. Makes lengths `r − l` and removes most
off-by-ones.

## Hashing — [file 03](03-hashing.md)

**Hash function** — maps a key to a bucket index. Must be deterministic; should spread keys
evenly.

**Collision** — two keys in the same bucket. Resolved by chaining or probing; too many make
lookups slow.

**Load factor** — keys ÷ buckets. When it passes a threshold the table resizes, which is why
insert is amortised.

**Hash flooding (HashDoS)** — an attack that picks keys designed to collide. Mitigated by
randomised hash seeds (Go, Rust).

**Complement lookup** — at element x, look up the partner you'd need (`target − x`) among what
you've already seen.

**Canonical form** — a key that's equal for everything in an equivalence class (sorted letters for
anagrams).

## Linked lists — [file 04](04-linked-lists.md)

**Singly / doubly linked list** — nodes with a `next` pointer, or `next` and `prev`.

**Dummy (sentinel) node** — a throwaway node before the head so the head needs no special case.

**Fast and slow pointers** — two pointers at different speeds or with a fixed gap; finds the
middle, the n-th from the end, or a cycle.

**Floyd's cycle detection (tortoise and hare)** — fast/slow pointers meet inside a cycle; restarting
one from the head finds the entry. Attributed to Floyd by Knuth, possibly folklore.

**LRU cache** — evicts the least recently used item; hash map into a doubly linked list gives O(1)
operations.

## Stacks and queues — [file 05](05-stacks-and-queues.md)

**Stack (LIFO)** / **Queue (FIFO)** / **Deque** — last-in-first-out; first-in-first-out; both ends.

**Monotonic stack** — a stack of unresolved elements whose values stay sorted because anything that
would break the order has just been answered and popped. Solves "next greater/smaller".

**Monotonic deque** — the same, with expiry from the front; gives sliding-window max/min in O(n).

## Binary search — [file 06](06-binary-search.md)

**Monotone predicate** — a yes/no question whose answers go `false … false true … true` over a
range. The only thing binary search needs.

**Lower bound / upper bound** — first index with `a[i] >= x`; first index with `a[i] > x`.

**Binary search on the answer** — binary search over the range of possible answers, using a
feasibility check as the predicate.

## Recursion and backtracking — [file 07](07-recursion-and-backtracking.md)

**Base case / progress** — the two conditions that make recursion terminate and correct, mirroring
proof by induction.

**Recursive leap of faith** — assume the function is correct on smaller inputs; check the
combination and the base case.

**Recursion tree** — the calls drawn as a tree; time ≈ nodes × work per node, space ≈ height.

**Backtracking** — depth-first search over a decision tree with choose / explore / un-choose on a
shared state.

**Pruning** — skipping a branch you can prove contains no answer.

## Trees — [file 08](08-trees.md)

**Root / leaf / depth / height** — the top node; a node without children; edges from the root;
the largest depth.

**Balanced / degenerate** — height O(log n) / height n − 1 (a list in disguise).

**Pre-order / in-order / post-order** — node before / between / after its children.

**BFS by levels** — process the queue one whole level per round.

**Binary search tree (BST)** — every value in a node's left subtree is smaller and every value in
its right subtree larger. "Every", not just the children.

**Lowest common ancestor (LCA)** — the deepest node with both targets in its subtree.

**Self-balancing tree** — a BST that restructures to keep height O(log n): red-black trees, AVL
trees; B-trees for disk and cache-friendly layouts.

## Heaps — [file 09](09-heaps.md)

**Binary heap** — a complete binary tree in an array where each parent is ≤ its children
(min-heap) or ≥ (max-heap). Not sorted.

**Sift up / sift down** — the O(log n) repairs after push and pop.

**Heapify** — building a heap from n items in O(n).

**Priority queue** — the abstract "give me the most urgent item" type; a heap is the usual
implementation.

**Top-k** — keep a size-k heap ordered so the *worst* of the shortlist is on top.

**Lazy deletion** — leave outdated heap entries in place and skip them when popped.

**Quickselect** — partition-based selection of the k-th element; O(n) average, O(n²) worst.

## Graphs — [file 10](10-graphs.md)

**Vertex, edge, directed, undirected, weighted** — the parts of a graph and their kinds. V and E
are their counts.

**Adjacency list / matrix** — neighbours per vertex (O(V + E) memory) / a V×V table (O(V²)).

**Implicit graph** — vertices and edges generated on demand (grid cells, puzzle states).

**BFS / DFS** — level-by-level with a queue / as deep as possible with a stack or recursion. Both
O(V + E).

**Multi-source BFS** — BFS started from several vertices at distance 0.

**In-degree** — the number of edges into a vertex.

**Topological sort (Kahn's algorithm)** — repeatedly output a vertex of in-degree 0; fewer than V
outputs means a cycle.

**DAG** — directed acyclic graph: exactly the graphs that have a topological order.

**Dijkstra's algorithm** — shortest paths with non-negative weights by always expanding the closest
unsettled vertex. O((V + E) log V) with a binary heap.

**Relaxation** — improving `dist[v]` via an edge `u → v` when `dist[u] + w < dist[v]`.

**Bellman–Ford** — shortest paths allowing negative weights, O(VE); detects negative cycles.

**Union-find (disjoint-set union, DSU)** — a forest of sets supporting `Find` and `Union`.

**Union by rank / path compression / path halving** — the two tricks that make union-find
amortised O(α(n)); halving is a one-pass form of compression.

**α(n), inverse Ackermann** — a function that is at most 4 for any practical n.

## Greedy and intervals — [file 11](11-greedy-and-intervals.md)

**Greedy algorithm** — commits to the locally best choice at each step and never revisits it.

**Exchange argument** — proof that greedy is optimal by swapping its choice into any optimal
solution without making it worse.

**Interval scheduling** — keep the most non-overlapping intervals; sort by end.

**Sweep line** — process start and end events in time order, tracking a running count.

**Difference array** — `diff[start] += k`, `diff[end] −= k`, then a prefix sum; range updates in O(1)
each.

## Dynamic programming — [file 12](12-dynamic-programming.md)

**Optimal substructure** — the answer is built from answers to smaller instances of the same
problem.

**Overlapping subproblems** — those smaller instances repeat. Without this, it's divide and
conquer.

**State** — the smallest description of "where I am" that's enough to finish optimally without
knowing how I got here.

**Transition (recurrence)** — how a state's answer is computed from other states, one per possible
decision.

**Memoisation (top-down)** — recursion plus a cache.

**Tabulation (bottom-up)** — filling a table in dependency order with loops.

**0/1 vs unbounded knapsack** — each item at most once vs any number of times; in a one-row table,
loop capacity downwards vs upwards.

**Pseudo-polynomial** — polynomial in the numeric *value* of an input rather than its size.

**Kadane's algorithm** — maximum subarray sum via "best sum ending here".

**Patience sorting** — the O(n log n) LIS method keeping the smallest tail for each length.

## Tries and bits — [file 13](13-tries-and-bits.md)

**Trie (prefix tree)** — a tree whose root-to-node paths spell prefixes; O(L) insert and prefix
queries.

**Bitmask** — an integer used as a set of small numbers; bit i set means i is in the set.

**Popcount** — the number of set bits.

**`x & (x − 1)`** — clears the lowest set bit.

**`x & −x`** — isolates the lowest set bit.

**Two's complement** — the signed-integer representation where `−x = ~x + 1`; why the tricks
above work, and why negating the minimum value overflows.
