# Rewriting History Safely — A Git Course

A course on the half of git that scares people: rebasing, force-pushing, and reading the
state you end up in afterwards — plus one chapter on the identity stamped into every commit
you make, because that is history too, and just as hard to take back. Two later chapters
cover the merges you can't simply take back (reverting one, and the environment branches
that make it necessary) and the byte-level habits that keep diffs, blame and renames
honest: line endings, formatting commits, and rename detection.

**This is reference learning material.** The concepts are general git. The worked example
running through it is real (names changed) — a rebase of `acme/ledger`'s `feat/env-gate` onto a
master that had moved 91 commits ahead, done on 2026-08-24 — because an abstract rebase
teaches you nothing and a real one teaches you the four things that actually go wrong.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- I use the mental model before the commands, and *why* before *how*.
- Every file ends with **Check yourself** questions. Answers are in `12-exercises.md`.
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

### Part 5 — Merges you can't take back, and bytes you didn't mean to change

Added later, so they carry the next free numbers. File 09 belongs right after file 04 in
spirit: it needs files 01 and 02 (snapshots, parents, merge commits), and its last section,
on environment branches, stands alone. File 10 needs only file 01 §1.1. Read either when
the problem in its first paragraph sounds familiar.

| # | File | After this you can… |
|---|------|---------------------|
| 9 | [Reverting a merge](09-reverting-a-merge.md) | Explain why a reverted branch won't come back when merged again, pick the right recovery, and argue for one trunk with flags over branch-per-environment |
| 10 | [Attributes, line endings and blame](10-attributes-line-endings-and-blame.md) | Pin line endings in `.gitattributes`, renormalise a mixed repo, keep formatting sweeps out of blame, and land renames so git can follow them |

### Reference

| # | File | |
|---|------|--|
| 11 | [Glossary](11-glossary.md) | Look things up |
| 12 | [Exercises & answers](12-exercises.md) | Every "Check yourself" question, with worked answers |

## The primary sources

The chapters cite these throughout. Where a chapter quotes behaviour, it's from one of them.

| Source | What it settles |
|---|---|
| [git reference manual](https://git-scm.com/docs) — `git-revert`, `git-rebase`, `git-merge`, `git-diff`, `git-blame`, `git-config`, `gitattributes`, `gitignore` | What each command and setting actually does |
| [How to revert a faulty merge](https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.adoc) | Why a reverted merge is sticky, in Linus Torvalds' and Junio Hamano's words |
| [Pro Git](https://git-scm.com/book/en/v2), §3.4 and §7.8 | Long-running branches; undoing merges |
| [GitHub Docs: removing sensitive data](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) | What a history rewrite after a leak does and doesn't fix |
| [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) and [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html) | The case for one trunk, and flags instead of branches |

## If you're short on time

- **10 minutes, you just want to push:** file 05.
- **30 minutes:** file 01, then 05, then the summary table in 04.
- **You are about to rebase something scary:** file 06 first (set up the safety net), then 03.
- **Something already went wrong:** file 06, §6.7.
- **New laptop, or you code for work and for yourself on one machine:** file 08, §8.3 and §8.5.
- **You reverted a merge and the feature won't come back:** file 09, §9.3 and §9.5.
- **A diff is far bigger than the change, or blame points at a reformat:** file 10, §10.1 and §10.6.
- **A secret got committed:** file 06, §6.8. Rotate first, then read the rest.

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

Merges have the same blind spot as rebases, so build and test the *merge result*, not just
the branch. And a merge can't be taken back: reverting one removes its content but leaves
its ancestry, so merging the same branch again brings nothing back until you revert the
revert, or re-land the work as fresh commits. Most reverted merges come from keeping one
branch per environment, which is better replaced by one trunk, configuration per
environment, and runtime flags. Finally, git compares bytes and infers the rest, so commit
a `.gitattributes` that pins line endings, keep formatter-only changes and renames in
commits of their own, and treat any committed secret as leaked: rotate it first.

## How to use me

Ask anything, including:

- "Explain the lease again but simpler"
- "I'm mid-rebase and confused about which side is HEAD"
- "Walk me through what `git rev-list --left-right --count` is telling me"
- "Is it safe to force-push this? Here's my situation…"
- "I think I lost a commit"
- "Why does my IDE say 99 outgoing when I wrote 8?"
- "Which email will a commit in this folder get, and why?"
- "We reverted a merge last week. How do we re-land it?"
- "Why is this diff 1,300 lines when I changed 40?"
- "A password got committed. What now?"

I'll add files here as we go if a topic earns its own page.
