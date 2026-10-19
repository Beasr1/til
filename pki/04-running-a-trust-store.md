# 4. Running a trust store

Assumes files [01](01-what-a-certificate-is.md)–[03](03-four-chains-compared.md).

## The problem

Files 01–03 are about what a verifier does. This one is about what you have to *operate*, and
it exists because of a question with a surprising answer:

> How many CA certificates do I actually mount in my backend? Hundreds? They rotate — is there
> a self-updating feed?

Reasonable, and it comes from web PKI instincts, where the answer really is "hundreds, and
your operating system quietly updates them". For the other three chains in file 03 that answer
is wrong by one to two orders of magnitude, and for one of them the correct number is zero.

The spread is the lesson:

| Chain | Anchors you mount | Update mechanism |
|---|---|---|
| Web TLS | **~130–190** | your OS package manager, invisibly |
| eID / travel documents | **a handful per scheme** | signed lists you re-fetch on a cadence |
| Android attestation | **2** | effectively never; you pin them |
| APK signing | **0** — it isn't a chain | no mechanism; a pinned digest you change by hand |

## 4.1 You mount anchors, not CAs

Most of the confusion dissolves here. "All the CAs" names two different populations with
completely different operational needs.

| | **Anchors** | **Intermediates** |
|---|---|---|
| Where they must be at verification time | **already in your store** | anywhere — they arrive *in the chain*, or you fetch them via AIA |
| Who supplies them | you, out of band | usually the party presenting the certificate |
| How many | few | more, and they churn |
| Do you mount them? | **yes** | **no** — and doing so changes your policy (file 02 §2.6) |

So an issuer with 3 roots and 12 live issuing CAs is **3 anchors**, not 15. The twelve arrive
in the chain. Counting them into your store is the single most common way people arrive at
"hundreds" for a scheme that needs three.

> **Teacher's aside.** There's a reason the hierarchy exists at all, and it's the same reason
> your store stays small. Roots are kept offline and used rarely — file 01 §1.6's damage
> control — precisely *so that* the certificates doing daily work can be replaced without
> anybody touching a trust store. That is the design working. If you find yourself updating
> anchors often, you have either anchored too low (§4.4) or you are tracking the wrong
> population. **A trust store that churns is a symptom, not a workload.**

## 4.2 The actual numbers

Worth measuring rather than assuming. On this laptop, today, three different programs disagree
about how many web PKI roots exist:

```
/etc/ssl/cert.pem                              128 certificates   (the OpenSSL bundle)
/opt/homebrew/etc/ca-certificates/cert.pem     193 certificates   (Homebrew's ca-certificates)
SystemRootCertificates.keychain                158 certificates   (macOS itself)
```

Three stores, one machine, no two agreeing — which is file 02's "a chain one verifier builds,
another won't" showing up as an *inventory* difference rather than a search difference. "About
a hundred and something" is the right order of magnitude for web TLS and there is no single
true number.

Against that, Google's attestation roots, fetched live from
`https://android.googleapis.com/attestation/root`:

```
2 roots   (2,690 bytes of JSON)
  serialNumber=f92009e853b6b045                            RSA 4096,  2022-03-20 → 2042-03-15
  C=US,O=Google LLC,OU=Android,CN=Key Attestation CA1      ECDSA P-384, 2025-07-17 → 2035-07-15
```

Two. Valid to 2042 and 2035. That is the whole trust store for Android key attestation, and it
is not going to keep you busy.

### A real anchor set, measured

Abstract claims about "a handful, across key changeovers" are easy to make. Here is an actual
one: the issuing CA set of a national eID scheme — the UAE scheme that
[`smartcards/06`](../smartcards/06-proving-genuineness.md) already uses as a worked example —
inspected with `openssl x509`. Six certificates, and every structural claim in files 01 and 02
shows up in them.

| Issued | Subject (RFC2253, abbreviated) | Expires |
|---|---|---|
| 2013-12 | `SN=5, C=AE, O=Emirates Identity Authority, OU=PRIDC, CN=Citizen and Resident…` | 2026-12 |
| 2016-12 | `SN=6, …` same O and OU | 2029-12 |
| 2019-12 | `SN=7, …` same O and OU | 2032-12 |
| 2019-07 | `CN=Citizen and Resident…, OU=Identity, O=Identity And Citizenship Authority, C=AE, serialNumber=1` | 2032-07 |
| 2021-07 | …`serialNumber=1` — **the same DN as the row above** | 2034-07 |
| 2024-07 | …`serialNumber=4` | 2037-07 |

