# 5. Stacks and queues, including monotonic stacks and deques

Some problems need you to remember things in a particular order, and to get back the most
recent thing, or the oldest thing, instantly. Searching a list for "the most recent unmatched
bracket" or "the nearest taller building to the left" costs O(n) every time, and doing it for
every element is O(n²). A stack or a queue keeps exactly the candidates that still matter, in
exactly the order you'll want them, so each element is handled a constant number of times.

## 5.1 The two containers

| | Stack | Queue |
|---|---|---|
| Order | **LIFO**: last in, first out | **FIFO**: first in, first out |
| Picture | a stack of plates | a queue at a ticket counter |
| Operations | push on top, pop from top | push at the back, pop from the front |
| Used for | matching, undo, "most recent unresolved", DFS | processing in arrival order, BFS ([file 10](10-graphs.md)) |

A **deque** (double-ended queue, pronounced "deck") allows push and pop at both ends. It's what
the monotonic window in §5.4 needs.

How each language spells them, and the costs that aren't obvious:

| | Go | Rust | C++ | JavaScript |
|---|---|---|---|---|
| Stack | slice: `s = append(s, x)`, `s = s[:len(s)-1]` | `Vec`: `push`, `pop` | `vector` or `stack` | `Array`: `push`, `pop` |
| Queue | slice plus a head index | `VecDeque` | `queue` / `deque` | array plus a head index |
| Deque | slice plus a head index, if you only pop the front | `VecDeque` | `deque` | array plus a head index |

