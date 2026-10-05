# 7. Exercises and answers

Try to answer before reading. Getting one wrong is data, not failure.

---

## [File 01](01-foundations.md) — Foundations

**1. "It's base64-encoded, so it can't be altered without us noticing."**

The attacker decodes the payload (the algorithm is public, RFC 4648), edits the bytes,
and re-encodes. The result is valid base64 of different content. Nothing fails because
there's nothing *to* fail: base64 has no key and no check value, so the receiver has no
way to tell "encoded by the sender" from "encoded by anyone". The colleague has confused
"unreadable to a human glancing at it" with "protected".

**2. SHA-256 on the same page as the file.**

It defends against **corruption**: a truncated or garbled download won't match. It does
nothing against **tampering** or **impersonation** by anyone who can change the page —
they replace the file and recompute the digest, which needs no secret. The change that
matters is getting the digest from a **channel the attacker can't touch**: a different,
independently secured place, or from the author directly. Then second-preimage resistance
does real work, because the attacker would need different bytes matching a digest they
don't control.

**3. Why ignoring a bad CRC-24 is safe in RFC 9580.**

Because the CRC was never the security mechanism. It's keyless, so an attacker recomputes
it; it only ever caught accidents. Integrity against attackers comes from the signature
over the content (or the integrity protection built into encrypted packets). If the armour
was corrupted, the signature check fails anyway; if it was tampered with, the CRC wouldn't
have caught it. The footer added cost and no protection, which is exactly RFC 9580's
stated rationale.

**4. HMAC for public verification of your notes.**

Two options for the key, both fatal. Keep it secret: readers can't compute the tag, so
they can't verify anything. Publish it so they can verify: now every reader can compute
valid tags for any text, so a forger's tag is indistinguishable from yours. HMAC's same
key creates and checks; "anyone can check, only I can create" needs the two capabilities
split, which is what a key pair does.

**5. The disputed webhook.**

Neither side can be proven right by the tag. Both hold the key, so the receiver could have
made the tag themselves; the valid tag shows only that *some* key holder made it. A
signature by the provider's private key would settle "did the provider's key sign this
request?" — the receiver doesn't hold that key. What you'd still have to establish is that
the public key really is the provider's ([file 02](02-trust-anchors.md)), that their private key hadn't leaked,
and, if timing matters, when it was signed — signatures in this course carry no trusted
time.

**6. "A signature is just a hash plus a key."**

The difference isn't strength; it's **who can do what**. With HMAC, the set of parties who
can check equals the set who can create: every verifier is also a potential forger. With
a signature, checking needs only the public key, so you can hand verification to the whole
world without handing anyone the ability to create. That's what makes signatures work for
third parties and disputes: a verifier can't have made the signature themselves. A
perfectly guarded HMAC key still has to be *shared with every verifier*, so it can never
give you that.

---

## [File 02](02-trust-anchors.md) — Trust anchors

**1. Everything from one host.**

You've learned that *the key on that page* signed *that text* — nothing about whether
that key is the claimed person's. An attacker controlling the host can write new text,
sign it with their own key, and replace the published key; your check passes identically.
It's the hash-next-to-the-data mistake one rung up. You need the key from somewhere the
attacker doesn't also control.

**2. Why cross-checking against GitHub helps.**

Because the two sources are **independently secured**: taking over the signer's website
doesn't give you their GitHub account, and vice versa. An attacker who changes only one
creates a disagreement a careful verifier sees. It doesn't stop an attacker who
compromises both, or GitHub itself serving the wrong key, and it doesn't help a verifier
who never checks the second source. Independence, not GitHub's authority, is doing the
work.

**3. The stolen laptop.**

