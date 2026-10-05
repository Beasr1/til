# Rewriting History Safely — A Git Course

A course on the half of git that scares people: rebasing, force-pushing, and reading the
state you end up in afterwards — plus one chapter on the identity stamped into every commit
you make, because that is history too, and just as hard to take back.

**This is reference learning material.** The concepts are general git. The worked example
running through it is real (names changed) — a rebase of `acme/ledger`'s `feat/env-gate` onto a
master that had moved 91 commits ahead, done on 2026-08-24 — because an abstract rebase
teaches you nothing and a real one teaches you the four things that actually go wrong.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- I use the mental model before the commands, and *why* before *how*.
- Every file ends with **Check yourself** questions. Answers are in `10-exercises.md`.
- Every dangerous command comes with what it destroys and how to get it back.

## The one thing to understand first

Almost every confusing thing in git — why a push is rejected, why your IDE says you have 99
outgoing changes when you wrote 8, why `git pull` after a rebase creates a mess — follows
from a single fact:

> **A commit's identity is computed from its content *and its parent*. Change either and it
> is not the same commit any more. It is a new commit that looks like the old one.**

Rebasing changes parents. So rebasing replaces every commit it touches. Everything else in
this course is a consequence.

## Reading order

### Part 1 — The model

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What a commit actually is](01-what-a-commit-is.md) | Explain why a rebase changes SHAs, and why that is not a bug |
| 2 | [Rebase vs merge](02-rebase-vs-merge.md) | Choose one deliberately, and say what each costs |

### Part 2 — Doing it

| # | File | Covers |
|---|------|--------|
| 3 | [A real rebase, start to finish](03-a-real-rebase.md) | The whole sequence, including the four conflict shapes and a submodule ambush |
| 4 | [**The conflicts git cannot see**](04-conflicts-git-cannot-see.md) | ⭐ The part nobody warns you about: clean merges that are wrong |
| 5 | [Force-with-lease](05-force-with-lease.md) | Push a rewritten branch without overwriting someone's work |

### Part 3 — Not losing anything

| # | File | Covers |
|---|------|--------|
| 6 | [Safety nets](06-safety-nets.md) | Backup refs, reflog, `ORIG_HEAD`, `range-diff`, stashes across a rewrite |
| 7 | [Reading divergence](07-reading-divergence.md) | What "outgoing changes" really compares, and the two-dot/three-dot trap |

### Part 4 — Who the history says you are

Added later, so it carries the next free number. It needs only file 01 (§1.1, the `author`
and `committer` lines); its section on rewriting identities leans on file 05. Read it any
time — ideally before your first commit on a new machine.

| # | File | After this you can… |
|---|------|---------------------|
| 8 | [Who a commit says it's from](08-commit-identity.md) | Give work and personal folders different identities automatically, catch the wrong one before it's committed, and know what fixing it later costs |

### Reference

| # | File | |
|---|------|--|
| 9 | [Glossary](09-glossary.md) | Look things up |
| 10 | [Exercises & answers](10-exercises.md) | Every "Check yourself" question, with worked answers |

## If you're short on time

- **10 minutes, you just want to push:** file 05.
- **30 minutes:** file 01, then 05, then the summary table in 04.
- **You are about to rebase something scary:** file 06 first (set up the safety net), then 03.
- **Something already went wrong:** file 06, section "Getting it back".
- **New laptop, or you code for work and for yourself on one machine:** file 08, §8.3 and §8.5.

## The one-paragraph summary of everything

Your branch was cut from master at some point; master has moved on. **Rebase** replays your
commits onto master's new tip, which produces a clean straight-line history but *replaces*
every one of your commits with a new one carrying a different SHA. If you had already pushed
the branch, the remote still holds the originals, so git refuses your next push as a
non-fast-forward. The fix is not `git pull` — that merges the old chain back in and gives you
both copies of everything — it is `git push --force-with-lease`, which overwrites the remote
*only if* it is still exactly where you last saw it, so you cannot silently discard a
colleague's push. Along the way git will hand you text conflicts, which are the easy ones;
the dangerous ones are the merges git resolves cleanly and wrongly, because meaning is not
text — a type that changed upstream, an ID someone else claimed, a version number already
released. So a rebase is not finished when `git status` is clean. It is finished when the
thing builds, the tests pass, and `git range-diff` shows you exactly which commits changed
and why.

And separately: the name and email on every commit are plain strings copied from your
config, unverified, and hashed in for good — so set them per folder with a conditional
include, guard them with a check, and remember that only a signature proves who made a
commit.

## How to use me

Ask anything, including:

- "Explain the lease again but simpler"
- "I'm mid-rebase and confused about which side is HEAD"
- "Walk me through what `git rev-list --left-right --count` is telling me"
- "Is it safe to force-push this? Here's my situation…"
- "I think I lost a commit"
- "Why does my IDE say 99 outgoing when I wrote 8?"
- "Which email will a commit in this folder get, and why?"

I'll add files here as we go if a topic earns its own page.
