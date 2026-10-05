# 8. Who a commit says it's from

You have one laptop, two jobs for git — the work you're paid for and the things you build
on your own — and one `~/.gitconfig`. Unless you have done something about it, every commit
on that machine carries the same name and email, whichever folder it was made in. This file
is about what that email actually is, why it matters more than it looks, and how to make the
right one appear without having to remember.

It assumes [file 1](01-what-a-commit-is.md) §1.1 — that a commit is a record with `author`
and `committer` lines hashed into its SHA. Nothing else.

## 8.1 The problem: it's just a string you typed

Look at a commit object again:

```sh
$ git cat-file -p HEAD
tree 5dd09888d3e38873ce63c7d868b4e76fac868c49
author Sam Example <sam@personal.example> 1791178421 +0400
committer Sam Example <sam@personal.example> 1791178422 +0400

feat: first page
```

Where did `sam@personal.example` come from? Not from a login, not from a key, not from any
check. Git copied it out of a config file. Change the config and the next commit says
whatever you put there — a colleague's name, a made-up address, anyone's.

git-commit's own documentation is blunt about it: the name "has no effect on
authentication" ([git-commit, *Commit information*](https://git-scm.com/docs/git-commit#_commit_information)).
Pushing needs credentials; *committing* needs nothing. The push credential and the commit
identity are unrelated — you can push with your account a commit that claims to be from
anybody.