One key for everything: the thief can log in to every server with that key in
`authorized_keys`, push to every repository the account can write to, sign files and
statements as the owner, and make commits GitHub marks Verified (if the key is registered
for signing and they use the owner's verified committer email). Separate keys, signing key
only on the laptop: signatures and Verified commits, yes; logging in to servers or
pushing over SSH, no — they'd need another credential to get commits *onto* GitHub. The
blast radius shrank from "everything" to "signatures".

**4. Removing a leaked key from GitHub.**

No. GitHub stores a persistent verification record when it first verifies a commit and
"will not re-verify previously signed commits or retroactively adjust their verification
status" when the key changes. A plausible reason: a badge that flips whenever someone
rotates a key would make history unstable and punish routine rotation. The consequence for
you: the badge on commits made during the leak window can't be undone by revocation, so
the speed of detection and removal is what bounds the damage. You may also need to
identify and revert those commits.

**5. What CT gives you.**

**Visibility**, not prevention. Every certificate browsers will accept ends up in public
append-only logs, so a misissued certificate for your domain is publicly recorded and can
be found. RFC 9162 §11.2 says the logs themselves don't detect misissuance; they rely on
interested parties to monitor. So it only helps you if **you (or a service acting for
you) watch the logs** for certificates for your domains that you didn't request, and act
when one appears.

---

## [File 03](03-ssh-signatures.md) — SSH signatures in practice

**1. `verify -I` versus `find-principals`.**

`verify` answers "did this named signer, whom I trust for this name and namespace, sign
this exact message?" — it looks up keys for the principal you give, checks the namespace
and validity options, and checks the signature over the message. `find-principals` answers
a different question: "which entries in my trust file own the key embedded in this
signature?" It doesn't look at the message at all. `verify` makes you name the signer up
front because the signature contains a key, not a name; without `-I` you'd be asking
"was this signed by *someone* in my file?", which lets any trusted key stand in for any
other — a key trusted for one person could vouch for content claimed to be another's.

**2. Gating on `check-novalidate`.**

The attacker generates their own key, writes whatever file they like, and signs it with
`ssh-keygen -Y sign -n file`. The signature embeds the attacker's public key.
`check-novalidate` checks the signature against that embedded key — it verifies — and exits
0. The CI job proceeds. At no point did anything compare the key to a trusted list. The
fix is `-Y verify` with an `allowed_signers` file under your control.

**3. Relabelling the namespace.**

On verification, `sshsig.c` rebuilds the signed data using the namespace the verifier
expects (`git`, passed with `-n` by git) and also refuses outright if the blob's namespace
differs. With the field edited to `git`, the strings match, so the second check passes —
but the signature was computed over a structure containing `file`. The verifier computes
over `git`, gets different signed data, and the cryptographic check fails. If the namespace
were only stored in the blob but *not* signed, it would be a mere label: editing it would
be undetectable, and the namespace would protect nothing. Its value comes entirely from
being inside the signed bytes.

**4. Identical Ed25519 signatures on two days.**

Ed25519 is deterministic and SSHSIG adds no randomness, so identical signatures from the
same key mean identical signed data: the same message, namespace and hash algorithm (to
cryptographic certainty). About *when*, you can conclude nothing — no timestamp is signed,
so a signature made yesterday and one made a year ago for the same message are the same
bytes; one may simply be a copy of the other. With ECDSA, two independent signings of the
same message produced different bytes in our test, so identical ECDSA signatures would
strongly suggest a copy of a single signature rather than two signings.

**5. `namespaces="file"` versus the protocol namespace.**

The protocol binding stops a signature from being **relabelled**: a signature made for
`file` can't be presented as a `git` one. The allowed-signers restriction stops a key from
being **used** in other namespaces at all, as far as this verifier is concerned: even a
fresh, genuine `git` signature by that key is refused. You notice the difference when the
key leaks (or is misused by a tool): a thief can make brand-new signatures in any
namespace — the protocol is happy — but your verifiers still only accept that key for
`file`.

**6. Following the publisher's own verification instructions.**

The reader has proven the signature matches the key Alice's page told them to use —
which is self-certification if the key came from the same place as the content ([file 02](02-trust-anchors.md)
§2.1). The extra step: get the key, or its fingerprint, from an **independent source** —
for instance GitHub's `ssh_signing_keys` endpoint for Alice's account — and confirm it
matches before trusting the result.

---

## [File 04](04-signing-in-automation.md) — Signing in automation

**1. `umask` before, not `chmod` after.**

`umask` applies at file creation, so the key file is born owner-only. With `chmod`
afterwards, the file is created with the default permissions (often world-readable, 0644)
and exists that way until the `chmod` runs. Any other process or user on the machine
watching the directory can open it in that window, and an open file descriptor survives a
later `chmod`. On a shared runner that's a real exposure. (`ssh-keygen` also refuses to use
a key that's still too readable, so getting this wrong can fail the job as well.)

**2. Verifying against a key derived from the secret.**

A public key derived from the CI secret always matches the CI secret, so that check
passes even when the secret is the **wrong key** — an old key, a test key, one that was
never published. Every reader's verification would fail, and your gate wouldn't notice.
Verifying against the committed `allowed_signers` (the file readers use) catches it,
because it checks the thing the reader checks.

**3. Why "sign exited 0" isn't enough.**

Any two of: the secret holds a key that isn't in the published `allowed_signers`; a later
step changes the bytes (whitespace trimming, newline normalisation, template rendering)
after signing; the signature was made with a different namespace from the one readers are
told to use; the published `allowed_signers` wasn't updated after a rotation. In each case
signing succeeded and every reader's verify fails. Only running the reader's check on the
final bytes catches all of them.

**4. `valid-before` after a leak.**

A reader verifying an old, legitimate signature today: the verification time defaults to
now, which is after `valid-before`, so it **fails** — "key has expired". An attacker who
persuades a reader to pass `-O verify-time=` with last year's date gets their forgery
**accepted**, because the time is inside the window. Both are wrong for the same reason:
an SSH signature contains no timestamp, so the verifier has no evidence of when it was
made. `valid-before` can only compare *the verification time* to a date; it can't separate
pre-leak from post-leak signatures. Re-signing what you still stand behind with a new key
is the real fix.

**5. Recreating `Don't forget: $5 off.\n`.**

`echo 'Don't forget: $5 off.'` — the apostrophe in "Don't" closes the single-quoted
string; the rest is parsed as shell, and the trailing `'` opens an unterminated quote: a
syntax error. `echo "Don't forget: $5 off."` — the apostrophe is fine now, but `$5` is
expanded as the fifth positional parameter, almost certainly empty, so you get
`Don't forget:  off.`. Wrong bytes, signature fails, and no error message. The base64
version is `echo '<b64>' | base64 -d > msg.txt`: the base64 alphabet has no quotes, `$`,
backticks or backslashes, so single quotes are always safe, and decoding restores every
byte, including the apostrophe, the dollar sign and the trailing newline. (Exact output
from `printf "Don't forget: \$5 off.\n" | base64` is the thing to paste; generate it, don't
type it.)

**6. Where the verify gate goes.**

After the formatter, immediately before publishing, on the exact bytes that will be
published. Placed before the formatter, it would report success — the signature does
match the pre-formatting bytes — and you'd publish formatted bytes that no reader can
verify. That's the reason for the rule "verify the bytes you publish, not an intermediate
copy". Better still, fix the ordering so signing happens last, after all transformations.

---

## [File 05](05-git-commit-signing.md) — Commit signing with SSH keys

**1. Before and after signing.**

Before: nothing. Author and committer are free-text config; anyone can claim to be you.
After you sign: *your* commits now carry evidence of your key, but a reader who never
checks signatures sees exactly what they saw before — and a forged, unsigned commit with
your name looks the same to them as before. Signing only helps verifiers. That's the
argument for vigilant mode or required signed commits: they make the absence of a
signature visible or blocking.

**2. `allowed_signers` committed in the repository.**

It's self-protecting only if the branch **enforces signed, verified commits** — then
changing the file requires a commit signed by a key already in it, which is the property
git's docs describe. Without enforcement, anyone with push access can add their own key to
the file in an unsigned commit, and from then on their signatures verify as trusted. The
trust file is then protected by push access, not by signatures.

**3. What "Verified" doesn't establish.**

Any three of: that the **author** field is true (GitHub binds the committer); that the key
wasn't stolen; that the code was reviewed or is safe; that the commit was made at the time
it claims; that the key is still trusted today (persistent records keep old verdicts after
removal); that the person behind the GitHub account is who you think.

**4. Uploaded as an authentication key only.**

GitHub keeps authentication keys and signing keys as separate lists and checks commit
signatures against **signing** keys. The key is on the account, but not registered for
signing, so GitHub finds no signing key matching the signature (the API's `unknown_key`
reason, "has not been registered with any user's account", is the closest documented
code). Upload the same public key again as a signing key.

**5. The backdated commit.**

Git passes `-Overify-time=<commit timestamp>` to `ssh-keygen` so that signatures made while
a key was valid stay valid after rotation. But the commit timestamp is part of the commit,
written by whoever made it. Set it inside the key's validity window and the
`valid-before` check passes, so a leaked key can still produce verifying commits. A plain
`ssh-keygen -Y verify` of a file signature uses the **verifier's** clock by default, and
only a time the *verifier* chooses to pass. The signer can't influence it. The lesson:
validity windows are only as trustworthy as whoever chooses the verification time, and it
should never be the party whose signature you're checking.

---

## Things to try

1. **Rebuild §1.3 yourself.** Copy the CRC-24 C code from RFC 9580 §6.1.1, check it gives
   `21CF02` for `123456789`, then change a message and recompute. Feel how little it
   protects.
2. **Decode a signature by hand.** Take any `.sig` from `ssh-keygen -Y sign`, base64-decode
   the middle, and label every field against `PROTOCOL.sshsig` §2 with `xxd`. Then change
   one byte of the signature field and confirm `verify` fails.
3. **Forge with `check-novalidate`.** Generate a second key, sign someone else's message,
   and watch `check-novalidate` print "Good". Then watch `verify` and `find-principals`
   refuse it.
4. **Build the CI gate.** In a scratch repository, write a workflow that signs a file with a
   key from a secret, verifies against a committed `allowed_signers`, and only then
   uploads an artifact. Break it four ways (wrong key, reformatting step, wrong namespace,
   stale trust file) and confirm each fails the job.
5. **Backdate a commit.** Reproduce [file 05 §5.4](05-git-commit-signing.md) in a scratch repository with a throwaway
   key. Then put the key in `gpg.ssh.revocationFile` and see what changes.
6. **Cross-check a real key.** Pick a public GitHub user who signs commits, fetch
   `GET /users/{username}/ssh_signing_keys`, build an `allowed_signers` line, and verify one
   of their commits locally with `git verify-commit`.

---

## Questions worth asking me

- "Walk me through the SSHSIG signed-data structure byte by byte for my own signature."
- "How does an OpenPGP signature differ from an SSHSIG one — what does each carry that the
  other doesn't?"
- "Show me what the `string` encoding in RFC 4253 looks like and why length prefixes
  matter for preventing ambiguity."
- "How would I add a trustworthy timestamp to an SSH signature? What are the options and
  what does each trust?"
- "What's the difference between SSH certificates (`cert-authority`) and plain keys in
  `allowed_signers`, and when would I want a CA?"
- "Why is Ed25519 deterministic while ECDSA historically isn't, and what went wrong when
  ECDSA nonces were reused?"
- "How do webhook providers typically structure HMAC signatures — and how do they stop
  replays?"
- "Is my mental model right? Here's what I think a signature proves: …"
