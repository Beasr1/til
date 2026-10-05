# 10. Exercises & answers

Every "Check yourself" question, with worked answers. Read the answer only after you have
committed to one — the wrong answers here are the ones that cost afternoons.

---

## File 1 — What a commit actually is

**1. Identical contents and message, different parents. Same commit?**
Different. The parent is hashed into the SHA. This is precisely why rebasing — which
changes nothing but parents — produces entirely new commits.

**2. `git commit --amend` on a message typo. What happened to the original?**
Nothing happened *to* it. A new commit was built and the branch now points there. The
original is still in `.git`, unreferenced, findable via `git reflog` until gc collects it
(30 days for unreachable entries).

**3. Why is deleting a 10,000-commit branch as fast as a 1-commit one?**
A branch is a 41-byte file containing one SHA. Deleting it removes the pointer. The commits
are untouched — they simply become unreachable.

**4. `HEAD detached at 3817ce18` mid-rebase. Lost your branch?**
No. Rebase builds the new chain with nothing pointing at it yet, so HEAD points at a raw
commit. When the last commit lands, git moves the branch ref and reattaches. `--abort` at
any point restores everything.

**5. "I moved my commit onto master." What did git do?**
It built a *new* commit with the same tree-change and message on a different parent. The
original still exists. Nothing moved; something was copied.

---

## File 2 — Rebase vs merge

**1. After `git rebase master`, how many commits kept their hash?**
None of the rebased ones — every parent changed. (A commit already on master is skipped
rather than rebased, but that is not "yours" any more.)

**2. Colleague based work on your branch; you rebase and force-push.**
They must rebase their work onto your new commits — `git rebase --onto <your-new-tip>
<your-old-tip> their-branch`. Whose fault: yours, for rebasing commits someone had built
on. That is the line the golden rule draws, and pushing is not it.

**3. Why does a merge commit have two parents?**
Because it joins two histories, and the parent list is how a commit records what came
before. Two ancestries → two parents.

**4. "Behind its remote counterpart" after a rebase — accurate?**
No. You are diverged: neither tip is an ancestor of the other. Git emits that hint for any
non-fast-forward because it cannot distinguish the cases.

**5. Three weekly merge commits vs a rebase — what does a reviewer see?**
A branching graph where their diff includes master's changes tangled with yours, plus
three commits whose content is "I synced". Rebased, they see only your commits, each
diffed against code that will actually ship.

---

## File 3 — A real rebase

**1. `<<<<<<< HEAD` mid-rebase — yours or upstream's?**
Upstream's — the side you are rebasing onto. Your commit is the labelled side below the
`=======`. The reverse of a merge.

**2. `modify/delete`, you `git rm` and continue. What might you have lost?**
Your edit. The file is gone, but the *intent* of your change usually still applies at
whatever path upstream moved the code to. Deleting resolves the conflict and silently
drops the work.

**3. Why not hand-merge `Cargo.lock`?**
It is generated, and internally consistent by construction: hashes, resolved versions and
the dependency graph must agree. A hand-merge can produce a file that parses, looks
plausible, and describes a tree that cannot exist. Take one side and regenerate.

**4. `range-diff` shows `<` on one of your commits.**
It exists in the old range and not the new one: the rebase dropped it. Usually
`--skip` pressed in haste, or a commit git thought was already upstream. Recover it from
the backup branch and cherry-pick it back.

**5. Clean rebase, build fails on a dependency feature that "does not exist".**
Submodules. The rebase moved the recorded gitlink; the submodule's working copy is still
on the old commit. `git submodule update --init --recursive`.

**6. The one command that undoes any mid-rebase mess?**
`git rebase --abort`.

---

## File 4 — The conflicts git cannot see

**1. Why no conflict marker for the `RequestId` change?**
The two sides edited different *lines*. Git merges text by line, and both edits applied
cleanly. That the merged result no longer type-checks is a fact about Rust, which git does
not read.

**2. Tip builds but `git bisect` still breaks.**
Rebase commits each replayed commit before anything is compiled, so an intermediate commit
can be broken even though the tip is fine. Fix properly by folding the repair into the
commit that needs it; pragmatically, one repair commit at the tip plus a note.

