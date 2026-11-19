# 9. Reverting a merge

A merged branch broke something, so somebody pressed **Revert**. A few weeks later the
branch is fixed and merged again. Git reports success, and the feature still isn't there.
Nothing failed and nothing conflicted. The content just never came back.

That isn't a bug. Git is doing exactly what the revert told it to, and this file is about
what that was.

It assumes [file 1](01-what-a-commit-is.md) §1.1–1.2 (a commit is a snapshot plus its
parents) and [file 2](02-rebase-vs-merge.md) §2.2 (a merge commit has two parents). §9.8,
on keeping one branch per environment, stands on its own: read it if your team does that,
because it's where most reverted merges come from.

## 9.1 What a merge actually records

A merge commit holds a snapshot and **two parents**. The snapshot is the content. The
parents are the history, and they make a promise that every later merge relies on:
*everything reachable from here has been incorporated.*

When you next merge, git doesn't walk the branches commit by commit. It finds the **merge
base**, the most recent commit that both sides have in their history, and does a
three-way merge of three snapshots: the base, yours, and theirs. Whatever each side changed
since the base gets combined. Anything at or before the base counts as already dealt with.

So once a topic branch's commits `A` and `B` are in main's ancestry, the merge base for any
later merge of that branch sits at `B` or after it. `A` and `B` are behind it, and git never
looks at them again.

## 9.2 What a revert records

`git revert X` makes a **new, ordinary commit** whose change is the inverse of `X`'s. It
doesn't remove `X`. For a commit with one parent, that's the whole story.

A merge has two parents, so "the inverse of its change" is ambiguous: changed relative to
which parent? You have to say, with `-m`:

```sh
git revert -m 1 M      # undo what M brought in, keeping parent 1's side
```

Parent 1 is the branch that was checked out when someone ran `git merge`; parent 2 is the
tip of the branch that was merged in. `-m 1` keeps parent 1's content and drops what came
from parent 2 ([Pro Git §7.8, "Undoing Merges"](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)).

