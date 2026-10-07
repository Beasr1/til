# 11. Greedy algorithms and intervals

Some optimisation problems let you make the choice that looks best *right now*, never revisit
it, and still end up with the best overall answer. When that works, it's usually the fastest
possible solution: often one sort and one pass. The difficulty is that it very often doesn't
work, and a wrong greedy algorithm looks exactly as convincing as a right one. This chapter is
about telling them apart, and about intervals, the family of problems where greedy works most
often.

## 11.1 What "greedy" means, and why it needs a proof

A **greedy algorithm** builds a solution one decision at a time, at each step picking the
option that's best by some simple local rule, and never undoing a decision.

The classic illustration of why that's dangerous is making change. With Indian coins of 1, 2, 5
and 10, "always take the largest coin that fits" gives the fewest coins for any amount. Change
the denominations to 1, 3 and 4, and for 6 the greedy rule takes 4 + 1 + 1 (three coins) while
3 + 3 uses two. Same algorithm, same kind of problem; it's correct for one set of coins and wrong
for another. The difference isn't visible in the code.

> **Teacher's aside.** In an interview, a greedy idea is a *hypothesis*, not an answer. Before
> coding it, do two things. First, **try to break it**: spend a minute hunting for a small
> counterexample, the way 1-3-4 breaks the coin rule. Second, if you can't break it, sketch why
> it's right with an **exchange argument**: "take any optimal solution that differs from mine at
> the first choice; swap in my choice; it's no worse; repeat." If the swap step holds, greedy is
> optimal. If you can't make it hold, suspect dynamic programming
> ([file 12](12-dynamic-programming.md)), which considers every choice instead of committing to
> one. Interviewers often ask "why is greedy correct here?" precisely to see whether you know it
> needs a reason.

The standard textbook treatment of greedy proofs, including the exchange argument and its
cousin "greedy stays ahead", is chapter 4 of Kleinberg and Tardos, *Algorithm Design*.

## 11.2 A warm-up: Jump Game

Each `nums[i]` is the maximum jump length from index `i`. Can you reach the last index?

Brute force explores every jump sequence. The greedy observation: you don't care *how* you reach
a position, only the **furthest index reachable so far**. Walk left to right; if you ever stand
on an index beyond that reach, you're stuck.

```go
func canJump(nums []int) bool {
	reach := 0
	for i, x := range nums {
		if i > reach {
			return false
		}
		reach = max(reach, i+x)
	}
	return true
}
```

```
nums = [3, 2, 1, 0, 4]
i   x   i > reach?   reach after
0   3   no           3
1   2   no           3
2   1   no           3
3   0   no           3
4   4   4 > 3 → stuck: false
```

**Why it's right:** every index ≤ `reach` is genuinely reachable (you can stop short on any
jump), and nothing beyond `reach` is. O(n) time, O(1) space.

## 11.3 Intervals: sort first

An **interval** is a pair `[start, end]`: a meeting, a booking, a time range, a range of IP
addresses. Unsorted, interval questions need you to compare every pair. Sorted, overlapping
intervals become neighbours, and one pass is enough. The only question is *which* end to sort by.

| Question | Sort by | Why |
|---|---|---|
| Merge overlapping intervals | **start** | each interval can only overlap the run you're currently building |
| Keep the most non-overlapping intervals | **end** | finishing earliest leaves the most room for the rest |
| How many overlap at once (rooms needed) | starts and ends separately | it's a count of events over time |

### Merging overlapping intervals

Sort by start. Walk through; if the next interval starts before the current merged run ends, it
overlaps, so extend the run. Otherwise, start a new run.

```go
func merge(intervals [][]int) [][]int {
	slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[0], b[0]) })
	out := [][]int{}
	for _, iv := range intervals {
		if n := len(out); n > 0 && iv[0] <= out[n-1][1] { // overlaps the last kept interval
			out[n-1][1] = max(out[n-1][1], iv[1])
		} else {
			out = append(out, []int{iv[0], iv[1]})
		}
	}
	return out
}
```

<details><summary>Rust</summary>

```rust
fn merge(mut intervals: Vec<[i32; 2]>) -> Vec<[i32; 2]> {
    intervals.sort_unstable_by_key(|iv| iv[0]);
    let mut out: Vec<[i32; 2]> = Vec::new();
    for iv in intervals {
        match out.last_mut() {
            Some(last) if iv[0] <= last[1] => last[1] = last[1].max(iv[1]),
            _ => out.push(iv),
        }
    }
    out
}
```