Five things worth taking from that table.

**All six are self-signed roots**, and a card's certificate is issued *directly* by one of them —
subject equals issuer on the CA, and the card's chain is a **path of one**. So this scheme's
anchor count and its CA count are the same number, which is not true in general (§4.1) and is
why you have to look rather than assume.

**The organisation was renamed mid-scheme.** Three CAs say `O=Emirates Identity Authority,
OU=PRIDC`; three say `O=Identity And Citizenship Authority, OU=Identity`. The `CN` is unchanged
throughout. A verifier matching on `CN` alone sees one authority; there are two naming regimes
and six keys.

**The generation marker moved to a different attribute.** The older three carry it in
`SN` — OID 2.5.4.4, *surname* — and the newer three in `serialNumber`, OID 2.5.4.5. Same
intent, different attribute type, so any parsing that reached for one of them breaks at the
generation boundary.

**Two certificates share a byte-identical subject DN and have different keys.** Both the 2019-07
and 2021-07 certificates hash to the same `openssl x509 -subject_hash` (`3e24970c`), and their
subject key identifiers differ. This is the exact situation RFC 5280 §4.2.1.1 says the AKI
extension exists for, encountered in production:

> ⚠️ With two anchors sharing a DN, **selecting a CA by subject name alone is ambiguous** — you
> have a 50% chance of picking the wrong key, and the symptom is a signature failure that looks
> like a bad certificate. AKI → SKI matching is not an optimisation here; it is the only thing
> that disambiguates. This is also why OpenSSL's hashed CA directories name files
> `<subject_hash>.0`, `<subject_hash>.1` and so on — the numeric suffix exists precisely because
> subject-hash collisions are expected. A trust store keyed on subject DN, with no suffix
> handling, silently holds one of these two.

**The cadence is visible.** A new CA roughly every two to three years, each with a **13-year**
validity. Six live at once is the steady state of those two numbers, and it is the whole
answer to "how many" for this scheme: not hundreds, and not three — six, because a long
validity period and a steady issuance rate multiply.

## 4.3 How each one updates

