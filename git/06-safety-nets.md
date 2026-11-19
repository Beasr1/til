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

## 6.8 The reverse problem: a secret you committed

Everything above works because git keeps things. That same property is the problem when what
you committed is a password, a private key or a token. Now you want something gone, and git's
whole design is about not losing it.

**Deleting it in a later commit doesn't remove it.** Commits are snapshots
([file 1](01-what-a-commit-is.md) §1.1), so every commit made while the file was tracked still
contains it, and every clone and fork has those commits. `.gitignore` doesn't help either:
"Files already tracked by Git are not affected" ([gitignore](https://git-scm.com/docs/gitignore)).
The documented fix, `git rm --cached` and then a `.gitignore` entry, is there "to stop the file
from being reintroduced in later commits". Later ones only.

```sh
git rm --cached .env             # stop tracking it from the next commit on
echo ".env" >> .gitignore        # and stop it coming back
git log --all --oneline -- .env  # every commit that still holds a copy
```

That fixes the future. The past still holds the value.

**So rotate first.** GitHub's guide to removing sensitive data starts there: "as a first step
you need to revoke and/or rotate that secret", and once you have, "Going through the extra
steps to rewrite the history and remove the secret may not be warranted"
([GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)).
OWASP agrees: "Keys that were exposed should undergo immediate revocation"
([Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)).
A rotated secret in history is a dead string. An unrotated one is a live credential you've
hidden from yourself and nobody else. That holds for a private repository too: everyone with
read access, every CI runner and every laptop that ever cloned it has a copy, and you can't
list them.

**Rewrite history only when the repository itself must be scrubbed**, for example personal
data that has to go, or a private repository about to be made public. The tool is
`git filter-repo`, which "is now recommended by the git project" over `filter-branch`
([git-filter-repo](https://github.com/newren/git-filter-repo)). GitHub's guide uses its
`--sensitive-data-removal` flag, which needs filter-repo 2.47 or later:

```sh
git filter-repo --sensitive-data-removal --invert-paths --path config/.env   # drop a file from every commit
git filter-repo --sensitive-data-removal --replace-text ../passwords.txt     # or replace strings everywhere
```

Know the bill before you start. From the same GitHub guide:

| Cost | Why |
|---|---|
| Every SHA from the first affected commit on changes | It rebuilds history ([file 1](01-what-a-commit-is.md) §1.2), so you force-push every branch and tag ([file 5](05-force-with-lease.md)) and everyone resets or reclones |
| Clones and forks keep the old commits | "You cannot remove sensitive data from other users' clones of your repository", and a fork that has the commit keeps it reachable |
| Signatures stop verifying | "signatures would no longer be valid", because the signed commits are gone |
| Open pull requests | GitHub recommends merging or closing them first |
| The host's copies | GitHub Support has to remove cached views and dereference affected pull requests |
| Recontamination | One push from an old clone brings it all back |

It's the same bill as rewriting an email address ([file 8](08-commit-identity.md) §8.6), for
the same reason.

**Prevention is cheaper than either.**

- Reference secrets from wherever they live (a vault, the CI system's secret variables, an
  encrypted file) from the very first commit. "I'll move it to secrets later" is how a literal
  key ends up in a tracked template.
- Example config holds obvious placeholders and nothing else. A `.env.example` gets copied
  everywhere, and a real-looking value in it ends up in production or in someone's fork:

  ```sh
  # .env.example: every value is fake, and looks fake
  DATABASE_URL=postgres://app:CHANGE_ME@localhost:5432/orders
  SIGNING_KEY=replace-with-a-key-from-the-secret-store
  ```

- Scan for secrets before the commit and again in CI. GitHub suggests "a well-known tool in a
  pre-commit hook like git-secrets or gitleaks", and push protection, which blocks pushes
  "which contain hardcoded secrets". A pre-commit hook lives in one clone and can be skipped,
  so the CI scan is the one that counts.

If the leaked secret was a signing key, rotation has an extra twist, because signatures it
already made can't be dated: see the signatures course,
[file 04 §4.6](../signatures/04-signing-in-automation.md).

> **Teacher's aside.** "We removed it in a later commit" is a statement about the tip of one
> branch. A leak is a statement about history, and history is the one thing this whole
> chapter has been telling you git refuses to lose. The question after a leak is never "is
> it still in the repo?" It's "has the credential stopped working?"

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
8. Six months ago a teammate committed `.env` with a database password, then ran
   `git rm --cached .env` and added it to `.gitignore`. What is still exposed, to whom, and
   what do you do first?
9. When is rewriting history worth it after a leak, and name two things the rewrite still
   doesn't fix.
