# 13. Tries and bit manipulation

Two smaller topics that each come up often enough to need their own tools. A **trie** answers
questions about *prefixes* of strings, which a hash set can't do without scanning every word.
**Bit manipulation** treats an integer as a row of 64 switches, which turns some set operations
into single CPU instructions and makes a few puzzle problems trivial. Neither is deep, and both
have traps that cost time if you haven't seen them.

## 13.1 Tries

### The problem

You have 10^5 words and a stream of queries: "is there any word starting with `app`?" A hash set
answers "is `apple` a word?" in O(1), but for a prefix it has nothing better than checking every
word: O(n × L) per query.

### The intuition

Think of how your phone's contacts list narrows as you type. Type `A` and you're among the A
names; `An` narrows it further; the prefix you've typed *is* a location in the list. A **trie**
(from re*trie*val; usually pronounced "try") builds that structure explicitly: a tree where each
edge is a character and each node represents the prefix spelled by the path from the root. Words
that share a prefix share those nodes. A flag on a node marks "a word ends here".

```
insert: app, apple, apt, bat

            (root)
           /      \
          a        b
          |        |
          p        a
         / \       |
        p●  t●     t●
        |
        l
        |
        e●          ● = a word ends here
```

`app` is a word and also a prefix of `apple`, which is why "ends here" has to be a flag and can't
be inferred from "has no children".

```go
type Trie struct {
	children [26]*Trie
	isWord   bool
}

func (t *Trie) Insert(word string) {
	node := t
	for i := 0; i < len(word); i++ {
		c := word[i] - 'a'
		if node.children[c] == nil {
			node.children[c] = &Trie{}
		}
		node = node.children[c]
	}
	node.isWord = true
}

func (t *Trie) walk(s string) *Trie {
	node := t
	for i := 0; i < len(s) && node != nil; i++ {
		node = node.children[s[i]-'a']
	}
	return node
}

func (t *Trie) Search(word string) bool  { n := t.walk(word); return n != nil && n.isWord }
func (t *Trie) StartsWith(p string) bool { return t.walk(p) != nil }
```

<details><summary>Rust</summary>

```rust
#[derive(Default)]
struct Trie {
    children: [Option<Box<Trie>>; 26],
    is_word: bool,
}

impl Trie {
    fn insert(&mut self, word: &str) {
        let mut node = self;
        for b in word.bytes() {
            node = node.children[(b - b'a') as usize].get_or_insert_with(Default::default);
        }
        node.is_word = true;
    }
    fn walk(&self, s: &str) -> Option<&Trie> {
        let mut node = self;
        for b in s.bytes() {
            node = node.children[(b - b'a') as usize].as_deref()?;
        }
        Some(node)
    }
    fn search(&self, word: &str) -> bool {
        self.walk(word).map_or(false, |n| n.is_word)
    }
    fn starts_with(&self, prefix: &str) -> bool {
        self.walk(prefix).is_some()
    }
}
```

A trie is a tree with a single owner per node, so `Box` fits naturally; this is one structure
that's no harder in Rust than elsewhere.

</details>

<details><summary>C++</summary>

```cpp
struct Trie {
    array<unique_ptr<Trie>, 26> children;
    bool isWord = false;

    void insert(const string& word) {
        Trie* node = this;
        for (char ch : word) {
            auto& child = node->children[ch - 'a'];
            if (!child) child = make_unique<Trie>();
            node = child.get();
        }
        node->isWord = true;
    }
    const Trie* walk(const string& s) const {
        const Trie* node = this;
        for (char ch : s) {
            node = node->children[ch - 'a'].get();
            if (!node) return nullptr;
        }
        return node;
    }
    bool search(const string& w) const { auto n = walk(w); return n && n->isWord; }
    bool startsWith(const string& p) const { return walk(p) != nullptr; }
};
```

</details>

<details><summary>JavaScript</summary>

```js
class Trie {
  constructor() {
    this.children = new Map();
    this.isWord = false;
  }
  insert(word) {
    let node = this;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new Trie());
      node = node.children.get(ch);
    }
    node.isWord = true;
  }
  walk(s) {
    let node = this;
    for (const ch of s) {
      node = node.children.get(ch);
      if (!node) return null;
    }
    return node;
  }
  search(word) { const n = this.walk(word); return n !== null && n.isWord; }
  startsWith(prefix) { return this.walk(prefix) !== null; }
}
```

