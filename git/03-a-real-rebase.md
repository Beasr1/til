# 3. A real rebase, start to finish

The worked example is `acme/ledger`, 2026-08-24: a six-commit feature branch cut from a
master that had since moved **91 commits** ahead, including a crate rename and a
release. Every conflict below is one that actually happened.

## 3.1 Preflight — four things before you type `rebase`

**1. Know how far behind you are.** Fetch first; your local `master` is a stale copy too.

```sh
git fetch origin --prune
git rev-list --left-right --count master...origin/master
# 0    91          ← 0 commits only I have, 91 only the remote has
```

**2. Get a clean tree.** Rebase refuses to run with modified tracked files. Stash them
*with a message* — a stash list of eleven `WIP on master` entries is useless:

```sh
git stash push -m "pre-rebase: aegis iso diagnostics, wasm-trace, dev keys"
```

**3. Make a backup ref.** One command, free, and it turns "I destroyed my branch" into a
non-event:

```sh
git branch backup/env-gate-prerebase
```

The reflog (§6) would also save you, but a named branch is something you can `git diff`
against without decoding `HEAD@{17}`.

**4. Move your base to where you're rebasing onto.** Local `master` was 0 ahead / 91
behind, so fast-forwarding it is safe:

```sh
git branch -f master origin/master
```

## 3.2 Run it

```sh
git rebase master
```

Git replays your commits oldest-first. Each one either applies cleanly or stops with a
conflict, at which point you are mid-rebase with a detached HEAD (§1.3) and exactly three
options:

| Command | Meaning |
|---|---|
| `git rebase --continue` | I resolved it; carry on |
| `git rebase --skip` | Drop this commit entirely — upstream already has the change |
| `git rebase --abort` | Put everything back exactly as it was. Always available. |

`--continue` opens an editor for the commit message. To keep it non-interactive:

```sh
GIT_EDITOR=true git rebase --continue
```

## 3.3 The four conflict shapes

### (a) Both sides edited the same lines

The ordinary one. An import list where upstream added two names and you added two others:

```rust
<<<<<<< HEAD
    CancelPayload, CheckAccountStatusPayload, DefaultTimeouts, …, RequestId,
=======
    CheckAccountStatusPayload, DefaultTimeouts, RegisterSandboxDevicePayload, …,
>>>>>>> 592654c9 (feat(sandbox): sync an account…)
```

Resolution is the union of both. **Which side is which** trips everyone up, so learn it
once:

> During a **rebase**, `HEAD` is the side you are rebasing *onto* — upstream. The labelled
> side is *your* commit. This is the opposite way round from a merge, because rebase
> replays your work on top of theirs, so "ours" and "theirs" swap.

### (b) `modify/delete` — upstream moved the file

```
CONFLICT (modify/delete): crates/bindings/web-host/Cargo.toml
  deleted in HEAD and modified in 3257ffb4
```

Upstream had moved that crate to `crates/services/web-host`. Your commit edited the file
at its old path. Nothing to merge — the file is gone.

Resolution is two steps, and only the first is obvious: `git rm` the dead path, **then
re-apply your intent at the new one**. Here that meant taking the version bump that had
been applied to the old manifest and putting it on the new one. Delete without the second
half and the rebase completes green with your change silently dropped.

### (c) Rename-following — read the markers, they tell you

```
<<<<<<< HEAD:crates/services/web-host/src/handler.rs
…
>>>>>>> 592654c9 (…):crates/bindings/web-host-core/src/handler.rs
```

Git detected the rename and merged your edit into the file's new home. The two paths after
the colons are the clue. This is a *good* outcome — but note that git matched the file by
content similarity, not by knowledge. A heavily-edited moved file will not be detected,
and you get shape (b) instead.

### (d) Lock files — never hand-merge

`Cargo.lock`, `package-lock.json`, `Gemfile.lock`. These are generated. Merging them by
hand produces a file that is valid-looking and wrong. Take one side wholesale and
regenerate:

```sh
git show master:Cargo.lock > Cargo.lock   # take upstream's
cargo metadata --format-version 1 >/dev/null   # let the tool rewrite it
git add Cargo.lock
```

## 3.4 The ambush: submodules

Regenerating the lock failed with something that had nothing to do with the rebase:

```
error: failed to select a version for `cipherkit`.
  package `ledger-logging` depends on `cipherkit` with feature `age-async`
  but `cipherkit` does not have that feature.
```

The 91 upstream commits included a **submodule pointer bump**. Checking out master's tree
updated the recorded gitlink, but the submodule's own working directory stays where it
was until you say so:

```sh
git submodule update --init --recursive external/cipherkit
```

The general lesson: a submodule pin is content in the superproject, so it rebases like any
other change — but the checkout is a separate step git will not do for you. If a build
breaks after a rebase with a dependency error that makes no sense, check your submodules
before you check your sanity.

## 3.5 Verify — three levels

`git status` being clean means the *rebase* succeeded. It says nothing about whether the
result is correct.

**1. Does it build?** Run the real gate, not a subset. In this case that surfaced six
compile errors nothing in the rebase had flagged — see [file 4](04-conflicts-git-cannot-see.md).

**2. Did every commit survive?** `git range-diff` compares two versions of the same branch
and pairs the commits up:

```sh
git range-diff <old-base>..backup/env-gate-prerebase 3817ce18..HEAD
```

```
1:  e75c6345 = 1:  daba62aa feat(api): carry the device serial + client cert on the validate request
2:  592654c9 ! 2:  11fa9626 feat(sandbox): sync an account from upstream and register…
3:  ddfdc2a6 = 3:  e0fa0d1a feat(console): add a Sandbox tab…
4:  e5afade7 ! 4:  21b486cf test(sandbox): exercise account sync…
5:  3257ffb4 ! 5:  d030ea9e chore(release): bump the artifacts…
6:  a61be947 = 6:  4f38a3a2 feat(core): stamp request provenance…
-:  -------- > 7:  da85b4eb fix(web-host): take the typed RequestId…
```

Read it as:

| Marker | Meaning |
|---|---|
| `=` | identical after rebasing — nothing to review |
| `!` | content changed — **exactly where you resolved a conflict**; review these |
| `>` | present only in the new range (a new commit) |
| `<` | present only in the old range (**a dropped commit** — investigate) |

Three `!`s and three conflicts resolved. They matched, which is the check.

**3. Did upstream take something you assumed was yours?** That is file 4, and it is the
one that bites.

## 3.6 Restoring the stash

```sh
git stash pop
```

Expect this to conflict if upstream deleted or moved a file your stash touched — it did
here, because the aegis USB code had moved crates. Two things worth knowing:

- **A failed `pop` does not drop the stash.** Git says so: *"The stash entry is kept in
  case you need it again."*
- You can still read the stashed version of a file whose path no longer exists:

```sh
git diff 'stash@{0}^' 'stash@{0}' -- path/that/is/gone/
```

Resolve by deleting the stale paths and porting the work to wherever upstream moved it —
which is a code task, not a git one.

---

## Check yourself

1. Mid-rebase, a conflict shows `<<<<<<< HEAD`. Is that your change or upstream's?
2. You hit `modify/delete` on a file upstream deleted. You `git rm` it and continue.
   What might you have just silently lost?
3. Why is hand-merging `Cargo.lock` a bad idea even when the conflict looks trivial?
4. `git range-diff` shows `<` against one of your commits. What does that mean and what
   should you do?
5. After a clean rebase your build fails on a dependency feature that "does not exist".
   What is the first thing to check?
6. What is the one command that makes any mid-rebase mess disappear?
