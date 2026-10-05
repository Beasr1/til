# 5. Force-with-lease

You rebased. You pushed. Git said no. This file is what to do, and why the obvious two
answers are both wrong.

## 5.1 The rejection

```
 ! [rejected]   feat/env-gate -> feat/env-gate (non-fast-forward)
error: failed to push some refs to 'https://git.example.com/acme/ledger.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. If you want to integrate the remote changes,
hint: use 'git pull' before pushing again.
```

**The hint is wrong for your situation.** It is written for the common case (you really
are behind), and git cannot tell the difference. What "non-fast-forward" actually means is
narrow and precise:

> The remote's current commit is **not an ancestor** of the commit you are pushing.

Fast-forward = "I can get from where the remote is to where you are by only walking
forward." After a rebase you cannot: the remote holds `X──Y──Z` and you hold
`X'──Y'──Z'`, and neither chain contains the other. Not ahead, not behind — **diverged**.

Git refuses because overwriting would make the remote's commits unreachable, and it has no
way to know whether that is your intent or a catastrophe.

## 5.2 The two wrong fixes

**`git pull`** — what the hint says. It merges the old chain back into the new one, so
every change exists twice with a merge commit joining them, and you get conflicts between
your work and itself. See [file 2](02-rebase-vs-merge.md) §2.5. This is the single most
common way people ruin a rebase they had done correctly.

**`git push --force`** — works, and is a loaded gun. It means *make the remote match me,
whatever is there*. If a colleague pushed to that branch in the last ten minutes, their
commits are now unreachable and neither of you gets a warning. If you were on the wrong
branch, likewise.

## 5.3 The right fix

```sh
git push --force-with-lease origin feat/env-gate
```

Read it as: **overwrite the remote, but only if it is still exactly where I last saw it.**

The "lease" is your remote-tracking ref — `refs/remotes/origin/<branch>`, the local
snapshot updated by `fetch`/`pull`. The push sends "I expect you to be at `a61be947`", and
the server does a compare-and-swap: if the branch is still at `a61be947`, replace it;
otherwise reject.

So it protects the exact case `--force` doesn't: someone pushed while you weren't looking.
Their commit moved the branch off the value you leased, and your push bounces instead of
erasing them.

Before pushing, you can see precisely what you are about to discard:

```sh
git fetch origin feat/env-gate         # confirm the lease value is current
git log --oneline origin/master..origin/feat/env-gate
# a61be947 feat(core): stamp request provenance…    ← the 6 commits about to be replaced
# …
```

If that list is only your own pre-rebase commits, you are fine. If someone else's message
is in it, stop.

## 5.4 The gotcha that makes the lease lie

`--force-with-lease` trusts your remote-tracking ref. Anything that updates that ref
*without you reading the new commits* silently renews the lease on work you have never
seen. The usual culprits:

- **your IDE's auto-fetch** (VS Code's `git.autofetch`, JetBrains' background fetch)
- a `git fetch` in a shell prompt or status-line integration
- `git fetch --all` you ran for an unrelated reason

Sequence: colleague pushes → your editor auto-fetches → `origin/branch` now points at
their commit → your lease is against *their* commit → the compare-and-swap succeeds →
their work is gone. The safety net checked the thing that had already been updated.

Two ways to close it:

**Name the value explicitly.** Then the lease is a number you chose, not a ref that moves:

```sh
git push --force-with-lease=feat/env-gate:a61be947 origin feat/env-gate
```

**Or use `--force-if-includes`** (git ≥ 2.30), which additionally requires that the remote
tip is actually reachable from your local history — i.e. you have integrated it, not merely
fetched it:

```sh
git push --force-with-lease --force-if-includes origin feat/env-gate
```

That is the strongest of the three and the one to reach for by default on a shared branch.

## 5.5 What no flag protects against

- **Work that was never pushed.** If a colleague has local commits on your branch, the
  server never knew; your force-push is fine and their rebase is their problem.
- **Someone who already pulled the old chain.** Their clone still has `X──Y──Z` and their
  next pull will try to merge it back into your new history. They need
  `git fetch && git reset --hard origin/<branch>` — tell them, don't let them discover it.
- **Protected branches.** Correctly so. If force-pushing `master` is what you want, the
  answer is almost always `git revert` instead: a new commit that undoes the change,
  leaving history intact for everyone who already has it.

## 5.6 The habit

Make the safe one the easy one:

```sh
git config --global alias.pushf 'push --force-with-lease --force-if-includes'
```

and then never type `--force` again. If `pushf` is rejected, that rejection is information:
something changed on the remote. Go look at it before you insist.

---

## Check yourself

1. In one sentence, what does "non-fast-forward" actually mean?
2. Your push is rejected after a rebase. Are you behind the remote? What is true instead?
3. What specific thing does `--force-with-lease` compare, and against what?
4. Your editor auto-fetches every 3 minutes. Explain how that can make
   `--force-with-lease` overwrite a colleague's commit, step by step.
5. Which flag closes that hole, and what does it require?
6. A colleague already pulled the branch you just force-pushed. What do they run?
7. When is `git revert` the right answer instead of a force-push?
