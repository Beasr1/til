# 9. Heaps and top-k

A stream of numbers is arriving and after each one you need the current smallest. Or the 10
largest. Or the median. Re-sorting after every arrival costs O(n log n) each time. A sorted
array makes the lookup O(1) but every insert O(n). A **heap** sits in between: it doesn't keep
everything sorted, only enough order to get at the extreme element instantly, and that partial
order costs O(log n) to maintain per insert or removal.

## 9.1 What a heap is

### The intuition

Think of a hospital emergency department. Nobody cares about the exact order of all the
patients waiting, only about *who is next*. Triage keeps the most urgent case at the front, and
when someone new arrives, they're slotted roughly into place, not into an exact ranking. That
"I only need the top" relaxation is what makes a heap cheap.

### The structure

A **binary heap** is a complete binary tree (every level full except possibly the last, which
fills from the left) with the **heap property**: every parent is ≤ its children, for a
**min-heap**. (A **max-heap** flips it.) The smallest element is therefore at the root.
Siblings are in no particular order, so a heap is *not* sorted.

Because the tree is complete, it fits in an array with no pointers:

```
         1                    index:  0  1  2  3  4  5
       /   \                  value:  1  3  2  7  4  5
      3     2
     / \   /                  children of i:  2i+1, 2i+2
    7   4 5                   parent of i:    (i−1)/2  (integer division)
```

Two repair operations keep the property true:

- **Sift up** (after a push): append the new element at the end, then swap it with its parent
  while it's smaller than the parent. At most one swap per level: O(log n).
- **Sift down** (after a pop): move the last element into the root, then swap it with its
  smaller child while it's bigger than that child. Also O(log n).

| Operation | Cost |
|---|---|
| peek at min | O(1) |
| push | O(log n) |
| pop min | O(log n) |
| build a heap from n items at once (**heapify**) | **O(n)** |
| find or delete an arbitrary element | O(n) to find it; not what heaps are for |

Heapify being O(n) rather than O(n log n) surprises people. Sifting down from the bottom
upwards, half the nodes are leaves (no work), a quarter sit one level up (at most one swap), and
so on, and that sum converges to O(n). CLRS chapter 6 (`BUILD-MAX-HEAP`) has the proof, and Go's
`container/heap` documents `Init` as O(n).

## 9.2 Heaps in four languages

This is the container whose defaults differ most between languages, so learn your own
language's version cold.

| | Go | Rust | C++ | JavaScript |
|---|---|---|---|---|
| Type | `container/heap` functions over **your** type implementing `heap.Interface` | `std::collections::BinaryHeap<T>` | `std::priority_queue<T>` | **none built in** |
| Default order | whatever your `Less` says | **max-heap** | **max-heap** | — |
| Get a min-heap | `Less` returns `h[i] < h[j]` | wrap items in `std::cmp::Reverse` | `priority_queue<T, vector<T>, greater<T>>` | write one |
| Custom order | in `Less` | implement `Ord`, or push tuples whose natural order is what you want | a comparator type | your comparator |