**3. Duplicate migration `V017`. Why no conflict?**
Two different files were added — `V017__add_index.sql` and `V017__something_else.sql`.
Nothing overlaps textually. What breaks is the migration runner, which sees two migrations
claiming one version, at deploy time.

**4. "Silence in production tooling" — why worse than a red build?**
A red build stops the MR. Silence lets it merge and ship, and the failure surfaces as
"the customer says the fix isn't in the release" days later, with nothing in CI to point
at. Absence of a signal is not a passing signal.

**5. Three shared sequences.**
Any three of: test/scenario IDs, error codes, migration numbers, port numbers, enum
discriminants, protobuf/capnp field numbers, feature flags, fixture ids.

**6. What does a rebase guarantee, in one sentence?**
That your text survives — nothing about the assumptions that text encoded.

---

## File 5 — Force-with-lease

**1. "Non-fast-forward" in one sentence.**
The remote's current commit is not an ancestor of the commit you are pushing, so the
remote pointer cannot slide forward to yours without abandoning commits.

**2. Rejected after a rebase — are you behind?**
No: diverged. The remote holds your pre-rebase chain, you hold the rewritten one, and
neither contains the other.

**3. What does `--force-with-lease` compare, against what?**
The remote branch's current value against your remote-tracking ref (`origin/<branch>`) —
a compare-and-swap. It overwrites only if the remote is still where you last saw it.

**4. How auto-fetch defeats it.**
Colleague pushes → your editor auto-fetches in the background → `origin/<branch>` now
points at their commit → your lease is against *their* value → the compare-and-swap
succeeds → their commit is discarded. The check passed against a ref that had already been
updated behind you.

**5. Which flag closes it?**
`--force-if-includes` (git ≥ 2.30): it also requires the remote tip to be reachable from
your local history — you must have *integrated* their commit, not merely fetched it.

**6. Colleague already pulled the branch you force-pushed.**
`git fetch && git reset --hard origin/<branch>` — after saving anything local they care
about. Tell them; don't let their next `pull` merge the old chain back in.

**7. When is `revert` right instead of a force-push?**
Whenever the history is shared and people have built on it — typically anything on
`master`. Revert adds a commit that undoes the change; nobody's clone is invalidated.

---

## File 6 — Safety nets

**1. Why does a backup branch cost nothing?**
It is one 41-byte file containing a SHA. The commits it references already exist.

**2. `ORIG_HEAD` vs `HEAD@{1}`.**
`ORIG_HEAD` is set only by `rebase`/`merge`/`reset`/`pull` — "before the last big
operation". `HEAD@{1}` is the previous position of HEAD from the reflog, which any
checkout or commit also moves. They often coincide; they are not the same thing.

**3. Colleague force-pushed over your commits — whose reflog helps?**
Yours, if you had those commits locally: they are unreachable but present. Theirs will not
have them unless they had fetched them. Failing both, the server may still hold the objects
— ask before assuming.

**4. Unreachable reflog entry lifetime?**
30 days (`gc.reflogExpireUnreachable`); 90 for reachable ones.

**5. `stash pop` conflicts, you panic and `git stash` again.**
Now you have two entries: the original (the pop failed, so it was kept) and a new one
holding your half-resolved conflict state, quite possibly with markers in the files.
Untangling that is worse than the conflict was.

**6. `range-diff` shows `!` on a commit you had no conflict in.**
Git resolved something automatically — a rename it followed, or context that shifted. Read
that entry. It is exactly the class of silent change file 4 is about.

**7. Risk of `rerere`?**
It faithfully replays a resolution you got *wrong* the first time, without asking. If a
rebase resolves suspiciously easily, verify rather than trusting it.

---

## File 7 — Reading divergence

**1. What does `origin/main` point at?**
Your local cache of where the remote's `main` was at your last fetch/pull/push. Updated
only by those; never live.

**2. 99 outgoing, 8 commits written.**
91 upstream commits + your 8. The comparison is against the *branch's* remote-tracking
ref, which still sits on the pre-rebase base — so every master commit since looks new
relative to it.

**3. "What did I add on this branch"?**
`git log --oneline origin/master..HEAD`. Not the tracking ref, which answers a different
question ("what have I not pushed") and is skewed by any rewrite.

