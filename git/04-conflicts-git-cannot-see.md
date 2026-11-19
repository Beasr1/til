# 4. The conflicts git cannot see

⭐ This is the file to read twice.

Git merges **text**. It has no idea what the text means. So the dangerous outcome of a
rebase is not the conflict it stops on — you cannot ignore those. It is the hunk it merges
*cleanly and wrongly*, and reports nothing about.

The first three examples below come from one rebase, and none of them produced a conflict
marker. §4.4 adds three more from ordinary merges, because a merge has the same blind spot.

## 4.1 Semantic: the type changed underneath you

Upstream had changed how request ids are represented — from `Option<u64>` to a typed
`RequestId`. Your commit added two new handlers taking the old type. Both sides touched
*different lines in the same file*, so git merged them without a murmur.

```rust
// merged clean. six compile errors.
async fn sync_sandbox_account(&self, payload: …, request_id: Option<u64>) -> Response
//                                                             ^^^^^^^^^^^ upstream now passes RequestId
```

The compiler caught it, which makes this the *benign* member of the family. Everything
git can't see that the compiler also can't see is worse.

**The rule this gives you:** a rebase is not done when `git status` is clean. It is done
when the thing builds and the tests pass. Run the project's real gate, not `cargo check`
on the crate you remember touching.

There is a subtlety worth knowing here. Rebase replays commits one at a time, and each
intermediate commit is committed *before* you build anything. So a rebased branch can have
a green tip and a middle commit that does not compile — which quietly breaks `git bisect`
later. Fixing it properly means folding the repair into the commit that needs it; fixing
it pragmatically means one repair commit at the tip and a note in the MR. Know which you
chose.

## 4.2 Namespace: someone took your numbers

The branch had added end-to-end scenarios **S26** and **S27** to a test-plan document, and
the test files referenced them by name. While the branch sat, master added **S26 through
S41** for entirely different features.

Git merged both tables happily — different lines of a markdown table, no overlap. The
result was a document with two S26s meaning two different things, and a repo where
`grep S27` returned both a sandbox test and a transport test.

Nothing red. Nothing to build. Just a document that had quietly become wrong.

The fix was to renumber the *unmerged* side (S26/S27 → **S42/S43**) — because the rule in
that repo is that landed scenario IDs never move, and master's had landed. Then grep for
the old identifiers across code and docs, because a stale ID in a test comment now points
at someone else's scenario.

**Where this class hides:** anything humans allocate from a shared sequence.

- test/scenario IDs
- error codes (a repo that says "codes are permanent" means it)
- migration numbers — `V017__add_column.sql` twice is a fun afternoon
- port numbers, feature flags, enum discriminants, protobuf/capnp field numbers
- fixture ids, seed data primary keys

If your branch allocated a number from a shared range, re-check that range after rebasing.
Always.

## 4.3 Release: upstream shipped the version you were bumping to

The nastiest of the three, because the failure mode is **silence in production tooling**.

The branch bumped every changed artifact by one minor — capi `0.22.0 → 0.23.0`, and so on
for seven SDKs. Meanwhile master had *released* those exact numbers for its own work.

After the rebase, the manifests said `0.22.0`… and the release lock said `0.22.0`. The
repo's CI publishes an artifact only when its version differs from the lock, so:

```sh
$ cargo xtask changed
api-prod-v1        # ← that's all. Eight other changed artifacts, invisible.
```

Nothing failed. The MR would have gone green, merged, and shipped **none** of the SDK
changes, because every release job would have been skipped as a no-op. The only signal was
a tool nobody runs by habit.

**The rule:** if your branch touches versions, changelogs, or anything else whose meaning
is "differs from what is published", re-derive it after a rebase rather than trusting what
the merge produced. Ask the build system what it thinks changed, out loud.

## 4.4 Merges: clean, and still broken

