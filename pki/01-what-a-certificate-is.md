# 1. Foundations — certificates, issuers, and what a CA actually is

**Skip this if** you can already say, without looking anything up, why the `issuer` field of a
certificate is only a hint while the signature is the proof, and what a certificate has to
assert before its key is allowed to sign another certificate. Everything later in this course
assumes those two.

## The problem is the word "authority"

Here is a public key:

```
04 8d 1a 7f  … 65 bytes of elliptic curve point …
```

Nothing in those bytes says whose it is. They are a number. A number has no owner.

So the question every certificate exists to answer is: **whose key is this?** And the standard
answer is "a Certificate Authority says so" — which is where most people's mental model goes
wrong, because the phrase invites you to picture an institution. A building. A company with a
compliance department.

That picture makes the mechanics impossible to reason about. You end up asking questions like
"is DigiCert a CA?" or "is Google the CA for Android?", which sound sensible and have no
mechanical answer. The questions that *do* have answers are much smaller, and this chapter is
about getting to them.

[`http/05`](../http/05-certificates-and-trust.md) covers why vouching beats a global directory,
in a TLS setting. This chapter is the layer under that: what the object is made of, what the
issuer relationship actually consists of, and why a chain exists at all.

## 1.1 Three roles, and only three

A certificate involves exactly three roles. Keeping them apart is most of the work.

| Role | Who it is | What they hold |
|---|---|---|
| **Subject** | whoever the certificate is *about* | the private key matching the certificate's public key |
| **Issuer** | whoever *signed* the certificate | a different private key, the one that made the signature |
| **Relying party** | whoever is *reading* it, later, to decide something | nothing but the certificate and a trust anchor |

Two things follow immediately, and both matter.

**These are roles in one certificate, not kinds of organisation.** The same keypair is the
subject of one certificate and the issuer of the next. That is not a coincidence or an
optimisation — *that relationship is what a chain is*. A chain is a list of certificates where
each one's subject is the next one's issuer.

**The relying party was not present.** Nobody was online when the certificate was signed, and
nobody needs to be online now. A certificate is a statement addressed to no one in
particular, signed once, checkable by anyone, forever. That offline-checkability is the whole
reason the format won.

## 1.2 What's in one, grouped by the question it answers

Field-by-field tables of X.509 are everywhere and they teach badly, because the fields arrive
in encoding order rather than in the order you need them. Here they are grouped by what you'd
actually be asking.

| The question you're asking | The field that answers it | RFC 5280 |
|---|---|---|
| Whose key is this? | `subject`, and `subjectAltName` for machine names | §4.1.2.6, §4.2.1.6 |
| What *is* the key? | `subjectPublicKeyInfo` | §4.1.2.7 |
| Who says so? | `issuer` — a **name** | §4.1.2.4 |
| …and which of their keys? | `authorityKeyIdentifier` (AKI) | §4.2.1.1 |
| Why should I believe any of it? | `signatureValue`, over the DER of everything above | §4.1.1.3 |
| For how long? | `validity` — `notBefore`, `notAfter` | §4.1.2.5 |
| Which certificate is this, for revocation? | `serialNumber` — unique **per issuer**, not globally | §4.1.2.2 |
| What is the key allowed to do? | `keyUsage`, `extendedKeyUsage` | §4.2.1.3, §4.2.1.12 |
| **May this key sign other certificates?** | `basicConstraints` | §4.2.1.9 |
| How will a child certificate point back at me? | `subjectKeyIdentifier` (SKI) | §4.2.1.2 |

Read the right-hand column of the last three rows again. `keyUsage`, `basicConstraints` and
`subjectKeyIdentifier` are the fields that turn a certificate from a *statement about a key*
into a *link in a chain*. Without them every certificate is a leaf.

## 1.3 The issuer is a name. The CA is a key.

This is the section that answers "what exactly is an issuer CA", and it is one idea.

The `issuer` field holds a **Distinguished Name** — a structured string, something like
`C=US, O=Google LLC, OU=Android, CN=Key Attestation CA1`. It is a label. The issuer chose it.

An attacker can put that exact string in a certificate they generated five seconds ago. It
costs nothing. There is no registry, no uniqueness check, no enforcement. The same is true of
the AKI: it's an identifier the issuer picked, and a forger can copy it byte for byte.

What a forger cannot produce is a **signature that verifies under the issuer's public key**.

```
    ┌──────────────────────── the certificate ────────────────────────┐
    │                                                                 │
    │  issuer:  "C=US, O=Google LLC, CN=Key Attestation CA1"          │  ← a HINT.
    │  AKI:     52:32:bb:2c:fb:46:43:9b…                              │    freely forgeable.
    │                                                                 │    tells you which
    │                                                                 │    key to TRY.
    │  …                                                              │
    │                                                                 │
    │  signatureValue:  30:65:02:30:44:df:8c:f3…                      │  ← the PROOF.
    │                                                                 │    verifies under
    └─────────────────────────────────────────────────────────────────┘    exactly one key,
                                                                           or under none.
```

