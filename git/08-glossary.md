# 8. Glossary

Terms in the order you meet them, not alphabetically — look things up with your editor's
search.

## Objects and refs

**Commit** — a snapshot of the whole tree, plus parent(s), author, committer, message. Its
SHA hashes all of that, so changing any part yields a different commit. Not a diff.

**Tree** — the directory listing a commit points at. Files are **blobs**.

**Ref** — a name pointing at a commit. `refs/heads/x` is a branch, `refs/tags/v1` a tag,
`refs/remotes/origin/x` a remote-tracking ref.

**HEAD** — what you have checked out. Usually `ref: refs/heads/<branch>`; during a rebase,
a raw SHA (**detached HEAD**).

**ORIG_HEAD** — where HEAD was before the last `rebase`/`merge`/`reset`/`pull`. One-step undo.

**Remote-tracking ref** (`origin/x`) — your local *cache* of where the remote branch was
when you last fetched. Not live.

**Upstream / tracking branch** — the remote-tracking ref a local branch is paired with;
what `@{upstream}`, `git status`'s "ahead/behind" and your IDE's "outgoing" compare against.

**Gitlink** — the recorded commit of a submodule, stored in the superproject's tree like
any other entry. Rebases like content; does **not** check itself out.

## Operations

**Fast-forward** — the remote's tip is an ancestor of yours, so the pointer can slide
forward with no new commit. A push that is not a fast-forward is rejected by default.

**Diverged** — neither tip is an ancestor of the other. What a rebase-then-push produces.

**Merge** — a new commit with two parents joining two histories. Preserves hashes.

**Rebase** — replay commits onto a new base, creating new commits with new SHAs. `--onto`
lets you pick the base and the range independently.

**Cherry-pick** — replay one commit elsewhere. Same content, new SHA — so "is it on
master" by SHA says no even when the change is there.

**Squash / fixup** — collapse commits into one. Same rewrite rules apply.

**Revert** — a *new* commit that undoes an old one. The safe way to withdraw something
already published, because it adds history instead of rewriting it.

## Rebase mechanics

**`--continue` / `--skip` / `--abort`** — resolve and carry on / drop this commit / put
everything back.

**`HEAD` during a rebase** — the side you are rebasing **onto** (upstream). The labelled
side is your commit. Opposite of a merge.

**`modify/delete` conflict** — one side edited a file the other deleted. Common when
upstream moved or renamed a crate. Deleting the stale path is only half the fix; the other
half is re-applying your intent at the new location.

**Rename detection** — git matches a deleted path to an added one by content similarity.
When it works you get one merged file (markers show both paths). When the file changed too
much, you get `modify/delete` instead.

**rerere** — *reuse recorded resolution*. Memorises conflict resolutions and replays them.
`git config --global rerere.enabled true`.

## Push safety

**`--force`** — make the remote match me, whatever is there. No checks.

**`--force-with-lease`** — compare-and-swap: overwrite only if the remote is still at the
value my remote-tracking ref says. Defeated by background auto-fetch, which renews the
lease on commits you never read.

**`--force-with-lease=<ref>:<sha>`** — the same, with the expected value stated explicitly
instead of read from a ref that moves.

**`--force-if-includes`** (git ≥ 2.30) — additionally requires that the remote tip is
reachable from your local history, i.e. you actually integrated it rather than merely
fetching it.

## Inspection

**`git range-diff <old-range> <new-range>`** — pairs up two versions of the same branch.
`=` unchanged · `!` changed · `>` new only · `<` **dropped**. The verifier for a rebase.

**`git rev-list --left-right --count A...B`** — two numbers: commits only on A, commits
only on B.

**`git merge-base --is-ancestor X Y`** — exit status 0 if X is on Y's history. The precise
form of "is this already on master".

**`git reflog`** — local log of every ref movement, addressable as `HEAD@{n}`. Not pushed,
not fetched, expires (90 days reachable / 30 unreachable).

**`git fsck --lost-found`** — finds unreferenced objects when the reflog has been pruned.

## The dots

| | `log` / `rev-list` | `diff` |
|---|---|---|
| `A..B` | on B, not on A | A's tip vs B's tip |
| `A...B` | on either, not both (symmetric) | **merge base** of A and B, vs B |

Two idioms that are almost always what you want:

```sh
git log --oneline master..HEAD     # what my branch adds
git diff master...HEAD             # what my branch changed, ignoring master's moves
```
