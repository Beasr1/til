# 8. Trees and binary search trees

A file system, an org chart, a JSON document, the B-tree inside a Postgres index: hierarchy is
everywhere, and a list can't represent it without losing the structure. Trees represent it
directly, and most tree problems then come down to one of two moves. Either **ask each child
for an answer and combine** (depth-first), or **sweep level by level** (breadth-first). Once you
can do both reliably, and know which value to return versus which to record, the classic tree
problems stop being separate puzzles.

## 8.1 Vocabulary

A **binary tree** is a set of nodes where each node has at most two children, `Left` and
`Right`, and every node except the **root** has exactly one parent.

```go
type TreeNode struct {
	Val         int
	Left, Right *TreeNode
}
```

```
            5            ← root, depth 0
          /   \
         1     4         ← depth 1
              / \
             3   6       ← leaves (no children), depth 2

height of the tree = 2 edges on the longest root-to-leaf path
(LeetCode's "maximum depth" counts nodes instead, so it says 3 — check which one a problem means)
```

| Term | Meaning |
|---|---|
| **Leaf** | a node with no children |
| **Depth** of a node | edges from the root down to it |
| **Height** of a tree | the largest depth; a single node has height 0 |
| **Subtree** | a node together with all its descendants |
| **Balanced** | height O(log n); informally, no branch is much longer than the others |
| **Degenerate** | every node has one child: a linked list in disguise, height n − 1 |

Height decides almost every complexity in this chapter. Balanced, it's about log₂ n (20 for a
million nodes). Degenerate, it's n.

## 8.2 Depth-first: ask your children

### The pattern

Most tree problems have this shape: *the answer at a node can be computed from the answers at
its children.* Write a recursive function, apply the leap of faith from
[file 07 §7.1](07-recursion-and-backtracking.md#71-recursion-without-the-mysticism) ("assume it
works for both children"), combine, and handle `nil`.

```go
func maxDepth(root *TreeNode) int {
	if root == nil {
		return 0
	}
	return 1 + max(maxDepth(root.Left), maxDepth(root.Right))
}
```

This is **post-order**: the children are finished before the node does its own work.
**Complexity:** O(n) time, since each node is visited once; O(h) stack.

### Two values: what you return versus what you record

Harder tree problems need the function to compute two different things, and confusing them is
the classic mistake.

**Binary Tree Maximum Path Sum:** a path is any sequence of connected nodes, which may bend
once at some node (go up the left side, through the node, down the right). Find the largest sum.

At each node, two different quantities matter:

- **The best path that bends here** may use both children's branches: `val + left + right`.
  That's a candidate for the overall answer, so you *record* it.
- **What you return to your parent** must be a path the parent can extend. A path that already
  bends here can't bend again at the parent, so you may only return *one* side:
  `val + max(left, right)`.

```go
func maxPathSum(root *TreeNode) int {
	best := math.MinInt
	var gain func(n *TreeNode) int // best downward path starting at n
	gain = func(n *TreeNode) int {
		if n == nil {
			return 0
		}
		l := max(gain(n.Left), 0) // a negative branch is worse than no branch
		r := max(gain(n.Right), 0)
		best = max(best, n.Val+l+r) // path that bends at n: may use both sides
		return n.Val + max(l, r)    // path handed to the parent: one side only
	}
	gain(root)
	return best
}
```

Trace on the tree `-10 → (9, 20 → (15, 7))`:

```
node   l (clamped)   r (clamped)   bends here: val+l+r   best   returns val+max(l,r)
 9         0             0               9                 9         9
15         0             0              15                15        15
 7         0             0               7                15         7
20        15             7              42                42        35
-10        9            35              34                42        25
answer: 42   (15 → 20 → 7)
```

> **Teacher's aside.** Whenever a tree problem asks about a path, a diameter, or anything that
> can pass *through* a node rather than just *down* from it, the function needs to return one
> thing and record another. Ask yourself: *"what does my parent need from me?"* (that's the
> return value) and *"what's the best answer that has its top at me?"* (that goes in the
> recorded best). Diameter of a binary tree is the same split: return `1 + max(l, r)` (height),
> record `l + r` (the path bending here). Mixing the two is the single most common wrong answer
> to these problems, because it passes the small examples where the best path happens to go
> straight down.

### Lowest common ancestor

The lowest node that has both `p` and `q` in its subtree. Ask each child "did you find p or q?".
If both children say yes, *this* node is where the paths split.

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
	if root == nil || root == p || root == q {
		return root
	}
	l := lowestCommonAncestor(root.Left, p, q)
	r := lowestCommonAncestor(root.Right, p, q)
	if l != nil && r != nil {
		return root // p and q were found on different sides: root is the split point
	}
	if l != nil {
		return l
	}
	return r
}
```

Returning `root` as soon as it equals `p` handles the case where `p` is an ancestor of `q`
without searching below it. It relies on the problem's guarantee that both nodes exist in the
tree; if they might not, you have to keep searching to confirm the other one.

## 8.3 The three depth-first orders

All three visit every node; they differ in *when* the node does its own work relative to its
children. (The example tree from §8.1 isn't a BST, which is why its in-order below isn't sorted.)

| Order | Node visited | On the tree above | Typical use |
|---|---|---|---|
| **Pre-order** | before its children | 5, 1, 4, 3, 6 | copying or serialising a tree (the root comes first, so you can rebuild top-down) |
| **In-order** | between left and right | 1, 5, 3, 4, 6 | on a BST, visits values in **sorted order** |
| **Post-order** | after its children | 1, 3, 6, 4, 5 | anything that needs the children's answers first (depth, path sums, deleting) |

Two famous reconstruction facts: pre-order plus in-order (with distinct values) determines a
binary tree uniquely, because pre-order tells you the root and in-order tells you which values
lie left and right of it. Pre-order alone doesn't, unless you also record where the `nil`s are,
which is how serialisation works.

## 8.4 Breadth-first: level by level

When the question is about **levels** ("the rightmost node at each depth", "average of each
level", "minimum depth"), use a queue. Process the tree one level at a time: everything in the
queue at the start of a round is exactly one level.

```go
func levelOrder(root *TreeNode) [][]int {
	var out [][]int
	if root == nil {
		return out
	}
	queue := []*TreeNode{root}
	for len(queue) > 0 {
		level := make([]int, 0, len(queue))
		var next []*TreeNode
		for _, n := range queue { // everything in queue is exactly one level
			level = append(level, n.Val)
			if n.Left != nil {
				next = append(next, n.Left)
			}
			if n.Right != nil {
				next = append(next, n.Right)
			}
		}
		out = append(out, level)
		queue = next
	}
	return out
}
```

On the tree above: `[[5] [1 4] [3 6]]`. **Complexity:** O(n) time; O(w) space where w is the
widest level, which can be about n/2 for a complete tree.

DFS or BFS, then? For "minimum depth", BFS can stop at the first leaf it meets, whereas DFS must
explore everything. For "is there a path with sum k", DFS uses O(h) memory where BFS would hold a
whole level. Both are O(n) time in the worst case.

## 8.5 Binary search trees

### The invariant, stated correctly

A **binary search tree (BST)** keeps, for every node, **all** values in its left subtree smaller
than it, and **all** values in its right subtree larger. That gives O(h) search, insert and
delete: at each node you know which half to go to, as in binary search.

The word "all" is the trap. Checking only that each left child is smaller than its parent and
each right child larger isn't enough:

```
        5
      /   \
     4     6
          / \
         3   7      ← 3 < 6, fine locally, but 3 is in 5's RIGHT subtree and 3 < 5
