# 4. Linked lists

Inserting into the middle of an array means shifting everything after it: O(n). A linked list
makes that O(1), provided you already hold the node next to where the insertion goes. In
everyday backend code you'll rarely choose one, because arrays are faster in practice. In
interviews they come up all the time, for a different reason: they test whether you can
change pointers in the right order without losing half the structure. Most of this chapter is
about that bookkeeping, plus two ideas (the dummy head and fast/slow pointers) that make it
manageable.

## 4.1 The model

A **singly linked list** is a chain of nodes, each holding a value and a pointer to the next
node. The last node points to nothing (`nil`). You hold the list by holding its first node,
the **head**.

```
head
 │
 ▼
┌───┬───┐   ┌───┬───┐   ┌───┬───┐
│ 1 │ ●─┼──►│ 2 │ ●─┼──►│ 3 │ ╳ │
└───┴───┘   └───┴───┘   └───┴───┘
```

```go
type ListNode struct {
	Val  int
	Next *ListNode
}
```

| Operation | Array / slice | Singly linked list |
|---|---|---|
| Read the i-th element | O(1) | O(i): walk from the head |
| Insert/delete at the front | O(n): shift everything | O(1) |
| Insert/delete after a node you hold | O(n) | O(1) |
| Find a value | O(n) | O(n) |
| Memory layout | contiguous: cache-friendly | scattered: each step can miss the cache |

That last row is why arrays usually win in real code, even for operations where the list has
the better Big-O ([file 01 §1.3](01-foundations.md#what-big-o-hides)). A **doubly linked list**
adds a `Prev` pointer, which allows O(1) deletion of a node you hold. §4.5 uses one.

> **Teacher's aside.** Linked-list problems aren't hard because the ideas are deep. They're
> hard because every bug is an **order-of-assignment** bug: you overwrite `cur.Next` before
> saving it, and the rest of the list is gone. You can't see that in your head reliably, and
> nobody can. So draw it. Three boxes, arrows, and then do each assignment on paper, crossing
> out the old arrow. Interviewers expect you to draw; it reads as care rather than weakness.
> The candidates who skip drawing are the ones debugging a lost tail at minute 35.

## 4.2 Reversal: the core move

### The problem

Reverse a singly linked list in place. It's a warm-up on its own, and it's also a step inside
harder problems: reorder a list, check whether a list is a palindrome, reverse in groups of k.

### The intuition

Walk down the list turning each arrow round. The catch is that once you turn `cur.Next` to
point backwards, you've lost your way forwards. So keep three names: `prev` (the already
reversed part), `cur` (the node being flipped), and `next` (the rest, saved before you cut the
link). Each step is the same four assignments, always in the same order.

```go
func reverseList(head *ListNode) *ListNode {
	var prev *ListNode
	cur := head
	for cur != nil {
		next := cur.Next // 1. remember the rest before we cut the link
		cur.Next = prev  // 2. point this node backwards
		prev = cur       // 3. advance prev
		cur = next       // 4. advance cur
	}
	return prev
}
```

<details><summary>Rust</summary>

```rust
pub struct ListNode {
    pub val: i32,
    pub next: Option<Box<ListNode>>,
}

fn reverse_list(mut head: Option<Box<ListNode>>) -> Option<Box<ListNode>> {
    let mut prev = None;
    while let Some(mut node) = head {
        head = node.next.take(); // detach the rest
        node.next = prev; // point backwards
        prev = Some(node);
    }
    prev
}
```

</details>

<details><summary>C++</summary>

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int v, ListNode* n = nullptr) : val(v), next(n) {}
};

ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
        ListNode* next = head->next;
        head->next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
```

</details>

<details><summary>JavaScript</summary>

```js
function reverseList(head) {
  let prev = null;
  while (head) {
    const next = head.next;
    head.next = prev;
    prev = head;
    head = next;
  }
  return prev;
}
```

</details>

The Rust version is worth reading even if you never write Rust. `Option<Box<ListNode>>` means
each node is *owned* by the one before it, so you can't just copy a pointer. `take()` moves
the rest of the list out of `node.next`, leaving `None` behind. That's step 1 of the Go version
made explicit: the compiler won't let you lose the tail by accident. (It also means a list
with a cycle can't be built from `Box` at all, since nothing can have two owners. The
classic tour of this is
[*Learning Rust With Entirely Too Many Linked Lists*](https://rust-unofficial.github.io/too-many-lists/).)

### Worked example

`1 → 2 → 3`:

```
          before step                 after the four assignments
