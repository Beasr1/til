# 6. Binary search, including binary search on the answer

Linear search asks every element "are you the one?". If the data has an order you can
exploit, one question can rule out half of what's left, and log₂(10^9) is only about 30. That
much is familiar. What's less familiar is how often the "order" isn't a sorted array at all
but a yes/no question whose answer flips once, and how often people who know binary search
still get the boundaries wrong. This chapter fixes both with one template.

## 6.1 Why the boundaries are hard

Binary search has a reputation. Jon Bentley gave it as an exercise to professional
programmers, and in *Programming Pearls* (column 4) reports that after testing their own code,
ninety percent of them found bugs in it. In 2006 Joshua Bloch showed that the binary search in
Bentley's own book, *and* the one he had written for the Java standard library, overflowed on
arrays of about 2^30 elements because `(low + high) / 2` exceeds the largest `int`
([Google Research blog, 2006](https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/)).
The JDK bug sat there for nine years.

The difficulty isn't the idea. It's that there are a dozen near-identical variants (`lo < hi`
or `lo <= hi`, `hi = mid` or `hi = mid − 1`, return `lo` or `hi`), and mixing two of them
gives an off-by-one or an infinite loop. The cure is to stop memorising variants and use one
formulation where the boundaries follow from a stated invariant.

## 6.2 One template: find the first `true`

### The reframe

> **Teacher's aside.** Stop thinking of binary search as "find x in a sorted array". Think of it
> as: **there is a yes/no question about each position, the answers go `false … false true …
> true`, and you want the first `true`.** "Find x" is the special case where the question is
> `a[i] >= x`. Once you think this way, the same ten lines solve lower bound, upper bound, first
> bad version, square root, minimum speed, minimum capacity, and so on. The only thing that
> changes is the question. The boundary arithmetic never changes, so you stop getting it wrong.

The question must be **monotone**: once it becomes true, it stays true. That's the only
requirement. Sortedness is just one way to get it.

```go
// firstTrue returns the smallest x in [lo, hi] with ok(x) true, assuming ok is
// false...false true...true over that range. Returns hi+1 if none is true.
func firstTrue(lo, hi int, ok func(int) bool) int {
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if ok(mid) {
			hi = mid - 1 // mid works; the answer is mid or something left of it
		} else {
			lo = mid + 1 // mid fails; the answer is strictly right of it
		}
	}
	return lo
}
```

**Why it's right**, as an invariant you can say out loud: everything left of `lo` is known
false, and everything right of `hi` is known true. Each iteration shrinks the unknown range
`[lo, hi]` by at least one, so the loop ends, and it ends with `lo = hi + 1`. At that point
nothing is unknown: `lo` is the first position known true (or `hi + 1` of the original range,
if nothing was).

### Worked example: lower bound

The first index with `a[i] >= 3` in `a = [1, 3, 3, 5]`:

```go
func lowerBound(a []int, target int) int {
	return firstTrue(0, len(a)-1, func(i int) bool { return a[i] >= target })
}
```

```
         a:  1  3  3  5          ok(i) = a[i] >= 3:   F  T  T  T
lo  hi   mid   a[mid]   ok?    move
0   3     1      3      T      hi = 0
0   0     0      1      F      lo = 1
1   0     stop: lo > hi  →  return 1
```

Returns 1, the first 3. Searching for 6 returns 4 (`len(a)`, "insert at the end"); searching an
empty slice returns 0. No special cases.

| Want | Question to pass |
|---|---|
| first index with `a[i] >= x` (lower bound) | `a[i] >= x` |
| first index with `a[i] > x` (upper bound) | `a[i] > x` |
| does `x` occur? | lower bound `i`, then check `i < n && a[i] == x` |
| last index with `a[i] <= x` | upper bound minus one |
| count of `x` | upper bound − lower bound |

**Complexity:** O(log n) calls to the question.

> ⚠️ **Compute `mid` as `lo + (hi − lo)/2`.** `(lo + hi)/2` overflows when `lo + hi` exceeds
> the integer type. With Go `int32`s of 2·10^9 and 2.1·10^9, `(lo+hi)/2` gives −97483648 (I ran
> it). Go's `int` is 64 bits on every judge you'll meet, so array indices are safe in practice,
> but binary search *on the answer* (§6.4) often searches value ranges near the type's limit,
> and in C++ signed overflow is undefined behaviour rather than a wrong number. In JavaScript,
> don't use the `(lo + hi) >> 1` idiom on large values: `>>` truncates to 32 bits.

### What the standard libraries give you

