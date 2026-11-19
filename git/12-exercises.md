# 12. Exercises & answers

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

**7. Cherry-picked and rebased copies merge clean, then fail with a redeclaration.**
Git only compares overlapping regions. Each copy added the constant at a different place in
the file, and relative to the merge base each addition sat in an area the other side hadn't
touched, so both were "incorporated in the final result verbatim". Git has no idea the two
copies are the same change: a cherry-pick is a new commit with a new SHA, linked to the
original by nothing. What would have caught it is a pipeline that builds the *merge result*
rather than the branch: GitLab's merged results pipelines, or a GitHub Actions
`pull_request` workflow, which runs on the `refs/pull/<n>/merge` commit. Each branch was
green on its own.

**8. `node --check` and a container build pass; a merge dropped a named export.**
The service fails to start at all. With native ES modules, an import of a name the target
module doesn't export is a `SyntaxError` thrown while the module graph is *linked*, which
happens before any module's code runs, so it's the whole process rather than the one route
that uses the name. `node --check` only parses, so it passes, and a container build only
copies files. The cheapest catch is one CI step that loads the program:
`node --input-type=module -e "await import('./server.js')"` (or the module that builds the
app, if importing the entry point starts listening). It exits non-zero on the bad import.

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

**8. `.env` committed six months ago, then `git rm --cached` and `.gitignore`.**
The password is in every commit from the one that added `.env` up to the one that untracked
it. Anyone who has cloned or fetched since then has it: teammates, CI runners, any fork, any
backup. `.gitignore` never affected it ("Files already tracked by Git are not affected"),
and `rm --cached` only stopped later commits from carrying it. So the first thing is to
rotate the password: issue a new one, deploy it, and revoke the old one. After that the copy
in history is a dead string. Then decide separately whether the history needs scrubbing.

**9. When is a rewrite worth it, and what doesn't it fix?**
When the repository itself must not contain the data. Personal data is the obvious case, or
a private repository that's about to go public. For an ordinary credential that's been
rotated, GitHub's own guide says the rewrite "may not be warranted". It doesn't fix: clones
and forks that already have the commits; any copy cached by the host until support removes
it; the secret being valid, which only rotation fixes; and the risk of one push from an old
clone putting it all back. It also changes every SHA from the first affected commit, which
breaks signatures and costs everyone a reset or a reclone.

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

## File 9 — Reverting a merge

