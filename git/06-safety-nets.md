# 6. Safety nets

The reason experienced people rebase casually is not steadier hands. It is that they know
git almost never deletes anything, and they know the four commands that get it back.

## 6.1 Before: two seconds of insurance

```sh
git branch backup/<branch>-prerebase        # a name you can diff against
git stash push -m "what this actually is"   # not "WIP on master" for the eleventh time
```

The backup branch costs 41 bytes and turns the worst case into a `git reset --hard
backup/…`. Do it every time; you will use it once a year and be glad.

## 6.2 During: abort is total

```sh
git rebase --abort
```

Puts the branch pointer, the working tree and the index back exactly as they were before
you started. There is no partial state to clean up. If a rebase is going badly — six
conflicts in and you have lost the thread — aborting and restarting with a smaller step
(`git rebase --onto`, or rebasing in two hops) is almost always faster than pushing
through.

## 6.3 After: the reflog

Every time a ref moves, git writes a line recording where it was:

```sh
$ git reflog
4f38a3a HEAD@{0}: rebase (finish): returning to refs/heads/feat/env-gate
d030ea9 HEAD@{1}: rebase (pick): chore(release): bump the artifacts…
a61be94 HEAD@{2}: commit: feat(core): stamp request provenance…
```

`HEAD@{2}` is a real revision — pass it anywhere a SHA goes. So the universal undo is:

```sh
git reset --hard HEAD@{2}            # I want to be where I was two moves ago
git branch rescue a61be947           # or: give the old tip a name and inspect it
```

Two things to know about its limits:

- **The reflog is local and per-clone.** It is not pushed, not fetched, and a fresh clone
  has none. Your colleague's mistake is not recoverable from your reflog.
- **It expires.** 90 days for reachable entries, **30 days** for unreachable ones
  (`gc.reflogExpire` / `gc.reflogExpireUnreachable`). "Git never loses anything" has a
  calendar attached.

`ORIG_HEAD` is the same idea for one step: `rebase`, `merge`, `reset` and `pull` all set
it to where you were before, so `git reset --hard ORIG_HEAD` undoes the last one of those.

## 6.4 After: did the rebase preserve everything?

`git range-diff` is the verifier, and the reason to keep that backup branch:

```sh
git range-diff <old-base>..backup/mybranch-prerebase <new-base>..HEAD
```

Markers: `=` unchanged · `!` content changed · `>` only in the new range · `<` **only in
the old range — a dropped commit**.

The check that matters: the number of `!` lines should equal the number of conflicts you
resolved, and there should be no `<`. If a commit changed that you did *not* touch, read
that entry — git resolved something for you and you should know what.

⚠️ Get the ranges right. `range-diff A...B` (three dots) means something else entirely and
will show you ninety unrelated commits. The form you want is two explicit ranges:
`old-base..old-tip new-base..new-tip`. (This is the same two-dot/three-dot trap as
[file 7](07-reading-divergence.md) §7.5.)

## 6.5 Stashes across a rewrite

A stash is a commit too (two, actually — index and worktree), which is why it survives
things it has no right to.

- **A failed `git stash pop` keeps the entry.** Git tells you: *"The stash entry is kept
  in case you need it again."* Do not re-stash in a panic; you will end up with two.
- **`pop` conflicts if upstream deleted or moved the file.** Expected after a rebase that
  pulled in a refactor.
- **You can read a stash's version of a path that no longer exists:**

```sh
git diff 'stash@{0}^' 'stash@{0}' -- crates/services/aegis-service/src/usb/
```

  That is how you recover work aimed at a file upstream moved: read the diff, apply it by
  hand at the new location.
- **`git stash list` is a graveyard.** Sixteen entries called `WIP on master` are sixteen
  things you will never dare delete. Always `-m`.

## 6.6 Repeated conflicts: rerere

If you rebase the same branch repeatedly (or rebase, abort, rebase), you resolve identical
conflicts over and over. Git can memorise them:

```sh
git config --global rerere.enabled true
```

*Reuse recorded resolution*: it hashes the conflict, remembers what you did, and replays
it next time the same conflict appears. Worth turning on permanently; the one caveat is
that it will silently re-apply a resolution you got *wrong* the first time, so if a rebase
resolves suspiciously easily, check the result.

## 6.7 The order to try things

1. Still mid-rebase? → `git rebase --abort`
2. Just did the wrong thing? → `git reset --hard ORIG_HEAD`
3. Several steps back? → `git reflog`, find it, `git reset --hard HEAD@{n}`
4. Lost a commit entirely? → `git fsck --lost-found`, or the backup branch you made
5. Force-pushed over someone? → *their* reflog, or the server's (GitLab/GitHub keep
   unreachable objects for a while; ask an admin before you assume it is gone)

---

## Check yourself

1. Why does a backup branch cost effectively nothing?
2. What is the difference between `ORIG_HEAD` and `HEAD@{1}`?
3. Your colleague force-pushed over your commits. Can your reflog recover them? Can theirs?
4. How long does git keep an unreachable reflog entry by default?
5. A `git stash pop` conflicts and you panic and run `git stash` again. What have you
   just done?
6. `range-diff` shows a `!` on a commit you never had a conflict in. What does that tell
   you, and what should you do?
7. What is the risk of enabling `rerere`?
