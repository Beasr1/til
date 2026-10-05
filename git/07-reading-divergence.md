# 7. Reading divergence

Your IDE says **↑99 ↓6**. You wrote eight commits. This file is why those numbers are
both correct and how to ask git a question that means what you think it means.

## 7.1 Three refs, easily confused

| Ref | What it is | Updated by |
|---|---|---|
| `feat/x` | your local branch | your commits |
| `origin/feat/x` | **your local cache** of where the remote's branch was, last time you looked | `fetch`, `pull`, `push` |
| the actual remote branch | the truth, on the server | anyone |

The middle one is the one people forget exists. `origin/feat/x` is not a live view — it is
a snapshot on your disk, and it is stale from the moment anyone else pushes. Every "how far
ahead am I" number your tools show is computed against that snapshot, which is why
`git fetch` changes numbers without changing any code.

To see the true remote state without touching anything local:

```sh
git ls-remote --heads origin feat/env-gate
# a61be9471af49f155e2a5807dcdb8a646eedeea5   refs/heads/feat/env-gate
```

## 7.2 The upstream, and what "outgoing" compares

A branch may have an **upstream** (tracking) ref, set by `push -u` or by cloning:

```sh
git rev-parse --abbrev-ref --symbolic-full-name '@{upstream}'
# origin/feat/env-gate
```

That is what IDEs mean by "outgoing" and "incoming": commits you have that the upstream
snapshot doesn't, and vice versa. **Not** "commits not on master". This single confusion
accounts for most surprised faces.

## 7.3 Counting properly

```sh
git rev-list --left-right --count HEAD...origin/feat/env-gate
# 99      6
#  ^       ^
#  |       └── only on the right side (the remote): 6
#  └────────── only on the left side (me): 99
```

The real case: the branch had been pushed *before* being rebased onto a master that was 91
commits ahead. So:

```
99 outgoing = 91 master commits + 8 of mine
 6 incoming = my own 6 pre-rebase commits, now superseded
```

The 91 look outgoing because the comparison is against the **branch's** remote snapshot,
which still sits on the old base — relative to *it*, every master commit since is new.
They are already on `origin/master`; they are not "mine" in any meaningful sense.

## 7.4 The two questions actually worth asking

**"What did I write?"** — compare against the integration branch, not the tracking ref:

```sh
git log --oneline origin/master..HEAD          # 8 commits. The honest answer.
```

**"Is this commit already on master?"** — ask directly rather than eyeballing a graph:

```sh
git merge-base --is-ancestor <sha> master && echo yes || echo no
```

Exit status, so it scripts. Useful when a cherry-pick may have landed the same change
under a different hash (it will say `no` — same content, different commit; see §1).

## 7.5 Two dots vs three dots

The single most common way to ask git the wrong question.

| Form | For `log` / `rev-list` | For `diff` |
|---|---|---|
| `A..B` | commits reachable from B but not A | diff A's tip against B's tip |
| `A...B` | commits reachable from either **but not both** (symmetric difference) | diff from the **merge base** of A and B to B |

Note the cruel part: for `log` the three-dot form is *symmetric*, and for `diff` it is not
— it means "changes B introduced since they diverged". Same punctuation, different
meaning, depending on the command.

Two idioms worth memorising because they are always what you want:

```sh
git log --oneline master..HEAD               # what my branch adds
git diff master...HEAD                       # what my branch changed, ignoring master's moves
```

And where it bites: `git range-diff A...B` is neither of those — it wants two ranges. Give
it a three-dot expression and it happily reports ninety unrelated upstream commits as
"new", which looks alarming and means nothing. Correct form:

```sh
git range-diff <old-base>..<old-tip> <new-base>..<new-tip>
```

## 7.6 Reading a diverged state

Diverged means: neither tip is an ancestor of the other. Confirm before deciding:

```sh
git rev-list --left-right --count HEAD...@{upstream}
```

- `N  0` → you are simply ahead. Plain `push` works.
- `0  N` → simply behind. `pull` (ideally `--rebase`) works.
- `N  M` → **diverged**, both non-zero. Now decide *why*:
  - you rebased and the remote holds your old chain → force-with-lease ([file 5](05-force-with-lease.md))
  - someone else pushed real work → integrate it; never force
  - both → integrate theirs first, then force-with-lease, and check the leased value by hand

The numbers do not tell you which. Look at the commits:

```sh
git log --oneline @{upstream} --not HEAD      # what's on the remote that I don't have
```

If every line is a message you recognise as your own pre-rebase work, it is case one. If
someone else's name is in there, it is not.

---

## Check yourself

1. What exactly does `origin/main` point at, and what updates it?
2. Your IDE says 99 outgoing but you wrote 8 commits. Give the arithmetic.
3. Which command answers "what did I add on this branch" — and why not the tracking ref?
4. `git log A..B` vs `git log A...B` — difference in one sentence.
5. `git diff A...B` — what is it diffing, exactly?
6. `git rev-list --left-right --count HEAD...@{upstream}` prints `4  3`. What state is
   that, and what do you check next?
7. A commit's content is on master but `merge-base --is-ancestor` says no. Explain.