</details>

<details><summary>C++</summary>

```cpp
vector<vector<int>> merge(vector<vector<int>> intervals) {
    sort(intervals.begin(), intervals.end(),
         [](const vector<int>& a, const vector<int>& b) { return a[0] < b[0]; });
    vector<vector<int>> out;
    for (auto& iv : intervals) {
        if (!out.empty() && iv[0] <= out.back()[1])
            out.back()[1] = max(out.back()[1], iv[1]);
        else
            out.push_back(iv);
    }
    return out;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function merge(intervals) {
  intervals.sort((a, b) => a[0] - b[0]); // a comparator is mandatory: the default sorts as strings
  const out = [];
  for (const [s, e] of intervals) {
    if (out.length && s <= out[out.length - 1][1]) {
      out[out.length - 1][1] = Math.max(out[out.length - 1][1], e);
    } else {
      out.push([s, e]);
    }
  }
  return out;
}
```

</details>

```
input:  [8,10] [1,3] [2,6] [15,18]
sorted: [1,3] [2,6] [8,10] [15,18]

iv        overlaps last?       out
[1,3]     (empty)              [1,3]
[2,6]     2 ≤ 3 → yes          [1,6]          (end = max(3, 6))
[8,10]    8 ≤ 6 → no           [1,6] [8,10]
[15,18]   15 ≤ 10 → no         [1,6] [8,10] [15,18]
```

**Complexity:** O(n log n) for the sort, O(n) for the pass.

None of the four sorts needs to be stable here, which is why the Rust version can use
`sort_unstable_by_key`. A stable sort matters only when equal keys must keep their input order.

> ⚠️ **Two details that cost people the problem.** First, `max(out[n-1][1], iv[1])`, not just
> `iv[1]`: an interval can be swallowed entirely (`[1,10]` then `[2,3]`), and overwriting the end
> would *shrink* the run. Second, decide whether touching intervals overlap. `[1,4]` and `[4,5]`
> merge under `<=` and don't under `<`. LeetCode 56 says they merge; a calendar system where
> meetings are half-open `[start, end)` would say they don't. Ask.