All of those operations are O(1) or amortised O(1), *except* the obvious way of popping the
front in JavaScript. [File 01 §1.7](01-foundations.md#17-languages-what-this-course-uses-and-what-you-should)
has the measurements: `Array.prototype.shift()` went quadratic on Node 24, taking 9.9 s to drain
4·10^5 elements. Use a head index.

## 5.2 Stacks for matching

### The problem

Is a string of brackets like `"([]{})"` balanced? Every closer must match the most recent
**unmatched** opener. "Most recent" is the word that means stack.

```go
func isValid(s string) bool {
	pair := map[byte]byte{')': '(', ']': '[', '}': '{'}
	stack := []byte{}
	for i := 0; i < len(s); i++ {
		c := s[i]
		if open, isClose := pair[c]; isClose {
			if len(stack) == 0 || stack[len(stack)-1] != open {
				return false
			}
			stack = stack[:len(stack)-1]
		} else {
			stack = append(stack, c)
		}
	}
	return len(stack) == 0
}
```

Two failure modes, both handled: a closer with nothing to match (empty stack), and openers left
over at the end (`"(("`, caught by the final `len(stack) == 0`).

The same shape evaluates **Reverse Polish notation** (push numbers; on an operator, pop two,
push the result) and simplifies Unix paths (push directory names; `..` pops).

### A queue from two stacks

A classic design question: build a FIFO queue using only stacks. Keep an `in` stack for
pushes and an `out` stack for pops. When `out` is empty and you need to pop, move everything
from `in` to `out`, which reverses it, so the oldest element is now on top. Any single pop can
cost O(n), but each element is moved from `in` to `out` exactly once in its life, so n
operations cost O(n) in total: **amortised O(1)**, the same argument as
[file 01 §1.4](01-foundations.md#14-amortised-cost-and-space).

## 5.3 Monotonic stack

### The problem

**Daily temperatures:** for each day, how many days until a warmer one? The brute force scans
forwards from each day: O(n²) for a decreasing input.

### The intuition

Stand on each day and look back at the days still waiting for their answer. If today is warmer
than some of them, today *is* their answer, and they can stop waiting. The days still waiting
never get warmer from oldest to newest: if a newer day were warmer than an older one, it would
already have answered it. (Equal temperatures can wait side by side, because "warmer" means
strictly warmer.)

So keep a stack of **unresolved days**, whose temperatures never increase from bottom to top.
Each new day pops every cooler day off the top, answering each one, and then waits on the stack
itself.

> **Teacher's aside.** "Monotonic stack" sounds like a data structure. It's better thought of
> as a **to-do list of unanswered questions**, where the questions happen to come out in
> sorted order. The monotonic property isn't something you enforce for its own sake. It falls
> out of the fact that anything which would break the order has just been answered and
> removed. When you can't work out whether to keep the stack increasing or decreasing, ask
> instead: *which elements are still waiting, and what would answer them?* The order follows.

```go
func dailyTemperatures(temps []int) []int {
	ans := make([]int, len(temps))
	stack := []int{} // indices; temps at these indices never increase bottom->top
	for i, t := range temps {
		for len(stack) > 0 && temps[stack[len(stack)-1]] < t {
			j := stack[len(stack)-1]
			stack = stack[:len(stack)-1]
			ans[j] = i - j // i is the first warmer day after j
		}
		stack = append(stack, i)
	}
	return ans // anything left on the stack never got a warmer day: stays 0
}
```

<details><summary>Rust</summary>

```rust
fn daily_temperatures(temps: &[i32]) -> Vec<i32> {
    let mut ans = vec![0; temps.len()];
    let mut stack: Vec<usize> = Vec::new();
    for (i, &t) in temps.iter().enumerate() {
        while let Some(&j) = stack.last() {
            if temps[j] >= t {
                break;
            }
            stack.pop();
            ans[j] = (i - j) as i32;
        }
        stack.push(i);
    }
    ans
}
```

</details>

<details><summary>C++</summary>

```cpp
vector<int> dailyTemperatures(const vector<int>& temps) {
    vector<int> ans(temps.size(), 0), stack;
    for (int i = 0; i < (int)temps.size(); ++i) {
        while (!stack.empty() && temps[stack.back()] < temps[i]) {
            int j = stack.back();
            stack.pop_back();
            ans[j] = i - j;
        }
        stack.push_back(i);
    }
    return ans;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function dailyTemperatures(temps) {
  const ans = new Array(temps.length).fill(0);
  const stack = [];
  for (let i = 0; i < temps.length; i++) {
    while (stack.length && temps[stack[stack.length - 1]] < temps[i]) {
      const j = stack.pop();
      ans[j] = i - j;
    }
    stack.push(i);
  }
  return ans;
}
```

</details>

### Worked example

`temps = [73, 74, 75, 71, 69, 72, 76, 73]`. The stack shows `index(temperature)`:

```
i   t    pops (answer set)                  stack after push
0   73   —                                  0(73)
1   74   0 → ans[0] = 1                     1(74)
2   75   1 → ans[1] = 1                     2(75)
3   71   —                                  2(75) 3(71)
4   69   —                                  2(75) 3(71) 4(69)
5   72   4 → ans[4] = 1, 3 → ans[3] = 2     2(75) 5(72)
6   76   5 → ans[5] = 1, 2 → ans[2] = 4     6(76)
7   73   —                                  6(76) 7(73)
end: 6 and 7 never popped → ans = [1, 1, 4, 2, 1, 1, 0, 0]
```

**Complexity:** every index is pushed once and popped at most once, so the inner loop runs at
most n times *in total*: O(n) time, O(n) space for the stack.

> ⚠️ **Store indices, not values, and decide `<` versus `<=` on purpose.** You need the index
> to compute a distance (`i − j`) or to write `ans[j]`, and you can always get the value back
> from the index. With duplicates, `<` means an equal temperature does *not* answer a waiting day
> ("strictly warmer"); `<=` means it does. In "largest rectangle in histogram" the choice decides
> whether equal bars are counted once or twice. Write down which one the problem wants before
> you write the loop.

### Where else it applies

Anything phrased as "for each element, the nearest element to the left/right that is
bigger/smaller":

| Question | Stack holds | Pop when the new element is… |
|---|---|---|
| Next greater element | indices of decreasing values | greater |
| Next smaller element | indices of increasing values | smaller |
| Largest rectangle in a histogram | indices of increasing heights | smaller; the popped bar's rectangle ends here |
| Trapping rain water (stack version) | indices of decreasing heights | taller; water fills above the popped bar |
| Stock span ("how many days back was the price ≤ today") | indices of decreasing prices | ≥ today's price |

## 5.4 Monotonic deque: sliding window maximum

### The problem

Return the maximum of every window of size k. Recomputing each window is O(nk). A heap gets
O(n log n), with some awkwardness over removing elements that have left the window. You can do
O(n).

### The intuition

An element is **useless** as soon as a larger element arrives after it: the newer one is both
bigger *and* will stay in the window longer, so the older one can never be a maximum again. Drop
it. What's left is a decreasing sequence of candidates, oldest at the front. The front is the
current maximum. Two kinds of removal happen:

- **From the back:** a new element evicts every smaller candidate (monotonic-stack behaviour).
- **From the front:** the front candidate expires when it slides out of the window.

Needing both ends is what makes this a deque.

```go
func maxSlidingWindow(nums []int, k int) []int {
	dq := []int{} // indices; nums at these are decreasing front->back
	head := 0     // dq[head:] is the live deque; avoids O(n) removal from the front
	out := []int{}
	for i, x := range nums {
		if head < len(dq) && dq[head] <= i-k { // front has slid out of the window
			head++
		}
		for len(dq) > head && nums[dq[len(dq)-1]] <= x { // smaller ones can never be a max again
			dq = dq[:len(dq)-1]
		}
		dq = append(dq, i)
		if i >= k-1 {
			out = append(out, nums[dq[head]])
		}
	}
	return out
}
```

`nums = [1, 3, −1, −3, 5, 3, 6, 7]`, `k = 3`. The deque shows `index(value)`:

```
i   x    expire front?   evict from back        deque after         window max
0    1   —               —                      0(1)
1    3   —               0(1)                   1(3)
2   −1   —               —                      1(3) 2(−1)          3
3   −3   —               —                      1(3) 2(−1) 3(−3)    3
4    5   1(3): 1 ≤ 4−3   3(−3), 2(−1)           4(5)                5
5    3   —               —                      4(5) 5(3)           5
6    6   —               5(3), 4(5)             6(6)                6
7    7   —               6(6)                   7(7)                7
```

**Complexity:** each index enters once and leaves once (from one end or the other): O(n) time,
O(k) live entries.

The head-index trick works here because the deque only ever pops from the front by moving
forward. In Rust or C++ you'd use `VecDeque` / `std::deque` with `pop_front` and not think about
it.

### A cousin: the min stack

"A stack that also returns its minimum in O(1)" is solved by pushing, alongside each value, the
minimum *at that moment*. Popping restores the previous minimum automatically, because it was
stored with the element below. It's the same idea as the monotonic stack: store the answer at
the time it's known, rather than searching for it later.

## 5.5 How to recognise these

| You see… | Think… |
|---|---|
| Brackets, nesting, "most recent unmatched", undo | stack |
| Evaluate an expression, simplify a path | stack |
| "Next/previous greater/smaller element" | monotonic stack of indices |
| Areas bounded by the nearest shorter bar | monotonic stack (histogram) |
| Max/min of every window of size k | monotonic deque |
| Process in arrival order, shortest path in steps | queue (BFS, [file 10](10-graphs.md)) |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | Matching with a stack; the two failure modes |
| [155. Min Stack](https://leetcode.com/problems/min-stack/) | Storing the answer alongside each element |
| [232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | Amortised analysis you can explain |
| [150. Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) | Operand order when popping (`a − b`, not `b − a`), and integer division towards zero |
| [496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/) | The monotonic stack plus a hash map for lookups |
| [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures/) | The canonical monotonic stack |
| [84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | The hard one: what each pop means, and the sentinel at the end |
| [239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | The monotonic deque, both kinds of removal |

---

## Check yourself

1. In `isValid`, which input would be wrongly accepted if you dropped the final
   `len(stack) == 0` check? Which would crash or be wrongly handled if you dropped the
   `len(stack) == 0` check inside the loop?
2. The two-stack queue sometimes takes O(n) for a single pop. Explain precisely why n
   operations still take only O(n) in total.
3. In `dailyTemperatures`, the temperatures on the stack never increase from bottom to top,
   though equal values can sit together. Prove that from the code, without assuming it, and say
   which single character of the code would make it *strictly* decreasing.
4. Run `dailyTemperatures` on `[70, 70, 71]` as written, then with `<` changed to `<=`. Which
   output is right for "a strictly warmer day", and what does the other one compute?
5. In `maxSlidingWindow`, why is it safe to expire at most **one** element from the front per
   iteration, rather than looping?
6. Someone solves sliding window maximum with a max-heap of `(value, index)` and lazily discards
   the top while its index is outside the window. What's the complexity, and what input makes the
   heap grow largest?

(Answers in [`15-exercises.md`](15-exercises.md).)