So why does the email matter at all? Because the hosting services use it to decide whose
commit it is. GitHub states it directly: it "links a commit to a user by matching the email
address in the commit header to an email address on a GitHub account"
([GitHub Docs](https://docs.github.com/en/pull-requests/committing-changes-to-your-project/troubleshooting-commits/why-are-my-commits-linked-to-the-wrong-user)).
GitLab does the equivalent with the addresses on a user's profile. The avatar next to a
commit, the contributions graph, the "authored by" link — all of it is a lookup on a string
you typed into a file.

### Author vs committer

Two identities, not one, and they diverge more often than you'd think:

| Field | Means | Changes when… |
|---|---|---|
| **author** | who wrote the change | rarely — `--author`, `--reset-author`, or a deliberate rewrite |
| **committer** | who last *made this commit object* | every rebase, cherry-pick or `--amend` |

A rebase (file 3) builds new commits, so it stamps *you* as committer on every one — but it
keeps each commit's original author. Cherry-pick a colleague's fix and the result says
"author: them, committer: you", which is the honest record. `git log` shows only the
author by default; `git log --format=fuller` shows both.

Both lines also carry a **timestamp and a UTC offset**. A commit records not just who but
*when*, in which timezone. Keep that in mind for §8.4.

## 8.2 Where the identity comes from

Git builds each identity from several layers, and the last one to speak wins. From
[git-commit](https://git-scm.com/docs/git-commit#_commit_information) and
[git-config, *Files*](https://git-scm.com/docs/git-config#FILES), lowest priority first:

| Layer | Where | Typical use |
|---|---|---|
| guessed | system user name + hostname (or the `EMAIL` env var) | never what you want — see §8.5 |
| system | `$(prefix)/etc/gitconfig` | machine-wide defaults, rare |
| global | `~/.gitconfig` or `$XDG_CONFIG_HOME/git/config` | your default identity |
| local | `.git/config` in the repo | one repo that's different |
| worktree | `.git/config.worktree` (only with `extensions.worktreeConfig`) | very rare |
| `author.email` / `committer.email` | any config file | when author and committer must differ |
| environment | `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL` | scripts, CI, one-off overrides |

Two refinements worth knowing. `author.email` and `committer.email` override `user.email`
for that one role and are themselves overridden by the environment variables. And
`--author="Name <addr>"` on the command line overrides the author for a single commit.

### Debugging: ask git where it got the answer

When a commit comes out with the wrong address, don't guess which file did it:

```sh
$ git config --show-origin --show-scope user.email
global  file:/home/sam/.gitconfig-personal   sam@personal.example
```

`--show-origin` names the file; `--show-scope` says which layer it counts as. Add
`--get-all` to see *every* value in the order git read them — the last line is the winner:

```sh
$ git config --show-origin --show-scope --get-all user.email
global  file:/home/sam/.gitconfig            sam@work.example
global  file:/home/sam/.gitconfig-personal   sam@personal.example
```

Config only tells you about config. To see the identity git will *actually* stamp, with
environment variables applied, ask for it directly:

```sh
git var GIT_AUTHOR_IDENT
git var GIT_COMMITTER_IDENT
# Sam Example <sam@personal.example> 1791178411 +0400
```

> ⚠️ **Run these inside the repo.** Outside any repository the per-folder rules in §8.3
> have nothing to match against, so `git config user.email` in your home directory reports
> the default — which tells you nothing about what a repo under `~/code/personal/` will use.

## 8.3 One identity per folder: conditional includes

The obvious fix — `git config user.email …` in each repo — works until the day you clone
something new and forget. What you want is a rule: *everything under this folder is
personal*. Git has exactly that.

An **include** pulls another config file in at the point where it appears. A
**conditional include** (`includeIf`) does so only if a condition holds. The one you want
matches on where the repository's `.git` directory lives:

```ini
# ~/.gitconfig — work is the default
[user]
	name = Sam Example
	email = sam@work.example

[includeIf "gitdir:~/code/personal/"]
	path = ~/.gitconfig-personal
```

```ini
# ~/.gitconfig-personal — only read for repos under ~/code/personal/
[user]
	email = sam@personal.example
```

Run in two repos, this gives:

```sh
~/code/personal/site $ git config --show-origin user.email
file:/home/sam/.gitconfig-personal   sam@personal.example

~/code/work/api      $ git config --show-origin user.email
file:/home/sam/.gitconfig            sam@work.example
```

(Demonstrated on git 2.50 with a throwaway `HOME`. The rules below are from
[git-config, *Conditional includes*](https://git-scm.com/docs/git-config#_conditional_includes).)

### The details that decide whether it matches

**The trailing slash matters.** A pattern ending in `/` has `**` added, so
`gitdir:~/code/personal/` becomes `~/code/personal/**` — "this folder and everything inside
it, recursively". Drop the slash and `gitdir:~/code/personal` matches only a `.git`
directory at *exactly* that path. No error, no warning: every repo underneath silently gets
the work address. I checked this; it is the single easiest way to get the whole setup wrong.

**`~/` expands to `$HOME`; `./` means the folder containing this config file.** A pattern
starting with neither (nor `/`) gets `**/` prepended, so `gitdir:personal/` matches a
folder called `personal` anywhere on disk — probably broader than you meant.

**It matches the `.git` directory, not your working folder.** For an ordinary clone that is
the same thing. For a linked worktree or a submodule, git uses the real location of the
`.git` directory, which may be somewhere else entirely.

**`gitdir/i:` is the case-insensitive form**, for filesystems that don't distinguish
`Personal` from `personal` — the default on macOS and Windows.

**Symlinks.** Outside `$GIT_DIR`, both the symlinked path and the real one match, but this
was not true in the first release of the feature (git 2.13). If your code folder is a
symlink and you are on an old git, write the real path.

**Order matters, because the last value wins.** The included file is spliced in *where the
`includeIf` line sits*. Put the include **above** the `[user]` block and the default email
is read second and overrides the personal one — the include "works", and has no effect.
Keep includes at the bottom.

### Matching on the remote instead: `hasconfig:remote.*.url:`

Since git 2.36 ([release notes](https://github.com/git/git/blob/master/Documentation/RelNotes/2.36.0.adoc)),
a condition can look at the repo's remote URLs instead of its location:

```ini
[includeIf "hasconfig:remote.*.url:git@github.com:sam-personal/**"]
	path = ~/.gitconfig-personal
```

Useful when your folders don't follow a neat split. Two limits, both from the docs and
both real:

- **No remote yet, no match.** `git init`, three commits, *then* `git remote add` — those
  three commits got the default identity. Folder rules don't have this gap; remote rules
  do.
- **A file included this way may not itself define remote URLs.** Git forbids it to break
  the chicken-and-egg loop of a file deciding whether it should be included.

| Condition | Matches on | Good for | Watch for |
|---|---|---|---|
| `gitdir:` | location of `.git` | a clean folder split | trailing `/`; worktrees |
| `gitdir/i:` | same, case-insensitively | macOS / Windows | — |
| `hasconfig:remote.*.url:` | any remote URL (git ≥ 2.36) | messy folders, one org per identity | commits made before the remote exists |
| `onbranch:` | checked-out branch name | branch-specific settings | not useful for identity |

## 8.4 Why it's worth the bother

It is tempting to treat a wrong email as cosmetic — a grey avatar on GitHub. Four things
make it more than that. None of them is dramatic on its own, and all of them are permanent.

**1. It may not link to the account you meant.** GitHub attributes by email (§8.1). Commit
to a personal project with a work address that isn't on your personal account and the
commits don't show as yours. Add the work address to your personal account to fix that, and
you have now tied your employer's address to your personal profile.

**2. You will lose the address.** A work email stops working the day you leave. History
doesn't change when you do: GitHub notes that commits made before you changed address "are
still associated with your previous email address"
([GitHub Docs](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address)).
Years of personal commits end up pointing at a mailbox nobody reads.

**3. It is published, and it stays published.** A commit's email is part of what its SHA
hashes. Push to a public repository and every clone, fork and archive has a copy. You can
rewrite your copy (§8.6); you cannot recall theirs.

**4. It is a record of who and when — and that can matter to an employer.**

> **Teacher's aside.** People assume the question "whose is this code?" is answered by
> what the code *does* — "it's nothing to do with my job, so it's mine". Many employment
> contracts don't put it that way. Invention-assignment and IP clauses commonly turn on
> **when** you made something, **with whose equipment**, and **whether it relates to the
> employer's business** — not on its content alone. For a concrete example of those three
> tests written down, California's Labor Code §2870(a) limits assignment clauses so they
> don't reach an invention developed "entirely on [the employee's] own time without using
> the employer's equipment, supplies, facilities, or trade secret information", with
> exceptions for inventions that relate to the employer's business or result from work
> done for them ([§2870](https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=LAB&sectionNum=2870)).
> That is one US state's statute, quoted only because it states the tests plainly; it may
> say nothing about where you live.
>
> Now look at what a commit records: an address, a timestamp, a timezone. A personal-project
> commit carrying your work address at 14:00 on a Tuesday is, at the least, a permanent
> public note that invites the question. I'm not aware of any rule that the email address
> by itself decides ownership — it is evidence, not a verdict — and **this is not legal
> advice**. Read your own contract, and ask a lawyer if it actually matters. The point for
> this course is narrower: keep the two identities apart so the record never suggests
> something that isn't true.

## 8.5 Guardrails that don't rely on memory

Conditional includes make the right identity the default. Guardrails stop the wrong one at
the moment you'd otherwise make the mistake — which matters because the mistake doesn't
look like anything. A commit with the wrong email is a perfectly normal commit.

### Option A: have no default at all

The bluntest guardrail is to remove the fallback. Set a name globally, set **no** email
globally, and turn off guessing:

```ini
# ~/.gitconfig
[user]
	name = Sam Example
	useConfigOnly = true

[includeIf "gitdir:~/code/work/"]
	path = ~/.gitconfig-work
[includeIf "gitdir:~/code/personal/"]
	path = ~/.gitconfig-personal
```

`user.useConfigOnly` tells git not to guess an email from your login and hostname
([git-config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-useruseConfigOnly)).
A repo outside both folders now refuses to commit:

```
fatal: no email was given and auto-detection is disabled
```

That error is the feature: an unclassified repo is a question git makes you answer, rather
than a guess it makes on your behalf.

### Option B: a pre-commit hook that checks the domain

When you want a positive check — "in *this* repo, only this domain" — a hook does it. Git
runs `.git/hooks/pre-commit` before creating a commit, and a non-zero exit aborts it
([githooks](https://git-scm.com/docs/githooks#_pre_commit)):

```sh
#!/bin/sh
# .git/hooks/pre-commit — refuse commits whose author email isn't on the allowed domain.
allowed='@personal.example'
email=$(git var GIT_AUTHOR_IDENT | sed -n 's/^.*<\(.*\)>.*$/\1/p')
case "$email" in
  *"$allowed") exit 0 ;;
  *) echo "pre-commit: author email is '$email', expected *$allowed" >&2
     echo "pre-commit: check 'git config --show-origin user.email'" >&2
     exit 1 ;;
esac
```

`git var GIT_AUTHOR_IDENT` is the useful bit: it reports the identity git is about to use,
including environment overrides and `--author` (I checked both are caught). Make it
executable with `chmod +x`.

Know what it doesn't do:

| Limitation | Why |
|---|---|
| Not cloned | `.git/hooks/` is not part of the repository; each clone needs it again |
| Skippable | `git commit --no-verify` bypasses `pre-commit`, by design |
| Only checks the author | add a second check on `GIT_COMMITTER_IDENT` if that matters to you |
| `core.hooksPath` is all-or-nothing | pointing it at a shared hooks folder (e.g. from inside `~/.gitconfig-personal`, so it applies to that folder only) makes git look **there instead of** each repo's `.git/hooks` — any repo-specific hooks stop running |

Option A catches "I didn't set anything". Option B catches "I set the wrong thing". Use A
everywhere; add B in the repos where a slip would be expensive.

### And on the server side

If you use GitHub's private `noreply` address (`ID+USERNAME@users.noreply.github.com`), the
**Block command line pushes that expose my email** setting rejects a push whose most recent
commit's author email is one of your private addresses
([GitHub Docs](https://docs.github.com/en/account-and-profile/how-tos/email-preferences/blocking-command-line-pushes-that-expose-your-personal-email-address)).
Two caveats from that description: it checks only the most recent commit, and only for
addresses on *your* account — a work address you never added there passes straight through.
A backstop, not a guardrail.

## 8.6 Fixing it after the fact

Remember file 1: the email is hashed into the SHA. So there is no "edit the author" — there
is only display-time remapping, or building new commits.

### Display only: `.mailmap`

A `.mailmap` file at the top of the repo tells git to *show* one identity as another
([gitmailmap](https://git-scm.com/docs/gitmailmap)):

```
# .mailmap — canonical identity, then the one in the commits
Sam Example <sam@personal.example> <sam@work.example>
```

`git shortlog` uses it, and `git log` does too by default (`log.mailmap` is true). Matching
on names and emails is case-insensitive. What it does **not** do is touch the commits: the
old address is still in every object, in every clone, readable with `git cat-file -p`. I
could not find GitHub documentation saying it applies `.mailmap` to its own attribution,
which is documented as a match on the email in the commit header — so treat mailmap as a
fix for how *git* displays history, not for the web UI and not for privacy.

### Real rewrite: `git filter-repo`

To change the addresses in the commits themselves you rebuild every affected commit — and,
as with any rebase, every descendant of the first one changes SHA too (file 1 §1.2). Git's
own manual steers you away from the built-in `filter-branch` and towards the separate
[`git filter-repo`](https://github.com/newren/git-filter-repo) tool
([git-filter-branch, *Warning*](https://git-scm.com/docs/git-filter-branch#_warning)).
It can apply the same mailmap file, this time for real:

```sh
git filter-repo --mailmap .mailmap
```

filter-repo is careful in ways worth noticing. By default it insists on a fresh clone, and
afterwards it removes the `origin` remote so you can't accidentally merge the old history
back in. Putting the result back means force-pushing **every** branch and tag, not one —
and everything in [file 5](05-force-with-lease.md) applies, multiplied: §5.5 in particular,
because every collaborator's clone now holds the old chain and must reset or reclone.

> ⚠️ A rewrite fixes *your* copy and the server's going forward. It does not reach forks,
> other clones, caches or archives that already hold the old commits. If the address was
> public, assume it stays public; the rewrite stops you adding to it.

For a personal repo nobody else has cloned, a rewrite is cheap. For anything shared, the
cost is everyone's afternoon, and mailmap may be the honest compromise.

## 8.7 If you need proof, not a label: signed commits

Everything above is about getting a *label* right. None of it proves anything — §8.1 says
anyone can type your address. The mechanism that does prove origin is a **signed commit**:
a cryptographic signature over the commit, made with a private key only you hold.

Git can sign with GPG (the default) or, since git 2.34, with an SSH key:

```ini
[gpg]
	format = ssh            # default is openpgp; x509 is the third option
[user]
	signingKey = ~/.ssh/id_ed25519.pub
[commit]
	gpgSign = true          # sign every commit, not just ones made with -S
```

(Keys from [git-config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-gpgformat);
the SSH-from-2.34 requirement is from
[GitHub Docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key).)

Upload the public key to your account as a *signing* key and GitHub shows a **Verified**
badge on commits whose signature checks out
([GitHub Docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)).

| | The email field | A verified signature |
|---|---|---|
| Who sets it | anyone with a text editor | only the holder of the private key |
| What it proves | nothing — it's a claim | this commit was signed by a key registered to that account |
| Survives a rebase? | the author's is copied across | no — new commit, so it must be re-signed |

Two things signing doesn't do. It doesn't pick the right email for you — signing a
personal commit with the wrong address gives you a verified commit with the wrong address.
And it says which *key* signed, not which person was at the keyboard. How signatures work,
key management and verification are outside this chapter beyond these basics.

---

## Check yourself

1. You cherry-pick a colleague's commit onto your branch and then rebase the branch. What
   do the `author` and `committer` lines of that commit say now, and why are they different?
2. Your `~/.gitconfig` has `[includeIf "gitdir:~/code/personal"]` and a repo at
   `~/code/personal/site` commits with your work address. What exactly is wrong, and which
   command would have shown you?
3. You fixed that, but moved the `includeIf` block above `[user]` while tidying. Personal
   repos are back to the work address with no error. Explain mechanically.
4. You rely on a `hasconfig:remote.*.url:` rule. You `git init` a new project, commit
   three times, then add the remote. Which address do those three commits carry, and what
   setup would have stopped the first one?
5. You find forty public commits on a personal repo with your work address. Someone
   suggests adding a `.mailmap`. What does that fix, what does it leave untouched, and what
   would a real fix cost if two friends have forked the repo?
6. A commit on GitHub shows your name, your work email and a **Verified** badge. What does
   each of those three things actually establish?
