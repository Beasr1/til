# 10. Attributes, line endings and blame

A merge request touches five files and shows 1,350 changed lines. Someone reviews it,
approves it, and it merges. It turns out 45 lines changed. The rest is every line ending in
those files flipping from LF to CRLF. The review couldn't have been real, the merge now
conflicts with everyone else's work in those files, and `git blame` on 1,300 lines points at
a commit that didn't change them.

Three habits stop that, and they share one idea. Git compares bytes and infers everything
else, including what a "line" is and whether a file was renamed. So you commit the rules you
want it to infer by, and you keep commits whose only job is to churn bytes away from commits
that change behaviour.

It assumes [file 1](01-what-a-commit-is.md) §1.1: a commit stores a snapshot, not a diff.

## 10.1 One byte or two

A line ending is either **LF**, the single byte `0x0A` (`\n`), which Linux and macOS tools
write, or **CRLF**, the two bytes `0x0D 0x0A` (`\r\n`), which is the Windows convention. Git
compares lines byte for byte, so `return x;\r\n` and `return x;\n` are different lines. An
editor or tool that rewrites a file's endings has changed every line in it, as far as git is
concerned.

When a diff looks too big to be true, check the endings before you read a line of it:

```sh
file src/orders.js              # "ASCII text, with CRLF line terminators" if it's CRLF
git diff --ignore-cr-at-eol     # the same diff with carriage-return-only differences hidden
git diff -w                     # ...or with every whitespace difference hidden
```

`--ignore-cr-at-eol` does what it says: "Ignore carriage-return at the end of line when doing
a comparison". `-w` is broader: it "ignores differences even if one line has whitespace where
the other line has none" ([git-diff](https://git-scm.com/docs/git-diff)). If 1,350 lines
shrink to 45, send the merge request back rather than merging it. The hidden lines are still
in the commit, and they'll conflict with every other branch that touches those files.

## 10.2 Where git converts: on the way in, and on the way out

