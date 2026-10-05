# 1. What a commit actually is

You need this file even though you already use git every day, because every confusing
thing in the rest of the course is a consequence of one fact that git never states out loud.

## 1.1 The object

A commit is not a diff. It is a small record containing:

```
tree      <hash of the entire directory contents at this moment>
parent    <hash of the commit before it>
author    Ishaan <…>  1756032151 +0400
committer Ishaan <…>  1756032151 +0400

feat(core): stamp request provenance into the sandbox records
```

The commit's **name** — the SHA you see as `a61be947` — is a hash of all of that, together.
Not of the message. Not of the changes. Of the snapshot, the parent, the author, the
timestamps, and the message, as one blob.

Diffs are *computed*, not stored. When git shows you what a commit "changed", it is
comparing that commit's tree against its parent's tree, on the spot.

## 1.2 The three consequences

**A commit cannot be edited.** Change the message and the hash changes. Change one
character in one file and the hash changes. Change *the parent* — which is what rebasing
does — and the hash changes, even though not one byte of your code moved.

So `git commit --amend` does not amend anything. It builds a *new* commit and points the
branch at it. The old one is still on disk, unreferenced, until garbage collection gets it
(§6). Same for rebase, same for `reset --hard`. Git almost never destroys; it stops
pointing.

**A branch is a pointer, not a container.** Literally:

```sh
$ cat .git/refs/heads/master
3817ce1832ee5b6a7a2b0e5cbf95bd8b5d3f1c4a
```

That is the whole branch. 41 bytes. "The commits on master" is not a list git stores — it
is whatever you reach by following parents back from that one hash. Which is why deleting
a branch is instant regardless of size, and why creating one is free.

**"Moving a commit to another branch" is impossible.** There is no such operation. What
cherry-pick and rebase actually do is *build a new commit* with the same tree-change and
message, on a different parent. It looks like the same commit. It has a different name, so
it is a different commit. Git will happily let both exist at once — that is exactly the
state you land in after rebasing a branch you had already pushed (§5).

## 1.3 HEAD, and why rebase detaches it

`HEAD` is a pointer to *what you have checked out* — normally a branch name:

```sh
$ cat .git/HEAD
ref: refs/heads/feat/env-gate
```

Mid-rebase you will see `HEAD detached at 3817ce18` or commits landing on
`[detached HEAD 11fa9626]`. That is not damage. Rebase replays your commits one at a time
onto the new base, and until it finishes there is no branch to point at the half-built
chain — so HEAD points straight at a commit instead. When the last one lands, git moves
your branch pointer to the new tip and re-attaches HEAD.

```
before rebase                     during                          after
                                  (HEAD detached)
master   ──A──B──C                master ──A──B──C──D             master ──A──B──C──D
                \                                    \                              \
mine      ────────X──Y            mine ────X──Y       X'          mine ──────────────X'──Y'
                                       (still here)  (HEAD)
```

Note `X` and `Y` in the middle picture: your original commits still exist during the
rebase. If it goes wrong, `git rebase --abort` just points the branch back at `Y` and it
is as if nothing happened.

## 1.4 Why any of this matters

Because from here, these all become the same fact:

- Why can't I push after rebasing? → your commits have new names; the remote has the old ones.
- Why does `git pull` make a mess after a rebase? → it merges the old names back in beside the new ones.
- Why did my IDE say 99 outgoing? → it compares two pointers, and one of them is stale.
- Why is `--force-with-lease` safe? → it checks a pointer hasn't moved before overwriting it.

---

## Check yourself

1. Two commits have identical file contents and identical messages, but different parents.
   Same commit or different?
2. You run `git commit --amend` to fix a typo in a message. What happened to the original
   commit object?
3. Why does deleting a 10,000-commit branch take the same time as deleting a 1-commit one?
4. During a rebase you see `HEAD detached at 3817ce18`. Have you lost your branch?
5. Someone says "I moved my commit onto master". What did git actually do?