| Chain | Who publishes | Cadence | How you get it |
|---|---|---|---|
| **Web TLS** | root programmes (Mozilla's is the one most bundles derive from) | continuous, in the background | your base image's `ca-certificates` package. **Genuinely self-updating** — it's an OS upgrade |
| **Travel documents** | ICAO PKD master lists | "updated at 3-monthly intervals as appropriate" ([ICAO PKD](https://www.icao.int/icao-pkd/icao-master-list)) | download the signed master list and re-extract. Note ICAO says there is *no obligation* on participants to publish new CSCAs within any period — so gaps are expected |
| **EU qualified certificates** | the European Commission's LOTL, pointing at each member state's list | `NextUpdate` must be no more than **6 months** after issue (ETSI TS 119 612 §5.3.15) | fetch the LOTL, verify its XML signature, follow it to the national lists, verify each |
| **Android attestation** | Google, as JSON | the roots barely move; a new one was announced to begin issuing 1 February 2026 — the first change in years | fetch once, pin, review on announcement. Google's own library ships them as a **static `roots.json`**, not a live fetch |
| **APK signing** | nobody | when a developer rotates their key | they tell you, or your users break |

Two observations that matter more than the table.

**The signed-list formats are the real "self-updating feature".** A master list and the LOTL
are not just downloads — they are *signed* bundles of anchors, which means re-fetching them is
a cryptographically checkable operation rather than an act of faith. That is what distinguishes
a trust-distribution mechanism from a URL.

**For Android attestation, the thing you poll is not the roots.** It's the revocation list, and
it has a very different rhythm — see §4.6.

## 4.4 Rotation: three different clocks

"They rotate frequently" is true, but of a layer you aren't storing. Separate the clocks:

| What rotates | How often | Does your store change? |
|---|---|---|
| **Anchors / roots** | years to decades. Google's attestation roots: 20-year and 10-year validity | **yes** — but rarely, and with warning |
| **Issuing CAs / intermediates** | months to years. Routine | **no**, if you anchored at the root. **Yes, every time**, if you pinned the intermediate |
| **Leaf certificates** | days to months. RKP attestation certs: "valid for up to two months before they expire and are rotated" | no |
| **Revocation data** | daily | not the store — but you must refresh it |

The middle row is the one with a real decision in it, and it's file 02 §2.6's trade-off in
operational form:

- **Anchor at the root** — the scheme can add and retire issuing CAs without telling you, and
  everything keeps working. You inherit CAs you never approved.
- **Anchor at the intermediate** — you approve each issuing CA explicitly, and nothing new
  works until you add it. Tighter, and now the scheme's rotation schedule is *your* operational
  calendar, with an outage waiting at the end of every missed one.

Neither is wrong. But "I'll pin the intermediate, it's stricter" is where "my trust store
needs constant updating" usually comes from — and it's a choice, not a property of PKI.

And it's a false binary: if what you want is to *approve each issuing CA*, there's a third
option that costs the same effort and keeps more checking. That's §4.8.

## 4.5 Expiry, and a store that is correct but stale

File 01 §1.6 showed that implementations **disagree** about whether an anchor's validity period
is checked: OpenSSL refuses an expired anchor (`error 10 at 2 depth`), Android deliberately
ignores `notAfter` on anchors. That disagreement is a live operational hazard, not trivia.

It means an anchor's expiry date can cause an outage on some of your stacks and not others,
and the failure is well-signposted only if you were already looking. Two consequences worth
building in:

- **Alert on anchor expiry well ahead** — months, not days. An anchor cannot be renewed by
  anyone upstream; replacing it is a change *you* make.
- **Don't infer from one stack.** Your Go service and your OpenSSL-based sidecar can disagree
  about the same expired root, which looks like a bug in one of them and is a difference in
  policy.

And the inverse case, from Android attestation: pre-2021 devices carry attestation certificates
whose chains terminate at a root Google tells you to trust "regardless of certificate validity
period". A verifier that enforces anchor expiry strictly rejects a large population of
perfectly honest devices. Correctness here is a per-scheme policy question, not a universal
rule.

## 4.6 Should the store update itself?

The honest trade, because "self-updating" is not free:

| Approach | Gets you | Costs you |
|---|---|---|
| **Baked into the image** | one artefact; nothing to fail at runtime; the store is reviewable in a diff | a rebuild and redeploy to change anything |
| **Mounted / fetched at startup** | rotation without a rebuild | a runtime dependency that can be misconfigured or unavailable — and `SSL_CERT_FILE` *replaces* the list rather than adding to it ([`http/05`](../http/05-certificates-and-trust.md)) |
| **Fetched on a schedule** | tracks signed lists (LOTL, master lists) properly | **a supply chain** — see below |
| **Inherited from the OS** | genuinely zero effort, the right answer for web TLS | you trust your base image's maintainers, and a minimal image has *zero* roots |

> ⚠️ **A self-updating trust store is a channel into the thing your whole system bottoms out
> on.** Whoever controls the feed controls what you trust — which is the trust-anchor problem
> recursing one level up. The mitigations are specific:
>
> - **Prefer signed bundles.** The LOTL and ICAO master lists are signed precisely so the
>   update is verifiable; verify the signature against a pinned signer and refuse on failure.
> - **For an unsigned JSON endpoint** — Google's roots list is plain HTTPS — your anchor for the
>   anchors is the web PKI. That's usually acceptable for *two certificates you could equally
>   pin by hand*, which is exactly why pinning them is the better call here.
> - **Fail closed on anchors, deliberately on revocation.** If an anchor fetch fails, keep the
>   last good set and alert — never start with an empty store, because "no anchors" and
>   "untrusted" produce the same error (file 02 §2.5). If a *revocation* fetch fails, decide
>   consciously whether you accept or refuse; both are defensible and the default usually
>   isn't a decision.
> - **Treat the store as reviewable state.** An anchor appearing in your store should be as
>   visible as a dependency being added.

## 4.7 The Android attestation case, concretely

Since this is the one most readers arrive for, here is the whole operational footprint:

| Thing | How many | Where it lives | Refresh |
|---|---|---|---|
| Attestation **roots** | **2** | baked in, pinned by digest | effectively never; review when Google announces one (as in Feb 2026) |
| Attestation **intermediates** | 0 to mount | arrive in the chain | n/a |
| **Revocation list** | one 180 KB JSON | fetched | **daily** — `cache-control: public, max-age=86400`. Honour it; don't fetch per verification |
| **App identity** — `(package, signing digest)` pairs | one per app you accept | **configuration, not code** | whenever a client rotates their signing key ([`android/11`](../android/11-key-attestation-in-depth.md) §11.5) |

So: two certificates that outlive most software, one daily JSON fetch, and a list of app
digests that changes when your clients change. Nothing resembling "hundreds of CAs rotating
frequently" — that shape belongs to web TLS, where the OS already handles it for you.

The one that will actually generate operational work is the bottom row, and it isn't a CA at
all.

## 4.8 Constraining *which* issuer, without moving your anchor

§4.4 offered two choices — anchor at the root and inherit every issuing CA, or anchor at the
intermediates and approve each one. There is a third, and for most people it's the right one.

The question that gets you there is: *what if I want to check all 12 issuing CAs — not just
trust them implicitly, but actually verify which one signed this leaf?* That's a legitimate
and often sensible policy. It just doesn't require touching your trust store.

### Three readings of "check at the child level"

| Reading | Mechanism | Verdict |
|---|---|---|
| Read the leaf's `issuer` DN and AKI and compare them to your list of 12 | string/bytes comparison on the leaf | **No.** Those are hints, and they are attacker-supplied (file 01 §1.3). A forged leaf carries any `issuer` and any AKI it likes. You'd be checking a label |
| Put the 12 in the trust store as anchors | trust-store change | Works, but §4.4's cost: **nothing above them is checked any more**, so a root-level revocation of an intermediate becomes invisible |
| Anchor at the 3 roots, then **assert on the path the verifier built** | post-validation allow-list | ✅ **This one.** Full validation to the root *and* explicit approval of the issuing CA |

The third keeps both properties you want. The chain is validated end to end — signatures,
constraints, and crucially the intermediate's *own* revocation status — and then you
additionally require that the intermediate in the validated path is one you recognise.

```
  1. path-validate leaf → … → one of your 3 roots        (all of file 02 §2.2 applies)
  2. take the chain the verifier RETURNED
  3. assert: chain[1] is in my set of 12                 ← the allow-list
```

Note the ordering. The allow-list runs **after** validation and operates on validation's
*output*. It is not a replacement for any check; it's an extra one.

### What to pin on

Not all identifiers survive routine events. Here are the real values for the two cross-signed
intermediates from §2.3 — same key, same subject, two different issuers:

```
inter.pem         issuer = CN=Example Root CA
inter-cross.pem   issuer = CN=Example Legacy Root CA

  SPKI pin   nRQZtNxfQhp4pYhYmHpIC2YYJuTvRjAGZsJfudD23w4=   ← IDENTICAL
  SKI        3D:B5:37:E4:51:E1:5F:1D:13:6A:54:6C:B4:72:…    ← IDENTICAL
  cert hash  91fc92a4…  vs  1a08274c…                       ← DIFFERENT
```

| Pin on | Survives cert renewal with the same key? | Survives cross-signing? | Notes |
|---|---|---|---|
| **SPKI hash** — `sha256` of the DER `subjectPublicKeyInfo` | **yes** | **yes** | The identifier of the *key*. The right default |
| **SKI** | yes, if the CA derives it from the key | yes | Equivalent in practice, but it's a value the CA *chose*, not a computed digest |
| Subject DN | yes | yes | Too coarse alone — but fine as a secondary filter |
| **Full certificate hash** | **no** | **no** | Breaks on benign renewal, and on a cross-signed path. Avoid |

That table is the argument in one line: a CA renewing its intermediate, or adding a
cross-signed copy, is routine and must not page you. **Pin the key, not the certificate.**

```sh
openssl x509 -in inter.pem -pubkey -noout \
  | openssl pkey -pubin -outform der \
  | openssl dgst -sha256 -binary | openssl enc -base64
```

### The trap: assert on the built path, not the supplied bundle

File 02's lesson returns with teeth. The certificates the peer sent are *a hint*; the chain
your verifier built is the fact. An allow-list applied to "the second certificate in the
bundle the client sent" is checking attacker-controlled ordering.

| Library | The built chain |
|---|---|
| OpenSSL | `X509_STORE_CTX_get0_chain(ctx)` — and `X509_STORE_CTX_get_num_untrusted(ctx)` tells you how much of it came from the untrusted pool |
| Go | `Certificate.Verify` returns `([][]*Certificate, error)` — "one or more chains where the first element of the chain is `c` and the last element is from `opts.Roots`" (Go 1.26.5) |

The two differ in a way that matters here. Go returns **all** the chains it found; OpenSSL
builds one — "if more than one chain is possible, only one is taken" (§2.4). So:

- **In Go**, decide the quantifier deliberately: the usual correct policy is *at least one
  returned chain satisfies the allow-list*, and you then proceed on **that** chain, not on
  whichever happened to be first.
- **In OpenSSL**, §2.4's hazard reappears in a new costume: the single chain it built may fail
  your allow-list *while a conforming path existed in the same inputs* and simply wasn't
  explored. The failure looks like "unapproved issuer" and is really "unlucky path choice".
  Cross-signed intermediates are exactly where this bites — which is another reason to pin on
  SPKI, since both cross-signed variants then satisfy the list identically.

### When an allow-list is the wrong tool

It assumes intermediates are **few and stable**. Where they aren't, it is actively harmful.

Android attestation under RKP is the clean counter-example: attestation keys are per-app,
rotated, and their certificates are "valid for up to two months before they expire and are
rotated" ([`android/11`](../android/11-key-attestation-in-depth.md) §11.2). An allow-list of
those intermediates would need updating continuously and would fail constantly — you would be
re-implementing the problem RKP was designed to remove.

What you constrain there instead is the *kind* of path and the *contents* of the leaf:
`ProvisioningMethod` (`FACTORY_PROVISIONED` vs `REMOTELY_PROVISIONED`), the security level, the
boot state, and the app identity — none of which are per-certificate identities.

> **Teacher's aside.** The general rule is worth more than either example: **allow-list
> identities that are stable, and constrain *properties* when they aren't.** An issuing CA that
> lives five years is an identity — name it. A per-app attestation key that lives two months is
> not something you can name in advance, so you assert on what it *claims* instead. The mistake
> isn't choosing the wrong one, it's applying an identity allow-list to a population that
> rotates, then concluding the scheme is broken when your list goes stale weekly.

### Churn rate decides the delivery mechanism

"Allow-list identities that are stable" is the right rule and it hides a second question:
**stable compared to what?** Stability isn't a property of the mechanism, it's a rate — and the
rate decides where the list has to live, not whether you should have one.

The two cases in this course sit at opposite ends, and the contrast is the lesson:

| | **eID issuing CAs** (§4.2) | **App identities** ([`android/11`](../android/11-key-attestation-in-depth.md) §11.5) |
|---|---|---|
| What's in the list | 6 CA keys | `(package name, signing digest, min version)` per app |
| New entries | one every 2–3 years | whenever a client onboards |
| Entries changing | effectively never — a CA key is fixed for its 13-year life | version every release; digest on key rotation |
| Reasonable home | **a config file in the image** | **a datastore with an admin path** |
| Cost of a stale list | a card generation stops verifying, eventually | a client's entire user base fails, immediately |

So the honest answer to "should the allow-list be config or infrastructure?" is: *it depends on
the churn rate*, and you should measure it rather than guess. Six entries changing every couple
of years is a file you edit and redeploy. A roster of third-party apps is not.

**But decompose before you conclude it's high-churn**, because "app identity changes
frequently" bundles three dimensions with completely different rates — and the fastest one is
free:

| Dimension | Rate | How to handle it |
|---|---|---|
| **Package version** | every release — days or weeks | **Don't list versions. Use the floor.** Google's matcher is `candidate.version >= expected.version` (`Extension.kt:724–726`), so you set a *minimum* once and routine releases sail past it without touching your config |
| **Signing certificate digest** | rare — key rotation is uncommon and deliberate | List **both** digests across an overlap window. The matcher accepts *any* expected digest present in the candidate's set, so old and new coexist cleanly. Needs the client to tell you *before* they rotate |
| **The roster of apps** | the real churn — a business event, not a technical one | A datastore you can change without a deploy, plus an onboarding process that collects the right digest ([`android/11`](../android/11-key-attestation-in-depth.md) §11.5: the **app signing** certificate, not the upload certificate) |

Notice what the first row does. The highest-churn dimension is absorbed by a **property check**
rather than an identity list — which is the §4.8 aside's "constrain properties when they
rotate", applied *within* a single allow-list rather than instead of one. A design that lists
exact versions has invented a weekly maintenance task that the protocol was built to avoid.

The version floor also earns its keep as a control rather than just a convenience: raising it
is how you refuse a known-bad old build. Keep it **lagging** your current release — move it
only when you actually need to retire a version, because every bump locks out users who haven't
updated yet.

> **Teacher's aside.** Generalising past both examples: when an allow-list feels unmaintainable,
> the useful move is usually not to abandon it but to **find the dimension doing the churning
> and replace that dimension with a predicate.** Versions become a floor. Rotating per-app
> attestation keys become `ProvisioningMethod` plus a boot-state check. What's left is a much
> smaller set of genuinely stable identities, which is exactly what an allow-list is good at.
> "This list changes too often" is nearly always a statement about one column of it.

### What the third option actually buys

| | Anchor at root only | **Root + allow-list** | Anchor at the 12 |
|---|---|---|---|
| Intermediate's revocation checked | ✅ | ✅ | ❌ |
| New issuing CA works without you | ✅ | ❌ (by design) | ❌ |
| You explicitly approve each issuer | ❌ | ✅ | ✅ |
| Distinguishable failure | ❌ both read "unknown authority" | ✅ "validated, but unapproved issuing CA" | ❌ reads as unknown authority |
| Operational burden | none | tracking 12 identities | tracking 12 identities |

The last two rows are the point. Against pinning the intermediates, the allow-list costs the
**same** operational effort and keeps strictly more checking — so if you want issuer control,
this is the cheaper way to get it. And the error quality is not a small thing: "chain validated
to our root, but the issuing CA is not on our approved list" is a message that tells an
operator exactly what happened and what to do, where `unable to get local issuer certificate`
tells them nothing (§2.5).

## Check yourself

1. An issuer has 3 roots and 12 live issuing CAs. How many certificates belong in your trust
   store, where do the others come from, and what changes about that answer if you decide to
   pin issuing CAs instead?
2. Two of a scheme's CA certificates have byte-identical subject DNs and different keys
   (§4.2). Explain what breaks in a verifier that finds its anchor by subject name, why the
   symptom misleads, and what OpenSSL's `<subject_hash>.0` / `.1` file naming has to do with it.
3. Three programs on one machine report 128, 158 and 193 web PKI roots. Explain why that's
   expected rather than a misconfiguration, and name the failure mode that makes the number
   `0` especially hard to debug.
4. Your team proposes fetching trust anchors from an HTTPS endpoint at startup so rotation
   doesn't need a redeploy. State the new trust dependency you've created, and say why the same
   proposal is more defensible for an ICAO master list than for a plain JSON endpoint.
5. A scheme rotates its issuing CA every 18 months and you want to approve each one explicitly.
   Describe what you've signed up for operationally, and the specific outage that occurs if you
   miss one.
6. An anchor in your store expires tonight. Predict what your OpenSSL-based service does, what
   an Android client does, and explain why "the certificate expired" is an incomplete
   diagnosis.
7. For Android key attestation, which single artefact actually needs regular refreshing, how
   often does its own HTTP response say to refresh it, and why is that a different kind of
   object from a trust anchor?
8. A national eID scheme's six anchors change once every two or three years; an SDK vendor's
   list of client app identities changes whenever a client onboards. Both are identity
   allow-lists. Say what differs about where each one should live, then name the one dimension
   of the app list that should not be a list at all.
9. You want to verify *which* of an issuer's 12 issuing CAs signed each leaf. Compare doing
   that by reading the leaf's `issuer` and AKI, by mounting the 12 as anchors, and by
   allow-listing against the built path — on what each one actually proves and what checking
   each one gives up.
10. You allow-list intermediates by full certificate hash. The CA renews one intermediate with
   the same key, and separately a cross-signed copy of another starts appearing in chains.
   Predict both outcomes, and say which pin would have survived both and why.

(Answers in [`06-exercises.md`](06-exercises.md).)
