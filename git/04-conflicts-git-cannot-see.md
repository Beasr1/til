# 4. The conflicts git cannot see

⭐ This is the file to read twice.

Git merges **text**. It has no idea what the text means. So the dangerous outcome of a
rebase is not the conflict it stops on — you cannot ignore those. It is the hunk it merges
*cleanly and wrongly*, and reports nothing about.

All three examples below come from one rebase, and none of them produced a conflict
marker.

## 4.1 Semantic: the type changed underneath you

Upstream had changed how request ids are represented — from `Option<u64>` to a typed
`RequestId`. Your commit added two new handlers taking the old type. Both sides touched
*different lines in the same file*, so git merged them without a murmur.

```rust
// merged clean. six compile errors.
async fn sync_sandbox_account(&self, payload: …, request_id: Option<u64>) -> Response
//                                                             ^^^^^^^^^^^ upstream now passes RequestId
```

The compiler caught it, which makes this the *benign* member of the family. Everything
git can't see that the compiler also can't see is worse.

**The rule this gives you:** a rebase is not done when `git status` is clean. It is done
when the thing builds and the tests pass. Run the project's real gate, not `cargo check`
on the crate you remember touching.

There is a subtlety worth knowing here. Rebase replays commits one at a time, and each
intermediate commit is committed *before* you build anything. So a rebased branch can have
a green tip and a middle commit that does not compile — which quietly breaks `git bisect`
later. Fixing it properly means folding the repair into the commit that needs it; fixing
it pragmatically means one repair commit at the tip and a note in the MR. Know which you
chose.

## 4.2 Namespace: someone took your numbers

The branch had added end-to-end scenarios **S26** and **S27** to a test-plan document, and
the test files referenced them by name. While the branch sat, master added **S26 through
S41** for entirely different features.

Git merged both tables happily — different lines of a markdown table, no overlap. The
result was a document with two S26s meaning two different things, and a repo where
`grep S27` returned both a sandbox test and a transport test.

Nothing red. Nothing to build. Just a document that had quietly become wrong.

The fix was to renumber the *unmerged* side (S26/S27 → **S42/S43**) — because the rule in
that repo is that landed scenario IDs never move, and master's had landed. Then grep for
the old identifiers across code and docs, because a stale ID in a test comment now points
at someone else's scenario.

**Where this class hides:** anything humans allocate from a shared sequence.

- test/scenario IDs
- error codes (a repo that says "codes are permanent" means it)
- migration numbers — `V017__add_column.sql` twice is a fun afternoon
- port numbers, feature flags, enum discriminants, protobuf/capnp field numbers
- fixture ids, seed data primary keys

If your branch allocated a number from a shared range, re-check that range after rebasing.
Always.

## 4.3 Release: upstream shipped the version you were bumping to

The nastiest of the three, because the failure mode is **silence in production tooling**.

The branch bumped every changed artifact by one minor — capi `0.22.0 → 0.23.0`, and so on
for seven SDKs. Meanwhile master had *released* those exact numbers for its own work.

After the rebase, the manifests said `0.22.0`… and the release lock said `0.22.0`. The
repo's CI publishes an artifact only when its version differs from the lock, so:

```sh
$ cargo xtask changed
api-prod-v1        # ← that's all. Eight other changed artifacts, invisible.
```

Nothing failed. The MR would have gone green, merged, and shipped **none** of the SDK
changes, because every release job would have been skipped as a no-op. The only signal was
a tool nobody runs by habit.

**The rule:** if your branch touches versions, changelogs, or anything else whose meaning
is "differs from what is published", re-derive it after a rebase rather than trusting what
the merge produced. Ask the build system what it thinks changed, out loud.

## 4.4 The general shape

Every one of these is the same failure:

> Git guarantees your **text** survives a rebase. It guarantees nothing about your
> **assumptions**.

Your commits encoded assumptions about the world at the moment you wrote them: this type
looks like that, S26 is free, 0.23.0 is unreleased. The rebase moved those commits three
weeks forward without re-checking a single one.

So after a rebase, ask what upstream took while you were away:

| Assumption | How to re-check |
|---|---|
| Types and signatures I call | Build. The whole thing. |
| Identifiers I allocated | `grep` the shared range for duplicates |
| Versions I bumped | Ask the release tooling what it considers changed |
| Files I edited | `git log --oneline <base>..master -- <paths>` — who else touched them? |
| Config/flags I added | Does the key already exist upstream with another meaning? |

That last query is worth putting in your fingers. Before rebasing:

```sh
git log --oneline HEAD..origin/master -- $(git diff --name-only $(git merge-base HEAD origin/master))
```

*What has upstream done to the files I touched?* Everything expensive is in that answer.

---

## Check yourself

1. Why did the `RequestId` change not produce a conflict marker?
2. A rebased branch's tip builds. Why might `git bisect` still break, and what would you
   do about it?
3. Your branch added migration `V017__add_index.sql`. Upstream also added a `V017`.
   Git reports no conflict. Why not, and what breaks?
4. What does it mean that the version collision produced "silence in production tooling"?
   Why is that worse than a red build?
5. Name three kinds of shared sequence a branch can allocate from.
6. In one sentence: what does a rebase actually guarantee?
