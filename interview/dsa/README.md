# Data Structures & Algorithms, by Pattern — A Course

Assumes [`interview/01`](../01-how-the-loop-works.md) — the shape of the interview loop.

A course on the data structures and algorithms that coding interviews test, organised by
**pattern**: the reusable reason why a problem can be solved faster than brute force. Not by
problem, and not by company.

**This is reference learning material.** It teaches the ideas so that you can solve problems
you haven't seen. Interview questions are a by-product: each chapter lists classic problems that
use its pattern, but the chapter is about the pattern. Nothing here is specific to any employer
or project.

It's written for a working backend engineer (roughly one to four years in, comfortable with Go,
SQL and queues) who studied this material once, hasn't needed most of it since, and wants it back
properly rather than as a list of memorised solutions.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Every pattern opens with the problem it solves and the intuition, before any code.
- Every chapter has a worked example traced step by step, and ends with **Check yourself**
  questions. Answers are in [`15-exercises.md`](15-exercises.md).
- Where something is a rule of thumb, a measurement from one machine, or a claim I couldn't
  verify, the sentence says so.

## Languages

Code is in **Go** first, the language this course assumes you use daily. For each chapter's core
template there are also **Rust**, **C++17** and **JavaScript** versions in collapsible blocks. Every
one of those templates was compiled and run against test cases before going in (Go 1.24, Rust 1.96,
Apple clang 17 with `-std=c++17` under AddressSanitizer and UBSan, Node 24).