So the two fields do completely different jobs:

| | `issuer` + AKI | `signatureValue` |
|---|---|---|
| Answers | "which key should I try?" | "was it that key?" |
| Is | a lookup hint | the only evidence in the document |
| If forged | you look up the wrong key, and the signature then fails | cannot be forged without the private key |
| Can be omitted | yes, at the cost of trying every anchor you hold | no |

And that gives the definition the whole course runs on:

> **A CA is a keypair.** "The issuer CA of this certificate" means *the keypair whose private
> half produced this signature*. Not a company. Not a building. A keypair, plus a certificate
> somewhere else that says that keypair is allowed to do this.

> **Teacher's aside.** The phrase "certificate authority" is doing two unrelated jobs and it's
> worth prising them apart permanently. There is an **organisation** — which has auditors,
> a certificate policy, a legal identity, and in a public PKI a browser-root-programme
> relationship. And there is a **key** — which signs. The organisation is why anyone agreed to
> trust the key; the key is the only part that participates in verification. When you verify a
> chain, the organisation is not involved in any step. This is why "is Acme a CA?" has no
> mechanical answer, and why **"may the key in *this* certificate sign certificates?"** has a
> precise one, below. Every time a PKI question feels unanswerable, check whether you've
> accidentally asked about the organisation.

## 1.4 What makes a certificate allowed to issue

A keypair may sign certificates when its own certificate says so. Two fields, and both are
checked during path validation (RFC 5280 §6.1.4):

| Field | Value needed | What it does |
|---|---|---|
| `basicConstraints` | `cA` **TRUE** | the permission itself. Absent or FALSE means this key may not issue certificates |
| `basicConstraints` | `pathLenConstraint` *n* | at most *n* further CA certificates may appear **below** this one. `0` means it may only issue leaves |
| `keyUsage` | `keyCertSign` | where `keyUsage` is present at all, it must include this bit |

This is not decoration. It is the only thing standing between "I have a valid certificate for
my own website" and "I can issue a certificate for anyone's website". A validator that checks
signatures and dates but skips `basicConstraints` will happily accept a chain where an
ordinary leaf certificate signed a certificate for something else entirely — which means every
holder of any valid certificate from that CA becomes a CA.

> ⚠️ `cA: TRUE` is a **claim inside a certificate**, so it is only as good as the signature
> over it and the chain above it. A self-signed certificate asserting `cA: TRUE` asserts
> nothing — anyone can make one. The claim gains force only when a key you already trust
> signed it.

## 1.5 Finding the issuer again when there are several

`issuer` narrows to an organisation's name; AKI narrows to *which of its keys*. Long-lived
issuers have several live at once — key changeover, new platform generations, renames — which
is exactly the case RFC 5280 §4.2.1.1 says AKI exists for.

This is not a theoretical nicety. [File 04](04-running-a-trust-store.md) §4.2 inspects a real
national eID scheme whose CA set contains **two certificates with byte-identical subject DNs
and different keys** — so name-based lookup there is a coin flip, and AKI → SKI matching is the
only thing that resolves it.

The mechanics of matching a leaf's AKI against a CA's SKI, what happens when you're missing
one of an issuer's CAs, and why two CA certificates that *print* identically can be different
names to a strict library, are worked through in detail in
[`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) §Which CA, when the
issuer has several. It's the same problem wherever you meet it; that's the best write-up of it
in this repo and this chapter won't repeat it.

## 1.6 Where the chain stops, and why that part isn't cryptography

Follow issuers upward and you must eventually stop. The certificate you stop at is the
**trust anchor**, and the reason you stop there is not in the certificate. It is that *you put
it there*.

```
  root / trust anchor      ← you possessed this before you started. Configuration.
       │ signs
  intermediate CA          ← cryptography from here down
       │ signs
  leaf / end-entity
```

Anchors are usually distributed as self-signed certificates, which confuses people into
thinking the self-signature is checked. It isn't — RFC 5280 §6.1 takes the anchor as an
*input* and excludes a self-signed anchor certificate from the path. There is a good
explanation of why that's correct rather than lazy in
[`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) §What "chains to" means,
and the families of ways to come to trust a key at all — out-of-band comparison, TOFU,
independent cross-checks, a vouching authority — are in
[`signatures/02`](../signatures/02-trust-anchors.md) §2.2.

The practical sign that anchors are a different kind of object: clients bend the rules for
them — **and disagree about which rules.** That disagreement is worth knowing, because it
decides whether an expired root works for you.