Sources: [`container/heap`](https://pkg.go.dev/container/heap),
[`BinaryHeap`](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html),
[`std::priority_queue`](https://en.cppreference.com/w/cpp/container/priority_queue).

> ⚠️ **Rust and C++ hand you a max-heap.** Port a Go solution that relied on `<` meaning "min" and
> you'll pop the largest element instead. The code runs, and the answer is wrong only on inputs
> where the order matters. In C++ the comparator is also backwards from what most people expect:
> `std::less` gives a *max*-heap, because the comparator says which element is lower *priority*.

Go's interface needs five methods. It looks heavy the first time; it's the same every time:

```go
type MinHeap []int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] } // < gives a min-heap
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
	old := *h
	x := old[len(old)-1]
	*h = old[:len(old)-1]
	return x
}
```

You call `heap.Push(h, x)` and `heap.Pop(h)`, **not** `h.Push` and `h.Pop`. The package
functions do the sifting and call your methods to do the slice work. Call your methods directly
and you append to the slice without restoring the heap property, with no error. `(*h)[0]` is the
minimum.

## 9.3 Top-k with a size-k heap

### The problem

Find the k-th largest element of an array (or the k largest, or the k most frequent).

### The intuition

> **Teacher's aside.** To find the k *largest*, use a **min**-heap. That sounds backwards until
> you think about what the heap is for. You're keeping a shortlist of the k best seen so far, and
> the question you ask about every new arrival is: *"is this better than the worst thing on my
> shortlist?"* The worst of the k largest is the smallest of them, so that's the one you need
> instant access to: the min. If a new element beats it, the minimum gets evicted. When the
> stream ends, the shortlist is the k largest, and its minimum, at the root, is the k-th largest.
> The same logic says: k smallest → max-heap. Remember the question, not the rule.

```go
func findKthLargest(nums []int, k int) int {
	h := &MinHeap{}
	for _, x := range nums {
		heap.Push(h, x)
		if h.Len() > k {
			heap.Pop(h) // evict the smallest; the k largest so far survive
		}
	}
	return (*h)[0] // smallest of the k largest = the k-th largest
}
```

<details><summary>Rust</summary>

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

fn find_kth_largest(nums: &[i32], k: usize) -> i32 {
    let mut heap = BinaryHeap::new(); // max-heap by default...
    for &x in nums {
        heap.push(Reverse(x)); // ...so wrap in Reverse to make it a min-heap
        if heap.len() > k {
            heap.pop();
        }
    }
    heap.peek().unwrap().0
}
```

</details>

<details><summary>C++</summary>

```cpp
int findKthLargest(const vector<int>& nums, int k) {
    // priority_queue is a max-heap by default; greater<int> makes it a min-heap
    priority_queue<int, vector<int>, greater<int>> heap;
    for (int x : nums) {
        heap.push(x);
        if ((int)heap.size() > k) heap.pop();
    }
    return heap.top();
}
```

</details>

<details><summary>JavaScript</summary>

JavaScript has no heap, so this block includes one. It's worth being able to write from
memory, since online assessments rarely provide a library. (LeetCode's JS environment is reported
to include `@datastructures-js/priority-queue`; I couldn't confirm that from LeetCode's own page,
and other platforms won't have it.)

```js
class MinHeap {
  constructor(less = (a, b) => a < b) {
    this.a = [];
    this.less = less;
  }
  get size() { return this.a.length; }
  peek() { return this.a[0]; }
  push(x) {
    const a = this.a;
    a.push(x);
    let i = a.length - 1;
    while (i > 0) { // sift up
      const p = (i - 1) >> 1;
      if (!this.less(a[i], a[p])) break;
      [a[i], a[p]] = [a[p], a[i]];
      i = p;
    }
  }
  pop() {
    const a = this.a, top = a[0], last = a.pop();
    if (a.length) {
      a[0] = last;
      let i = 0;
      for (;;) { // sift down
        const l = 2 * i + 1, r = l + 1;
        let m = i;
        if (l < a.length && this.less(a[l], a[m])) m = l;
        if (r < a.length && this.less(a[r], a[m])) m = r;
        if (m === i) break;
        [a[i], a[m]] = [a[m], a[i]];
        i = m;
      }
    }
    return top;
  }
}

function findKthLargest(nums, k) {
  const heap = new MinHeap();
  for (const x of nums) {
    heap.push(x);
    if (heap.size > k) heap.pop();
  }
  return heap.peek();
}
```

The `(i − 1) >> 1` is safe here because heap indices never approach 2^31. Pass a different
`less` for other orders: `(a, b) => a > b` for a max-heap, `(a, b) => a[0] < b[0]` for
`[priority, item]` pairs.

</details>

### Worked example

`nums = [3, 2, 1, 5, 6, 4]`, `k = 2`. The heap contents are shown sorted for readability; the
array inside is only heap-ordered.

```
x    after push      size > 2?   pop        heap (the 2 largest so far)
3    {3}             no                     {3}
2    {2, 3}          no                     {2, 3}
1    {1, 2, 3}       yes         1          {2, 3}
5    {2, 3, 5}       yes         2          {3, 5}
6    {3, 5, 6}       yes         3          {5, 6}
4    {4, 5, 6}       yes         4          {5, 6}
answer: root = 5
```

**Complexity:** n pushes and up to n pops on a heap of size ≤ k + 1: **O(n log k)** time, O(k)
space. When k is small next to n, that beats sorting.

### The alternatives, and when to mention them

| Approach | Time | Space | Note |
|---|---|---|---|
| Sort, take index n − k | O(n log n) | O(1)–O(n) | the honest baseline; say it first |
| Size-k min-heap | O(n log k) | O(k) | works on a **stream**; the usual expected answer |
| Heapify all n, pop k times | O(n + k log n) | O(n) | good when k is close to n |
| Quickselect | O(n) average, **O(n²) worst** | O(1) | Hoare's 1961 *Find*; randomise the pivot; doesn't work on a stream |

For "top k frequent", count with a hash map first ([file 03](03-hashing.md)), then run the
size-k heap over `(count, value)` pairs.

## 9.4 Two heaps: the running median

### The problem

Numbers arrive one at a time; after each, report the median.

### The intuition

Split everything seen so far into a **lower half** and an **upper half**. The median sits at the
boundary: the largest of the lower half and the smallest of the upper half. So keep the lower
half in a **max-heap** and the upper half in a **min-heap**. Keep their sizes equal, or let the
lower one hold one extra.

To add a number without thinking hard about which half it belongs in: push it into the lower
half, move the lower half's max up to the upper half (which guarantees every lower value ≤ every
upper value), then if the upper half has become bigger, move its min back down.

```go
type MedianFinder struct {
	lo *MinHeap // smaller half, stored negated so the min-heap acts as a max-heap
	hi *MinHeap // larger half
}

func NewMedianFinder() MedianFinder { return MedianFinder{&MinHeap{}, &MinHeap{}} }

func (m *MedianFinder) AddNum(x int) {
	heap.Push(m.lo, -x)
	heap.Push(m.hi, -heap.Pop(m.lo).(int)) // largest of the small half moves up
	if m.hi.Len() > m.lo.Len() {
		heap.Push(m.lo, -heap.Pop(m.hi).(int)) // keep lo the same size or one bigger
	}
}

func (m *MedianFinder) FindMedian() float64 {
	if m.lo.Len() > m.hi.Len() {
		return float64(-(*m.lo)[0])
	}
	return float64(-(*m.lo)[0]+(*m.hi)[0]) / 2
}
```

Storing the lower half **negated** in a min-heap is the standard way to get a max-heap without
writing a second type. In Rust it's the reverse trick: `BinaryHeap` is already a max-heap, so wrap
the *upper* half in `Reverse`.

Adding 1, 2, 3:

```
add   push −x to lo   move lo's max to hi   rebalance?           lo (as values)   hi     median
1     lo {1}          lo {}, hi {1}         hi bigger → move 1   {1}              {}     1
2     lo {1, 2}       lo {1}, hi {2}        equal, no            {1}              {2}    1.5
3     lo {1, 3}       lo {1}, hi {2, 3}     hi bigger → move 2   {1, 2}           {3}    2
```

**Complexity:** O(log n) per `AddNum`, O(1) per `FindMedian`.

> ⚠️ **Negating overflows at the extremes.** `−x` for the most negative integer is itself in
> two's complement, so `-math.MinInt` is `math.MinInt` in Go and undefined behaviour for `int` in
> C++. Fine for LeetCode's ranges (|x| ≤ 10^5 in problem 295); worth a sentence if the interviewer
> asks about the full integer range.

## 9.5 Merging k sorted sequences

Merging k sorted lists of N total elements: put the head of each list into a min-heap of size k,
pop the smallest, append it to the output, push that list's next element. Every element goes in
and out once, at O(log k) each: **O(N log k)**. Merging pairwise in sequence would cost O(Nk).
It's the same shape as the external merge sort that databases use when a sort doesn't fit in
memory.

The other heap technique worth knowing, **lazy deletion**, appears in Dijkstra
([file 10 §10.5](10-graphs.md#105-weighted-shortest-paths-dijkstra)): rather than finding and
removing an outdated entry (O(n) in a heap), leave it in and skip it when it's popped.

## 9.6 How to recognise a heap

| You see… | Think… |
|---|---|
| "k-th largest/smallest", "top k", "k closest" | size-k heap, ordered so the *worst* of the shortlist is on top |
| A stream, with "current min/max/median" after each item | heap; two heaps for the median |
| Merge k sorted things | min-heap of the k current heads |
| "Always process the cheapest/earliest/most urgent next" | heap as a priority queue (scheduling, Dijkstra) |
| Need min **and** to delete arbitrary items | heap with lazy deletion, or an ordered map |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [1046. Last Stone Weight](https://leetcode.com/problems/last-stone-weight/) | Basic max-heap use in your language; the warm-up |
| [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Size-k min-heap; discussing quickselect |
| [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Hash map counts, then a heap (or bucket sort for O(n)) |
| [973. K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | Size-k **max**-heap on distance; compare squared distances, no square roots |
| [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | K-way merge, O(N log k) |
| [295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Two heaps and the rebalancing invariant |
| [621. Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Heap simulation, or a counting formula once you see it |
| [253. Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) *(LeetCode Premium)* | Min-heap of end times = rooms in use |

---

## Check yourself

1. A min-heap is stored as `[1, 3, 2, 7, 4, 5]`. Push 0 and show the array after sift-up. Then
   pop and show the array after sift-down.
2. Why does heapify cost O(n) and not O(n log n)? Give the argument without the formal sum.
3. You need the k **smallest** elements of a stream. Which kind of heap, of what size, and what
   is the eviction rule? State the question you ask each new element.
4. A Go solution uses `h.Push(x)` instead of `heap.Push(h, x)` everywhere. It compiles. What
   happens, and on which inputs does it still return the right answer?
5. In the median finder, every `AddNum` pushes into `lo` first and then moves `lo`'s max to `hi`.
   Explain why this guarantees every value in `lo` is ≤ every value in `hi`, regardless of `x`.
6. Your interviewer says k can be as large as n/2 and memory is tight. Compare the size-k heap with
   quickselect for that case, and say what you'd do if the input were a stream you can't store.

(Answers in [`15-exercises.md`](15-exercises.md).)