The patterns themselves are language-independent. What differs is which container you reach for
and which defaults bite: Rust and C++ heaps are max-heaps, JavaScript has no heap or deque at all,
integer overflow means four different things in four languages. The container cheat sheet and the
table of traps are in [file 01 §1.7](01-foundations.md#17-languages-what-this-course-uses-and-what-you-should),
along with what I found about which online-assessment platforms support Rust. The short version:
**write in four languages to learn, interview in the one you're fluent in.**

## Reference sources

The primary sources behind the claims in this course. Language documentation is cited inline
where it's used.

| Source | What it's used for | Link |
|---|---|---|
| Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms* (CLRS), 4th ed., MIT Press 2022 | Asymptotic notation (ch. 3), heaps and heapify (ch. 6), amortised analysis (ch. 16) | [MIT Press](https://mitpressbookstore.mit.edu/book/9780262046305) |
| Laaksonen, *Competitive Programmer's Handbook* | The input-size → complexity table (§2.3) | [PDF, free](https://cses.fi/book/book.pdf) |
| Kleinberg, Tardos, *Algorithm Design*, Pearson 2005 | Greedy proofs: the exchange argument (ch. 4) | [Companion lecture slides (Kevin Wayne, Princeton)](https://www.cs.princeton.edu/~wayne/kleinberg-tardos/) |
| Bentley, *Programming Pearls*, 2nd ed., column 4 | How often professional programmers get binary search wrong | (book) |
| Knuth, "Big Omicron and Big Omega and Big Theta", SIGACT News 1976 | How computer science uses O, Ω, Θ | [doi:10.1145/1008328.1008329](https://doi.org/10.1145/1008328.1008329) |
| Hoare, "Algorithm 65: Find", CACM 1961 | Quickselect | [doi:10.1145/366622.366647](https://doi.org/10.1145/366622.366647) |
| Dijkstra, "A note on two problems in connexion with graphs", Numerische Mathematik 1959 | Dijkstra's algorithm | [doi:10.1007/BF01386390](https://doi.org/10.1007/BF01386390) |
| Kahn, "Topological sorting of large networks", CACM 1962 | Kahn's algorithm | [doi:10.1145/368996.369025](https://doi.org/10.1145/368996.369025) |
| Tarjan, "Efficiency of a good but not linear set union algorithm", JACM 1975 | Union-find's inverse-Ackermann bound | [doi:10.1145/321879.321884](https://doi.org/10.1145/321879.321884) |
| Tarjan, van Leeuwen, "Worst-case analysis of set union algorithms", JACM 1984 | Path halving achieves the same bound | [doi:10.1145/62.2160](https://doi.org/10.1145/62.2160) |
| Fredman, Tarjan, "Fibonacci heaps and their uses in improved network optimization algorithms", JACM 1987 | Dijkstra in O(E + V log V) | [doi:10.1145/28869.28874](https://doi.org/10.1145/28869.28874) |
| Bellman, *Dynamic Programming*, Princeton University Press 1957 | The origin of DP | (book) |
| Bloch, "Nearly all binary searches and mergesorts are broken", Google Research blog 2006 | The `(lo + hi) / 2` overflow | [blog post](https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/) |
| METR, "Measuring the impact of early-2025 AI on experienced open-source developer productivity", 2025 | The AI-and-DSA paragraph in file 01 | [blog post](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) |
| Canva Engineering, "Yes, you can use AI in our interviews", 2025 | The same paragraph | [blog post](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/) |
| LeetCode | Every "Asked in interviews" link; each slug was checked against LeetCode's API on 2026-10-08 | [leetcode.com](https://leetcode.com/) |

**About "Asked in interviews".** Which problems are "frequently asked" is anecdotal: it comes from
candidate reports on forums and prep sites, not from any measured data, and companies don't publish
their question banks. The lists are classic problems that exercise each pattern well. Don't read
them as predictions. One linked problem (253, Meeting Rooms II) needs LeetCode Premium and is marked.

## Reading order

### Part 1 — Foundations (prerequisite for everything)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | Say what a round is scoring, use Big-O and amortised cost precisely, turn constraints into a target complexity, work a problem out loud in six steps, and avoid your language's traps |

### Part 2 — Linear structures

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Arrays and strings](02-arrays-and-strings.md) | Replace O(n²) pair and subarray scans with two pointers, sliding windows and prefix sums, and know when a window *can't* work |
| 3 | [Hashing](03-hashing.md) | Trade memory for time with complement lookups, prefix-sum maps and canonical keys |
| 4 | [Linked lists](04-linked-lists.md) | Reverse, splice and cycle-detect without losing nodes; build an LRU cache |
| 5 | [Stacks and queues](05-stacks-and-queues.md) | Use a monotonic stack for "next greater" and a monotonic deque for window maxima |

### Part 3 — Searching and recursion

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Binary search](06-binary-search.md) | Write binary search with no off-by-ones from one invariant, and binary-search an answer space |
| 7 | [Recursion and backtracking](07-recursion-and-backtracking.md) | Trust recursion, enumerate subsets and permutations, and prune |

### Part 4 — Trees, heaps and graphs

| # | File | After this you can… |
|---|------|---------------------|
| 8 | [Trees and BSTs](08-trees.md) | Solve tree problems by asking the children, separate what you return from what you record, and validate a BST correctly |
| 9 | [Heaps and top-k](09-heaps.md) | Keep the k best of a stream, a running median, and a k-way merge, in your language's heap |
| 10 | [**Graphs**](10-graphs.md) | ⭐ Choose between BFS, DFS, topological sort, Dijkstra and union-find, and say why |

### Part 5 — Optimisation

| # | File | After this you can… |
|---|------|---------------------|
| 11 | [Greedy and intervals](11-greedy-and-intervals.md) | Solve interval problems with one sort, and prove (or break) a greedy idea |
| 12 | [**Dynamic programming**](12-dynamic-programming.md) | ⭐ Design a DP state from scratch, write it top-down or bottom-up, and know when the loop direction matters |

### Part 6 — Shorter topics

| # | File | After this you can… |
|---|------|---------------------|
| 13 | [Tries and bit manipulation](13-tries-and-bits.md) | Answer prefix queries with a trie, and use bits as small sets without your language's precedence and width traps |

### Reference

| # | File | |
|---|------|--|
| 14 | [Glossary](14-glossary.md) | Look things up |
| 15 | [Exercises & answers](15-exercises.md) | Every "Check yourself" question with a worked answer, things to try, and questions worth asking me |

## If you're short on time

- **30 minutes, interview tomorrow:** file 01 §1.6 (the method) and §1.7 (your language's traps),
  then the "How to recognise" table at the end of each chapter.
- **One week:** file 01, then 02, 03, 06, 08, 10, 12, in that order. Do every Check yourself
  question before reading its answer.
- **Online assessment on HackerRank or HackerEarth tomorrow:** file 01 §1.5 (reading constraints)
  and §1.7 (the platform's language versions, which may be old), then the "How to recognise" table
  of every chapter.
- **DP is your weak spot:** file 07 (recursion first), then 12, then the DP questions in file 15.

## The one-paragraph summary of everything

Almost every efficient algorithm here is a brute force with its repeated work removed, and the
patterns are names for *which* repeated work. Two pointers and sliding windows stop you re-scanning
overlapping ranges; prefix sums stop you re-adding them; hash maps stop you re-searching for a
partner; monotonic stacks keep only the elements still waiting for an answer; binary search throws
away half the candidates whenever a yes/no question is monotone, including over a range of possible
*answers*; heaps keep just enough order to hand you the best item in O(log n); BFS, Dijkstra and
union-find avoid re-exploring a graph; greedy commits to one choice when an exchange argument proves
it safe; and dynamic programming remembers the answer to every subproblem it will be asked again,
once you've found a *state* small enough to remember. The constraints tell you how much waste you
can afford (n ≤ 10^5 rules out quadratic), and the interview rewards showing that reasoning out
loud, then testing your own code, more than arriving silently at the optimum. Learn the patterns in
any language; interview in the one you know best, and know its defaults cold, because Rust and C++
heaps are max-heaps, JavaScript has no heap at all, and Go wraps on overflow without a word.

## How to use me

Ask anything, including:

- "Give me a problem I haven't seen, and make me do the six steps out loud."
- "Which pattern is this, and what in the problem statement should have told me?"
- "Explain monotonic stacks again, but simpler."
- "Trace this code for me on `[2, 7, 9, 3, 1]`, line by line."
- "What's the real complexity of my solution? Here it is…"
- "Show me this chapter's template in Rust, and tell me what the borrow checker is protecting me from."
- "Is my mental model of Dijkstra right? Here's what I think happens…"
- "I got this wrong in a mock. Diagnose which step of the method I skipped."

I'll add files here if a topic earns its own page; segment trees and Fenwick trees are the obvious
next candidates (see the answer to question 5 for file 02 in [file 15](15-exercises.md)).
