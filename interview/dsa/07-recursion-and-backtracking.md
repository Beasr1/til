# 7. Recursion and backtracking

"List every subset of these n numbers." For n = 3 you could write three nested loops. For n
given at runtime you can't, because the number of loops would have to depend on the input.
Recursion is how you write a variable number of nested loops. Backtracking is the discipline
that makes that recursion explore every possibility exactly once without its bookkeeping
getting tangled. Every later chapter that walks a tree or a graph, or fills a DP table top-down,
leans on this one.

## 7.1 Recursion without the mysticism

### The one idea

A recursive function solves a problem by assuming it can already solve **smaller instances of
the same problem**, then combining their answers. The assumption feels like cheating. It isn't,
as long as two things hold:

1. **A base case:** the smallest instances are answered directly, without recursion.
2. **Progress:** every recursive call is on a strictly smaller instance, so you must reach a
   base case.

Those are exactly the two parts of a proof by induction, and that's the right way to think
about it. When writing `maxDepth(node)`, don't trace the calls in your head. Say: *"assume
`maxDepth` is correct for both children; then the answer for this node is one more than the
larger of them."* Then check the base case. That habit, sometimes called the **recursive leap of
faith**, is what lets you write recursion that's correct the first time.

### What it costs

Each call takes a **stack frame** until it returns. Draw the calls as a tree, the **recursion
tree**, and the costs read straight off it:

| Cost | Read from the tree |
|---|---|
| Time | number of nodes × work done in each node (excluding its children) |
| Extra space | the tree's **height**, the deepest chain of frames alive at once, plus whatever you allocate |