| Language | First true / lower bound | Notes |
|---|---|---|
| Go | `sort.Search(n, f)` returns the smallest `i` in `[0, n)` with `f(i)` true | exactly `firstTrue`; `slices.BinarySearch` returns `(index, found)` |
| Rust | `slice.partition_point(pred)` | `pred` is true for the **prefix**, so pass `\|x\| x < target`; `binary_search` returns *some* matching index among duplicates, not necessarily the first |
| C++ | `std::lower_bound`, `std::upper_bound` | return iterators; subtract `begin()` for an index |
| JavaScript | none | write `firstTrue` |

Use them when the interviewer doesn't mind. Know how to write the loop for when they do.

## 6.3 Searching a rotated array

### The problem

A sorted array was rotated at an unknown point, like `[4, 5, 6, 7, 0, 1, 2]`. Find a target in
O(log n). The array isn't sorted, so the plain template doesn't directly apply.

### The intuition

Cut it at any `mid` and **at least one half is sorted**. You can tell which by comparing the
ends: if `nums[lo] <= nums[mid]`, the left half has no break in it. For the sorted half, a range
check tells you whether the target can be there. If it can, go there; if not, it's in the other
half.

```go
func searchRotated(nums []int, target int) int {
	lo, hi := 0, len(nums)-1
	for lo <= hi {
		mid := lo + (hi-lo)/2
		if nums[mid] == target {
			return mid
		}
		if nums[lo] <= nums[mid] { // left half [lo..mid] is sorted
			if nums[lo] <= target && target < nums[mid] {
				hi = mid - 1
			} else {
				lo = mid + 1
			}
		} else { // right half [mid..hi] is sorted
			if nums[mid] < target && target <= nums[hi] {
				lo = mid + 1
			} else {
				hi = mid - 1
			}
		}
	}
	return -1
}
```

The `<=` in `nums[lo] <= nums[mid]` matters when `lo == mid` (two elements left): the "left
half" is a single element, which is trivially sorted. Use `<` there and `[3, 1]` searching for
1 goes the wrong way.

This version assumes **distinct** values. With duplicates (`[1, 1, 1, 0, 1]`) you can't tell
which half is sorted when `nums[lo] == nums[mid] == nums[hi]`, and the worst case is
unavoidably O(n).

## 6.4 Binary search on the answer

### The problem

Koko has piles of bananas `[3, 6, 7, 11]` and 8 hours. Each hour she picks one pile and eats up
to `k` bananas from it. What's the smallest speed `k` that finishes in time?

There's no sorted array anywhere. But ask the yes/no question **"can she finish at speed k?"**
and notice: if speed 4 works, speed 5 works too. Feasibility is monotone in `k`. That's the
`false … false true … true` shape from §6.2, laid over the *space of possible answers*.

### The recipe