```

Every parent–child pair passes the local check; the tree is not a BST.

### Validating with bounds

The fix: pass down the range each subtree must lie within. Going left tightens the upper bound
to the current value; going right tightens the lower bound.

```go
func isValidBST(root *TreeNode) bool {
	var check func(n *TreeNode, lo, hi int64) bool
	check = func(n *TreeNode, lo, hi int64) bool {
		if n == nil {
			return true
		}
		v := int64(n.Val)
		if v <= lo || v >= hi {
			return false
		}
		return check(n.Left, lo, v) && check(n.Right, v, hi)
	}
	return check(root, math.MinInt64, math.MaxInt64)
}
```

<details><summary>Rust</summary>

```rust
use std::cell::RefCell;
use std::rc::Rc;

// LeetCode's Rust tree type: shared, mutable, nullable
pub struct TreeNode {
    pub val: i32,
    pub left: Option<Rc<RefCell<TreeNode>>>,
    pub right: Option<Rc<RefCell<TreeNode>>>,
}

fn is_valid_bst(root: Option<Rc<RefCell<TreeNode>>>) -> bool {
    fn check(node: &Option<Rc<RefCell<TreeNode>>>, lo: i64, hi: i64) -> bool {
        match node {
            None => true,
            Some(rc) => {
                let n = rc.borrow();
                let v = n.val as i64;
                v > lo && v < hi && check(&n.left, lo, v) && check(&n.right, v, hi)
            }
        }
    }
    check(&root, i64::MIN, i64::MAX)
}
```

</details>

<details><summary>C++</summary>

```cpp
struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int v, TreeNode* l = nullptr, TreeNode* r = nullptr) : val(v), left(l), right(r) {}
};