Height is the one that bites. [File 01 §1.4](01-foundations.md#space-complexity) has the depth
limits per language: about 10,000 frames in Node, 2 MiB for a spawned Rust thread, much more in
Go. Recursion that's O(n) deep on n = 10^5 isn't safe everywhere.

## 7.2 Backtracking: choose, explore, un-choose

### The problem

Generate all subsets of `[1, 2, 3]`.

### The intuition

Think of building one answer at a time in a single shared basket, `path`. At each step, you have
a choice of what to put in next. You **choose** an item (put it in the basket), **explore**
everything that can follow from that choice (recurse), then **un-choose** it (take it back
out) so the basket is exactly as it was before, ready for the next choice. Because you always
restore the basket, one basket serves the whole search.

The calls form a **decision tree**, and backtracking is a depth-first walk over it. You never
build the tree in memory. It only exists as the sequence of calls.

```
                         walk(0)  path=[]          record []
          ┌───────────────────┼─────────────────────┐
       +1 │                +2 │                  +3 │
   walk(1) [1]          walk(2) [2]           walk(3) [3]
     record              record                 record
     ┌────┴─────┐          │
  +2 │       +3 │       +3 │
walk(2) [1,2]  walk(3) [1,3]   walk(3) [2,3]
  record        record          record
     │
  +3 │
walk(3) [1,2,3]
  record
```

Starting each level's loop at `start` (one past the last index chosen) is what stops `[2, 1]`
appearing after `[1, 2]`: you only ever add elements to the right of the last one.

```go
func subsets(nums []int) [][]int {
	var out [][]int
	var path []int
	var walk func(start int)
	walk = func(start int) {
		out = append(out, slices.Clone(path)) // copy! path keeps changing after this
		for i := start; i < len(nums); i++ {
			path = append(path, nums[i]) // choose
			walk(i + 1)                  // explore
			path = path[:len(path)-1]    // un-choose
		}
	}
	walk(0)
	return out
}
```

<details><summary>Rust</summary>

```rust
fn subsets(nums: &[i32]) -> Vec<Vec<i32>> {
    fn walk(nums: &[i32], start: usize, path: &mut Vec<i32>, out: &mut Vec<Vec<i32>>) {
        out.push(path.clone());
        for i in start..nums.len() {
            path.push(nums[i]);
            walk(nums, i + 1, path, out);
            path.pop();
        }
    }
    let mut out = Vec::new();
    walk(nums, 0, &mut Vec::new(), &mut out);
    out
}
```

</details>

<details><summary>C++</summary>

```cpp
vector<vector<int>> subsets(const vector<int>& nums) {
    vector<vector<int>> out;
    vector<int> path;
    function<void(int)> walk = [&](int start) {
        out.push_back(path);  // push_back copies the vector
        for (int i = start; i < (int)nums.size(); ++i) {
            path.push_back(nums[i]);
            walk(i + 1);
            path.pop_back();
        }
    };
    walk(0);
    return out;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function subsets(nums) {
  const out = [], path = [];
  const walk = (start) => {
    out.push([...path]); // copy: path keeps changing
    for (let i = start; i < nums.length; i++) {
      path.push(nums[i]);
      walk(i + 1);
      path.pop();
    }
  };
  walk(0);
  return out;
}
```

</details>

Output order: `[] [1] [1 2] [1 2 3] [1 3] [2] [2 3] [3]`, which is the order a depth-first walk
visits the tree above.

**Complexity:** there are 2ⁿ subsets, and copying each costs up to n, so O(n · 2ⁿ) time. The
recursion is n deep: O(n) extra space, not counting the output.

> **Teacher's aside.** The copy on the first line of `walk` is where backtracking bugs live,
> and the four languages fail differently. Without `slices.Clone`, the Go version printed this
> when I ran it:
>
> ```
> [[] [1] [1 2] [3 3 3] [3 3] [3] [3 3] [3]]
> ```
>
> It's not even consistently wrong. A Go slice is a view onto a backing array. `out` stored
> views, and `path` kept writing into the same array. `[1]` and `[1 2]` survived only because
> `append` happened to reallocate a bigger array right after they were recorded; from then on
> every record shared one array, and they all show its final contents. A wrong answer that's
> partly right is far harder to debug than a crash. JavaScript fails the same way with
> `out.push(path)` (every entry is the same array object). C++ `push_back(path)` copies, so it
> works by default. Rust won't compile the aliasing version at all: you can't keep a reference to
> `path` in `out` while also mutating `path`. So the rule is: **record a copy, always.**

## 7.3 Permutations: tracking what's used

For permutations, order matters, so every unused element is a valid next choice, not just
those to the right. Track what's in the basket with a `used` array.

```go
func permute(nums []int) [][]int {
	var out [][]int
	used := make([]bool, len(nums))
	path := make([]int, 0, len(nums))
	var walk func()
	walk = func() {
		if len(path) == len(nums) {
			out = append(out, slices.Clone(path))
			return
		}
		for i, x := range nums {
			if used[i] {
				continue
			}
			used[i] = true
			path = append(path, x)
			walk()
			path = path[:len(path)-1]
			used[i] = false
		}
	}
	walk()
	return out
}
```

Note that the un-choose step undoes **both** pieces of state, `path` and `used`. Forget to reset
`used[i]` and later branches think the element is still taken. You get fewer permutations and
no error.

**Complexity:** n! leaves, each copied in O(n): O(n · n!). That's about 3.6·10^7 at n = 10 and
more than 10^9 beyond n ≈ 12, which is why [file 01's constraint
table](01-foundations.md#15-from-constraints-to-a-target-complexity) says n ≤ 10 for O(n!).

## 7.4 Pruning: cutting branches early

Backtracking is brute force. It becomes *fast* brute force when you can prove that a branch
can't lead to an answer and skip it without exploring it. That's **pruning**, and it's where the
interview discussion usually goes.

### Combination sum: prune by sorting

Find all combinations of candidates (reusable) summing to a target. Sort first, and the moment
one candidate is bigger than what's left, every later one is too: `break`, not `continue`.

```go
func combinationSum(cands []int, target int) [][]int {
	slices.Sort(cands)
	var out [][]int
	var path []int
	var walk func(start, remain int)
	walk = func(start, remain int) {
		if remain == 0 {
			out = append(out, slices.Clone(path))
			return
		}
		for i := start; i < len(cands); i++ {
			if cands[i] > remain {
				break // sorted, so every later candidate is too big as well: prune
			}
			path = append(path, cands[i])
			walk(i, remain-cands[i]) // i, not i+1: the same number may be reused
			path = path[:len(path)-1]
		}
	}
	walk(0, target)
	return out
}
```

`walk(i, …)` versus `walk(i+1, …)` is the entire difference between "each element may be reused"
and "each element used at most once". Read the problem statement twice for which one it wants.

### N-Queens: prune with O(1) conflict checks

Place n queens on an n×n board so that none attack each other. Place one queen per row; for each
row, try each column. The pruning is to keep sets of attacked columns and diagonals, so that "is
this square attacked?" is O(1) instead of a scan of the board.

Diagonals have a neat property: along one direction `r + c` is constant, along the other `r − c`
is constant.

```
r+c:  0 1 2 3        r−c:  0 −1 −2 −3
      1 2 3 4              1  0 −1 −2
      2 3 4 5              2  1  0 −1
      3 4 5 6              3  2  1  0
```

```go
func totalNQueens(n int) int {
	cols := make([]bool, n)
	diag := make([]bool, 2*n) // r+c is constant along one diagonal direction
	anti := make([]bool, 2*n) // r-c+n is constant along the other
	count := 0
	var place func(r int)
	place = func(r int) {
		if r == n {
			count++
			return
		}
		for c := range n {
			if cols[c] || diag[r+c] || anti[r-c+n] {
				continue // attacked: prune this whole subtree
			}
			cols[c], diag[r+c], anti[r-c+n] = true, true, true
			place(r + 1)
			cols[c], diag[r+c], anti[r-c+n] = false, false, false
		}
	}
	place(0)
	return count
}
```

`r − c` ranges from −(n−1) to n−1, so it's shifted by `+n` to make a valid index. For n = 8 this
returns 92, the known count.

### Duplicates in the input

With `[1, 2, 2]`, plain `subsets` produces `[1, 2]` twice (once with each 2). The fix: **sort**,
then at each depth skip a value equal to the one just tried *at the same depth*:

```go
for i := start; i < len(nums); i++ {
	if i > start && nums[i] == nums[i-1] {
		continue // same value at the same depth: would repeat a subset
	}
	// choose, explore, un-choose as before
}
```

`i > start` (not `i > 0`) is what allows `[2, 2]`: the second 2 may follow the first one at a
*deeper* level, it just can't replace it at the *same* level. For `[1, 2, 2]` this gives
`[] [1] [1 2] [1 2 2] [2] [2 2]`.

> ⚠️ **`i > 0` instead of `i > start` silently drops valid answers.** It forbids the second 2
> from ever being chosen after the first, so `[1, 2, 2]` and `[2, 2]` vanish. Nothing crashes;
> the output just looks plausible and is short.

## 7.5 Complexity, at a glance

| Problem shape | Leaves of the tree | Typical time |
|---|---|---|
| Subsets of n | 2ⁿ | O(n · 2ⁿ) |
| Permutations of n | n! | O(n · n!) |
| Combinations of k from n | C(n, k) | O(k · C(n, k)) |
| Each position has b choices, depth d (letter combinations, word search) | b^d | O(b^d), before pruning |

Pruning improves the real running time, often dramatically, but rarely the worst-case bound you
can prove. Say that if asked: "worst case is still exponential; pruning makes typical inputs
fast".

## 7.6 How to recognise backtracking

| You see… | Think… |
|---|---|
| "Return **all** …", "generate every …", "list each …" | backtracking: the output itself is exponential |
| n ≤ 10–20 | the setter is allowing exponential time |
| Placement under constraints (queens, sudoku) | one decision per row/cell, prune with O(1) checks |
| Search for a path in a grid that can't reuse cells | DFS with mark/unmark (backtracking on the grid) |
| "Count the ways" or "is it possible", **not** "list them" | try DP first ([file 12](12-dynamic-programming.md)); backtracking may be far too slow |

That last row matters. If you only need the *number* of answers, the same subproblems often
repeat, and memoising them turns exponential into polynomial. That's where dynamic programming
starts.

## Asked in interviews

| Problem | What it tests |
|---|---|
| [78. Subsets](https://leetcode.com/problems/subsets/) | The template, and recording a copy |
| [90. Subsets II](https://leetcode.com/problems/subsets-ii/) | Skipping duplicates at the same depth: `i > start` |
| [46. Permutations](https://leetcode.com/problems/permutations/) | `used` tracking; undoing *all* state |
| [39. Combination Sum](https://leetcode.com/problems/combination-sum/) | Reuse via `walk(i)`, pruning via sort + `break` |
| [17. Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) | Fixed depth, b choices per level |
| [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | Pruning by a validity invariant (closers never exceed openers) |
| [79. Word Search](https://leetcode.com/problems/word-search/) | Backtracking on a grid: mark the cell, recurse, unmark |
| [51. N-Queens](https://leetcode.com/problems/n-queens/) | O(1) conflict checks with diagonal indices |

---

## Check yourself

1. Write the recursive leap-of-faith argument for `maxDepth` of a binary tree. What exactly
   are you allowed to assume, and what must you check separately?
2. In `subsets`, why does starting the loop at `start` rather than 0 prevent `[2, 1]` from being
   generated? What would you get if you started at 0 and kept everything else the same?
3. Explain the Go output `[[] [1] [1 2] [3 3 3] [3 3] [3] [3 3] [3]]` precisely: why are exactly
   the first three entries correct?
4. In `permute`, you forget `used[i] = false`. For input `[1, 2, 3]`, how many permutations are
   returned, and which?
5. `combinationSum` uses `break` when `cands[i] > remain`. Why is `break` correct here, and what
   precondition makes it correct? What changes if you drop the sort?
6. Generate Parentheses asks for all valid strings of n pairs. Describe the decision tree, the
   two pruning rules, and why the result has no invalid strings even though you never check
   validity at the leaves.

(Answers in [`15-exercises.md`](15-exercises.md).)