The result, call it `W`, has one parent (`M`) and a snapshot equal to main as if the merge
had never happened. But `M` is still there, with both its parents, and the promise from §9.1
still stands. The manual says what that means
([git-revert, `-m`](https://git-scm.com/docs/git-revert)):

> Reverting a merge commit declares that you will never want the tree changes brought in by
> the merge. As a result, later merges will only bring in tree changes introduced by commits
> that are not ancestors of the previously reverted merge.

Linus Torvalds put it in one line, in the mailing-list thread the git project keeps as a
how-to ([revert-a-faulty-merge](https://github.com/git/git/blob/master/Documentation/howto/revert-a-faulty-merge.adoc)):
"Yes, it undoes the data, but no, it doesn't undo history."

## 9.3 Why the second merge brings nothing back

Draw it:

```
main    ──o──o──M──W──o──o──M2
           \   /           /
topic       A──B──────────C
```

`M` merged `A` and `B`. `W` reverted `M`. Then someone fixed the bug with `C` and merged the
branch again at `M2`.

The merge base for `M2` is **`B`**: it's in both histories. So the three-way merge asks two
questions. What changed on topic since `B`? Only `C`. What changed on main since `B`? Among
other things, `W`, which deleted `A`'s and `B`'s work. Git applies both. The result has `C`'s
fix and none of the feature `C` was fixing.

Had nobody added `C`, the second merge would have had nothing to do at all. Pro Git shows it
([§7.8](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)):

```sh
$ git merge topic
Already up-to-date.
```

> **Teacher's aside.** People read "Revert" as *unmerge*. There's no such operation. A merge
> is permanent ancestry; a revert is one more change on top that happens to cancel some
> content. Git's merge only ever looks at three snapshots, and from where it stands `W` is an
> ordinary edit main made after the base, a deletion it has every reason to keep. The
> ancestry tools agree with it: until `C` exists, `git branch --merged main` lists topic, and
> `git merge-base --is-ancestor B main` ([file 7](07-reading-divergence.md) §7.4) says yes.
> Ancestry answers "was it merged?" It never answers "is its content here?"

## 9.4 How it bites in practice

Two shapes, both common on teams that merge with a button in a web UI.

**Pulled before a release, never came back.** A feature merged into an integration branch
has to come out before a release, so its merge is reverted. Later the same branch, with a
fix or two on top, is merged again. The merge succeeds and only the fixes arrive. In one case
I saw, a reverted merge on a long-lived environment branch took out a handful of config files
and a piece of middleware. Every later merge of the same branches reported success, and the
missing files were only noticed days later and re-added by hand in a series of "restore"
commits.

**The wrong-direction merge.** A branch headed for a stable environment has the less stable
integration branch merged into it, which is one click in most web UIs. That would ship every
unreleased change on the integration branch, so the merge is reverted at once. But now the
branch's history contains every one of those integration commits, and `W` has marked all of
them unwanted. When that work is promoted properly later, it doesn't arrive, because as far
as git is concerned the branch already has it.

## 9.5 Three ways back

| Recovery | How | Use it when | What it costs |
|---|---|---|---|
| **Revert the revert** | `git revert W`, *then* merge the branch as usual | The original commits are still what you want, and you're adding fixes on top | One more commit, and a history that flip-flops, so the message must say why (§9.7) |
| **Fresh commits** | `git rebase --no-ff <base>` on the topic branch, or cherry-pick its commits onto a new branch, then merge | You'd rather re-land the work as new commits than resurrect the old ones | New SHAs; the old branch is abandoned |
| **Recreate the branch** | Branch again from the *right* base and cherry-pick only the commits that belong | The wrong-direction merge: the branch's own history is now polluted | A new branch and a new merge request; anyone working on the old one has to move |

**Revert the revert** is Pro Git's answer: "un-revert the original merge, since now you want
to bring in the changes that were reverted out, **then** create a new merge commit". The
order matters, and the how-to stresses it too: "you would want to first revert the previous
revert". Merge first and you land `C` without the feature, exactly as in §9.3.

**Fresh commits** work because new commits aren't ancestors of anything on main, so the merge
base falls back to where the branch started. `git rebase --no-ff` exists partly for this.
The manual says it "recreates the topic branch with fresh commits so it can be remerged
successfully without needing to 'revert the reversion'"
([git-rebase](https://git-scm.com/docs/git-rebase)), and the how-to agrees that a branch
"rebuilt from scratch" can be re-merged "without doing anything else fancy".

**For the wrong-direction merge, neither of those is enough.** Reverting `W` would bring
back the whole integration branch, which is the thing you were trying not to ship. The
branch itself is the problem, so replace it:

```sh
git switch -c feature-v2 origin/release           # the base this work should have had
git cherry-pick <feature commits, oldest first>   # only the feature's commits, never the merge
```

Then close the old merge request and say why in the new one.

> ⚠️ A revert of the revert re-applies *exactly* what `W` removed, no more and no less. If
> the original merge carried something you still don't want, reverting `W` brings that back
> too. Read `git show --stat W` before you revert it.

The how-to closes with the advice that matters most: "there's nothing wrong with reverting a
merge, but from a workflow angle it's something that you generally should try to avoid."
The cheap alternative is landing risky work behind a flag (§9.8), so that turning it off
needs no revert at all.

## 9.6 Check by content, not by ancestry

After any of this, the question you care about is *is everything there that should be?*
Only a content comparison can answer it:

```sh
git diff --stat staging main                      # every file whose content differs, tip against tip
git log --oneline --grep='This reverts' staging   # reverts on a branch, found by their default message
```

`git diff A B` compares the two tips directly ([file 7](07-reading-divergence.md) §7.5).
That's what you want here: it ignores what the ancestry claims and compares content. Every
difference it lists should be one you can explain.

The `--grep` relies on git's default revert message, whose body starts "This reverts"
followed by the commit it undid ([git-revert, `--reference`](https://git-scm.com/docs/git-revert)).
A revert whose message was rewritten from scratch won't match, which is one more reason to
keep that line and add to it rather than replace it.

## 9.7 Say why in the message

The default revert message records *what* was undone and nothing about *why*. When the
revert is itself reverted a week later, the history reads as a flip-flop that nobody can
explain. The manual is blunt ([git-revert, DISCUSSION](https://git-scm.com/docs/git-revert)):
"it is *strongly* recommended to explain why the original commit is being reverted". It also
notes that reverting reverts produces subjects like `Reapply "Reapply "<original-subject>""`,
and asks you to reword them.

```
Revert "Merge branch 'feat/refund-export'"

This reverts commit 4f1c2d9….

Refund export must not ship in 2.3: the downstream ledger doesn't accept
the new field yet. To re-land it, revert this commit and then merge any
fixes. Re-merging the branch on its own will bring nothing back.
```

The last two lines are the ones that save the next person a day.

## 9.8 Environment branches: where most of this comes from

Every story in §9.4 happened on a team that kept one long-lived branch per environment
(dev, staging, production, sometimes one per cloud or per customer install) and merged each
feature into each of them separately.

That arrangement turns every change into a multi-way merge. The branches drift apart,
because each one receives features in a different order, plus its own hotfixes and its own
per-environment edits. So the same conflict comes up three or four times and is resolved by
hand each time, often by different people. Sooner or later one of those resolutions picks the
stale side, and a fix that landed weeks ago quietly reverts on one branch. In one case I saw,
a renamed URL query parameter went back to its old name on one environment, with no conflict
reported and no test failing. Configuration gets the same treatment. Two infrastructure
branches that had diverged by hundreds of commits each way meant every service change had to
be ported across by hand in numbered "sync" commits, and lines went missing in the merges.

Reverts make it worse. A revert on one environment branch rides along the next time that
branch is merged into another one, which is §9.3 happening across branches.

### Sources disagree, and the disagreement is the lesson

Pro Git describes long-running branches approvingly
([§3.4](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows)): a stable
`master`, a less stable `develop`, and "work silos, where sets of commits graduate to a more
stable silo when they're fully tested." Trunk-based development says the opposite. Everyone
commits to one trunk and resists long-lived branches. Release branches exist, but they "are
**not** merged back into trunk", and a bug is fixed on trunk and then cherry-picked to the
release branch, never the other way round
([trunkbaseddevelopment.com, "Branch for release"](https://trunkbaseddevelopment.com/branch-for-release/)).

They're less opposed than they look. In Pro Git's model the branches are points on **one
line** of history: "The stable branches are farther down the line in your commit history,
and the bleeding-edge branches are farther up." `master` is always an ancestor of `develop`,
and work only ever moves towards stability.

Environment branches break that the moment they hold *different* commits: a feature merged
into each one separately, a hotfix applied to only one, a back-merge from staging into dev.
Now they aren't points on one line but separate lines, and everything in this file applies to
every pair of them. The evidence for that is the failure stories above. The evidence for
Pro Git's version working is that it never needs them: when the stable branch is just an
older point on the same line, there's nothing to resolve.

| | Branch per environment | One trunk |
|---|---|---|
| A change reaches production by | being merged into every branch, separately | the same commit being deployed further |
| What differs between environments | code, config, and whatever each conflict resolution did | configuration only |
| Hiding unfinished work | comment it out on one branch | a runtime flag, off where it isn't ready |
| Withdrawing a feature | revert it on each branch (and meet §9.3 later) | turn the flag off |
| "Is staging the same as production?" | `git diff staging production`, and investigate | yes, if they run the same commit |

### What replaces it

**Vary environments with configuration, on one branch.** Ansible's own sample layout keeps
one inventory directory per environment, `inventories/production/` and
`inventories/staging/`, each with its own `group_vars/` and `host_vars/`, side by side on the
same branch with the roles shared
([Ansible, "Sample Ansible setup"](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)).
The docs admit it's "harder to maintain, because there are more files." It's still one
branch: a change to a shared role reaches every environment in one commit.

**Gate unfinished features with a flag, not with commented-out code.** Commenting a route out
on one branch makes that branch's code differ from every other branch's, and the next merge
has to reconcile them. That's exactly where the conflicts come from. A flag keeps the code
identical everywhere and moves the difference into configuration. Pete Hodgson's article on
feature toggles says release toggles "allow incomplete and un-tested codepaths to be shipped
to production as latent code" ([martinfowler.com](https://martinfowler.com/articles/feature-toggles.html)).

```go
type Flags struct {
	Refunds bool // set per tenant or per environment, from config
}

func navItems(f Flags) []string {
	items := []string{"Dashboard", "Orders"}
	if f.Refunds {
		items = append(items, "Refunds")
	}
	return items
}
```

<details><summary>Rust</summary>

```rust
struct Flags {
    refunds: bool, // set per tenant or per environment, from config
}

fn nav_items(f: &Flags) -> Vec<&'static str> {
    let mut items = vec!["Dashboard", "Orders"];
    if f.refunds {
        items.push("Refunds");
    }
    items
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <string>
#include <vector>

struct Flags {
    bool refunds = false;  // set per tenant or per environment, from config
};

std::vector<std::string> navItems(const Flags& f) {
    std::vector<std::string> items{"Dashboard", "Orders"};
    if (f.refunds) items.push_back("Refunds");
    return items;
}
```

</details>

<details><summary>JavaScript</summary>

```js
// flags: { refunds: boolean }, set per tenant or per environment, from config
export function navItems(flags) {
  const items = ["Dashboard", "Orders"];
  if (flags.refunds) items.push("Refunds");
  return items;
}
```

</details>

Every branch carries that code. Staging's config says `true`, production's says `false`, and
there's nothing for a merge to get wrong. The same article warns that toggles are "inventory
which comes with a carrying cost": each one is a fork in the code that has to be tested both
ways and deleted once the feature is settled. A divergent branch has the same cost, and
nobody tests it both ways either.

**Promote a commit, not a merge.** Build once from trunk, tag the commit, deploy the tag to
staging, and when staging is happy deploy the *same* tag to production. Nothing is merged on
the way, so nothing can be resolved wrongly on the way. When a release needs a fix, fix trunk
first and cherry-pick to the release branch.

```sh
git tag -a v2.3.0 -m "Release 2.3.0" <sha>    # the commit staging tested
git push origin v2.3.0                        # production deploys this tag
```

---

## Check yourself

1. You run `git revert -m 1 M` on main and then, with no new commits on the branch,
   `git merge topic`. What does git do, and which commit is the merge base?
2. After that revert, a colleague adds a fix `C` to topic and merges it. Which changes reach
   main, and which don't? Say it in terms of the three snapshots git compares.
3. `git branch --merged main` lists topic, but topic's feature isn't on main. How can both be
   true, and which command tells you what's actually there?
4. Why does reverting a merge need `-m` when reverting an ordinary commit doesn't?
5. A release branch had the integration branch merged into it by mistake, and that merge was
   reverted. Why is reverting the revert the wrong fix here, and what's the right one?
6. Your team keeps dev, staging and production branches and merges every feature into each.
   Name two ways a fix can vanish from one of them with no conflict reported, and say what
   you'd replace the arrangement with.