A smaller trap in comparators: `func(a, b []int) int { return a[0] - b[0] }` is a common Go and
JS idiom, and in Go it overflows if the values span more than the `int` range. `cmp.Compare`
(Go 1.21+) can't. In JavaScript the subtraction is safe for ordinary numbers; what isn't safe is
omitting the comparator ([file 01 §1.7](01-foundations.md#the-defaults-that-bite)).

## 11.4 Interval scheduling: sort by end

### The problem

Given intervals, remove the fewest so that the rest don't overlap. Equivalently: keep the most
non-overlapping intervals.

### The greedy, and why it's right

Sort by **end time**. Keep the first interval; then keep each next interval that starts at or
after the end of the last one kept.

```go
func eraseOverlapIntervals(intervals [][]int) int {
	slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[1], b[1]) })
	removed, end := 0, math.MinInt
	for _, iv := range intervals {
		if iv[0] >= end { // fits after the last kept one: keep it
			end = iv[1]
		} else {
			removed++
		}
	}
	return removed
}
```

The exchange argument, in full, because it's the model for all of them: take any optimal set of
kept intervals, sorted by end. Its first interval ends no earlier than the interval greedy picked
first (greedy picked the earliest end of all). Swap greedy's choice in. It ends at least as early,
so it can't clash with anything the optimal set kept later. The set is still valid and the same
size. Now repeat the argument on what remains. Greedy's choices can replace an optimal solution's
choices one at a time without ever losing an interval, so greedy keeps as many as optimal does.

Why sorting by *start* fails: `[1,100]`, `[2,3]`, `[4,5]`. By start, you keep `[1,100]` and throw
away the other two; by end, you keep `[2,3]` and `[4,5]`.

**Minimum arrows to burst balloons** is the same algorithm: an arrow at the end of the first
balloon bursts everything that started before then; the next arrow goes at the end of the first
balloon it missed.

## 11.5 Counting overlaps: the sweep line

### The problem

Given meetings, how many rooms are needed? That's the largest number of meetings in progress at
any single moment.

### The intuition

Forget which meeting is which. Picture walking along the timeline: every start is "+1 room in
use", every end is "−1". The answer is the highest the running count gets. So sort all the starts
and all the ends **separately** and merge them in time order: a **sweep line**.

```go
// minMeetingRooms: the most intervals overlapping at any instant. Intervals are [start, end).
func minMeetingRooms(intervals [][]int) int {
	starts := make([]int, len(intervals))
	ends := make([]int, len(intervals))
	for i, iv := range intervals {
		starts[i], ends[i] = iv[0], iv[1]
	}
	slices.Sort(starts)
	slices.Sort(ends)
	rooms, best, e := 0, 0, 0
	for _, s := range starts {
		for ends[e] <= s { // a meeting ending at s frees its room before this one starts
			rooms--
			e++
		}
		rooms++
		best = max(best, rooms)
	}
	return best
}
```

`[[0,30], [5,10], [15,20]]`: starts `0 5 15`, ends `10 20 30`.

```
start   ends ≤ start released    rooms   best
  0     —                          1       1
  5     —                          2       2
 15     10                         2       2
answer: 2
```

The `<=` encodes "a meeting ending at 10 frees its room for one starting at 10". The alternative
is a min-heap of end times, popping every meeting that has ended before pushing the new one; the
heap's size is the rooms in use ([file 09](09-heaps.md)). Both are O(n log n).

**Car pooling** is the same sweep with passenger counts instead of +1, and when the timeline is
small (LeetCode 1094 limits locations to 1,000), a **difference array** replaces the sort:
`diff[start] += k`, `diff[end] -= k`, then a prefix sum ([file 02 §2.3](02-arrays-and-strings.md#23-prefix-sums)).

## 11.6 How to recognise greedy and intervals

| You see… | Think… | But check… |
|---|---|---|
| Intervals, bookings, ranges, "overlap" | sort, then one pass | which end to sort by |
| "Maximum number of non-overlapping…", "minimum removals", "minimum arrows" | sort by end, keep greedily | touching = overlapping? |
| "How many at once", "rooms", "peak load" | sweep line or heap of end times | half-open or closed intervals |
| "Can you reach", "minimum jumps" | track the furthest reach | — |
| "Minimum coins/cost" with arbitrary values | **probably DP**, not greedy | try to break the greedy rule first |
| Choosing items by ratio (fractional knapsack) | greedy by value/weight | the 0/1 version (items can't be split) needs DP |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [55. Jump Game](https://leetcode.com/problems/jump-game/) | Furthest reach; the simplest greedy proof |
| [45. Jump Game II](https://leetcode.com/problems/jump-game-ii/) | Greedy as implicit BFS by levels |
| [134. Gas Station](https://leetcode.com/problems/gas-station/) | A greedy restart argument that surprises people |
| [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Sort by start; `max` on the end |
| [57. Insert Interval](https://leetcode.com/problems/insert-interval/) | Three phases (before, overlapping, after) without re-sorting |
| [435. Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Sort by end, and the exchange argument |
| [452. Minimum Number of Arrows to Burst Balloons](https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/) | The same greedy, closed intervals; overflow in comparators |
| [763. Partition Labels](https://leetcode.com/problems/partition-labels/) | Intervals hidden in a string (first and last occurrence) |
| [1094. Car Pooling](https://leetcode.com/problems/car-pooling/) | Sweep line, or a difference array |

---

## Check yourself

1. With coins `{1, 5, 10, 25}` greedy change-making is optimal. Find the smallest amount where it
   fails for `{1, 10, 25}`, and say what that tells you about proving greedy correct by testing
   examples.
2. In `merge`, what goes wrong if you write `out[n-1][1] = iv[1]` instead of taking the max? Give
   the smallest failing input.
3. Write out the exchange argument for "sort by end" in your own words, and show concretely why the
   same argument fails for "sort by shortest interval first" with `[1,5]`, `[4,7]`, `[6,10]`.
4. In `minMeetingRooms`, why is it valid to sort starts and ends independently, losing which end
   belongs to which start?
5. `[1,4]` and `[4,5]`: for each of `merge`, `eraseOverlapIntervals` and `minMeetingRooms` as
   written, do they count as overlapping? Is that consistent, and does it matter?
6. Gas Station: if the total gas is at least the total cost, a valid start exists, and it's the
   index right after the point where the running tank was lowest. Argue why.

(Answers in [`15-exercises.md`](15-exercises.md).)
