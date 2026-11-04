# Certificate Chains — A Course

What a certificate actually is, what an "issuer CA" actually means, how a chain gets *built*
before it gets *checked*, why the same machinery answers four completely different
questions depending on where you meet it, and how a signature stays verifiable after its
certificate has expired.

**This is reference learning material.** It's about RFC 5280, RFC 4158 and the published
behaviour of real verifiers — not about any particular system you might be building. Nothing
here is project-specific.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- *Why* before *how*: each chapter opens with the problem that made the mechanism necessary.
- Every file ends with **Check yourself** questions. Answers are in
  [`07-exercises.md`](07-exercises.md).
- Command outputs are real, from OpenSSL 3.6.4 on macOS, against a throwaway chain you can
  rebuild — the recipe is in the exercises file.

## Why this folder exists separately

Four other courses here each needed a slice of certificate theory and explained that slice
locally, in the accent of its own domain. That's four partial explanations and no foundation:

| Course | Its slice |
|---|---|
| [`http/05`](../http/05-certificates-and-trust.md) | TLS: the three questions a client asks, where the OS trust store lives, why an empty container trusts nothing |
| [`signatures/02`](../signatures/02-trust-anchors.md) | the families of ways to come to trust a key at all, and Certificate Transparency |
| [`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) | X.509 in anger: AKI/SKI matching, DER-vs-BER refusals, OCSP's scope |
| [`android/11`](../android/11-key-attestation-in-depth.md) | attestation: a chain whose payload is in an extension |

This course is the layer underneath all four. It does **not** repeat them — where one of them
is the best treatment of something, these chapters link to it and move on. `smartcards/07` in
particular goes deeper on X.509 edge cases than anything here, and is worth reading after
file 02 regardless of whether you care about cards.

## The primary sources

| Source | What it settles | Link |
|---|---|---|
| **RFC 5280** | the X.509 profile; path validation (§6.1) — the normative algorithm | <https://www.rfc-editor.org/rfc/rfc5280> |
| **RFC 4158** | certification path **building** — informational, and explicit that building is unspecified | <https://www.rfc-editor.org/rfc/rfc4158> |
| **ITU-T X.690** | BER and DER; why one encoding per value matters | <https://www.itu.int/rec/T-REC-X.690> |
| **RFC 6960** | OCSP — what a revocation answer does and doesn't cover | <https://www.rfc-editor.org/rfc/rfc6960> |
| **RFC 3161** | time-stamping: the request, the token, the TSA's obligations, the signature-time-stamp attribute (Appendix A) | <https://www.rfc-editor.org/rfc/rfc3161> |
| **ETSI EN 319 142-1 V1.2.1** | PAdES: the four baseline levels (§6), the DSS (§5.4.2), document time-stamps (§5.4.3) | <https://www.etsi.org/deliver/etsi_en/319100_319199/31914201/01.02.01_60/en_31914201v010201p.pdf> |
| **ETSI EN 319 102-1 V1.4.1** | AdES validation: proof of existence, best-signature-time, the INDETERMINATE sub-indications (§5.5) | <https://www.etsi.org/deliver/etsi_en/319100_319199/31910201/01.04.01_60/en_31910201v010401p.pdf> |
| **`openssl-verification-options(1)`** | what a real verifier does and which checks are off by default | <https://docs.openssl.org/master/man1/openssl-verification-options/> |
| **ICAO PKD** | CSCA master lists — how travel-document trust anchors are distributed, and on what cadence | <https://www.icao.int/icao-pkd/icao-master-list> |
| **ETSI TS 119 612** | the EU Trusted List format; `NextUpdate` bounded at 6 months (§5.3.15) | <https://www.etsi.org/deliver/etsi_ts/119600_119699/119612/01.02.01_60/ts_119612v010201p.pdf> |
| **`X509_verify_cert(3ssl)`** | how OpenSSL *builds* a path, its alt-chain retry, and which checks an anchor escapes | `man 3 X509_verify_cert` |
| **AOSP, Key and ID attestation** | the attestation extension's schema | <https://source.android.com/docs/security/features/keystore/attestation> |

## Reading order

Read 1 then 2. They build.

| # | File | After this you can… |
|---|---|---|
| 1 | [Foundations — certificates, issuers, and what a CA actually is](01-what-a-certificate-is.md) | Say what an issuer CA *is*, why `issuer` is a hint and the signature is the proof, and what a certificate must assert before its key may sign another |
| 2 | [**Building a chain, and walking it**](02-building-and-walking-a-chain.md) | ⭐ Tell path *building* failures from path *validation* failures, read OpenSSL's error codes for which job broke, and explain why a chain one verifier builds another won't |
| 3 | [Four chains, compared](03-four-chains-compared.md) | Say what "verify the certificate" means in web TLS, an eID card, Android attestation and APK signing — and why one of those has no chain at all |
| 4 | [**Running a trust store**](04-running-a-trust-store.md) | ⭐ Answer "how many CAs do I mount, and who updates them?" — anchors vs intermediates, the real counts, the three rotation clocks, and whether a self-updating store is a good idea |
| 5 | [Signatures that outlive their certificate](05-signatures-that-outlive-their-certificate.md) | Say what a timestamp proves and what it doesn't, read an RFC 3161 token, place PAdES B-B / B-T / B-LT / B-LTA, and spot the re-save that breaks every PDF signature |

Reference, not reading:

| # | File | |
|---|---|---|
| 6 | [Glossary](06-glossary.md) | Every term, in one place |
| 7 | [Exercises & answers](07-exercises.md) | Worked answers to every **Check yourself**, plus the recipe for the test chain and things to try |

## If you're short on time

| You want | Read |
|---|---|
| The whole model in twenty minutes | the summary below, then file 01 §1.3 |
| To debug "unable to get local issuer certificate" right now | file 02 §2.5, then §2.4 |
| To stop confusing app signing with attestation | file 03 §3.3 and §3.4, then [`android/11`](../android/11-key-attestation-in-depth.md) |
| To understand what your verifier is *not* checking | file 02 §2.2 and §2.6 |
| To size the trust store you have to operate | file 04 §4.1 and §4.2 |
| To decide whether to let your trust store self-update | file 04 §4.6 |
| To decide where an allow-list should live (config or datastore) | file 04 §4.8 §Churn rate |
| To know whether a signed PDF will still verify in ten years | file 05 §5.4, then the table in §5.8 |

## The one-paragraph summary of everything

A certificate is a signed claim binding a **public key** to a **name**, made by an **issuer**
and read later by a **relying party** who was not present — and those are roles in one
certificate, not kinds of organisation, which is why the same key is the subject of one
certificate and the issuer of the next. That relationship *is* a chain. The single most
load-bearing idea is that the `issuer` field is a **name** — freely forgeable, useful only as a
hint about which key to try — while the **signature** is the only evidence in the document; so
"the issuer CA" means *the keypair whose private half signed this*, and a CA is a keypair with
a certificate saying it may sign certificates (`basicConstraints` `cA`, plus `keyCertSign`),
which is the check that stops any ordinary certificate holder from minting certificates for
anything. Turning a leaf and a bag of certificates into a chain is **path building** — a
*search*, deliberately left unspecified by the standards (RFC 4158), so implementations
disagree and a chain one verifier finds another won't; checking a chain once built is **path
validation** — an *algorithm* (RFC 5280 §6.1), so implementations agree. Reading an error for
which of those two jobs failed is most of debugging: `unable to get local issuer certificate`
means you are missing a certificate, not that theirs is bad. Every chain terminates at a
**trust anchor**, whose own signature is never checked, because what makes it an anchor is that
*you configured it* — the one part of the system that is administration rather than
cryptography, and therefore the part that differs most between schemes. Which is the sting:
identical machinery answers four different questions. Web TLS asks whether a key controls a
domain, with hundreds of anchors any of which may vouch for any name. An eID card asks who a
person is, with a handful of anchors pinned from out of band. Android attestation asks where a
key *lives*, leaving its subject name an empty constant and putting the whole payload in an
extension. And APK signing has **no chain at all** — self-signed, no CA, no issuer to collect —
anchored by a pinned digest. In all of them a chain establishes provenance and only a **freshly
signed challenge** proves possession, so a chain without a challenge accepts a copied
certificate and a challenge without a chain accepts a key the attacker made up. Operationally
the thing you mount is **anchors, not CAs** — intermediates arrive in the chain — so the store
is hundreds of certificates only for web TLS, where the OS updates them for you, and is two
certificates valid into the 2040s for Android attestation; what rotates frequently is the
layer *below* your anchors, which is the hierarchy doing its job, and a store that churns
usually means you anchored too low. If you want to approve each issuing CA rather than inherit
them, the move is an **allow-list against the chain your verifier built** — not relocating your
anchor, which costs the same effort and silently retires every check above it — pinned on the
**SPKI hash** so routine renewal and cross-signing don't break it; and where such a list feels
unmaintainable, the fix is usually to find the one column doing the churning and replace it
with a predicate, as a version **floor** replaces a list of versions. Finally, everything above
is checked **at a time**, and for a signature that has to verify for years the time that matters
is when it was made — which the signature itself cannot prove, because the only date in it is
the signer's claim. A **timestamp** from a third-party TSA over the signature value is the proof
of existence that lets a verifier say "revoked, but after it signed"; without one, revocation or
expiry makes an old signature not invalid but **indeterminate**. Embedding the chains and
revocation data (PAdES B-LT, including the TSA's own) keeps that check possible after the CA's
services disappear, and renewing document timestamps (B-LTA) keeps it possible after the TSA's
certificate and the algorithms age — all of it appended as incremental updates, because a PDF
signature covers exact bytes and any full re-save after signing breaks it.

## How to use me

These notes are a starting point. Good questions to bring back:

- "Here's a chain and a trust store — walk me through what my verifier will do with it."
- "`openssl verify` says error 20. Which job failed, and what am I missing?"
- "Why does this validate with `openssl` and fail in Go?"
- "Should I anchor on this scheme's root or its current issuing CA? What do I give up either way?"
- "Is this thing I'm checking a chain at all, or should I be pinning a digest?"
- "Draw the two paths a cross-signed intermediate creates, for my trust store."
- "What does 'certificate verified' in this log line actually license me to conclude?"
- "How many anchors does this scheme actually need me to mount, and what arrives in the chain?"
- "Should this trust store self-update? Walk me through the supply-chain risk for my case."
- "Here's a signed PDF — which PAdES level is it, and what will a validator say about it in ten years?"
- "Our signer has no outbound network. Which long-term validation options are left, and what does each cost?"