The JS version uses a `Map` for children rather than a 26-slot array, so it handles any
alphabet at the cost of a hash lookup per character.

</details>

**Complexity:** insert, search and prefix check are O(L) for a string of length L, independent
of how many words are stored. Memory is the cost: up to one node per character inserted, and with
a fixed 26-slot array each node holds 26 pointers (208 bytes on a 64-bit machine) even if it uses
one.

| Children stored as | Lookup | Memory per node | Use when |
|---|---|---|---|
| fixed array `[26]` | O(1) index | 26 pointers, mostly empty | lowercase a–z, as most problems specify |
| hash map | O(1) expected | proportional to actual children | large or unknown alphabets (Unicode) |

> **Teacher's aside.** A trie earns its place when you're doing **many** prefix operations
> against a **fixed** set of words, and especially when you'd otherwise restart a search for
> every word. Word Search II is the clearest case. Searching a grid for each of 10^4 words
> separately repeats the same exploration endlessly. Put all the words in a trie and do one
> backtracking walk over the grid ([file 07](07-recursion-and-backtracking.md)), moving down the
> trie as you move across the grid. Any path whose prefix isn't in the trie is abandoned at once,
> for all words together. For a one-off "does this word exist?", a hash set is simpler and
> faster; reaching for a trie there is over-engineering.

**Wildcard search** (`.` matches any letter) is a DFS over the trie: at a `.`, try every child.
**Longest common prefix** of a word list is the path from the root while each node has exactly one
child and isn't a word end.

## 13.2 Bit manipulation

### The idea

An integer is a row of bits. Treat a 64-bit integer as a **set of up to 64 small numbers**, where
bit `i` set means "i is in the set", and the set operations become single instructions:

| Set operation | Bits | Go | Rust | C++ | JS (32-bit only) |
|---|---|---|---|---|---|
| union | OR | `a \| b` | `a \| b` | `a \| b` | `a \| b` |
| intersection | AND | `a & b` | `a & b` | `a & b` | `a & b` |
| symmetric difference | XOR | `a ^ b` | `a ^ b` | `a ^ b` | `a ^ b` |
| complement | NOT | `^a` (unary) | `!a` | `~a` | `~a` |
| difference a − b | AND NOT | `a &^ b` | `a & !b` | `a & ~b` | `a & ~b` |
| add element i | | `a \| 1<<i` | `a \| 1 << i` | `a \| (1LL << i)` | `a \| (1 << i)` |
| contains i? | | `a>>i&1 == 1` | `(a >> i) & 1 == 1` | `((a >> i) & 1) == 1` | `((a >> i) & 1) === 1` |
| count elements | popcount | `bits.OnesCount64(a)` | `a.count_ones()` | `std::popcount` (C++20), `__builtin_popcountll` | none built in |

Note the NOT column: Go spells bitwise NOT as unary `^`, Rust as `!`, and C++ and JS as `~`.

> ⚠️ **Operator precedence differs, and in C++ and JavaScript it bites.** `x & 1 == 0` looks like
> "x is even". In C++ and JavaScript, `==` binds tighter than `&`, so it parses as
> `x & (1 == 0)`, which is `x & 0`: always 0. I ran it with `x = 4` (even): C++ printed `0`, JS
> printed `0`. Go and Rust give `&` higher precedence than `==`, so the same text prints `true` in
> both. Parenthesise bitwise expressions in every language and the question never comes up.

> ⚠️ **JavaScript bitwise operators work on 32-bit signed integers.** Every operand is converted
> to a 32-bit integer first: `1 << 31` is `-2147483648`, and `2**32 | 0` is `0` (I checked both
> on Node 24). Use `>>> 0` to read a result as unsigned, and `BigInt` (`1n << 40n`) for wider
> masks. In C++, `1 << 40` overflows a 32-bit `int`; write `1LL << 40`.

### The tricks worth knowing

| Expression | Does | Why |
|---|---|---|
| `x & (x − 1)` | clears the lowest set bit | subtracting 1 flips the lowest 1 and every 0 below it |
| `x & −x` | isolates the lowest set bit | two's complement: `−x` = `~x + 1` |
| `x & (x − 1) == 0` (for x > 0) | x is a power of two | a power of two has exactly one set bit |
| `a ^ a = 0`, `a ^ 0 = a` | XOR cancels pairs | each bit flips twice and returns |
| `x >> 1`, `x << 1` | divide / multiply by 2 | for non-negative integers |