**1. `git revert -m 1 M`, then `git merge topic` with no new commits.**
Nothing happens: "Already up-to-date." Topic's tip `B` is already in main's history (it's
`M`'s second parent), so the merge base is `B` itself and topic has nothing main doesn't
already reach. Git never looks at content to decide this. `W` took the content out, but
the ancestry still says it was merged.

**2. A fix `C` is added to topic and merged.**
Only `C`'s change arrives. The three snapshots are the base (`B`), main's tip, and topic's
tip (`C`). Topic's side since `B` is just `C`. Main's side since `B` includes `W`, which
deleted `A`'s and `B`'s work, and git keeps both sides' changes. So main ends up with a fix
for a feature it doesn't have. The fix is to revert `W` *first*, then merge.

**3. `branch --merged` lists topic, but the feature isn't on main.**
`--merged` asks whether topic's tip is reachable from main, which is about ancestry, and it
is, through `M`. Whether the content is there is a different question, and `W` changed the
answer. `git diff main topic` (or `--stat`) compares content tip to tip and shows the
feature missing. Ancestry answers "was it merged?" Only a diff answers "is it here?"

**4. Why `-m` for a merge but not for an ordinary commit?**
Reverting means applying the inverse of a commit's change, and a change is always relative
to a parent. An ordinary commit has one parent, so there's one answer. A merge has two, and
its change relative to each is different: relative to parent 1 it's "everything topic
brought in"; relative to parent 2 it's "everything main did meanwhile". `-m 1` picks the
first, which is almost always what you mean.

**5. Wrong-direction merge into a release branch, reverted. Why not revert the revert?**
Because `W` removed exactly the thing you must not ship: every unreleased change on the
integration branch. Reverting `W` puts all of it back on the release branch. And the
release branch's history still claims all those commits as merged, so when they're
promoted properly later they won't arrive. The branch itself is polluted. Recreate it from
the right base (`git switch -c feature-v2 origin/release`), cherry-pick only the feature's
own commits, open a new merge request, and close the old one with a note.

**6. Dev, staging and production branches, every feature merged into each.**
Any two of these. A conflict resolved by hand on one branch picks the stale side and
silently undoes an earlier fix, and with three or four branches the same conflict gets
resolved three or four times by different people. A revert on one branch travels into
another the next time those branches are merged, and then later merges of the original work
bring nothing (§9.3). A per-branch edit, like a route commented out on staging only, gets
"resolved" away by a merge in either direction. Replace it with one trunk: environments
differ by configuration (per-environment inventories or variable files on the same branch),
unfinished features hide behind runtime flags, and a release is a tagged commit deployed to
staging and then the same tag to production. Release fixes go to trunk first and are
cherry-picked to the release branch.

---

## File 10 — Attributes, line endings and blame

**1. 1,350 changed lines, 45 without carriage returns.**
The branch rewrote those files' line endings from LF to CRLF, so every line differs by one
byte, and 45 lines changed for real. Beyond review, the commit really does change every
line. Any other branch that edits those files now overlaps it everywhere and conflicts,
and `git blame` attributes 1,300 lines to a commit that didn't touch their meaning. Send it
back and have the endings restored, rather than merging and fixing later.

**2. No `.gitattributes`; Windows colleague with `core.autocrlf=true`; you with nothing set
and a CRLF editor.**
Your colleague's commits contain LF: `autocrlf=true` acts like `text=auto` on every file, so
text is normalised to LF when it's added, and converted to CRLF only in their working tree.
Yours contain CRLF: with no `text` attribute and no `core.autocrlf`, nothing converts, so
whatever your editor wrote goes in as-is. You're the one producing the whole-file rewrite,
on the "safe" platform. A committed `* text=auto` (or `text=auto eol=lf`) would have
normalised your commits regardless of your config.

**3. `* text=auto eol=lf` committed, but old CRLF files stay CRLF.**
`text=auto` deliberately leaves alone a file that's "already in Git with CRLF endings". The
attributes protect new work only. To convert the existing files, from a clean tree:
`git add --renormalize .`, check `git status`, and commit that on its own. Then list the
commit in `.git-blame-ignore-revs`, and tell anyone with an older branch that
`git merge -X renormalize` exists for exactly this.

**4. Why add the SHA only after the formatting commit lands; what does squash-merge break?**
The ignore file lists commit IDs, and a rebase or squash makes a new commit with a new ID,
so a SHA written down before the merge may not exist on the main branch afterwards. If the
host squash-merges a merge request that has the formatting commit and a feature commit in
it, they become one commit. Ignoring that in blame would hide the feature too, and not
ignoring it leaves every reformatted line blamed on the feature. Give the sweep its own
merge request, and add its SHA in a follow-up once it's on the main branch.

**5. Move plus heavy rewrite in one commit, teammate edits the old path.**
Git doesn't record the move. It pairs your deleted and added paths only if they're at least
50% similar (the default). If your rewrite keeps it above that, the merge follows the rename
and applies your teammate's edits at the new path, conflicting wherever they overlap your
rewrite, which is most places. Below it, git sees a deletion plus an unrelated new file, and
your teammate's work becomes a `modify/delete` conflict with nowhere to go automatically.
Split it into a pure move first (100% similar, `R100` in `--name-status`), landed quickly,
then the edits. Now the merge follows the rename for certain, and any conflicts are
ordinary edit-against-edit ones at the new path.

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
7. **Make a revert sticky.** In a scratch repo, merge a two-commit branch with `--no-ff`,
   revert the merge with `-m 1`, then merge the branch again and read the output. Add one
   commit to the branch and merge once more; check with `git diff` which of the three
   changes made it. Then revert the revert and watch the original two come back.
8. **Rebuild instead.** Repeat exercise 7 up to the revert, then `git rebase --no-ff` the
   branch onto its own base and merge it. Compare the result with the revert-the-revert
   version.
9. **Make the CRLF diff.** Commit a file with LF, convert it to CRLF
   (`perl -pi -e 's/\n/\r\n/' file` works on macOS and Linux alike), and compare
   `git diff --stat` with `git diff --ignore-cr-at-eol`. Then commit `* text=auto eol=lf`
   and `git add --renormalize .`, and see the change disappear.
10. **Push a rename under the threshold.** Move a file and rewrite about two thirds of it in
    one commit, while another branch edits the original. Merge and read the conflict. Then
    redo it as a pure move plus a separate edit, and compare.

---

## Questions worth asking me

- "Here's my branch graph after a revert. Which recovery from file 9 §9.5 fits?"
- "We keep one branch per environment. What would moving to one trunk take, step by step?"
- "Should this repository pin `eol=lf`, or just `text=auto`?"
- "Which commits belong in our `.git-blame-ignore-revs`?"
- "A secret went into our history. Walk me through what to rotate, and whether to rewrite."
- "Why did this merge go wrong when git reported no conflict?"
