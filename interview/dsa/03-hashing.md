# 3. Hashing

A huge number of brute-force solutions contain the same inner loop: *scan everything to see
whether I've seen X before*. That scan is O(n), it runs n times, and the result is O(n²). A
hash map answers "have I seen X, and what did I record about it?" in expected O(1), which
turns the whole thing into O(n). This chapter is about seeing where that inner scan hides and
choosing what to store.

## 3.1 How a hash table works, just enough

### The problem it solves

An array gives O(1) access, but only by integer position. You want O(1) access by an
arbitrary key: a string, a pair, a large number.

### The intuition

Picture a post room with 64 pigeonholes. Every letter's address goes through a fixed
scrambling rule that produces a number from 0 to 63, and the letter goes into that pigeonhole.
To find a letter, apply the same rule to the address and look in one hole. You never search
the whole room.

That scrambling rule is the **hash function**. Two problems follow straight away:

- **Collisions.** Two different keys can scramble to the same hole. The table has to keep
  both and tell them apart, by chaining a small list per hole or by probing neighbouring
  holes. Either way, finding a key means hashing it *and* comparing it for equality against the
  few keys already there.
- **Filling up.** As the table fills, collisions multiply. So when the ratio of keys to holes
  (the **load factor**) passes a threshold, the table allocates a bigger one and re-inserts
  everything. That's the dynamic-array argument from
  [file 01 §1.4](01-foundations.md#14-amortised-cost-and-space) again, and it's why insert is
  *amortised* O(1).

Go's built-in `map` has used a Swiss-table design since
[Go 1.24](https://go.dev/doc/go1.24); Rust's `HashMap` is also Swiss-table based, with
SipHash 1-3 as its default hash. You don't need the internals for an interview. You need to
know which operations they make fast.

| Operation | Expected | Worst case |
|---|---|---|
| insert, lookup, delete | O(1) | O(n), if every key lands in one place |
| iterate all entries | O(n + capacity) | same |
| "smallest key", "next key above x" | **not supported** | use a sorted structure |

> **Teacher's aside.** "O(1)" is a statement about the *hash function*, not just the table. It
> assumes keys spread evenly. With a weak hash, an adversary can choose keys that all collide
> and turn every lookup into a scan. That is a real attack on web servers, called hash
> flooding. Go and Rust randomise their hash seed per process or per map, which is why Rust
> documents its `HashMap` as HashDoS-resistant. C++ is weaker here: on the libc++ I tested,
> `std::hash<int>` simply returns the integer itself, so keys that are multiples of the bucket
> count all land in the same bucket. Competitive programmers build inputs against this on
> purpose. In an interview, saying "expected O(1), assuming a good hash" is enough, and it
> shows you know the difference.

## 3.2 The complement lookup

### The problem

**Two Sum** (unsorted): find indices `i < j` with `nums[i] + nums[j] = target`. Brute force
checks all pairs, O(n²). The two-pointer trick from [file 02](02-arrays-and-strings.md) needs
sorting, which loses the original indices unless you carry them along.

### The intuition

Walk through the array once. At element `x`, the only partner that works is `target − x`, its
**complement**. If you've remembered every earlier element in a map from value to index, you
can ask "have I already seen `target − x`?" in O(1). You never look ahead, because when the
partner comes later, *it* will find *you*.

```go
func twoSum(nums []int, target int) (int, int) {
	seen := make(map[int]int) // value -> index
	for i, x := range nums {
		if j, ok := seen[target-x]; ok {
			return j, i
		}
		seen[x] = i
	}
	return -1, -1
}
```

`nums = [2, 7, 11, 15]`, `target = 9`:

```
i   x    need (9 − x)   seen before this step   found?
0   2         7          {}                       no → store 2:0
1   7         2          {2:0}                    yes → return (0, 1)
```

Check, *then* store. If you store first, `x = target/2` finds itself: `[3, …]` with target 6
would return `(0, 0)`.

**Complexity:** O(n) expected time, O(n) space. You've traded memory for time, which is the
usual price of hashing.

## 3.3 Prefix sum plus hash map

This is the pattern that sliding window can't handle
([file 02 §2.2](02-arrays-and-strings.md#when-sliding-window-fails)), and one of the most
reused ideas in the course.

### The problem

Count the contiguous subarrays whose sum is exactly `k`. **Numbers can be negative.** The brute
force is O(n²) subarrays, each summed in O(1) with prefix sums, so O(n²) overall.

### The intuition

From [file 02 §2.3](02-arrays-and-strings.md#23-prefix-sums): the sum of `nums[i..j]` is
`prefix[j+1] − prefix[i]`. So "this subarray sums to `k`" means

```
prefix[j+1] − prefix[i] = k      ⇔      prefix[i] = prefix[j+1] − k
```

Read the right-hand side as a question you ask at each position `j`: *how many earlier prefix
sums equal (current prefix − k)?* Each one marks the start of a subarray ending here with sum
`k`. That's the complement lookup from §3.2 again, but on prefix sums rather than raw values,
and *counting* matches rather than finding one.

```go
func subarraySum(nums []int, k int) int {
	count := map[int]int{0: 1} // prefix sum -> how many times seen; the empty prefix is 0
	sum, total := 0, 0
	for _, x := range nums {
		sum += x
		total += count[sum-k] // every earlier prefix equal to sum-k ends a subarray summing to k here
		count[sum]++
	}
	return total
}
```

<details><summary>Rust</summary>

```rust
use std::collections::HashMap;

fn subarray_sum(nums: &[i32], k: i32) -> i32 {
    let mut count: HashMap<i64, i32> = HashMap::from([(0, 1)]);
    let (mut sum, mut total) = (0i64, 0);
    for &x in nums {
        sum += x as i64;
        total += count.get(&(sum - k as i64)).copied().unwrap_or(0);
        *count.entry(sum).or_insert(0) += 1;
    }
    total
}
```

</details>

<details><summary>C++</summary>

```cpp
int subarraySum(const vector<int>& nums, int k) {
    unordered_map<long long, int> count{{0, 1}};
    long long sum = 0;
    int total = 0;
    for (int x : nums) {
        sum += x;
        auto it = count.find(sum - k);  // find, not [], so we don't insert zeros
        if (it != count.end()) total += it->second;
        ++count[sum];
    }
    return total;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function subarraySum(nums, k) {
  const count = new Map([[0, 1]]);
  let sum = 0, total = 0;
  for (const x of nums) {
    sum += x;
    total += count.get(sum - k) ?? 0;
    count.set(sum, (count.get(sum) ?? 0) + 1);
  }
  return total;
}
```

</details>

The Rust and C++ versions keep the running sum in 64 bits. With 2·10^4 elements of absolute
value up to 1,000 (LeetCode 560's limits) a 32-bit sum can't overflow, but the habit costs
nothing and the problem next door may have bigger limits.

### Worked example

`nums = [3, 4, −7, 1, 3, 3, 1, −4]`, `k = 7`:

```
x     sum   need (sum − 7)   count[need]   total   count after this step
 3     3        −4               0           0     {0:1, 3:1}
 4     7         0               1           1     {0:1, 3:1, 7:1}          [3,4]
−7     0        −7               0           1     {0:2, 3:1, 7:1}
 1     1        −6               0           1     … 1:1
 3     4        −3               0           1     … 4:1
 3     7         0               2           3     … 7:2                    [3,4,−7,1,3,3] and [1,3,3]
 1     8         1               1           4     … 8:1                    [3,3,1]
−4     4        −3               0           4     … 4:2
```

Answer: 4.

> ⚠️ **`count[0] = 1` before the loop isn't optional.** It represents the empty prefix, the
> "sum before anything". Without it, subarrays that start at index 0 are never counted: in the
> trace, `[3, 4]` and `[3, 4, −7, 1, 3, 3]` would both be missed and you'd return 2. The same
> seed appears in every prefix-plus-map problem, sometimes as `{0: -1}` when you store the
> *index* of the first occurrence rather than a count.

**Complexity:** O(n) expected time, O(n) space.

**Variants**, all the same idea with a different key:

| Question | Store per prefix | Lookup |
|---|---|---|
| How many subarrays sum to k? | count of each prefix sum | `sum − k` |
| Longest subarray summing to k? | **first** index of each prefix sum | `sum − k`, length `j − first` |
| Any subarray with sum divisible by k? | first index of each `sum mod k` | the same remainder again |
| Longest subarray with equal 0s and 1s? | treat 0 as −1, first index of each sum | the same sum again |

## 3.4 Choosing the key: grouping

### The problem

Group a list of words into anagrams: `["eat","tea","tan","ate","nat","bat"]` →
`[[eat,tea,ate],[tan,nat],[bat]]`.

### The intuition

Anagrams are "the same up to order". So find a **canonical form**, a key that's identical for
everything in a group and different across groups, and bucket by it. Two choices:

| Key | Cost per word of length m | Notes |
|---|---|---|
| The word with its letters sorted | O(m log m) | Simple, works for any alphabet |
| A count of each letter, `[26]` | O(m) | Faster; relies on a fixed alphabet |

```go
func groupAnagrams(words []string) [][]string {
	groups := make(map[[26]byte][]string) // arrays are comparable, so usable as keys
	for _, w := range words {
		var key [26]byte
		for i := 0; i < len(w); i++ {
			key[w[i]-'a']++
		}
		groups[key] = append(groups[key], w)
	}
	out := make([][]string, 0, len(groups))
	for _, g := range groups {
		out = append(out, g)
	}
	return out
}
```

(`byte` counts are fine when words are at most 100 letters, as in LeetCode 49. Longer words
would overflow a count and need `[26]int`.)

What can be a key differs sharply by language, and it's a classic place to lose ten minutes:

| Language | Composite key that works | The trap |
|---|---|---|
| Go | arrays (`[26]byte`) and structs of comparable fields | **slices can't be map keys**: `map[[]int]…` doesn't compile |
| Rust | `[u8; 26]`, tuples, `Vec<T>`: anything that is `Hash + Eq` | none worth worrying about |
| C++ | `std::string` | `unordered_map<pair<int,int>, …>` and `unordered_map<vector<int>, …>` don't compile: there's no default `std::hash` for them. Use `std::map` (ordered, only needs `<`), encode to a string, or write a hash |
| JavaScript | strings and numbers | `Map` compares objects **by identity**: `m.set([1,2], x); m.get([1,2])` returns `undefined`, because the second `[1,2]` is a different array. Use `key.join(",")` |

> ⚠️ **Go randomises map iteration order.** The spec says it "is not specified and is not
> guaranteed to be the same from one iteration to the next". The groups from `groupAnagrams`
> come out in a different order each run. LeetCode accepts any order here, but if a problem
> wants a specific order (say, first appearance), record that order separately; don't hope the
> map preserves it. JavaScript's `Map` *does* iterate in insertion order, which is the one
> place where JS code that "worked" can break when you port it to Go.

## 3.5 Sets: membership as a building block

A set is a map with no values. Its job is O(1) "is x present?". The interesting problems use
that to *avoid* work, not just to look things up.

**Longest consecutive sequence** (in O(n), unsorted): put everything in a set. Sorting would
make it easy but costs O(n log n). Instead, only start counting from a number `x` whose
predecessor `x − 1` is *absent*, because only then is `x` the start of a run. Every other
number is counted exactly once, as part of the run it belongs to.

```go
func longestConsecutive(nums []int) int {
	set := make(map[int]struct{}, len(nums))
	for _, x := range nums {
		set[x] = struct{}{}
	}
	best := 0
	for x := range set {
		if _, hasPrev := set[x-1]; hasPrev {
			continue // x is not the start of a run; its run gets counted from the start
		}
		length := 1
		for {
			if _, ok := set[x+length]; !ok {
				break
			}
			length++
		}
		best = max(best, length)
	}
	return best
}
```

It looks quadratic (a loop inside a loop), and it isn't. The inner loop runs only from run
starts, and the runs are disjoint, so across the whole function the inner loop runs at most n
times in total. That's the same amortised argument as the sliding window. Iterate over the
*set*, not the input: with many duplicates of a run start, iterating the input would redo
that run once per duplicate.

## 3.6 How to recognise hashing

| You see… | Store… |
|---|---|
| "Find two elements that…" (unsorted, need indices) | value → index; look up the complement |
| "Count subarrays with sum / xor / remainder…" | prefix value → count |
| "Longest subarray with…" (a prefix condition) | prefix value → first index |
| "Group things that are equivalent" | canonical form → list |
| "Have I seen this state before?" (cycle, duplicate) | set of states |
| "Most frequent", "first unique" | value → count, then one more pass |

And when not to: if you need **order** (smallest key, keys in a range, the next key above x),
a hash map can't help. Sort, use a heap ([file 09](09-heaps.md)), or use an ordered map
(`BTreeMap` / `std::map`; Go and JS have none built in).

## Asked in interviews

| Problem | What it tests |
|---|---|
| [1. Two Sum](https://leetcode.com/problems/two-sum/) | Complement lookup; check-then-store order |
| [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | The set as a seen-before detector; the warm-up |
| [242. Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Counting, and the fixed-alphabet shortcut |
| [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Designing a canonical key, and what your language allows as a key |
| [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | O(n) with a set; justifying why the nested loop is linear |
| [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/) | Prefix sum + map, with negatives; the `{0: 1}` seed |
| [523. Continuous Subarray Sum](https://leetcode.com/problems/continuous-subarray-sum/) | The remainder variant, storing first index rather than count |

---

## Check yourself

1. In Two Sum, what goes wrong if you insert `x` into the map *before* checking for its
   complement? Give an input that exposes it.
2. Why does `subarraySum` need the map to be seeded with `{0: 1}`? Describe exactly which
   subarrays are missed without it.
3. You want the **longest** subarray summing to `k`, not the count. What do you store per prefix
   sum, and why must you *not* overwrite it when the same prefix sum appears again?
4. In `longestConsecutive`, the outer loop iterates over the set rather than over `nums`.
   Construct an input where iterating over `nums` instead would make the function quadratic.
5. You port a working JavaScript solution to Go. It groups items with a `Map`, then returns
   the groups in iteration order, and LeetCode's checker requires first-appearance order. The
   Go version fails intermittently. Explain the mechanism and the fix.
6. A colleague keys an `unordered_map` by `std::pair<int,int>` and gets a wall of template
   errors. Give two fixes, and say what each costs.

(Answers in [`15-exercises.md`](15-exercises.md).)