**4. `A..B` vs `A...B` for `log`.**
`A..B` = reachable from B but not A. `A...B` = reachable from either but not both.

**5. `git diff A...B`?**
From the **merge base** of A and B to B — i.e. what B introduced since they diverged,
ignoring what A did meanwhile.

**6. `4  3` from `--left-right --count`.**
Diverged: 4 commits only yours, 3 only the remote's. Next, look at the three:
`git log --oneline @{upstream} --not HEAD`. Your own pre-rebase commits → force-with-lease.
Someone else's → integrate, never force.

**7. Content is on master but `--is-ancestor` says no.**
It arrived by cherry-pick, squash-merge or rebase — same change, different commit object.
Ancestry is by identity, not by content.

---

## File 8 — Who a commit says it's from

**1. Cherry-picked, then rebased: what do `author` and `committer` say?**
Author: your colleague — both operations copy the author across. Committer: you — each one
built a new commit object, and the committer is whoever did that. They differ because they
answer different questions: who wrote the change, and who last made this commit.
`git log --format=fuller` shows both.

**2. `gitdir:~/code/personal` (no trailing slash), wrong email in a repo below it.**
Without a trailing `/` the pattern matches a `.git` directory at exactly that path, not
anything underneath. The repo's `.git` is at `~/code/personal/site/.git`, which doesn't
match, so the include never fires and the global default wins. Run inside the repo,
`git config --show-origin user.email` shows the value coming from `~/.gitconfig` rather
than the personal file. Fix: `gitdir:~/code/personal/`.

**3. Include moved above `[user]`, work address is back.**
An include is spliced in at the point where it appears, and for a single-valued key the
last value read wins. Now git reads the personal email first, then the `[user]` block's
work email, which overrides it. Nothing is malformed, so nothing complains.
`--get-all --show-origin` shows both values in read order, with the wrong one last.

**4. `hasconfig:remote.*.url:`, three commits before `git remote add`.**
The default (work) address — the condition looks for a matching remote URL and there
wasn't one yet. Two setups would have caught the first commit: a folder rule (`gitdir:`)
for wherever you start new projects, or no default email at all plus
`user.useConfigOnly = true`, so the first commit fails with "no email was given" and you
have to choose.

**5. Forty public commits, someone suggests `.mailmap`.**
It fixes how *git* shows them: `git log` and `git shortlog` display the canonical identity.
It leaves every commit object untouched — the work address is still in each one, in every
clone and both forks, and I found no GitHub documentation saying it uses `.mailmap` for
attribution. A real fix is a full rewrite (`git filter-repo --mailmap`): new SHAs from the
first affected commit on, a force-push of every branch and tag, and both friends having to
reset or reclone (file 5 §5.5). Their forks keep the old commits whatever you do.

**6. Name, work email, and Verified. What does each establish?**
The name and email establish nothing — they're strings anyone can set. The badge
establishes that the commit's signature checked out against a signing key registered to a
GitHub account. It doesn't make the email the *right* one (signing doesn't choose your
address), and it identifies the key, not the person at the keyboard.

---

## Things to try

1. **Make a rebase conflict on purpose.** Two branches editing the same line, rebase one
   onto the other, and read the markers until you can say without hesitating which side is
   yours.
2. **Break a rebase and recover three ways.** `--abort` mid-flight; `reset --hard
   ORIG_HEAD` after it finishes; `reflog` after two more commits on top.
3. **Prove `range-diff` catches a drop.** Rebase a 3-commit branch, `--skip` the middle
   one deliberately, then find it with `range-diff` against your backup ref.
4. **Defeat your own lease.** In one clone push a commit; in another (with the branch
   checked out) `git fetch`, then `--force-with-lease`. Watch it discard the commit.
   Then repeat with `--force-if-includes` and watch it refuse.
5. **Allocate a colliding number.** Add a migration `V042` on a branch, add a different
   `V042` on master, rebase. Note that git says nothing at all.
6. **Watch an include fail silently.** In a fresh shell, `export HOME=$(mktemp -d)` so
   your real config is untouched. Set a default email and a
   `gitdir:` include for one folder. Confirm with `--show-origin` inside a repo there; then
   delete the trailing slash, then move the include above `[user]`, and watch the address
   flip back each time with no error.
