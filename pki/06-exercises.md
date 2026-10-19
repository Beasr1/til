# 6. Exercises & answers

Worked answers to every **Check yourself**, then the recipe for the test chain, things to try,
and questions to bring back. Read an answer only after you've committed to one.

---

## File 1 — Foundations

**1. `issuer` reads `CN=Key Attestation CA1, O=Google LLC`.**
It tells you **which key to go and try**, and nothing else. The field is a label chosen by
whoever built the certificate, with no registry behind it and no uniqueness enforcement; I can
generate a certificate carrying that exact DN in one `openssl req` invocation. Treated as
evidence it is worth zero.

It becomes worth something only in combination: you look up the certificate whose `subject`
equals that DN (and whose SKI matches the AKI, if present) in a trust store **you** populated,
then check that this certificate's signature verifies under that certificate's public key. At
that point the name has done its only job — narrowing the search — and the signature carries
the whole claim. If the lookup finds the right key and the signature fails, the name was a lie.

**2. `serialNumber` is `1`.**
Both conclusions assume the serial is globally meaningful, and it isn't: RFC 5280 §4.1.2.2
makes it unique **per issuer**. So "the first certificate ever issued" is unjustified because
every issuer has its own serial space and millions of CAs have issued a serial `1`; "fake" is
unjustified because `1` is a perfectly legal serial — Android attestation leaves it as the
constant `1` on *every* attestation certificate by design (file 03 §3.3).