1. Name the answer you're searching for (here, the speed).
2. Find a range certain to contain it: `1` to `max(piles)` (at that speed every pile takes one
   hour, so it's always enough when `h ≥ len(piles)`).
3. Write `feasible(candidate)`, usually a greedy O(n) simulation.
4. Check monotonicity: if `x` is feasible, is `x + 1`? (Or the reverse, for maximising.)
5. Run `firstTrue` over the range.

```go
func minEatingSpeed(piles []int, h int) int {
	hoursAt := func(speed int) int {
		total := 0
		for _, p := range piles {
			total += (p + speed - 1) / speed // ceil(p / speed) without floats
		}
		return total
	}
	return firstTrue(1, slices.Max(piles), func(speed int) bool { return hoursAt(speed) <= h })
}
```

<details><summary>Rust</summary>

```rust
fn first_true(mut lo: i64, mut hi: i64, ok: impl Fn(i64) -> bool) -> i64 {
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        if ok(mid) {
            hi = mid - 1;
        } else {
            lo = mid + 1;
        }
    }
    lo
}

fn min_eating_speed(piles: &[i64], h: i64) -> i64 {
    let hours_at = |speed: i64| piles.iter().map(|&p| (p + speed - 1) / speed).sum::<i64>();
    let top = *piles.iter().max().unwrap();
    first_true(1, top, |speed| hours_at(speed) <= h)
}
```

</details>

<details><summary>C++</summary>

```cpp
long long firstTrue(long long lo, long long hi, const function<bool(long long)>& ok) {
    while (lo <= hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid - 1;
        else lo = mid + 1;
    }
    return lo;
}

int minEatingSpeed(const vector<int>& piles, int h) {
    auto hoursAt = [&](long long speed) {
        long long total = 0;
        for (int p : piles) total += (p + speed - 1) / speed;
        return total;
    };
    int top = *max_element(piles.begin(), piles.end());
    return (int)firstTrue(1, top, [&](long long s) { return hoursAt(s) <= h; });
}
```

</details>

<details><summary>JavaScript</summary>

```js
function firstTrue(lo, hi, ok) {
  while (lo <= hi) {
    const mid = lo + Math.floor((hi - lo) / 2); // no `| 0` tricks: they truncate to 32 bits
    if (ok(mid)) hi = mid - 1;
    else lo = mid + 1;
  }
  return lo;
}

function minEatingSpeed(piles, h) {
  const hoursAt = (speed) => piles.reduce((t, p) => t + Math.ceil(p / speed), 0);
  return firstTrue(1, Math.max(...piles), (speed) => hoursAt(speed) <= h);
}
```

</details>

The C++ version sums hours in `long long`: with up to 10^4 piles of up to 10^9 bananas each
(LeetCode 875's limits) and speed 1, the total is 10^13, far past a 32-bit `int`. Go's `int` is
64-bit on judges and JS doubles are exact to 2^53, so they're fine. In JS, beware
`Math.max(...piles)` on large arrays: spreading passes every element as a separate argument. On
Node 24 it worked for 10^5 elements and threw `RangeError: Maximum call stack size exceeded` at
2·10^5. Koko's 10^4 piles are safe; past that, use a loop.

### Worked example

`piles = [3, 6, 7, 11]`, `h = 8`. Search range `[1, 11]`:

```
lo  hi   mid   hours at mid: ⌈3/m⌉+⌈6/m⌉+⌈7/m⌉+⌈11/m⌉   ≤ 8?   move
1   11    6    1 + 1 + 2 + 2 = 6                         yes    hi = 5
1    5    3    1 + 2 + 3 + 4 = 10                        no     lo = 4
4    5    4    1 + 2 + 2 + 3 = 8                         yes    hi = 3
4    3    stop  →  return 4
```

**Complexity:** O(log M) feasibility checks, each O(n), where M is the size of the answer range:
O(n log M). For n = 10^4 and M = 10^9 that's about 3·10^5 operations.

### How to spot it

The phrasing is the giveaway:

| Phrasing | Search over | Feasibility check |
|---|---|---|
| "minimum speed / capacity / rate such that it finishes in time" | the rate | simulate, count time |
| "minimise the maximum" (split an array into k parts) | the maximum allowed | greedily pack parts, count them |
| "maximise the minimum" (place k items as far apart as possible) | the minimum gap | greedily place, count placed |
| "k-th smallest" in a structure you can't sort cheaply | the value | count elements ≤ value |

If you can check a candidate answer faster than you can construct the best one, consider
binary search on the answer.

> ⚠️ **Get the direction and the range right before writing code.** If feasibility goes
> `true … true false … false` (maximising), either search for the first `false` and subtract one,
> or flip the question. And make sure the range really contains the answer. A `hi` that's one too
> small fails silently: `firstTrue` returns `hi + 1`, a number you never tested.

## 6.5 How to recognise binary search

| You see… | Think… |
|---|---|
| Sorted input, "find", "first", "last", "insert position" | lower/upper bound via `firstTrue` |
| Sorted then rotated | one half is always sorted |
| O(log n) explicitly required | binary search, even if it isn't obvious over what |
| "Minimum X such that…", "maximise the minimum", huge answer range | binary search on the answer |
| n ≤ 10^5 but values up to 10^9 or 10^18 | probably searching the value range, not the indices |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [704. Binary Search](https://leetcode.com/problems/binary-search/) | The loop and its invariant, with nothing else in the way |
| [34. Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/) | Lower and upper bound from one template |
| [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) | Deciding which half is sorted, with the `<=` edge |
| [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) | A predicate that isn't "compare with target": `nums[i] <= nums[last]` |
| [875. Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/) | Binary search on the answer, the gentlest version |
| [1011. Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) | Same, with the lower bound of the range being `max(weights)`, not 1 |
| [410. Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum/) | "Minimise the maximum": binary search beats the DP |
| [4. Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) | Binary search over a partition point; famously fiddly |

---

## Check yourself

1. Using the invariant from §6.2, explain why `firstTrue` can't loop forever, and why it returns
   `lo` rather than `hi`.
2. Write the question you'd pass to `firstTrue` to find the **last** index with `a[i] <= x`,
   and say what you return when no such index exists.
3. In `searchRotated`, construct the smallest input where using `<` instead of `<=` in
   `nums[lo] <= nums[mid]` returns the wrong answer, and trace it.
4. For Capacity to Ship Packages, why is the lower end of the search range `max(weights)` and
   not 1? What goes wrong in the feasibility check if you start at 1?
5. Prove that Koko's feasibility is monotone: if speed `k` finishes within `h` hours, so does
   `k + 1`.
6. You need the integer square root of `n ≤ 10^18` in Go. Write the question for `firstTrue`
   and identify the overflow trap in the obvious version of it.

(Answers in [`15-exercises.md`](15-exercises.md).)
