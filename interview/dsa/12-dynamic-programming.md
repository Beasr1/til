# 12. Dynamic programming

"How many ways can you climb n stairs, one or two steps at a time?" "What's the fewest coins
that make this amount?" "How similar are these two strings?" Each has exponentially many
candidate answers: every sequence of steps, every multiset of coins, every alignment. Backtracking
([file 07](07-recursion-and-backtracking.md)) would enumerate them all. Dynamic programming
notices that the enumeration keeps solving **the same smaller question** over and over, solves
each one once, and reuses the answer. That turns exponential into polynomial, and it's the
pattern people find hardest, mostly because they try to memorise recurrences instead of learning
how to find them. This chapter is about finding them.

## 12.1 Where DP comes from: the repeated subproblem

### Start with the obvious recursion

Climbing stairs: to reach step n, your last move was either one step (from n − 1) or two steps
(from n − 2). So the number of ways is

```
ways(n) = ways(n − 1) + ways(n − 2),      ways(0) = ways(1) = 1
```

Written as recursion, it's correct. It's also hopeless:

```
                          ways(5)
                 ┌───────────┴───────────┐
              ways(4)                  ways(3)
           ┌─────┴─────┐            ┌────┴────┐
        ways(3)      ways(2)     ways(2)    ways(1)
        ┌──┴──┐      ┌──┴──┐     ┌──┴──┐
    ways(2) ways(1) w(1) w(0)   w(1) w(0)
    ┌──┴──┐
  w(1)  w(0)
```

`ways(3)` is computed twice, `ways(2)` three times, and it gets worse exponentially. I counted:
`ways(40)` makes **331,160,281** calls to compute the answer 165,580,141, and took 282 ms in Go
on my machine. `ways(50)` would make about 123 times as many.

But there are only 41 *different* questions in there: `ways(0)` to `ways(40)`. Answer each one
once, remember it, and the whole thing is 41 steps.

### The two conditions

That's the entire idea, and it applies whenever two things are true:

| Condition | Meaning | In climbing stairs |
|---|---|---|
| **Optimal substructure** | the answer to the big problem can be built from answers to smaller instances of the same problem | `ways(n)` is built from `ways(n−1)` and `ways(n−2)` |
| **Overlapping subproblems** | the same smaller instances are needed again and again | `ways(3)` is needed by both `ways(5)` and `ways(4)` |

Without overlap, remembering answers saves nothing: that's plain divide and conquer, like merge
sort. Without substructure, there's no recurrence to write. The name comes from Richard Bellman,
who developed the method in the 1950s (*Dynamic Programming*, Princeton University Press, 1957);
"programming" there means planning, as in a schedule, not writing code.

## 12.2 The real skill: designing the state

> **Teacher's aside.** People learn DP as a list of recurrences ("LCS is `dp[i-1][j-1] + 1`"),
> and then freeze on a problem not on the list. The recurrence is the easy part. The hard part,
> and the only part worth practising, is the **state**: the smallest description of "where I am"
> that's enough to finish the problem optimally *without knowing how I got here*. Ask two
> questions. **What decision am I making at each step?** (take this coin or not; match these two
> characters or not.) **What do I need to remember about earlier decisions to make the remaining
> ones well?** (the amount still to make; how far through each string I am.) If the honest answer
> to the second question is "everything", there's no useful DP. If it's "just these two numbers",
> those two numbers are your state, and the recurrence is just: try each decision, move to the new
> state.

The state for some classic problems, and what it *deliberately forgets*:

| Problem | State | What it forgets (and why that's fine) |
|---|---|---|
| Climbing stairs | which step I'm on | the route taken so far: it doesn't affect the ways onward |
| Coin change | the amount still to make | which coins made up the rest: only the remainder matters |
| House robber | which house; whether I robbed the previous one | earlier houses: only the neighbour constrains the next choice |
| Longest common subsequence | how far through each string: `(i, j)` | the matched characters so far |
| 0/1 knapsack | which item I'm on; capacity left | which items were taken: only the remaining capacity matters |
| Stock with cooldown | day; holding / not holding / cooling down | the price paid: it's already accounted for in the profit |

## 12.3 A recipe

Once you have a state, the rest is mechanical:

1. **Define the state in words**, precisely. "`dp[a]` = the fewest coins that sum to exactly `a`."
   Not "dp of a". If you can't say it in a sentence, you don't have it yet.
2. **Write the transition**: try every decision from this state. "Last coin was `c`, so
   `dp[a] = 1 + min over c of dp[a − c]`."
3. **Base cases**: the states you can answer without deciding anything. `dp[0] = 0`.
4. **Order**: compute a state only after the states it depends on. Here, increasing `a`.
5. **Where's the answer?** `dp[amount]`, or the max over the table, or `dp[n][m]`.
6. **Complexity** = (number of states) × (work per transition). Here `amount` states × `len(coins)`
   choices.

## 12.4 Top-down and bottom-up

There are two ways to "solve each subproblem once", and they compute the same table.

**Top-down (memoisation):** write the plain recursion, and cache each result the first time it's
computed. Coin change:

```go
func coinChangeMemo(coins []int, amount int) int {
	memo := make(map[int]int)
	var best func(a int) int // -1 means impossible
	best = func(a int) int {
		if a == 0 {
			return 0
		}
		if v, ok := memo[a]; ok {
			return v
		}
		res := -1
		for _, c := range coins {
			if c <= a {
				if sub := best(a - c); sub != -1 && (res == -1 || sub+1 < res) {
					res = sub + 1
				}
			}
		}
		memo[a] = res
		return res
	}
	return best(amount)
}
```

**Bottom-up (tabulation):** fill the table in dependency order with loops, no recursion.

```go
func coinChange(coins []int, amount int) int {
	const inf = math.MaxInt32
	dp := make([]int, amount+1) // dp[a] = fewest coins summing to a
	for a := 1; a <= amount; a++ {
		dp[a] = inf
		for _, c := range coins {
			if c <= a && dp[a-c] != inf {
				dp[a] = min(dp[a], dp[a-c]+1)
			}
		}
	}
	if dp[amount] == inf {
		return -1
	}
	return dp[amount]
}
```

<details><summary>Rust</summary>

```rust
fn coin_change(coins: &[usize], amount: usize) -> i32 {
    const INF: u32 = u32::MAX;
    let mut dp = vec![INF; amount + 1];
    dp[0] = 0;
    for a in 1..=amount {
        for &c in coins {
            if c <= a && dp[a - c] != INF {
                dp[a] = dp[a].min(dp[a - c] + 1);
            }
        }
    }
    if dp[amount] == INF { -1 } else { dp[amount] as i32 }
}
```

The `dp[a - c] != INF` check isn't only about correctness here: `u32::MAX + 1` would panic in a
debug build ([file 01 §1.7](01-foundations.md#the-defaults-that-bite)).

</details>

<details><summary>C++</summary>

```cpp
int coinChange(const vector<int>& coins, int amount) {
    const int INF = INT_MAX;
    vector<int> dp(amount + 1, INF);
    dp[0] = 0;
    for (int a = 1; a <= amount; ++a)
        for (int c : coins)
            if (c <= a && dp[a - c] != INF) dp[a] = min(dp[a], dp[a - c] + 1);
    return dp[amount] == INF ? -1 : dp[amount];
}
```

Here the guard prevents `INT_MAX + 1`, which is signed overflow: undefined behaviour, not just a
wrong number.

</details>

<details><summary>JavaScript</summary>

```js
function coinChange(coins, amount) {
  const dp = new Array(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (let a = 1; a <= amount; a++) {
    for (const c of coins) {
      if (c <= a) dp[a] = Math.min(dp[a], dp[a - c] + 1);
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

JavaScript is the one language where the guard is unnecessary: `Infinity + 1` is still
`Infinity`.

</details>

### Worked example

`coins = [1, 3, 4]`, `amount = 6`, which is the case where greedy fails
([file 11 §11.1](11-greedy-and-intervals.md#111-what-greedy-means-and-why-it-needs-a-proof)):

```
a    last coin 1        last coin 3        last coin 4        dp[a]
0    —                  —                  —                  0
1    dp[0]+1 = 1        —                  —                  1
2    dp[1]+1 = 2        —                  —                  2
3    dp[2]+1 = 3        dp[0]+1 = 1        —                  1
4    dp[3]+1 = 2        dp[1]+1 = 2        dp[0]+1 = 1        1
5    dp[4]+1 = 2        dp[2]+1 = 3        dp[1]+1 = 2        2
6    dp[5]+1 = 3        dp[3]+1 = 2        dp[2]+1 = 3        2   (3 + 3)
```

**Complexity:** `amount` states × `len(coins)` transitions: O(amount × k) time, O(amount) space.

### Which to use

| | Top-down (memo) | Bottom-up (table) |
|---|---|---|
| Writing it | start from the recursion; usually quicker to get right | must work out the evaluation order |
| Computes | only the states actually reached | every state, reached or not |
| Stack | recursion depth can equal the number of states: a risk at 10^5 ([file 01 §1.4](01-foundations.md#space-complexity)) | none |
| Space optimisation | hard | easy: often keep only the last row (§12.5) |
| Constant factor | map lookups and calls; slower | array indexing; faster |

A good interview path: say the recursion, memoise it (that's already a correct, efficient answer),
then convert to bottom-up if asked or if depth is a concern.

## 12.5 One-dimensional: keeping only what you need

**House robber:** houses in a row with values; you can't rob two adjacent houses. Maximise the
total. State: at each house, the best total so far **if I took the previous house** and **if I
didn't**. Each new house only looks one step back, so two variables replace the whole table.

```go
func rob(nums []int) int {
	take, skip := 0, 0 // best total if we took / skipped the previous house
	for _, x := range nums {
		take, skip = skip+x, max(take, skip)
	}
	return max(take, skip)
}
```

```
nums = [2, 7, 9, 3, 1]
x    take = skip + x      skip = max(take, skip)
2       0 + 2 = 2              max(0, 0) = 0
7       0 + 7 = 7              max(2, 0) = 2
9       2 + 9 = 11             max(7, 2) = 7
3       7 + 3 = 10             max(11, 7) = 11
1      11 + 1 = 12             max(10, 11) = 11
answer: max(12, 11) = 12        (2 + 9 + 1)
```

Go's simultaneous assignment `take, skip = skip+x, max(take, skip)` evaluates both right-hand
sides with the *old* values, which is exactly what's needed. In a language without it, save the
old `take` first; overwriting it before computing `skip` is a classic slip.

**Maximum subarray (Kadane's algorithm)** is the same shape: the best sum of a subarray *ending
here* is either this element alone or this element plus the best ending at the previous one.

**Word break** is one-dimensional over *prefixes*: `ok[i]` = the first `i` characters can be split
into dictionary words. The decision is where the last word starts, at some `j < i`; the prefix
before it must itself be splittable.

```go
func wordBreak(s string, words []string) bool {
	dict := make(map[string]bool)
	for _, w := range words {
		dict[w] = true
	}
	ok := make([]bool, len(s)+1) // ok[i]: s[:i] can be segmented
	ok[0] = true
	for i := 1; i <= len(s); i++ {
		for j := 0; j < i; j++ {
			if ok[j] && dict[s[j:i]] {
				ok[i] = true
				break
			}
		}
	}
	return ok[len(s)]
}
```

`ok[0] = true` is the empty prefix again: zero characters can always be segmented (into zero
words).

## 12.6 Two-dimensional: two sequences

**Longest common subsequence:** the longest sequence of characters appearing in order (not
necessarily adjacent) in both strings. State `dp[i][j]` = LCS length of the first `i` characters of
`a` and the first `j` of `b`. The decision concerns the last characters:

- if `a[i−1] == b[j−1]`, use them both: `dp[i−1][j−1] + 1`;
- otherwise, one of them isn't in the LCS: `max(dp[i−1][j], dp[i][j−1])`.

```go
func longestCommonSubsequence(a, b string) int {
	dp := make([][]int, len(a)+1)
	for i := range dp {
		dp[i] = make([]int, len(b)+1)
	}
	for i := 1; i <= len(a); i++ {
		for j := 1; j <= len(b); j++ {
			if a[i-1] == b[j-1] {
				dp[i][j] = dp[i-1][j-1] + 1
			} else {
				dp[i][j] = max(dp[i-1][j], dp[i][j-1])
			}
		}
	}
	return dp[len(a)][len(b)]
}
```

`a = "abcde"`, `b = "ace"`; row 0 and column 0 are the empty prefix:

```
          ""   a    c    e
     ""    0   0    0    0
     a     0   1↖   1    1
     b     0   1    1    1
     c     0   1    2↖   2
     d     0   1    2    2
     e     0   1    2    3↖        ↖ = characters matched; answer 3 ("ace")
```

**Complexity:** O(nm) time and space; space drops to O(min(n, m)) by keeping two rows.

The extra row and column for the empty prefix play the same role as `prefix[0] = 0` in
[file 02 §2.3](02-arrays-and-strings.md#23-prefix-sums): they make the base cases zero and remove
every `if i == 0` from the loop. **Edit distance** is the same table with three choices
(insert, delete, replace) instead of two.

## 12.7 Knapsack: the direction of the loop matters

**Partition equal subset sum:** can the array be split into two parts with equal sums? That's
"is there a subset summing to half the total?", a 0/1 knapsack where each item is used at most
once. State `can[s]` = some subset of the items processed so far sums to `s`.

```go
func canPartition(nums []int) bool {
	total := 0
	for _, x := range nums {
		total += x
	}
	if total%2 == 1 {
		return false
	}
	target := total / 2
	can := make([]bool, target+1) // can[s]: some subset of items seen so far sums to s
	can[0] = true
	for _, x := range nums {
		for s := target; s >= x; s-- { // downwards, so each item is used at most once
			can[s] = can[s] || can[s-x]
		}
	}
	return can[target]
}
```

> ⚠️ **In a one-row knapsack, the loop direction decides whether items can repeat.** Going
> *downwards*, `can[s − x]` hasn't been updated for this item yet, so it reflects only earlier
> items: each item used at most once (0/1 knapsack). Going *upwards*, `can[s − x]` may already
> include this item, so it can be used again (unbounded knapsack). With the single item 2 and an
> upward loop, `can[2]` becomes true and then `can[4]` reads it and becomes true too: 2 + 2,
> from one item. Coin change loops upwards on purpose, because coins *are* reusable. Same code
> shape, opposite meaning, decided by one loop bound.

**Complexity:** O(n × target) time, O(target) space. This is **pseudo-polynomial**: polynomial in
the *value* of target, not in the size of the input. With values up to 10^9 it's useless, which is
why subset sum is NP-complete in general and still fast for LeetCode's limits (n ≤ 200, values
≤ 100 for problem 416).

## 12.8 When the DP itself can be beaten

**Longest increasing subsequence.** The direct DP is `dp[i]` = longest increasing subsequence
ending at `i`, computed from every earlier `j < i` with `nums[j] < nums[i]`: O(n²). A cleverer
state does it in O(n log n): `tails[k]` = the **smallest possible last element** of an increasing
subsequence of length `k + 1`. `tails` is always sorted, so each new element binary-searches its
place ([file 06](06-binary-search.md)):

```go
func lengthOfLIS(nums []int) int {
	tails := []int{} // tails[k] = smallest tail of any increasing subsequence of length k+1
	for _, x := range nums {
		i, _ := slices.BinarySearch(tails, x) // first tail >= x
		if i == len(tails) {
			tails = append(tails, x)
		} else {
			tails[i] = x
		}
	}
	return len(tails)
}
```

Keeping the smallest tail for each length is greedy reasoning (a smaller tail can only make it
easier to extend) layered on a DP state. `tails` is *not* itself an increasing subsequence of the
input; only its length is meaningful. The technique is known as **patience sorting**, after the
card game.

## 12.9 How to recognise DP

| You see… | Think… |
|---|---|
| "Number of ways to…" | DP; counting means adding over choices |
| "Minimum / maximum cost, length, profit" with choices that interact | DP; greedy only if you can prove it ([file 11](11-greedy-and-intervals.md)) |
| "Can you…" / "is it possible…" over subsets or splits | boolean DP (often knapsack-shaped) |
| Two strings or sequences, compare/align them | 2-D DP on prefixes `(i, j)` |
| A grid, moving only right/down | 2-D DP over cells |
| Backtracking works but n is too big, and only a count or an optimum is needed | memoise the backtracking: that *is* top-down DP |
| "Return all the actual combinations" | backtracking ([file 07](07-recursion-and-backtracking.md)); DP can't shrink the output |

And the honest part: DP problems vary a lot in difficulty, and some states are genuinely
non-obvious (interval DP, bitmask DP, DP on trees). The recipe in §12.3 doesn't make those easy; it
makes them *approachable*, because you know which question to work on (the state) instead of
staring at the whole problem.

## Asked in interviews

| Problem | What it tests |
|---|---|
| [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Seeing the recurrence; memo vs table vs two variables |
| [198. House Robber](https://leetcode.com/problems/house-robber/) | A state with a "did I take the previous one?" flag |
| [53. Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) | Kadane: best ending here |
| [322. Coin Change](https://leetcode.com/problems/coin-change/) | Unbounded knapsack; "impossible" sentinels; why greedy fails |
| [62. Unique Paths](https://leetcode.com/problems/unique-paths/) | Grid DP; reducing to one row |
| [139. Word Break](https://leetcode.com/problems/word-break/) | `ok[i]` over prefixes; a set for the dictionary |
| [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | O(n²) DP, then O(n log n) with binary search |
| [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | The canonical two-sequence table |
| [72. Edit Distance](https://leetcode.com/problems/edit-distance/) | Three transitions; base cases along the edges |
| [416. Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) | 0/1 knapsack and the loop direction |
| [309. Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/) | A state machine as the DP state |

---

## Check yourself

1. Merge sort has optimal substructure but isn't usually called DP. Which of the two conditions in
   §12.1 does it lack, and what does that mean for memoising it?
2. For **unique paths in a grid with obstacles** (move only right or down), define the state in one
   sentence, write the transition, and give the base cases. What changes for a cell holding an
   obstacle?
3. In `rob`, someone writes `take = skip + x` on one line and `skip = max(take, skip)` on the next.
   What does the function now compute for `[2, 7, 9, 3, 1]`, and why?
4. Convert `canPartition` to iterate `s` upwards. Give the smallest input where it now returns
   `true` but the right answer is `false`.
5. Word Break: `ok[i]` is true if `s[:i]` can be segmented. What's the time complexity of the
   version in the Go code above (assume `s[j:i]` and the map lookup cost O(i − j)), and how would
   you reduce it if the dictionary's words are all short?
6. You're given a problem: "choose a subset of jobs (start, end, profit), no two overlapping, to
   maximise profit". Greedy by end time worked in file 11 for *counting* jobs. Why doesn't it work
   here, and what's the DP state? Where does binary search come in?

(Answers in [`15-exercises.md`](15-exercises.md).)
