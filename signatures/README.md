# Signatures — What Integrity Mechanisms Actually Prove

A course on the question underneath every "signed", "verified" and "checksummed" label:
**how do I know this text is what someone actually said?** It climbs the ladder from
encodings and checksums through hashes and MACs to digital signatures, asks what each rung
proves and to whom, then turns to practice: SSH signatures by hand, in CI and in git, and
signing structured data — JSON — so a verifier in another language can check it.

**This is reference learning material.** It's about the published standards and the
behaviour of the tools, not about any particular system you might be building. Nothing
here is project-specific.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- I start from the problem each mechanism solves, and *why* before *how*.
- Every file ends with **Check yourself** questions. Answers are in `08-exercises.md`.
- Command outputs are real, from OpenSSH 10.0p2 and Git 2.50 on macOS, using throwaway
  keys; the serialiser outputs in file 06, and the code in files 01 §1.5.1, 04 §4.8 and 06,
  were run with Go 1.24.4, Node 26, Rust (`ed25519-dalek` 2.2, `hmac` 0.12) and libsodium. Where I couldn't verify something, the text says so.

## The primary sources

| Source | What it settles | Link |
|---|---|---|
| **RFC 4648** | Base64 and its alphabet | <https://www.rfc-editor.org/rfc/rfc4648> |
| **RFC 4880** (obsolete) | OpenPGP, including the CRC-24 armour checksum (§6, §6.1) | <https://www.rfc-editor.org/rfc/rfc4880> |
| **RFC 9580** | OpenPGP, July 2024; obsoletes 4880; makes CRC-24 optional and discouraged (§6.1) | <https://www.rfc-editor.org/rfc/rfc9580> |
| **FIPS 180-4** | SHA-256 | <https://csrc.nist.gov/pubs/fips/180-4/upd1/final> |
| **RFC 2104** | HMAC; truncated output (§5) | <https://www.rfc-editor.org/rfc/rfc2104> |
| **NIST SP 800-107 Rev. 1** (withdrawal decided 2022) | HMAC tag length and forgery likelihood (§5.3.3, §5.3.5) | <https://csrc.nist.gov/pubs/sp/800/107/r1/final> |
| **RFC 8725** | JWT best current practice: algorithm pinning, key entropy | <https://www.rfc-editor.org/rfc/rfc8725> |
| **RFC 8032** | Ed25519; determinism (§8.2) | <https://www.rfc-editor.org/rfc/rfc8032> |
| **RFC 6962 / RFC 9162** | Certificate Transparency v1 / v2 (both Experimental) | <https://www.rfc-editor.org/rfc/rfc6962>, <https://www.rfc-editor.org/rfc/rfc9162> |
| **`PROTOCOL.sshsig`** | The SSHSIG format: blob, signed data, namespaces | <https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.sshsig> |
| **`sshsig.c`** | How OpenSSH actually signs and verifies | <https://github.com/openssh/openssh-portable/blob/master/sshsig.c> |
| **ssh-keygen(1)** | `-Y` operations, `-O` options, ALLOWED SIGNERS | <https://man.openbsd.org/ssh-keygen.1> |
| **OpenSSH release notes** | When each signing feature arrived | <https://www.openssh.com/releasenotes.html> |
| **git-config(1)** | `gpg.format`, `user.signingKey`, `gpg.ssh.*` | <https://git-scm.com/docs/git-config> |
| **GitHub docs** | Published keys, signing keys, "Verified", Actions secrets | <https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification> |
| **RFC 8785** (Informational) | JCS, the JSON canonical form | <https://www.rfc-editor.org/rfc/rfc8785> |
| **RFC 7515 / RFC 7797** | JWS; detached content (App. F); unencoded payload (`b64`) | <https://www.rfc-editor.org/rfc/rfc7515>, <https://www.rfc-editor.org/rfc/rfc7797> |
| **RFC 7518 §3.4** | ES256 signatures as fixed-length `R‖S` | <https://www.rfc-editor.org/rfc/rfc7518#section-3.4> |
| **RFC 9421 / RFC 9530** | HTTP Message Signatures; `Content-Digest` | <https://www.rfc-editor.org/rfc/rfc9421>, <https://www.rfc-editor.org/rfc/rfc9530> |
| **Go `encoding/json`** | Map-key sorting, HTML escaping, `SetEscapeHTML`, `UseNumber` | <https://pkg.go.dev/encoding/json> |

## Reading order

### Part 1 — Foundations (prerequisite for everything else)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations — what each mechanism proves](01-foundations.md) | Say, for base64, CRC, SHA-256, HMAC and signatures, who can create one, who can check one, and what it proves; design a compact MAC-signed token and say how short its tag may be |
| 2 | [Trust anchors](02-trust-anchors.md) | Explain why a signature is only as good as your copy of the public key, and how to anchor it |