Nobody checks an anchor's *signature*. OpenSSL's `X509_verify_cert(3ssl)` is explicit that
"the signature on the chain's terminating certificate is not verified: trust is taken from its
presence in the trust store rather than from its signature". But the same page continues:
"**Validity periods are checked on every certificate, including the terminator.**" And it does
— an expired root really is refused:

```
$ openssl verify -CAfile expired-root.pem -untrusted inter.pem leaf.pem
C=GB, O=Example Scheme, CN=Example Expired Root CA
error 10 at 2 depth lookup: certificate has expired
```

Android takes the opposite view. Let's Encrypt's write-up of the DST Root CA X3 expiry notes
that "Android has intentionally chosen not to use the `notAfter` field of trust anchors"
([letsencrypt.org, December 2020](https://letsencrypt.org/2020/12/21/extending-android-compatibility/)),
which is exactly why an expired root kept working there after it had stopped working elsewhere.

Who's right? Android has the better of the argument on principle — an expiry date is a
*statement by an issuer*, and an anchor has no issuer above it to make that statement
meaningfully; you stop trusting an anchor by removing it, not by waiting. OpenSSL has the
better of it on caution, since a date is at least *some* signal and ignoring it silently is its
own hazard. What matters operationally is that **you cannot assume either behaviour**: the same
expired root is trusted by one stack and refused by another. File 04 §4.5 is about living with
that.

File 03 is largely about this one asymmetry, because *where the anchor comes from* is the thing
that differs most between the chains you'll actually meet.

## 1.7 The signature is over bytes, not over meanings

`signatureValue` covers the DER encoding of the `tbsCertificate` — the exact octets, not the
values they represent. Two encodings of the same logical certificate are two different signed
objects.

That's why certificates use **DER** (Distinguished Encoding Rules), the subset of BER that
permits exactly one encoding per value (ITU-T X.690 §11.1). If senders could choose, verifiers
would re-encode differently and disagree about what was signed.

The consequence you will meet in the field is that certificates are rejected for their
*encoding* rather than their contents, and one library accepts what another refuses. The
canonical case — a `BOOLEAN` written `01` where DER demands `FF` — and which libraries do what
about it, is in [`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) §Why one
library refuses what another accepts.

The rule to carry forward: **you may never repair a certificate whose signature you intend to
check.** Repairing an anchor is sometimes fine, because its signature is never checked. Those
are the only two cases and the difference is exactly §1.6.

## 1.8 The vocabulary

Everything above, named. These words get used interchangeably in the wild and mostly
shouldn't be.

| Term | Means |
|---|---|
| **End-entity certificate**, **leaf** | a certificate whose key is *not* allowed to sign certificates. The bottom of a chain |
| **Intermediate**, **subordinate CA** | a CA certificate that is itself issued by another CA |
| **Root** | a self-signed CA certificate, by convention the top |
| **Trust anchor** | the key a verifier *starts from*, because it was configured. Usually a root, but it can be an intermediate |
| **Certification path**, **chain** | an ordered list of certificates, each issued by the previous one, the first issued by the anchor |
| **Path building** | *finding* such a list. A search (file 02) |
| **Path validation** | *checking* one. An algorithm, RFC 5280 §6.1 (file 02) |
| **Relying party** | the verifier. The party the whole system exists to serve, and the only one with a decision to make |
| **Cross-certificate** | a certificate one CA issues for *another CA's* existing public key, giving that key a second path upward |
| **PKI** | the whole arrangement: keys, certificates, the policies behind them, and the distribution of anchors |

Note what is *not* in that list: "certificate" alone tells you nothing about which of these a
file is. `openssl x509 -text -noout -in whatever.pem` and read `basicConstraints`.

## Check yourself

1. You receive a certificate whose `issuer` reads `CN=Key Attestation CA1, O=Google LLC`.
   State precisely what that tells you, and what it would take for the field to be worth
   anything.
2. A certificate's `serialNumber` is `1`. Someone concludes it must be the first certificate
   ever issued by that CA, and someone else concludes the certificate is fake. Both are
   unjustified — why, and what *is* the serial for?
3. A validator checks, for every certificate in a path: the signature verifies under the
   issuer's key, and the current time is within `validity`. It does not look at
   `basicConstraints`. Construct the attack this permits, starting from an ordinary leaf
   certificate the attacker legitimately owns.
4. Explain why a self-signed certificate asserting `cA: TRUE` and `CN=Some National Root CA`
   proves nothing at all, and then explain how the *identical file* becomes load-bearing.
5. A colleague wants to fix a CA certificate that a strict parser rejects by editing one byte,
   and apply the same fix to the end-entity certificates that fail the same way. One of those
   is defensible and one destroys the thing being checked. Say which, and give the one-sentence
   reason that decides it.
6. "Is Acme Ltd a certificate authority?" Rewrite that as a question with a mechanical answer,
   and say which bytes you would look at.

(Answers in [`06-exercises.md`](06-exercises.md).)