**Single number:** every element appears twice except one. XOR everything; the pairs cancel.

```go
func singleNumber(nums []int) int {
	x := 0
	for _, n := range nums {
		x ^= n // pairs cancel: a ^ a = 0, a ^ 0 = a
	}
	return x
}
```

**Counting set bits** (Kernighan's method): each `x &= x − 1` removes one set bit, so the loop runs
once per set bit, not once per bit position.

```go
func popcount(x uint32) int {
	n := 0
	for x != 0 {
		x &= x - 1 // clears the lowest set bit
		n++
	}
	return n
}
```

In real code, use `math/bits.OnesCount32`, which compiles to a hardware instruction where one
exists. In an interview, writing the loop shows you know why it works.

**Counting bits for 0..n** in O(n) is a tiny DP ([file 12](12-dynamic-programming.md)): `i` has
the same bits as `i >> 1`, plus its own lowest bit.

```go
func countBits(n int) []int {
	ans := make([]int, n+1)
	for i := 1; i <= n; i++ {
		ans[i] = ans[i>>1] + (i & 1) // i has i/2's bits plus its own last bit
	}
	return ans
}
```

### Bitmasks as subsets

For n ≤ 20, every subset of n items is a number from 0 to 2ⁿ − 1, and bit `i` says whether item
`i` is in it. That gives a loop-based alternative to backtracking for generating subsets:

```go
func subsetsByMask(nums []int) [][]int {
	n := len(nums)
	out := make([][]int, 0, 1<<n)
	for mask := 0; mask < 1<<n; mask++ { // each mask is one subset: bit i set = take nums[i]
		var s []int
		for i := range n {
			if mask>>i&1 == 1 {
				s = append(s, nums[i])
			}
		}
		out = append(out, s)
	}
	return out
}
```

It's also the basis of **bitmask DP**, where the state includes "which items have been used so
far" as a mask: travelling-salesman-style problems in O(2ⁿ · n²). That's the n ≤ 20 row of the
constraint table in [file 01 §1.5](01-foundations.md#15-from-constraints-to-a-target-complexity).

**Maximum XOR of two numbers** combines both halves of this chapter: insert each number's bits,
most significant first, into a binary trie (two children per node), then for each number walk the
trie choosing the *opposite* bit wherever possible.

## 13.3 How to recognise these

| You see… | Think… |
|---|---|
| "Starts with", autocomplete, many prefix queries | trie |
| Find many dictionary words in a grid or text at once | trie + backtracking |
| "Every element appears twice/k times except one" | XOR, or count bits per position |
| "Without using + or −", "power of two", "count 1 bits" | bit tricks |
| n ≤ 20 and you need to track which items are used | bitmask as state |
| Maximum XOR of a pair | binary trie, highest bit first |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | The structure, and "word ends here" as a flag |
| [211. Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) | Wildcards as DFS over the trie |
| [212. Word Search II](https://leetcode.com/problems/word-search-ii/) | Trie-guided backtracking; pruning finished words |
| [136. Single Number](https://leetcode.com/problems/single-number/) | XOR cancellation |
| [191. Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | `x & (x − 1)` |
| [338. Counting Bits](https://leetcode.com/problems/counting-bits/) | A bit-shaped DP |
| [268. Missing Number](https://leetcode.com/problems/missing-number/) | XOR (or sum) of indices and values |
| [421. Maximum XOR of Two Numbers in an Array](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/) | Binary trie, greedy from the top bit |

---

## Check yourself

1. Why does a trie node need an `isWord` flag? Give two words whose insertion would make
   "is a word" impossible to infer from the tree's shape alone.
2. You store 10^5 random 10-letter words in a trie with `[26]` child arrays. Estimate the number of
   nodes and the memory, and compare with a hash set of the same words.
3. Show step by step that `12 & 11` clears the lowest set bit of 12, and explain why `x & (x − 1)`
   does that for every positive x.
4. Every element of an array appears three times except one, which appears once. XOR no longer
   works. Why not, and what does work in O(n) time and O(1) space?
5. A JavaScript solution uses `mask | (1 << i)` to track which of 40 items are used. It passes small
   tests and fails large ones. What's happening, and what are two fixes?

(Answers in [`15-exercises.md`](15-exercises.md).)
