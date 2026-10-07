# 2. Arrays and strings: two pointers, sliding window, prefix sums

The obvious solution to most array problems looks at every pair of positions, or every
contiguous subarray, and that's O(n²) or worse. With n ≤ 10^5 that's 10^10 steps, so it's
out ([file 01 §1.5](01-foundations.md#15-from-constraints-to-a-target-complexity)). The three
patterns in this chapter each remove a different kind of repeated work from that brute force.

| Pattern | The repeated work it removes | Typical result |
|---|---|---|
| **Two pointers** | Checking pairs that can't possibly be the answer | O(n²) → O(n) |
| **Sliding window** | Re-scanning a subarray that overlaps the last one you scanned | O(n²) → O(n) |
| **Prefix sums** | Re-adding the same elements for every range query | O(n) per query → O(1) |

## 2.1 Two pointers

### The problem

Given a **sorted** array and a target, find two elements that add up to the target. Brute force
tries all n(n−1)/2 pairs. Sortedness is information the brute force ignores.

### The intuition

Put one finger on the smallest element and one on the largest. Add them.

- If the sum is **too small**, the left element is hopeless: it's already paired with the
  largest thing available and still falls short. Every other pairing would be smaller still. So
  discard it and move the left finger right.
- If the sum is **too big**, the right element is hopeless by the mirror argument. Move the right
  finger left.

Each step throws away one element *together with every pair it could have been in*. That's
why it's linear: you aren't checking pairs faster, you're proving whole rows of pairs
irrelevant without looking at them.

> **Teacher's aside.** People describe two pointers as "a technique with two indices". That
> makes it sound like syntax, and it's why they can't tell when to use it. The real content is
> an **elimination argument**: at every step you can prove that one candidate is useless, so you
> drop it. Before using two pointers, say that argument out loud ("if the sum is too small,
> nothing pairs with `nums[lo]`, because…"). If you can't finish the sentence, two pointers is
> the wrong tool, however much the problem looks like it.

### Worked example

`nums = [1, 3, 4, 6, 8, 11]`, `target = 10`:

```
step   lo  hi   nums[lo] + nums[hi]   verdict       move
  1     0   5        1 + 11 = 12      too big       hi--   (11 is useless)
  2     0   4        1 +  8 =  9      too small     lo++   (1 is useless)
  3     1   4        3 +  8 = 11      too big       hi--   (8 is useless)
  4     1   3        3 +  6 =  9      too small     lo++   (3 is useless)
  5     2   3        4 +  6 = 10      found         return (2, 3)
```

```go
func twoSumSorted(nums []int, target int) (int, int) {
	lo, hi := 0, len(nums)-1
	for lo < hi {
		sum := nums[lo] + nums[hi]
		switch {
		case sum == target:
			return lo, hi
		case sum < target:
			lo++ // need a bigger sum: only moving lo right can give one
		default:
			hi-- // need a smaller sum: only moving hi left can give one
		}
	}
	return -1, -1
}
```

**Complexity:** each iteration moves `lo` up or `hi` down by one, and they start n − 1 apart, so
at most n − 1 iterations: O(n) time, O(1) space.

### The other shape: same direction

Two pointers can also move the same way at different speeds. A **read** pointer scans every
element; a **write** pointer marks where the next kept element goes. That's how you remove
duplicates from a sorted array, or move zeros to the end, in place, in one pass. Floyd's
fast/slow cycle detection ([file 04](04-linked-lists.md)) is the same idea on a linked list.

### Extensions worth knowing

- **3Sum.** Sort, then for each `i` run two-pointer on the rest looking for `−nums[i]`.
  O(n log n) to sort plus O(n) per `i` gives O(n²). Skipping equal neighbours handles
  duplicates.
- **Container with most water.** Not sorted, but the elimination argument still works: the area
  is limited by the shorter wall, so moving the *taller* pointer inward can never help. Move
  the shorter one.

## 2.2 Sliding window

### The problem

Find the length of the longest substring of `s` with no repeated character. There are O(n²)
substrings. Checking each one from scratch costs O(n) more, so O(n³); keeping a set as you
extend each start gets you to O(n²). Still too slow for n = 5·10^4.

The waste is that the substring starting at `i` and the one starting at `i+1` overlap almost
completely, yet the brute force re-examines the overlap from scratch.

### The intuition

Picture a caterpillar crawling along the string. Its front end (`right`) always moves forward,
eating one character. If what it ate makes its body invalid (a repeat), the back end (`left`)
creeps forward until the body is valid again. The back never moves backwards, and the front
never moves backwards. Each character is eaten once and left behind once.

That only works because of a property worth naming: **if a window is invalid, every window
containing it is invalid too.** A substring with a repeat still has the repeat if you make it
longer. So once the window breaks, there's no point extending it further from the same
`left`, and shrinking from the left is the only move that can help.

### The template

The general shape, for "longest window satisfying a condition":

```
left = 0
for right in 0 ..< n:
    add s[right] to the window state
    while the window is invalid:
        remove s[left] from the window state
        left++
    record the answer for window [left, right]
```

For this problem there's a shortcut: instead of removing one character at a time, remember
where you last saw each character and jump `left` straight past it.

```go
func lengthOfLongestSubstring(s string) int {
	last := make(map[byte]int) // char -> index where we last saw it
	best, left := 0, 0
	for right := 0; right < len(s); right++ {
		c := s[right]
		if i, seen := last[c]; seen && i >= left {
			left = i + 1 // shrink: jump past the earlier copy
		}
		last[c] = right
		best = max(best, right-left+1)
	}
	return best
}
```

<details><summary>Rust</summary>

```rust
use std::collections::HashMap;

fn length_of_longest_substring(s: &str) -> usize {
    let bytes = s.as_bytes();
    let mut last: HashMap<u8, usize> = HashMap::new();
    let (mut best, mut left) = (0, 0);
    for (right, &c) in bytes.iter().enumerate() {
        if let Some(&i) = last.get(&c) {
            if i >= left {
                left = i + 1;
            }
        }
        last.insert(c, right);
        best = best.max(right - left + 1);
    }
    best
}
```

</details>

<details><summary>C++</summary>

```cpp
int lengthOfLongestSubstring(const string& s) {
    unordered_map<char, int> last;
    int best = 0, left = 0;
    for (int right = 0; right < (int)s.size(); ++right) {
        char c = s[right];
        auto it = last.find(c);
        if (it != last.end() && it->second >= left) left = it->second + 1;
        last[c] = right;
        best = max(best, right - left + 1);
    }
    return best;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function lengthOfLongestSubstring(s) {
  const last = new Map(); // char -> last index
  let best = 0, left = 0;
  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    if (last.has(c) && last.get(c) >= left) left = last.get(c) + 1;
    last.set(c, right);
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

</details>

All four index **bytes**, which is right for the ASCII inputs LeetCode uses. For arbitrary
Unicode text you'd iterate over runes (Go), `char`s (Rust) or code points (JS `for…of`); C++
`std::string` has no built-in notion of a character beyond a byte.

### Worked example

`s = "abcabcbb"`:

```
right  char  last seen?        left   window      len   best
  0     a    no                  0    a            1     1
  1     b    no                  0    ab           2     2
  2     c    no                  0    abc          3     3
  3     a    at 0, 0 ≥ 0 → yes   1    bca          3     3
  4     b    at 1, 1 ≥ 1 → yes   2    cab          3     3
  5     c    at 2, 2 ≥ 2 → yes   3    abc          3     3
  6     b    at 4, 4 ≥ 3 → yes   5    cb           2     3
  7     b    at 6, 6 ≥ 5 → yes   7    b            1     3
```

> ⚠️ **The `i >= left` check is the whole bug surface.** Run `"abba"` without it. At `right = 3`
> the map says `a` was last seen at 0, so `left` jumps *back* from 2 to 1 and the window `"bba"`
> is reported as length 3. The answer is 2. An index older than `left` is outside the window and
> must be ignored. Generally: `left` must never decrease.

**Complexity:** `right` advances n times; `left` only increases and never passes `right`, so
the total work across all iterations is O(n), even when the general template has an inner
`while`. That's an amortised argument, the same kind as in
[file 01 §1.4](01-foundations.md#14-amortised-cost-and-space). Space is O(k) for k distinct
characters.

### Fixed-size windows

If the problem fixes the window length at k ("maximum sum of any k consecutive elements"),
there's nothing to decide: add `nums[right]`, subtract `nums[right-k]` once `right ≥ k`, record.
The trap is the first k − 1 iterations, where the window isn't full yet.

### When sliding window fails

The pattern needs *monotonicity*: shrinking an invalid window must be the only way to fix it.
"Longest subarray with sum ≤ k" works when all numbers are positive, because adding a number
only increases the sum. **Add negative numbers and it breaks**: a window over the limit might
come back under it by growing to include a negative. For sums with negatives, use prefix sums
plus a hash map ([file 03 §3.3](03-hashing.md#33-prefix-sum-plus-hash-map)).

## 2.3 Prefix sums

### The problem

You get an array and then many queries "what is the sum of `nums[l..r]`?". Each query costs
O(n) by looping. With q queries that's O(nq).

### The intuition

Think of a car's odometer. To find how far you drove between Pune and Lonavala you don't
re-measure the road, you subtract two odometer readings. Precompute the odometer once; every
range is then one subtraction.

Define `prefix[i]` as the sum of the first `i` elements (so `prefix[0] = 0`). Then the sum of
`nums[l..r]` inclusive is `prefix[r+1] − prefix[l]`.

```
index:        0    1    2    3    4    5
nums:         3    1    4    1    5    9
prefix:   0   3    4    8    9   14   23
          ↑   ↑
   prefix[0]  prefix[1] = nums[0]

sum(nums[1..3]) = prefix[4] − prefix[1] = 9 − 3 = 6     (1 + 4 + 1 ✓)
```

```go
type RangeSum struct{ prefix []int }

func NewRangeSum(nums []int) RangeSum {
	prefix := make([]int, len(nums)+1) // prefix[i] = sum of nums[0:i]
	for i, x := range nums {
		prefix[i+1] = prefix[i] + x
	}
	return RangeSum{prefix}
}

// Sum of nums[l..r] inclusive.
func (r RangeSum) Sum(l, rr int) int { return r.prefix[rr+1] - r.prefix[l] }
```

**Complexity:** O(n) to build, O(1) per query, O(n) extra space.

> ⚠️ **Make the prefix array one longer than the input.** With `prefix[0] = 0` the formula
> `prefix[r+1] − prefix[l]` works for every range, including ones starting at 0. Without the
> leading zero you need a special case for `l = 0`, and that special case is where the
> off-by-one lives. It's the same half-open convention as Go's `s[l:r]`: start inclusive, end
> exclusive.

### Where prefix sums turn up in disguise

- **Product of array except self.** A prefix *product* from the left and a suffix product from
  the right; the answer at `i` is `left[i] × right[i]`. Same idea, different operator, no
  division needed.
- **2-D range sums.** `P[i][j]` = sum of the rectangle from (0,0) to (i−1, j−1); a rectangle
  query is four lookups with inclusion–exclusion.
- **"Number of subarrays summing to k".** Prefix sums turn "subarray sum" into "difference of
  two prefixes", and a hash map finds matching prefixes. That's
  [file 03 §3.3](03-hashing.md#33-prefix-sum-plus-hash-map).

## 2.4 How to recognise these

| You see… | Think… |
|---|---|
| Sorted array, find a pair or triple with some sum | two pointers from both ends |
| "In place", "O(1) extra space", remove/move elements | same-direction read/write pointers |
| "Longest / shortest **contiguous** subarray or substring such that…" | sliding window, *if* the condition is monotone |
| "At most k distinct", "no repeats", "contains all of…" | variable sliding window with a count map |
| Many range-sum queries on a fixed array | prefix sums |
| Subarray sum condition **with negatives** | prefix sums + hash map, not a window |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [167. Two Sum II – Input Array Is Sorted](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | The elimination argument in its purest form |
| [15. 3Sum](https://leetcode.com/problems/3sum/) | Sort + two pointers, and skipping duplicates without missing answers |
| [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Two pointers on unsorted data; can you justify which pointer moves? |
| [3. Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Variable window, and the stale-index bug |
| [424. Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | A window whose validity depends on a count, not a set |
| [76. Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | Shortest window: shrink *while* valid, recording as you go |
| [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Prefix/suffix products, no division |
| [42. Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Prefix maxima, or two pointers with a subtle elimination argument |

---

## Check yourself

1. In two-sum-sorted, the sum is too small and you move `lo` right. Someone says, "but what if
   `nums[lo]` pairs with an element left of `hi` that you haven't tried?" Answer them.
2. Why is the sliding-window template O(n) even though it has a `while` loop inside a `for`
   loop? Make the argument precise.
3. Trace `lengthOfLongestSubstring("tmmzuxt")` and give the answer. Which iteration would go
   wrong without the `i >= left` check?
4. "Longest subarray with sum at most k" — the array contains negative numbers. Construct a
   small input where the sliding window gives the wrong answer, and say what breaks.
5. You need the sum of `nums[l..r]` for 10^5 queries but the array also receives 10^5 point
   updates interleaved with them. What does an update cost with a plain prefix array, and why
   does that make prefix sums the wrong tool here?

(Answers in [`15-exercises.md`](15-exercises.md).)