For a path git treats as text, line endings can be converted at two moments
([gitattributes, `text`](https://git-scm.com/docs/gitattributes)):

```
working tree  ──── git add ────▶   index and commits   ──── checkout ────▶  working tree
(LF or CRLF)    normalised to LF        (LF)            converted per eol    (LF or CRLF)
```

On the way in: "When a matching file is added to the index, the file's line endings are
normalized to LF in the index." On the way out: "its line endings may be converted from LF to
CRLF depending on the `eol` attribute, the Git config, and the platform."

So the repository can hold LF while each working tree holds whatever its owner's tools
prefer. That's the design working. The trouble starts when nothing tells git a path *is*
text, and nothing converts at all.

## 10.3 Who decides: each clone, or the repository

| Setting | Where it lives | What it does |
|---|---|---|
| `core.autocrlf=true` | one clone's config | "the same as setting the `text` attribute to 'auto' on all files and core.eol to 'crlf'" |
| `core.autocrlf=input` | one clone's config | normalises on the way in; "no output conversion is performed" |
| `core.eol` | one clone's config | the working-tree ending for text files (`lf`, `crlf`, `native`; default `native`), "ignored if `core.autocrlf` is set to `true` or `input`" |
| `text`, `text=auto` | `.gitattributes`, committed | marks paths as text, so they're normalised to LF in the index |
| `eol=lf`, `eol=crlf` | `.gitattributes`, committed | the working-tree ending for those paths, on every machine |

(The config quotes are from [git-config](https://git-scm.com/docs/git-config).)

The rule that connects the two halves: "If the `text` attribute is unspecified, Git uses the
`core.autocrlf` configuration variable to determine if the file should be converted"
([gitattributes](https://git-scm.com/docs/gitattributes)). So in a repository with no
`.gitattributes`, every clone decides for itself, from config nobody else can see.

That's how you get two bugs that look unrelated:

- **The whole-file rewrite.** Someone with no conversion configured and an editor that writes
  CRLF commits a file. Nothing normalises it, so CRLF goes into the repository and every line
  of the file changes.
- **The check that fails on one machine.** A Windows clone with `core.autocrlf=true` checks
  files out as CRLF over an LF index, which is correct. Then a formatting check normalises
  CRLF to LF in memory and compares the result with the bytes on disk, sees a difference, and
  reports files as unformatted on that machine only. Nothing in the repository is wrong. The
  tool is reading a working tree that one clone's config converted.

> **Teacher's aside.** `core.autocrlf` is a personal preference: "here's how I like my
> working tree." `.gitattributes` is the project's rule: "here's what these files are."
> People reach for the config because it's what the internet tells them to set, and then
> every new clone, CI runner and container starts from a different preference. Line endings
> are a property of the project, so the rule belongs in a file the project commits.

## 10.4 A `.gitattributes` that pins it

```gitattributes
# Text: LF in the repository, LF in every working tree, on every OS
*         text=auto eol=lf

# Files whose consumer needs CRLF on disk
*.bat     text eol=crlf
*.cmd     text eol=crlf

# Never convert, diff as text, or text-merge these
*.png     binary
*.jpg     binary
```

What each part does, from [gitattributes](https://git-scm.com/docs/gitattributes):

- `text=auto` lets git decide per file: "Git decides by itself whether the file is text or
  binary."
- `eol=lf` "uses the same line endings in the working directory as in the index when the file
  is checked out", which is LF. Every working tree gets LF, whatever the clone's
  `core.autocrlf` says, because the attribute is now specified.
- The carve-outs come after the `*` line because "a later line overrides an earlier line.
  This overriding is done per attribute." So `*.bat` keeps `text` and gets `eol=crlf`.
- `binary` is a built-in macro for `-diff -merge -text`.

Choosing `eol=lf` for everything is a choice, not the only answer. The docs' own example is
just `* text=auto`, which normalises the repository and leaves each working tree to its
platform. Pin `eol=lf` when tools compare bytes on disk (formatters, generators, checksum
tests) and you want them to agree on every OS.

> ⚠️ `text=auto` doesn't fix files that are already committed with CRLF. The docs: "If it is
> text and the file was not already in Git with CRLF endings, line endings are converted on
> checkin and checkout ... Otherwise, no conversion is done." Committing the attributes file
> protects new work. Existing files need §10.5.

## 10.5 Renormalising a repository that's already mixed

From a clean working tree, as [gitattributes](https://git-scm.com/docs/gitattributes) gives it:

```sh
git add --renormalize .
git status                  # the files that will be normalised
git commit -m "Introduce end-of-line normalization"
```

If a file shows up that shouldn't be converted, give it `-text` in `.gitattributes` and run
it again.

Make that commit on its own, with nothing else in it. It touches every line of every file it
converts, so it's exactly the kind of commit §10.6 says to hide from blame.

Branches cut before it will collide with it: "merging anything where the attribute is not in
place would normally cause merge conflicts." The merge strategy has an option for precisely
this. `git merge -X renormalize main` "runs a virtual check-out and check-in of all three
stages of any file which needs a three-way merge" and "is meant to be used when merging
branches with different clean filters or end-of-line normalization rules"
([git-merge](https://git-scm.com/docs/git-merge)).

## 10.6 Formatting commits, and keeping them out of blame

A one-line fix goes in. The editor formats the file on save and rewrites 2,000 lines of it. It
costs three things:

| Cost | Why |
|---|---|
| Review | The one line that matters is somewhere in 2,000 |
| Conflicts | Every teammate with an edit anywhere in that file now overlaps yours |
| Blame | Every reformatted line now says *you*, *today*, *"fix rounding"* |

**Keep behaviour changes and formatter-only changes in separate commits.** If the file needs
formatting, format it in a commit that does nothing else, and say so in the message. A
repo-wide reformat is one agreed sweep, done once, with the formatter's config committed and
checked in CI so it never has to be done again.

Then list the sweep in a file blame knows to skip:

```
# .git-blame-ignore-revs
# Apply the formatter to the whole codebase
<full 40-character SHA of the formatting commit>
```

```sh
git config blame.ignoreRevsFile .git-blame-ignore-revs     # once per clone
git blame --ignore-revs-file .git-blame-ignore-revs file   # or per command
```

The format is "one unabbreviated object name per line", and "comments beginning with `#` are
ignored". An ignored commit's lines "will be blamed on the previous commit that changed that
line or nearby lines". With `blame.markIgnoredLines` set, those lines are marked with a `?`,
so you know blame had to guess ([git-blame](https://git-scm.com/docs/git-blame)).

GitHub reads the same file automatically if it's "in the root directory of your repository",
and shows a banner when it's hiding revisions. It isn't complete: "If the commit was the last
to modify a line, it will still appear in blame"
([GitHub Docs](https://docs.github.com/en/repositories/working-with-files/using-files/viewing-and-understanding-files)).

> ⚠️ The SHA has to be the one that landed. A rebase or squash-merge builds a new commit with a
> new SHA ([file 1](01-what-a-commit-is.md) §1.2), so add the entry in a follow-up commit
> after the sweep is on the main branch. And if your host squash-merges, a formatting commit
> inside a feature merge request gets folded into the feature, which undoes the whole point.
> Give the sweep its own merge request.

## 10.7 A rename is a guess

Git doesn't record renames. A commit is a snapshot ([file 1](01-what-a-commit-is.md) §1.1),
so a move is stored as "this path is gone, that path is new". Diff and merge *infer* the
rename by comparing contents: a deleted file and an added file are paired as a rename when
they're similar enough. The default threshold is 50%. `-M90%` means "consider a delete/add
pair to be a rename if more than 90% of the file hasn't changed"
([git-diff, `--find-renames`](https://git-scm.com/docs/git-diff)). Merges do the same: for the
default `ort` strategy, rename detection "is the default"
([git-merge](https://git-scm.com/docs/git-merge)).

So suppose you move a file *and* rewrite most of it in one commit, while a teammate edits it at
the old path:

- **Still above the threshold:** git pairs the two paths and merges your teammate's edits
  into the new location, conflicting wherever your rewrite and their edits overlap, which is
  most places.
- **Below it:** git sees a deleted file and an unrelated new one. Your teammate's edit is to a
  file you deleted, which is a `modify/delete` conflict ([file 3](03-a-real-rebase.md) §3.3),
  and their work has nowhere to go automatically.

**Rename in a commit of its own, with no content changes, landed quickly and announced.** A
pure move is 100% similar, so every tool follows it. Make the edits in the commits after it.
To confirm git sees it as a rename:

```sh
git diff --name-status -M HEAD~1 HEAD
# R100    src/analytics.js    src/dashboard/analytics.js
```

Status `R` is "always followed by a score (denoting the percentage of similarity between the
source and target of the move or copy)". Anything other than `R100` means the move commit
also changed content.

If a merge has already gone wrong, you can lower the threshold for that one merge with
`git merge -X find-renames=<n>`. That can rescue it, at the risk of pairing files that only
look alike.

---

## Check yourself

1. A merge request shows 1,350 changed lines across five files, and
   `git diff --ignore-cr-at-eol` shows 45. What happened, and why does it matter beyond the
   review?
2. A repository has no `.gitattributes`. Your Windows colleague has `core.autocrlf=true`. You
   have nothing set, and an editor that writes CRLF. What ends up in the repository from each
   of you, and which of you produces the whole-file rewrite?
3. You commit `* text=auto eol=lf` to a repository where some files were committed with CRLF
   years ago. Why doesn't that fix them, and what do you run?
4. Why does the SHA go into `.git-blame-ignore-revs` only after the formatting commit lands,
   and what goes wrong if your host squash-merges the merge request it's in?
5. You move a file and rewrite most of it in one commit, while a teammate edits it at the old
   path. Predict what the merge does, and how splitting your commit in two changes the
   outcome.