step   prev   cur   rest           reversed part      still to do
 1     nil     1    2→3            1→nil              2→3
 2      1      2    3              2→1→nil            3
 3      2      3    nil            3→2→1→nil          (none)
cur is nil → return prev = 3
```

**Complexity:** O(n) time, O(1) space. A recursive version is shorter but uses O(n) stack,
which for a 10^5-node list is a real risk in some languages
([file 01 §1.4](01-foundations.md#space-complexity)).

> ⚠️ **Swap steps 1 and 2 and you lose the list.** `cur.Next = prev` first, then
> `next := cur.Next`, and `next` is now `prev`, which on the first iteration is `nil`. The
> loop ends after one node and returns a one-element list, with no error. The rest of the list
> is unreachable. The fix is mechanical: the first line inside the loop always saves
> `cur.Next`.

## 4.3 The dummy head

### The problem

"Remove the n-th node from the end." Removing node `x` means changing the `Next` of the node
*before* `x`. But if `x` is the head, there's no node before it, so the code needs a special
case, and special cases are where bugs live.

### The idea

Create a throwaway node, the **dummy** (or sentinel), whose `Next` is the head. Now every real
node, including the head, has a predecessor. Do the work, then return `dummy.Next`, which is
the head of the result whether or not the original head survived.

Combine that with a **gap**: send a `fast` pointer n + 1 steps ahead of `slow`, then move both
until `fast` falls off the end. `slow` is then exactly one node before the one to remove.

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
	dummy := &ListNode{Next: head}
	fast, slow := dummy, dummy
	for i := 0; i <= n; i++ { // open a gap of n+1 links
		fast = fast.Next
	}
	for fast != nil {
		fast, slow = fast.Next, slow.Next
	}
	slow.Next = slow.Next.Next // slow sits just before the victim
	return dummy.Next
}
```

Remove the 2nd from the end of `1 → 2 → 3 → 4 → 5`:

```
after the gap loop:   D  1  2  3  4  5  nil
                      s        f              (f is n+1 = 3 links ahead)
after the walk:       D  1  2  3  4  5  nil
                               s        f
slow.Next = slow.Next.Next  →  1 → 2 → 3 → 5
```

With `[1]` and `n = 1`, `slow` stays on the dummy and the head itself is removed; no special
case needed.

## 4.4 Fast and slow pointers

Two pointers moving at different speeds along the same list find things without extra
memory.

### The middle

Move `slow` one step and `fast` two steps at a time. When `fast` reaches the end, `slow` has
gone half as far: it's at the middle. One pass, O(1) space.

### Cycle detection (Floyd's tortoise and hare)

**The problem:** does the list loop back on itself, and if so, where does the loop start?
A set of visited nodes answers both in O(n) space. Can you do it in O(1)?

**The intuition:** two runners on a circular track, one twice as fast. If the track is a
circle, the fast one eventually laps the slow one and they meet. If it's a straight road, the
fast one reaches the end. Once both are inside the loop, the gap between them closes by one
node per step, so they can't jump past each other.

**Finding the entry** needs one more observation. Let `μ` be the number of steps from the head
to the loop's entry, `λ` the loop's length, and `a` how far into the loop they meet. When they
meet, slow has walked `μ + a` and fast has walked twice that. Fast's extra distance is whole
laps:

```
2(μ + a) = μ + a + kλ      ⇒      μ + a = kλ      ⇒      μ = kλ − a
```

So walking `μ` steps from the meeting point (which is `a` steps into the loop) lands at
`a + μ = kλ`, a whole number of laps: the entry. And `μ` is also the distance from the head
to the entry. Start one pointer at the head and one at the meeting point, move both one step at
a time, and they meet at the entry.

```go
func detectCycle(head *ListNode) *ListNode {
	slow, fast := head, head
	for fast != nil && fast.Next != nil {
		slow, fast = slow.Next, fast.Next.Next
		if slow == fast { // they met inside the cycle
			p := head
			for p != slow { // phase 2: walk from head and from the meeting point at the same speed
				p, slow = p.Next, slow.Next
			}
			return p
		}
	}
	return nil
}
```

**Complexity:** O(n) time, O(1) space.

