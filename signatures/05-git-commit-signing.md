# 5. Commit signing with SSH keys

## 5.1 The problem

A git commit records an author and a committer, and both are just strings you type into
`git config`. Nothing stops anyone writing `user.email = alice@example.com` and pushing.
So "this commit is by Alice" is, by default, an unverified claim — the same impersonation
failure as [file 01 §1.1](01-foundations.md).

Git can attach a signature to a commit or tag. Since Git 2.34 (per
[GitHub's docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification#ssh-commit-signature-verification)),
that signature can be an SSH signature, made by exactly the `ssh-keygen -Y` machinery of
[file 03](03-ssh-signatures.md), in the `git` namespace.

## 5.2 Configuration

All from [git-config(1)](https://git-scm.com/docs/git-config):

| Setting | What it does |
|---|---|
| `gpg.format = ssh` | Use SSH instead of the default `openpgp` (other value: `x509`) |
| `user.signingKey` | With `gpg.format=ssh`: a path to your private key, or to the public key when the private half is in `ssh-agent`, or a literal `key::ssh-ed25519 AAAA…` |
| `commit.gpgSign = true` / `tag.gpgSign = true` | Sign every commit / tag without passing `-S` |
| `gpg.ssh.allowedSignersFile` | The file of keys *you* trust when **verifying** — the [file 03 §3.5](03-ssh-signatures.md) format |
| `gpg.ssh.revocationFile` | A KRL or list of revoked keys; matching signatures "will show as invalid" |
| `gpg.ssh.program` | Defaults to `ssh-keygen` |

A complete local demonstration, in a scratch repository with a throwaway key:

```sh
git config gpg.format ssh
git config user.signingKey ../k.pub      # private half k sits next to it
git config commit.gpgSign true
git commit -m first
```

The signature lives inside the commit object, in a `gpgsig` header:

```
$ git cat-file commit HEAD
tree df55a7dce59d040dc7819c1e241082965a80ebd9
author Alice <alice@example.com> 1791178569 +0400
committer Alice <alice@example.com> 1791178569 +0400
gpgsig -----BEGIN SSH SIGNATURE-----
 U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg3aQU0A1vzwflbZBJP8OF2+2Jt+
 NTKiMUwcUPT8IuJ8YAAAADZ2l0AAAAAAAAAAZzaGE1MTIAAABTAAAAC3NzaC1lZDI1NTE5
 ...
 -----END SSH SIGNATURE-----

first
```

The same SSHSIG blob as [file 03 §3.3](03-ssh-signatures.md) — the `Z2l0` you can spot in the second line is the
base64 of the namespace `git`.

Verifying needs a trust file, and git won't guess one:

```
$ git verify-commit HEAD
error: gpg.ssh.allowedSignersFile needs to be configured and exist for ssh signature verification
$ git config gpg.ssh.allowedSignersFile ../allowed_signers
$ git verify-commit HEAD
Good "git" signature for alice@example.com with ED25519 key SHA256:30/gF7srI49GpVd44oIkphGD6NdBu9bSjQx4G8ffRsM
```

`git log --show-signature` shows the same line per commit.

The git docs are frank about where that trust file comes from: it "can be set to a location
outside of the repository and every developer maintains their own trust store", or it can
live inside a repository that only allows signed commits, so "only committers with an
already valid key can add or change keys in the keyring." That second guarantee holds only
if something actually *enforces* signed commits on that branch — otherwise anyone who can
push can add their own key to the file.

## 5.3 What GitHub's "Verified" means

GitHub verifies signatures itself, against keys registered on accounts, and shows a badge.
From [About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification):

| Status | Meaning (GitHub's words, condensed) |
|---|---|
| **Verified** | Signed, and the signature was successfully verified |
| **Unverified** | Signed, but the signature could not be verified |
| *No verification status* | Unsigned (the default; vigilant mode changes this) |

With **vigilant mode** enabled, unsigned commits attributed to you are flagged
**Unverified**, and a signed, verified commit with an author who has vigilant mode on but
isn't the committer is **Partially verified**, because "the commit signature doesn't guarantee the consent of
the author."

For SSH specifically:

- The key has to be uploaded as a **signing key**. An authentication key doesn't count;
  to use one key for both you "need to upload it twice" ([file 02 §2.3](02-trust-anchors.md)).
- GitHub ties the signature to an account through the **committer email**. Its REST API
  lists the reasons a signature can fail verification, including `no_user` ("No user was
  associated with the committer email address in the commit"), `unverified_email` (the
  address is on an account but not verified there) and `unknown_key` ("The key that made
  the signature has not been registered with any user's account")
  ([REST API: commits](https://docs.github.com/en/rest/commits/commits)). That table is
  written largely in OpenPGP terms, and I couldn't find a GitHub page that spells out the
  SSH rules separately; the email-must-be-verified requirement is clearly documented for
  GPG keys and, from the shared reason codes, appears to apply to SSH too — confirm
  before quoting it as GitHub policy for SSH.
- Once verified, a **persistent verification record** is stored and kept even if the key
  is later removed ([file 02 §2.4](02-trust-anchors.md)).

So "Verified" means, roughly: *a key registered as a signing key on the account that owns
this committer email signed this commit, as checked when GitHub first saw it.* It does not
mean the author field is true (only the committer is bound), that the code is reviewed, or
that the key wasn't stolen.

## 5.4 Git and time: where the signer picks the clock

[File 03 §3.8](03-ssh-signatures.md) showed that SSH signatures carry no timestamp, so `valid-after` and
`valid-before` in `allowed_signers` are checked against whatever time the verifier
supplies. Git supplies one: the **commit's own timestamp**. In `gpg-interface.c` it passes
`-Overify-time=` set from the signed payload's timestamp to `ssh-keygen`. The git-config
docs describe the intent: "Git will mark signatures as valid if the signing key was valid at
the time of the signature's creation. This allows users to change a signing key without
invalidating all previously made signatures."

That's a good feature for honest rotation. The catch is that the commit timestamp is
written by whoever makes the commit. Here's a key retired on 1 January 2026, in a scratch
repository:

```
alice@example.com namespaces="git",valid-before="20260101" ssh-ed25519 AAAA…
```

A commit made today with that key correctly fails:

```
$ git verify-commit HEAD
../as_git:1: key has expired: verify time 2026-10-05T09:36:09 > valid-before 2026-01-01T00:00:00
No principal matched.
```

The same key, today, signing a commit whose dates are set into 2025:

```
$ GIT_COMMITTER_DATE='2025-06-01T12:00:00Z' GIT_AUTHOR_DATE='2025-06-01T12:00:00Z' git commit -m backdated
$ git verify-commit HEAD
Good "git" signature for alice@example.com with ED25519 key SHA256:30/gF7srI49GpVd44oIkphGD6NdBu9bSjQx4G8ffRsM
```

> ⚠️ **`valid-before` does not contain a leaked key in git.** Anyone holding the key can
> backdate a commit to inside the validity window and it verifies. `valid-before` is the
> right tool for *retiring* a key you still control. For a key that **leaked**, remove it
> from the trust file or list it in `gpg.ssh.revocationFile`, and accept that old
> signatures by it can no longer be told apart from forgeries — exactly the conclusion of
> [file 04 §4.6](04-signing-in-automation.md).

---

## Check yourself

1. Before any signing, what stops someone pushing a commit with your name and email on it?
   After you sign all your commits, what has changed for a reader who never checks
   signatures?
2. You commit `allowed_signers` into a repository and point `gpg.ssh.allowedSignersFile`
   at it. Under what condition does that file become self-protecting, and what goes wrong
   if that condition isn't met?
3. A commit shows "Verified" on GitHub. List three things that badge does *not* establish.
4. You use one key for both GitHub login and signing but uploaded it only as an
   authentication key. Your signed commits show "Unverified". Why, given the signature is
   mathematically fine?
5. Explain, mechanically, why a backdated commit signed with a key that has
   `valid-before` in the past still verifies in git. Why doesn't the same trick work
   against a plain `ssh-keygen -Y verify` of a file signature — and what does that tell
   you about who should choose the verification time?

(Answers in [`07-exercises.md`](07-exercises.md).)
