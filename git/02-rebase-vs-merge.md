# 2. Rebase vs merge

Both answer the same question — *my branch was cut from master a while ago and master has
moved on; how do I get its changes?* — and they answer it differently enough that teams
pick one and stick to it.

## 2.1 The situation

```
master  ──A──B──C──────D──E──F        ← 91 commits happened while you worked
              \
mine           X──Y──Z                ← your 8
```

Your branch's parent chain still ends at `C`. Everything after `C` is invisible to you:
you are writing code against a three-week-old tree. That is fine until it isn't — until
someone renames the crate your commit edits.

## 2.2 Merge

`git merge master` creates one new commit with **two parents**, joining the histories:

```
master  ──A──B──C──────D──E──F
              \              \
mine           X──Y──Z────────M
```

- Your commits keep their hashes. Nothing you already pushed is invalidated.
- The history records what actually happened, forks and all.
- The graph gets wide. On a busy repo, `git log --graph` becomes unreadable and
  `git bisect` has more shapes to walk.
- Do it repeatedly and your branch accumulates `M1`, `M2`, `M3` — merge commits that say
  nothing except "I synced on a Tuesday".

## 2.3 Rebase

`git rebase master` replays each of your commits onto master's tip, in order:

```
master  ──A──B──C──────D──E──F
                             \
mine                          X'──Y'──Z'
```

- The history is linear. `git log` reads as a story; every commit's diff is against the
  code as it will actually ship.
- `X'` is a *new commit* (§1). Same change, same message, different parent, different
  hash. `X` still exists but nothing points at it.
- Anything referring to the old hashes is now stale: a pushed branch, a review comment
  citing a SHA, a note in a ticket.

## 2.4 Which, and why a team must agree

The trade is **fidelity vs legibility**. Merge records what happened; rebase produces
what you wish had happened — a branch that looks as though you started from today's master
and wrote it in one clean run.

`ledger` picks rebase, and says so out loud: *"sync by rebase — never merge master into
your branch."* That is the right shape for trunk-based work with short branches and one
MR per change. A long-lived integration branch with six people on it wants merges instead,
because rebasing rewrites commits those six people have already built on.

Which gives the actual rule, the one worth memorising:

> **Rebase commits nobody else has built on. Merge when they have.**

Note what that rule does *not* say. It does not say "never rebase pushed commits". Pushing
is not the line — *someone else basing work on them* is. A feature branch only you touch is
safe to rebase and force-push all day, and doing so is normal (§5). The stricter folklore
version ("never rewrite published history") is a useful default for a shared trunk and
wrong for a personal branch.

## 2.5 The trap: `git pull` after a rebase

You rebase. You push. Git rejects it. The hint says:

```
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. If you want to integrate the remote changes,
hint: use 'git pull' before pushing again.
```

**Do not do that.** You are not behind. The remote holds `X──Y──Z` (your old commits) and
you hold `X'──Y'──Z'` (the same work, new names). `git pull` merges the two, and you end up
with every change twice, joined by a merge commit:

```
                     ┌── X ──Y──Z ────┐
remote-as-was ───────┤                ├── M     ← every change committed twice
your rebase   ───────└── X'──Y'──Z' ──┘
```

Then the conflicts arrive, because both sides "changed" the same lines. The fix for a
rejected push after a rebase is never `pull` — it is `--force-with-lease`, which is
[file 5](05-force-with-lease.md).

If you want `pull` to stop suggesting merges entirely:

```sh
git config --global pull.rebase true
```

## 2.6 What rebase is *not* for

- **Not for combining two people's branches.** That is a merge, or a PR.
- **Not for "cleaning up" after review comments cite your SHAs.** Fine, but expect the
  links to rot; say so in the PR.
- **Not a substitute for small commits.** A rebase of one enormous commit still conflicts
  as one enormous conflict.

---

## Check yourself

1. After `git rebase master`, how many of your commits kept their original hash?
2. Your colleague has a branch based on yours. You rebase yours and force-push. What do
   they now have to do, and whose fault is it?
3. Why does a merge commit have two parents when an ordinary commit has one?
4. The push hint says you are "behind its remote counterpart" after a rebase. Is that
   accurate? What is actually true?
5. Your branch has three merge commits from syncing master weekly. What does a reviewer
   see in `git log --graph` that they would not see had you rebased?
