# 6. Glossary

Reference, not reading. Skim once, then come back when a term turns up.

## The ladder

| Term | Meaning |
|---|---|
| **Encoding** | A public, reversible change of representation. No key; proves nothing. Base64 is one ([file 01 §1.2](01-foundations.md)) |
| **Base64** | Bytes written with 64 printable characters (`A–Z a–z 0–9 + /`, `=` padding). RFC 4648 §4 |
| **Checksum** | A short value from a public function, for catching *accidental* corruption |
| **CRC** | Cyclic redundancy check — a checksum family designed for channel errors |
| **CRC-24** | The 24-bit CRC in OpenPGP's ASCII-armour footer. Described in RFC 4880 §6; optional and discouraged in RFC 9580 §6.1 |
| **Cryptographic hash** | A keyless public function with a fixed-size output, designed so finding collisions or second preimages is infeasible. SHA-256 is one (FIPS 180-4) |
| **Digest** | A hash function's output |
| **Collision resistance** | Infeasible to find *any* two inputs with the same digest |
| **Second-preimage resistance** | Infeasible to find a *different* input with the same digest as a given one |
| **MAC** | Message authentication code — a tag computed with a secret key; the same key creates and checks it |
| **HMAC** | The standard MAC built from a hash function. RFC 2104 |
| **Digital signature** | A tag made with a private key and checkable with the matching public key |
| **Key pair** | A private key (signs, kept secret) and a public key (verifies, published) |

## Trust

| Term | Meaning |
|---|---|
| **Trust anchor** | A key you accept on grounds outside the signature system; every chain of verification ends in one |
| **Fingerprint** | A short hash of a public key, for comparing keys by eye. `ssh-keygen -lf` |
| **Out-of-band** | Through a separate channel from the one being protected |
| **TOFU** | Trust on first use: accept the first key seen, alarm on change |
| **Cross-checking** | Confirming a key against several independently run sources |
| **Authentication key** | A key used to log in (e.g. `git push` over SSH). Listed by `GET /users/{username}/keys` on GitHub |
| **Signing key** | A key used to sign data. GitHub lists them separately: `GET /users/{username}/ssh_signing_keys` |
| **Blast radius** | What an attacker can do with one leaked secret |
| **Certificate authority (CA)** | A party that signs certificates binding public keys to names |
| **Certificate Transparency (CT)** | Public, append-only logs of issued TLS certificates so misissuance can be noticed. RFC 6962 (v1), RFC 9162 (v2) — both Experimental |
| **SCT** | Signed certificate timestamp — a log's signed promise that a certificate has been or will be logged |
| **Merkle tree** | A tree of hashes letting a log prove an entry is included and that history wasn't rewritten |

## SSH signatures

| Term | Meaning |
|---|---|
| **SSHSIG** | OpenSSH's signature format. Spec: `PROTOCOL.sshsig` in openssh-portable |
| **Armoured signature** | The `-----BEGIN SSH SIGNATURE-----` text form: header, base64 blob, footer |
| **Detached signature** | A signature kept in a separate file from the message |
| **Magic preamble** | The bytes `SSHSIG` at the start of both the blob and the signed data |
| **Namespace** | A string naming the signature's purpose (`file`, `email`, `git`, `name@your.domain`). Included in the signed data; stops cross-protocol reuse |
| **Cross-protocol attack** | Getting a signature made for one purpose accepted for another |
| **`hashalg`** | Hash applied to the message before signing: `sha512` (default) or `sha256` |
| **Principal** | A `USER@DOMAIN` identity in an allowed signers file |
| **Allowed signers file** | The verifier's list of trusted keys: principals, options, key type, key. ssh-keygen(1) ALLOWED SIGNERS |
| **`namespaces=`** | Allowed-signers option restricting a key to some namespaces |
| **`valid-after` / `valid-before`** | Allowed-signers options bounding when a key is accepted — against the *verification* time |
| **`verify-time`** | `-O` option setting the time used for those checks; defaults to now |
| **`-Y sign`** | Produce a signature |
| **`-Y verify`** | Check a signature against a named principal in an allowed signers file. Exit 0 = good |
| **`-Y find-principals`** | Look up which principals own the key in a signature |
| **`-Y match-principals`** | Look up allowed-signers entries matching a name |
| **`-Y check-novalidate`** | Check structure and maths against the *embedded* key only. Not a trust check |
| **KRL** | Key revocation list — OpenSSH's compact format for revoked keys |
| **Deterministic signature** | Same key and message always give the same signature. True of Ed25519 (RFC 8032 §8.2) |

## Automation and git

| Term | Meaning |
|---|---|
| **umask** | Process setting masking permission bits off newly created files. `077` = owner only |
| **Verify before publish** | Running the reader's exact verification in the pipeline and failing if it doesn't pass |
| **Rotation** | Replacing a key with a new one, and updating everywhere it's trusted |
| **`gpg.format`** | Git setting choosing `openpgp` (default), `x509` or `ssh` |
| **`user.signingKey`** | Git setting naming the key to sign with |
| **`gpg.ssh.allowedSignersFile`** | Git setting naming the trust file used to verify SSH signatures |
| **`gpgsig`** | The commit-object header holding the signature |
| **Verified (GitHub)** | GitHub checked the signature against a key registered on an account |
| **Vigilant mode** | GitHub setting that flags your unsigned commits as Unverified |
| **Persistent verification record** | GitHub's stored verdict for a commit, kept even after the key is removed |
