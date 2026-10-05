# 3. SSH signatures in practice

## 3.1 The problem

You want "only I can sign, anyone can verify" ([file 01 §1.6](01-foundations.md)) for ordinary files and text.
The traditional tool is OpenPGP (GnuPG), which brings its own key format, keyrings, trust
model and a lot of options. Most engineers already have something simpler installed and
already manage keys for it: OpenSSH.

Since **OpenSSH 8.1** (October 2019), `ssh-keygen` can sign and verify arbitrary data with
ordinary SSH keys. The release notes describe it as "an experimental lightweight signature
and verification ability… Signatures embed a namespace that prevents confusion and attacks
between different usage domains (e.g. files vs email)"
([OpenSSH release notes](https://www.openssh.com/releasenotes.html), 8.1). `find-principals`
arrived in 8.2 and `match-principals` in 8.9. Git builds its SSH commit signing on exactly
this machinery ([file 05](05-git-commit-signing.md)).

Primary sources for this chapter:

- the **SSHSIG** format: [`PROTOCOL.sshsig`](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.sshsig)
  in openssh-portable;
- the commands: [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen.1), sections on `-Y`,
  `-O` and **ALLOWED SIGNERS**;
- the implementation: [`sshsig.c`](https://github.com/openssh/openssh-portable/blob/master/sshsig.c).

Outputs below are real, from OpenSSH 10.0p2 on macOS, with throwaway keys.

## 3.2 Sign and verify, end to end

```sh
ssh-keygen -t ed25519 -N '' -C 'alice@example.com' -f k   # throwaway key pair: k, k.pub
printf 'I wrote this.\n' > msg.txt
ssh-keygen -Y sign -f k -n file msg.txt                     # writes msg.txt.sig
```

```
Signing file msg.txt
Write signature to msg.txt.sig
```

```
$ cat msg.txt.sig
-----BEGIN SSH SIGNATURE-----
U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg3aQU0A1vzwflbZBJP8OF2+2Jt+
NTKiMUwcUPT8IuJ8YAAAAEZmlsZQAAAAAAAAAGc2hhNTEyAAAAUwAAAAtzc2gtZWQyNTUx
OQAAAEADJEQji+OHuBBlbcOCasz9+CMb/bSHJ0X89hp5gXzOLDCZNhdm/kNv2900BRu7UJ
lOBBb/AgIW0/YEfDHpbewD
-----END SSH SIGNATURE-----
```

That's a **detached** signature — it lives in its own file, and the message is unchanged.
To verify, the verifier needs three things besides the message and signature: a list of
keys they trust (`-f allowed_signers`), the identity they expect (`-I`), and the namespace
(`-n`):

```
$ ssh-keygen -Y verify -f allowed_signers -I alice@example.com -n file -s msg.txt.sig < msg.txt
Good "file" signature for alice@example.com with ED25519 key SHA256:30/gF7srI49GpVd44oIkphGD6NdBu9bSjQx4G8ffRsM
$ echo $?
0
```

The man page is explicit that the **exit status** is the result: "Successful verification
by an authorized signer is signalled by ssh-keygen returning a zero exit status." Scripts
should test that, not grep the message.

Note the message goes in on **standard input**. Verification is over the exact bytes you
feed it — [file 04 §4.7](04-signing-in-automation.md) is about how easily those bytes change.

## 3.3 What's inside the armour

The armoured block is a header, base64, and a footer. Decode the middle and you get the
**SSHSIG blob**, laid out in `PROTOCOL.sshsig` §2:

```
byte[6]   MAGIC_PREAMBLE   "SSHSIG"
uint32    SIG_VERSION      0x01
string    publickey
string    namespace
string    reserved
string    hash_algorithm
string    signature
```

(`string` is SSH's wire encoding: a 4-byte big-endian length, then the bytes.) Here's the
blob from §3.2:

```
$ sed '1d;$d' msg.txt.sig | tr -d '\n' | base64 -d | xxd | head -6
00000000: 5353 4853 4947 0000 0001 0000 0033 0000  SSHSIG.......3..
00000010: 000b 7373 682d 6564 3235 3531 3900 0000  ..ssh-ed25519...
00000020: 20dd a414 d00d 6fcf 07e5 6d90 493f c385   .....o...m.I?..
00000030: dbed 89b7 e353 2a23 14c1 c50f 4fc2 2e27  .....S*#....O..'
00000040: c600 0000 0466 696c 6500 0000 0000 0000  .....file.......
00000050: 0673 6861 3531 3200 0000 5300 0000 0b73  .sha512...S....s
```

| Bytes | Field | Value here |
|---|---|---|
| `53 53 48 53 49 47` | magic | `SSHSIG` |
| `00 00 00 01` | version | 1 |
| `00 00 00 33` + 51 bytes | public key | `ssh-ed25519` + 32-byte key |
| `00 00 00 04` `file` | namespace | `file` |
| `00 00 00 00` | reserved | empty |
| `00 00 00 06` `sha512` | hash algorithm | `sha512` |
| `00 00 00 53` + 83 bytes | signature | `ssh-ed25519` + 64-byte Ed25519 signature |

Two things worth noticing:

- **The signer's public key is inside the signature.** That's convenient (the verifier can
  look it up) and dangerous (it's attacker-supplied). §3.6 is about that danger.
- **The signer's *name* is not.** There's no identity in the blob. "alice@example.com" in
  the verify output came from the verifier's `allowed_signers` file, not from the
  signature.

### What actually gets signed

The private key doesn't sign your message directly. `PROTOCOL.sshsig` §3 says it signs
this structure:

```
byte[6]   MAGIC_PREAMBLE   "SSHSIG"
string    namespace
string    reserved
string    hash_algorithm
string    H(message)
```

So the message is first hashed — `sha512` by default (`-O hashalg=sha256` is the only
alternative; [ssh-keygen(1) `-O`](https://man.openbsd.org/ssh-keygen.1)) — and the digest
is wrapped with the preamble and namespace. The spec gives the reason for pre-hashing:
"to limit the amount of data presented to the signature operation, which may be of concern
if the signing key is held in limited or slow hardware or on a remote ssh-agent."

The `SSHSIG` preamble is there "to ensure that manual signatures can never be confused
with any message signed during SSH user or host authentication."

A small spec-versus-practice discrepancy: §1 says the base64 "SHOULD be broken up by
newlines every 76 characters", but both the spec's own example and the output above wrap
at **70**. Nothing depends on it — verifiers ignore the line breaks — but don't write a
parser that assumes 76.

## 3.4 Namespaces: why a signature knows what it's for

Suppose one key signs both release files and git commits. Without anything else in the
signed data, a signature is just "Alice's key signed these bytes". If someone could
arrange for bytes Alice signed as a *file* to also be meaningful as, say, a *commit* or an
*email*, they could take her file signature and present it in the other context. That's a
**cross-protocol attack**: a signature valid in one domain accepted in another.

The **namespace** closes it. `PROTOCOL.sshsig`: "The purpose of the namespace value is to
specify a unambiguous interpretation domain for the signature, e.g. file signing. This
prevents cross-protocol attacks caused by signatures intended for one intended domain
being accepted in another. The namespace value MUST NOT be the empty string."

The crucial detail is *where* the namespace lives. It's in the blob (so you can see it),
but it's also **inside the signed data** (§3.3). And on verification, `sshsig.c` rebuilds
the signed data using the namespace **the verifier expects** — the one passed with `-n` —
not the one the blob claims, and separately refuses if the two differ:

```
$ ssh-keygen -Y verify -f allowed_signers -I alice@example.com -n git -s msg.txt.sig < msg.txt
Couldn't verify signature: namespace does not match
Could not verify signature.
```

So editing the namespace field in the blob doesn't help an attacker: the signature was
computed over `file`, and no amount of relabelling makes it a signature over `git`.

Namespaces are arbitrary strings. The man page names `file` and `email` and recommends
"names following a NAMESPACE@YOUR.DOMAIN pattern" for custom uses. Git uses `git` (it
passes `-n git`; see `gpg-interface.c` in [git/git](https://github.com/git/git)).

| Namespace | Used by |
|---|---|
| `file` | General file signing (man page example) |
| `email` | Email signing (man page example) |
| `git` | Git commit and tag signatures |
| `something@your.domain` | Your own applications — recommended form for custom uses |

## 3.5 The `allowed_signers` file

The verifier's trust anchors ([file 02](02-trust-anchors.md)) live in an **allowed signers** file. The format is
in [ssh-keygen(1), ALLOWED SIGNERS](https://man.openbsd.org/ssh-keygen.1#ALLOWED_SIGNERS):
"Each line of the file contains the following space-separated fields: principals, options,
keytype, base64-encoded key." It's deliberately modelled on `authorized_keys`.

```
# principals          options                    keytype      key
alice@example.com     namespaces="file"          ssh-ed25519  AAAAC3NzaC1lZDI1NTE5AAAAIN2k...
```

| Field | Meaning |
|---|---|
| **principals** | Comma-separated `USER@DOMAIN` patterns. `-I` on the verify command must match one |
| **options** (optional) | Comma-separated, no spaces outside quotes, case-insensitive keywords |
| `namespaces="…"` | Only accept this key for these namespaces |
| `valid-after=` / `valid-before=` | Only accept the key for signatures checked at times in this window (§3.8, [file 04 §4.6](04-signing-in-automation.md)) |
| `cert-authority` | The key is a CA; accept certificates it signed |
| **keytype, key** | The public key, as in a `.pub` file |

The `namespaces=` option gives you key-level policy on top of the protocol-level
namespace. A key restricted to `file` refuses even a perfectly valid `git` signature:

```
$ ssh-keygen -Y verify -f allowed_signers -I alice@example.com -n git -s git.sig < msg.txt
allowed_signers:1: key is not permitted for use in signature namespace "git"
Could not verify signature.
```

Note what the file is: a **local decision** by the verifier about which keys they trust
for which names. The signer doesn't control it. Publishing your own `allowed_signers` line
is a convenience for readers — and, if they fetch it from the same place as the signed
content, exactly the same-channel problem as [file 02 §2.1](02-trust-anchors.md).

## 3.6 The verbs, and the trap in `check-novalidate`

`ssh-keygen -Y` has five operations. They answer different questions, and confusing them
is a real bug.

| Operation | Inputs | Question it answers |
|---|---|---|
| `sign` | key, namespace, message | — produces a signature |
| `verify` | allowed_signers, identity `-I`, namespace, signature, message | "Did **this named, trusted signer** sign this message in this namespace?" |
| `find-principals` | allowed_signers, signature | "Which principals in my trust file own the key in this signature?" |
| `match-principals` | allowed_signers, identity `-I` | "Which entries in my trust file match this name?" |
| `check-novalidate` | namespace, signature, message | "Is this signature mathematically valid **for the key embedded in it**?" |

That last one deserves its man-page description in full: it "Checks that a signature
generated using ssh-keygen -Y sign has a valid structure. This does not validate if a
signature comes from an authorized signer."

Watch what that means when an attacker, Mallory, signs Alice's message with Mallory's own
key:

```
$ ssh-keygen -Y verify -f allowed_signers -I alice@example.com -n file -s m.sig < msg.txt
Could not verify signature.                                   # exit 255
$ ssh-keygen -Y find-principals -f allowed_signers -s m.sig
No principal matched.                                         # exit 255
$ ssh-keygen -Y check-novalidate -n file -s m.sig < msg.txt
Good "file" signature with ED25519 key SHA256:UAX/ROPs8SanI0NqczyXH9qZ25RwMv1FwecL9nBKQQ4
$ echo $?
0
```

> ⚠️ **`check-novalidate` says "Good" to anyone's signature.** The public key is inside the
> signature, so a structurally valid signature always checks out against *its own* key —
> that's [file 02 §2.1](02-trust-anchors.md)'s swapped-key attack, automated. Never gate anything on it. Use
> `verify` with an `allowed_signers` file you control. `check-novalidate` is for debugging
> ("is this file even well-formed?"), and the "novalidate" in its name means exactly that.

`find-principals` is the honest way to ask "who signed this?" when you don't know: it
looks the embedded key up in *your* trust file. A natural pattern is find-principals to
get the name, then `verify -I <that name>` to check the signature.

## 3.7 Same message, same key, same signature?

Sign `msg.txt` twice with the Ed25519 key and compare:

```
$ cmp first.sig msg.txt.sig && echo IDENTICAL
IDENTICAL
```

That's expected. [RFC 8032 §8.2](https://www.rfc-editor.org/rfc/rfc8032#section-8.2):
"EdDSA signatures are deterministic. This protects against attacks arising from signing
with bad randomness." And SSHSIG adds no randomness of its own: the signed structure in
§3.3 is preamble, namespace, reserved, hash name and message digest — no nonce, no salt,
no timestamp. Same inputs in, same bytes out.

It is **not** a property of SSH signatures in general. The same experiment with an ECDSA
key gave two different signatures, which is what you'd expect if a fresh random value is
drawn per signature (classic ECDSA does this). I haven't traced OpenSSH's ECDSA code path
to confirm the mechanism, so treat that as an observation, not a guarantee. And switching the hash (`-O hashalg=sha256`) changes the output even for
Ed25519, because the hash name and digest are part of what's signed.

| Key type | Same message, same settings, twice | Observed here |
|---|---|---|
| Ed25519 | Identical (RFC 8032 §8.2, plus no randomness in SSHSIG) | Identical |
| ECDSA (`-t ecdsa`) | Different | Different |

Why you might care:

- **Reproducibility.** With Ed25519 a pipeline can re-sign unchanged content and produce
  an unchanged signature file — no spurious diffs.
- **Equality leaks.** Two identical Ed25519 signatures from the same key reveal that the
  same message was signed. Usually harmless; occasionally not.
- **No proof of freshness.** A byte-identical signature can't show it was made *today*.
  Nothing in an SSH signature says when it was made.

## 3.8 What a signature doesn't carry: time

That last point generalises. An SSHSIG contains no timestamp. So when `allowed_signers`
says `valid-before="20260101"`, against *what time* is that checked? The man page's
`verify-time` option answers it: by default, the **current time** at verification, unless
the verifier passes `-O verify-time=…`.

```
$ ssh-keygen -Y verify -f as_file -I alice@example.com -n file -s first.sig < msg.txt
as_file:1: key has expired: verify time 2026-10-05T09:36:16 > valid-before 2026-01-01T00:00:00
Could not verify signature.
$ ssh-keygen -Y verify -f as_file -I alice@example.com -n file -O verify-time=20250601 -s first.sig < msg.txt
Good "file" signature for alice@example.com with ED25519 key SHA256:30/gF7srI49GpVd44oIkphGD6NdBu9bSjQx4G8ffRsM
```

The verifier picks the time, and the signature has no say. That's fine when the verifier
knows independently when something was signed. It's a problem when the time comes from
somewhere the signer controls — which is exactly what git does ([file 05 §5.4](05-git-commit-signing.md)), and why
key rotation after a leak is subtler than it looks ([file 04 §4.6](04-signing-in-automation.md)).

---

## Check yourself

1. `verify` needs `-I alice@example.com`, but `find-principals` doesn't need any identity.
   What is each command actually checking, and why does `verify` insist you name the
   signer up front?
2. A CI job checks incoming signed files with `ssh-keygen -Y check-novalidate -n file` and
   proceeds on exit 0. Describe an attack that passes, step by step.
3. An attacker takes a valid `file` signature, decodes it, changes the namespace field to
   `git` (adjusting the length prefix), re-armours it, and presents it with the same bytes
   as a commit signature. Trace what the verifier does and where it fails. What would
   change if the namespace were stored in the blob but *not* included in the signed data?
4. You see two signatures from the same Ed25519 key on two different days that are
   byte-identical. What can you conclude about the messages? What can you conclude about
   *when* each was made? Would your answer change for an ECDSA key?
5. Your `allowed_signers` line restricts a key to `namespaces="file"`. Someone argues this
   is redundant because the protocol already binds the namespace. What does each mechanism
   protect against, and when would you notice the difference?
6. Alice publishes `msg.txt` and `msg.txt.sig` and, next to them, "verify with: `<her
   allowed_signers line>`". A reader follows those instructions exactly. What has the
   reader proven, and what one extra step would change that?

(Answers in [`07-exercises.md`](07-exercises.md).)