A note on the name, since you may be asked: Knuth credited the algorithm to Robert Floyd in
1969 without a citation, and it doesn't appear in Floyd's published work, so it may be folklore
rather than one person's invention
([Wikipedia, *Cycle detection*](https://en.wikipedia.org/wiki/Cycle_detection), which gives the
history). Everyone calls it Floyd's anyway.

> ⚠️ **Check `fast != nil && fast.Next != nil`, in that order.** `fast.Next.Next` needs both.
> Reversing the order dereferences a nil pointer on even-length lists. That's a panic in Go and
> undefined behaviour in C++.

The same idea solves a famous array problem: **Find the Duplicate Number** treats each value as
a pointer to an index (`i → nums[i]`), which makes a linked structure whose cycle entry is the
duplicate.

## 4.5 Combining structures: the LRU cache

A **least recently used (LRU) cache** holds at most `capacity` items, and when it's full,
evicts whichever was used longest ago. `Get` and `Put` must both be O(1). It's one of the most
frequently reported design-flavoured coding questions.

Neither structure alone works:

| Structure | Lookup by key | Track recency / evict oldest |
|---|---|---|
| Hash map | O(1) | can't: no order |
| Doubly linked list in recency order | O(n) | O(1): move to front, drop from back |
| **Both: map from key → list node** | **O(1)** | **O(1)** |

The map gives you the node; holding the node is exactly the precondition for O(1) linked-list
surgery (§4.1). Go's standard library has a doubly linked list in `container/list`, which makes
this short:

```go
import "container/list"

type entry struct{ key, val int }

type LRUCache struct {
	cap   int
	order *list.List            // front = most recently used
	index map[int]*list.Element // key -> its node in order
}

func NewLRUCache(capacity int) LRUCache {
	return LRUCache{capacity, list.New(), make(map[int]*list.Element)}
}

func (c *LRUCache) Get(key int) int {
	el, ok := c.index[key]
	if !ok {
		return -1
	}
	c.order.MoveToFront(el) // O(1): we hold the node, so no search
	return el.Value.(*entry).val
}

func (c *LRUCache) Put(key, val int) {
	if el, ok := c.index[key]; ok {
		el.Value.(*entry).val = val
		c.order.MoveToFront(el)
		return
	}
	c.index[key] = c.order.PushFront(&entry{key, val})
	if c.order.Len() > c.cap {
		oldest := c.order.Back()
		c.order.Remove(oldest)
		delete(c.index, oldest.Value.(*entry).key) // why the node stores its key
	}
}
```

Interviewers sometimes ask you to write the doubly linked list yourself instead. If so, use
**two** dummies, a head and a tail, so that inserting and removing never meet a `nil`
neighbour. In other languages: Java's `LinkedHashMap` does this for you, as does Python's
`OrderedDict`; Rust and C++ usually mean writing the list by hand (C++ can use `std::list` and
store iterators in the map, since `std::list` iterators stay valid when other elements are
removed).

## 4.6 How to recognise these

| You see… | Think… |
|---|---|
| "Reverse", "reorder", "palindrome list", "in groups of k" | the four-step reversal, often after finding the middle |
| The head might be removed or replaced | a dummy head |
| "n-th from the end", "middle", one pass, O(1) space | two pointers with a gap or different speeds |
| "Is there a cycle / where does it start", O(1) space | Floyd |
| O(1) lookup **and** O(1) ordering or eviction | hash map pointing into a doubly linked list |
| "Merge sorted lists" | a dummy head and a tail pointer; with k lists, a heap ([file 09](09-heaps.md)) |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | The four assignments, in order |
| [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | The dummy head plus a tail pointer |
| [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) | Gap pointers, and removing the head without a special case |
| [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Fast/slow, and the nil-check order |
| [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/) | Phase 2: can you explain *why* it finds the entry? |
| [143. Reorder List](https://leetcode.com/problems/reorder-list/) | Three moves composed: middle, reverse the second half, interleave |
| [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | Combining a map and a doubly linked list; usually asked as a design question |
| [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/) | Seeing a linked list hidden in an array |

---

## Check yourself

1. In `reverseList`, what does the function return for an empty list, and why does that fall
   out of the loop without a special case?
2. You remove the dummy from `removeNthFromEnd` and start both pointers at `head`. Which input
   now crashes or gives a wrong answer, and why?
3. In Floyd's algorithm, why can't the fast pointer "jump over" the slow one inside the cycle
   and keep missing it forever?
4. For a list with tail length μ = 3 and cycle length λ = 4, simulate phase 1 and find the
   meeting point. Then check that phase 2 lands on the entry.
5. In the LRU cache, why does each list node store its *key* as well as its value? What exactly
   would you be unable to do in O(1) without it?
6. A teammate proposes an LRU cache built from a hash map plus a min-heap keyed by "last used
   time". What are the costs of `Get` and `Put`, and what does that design make awkward that the
   linked-list version does not?

(Answers in [`15-exercises.md`](15-exercises.md).)