What it's for: naming a certificate in a revocation lookup. Which is exactly why its
per-issuer scope is a trap — a revocation answer matched on serial alone can be an answer about
a different issuer's certificate with the same serial. That attack is constructed in
[`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) §What to check before
believing it.

**3. The validator skips `basicConstraints`.**
The attacker starts from something entirely legitimate: a valid certificate issued to them by
a CA in your trust store — their own domain, their own card, anything. They hold its private
key, which is the whole point.

They then use that key to sign a *new* certificate, putting whatever subject they like in it:
`example.com`, another person's name, anything. Now they present the chain
`forged-leaf → their-legitimate-cert → intermediate → root`.

Against the checks described, it passes. Every signature verifies — their key really did sign
the forged leaf, and the CA really did sign their certificate. Nothing is expired. The only
thing wrong with the chain is that their certificate was never permitted to act as a CA, and
that permission is exactly what `basicConstraints` records. So skipping that check promotes
**every holder of any certificate from your CAs into a CA**. File 02 §2.2 runs this for real
and shows it being refused three different ways.

**4. A self-signed certificate asserting `cA: TRUE` and a grand-sounding name.**
On its own it proves only that whoever generated the file held the matching private key —
which is trivially true of the person who generated it. Every field, including the name and
the `cA` flag, is a self-assertion. I can produce an indistinguishable file claiming to be any
national root CA in the world.

The identical bytes become load-bearing the moment you **put them in your trust store**,
because from then on the file isn't a claim being evaluated — it's the premise everything else
is evaluated *against*. Nothing about the file changed; its role did. And since its own
signature is never checked (§1.6), the only protection against someone substituting a
different file is the **fingerprint you compare at load time**. Trust is configuration; the
pin is what secures the configuration.

**5. Repairing an encoding: anchor yes, leaf no.**
Defensible on the **CA certificate you use as an anchor**; destructive on the end-entity
certificates.

The deciding sentence: *you may only edit the bytes of a certificate whose signature you never
check.* An anchor's signature is never checked (§1.6), so rewriting a byte changes nothing you
rely on — the name and public key, the parts you actually use, are untouched. Pin the
**original** file's fingerprint, not the repaired one, so you're still detecting substitution.

For a leaf the signature is over the exact DER of its `tbsCertificate`. Change one byte and
either verification fails or — much worse — you have built a lenient parser for
attacker-supplied input, which is the last place you want one. If leaves are non-conformant,
the fault is the issuer's and the fix is theirs.

**6. "Is Acme Ltd a certificate authority?"**
Rewrite as: **"Does the certificate at this path assert `basicConstraints: cA = TRUE` with
`keyCertSign` in its `keyUsage`, and does it chain to an anchor I hold?"**

```sh
openssl x509 -text -noout -in acme.pem | grep -A1 -E 'Basic Constraints|Key Usage'
```

Three bytes decide the first half: `30 03 01 01 FF` in the `basicConstraints` extension. The
original question has no mechanical answer because "Acme Ltd" is an organisation and
organisations don't appear in verification — keys do (§1.3).

---

## File 2 — Building and walking

**1. `error 20 at 1 depth lookup: unable to get local issuer certificate`.**
**Building** failed, not validation. At depth 1 the verifier had a certificate in hand and
could not find *its* issuer in either the trust store or the supplied bundle. It never got as
far as judging anything.

What to go and get: the issuer named in that depth-1 certificate's `issuer` field.
`openssl x509 -noout -issuer -in <that cert>` tells you whose certificate you need; the
certificate's AIA extension often tells you where to fetch it.

Why "the certificate is invalid" is wrong: nothing was found to be wrong *with* a certificate.
The verifier reported an incomplete input. The certificate in front of you may be flawless and
the gap may be entirely on your side — and maddeningly, the same error appears when the
certificate is from a scheme you have no business trusting at all, so the message cannot tell
you which situation you're in.

**2. Validates on your laptop, fails in a container.**
*Building:* the container has a different, or empty, trust store, so the root your laptop
supplied from `/etc/ssl/certs` isn't there. A minimal base image has **zero** roots, and "I
have no roots" produces the same error as "I don't trust this root" —
[`http/05`](../http/05-certificates-and-trust.md) is written around exactly this failure.

*Validation:* the container's clock is wrong, so a certificate is outside its `validity`
window — error 10 or its not-yet-valid twin. Or the container runs a different, stricter TLS
library that refuses an encoding your laptop's OpenSSL tolerates
([`smartcards/07`](../smartcards/07-verifying-the-card-certificate.md) §Why one library
refuses what another accepts).

The errors distinguish them, which is the point of §2.5.

**3. Three errors at once on a three-certificate path.**
The submitter took an ordinary end-entity certificate they legitimately hold and used its
private key to sign a new certificate — file 01 question 3's attack, actually attempted.

Each refusal is independent, which is why three fire:

- **79, invalid CA certificate** — the certificate being used as an issuer has `cA: FALSE`;
- **25, path length constraint exceeded** — the issuing CA above it said `pathlen:0`, so no
  further CA may appear below it, and the attacker's certificate is now trying to be one;
- **32, key usage does not include certificate signing** — the leaf's `keyUsage` is
  `digitalSignature` only.

Which came "first" depends on traversal order rather than importance; the useful observation is
that all three are *constraint* checks. Every signature in the chain is cryptographically
valid. This is the clearest demonstration available that checking signatures is not verifying a
chain.

**4. Two intermediates, same subject and same SKI, different issuers.**
**Legitimate and useful** because it's how a PKI reaches clients with different trust stores. One
CA key gets two certificates: one issued by the current root, one by an older root that
legacy clients still carry. New clients build to the new root, old clients build to the old
one, and nobody has to update anything. Let's Encrypt did precisely this to keep working on
old Android devices.

**Why behaviour becomes unspecified:** the leaf's AKI points at a *key*, and both candidates
have that key, so nothing in the leaf distinguishes them. Choosing between them is **path
building**, which RFC 4158 is informational about and RFC 5280 says nothing about — because,
as RFC 4158 puts it, validation "is unaffected by the method in which the certification path
was built". True of a path once chosen, and no help at all in choosing. §2.4's measurement
shows the cost: four runs, identical inputs but for bundle order, two failures, and a valid
path present every time.

OpenSSL documents exactly why it can't recover. Its retry, when the first attempt fails to
reach an anchor, is "with progressively shorter untrusted prefixes" — it backtracks by
*dropping certificates off the front*, not by trying a different candidate at the same
position. The sibling that would have worked is never reconsidered. And
`X509_verify_cert(3ssl)` states the general case outright: **"If more than one chain is
possible, only one is taken."**

**5. Re-anchoring from a root to its current issuing CA.**
You have stopped checking **everything above the intermediate** — in particular, whether the
root has revoked that intermediate. An anchor is never validated and never revocation-checked;
it's trusted because it's configured (§1.6). By promoting the intermediate you removed the only
runtime mechanism that could have told you it was withdrawn.

The event that now goes unnoticed: the scheme revokes that issuing CA — key compromise, say —
and your verifier keeps accepting every certificate under it, indefinitely. Retiring it becomes
a configuration change you have to make by hand, having first heard about it by some
non-technical route.

Worth saying plainly: pinning an intermediate is often still the **better** policy, because it
refuses future CAs you never agreed to. It's a trade, not a mistake. Just make it knowingly.

**6. Caching built chains by leaf serial number.**
Serials are unique per issuer, not globally (file 01 §1.2). So the cache key collides whenever
two issuers both emit a certificate with the same serial — and low serials (`1`, `2`, …) are
extremely common, with Android attestation using `1` for every leaf it ever issues.

What breaks: a lookup for issuer B's serial `1` hits the cached entry for issuer A's serial
`1`, and the verifier concludes it has already validated a chain it has never seen. If the
cache stores a *verdict*, that's a straightforward authentication bypass — present any
certificate whose serial collides with something already validated.

When: as soon as a second issuing CA exists. With exactly one issuer the bug is latent and
invisible, which is the worst property it could have. Key the cache on something actually
unique — the hash of the certificate's DER, or the (issuer DN, serial) pair.

---

## File 3 — Four chains

**1. An attacker with the issuer's private key.**

| Chain | What they can do |
|---|---|
| **Web TLS** | issue a certificate for **any** domain, accepted by every client trusting that CA. The worst case in the whole table, because the namespace isn't restricted |
| **eID card** | mint certificates for invented people in that national namespace. Every OCSP response signed under that CA also becomes worthless, honest ones included — so revocation can't save you; removing the anchor can |
| **Android attestation** | produce chains that validate to Google's root for keys that never touched secure hardware — the leaked-keybox scenario. Mitigated by the revocation list, and structurally by RKP's short-lived per-app keys |
| **APK signing** | **the question is malformed.** There is no issuer. The nearest equivalent is stealing the *developer's own* signing key, which lets them ship updates the system accepts as genuine — contained only by rotation via signing lineage |

The pattern: the first three are "the vouching key is compromised", and the fourth has no
vouching key because it has no chain.

**2. "The list of issuer CAs for our Android app's signing certificate."**
**What they'll find:** nothing, because it doesn't exist. APK signing certificates are
self-signed — `issuer` equals `subject` and names the developer's own key. Public code-signing
CAs aren't an alternative either: their certificate lifetimes are capped far below the
post-2033 validity Google Play requires.

**The control they actually want:** a **pinned SHA-256 digest** of the signing certificate,
compared directly. No hierarchy, no path building. For an app distributed through Play App
Signing, the digest must be of the **app signing** certificate Google re-signs with, not the
upload certificate.

**The other chain they may be thinking of:** the **key attestation** chain, which is real,
does terminate at CAs, and does need a trust anchor you hold — Google's attestation root. It is
a completely separate chain that says nothing about who wrote the app, except through the
signing digests carried in its `attestationApplicationId` field. Conflating the two is the
subject of [`android/11`](../android/11-key-attestation-in-depth.md).

**3. Fixed subject, serial `1`.**
Correct because there is nothing to name. The three other chains bind a key to an identity — a
domain, a person — so their subject names carry the payload. Attestation binds a key to a
**provenance claim**: this key was generated inside certified secure hardware, on a device in
this boot state. No person, no host, nothing to put in a DN. Filling it with something
invented would be worse, because verifiers would start matching on it.

Where the information went: the extension at OID `1.3.6.1.4.1.11129.2.1.17`, as a
`KeyDescription` — the key's properties, the challenge, and the verified-boot state. A
TLS-shaped verifier reads the subject, finds a constant, and concludes the certificate is
broken; it is reading the wrong field in a chain whose designers deliberately emptied it.

**4. Chain check and challenge, one without the other.**

| You run | eID card | Android attested key |
|---|---|---|
| **chain only** | any **copy** of a genuine card's certificate — the certificate is a readable file, so a clone presenting it passes. You have proven the issuer, not the card | an attestation that was genuinely produced *somewhere*, possibly long ago or by a different session. A captured chain replays |
| **challenge only** | a signature from *some* key. The holder generated a keypair, signed your nonce, and told you a name. You've proven possession of a key nobody vouched for | a key generated in **software** on an attacker's machine, which signs your nonce perfectly. Nothing links it to secure hardware |

Which is the pairing from file 03 §3.3's aside: the chain establishes provenance against an
out-of-band anchor, the challenge establishes possession *now*. Each alone admits an entire
class of attack the other closes.

**5. Hundreds of anchors versus a pinned handful.**
**What TLS's design buys:** universality without prior arrangement. You can verify a
certificate for a host you've never heard of, run by an organisation you have no relationship
with, with no configuration — which is the property that made the web work. Pinning gives that
up entirely: an eID verifier can only verify schemes someone deliberately added.

**What pinning buys:** elimination of the "any CA may vouch for any name" weakness. With one
pinned issuer there is no weakest-member problem, no need for Certificate Transparency to make
misissuance findable, and a misbehaving CA elsewhere in the world cannot affect you. TLS gives
that up: its integrity is that of its weakest trusted member, and the mitigation is detection
after the fact rather than prevention.

**6. Expired attestation certificates still trusted.**
No contradiction, because check 2 of file 02 §2.2 applies to certificates **in the path**, and
RFC 5280 §6.1 excludes the trust anchor from the path (file 01 §1.6). Google's guidance is
about chains terminating at a specific root: keep trusting the anchor regardless of its
validity period. An expiry date is a statement by an issuer about a certificate it signed — and
an anchor has no issuer above it to make that statement meaningfully, which is why Android
likewise ignores `notAfter` on TLS trust anchors.

The limit is worth stating, because the exception is narrower than it sounds: it covers the
*anchor*. For RKP chains the intermediate validity periods are deliberately short — a couple of
months — and Google is explicit that they must still be checked, because fast expiry is how
compromises get contained. So "ignore expiry in attestation chains" would be exactly the wrong
generalisation to draw.

One further wrinkle, which is why Google had to say this out loud at all: **not every verifier
agrees that anchors escape expiry.** OpenSSL checks validity on every certificate "including
the terminator" and really does refuse an expired root (file 01 §1.6). Android ignores
`notAfter` on anchors. So the same expired attestation root is trusted by one stack and
rejected by another, and Google's guidance is in effect instructing you to implement Android's
policy on your server — which your server's default library may not do.

---

## File 4 — Running a trust store

**1. Three roots, twelve issuing CAs.**
**Three** certificates belong in your store. The twelve arrive **in the chain** — the party
presenting a certificate supplies the intermediates needed to link it upward — or are fetched
from the leaf's AIA extension. You never mount them.

If you pin the issuing CAs instead, the answer flips to **twelve**, and so does the
operational posture: you now approve each issuing CA explicitly, nothing new works until you
add it, and the scheme's rotation schedule becomes your release calendar. You also stop
checking anything above those twelve, so a root-level revocation of one of them becomes
invisible to you (file 02 §2.6). Tighter policy, real cost — the point is to pick it rather
than drift into it.

And if the reason you wanted the twelve was to *approve each issuer* rather than to relocate
trust, you don't have to choose between those two at all — keep the 3 anchors and allow-list
the 12 against the built path (§4.8). Same effort, strictly more checking.

**2. Two CA certificates, identical subject DN, different keys.**
A verifier that looks up "the CA whose `subject` equals this leaf's `issuer`" finds **two
candidates** and has no basis in the name to choose. Depending on how its store is keyed it
either picks one arbitrarily, or silently holds only one of the two because the second
overwrote the first on load.

**Why the symptom misleads:** when it picks the wrong one, the failure is a **signature
verification failure**, not a lookup failure. So the error says something like "certificate
signature failure" or just refuses the chain — which reads as *"this certificate is forged or
corrupt"*, and sends you to investigate the card. The certificate is fine. You verified it
under the wrong key, because you selected the key by a name that two keys share.

**What fixes it:** `authorityKeyIdentifier` → `subjectKeyIdentifier` matching. The leaf's AKI
names the *key*, which is the thing that actually differs. This is RFC 5280 §4.2.1.1's stated
purpose — "where an issuer has multiple signing keys (either due to multiple concurrent key
pairs or due to changeover)" — and §4.2 is that sentence happening in production.

**The OpenSSL file naming:** a hashed CA directory stores each certificate as
`<subject_hash>.<n>`. The numeric suffix exists *because* subject-hash collisions are expected —
`3e24970c.0` and `3e24970c.1` are two different CAs with one subject name, and OpenSSL tries
each in turn. Any home-grown store keyed on subject DN (a map, a filename without a suffix, a
database unique index) has quietly reintroduced the bug the suffix was invented to avoid.

**3. 128, 158 and 193 on one machine.**
Expected, because a trust store is **not a protocol artefact** — it's a file an application
goes looking for, and different applications ship, derive or point at different bundles. The
OpenSSL bundle, Homebrew's `ca-certificates` package and the macOS keychain are three separately
maintained inventories, each with its own update cycle and its own inclusion decisions. There
is no canonical count to converge on.

The failure mode that makes `0` hard to debug: an empty store produces the **same error** as an
untrusted root — "unable to get local issuer certificate" / "self signed certificate in
certificate chain" — so nothing in the message says "you have no anchors at all". It surfaces
only on the first outbound TLS call, which in a minimal container image can be months after
the image was built, and looks convincingly like a problem with whatever host it finally tried
to reach ([`http/05`](../http/05-certificates-and-trust.md)).

**4. Fetching anchors over HTTPS at startup.**
**The new dependency:** your trust anchors now depend on the **web PKI** and on whoever controls
that endpoint and its DNS. You have made the thing everything bottoms out on into a runtime
input — the trust-anchor problem recursing one level up (file 01 §1.6). Compromise the feed and
you decide what the verifier trusts, with no signature to stop you.

**Why a master list is more defensible:** an ICAO master list, like the eIDAS LOTL, is a
**signed bundle**. You verify its signature against a signer you pinned, so re-fetching is a
cryptographically checkable operation and a hostile substitution fails closed. A plain JSON
endpoint has no such property — its only integrity is the TLS connection, which is a different
and much broader trust base.

The sharpest version of the argument: for a feed of **two certificates valid until 2035 and
2042**, you are taking on a supply-chain dependency to automate a change that happens roughly
once a decade. Pin them.

**5. An issuing CA rotating every 18 months, approved explicitly.**
You have signed up for a **recurring, externally scheduled operational task with a hard
deadline you don't control** — tracking the scheme's announcements, obtaining each new issuing
CA out of band, verifying it, and getting it into production before the scheme starts using it.
Also for keeping old anchors until the last certificate under them expires, so the store grows
before it shrinks.

The outage if you miss one: certificates issued under the **new** CA fail to verify, while
everything under the old one keeps working. So it's a *partial* failure that grows as the
scheme migrates — a trickle of rejections that becomes a flood — and it reports as
`unable to get local issuer certificate`, which is indistinguishable from a foreign or
fraudulent certificate (file 02 §2.5). You will debug it as an attack before you debug it as a
missing anchor.

**6. An anchor expiring tonight.**
**OpenSSL-based service:** rejects. `X509_verify_cert(3ssl)` checks validity periods "on every
certificate, including the terminator", so you get `error 10 ... certificate has expired` at
the anchor's depth, and every chain under that root fails.

**Android client:** keeps working. Android "has intentionally chosen not to use the `notAfter`
field of trust anchors".

**Why "the certificate expired" is incomplete:** it names a *fact* and not a *policy*, and the
policy is what determines the outcome. The same expired anchor is simultaneously fine and fatal
on two stacks in the same system. The useful diagnosis names both — which certificate, and
which verifier's anchor-expiry policy — because the remediation differs: replace the anchor, or
in some schemes (attestation) deliberately configure your verifier to ignore anchor expiry the
way the scheme's own clients do.

**7. The one artefact needing regular refresh in Android attestation.**
The **certificate revocation status list** at `https://android.googleapis.com/attestation/status`.
Its own HTTP response says how often: `cache-control: public, max-age=86400` — **daily**. Honour
that rather than fetching per verification.

Why it's a different kind of object from an anchor: an anchor is a **statement of policy you
authored** — it's in your store because you put it there, it needs no freshness, and it changes
when you decide it does. A revocation list is **evidence about the world**, generated by someone
else, whose value is entirely in being current; yesterday's copy is strictly worse than today's.
Anchors are configuration; revocation data is a feed. Confusing the two is how people end up
polling for roots that change once a decade while caching revocation data for a week.


**8. Six eID anchors versus a churning roster of app identities.**

Both are identity allow-lists; only the **rate** differs, and the rate is what decides the
delivery mechanism.

| | eID anchors | App identities |
|---|---|---|
| New entries | one every 2–3 years | whenever a client onboards |
| Where it belongs | **a config file in the image.** Six entries, changing on a multi-year cadence — a redeploy is entirely proportionate, and keeping it in the artefact makes it reviewable in a diff | **a datastore with an admin path.** Needing a deploy to onboard a client makes your release train the bottleneck for someone else's launch |
| Stale-list blast radius | a card generation gradually stops verifying | a client's entire user base fails at once |

The second column's urgency is the real argument. An eID anchor you forgot to add fails slowly
and visibly as a new card generation rolls out; a missing app identity fails every request from
that app immediately, so you need to fix it in minutes, which rules out anything requiring a
build.

**The dimension that should not be a list: the app version.** Google's matcher compares
`candidate.version >= expected.version` (`Extension.kt:724–726`), so you configure a **floor**
once and every routine release passes it untouched. Listing exact versions converts the
fastest-moving dimension in the system into per-release maintenance, for no security gain — the
floor already gives you the only thing an exact list would: the ability to refuse builds older
than some line you draw.

Keep the floor **lagging** your current release, and raise it only to retire a version, since
every bump locks out users who haven't updated.

The generalisation worth keeping: when an allow-list feels unmaintainable, find the column
doing the churning and replace **that column** with a predicate. What remains is usually a
small, genuinely stable set — which is what allow-lists are good at.

**9. Three ways to check which of 12 issuing CAs signed a leaf.**

| Approach | What it proves | What it gives up |
|---|---|---|
| Read the leaf's `issuer` DN and AKI | **Nothing.** Both are attacker-supplied fields in the certificate being judged. A forged leaf carries whichever issuer name and AKI its author typed (file 01 §1.3) | Everything — it isn't a check, it's a label comparison |
| Mount the 12 as anchors | That the leaf chains to one of twelve keys you configured. Genuine and strong | Everything *above* the twelve. The anchor is never validated, so if the root revokes an intermediate you keep accepting its certificates indefinitely (file 02 §2.6). You also must add each new issuing CA before it works |
| Anchor at the 3 roots, allow-list against the **built path** | That the leaf chains to a trusted root through an issuing CA you approved — both properties at once | Nothing in the checking. You still must track the 12, so the operational cost equals the previous row |

The third is the one to reach for. The first is a category error; the second trades away
upward checking for no gain over the third.

Two details decide whether the third is implemented correctly. It must assert on the chain the
verifier **returned** (`X509_STORE_CTX_get0_chain`, or Go's `Verify` return value), never on
the bundle the peer supplied — the supplied order is attacker-controlled (file 02 §2.4). And
in Go, which returns *multiple* chains, fix the quantifier deliberately: accept when **at least
one** returned chain satisfies the list, and then proceed on that chain.

**10. Pinning intermediates by full certificate hash.**

**The renewal:** the CA reissues the intermediate with the **same key** and a new validity
period. The certificate's DER changes, so its SHA-256 changes, so your pin no longer matches.
Every certificate issued under that intermediate starts failing — a total outage for that
issuing CA, caused by an entirely routine and legitimate action that nobody thought to tell
you about.

**The cross-sign:** a second root signs the same intermediate key, producing a certificate with
the same subject and the same SPKI but a different issuer and a different DER. Now *which*
certificate your pin matches depends on which path your verifier happened to build — and
OpenSSL builds only one (§2.4). The result is intermittent and input-order-dependent failure,
which is the worst possible presentation.

**What survives both:** the **SPKI hash** — `sha256` over the DER `subjectPublicKeyInfo`. It
identifies the *key*, and in both scenarios the key is unchanged. §4.8's measurement shows it
directly: the two cross-signed intermediates have identical SPKI pins and identical SKIs, and
different certificate hashes. (SKI would also survive, but it's a value the CA chose rather
than a digest you computed, so prefer the one you can verify yourself.)

The general principle: pin the thing that is *stable under the operations the issuer is
expected to perform*. Certificates get reissued; keys usually don't.

---

## The test chain used in file 02

Everything in file 02 §2.2 and §2.4 came from this. OpenSSL 3.6.4; it takes under a minute.

```sh
# root
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -noenc \
  -keyout root.key -out root.pem -days 3650 \
  -subj "/C=GB/O=Example Scheme/CN=Example Root CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

# issuing CA, pathlen:0 so nothing may be a CA below it
openssl req -newkey ec -pkeyopt ec_paramgen_curve:P-256 -noenc \
  -keyout inter.key -out inter.csr -subj "/C=GB/O=Example Scheme/CN=Example Issuing CA 1"
openssl x509 -req -in inter.csr -CA root.pem -CAkey root.key -out inter.pem -days 1825 -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign\nsubjectKeyIdentifier=hash\nauthorityKeyIdentifier=keyid\n")

# leaf
openssl req -newkey ec -pkeyopt ec_paramgen_curve:P-256 -noenc \
  -keyout leaf.key -out leaf.csr -subj "/C=GB/O=Example Scheme/CN=card-00412"
openssl x509 -req -in leaf.csr -CA inter.pem -CAkey inter.key -out leaf.pem -days 365 -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:FALSE\nkeyUsage=critical,digitalSignature\nsubjectKeyIdentifier=hash\nauthorityKeyIdentifier=keyid\n")

openssl verify -CAfile root.pem -untrusted inter.pem -show_chain leaf.pem
```

For §2.4's cross-sign, make a second root and issue the **same `inter.csr`** from it — same
subject, same public key, different issuer:

```sh
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 -noenc \
  -keyout root2.key -out root2.pem -days 3650 \
  -subj "/C=GB/O=Example Scheme/CN=Example Legacy Root CA" \
  -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign"
openssl x509 -req -in inter.csr -CA root2.pem -CAkey root2.key -out inter-cross.pem -days 1825 -sha256 \
  -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign\nsubjectKeyIdentifier=hash\nauthorityKeyIdentifier=keyid\n")

# then vary ONLY the order, and watch the verdict change
cat inter.pem inter-cross.pem > b.pem && openssl verify -CAfile root2.pem -untrusted b.pem leaf.pem
cat inter-cross.pem inter.pem > b.pem && openssl verify -CAfile root2.pem -untrusted b.pem leaf.pem
```

## Things to try

All of it on throwaway keys you generated; nothing here touches anyone else's infrastructure.

- **Confirm the two identical SKIs.** For `inter.pem` and `inter-cross.pem`, print
  `subjectKeyIdentifier` and the SHA-256 of the public key. Same key, same SKI, different AKI —
  then re-read file 02 §2.3 and the diagram will be about *your* files.
- **Reproduce the leaf-as-CA forgery.** Sign a certificate with `leaf.key` and verify it. Count
  the refusals. Then re-issue the leaf *without* `keyUsage` and see which refusals survive —
  that's `basicConstraints` carrying the check alone.
- **Break one byte.** `openssl asn1parse -in leaf.pem` to find the `cA` BOOLEAN, flip `FF`
  to `01`, and verify with OpenSSL and then with a strict library (Go's `crypto/x509` is two
  lines). Watch them disagree. Do it on the *root* as well and note that only one of the two
  edits is detectable.
- **Prove the anchor's self-signature isn't checked.** Verify a chain normally, then with
  `-check_ss_sig`. Then corrupt the root's signature bytes and repeat both.
- **Empty the trust store.** Run a verification with `-no-CAfile -no-CApath` and read the
  error. That is the container failure from [`http/05`](../http/05-certificates-and-trust.md),
  in one command.
- **Read a real attestation chain.** On your own device, generate a Keystore key with an
  attestation challenge, dump the chain, and `openssl x509 -text` the leaf. Find
  `CN=Android Keystore Key`, serial `1`, and the extension at `1.3.6.1.4.1.11129.2.1.17`
  ([`android/11`](../android/11-key-attestation-in-depth.md)).
- **Look at your own app's signing certificate.** `keytool -printcert -jarfile your.apk`.
  Confirm `issuer` equals `subject`, then try to find an issuer CA for it (file 03 §3.4).
- **Count your own trust stores.** `grep -c 'BEGIN CERTIFICATE'` over every CA bundle on your
  machine, and compare with what your OS keychain reports. Three different numbers is normal
  (file 04 §4.2).
- **Prove an expired anchor is a policy question.** Issue a root with `-not_before`/`-not_after`
  in the past, re-issue the intermediate under it, and verify the leaf. Watch OpenSSL refuse at
  the anchor's depth, then pass `-no_check_time` and watch it succeed. That gap is the
  OpenSSL-versus-Android disagreement, reproduced locally (files 01 §1.6, 04 §4.5).
- **Read the cadence off the wire.** `curl -s -D - -o /dev/null https://android.googleapis.com/attestation/status`
  and look at `cache-control`. Then fetch the body and count entries by `status` and `reason`.
  That tells you both how often to refresh and how real the keybox problem is (file 04 §4.7).
- **Size a real anchor set.** Pick any scheme you work with and write down, separately, how many
  *roots* it has and how many *issuing CAs*. Only the first column goes in your store (file 04
  §4.1).
- **Hunt for a subject-hash collision in a real CA set.** For every CA certificate of any
  scheme you have, run `openssl x509 -in c.pem -noout -subject_hash -subject -nameopt RFC2253`
  and sort by hash. If two share one, confirm their SKIs differ — you have found file 04 §4.2's
  case, and your verifier had better be matching on AKI.
- **Compute SPKI pins and watch them survive a cross-sign.** For `inter.pem` and
  `inter-cross.pem`, print the SPKI pin and the full certificate hash:
  ```sh
  openssl x509 -in inter.pem -pubkey -noout | openssl pkey -pubin -outform der \
    | openssl dgst -sha256 -binary | openssl enc -base64     # SPKI pin
  openssl x509 -in inter.pem -outform der | openssl dgst -sha256   # cert hash
  ```
  The SPKI pins match and the cert hashes don't. That is the whole argument of file 04 §4.8 in
  two commands.
- **Build an issuer allow-list and break it on purpose.** In Go, call `Verify`, then assert the
  returned chain's `chain[1]` SPKI pin is in a set. Point the set at the wrong intermediate and
  confirm the failure says "unapproved issuer" rather than "unknown authority" — the error
  quality that §4.8's last table is about.

## Questions worth asking me

- "Here's a chain and my trust store — walk me through exactly what my verifier will do."
- "This fails with error 20 in production and works locally. Which job is failing, and what do I go and get?"
- "Should I anchor on this scheme's root or its current issuing CA? Argue both sides for my threat model."
- "Is the thing I'm verifying a chain at all, or should I be pinning a digest?"
- "Our verifier logs 'certificate verified'. Enumerate what that does and does not license me to conclude."
- "Walk me through what a cross-signed intermediate does to my particular trust store."
- "Which of these checks is my library skipping by default, and does it matter here?"
- "Compare how two schemes I name distribute their trust anchors, and which is more robust to a compromised CA."