Git's own description of a merge contains the whole problem
([git-merge, "How conflicts are presented"](https://git-scm.com/docs/git-merge)):

> Among the changes made to the common ancestor's version, non-overlapping ones (that is,
> you changed an area of the file while the other side left that area intact, or vice
> versa) are incorporated in the final result verbatim.

A conflict means *two edits overlapped*. No conflict means only that they didn't. Here are
three shapes from ordinary merges where that wasn't enough.

### The same change, arriving twice

One commit reached two branches by two routes: cherry-picked onto one, rebased onto the
other. Each copy is a separate commit as far as git knows ([file 1](01-what-a-commit-is.md)
§1.2), and each added the same constant, but at different places in the file, because the
surrounding code differed on each branch. When the branches merged, the two additions didn't
overlap, so git kept both, verbatim.

```go
package main

import "fmt"

// added by one copy, near the top
const DefaultOrderType = "standard"

func describe() string { return "orders" }

// added by the other copy, further down
const DefaultOrderType = "standard"

func main() { fmt.Println(describe(), DefaultOrderType) }
```

```
./main.go:11:7: DefaultOrderType redeclared in this block
	./main.go:6:7: other declaration of DefaultOrderType
```

The other three languages refuse it the same way. The errors below are what each toolchain
gave me (rustc 1.96, Apple clang 17, Node v26.13.0).

<details><summary>Rust</summary>

```rust
const DEFAULT_ORDER_TYPE: &str = "standard";
fn describe() -> &'static str { "orders" }
const DEFAULT_ORDER_TYPE: &str = "standard";
// error[E0428]: the name `DEFAULT_ORDER_TYPE` is defined multiple times
```

</details>

<details><summary>C++</summary>

```cpp
constexpr std::string_view kDefaultOrderType = "standard";
std::string_view describe() { return "orders"; }
constexpr std::string_view kDefaultOrderType = "standard";
// error: redefinition of 'kDefaultOrderType'
```

</details>

<details><summary>JavaScript</summary>

```js
export const DEFAULT_ORDER_TYPE = "standard";
export function describe() { return "orders"; }
export const DEFAULT_ORDER_TYPE = "standard";
// SyntaxError: Identifier 'DEFAULT_ORDER_TYPE' has already been declared
```

Nothing in the module runs: it fails to compile when it's loaded, and so does everything
that imports it.

</details>

The same thing happens with import lines that both sides added in slightly different spots
(Go says `fmt redeclared in this block`). This is the benign case, because the compiler
refuses a duplicate declaration. The next one isn't.

### Both sides of a conflict, kept

When a conflict does appear, most editors offer an "accept both" button. It's the right
answer when the two sides added *different* things, like two new imports or two new list
entries. It's the wrong answer when they're two versions of *one* thing:

```go
<<<<<<< HEAD
	cells = append(cells, fmt.Sprintf("%v", amount))
=======
	cells = append(cells, fmt.Sprintf("%.2f", amount))
>>>>>>> feature
```

Accept both and you get both lines. Both are valid, so it compiles, every test that doesn't
count cells passes, and the table shows every amount twice. Nothing but a person looking at
the screen will notice.

<details><summary>Rust</summary>

```rust
<<<<<<< HEAD
    cells.push(format!("{}", amount));
=======
    cells.push(format!("{:.2}", amount));
>>>>>>> feature
```

</details>

<details><summary>C++</summary>

```cpp
<<<<<<< HEAD
    cells.push_back(std::format("{}", amount));
=======
    cells.push_back(std::format("{:.2f}", amount));
>>>>>>> feature
```

</details>

<details><summary>JavaScript</summary>

```jsx
<<<<<<< HEAD
  <td>{amount.toLocaleString()}</td>
=======
  <td>{amount.toLocaleString(undefined, { minimumFractionDigits: 2 })}</td>
>>>>>>> feature
```

In the JSX version "both" is two adjacent cells, which render without complaint.

</details>

So after resolving, re-read every hunk where you kept both sides, and run the thing.

### A shared signature changed under a long-lived branch

A feature branch was green on its own CI for weeks. Meanwhile trunk landed a cross-cutting
refactor that gave a shared helper a context parameter and updated every caller it could
see. The branch's new handler, in a file the refactor never touched, still called the helper
the old way. The merge was clean. The build wasn't. This is §4.1 again, on a merge.

```go
// trunk, after the refactor: every caller now passes a context
func ensureToken(ctx context.Context, force bool) (string, error) { … }

// the feature branch's new handler, written against the old signature
_, err := ensureToken(false)
// not enough arguments in call to ensureToken
// 	have (bool)
// 	want (context.Context, bool)
```

<details><summary>Rust</summary>

```rust
fn ensure_token(ctx: &Ctx, force: bool) -> Result<String, Error> { … }
ensure_token(false)
// error[E0061]: this function takes 2 arguments but 1 argument was supplied
```

</details>

<details><summary>C++</summary>

```cpp
std::string ensureToken(const Context& ctx, bool force) { … }
ensureToken(false);
// error: no matching function for call to 'ensureToken'
```

</details>

<details><summary>JavaScript</summary>

```js
function ensureToken(ctx, force) { … }
ensureToken(false);
// No error at all: ctx is false and force is undefined. It fails at runtime, if you're lucky.
```

JavaScript is the odd one out: a missing argument is just `undefined`, so a changed
signature isn't even a load-time error.

</details>

In a compiled language the build catches it, *if anything builds the merge result*. In
JavaScript with native ES modules, the neighbouring mistake fails later and bigger: one side
renames an export, or a conflict resolution drops one, and another module still imports the
old name. The spec makes that a link-time error. When a module's imports are resolved, "If
resolution is either null or ambiguous, throw a SyntaxError exception"
([ECMAScript, InitializeEnvironment](https://tc39.es/ecma262/multipage/ecmascript-language-scripts-and-modules.html)).
Linking happens before any module's code runs, so one bad import anywhere in the graph stops
the whole process from starting, not just the route that uses it. In Node v26.13.0:

```
SyntaxError: The requested module './messages.js' does not provide an export named 'messages'
```

Neither module's top-level code had run. And `node --check` passed the same file, because it
parses and doesn't link. What caught it was loading the program:

```sh
node --input-type=module -e "await import('./server.js')"   # exits non-zero on a bad import
```

(If importing your entry point starts a server, import the module that builds the app
instead, or boot it once and hit a health check.)

### What catches all three

- **Test the merge result, not just the branch.** A branch's own pipeline tests the branch.
  GitLab's merged results pipelines "test a temporary merged commit that combines code from
  the source and target branches"
  ([GitLab docs](https://docs.gitlab.com/ci/pipelines/merged_results_pipelines/)). On GitHub
  Actions, a `pull_request` workflow runs on the "Last merge commit" of the
  `refs/pull/<number>/merge` ref
  ([GitHub docs](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)).
  Either way the merge is computed when the pipeline runs. If trunk moves after that, the
  result you tested isn't the one that will land.
- **Merge trunk into long-lived branches right after a cross-cutting refactor lands** (or
  rebase them). The breakage then shows up in the branch, small and early, owned by the
  person who understands it, instead of at merge time.
- **Make CI load the whole program.** For a compiled language that's the build. For an
  interpreted one it's at least importing the entry point, because a syntax check and a
  container build both pass code that can't start.

## 4.5 The general shape

Every one of these is the same failure:

> Git guarantees your **text** survives a rebase. It guarantees nothing about your
> **assumptions**.

Your commits encoded assumptions about the world at the moment you wrote them: this type
looks like that, S26 is free, 0.23.0 is unreleased. The rebase moved those commits three
weeks forward without re-checking a single one.

So after a rebase, ask what upstream took while you were away:

| Assumption | How to re-check |
|---|---|
| Types and signatures I call | Build. The whole thing. |
| Identifiers I allocated | `grep` the shared range for duplicates |
| Versions I bumped | Ask the release tooling what it considers changed |
| Files I edited | `git log --oneline <base>..master -- <paths>` — who else touched them? |
| Config/flags I added | Does the key already exist upstream with another meaning? |
| Code that arrived by two routes, or from both sides of a conflict | Build and run the merge result itself; re-read every "accept both" hunk |
| Exports and signatures other code depends on | Load the whole program: build it, or import the entry point |

That last query is worth putting in your fingers. Before rebasing:

```sh
git log --oneline HEAD..origin/master -- $(git diff --name-only $(git merge-base HEAD origin/master))
```

*What has upstream done to the files I touched?* Everything expensive is in that answer.

---

## Check yourself

1. Why did the `RequestId` change not produce a conflict marker?
2. A rebased branch's tip builds. Why might `git bisect` still break, and what would you
   do about it?
3. Your branch added migration `V017__add_index.sql`. Upstream also added a `V017`.
   Git reports no conflict. Why not, and what breaks?
4. What does it mean that the version collision produced "silence in production tooling"?
   Why is that worse than a red build?
5. Name three kinds of shared sequence a branch can allocate from.
6. In one sentence: what does a rebase actually guarantee?
7. One commit was cherry-picked onto one branch and rebased onto another, and the copies
   landed at different places in the same file. The branches merge with no conflict, and
   the build fails with a redeclaration. Why did git see nothing to conflict, and which CI
   setup would have caught it before the merge?
8. A Node service's CI runs `node --check` and builds a container image. A merge resolution
   drops a named export that another module imports. What happens at deploy, why is it the
   whole service rather than one route, and what's the cheapest CI step that would have
   caught it?
