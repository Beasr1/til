# 11. Glossary

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

## Reverts and branches

**Merge base** — the most recent commit both sides of a merge have in their history. A merge
combines what each side changed *since* it; anything at or before it counts as done.

**Mainline (`git revert -m <n>`)** — which parent of a merge to keep when reverting it.
`-m 1` keeps the branch that was merged into and drops what the merge brought in.

**Sticky revert** — not git's term, but the behaviour: reverting a merge undoes its content
and leaves its ancestry, so later merges of the same branch bring only commits made after
it. Fixed by reverting the revert *before* re-merging, or by re-landing the work as fresh
commits (`git rebase --no-ff`).

**Revert the revert** — `git revert <the revert>`: re-applies exactly what the revert
removed. Recent git titles it `Reapply "…"`.

**Environment branch** — a long-lived branch per deployment environment (dev, staging,
production). Diverges as soon as changes are merged into each separately.

**Trunk** — the single shared branch everyone integrates into, in trunk-based development.
Release branches are cut from it and never merged back.

**Feature flag (toggle)** — a runtime switch that turns code paths on or off by
configuration, so every environment can run the same commit.

## Attributes and line endings

**LF / CRLF** — line endings: `\n` (one byte) or `\r\n` (two). Git compares bytes, so
changing endings changes every line.

**`.gitattributes`** — a committed file assigning attributes to paths by pattern. When
several patterns match, the later line wins, per attribute.

**`text` / `text=auto`** — marks paths as text, so their endings are normalised to LF in the
index. `auto` lets git decide per file, and leaves files already committed with CRLF alone.

**`eol=lf` / `eol=crlf`** — the line ending a path gets in the working tree on checkout, on
every machine. Implies `text` if `text` is unset.

**`binary`** — built-in macro for `-diff -merge -text`.

**`core.autocrlf`** — a per-clone setting. `true` acts like `text=auto` everywhere plus CRLF
on checkout; `input` normalises on the way in only. Consulted only for paths with no `text`
attribute.

**Renormalise** — `git add --renormalize .` re-applies the attributes to files already
committed. `git merge -X renormalize` does the same for the three sides of a merge.

**`--ignore-cr-at-eol` / `-w`** — diff options that hide carriage-return-only, or all
whitespace, differences.

**`.git-blame-ignore-revs`** — a file of full SHAs that blame skips
(`blame.ignoreRevsFile`, or `--ignore-revs-file`). GitHub reads it from the repository root.

**Similarity index** — how much of a file survived between a deleted and an added path. At
50% or more (the default threshold, `-M`/`find-renames`) git calls the pair a rename;
`--name-status` shows it as `R` plus the score.

## Merge results

**Semantic conflict** — a merge that's textually clean and wrong: a duplicate declaration,
both sides of a conflict kept, a call to a signature that changed.

**Merged results pipeline** — CI that tests the temporary merge of a branch into its target
rather than the branch alone (GitLab's term; GitHub's `pull_request` workflows run on the
`refs/pull/<n>/merge` commit).

**Link-time error (ES modules)** — importing a name the target module doesn't export throws
a `SyntaxError` while the module graph is linked, before any module code runs.

## Secrets

**Rotation** — replacing a credential and revoking the old one. The first response to a
committed secret; deleting the file doesn't un-leak it.

**`git rm --cached`** — stop tracking a file from the next commit on. Earlier commits keep it.

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

## Identity

**Author** — who wrote the change. Kept across rebase and cherry-pick.

**Committer** — who made this particular commit object. Becomes *you* on every rebase,
cherry-pick or `--amend`.

**`user.name` / `user.email`** — where both identities normally come from. Plain strings:
git verifies neither. Overridden by `author.*` / `committer.*` keys, which are overridden by
`GIT_AUTHOR_*` / `GIT_COMMITTER_*` environment variables.

**`git config --show-origin --show-scope <key>`** — which file, and which layer
(system/global/local/worktree/command), a value came from. Add `--get-all` to see every
value; the last one wins.

**`git var GIT_AUTHOR_IDENT`** — the identity git would stamp right now, environment
included.

**Conditional include (`includeIf`)** — pull another config file in only when a condition
holds. Spliced in where it appears, so it must come *after* what it overrides.

**`gitdir:`** — `includeIf` condition matching the `.git` directory's path. A trailing `/`
means "and everything below"; without it, only that exact path. `gitdir/i:` is the
case-insensitive form.

**`hasconfig:remote.*.url:`** (git ≥ 2.36) — `includeIf` condition matching any remote URL.
Doesn't fire before the remote exists.

**`user.useConfigOnly`** — refuse to guess an email from login and hostname; with no email
configured, commits fail instead of going out with a guess.

**`.mailmap`** — maps old names/emails to canonical ones *for display* in `log` and
`shortlog`. Rewrites nothing.

**`git filter-repo`** — separate tool, recommended by git's own manual over
`filter-branch`, for rewriting history wholesale (e.g. replacing an email in every commit).
New SHAs throughout; force-push everything. Its `--sensitive-data-removal` flag (version
2.47 or later) is what GitHub's guide uses to scrub a leaked file or string.

**`noreply` address** — GitHub's `ID+USERNAME@users.noreply.github.com`, which attributes
commits to your account without publishing a real address.

**Signed commit** — a commit carrying a cryptographic signature (`gpg.format` = `openpgp`,
`x509` or `ssh`; key in `user.signingKey`). The only thing that proves which key made it;
GitHub shows **Verified** when the signature checks out against a key on an account.

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