bool check(TreeNode* n, long long lo, long long hi) {
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return check(n->left, lo, n->val) && check(n->right, n->val, hi);
}
bool isValidBST(TreeNode* root) { return check(root, LLONG_MIN, LLONG_MAX); }
```

</details>

<details><summary>JavaScript</summary>

```js
function isValidBST(root) {
  const check = (n, lo, hi) =>
    n === null || (n.val > lo && n.val < hi && check(n.left, lo, n.val) && check(n.right, n.val, hi));
  return check(root, -Infinity, Infinity);
}
```

</details>

Trace on the trap tree:

```
call                    lo     hi     val   in (lo, hi)?
check(5)               −∞     +∞      5     yes
  check(4)             −∞      5      4     yes   (leaf children: true)
  check(6)              5     +∞      6     yes
    check(3)            5      6      3     NO  → false, all the way up
```

> ⚠️ **The bounds must be wider than the values.** LeetCode 98 allows node values from −2^31 to
> 2^31 − 1. If you use 32-bit `int` bounds initialised to `MinInt32`/`MaxInt32`, a tree containing
> exactly 2^31 − 1 fails the strict `v < hi` check and is wrongly rejected. Use 64-bit bounds (as
> above), infinities in JavaScript, or nullable bounds meaning "unbounded". Go's `int` is 64-bit on
> judges, so `math.MinInt`/`math.MaxInt` would also work; the explicit `int64` makes the intent
> obvious.

An equivalent check: do an in-order traversal and confirm each value is strictly larger than
the previous one, since in-order on a BST is sorted (§8.3).

The Rust signature deserves a sentence. LeetCode's Rust trees are
`Option<Rc<RefCell<TreeNode>>>`: `Rc` because the harness and your code both hold pointers to
nodes, `RefCell` because you might mutate them, `Option` for `nil`. It's verbose, and
`rc.borrow()` is how you read a node. Rust users on LeetCode spend a real fraction of their time
on this type, which is one more reason to interview in your strongest language
([file 01 §1.7](01-foundations.md#17-languages-what-this-course-uses-and-what-you-should)).

### When the BST is unbalanced

Insert sorted data into a plain BST and every node goes right: a degenerate tree with O(n)
operations. **Self-balancing** trees fix that by restructuring on insert and delete so that the
height stays O(log n). You won't be asked to implement one in a standard interview, but you
should know they exist and where you've met them:

| Structure | Where you meet it |
|---|---|
| Red-black tree | the usual implementation behind C++ `std::map`/`std::set` (the standard specifies O(log n) operations, not the tree type) |
| B-tree | Rust's `BTreeMap`; Postgres's default index type is a B-tree variant |
| No ordered tree in the standard library | Go, JavaScript |

## 8.6 How to recognise the approach

| You see… | Think… |
|---|---|
| A value for the whole tree computed from subtrees (depth, size, balanced?) | post-order DFS returning that value |
| Path, diameter, anything that can bend at a node | return one thing, record another |
| "Level", "row", "depth-by-depth", "rightmost at each level", "minimum depth" | BFS by levels |
| BST, "k-th smallest", "sorted order" | in-order traversal |
| Validate or search a BST | pass bounds down / go one way at each node |
| Two nodes and their relationship | LCA: ask each subtree whether it found them |
| Rebuild or serialise | pre-order (plus in-order, or explicit nils) |

## Asked in interviews

| Problem | What it tests |
|---|---|
| [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | The recursive leap of faith, in three lines |
| [543. Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Return height, record the path that bends here |
| [102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | BFS level by level |
| [98. Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | The whole-subtree invariant, and bound overflow |
| [230. Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | In-order is sorted; stop early |
| [236. Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) | Combining "found" signals from both sides |
| [105. Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/) | Why two traversals determine a tree; a hash map for O(1) splits |
| [124. Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) | The return-vs-record split, with negative values |

---

## Check yourself

1. `maxDepth` uses O(h) stack. For a tree of 10^5 nodes, what's h in the best and worst case,
   and in which of the four course languages could the worst case realistically fail?
2. In `maxPathSum`, why clamp each child's gain at 0? Give a tree where removing the clamp gives
   the wrong answer.
3. Why does `maxPathSum` initialise `best` to `math.MinInt` rather than 0? What input breaks the
   0 version?
4. A candidate validates a BST by checking, at each node, that `left.Val < node.Val < right.Val`.
   Give the smallest tree that this wrongly accepts.
5. In `lowestCommonAncestor`, `p` is an ancestor of `q`. Trace what the function returns and
   explain why it never looks for `q` below `p`, and why that's still correct.
6. Kth Smallest in a BST is called many times on a tree that's also being modified. What extra
   information would you store per node to answer each query in O(h), and how would you use it?

(Answers in [`15-exercises.md`](15-exercises.md).)