### Part 2 — SSH signatures

| # | File | After this you can… |
|---|------|---------------------|
| 3 | [**SSH signatures in practice**](03-ssh-signatures.md) | ⭐ Sign and verify with `ssh-keygen -Y`, read an SSHSIG blob, write `allowed_signers`, avoid the `check-novalidate` trap |
| 4 | [Signing in automation](04-signing-in-automation.md) | Sign from CI without leaking the key, gate publishing on verification, rotate correctly, move exact bytes through a shell, and choose asymmetric keys over a shared secret between services |
| 5 | [Commit signing with SSH keys](05-git-commit-signing.md) | Configure git for SSH signing and say exactly what GitHub's "Verified" does and doesn't mean |

### Part 3 — Signing data formats

Needs Part 1 and the exact-bytes section of file 04 (§4.7); the SSH-specific chapters aren't a prerequisite.

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Signing JSON across languages](06-signing-json.md) | Say why "same JSON" isn't "same bytes", choose between shipping the bytes, JWS and JCS, recognise ECDSA encoding and low-S mismatches, and write test vectors both sides run |

### Reference

| # | File | |
|---|------|--|
| 7 | [Glossary](07-glossary.md) | Look things up |
| 8 | [Exercises & answers](08-exercises.md) | Worked answers to every Check yourself, plus things to try |

## If you're short on time

- **20 minutes:** [file 01 §1.7](01-foundations.md) (the table) and the Teacher's aside under it, then [file 02 §2.1](02-trust-anchors.md).
- **You're about to sign things with SSH keys:** 01 → 03 → 04.
- **You're wiring signing into a pipeline:** 03 §3.6, then all of 04.
- **You just want git commits to show "Verified" and to know what that's worth:** 02 §2.4, then 05.
- **Two services in different languages must agree on a signature:** 01 §1.6, 04 §4.7, then all of 06.

## The one-paragraph summary of everything

Every integrity mechanism is defined by **who can create it and who can check it**.
Base64 and checksums are keyless and public, so anyone can produce a valid one; they catch
accidents, never attackers. A cryptographic hash pins exact bytes, but only if the digest
reaches you over a channel the attacker can't touch — a hash printed next to the data
proves nothing about who made it. HMAC adds a shared secret, which stops outsiders but
makes every verifier a potential forger, so it suits two parties who share a key (webhooks)
or a service talking to its own later step (a fixed-layout, versioned token with a truncated
tag compared in constant time), and can't give the public "anyone can verify, only I can
sign". Digital signatures split the key: the private key signs, the public key verifies, and the remaining question becomes
**is this really their public key?** — answered by trust anchors such as independent
cross-checks, with signing keys kept separate from login keys to limit the damage of a
leak. OpenSSH's `ssh-keygen -Y` signs arbitrary data in the SSHSIG format, binding a
**namespace** into the signed bytes so a signature can't be reused across purposes, and
verifies against an `allowed_signers` file the verifier controls; `check-novalidate` trusts
the key embedded in the signature and must never be used as a gate. Signatures carry no
time, so validity windows are only as good as whoever picks the verification clock — in
git, that's the committer. In automation: keep a dedicated key as a CI secret, write it
with `umask 077`, delete it after, and **never publish a signature the pipeline hasn't just
verified the way a reader will**, on the exact bytes — trailing newline included. Between
services, a shared secret configured twice drifts and one shipped to clients is public,
which is the operational case for asymmetric keys with a pinned algorithm, a startup
pairing check and a signer interface. The same
exact-bytes rule is why signing JSON across languages fails: JSON is a data model with many
byte spellings, and serialisers differ silently on key order, escaping, whitespace and
numbers; either ship the signed bytes untouched (or in a JWS), or have both sides implement
a specified canonical form (RFC 8785), and pin it with shared test vectors — including the
signature's own encoding, since ECDSA has several.

## How to use me

Ask anything, including:

- "Explain why a hash next to the data proves nothing, but simpler"
- "Walk me through an SSHSIG blob byte by byte"
- "Draw the trust chain from a signature to a person, for my setup"
- "What would break if I used HMAC here instead of a signature?"
- "Here's my token layout — what happens when I need to add a field?"
- "Is my mental model right? Here's what I think `Verified` means…"
- "Show me how an OpenPGP signature compares with an SSH one"
- "My Go and Node services disagree on a signature — here are both outputs, what differs?"

I'll add files here as we go if a topic earns its own page.
