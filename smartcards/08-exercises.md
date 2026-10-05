# 8. Exercises

Worked answers to every **Check yourself**, then things to try on a real card.

---

## Chapter 1 — What a smart card is

**1. Storing a signed token and reading it back — what fails?**

Replay. The token is readable, so an attacker reads it once and writes it onto their own
card; every verifier then accepts it, because the signature is over the *data* and the data
is genuine. Signing proves the issuer produced that data. It says nothing about which
physical object is presenting it.

A smart card instead keeps a private key or a symmetric key that is never emitted, and
proves possession of it against a value chosen fresh by the verifier. The copy has the data
but not the key, so it fails the moment it is challenged.

**2. Two causes of "class not supported".**

First: you used `CLA=00` for a command the card only implements under a proprietary class
such as `0x80`. The card is saying "wrong language", not "no such feature".

Second, and unrelated to the card: you sent a **reader** pseudo-APDU that this reader
doesn't implement. The card never saw it. Treating that as information about the card is a
category error — it is information about the reader.

**3. Why synthesise a contactless ATR?**

The ATR is a power-up artefact of the contact interface: the chip emits it when the reader
applies voltage. A contactless card is energised by an RF field and answers the ISO 14443
anti-collision and select sequence instead, producing an ATS. PC/SC builds an ATR-shaped
string from the ATS so that host software — all of which expects an ATR — has something to
consume.

**4. "No card present" with a card in the slot.**

A dual-interface reader exposes each interface as a separate PC/SC reader name. Software
that connects to "the first reader" connects to whichever the enumeration returns first,
which may be the empty contact slot of a device whose card is on the antenna. Connect by
matching the name, not by index.

**5. 400 ms for 2 kB — why not slow flash?**

Because it isn't a storage device. The transfer is over a serial line at a few hundred
kilobits per second, in chunks of at most 256 bytes, each one a full command-response
round trip — and on T=0 each may take two round trips (chapter 4). The latency is protocol
overhead and clock speed, not media.

---

## Chapter 2 — The ATR

**1. `T0 = 0x6A`.**

High nibble `6` = `0b0110`: reading the bitmap as TA, TB, TC, TD from the low bit up,
bits 2 and 3 are set — **TB1 and TC1 present, TA1 and TD1 absent**. Low nibble `A` =
**ten historical bytes**.

**2. No `TD1` — which protocol?**

T=0. `TD1` is the only thing that advertises a protocol, and when it is absent the default
defined in ISO/IEC 7816-3 applies. You know before sending a single command, which is the
point: you must already know how to talk before you can ask anything.

**3. Why synthesised, and what gives it away?**

Covered in chapter 1 answer 3 for the "why". The giveaway is the fixed prefix `3B 8x 80 01`
— `T0` with only `TD1` present, `TD1 = 0x80`, then `0x01`. A contact card's ATR carries its
own interface bytes there instead, so that exact opening is not something a contact card
produces.

**4. Identical ATRs, different issuers — a fault?**

No. The ATR describes the silicon and its configuration — clock parameters, protocol,
historical bytes chosen at manufacture. Two cards built on the same chip with the same
configuration have the same ATR regardless of what applications are loaded or what data
they hold. ATRs are commonly shared across an entire product line.

**5. Two ways "version 2" can be wrong.**

The table is wrong: whoever maintained it mapped this ATR to the wrong model, or the
mapping was right once and the issuer reused the ATR for a later model.

The table is right but incomplete: this ATR is genuinely shared by two card models that
differ in ways the ATR cannot express, so *no* lookup could distinguish them. The code
returns a confident answer because a lookup always returns something.

Both fail silently. Neither produces an error, which is why the failure mode is a wrong
answer rather than a missing one.

---

**6. Five ATRs for one card programme — what can differ?**

Three things, none of which is what the card can do:

| What differs | Visible as | Does it change behaviour? |
|---|---|---|
| Interface parameters | `TA1` present or absent, and its value | Speed of the link, not the command set |
| Issuer versioning inside the historical bytes | one byte flipping, e.g. `30` → `31` | Nothing the standard defines — it is proprietary |
| Protocol support | `TD1` present, naming T=1 | **Yes** — see chapter 4 |

The `TCK` tells you about the third and only the third. It is present exactly when a
protocol other than T=0 is indicated, so its presence is a signal that `TD1` said
something. It tells you nothing about speed parameters, issuer versions or applications —
and it is a checksum, so it does not even tell you the ATR is *correct*, only that it was
not corrupted in transmission.

The general point: the ATR describes a *configuration*, and a configuration can be
re-spun without a single command changing. Any table keyed on the exact ATR string
therefore grows rows over the life of the programme, silently, and code that treats an
unknown ATR as an unknown card will start rejecting perfectly good ones.

**7. Decode `3B8A80018065A20131013D72D641A5`.**

Byte by byte:

```
3B 8A 80 01 80 65 A2 01 31 01 3D 72 D6 41 A5
│  │  │  │  └──────────────────────────┘  └─ TCK = A5
│  │  │  │   ten historical bytes
│  │  │  └─ TD2 = 01 → high nibble 0: nothing further; low nibble 1: T=1
│  │  └──── TD1 = 80 → high nibble 8: TD2 present; low nibble 0: T=0
│  └─────── T0  = 8A → only TD1 present; ten historical bytes
└────────── TS  = 3B, direct convention
```

So the card offers **T=0 first, then T=1**. First-named wins by default, so a reader that
does no negotiation speaks T=0 — the card supports both, which is not the same as the
reader using both.

The checksum is the XOR of every byte from `T0` through the last historical byte:

```
8A ⊕ 80 ⊕ 01 ⊕ 80 ⊕ 65 ⊕ A2 ⊕ 01 ⊕ 31 ⊕ 01 ⊕ 3D ⊕ 72 ⊕ D6 ⊕ 41 = A5
```

Matches the trailing byte. Note `TS` is excluded from the computation.

Without the final byte the ATR would be **malformed**, not merely shorter. `TCK` is
mandatory whenever any protocol other than T=0 is indicated, and `TD2` indicated T=1. A
strict reader should reject it; a lax one will consume `41` as the checksum, silently lose
the tenth historical byte, and hand you a card that looks subtly different from the one in
the slot.

## Chapter 3 — APDUs and status words

**1. Read 16 bytes from offset 0.**

```
00 B0 00 00 10
```

`CLA=00`, `INS=B0` READ BINARY, `P1 P2 = 0000` the offset, `Le=0x10`. Case 2 — no command
data, response expected.

**2. `62 82` and discarding the response.**

The data. `62 82` is a **warning**: it means the file ended before `Le` bytes could be
returned, and everything returned before the status word is valid. Treating any non-`9000`
as failure throws away a good short read — and then the retry asks for the same bytes and
gets the same warning, so it looks like a file that cannot be read at all.

**3. `6E00` then `6982` for the same instruction.**

`6E00` under `CLA=00`: the card does not implement this instruction *in the interindustry
class*. On its own that is ambiguous between "not supported" and "wrong class".

`6982` under `CLA=80`: the instruction **exists** and is gated on a security condition —
typically a PIN, possibly a secure channel. That resolves the ambiguity: the capability is
present.

Next step is to find out which condition, and to do so without spending a retry — see
answer 5.

**4. Why `6D00` on READ BINARY suggests selection.**

`READ BINARY` operates on a currently selected file. On a card following the GlobalPlatform
model, file commands are implemented by applets rather than by the card manager, so before
any selection there is nothing to implement them. The instruction becomes available the
moment an applet is selected — so `6D00` is a statement about context, not capability.

**5. Probing whether a PIN is required, for free.**

Attempt the operation itself with no credential presented, and read the status word:

| Response | Tells you |
|---|---|
| `69 82` | It exists and needs authentication — a PIN or a secure channel |
| `6E 00` | Wrong class; retry under `CLA=80` before drawing conclusions |
| `6D 00` | Not implemented in the current context — check what's selected |
| `90 00` | No authentication needed at all |

Nothing here presents a credential, so nothing decrements. What you must **not** do is
"test" a PIN — including an empty or dummy one, since some implementations count that as a
failed attempt.

---

## Chapter 4 — T=0 and T=1

**1. Next APDU after `6C 4E`.**

```
80 CA 00 66 4E
```

The identical command with `Le` replaced by the length the card named. Not `GET RESPONSE` —
that is for `61 XX`.

**2. Why T=0 cannot return both.**

T=0 is byte-oriented: an exchange carries either a data field or a status, with no framing
to delimit both in one direction. T=1 wraps each direction in a block with a length, so a
response block can contain data followed by the status bytes. The constraint is in what the
transport can frame, not in the command set.

**3. Identical code, one card fails.**

`TD1`. Present (advertising T=1) on the working card; absent on the failing one, which is
therefore T=0. The failing card is answering `61 XX` or `6C XX`, the code is treating that
as an error, and a capability probe consequently reports "not found" for data that was
sitting there.

**4. `SELECT` succeeds, later `READ BINARY` returns `69 85`.**

The `SELECT` returned `61 XX` — success, with its FCI waiting. The code read the `61` as a
failure, or ignored it, and never sent `GET RESPONSE`. Depending on the card, the selection
may not have completed, so nothing is selected when `READ BINARY` arrives, and "conditions
of use not satisfied" is the honest answer to a read with no current file. The cause is two
commands upstream of the symptom.

**5. Why duplicated handling is worse than one missing implementation.**

Because each copy fails differently. A missing `61` handler in the debug path loses FCIs; a
missing one in the read path loses data; a missing one in a discovery tool reports an empty
card. Three unrelated-looking bugs, none of which suggests the others, and fixing one
doesn't fix the rest. A single chokepoint has exactly one bug, which gets found once.

---

## Chapter 5 — How cards are organised

**1. AID selects, then `DF 0100` returns `6A82`.**

Either the DF genuinely does not exist on this card, or it exists but is not reachable from
where the AID selection landed you — if the application *is* `DF 0200`, then `0100` is a
sibling and cannot be selected as a child.

Tag `83` of the application's own FCI distinguishes them: it reports the FID of the DF you
are now in. If it says `0200`, the second explanation holds and you need to select from a
common parent — or accept that the card has no navigable MF at all.

**2. Why `6D00` is about the model.**

An instruction that is genuinely absent is absent always. `READ BINARY` returning `6D00`
before a selection and `9000` after cannot be a capability statement — it is the
GlobalPlatform split, where file commands live in applets and the card manager implements
almost nothing.

**3. FCI returned but nothing parsed.**

Either the card used a proprietary template rather than the standard `6F` wrapper, and your
parser found no tags it knows; or you are on T=0 and never fetched the FCI at all, so you
parsed an empty buffer.

The raw bytes tell you immediately: an empty buffer means the second, and a non-empty
buffer starting with something other than `6F` means the first.

**4. Zero files across `0001–02FF` on a populated card.**

The selection mode is one the card rejects — `P1=02` where it wants `P1=00`, giving `6A86`
for every FID, classified as "absent".

You scanned from the MF on a card whose files live inside applets, so there genuinely is
nothing at the level you looked.

You are on T=0 without completing the exchanges, so every response looked empty or was
misread.

**5. `83 02 02 00` in an applet's FCI.**

You are inside `DF 0200`. Files whose FIDs are children of `0200` — `0201`, `0203` and so
on — are selectable from here. Anything under a sibling DF such as `0100` is not, and will
answer `6A82` no matter how many times you ask.

---

## Chapter 6 — Proving genuineness

**1. "Mutual authentication OK" — what's established?**

At minimum, that the card accepted the terminal's cryptogram. That half is fakeable by a
clone running its own firmware, which can return success without verifying anything.

The question to ask is: **did we verify the card's own cryptogram?** Only that step
requires computing a MAC over the fresh challenge with the diversified key, which a clone
cannot do. If the implementation checked it, the result is an anti-clone proof; if it only
observed a success status, it is not.

**2. Why diversification limits the blast radius.**

Each card holds `KDF(master, card_id)`. Recovering one card's key gives you that key. Going
from it to the master, or to another card's key, would require inverting the KDF — which is
what the KDF is chosen to make infeasible.

It would stop being true if the derivation were reversible, if every card shared one key,
or if the "unique" input were predictable *and* the KDF weak enough to attack with known
plaintext-key pairs.

**3. Data signed, verified with a certificate from the same card.**

Proves: the data and the signature and the certificate are mutually consistent — nothing
has been altered since whoever held that private key signed it.

Does not prove: that the signer was the issuer, or that the card is genuine. A cloner
generates a keypair, issues themselves a certificate, signs their own data with it, and
writes all three to a card. Every check passes. The verification never consulted anything
the cloner didn't supply.

**4. `INTERNAL AUTHENTICATE` → "security status not satisfied".**

Good news. It establishes that the card implements asymmetric card-to-terminal
authentication — the mechanism exists and is merely gated. Compare "instruction not
supported", which would mean there is no such capability and no amount of authentication
would produce one.

It also tells you what to obtain next: the credential the gate wants.

**5. Extracting the CA from an OCSP response.**

Circular. The response's embedded chain is supplied by the same party whose authority you
are trying to establish, over an unauthenticated transport, and you have nothing
independent to check it against. An attacker who can answer your OCSP request can supply
their own chain.

It is acceptable only as a *convenience* when you already hold the CA's fingerprint from a
trustworthy source and verify the extracted certificate against it — at which point you are
trusting the fingerprint, not the response, and the extraction is just a download.

**6. A generation with neither challenge-response nor on-card signing.**

Establishable: that the data is internally consistent and unaltered since signing, provided
the signing certificate chains to a trust anchor you hold. Also that the data matches
whatever authoritative record you can check it against out of band.

Not establishable: that the physical card is genuine. With no way to demonstrate possession
of a secret, a byte-for-byte copy is indistinguishable from the original — by construction,
not by omission.

Needed from the issuer: the CA certificate for that generation with an out-of-band
fingerprint, so the data signatures stop being self-referential; and confirmation of
whether any card-authentication mechanism exists that you have not found. Without the
first, even the data check proves nothing.

**7. Older cards: `6D00` to GET CHALLENGE, `6982` to INTERNAL AUTHENTICATE. Newer: the reverse.**

The older generation uses the **asymmetric** mechanism — a PIN-gated private key in a PKI
application. `6D00` says there is no challenge-response to run, and `6982` says the signing
command exists and is merely locked.

The newer generation uses the **symmetric** mechanism — challenge, diversified key,
mutual authentication.

What each needs that the other doesn't:

| | older (asymmetric) | newer (symmetric) |
|---|---|---|
| Needs | a **PIN**, so a person present and consenting; and a **CA certificate** obtained out of band | an **HSM** holding the master key, reachable online |
| Doesn't need | any key service | any certificate, PIN, or person |

Which is why a terminal built for one reports the other as "unverifiable": it probes for the
mechanism it knows, gets "instruction not supported", and stops. The capability was in the
other family the whole time.

**8. Symmetric needs no CA. Does that make it stronger?**

No — it relocates the dependency rather than removing it.

Symmetric trust comes from **holding a secret correctly**: the master key in the HSM is the
anchor, and it is trustworthy because the issuer put it there. Compromise the master and the
whole fleet is forgeable; lose access to the HSM and you can authenticate nothing.

Asymmetric trust comes from **knowing a public key truthfully**: the CA certificate is the
anchor, and it is trustworthy because you obtained it out of band and pinned it. Nothing
secret is shared, so there is no master to steal — but accept the wrong CA, or accept one
handed to you by the card, and every clone verifies.

So the honest comparison is: one asks you to keep a secret safe, the other asks you to know
a public value truthfully. Different failure modes, neither strictly stronger. The scheme
usually picks on infrastructure grounds — whether a key service exists and is reachable —
not on cryptographic strength.

**9. "I was never given a PIN."**

Two situations, and they are not close to each other:

| | Posted-letter scheme (e.g. the German eID) | Enrolment-desk scheme (e.g. Emirates ID) |
|---|---|---|
| What happened | A PIN letter was sent with a one-time PIN under a scratch field. It was binned, lost, or never opened | The PIN is chosen in person straight after fingerprint and photo. If that step was skipped, no PIN was ever set |
| Card state | PKI application present, transport PIN unused | PKI application present, PIN not activated |
| Route out | Use the one-time PIN, or request a replacement letter | Reset at a service centre or kiosk, identity proved by **fingerprint** |

There is a third possibility that has nothing to do with the holder: the card genuinely has
no citizen PKI application, because that generation never carried one.

The probe that distinguishes them is a `SELECT` of the application:

| Response | Means |
|---|---|
| `6A82` | The application is not there. Nothing to PIN, and no letter would have helped |
| `9000` | It exists. The PIN question is real, and it is an administrative problem, not a card one |

Note what you must *not* do to find out: try a PIN. Every wrong `VERIFY` decrements the
retry counter, typically three deep, and a guess made "just to see" spends a life you
cannot get back without the holder's fingerprint or a PUK. The holder's memory is not
evidence about the card; the status word is.

**10. Border desk needs no secret; web service needs the PIN. Same chip, same cryptography.**

Because the gates are set by **what each key's output claims**, not by the algorithm behind
it.

The border desk runs Active Authentication or Chip Authentication. The key's output means
*"this chip is the one the issuer personalised"* — a statement about a physical object.
The holder is not asserting anything, so their consent is not the thing being collected.
What the terminal must prove instead is that it has the card in its hand, which it does by
opening a PACE channel with the CAN or MRZ printed on the document. Possession of the card
is the credential, and it is the right one: the check is about the card.

The web service reads personal data, and releasing it means *"the holder agrees to hand
this over"*. That is an assertion on the person's behalf, so the gate is the person: the
PIN, which only they hold. BSI TR-03110 encodes exactly this split by terminal role — an
Inspection System uses CAN or MRZ, an Authentication Terminal uses the PIN.

The mistake this question exists to kill: reading "asymmetric" and reaching for `VERIFY`.
`6982` from `INTERNAL AUTHENTICATE` says the mechanism is gated. It does not say the gate
is a PIN, and asking a holder for one to answer a question about their card's authenticity
is a category error that also happens to burn a retry.

**11. Building against the "test cards only" secure-messaging module.**

What the passing tests establish: that the developer's *code path* is wired correctly. The
API sequence is right, the handles are threaded through, the responses are parsed, the
error codes are handled. That is real and worth having.

What they establish about a live card: **nothing at all.** The software module holds test
keys, and it is paired with test cards personalised under those keys. A genuine card is
diversified from the issuer's real master, which lives in a SAM or an HSM the developer
does not have. The green test suite has never once verified a cryptogram a real card
produced.

The failure is specific and nasty: it surfaces at deployment, not at build time, and it
surfaces as `E_SM_CARD_NOT_GENUINE` against cards that are perfectly genuine. The natural
reading of that error is "this card is fake", so the first day of the rollout is spent
investigating the cards instead of the key module.

Two things follow for how you read a deployment guide. First, "test only" markers in a
config file are a statement about the **trust anchor**, not about feature completeness —
the same lesson as this chapter's trust-anchor trap, in a different costume. Second, the
thing you actually need to procure — a SAM, an HSM, or credentials for the issuer's online
key service — has a lead time measured in paperwork, so discovering the requirement in
integration testing is far too late.

**12. "Wrong length" at EXTERNAL AUTHENTICATE — key or envelope?**

Your **envelope**, not your key. A length or format rejection happens while the card is
still *parsing* the command — checking the claimed data length against what the command
expects — which is before it has looked at any cryptographic content. No key was exercised,
so **no retry or lock counter moved.** That is categorically different from a cryptogram
that parses correctly but fails its MAC/decryption check: that is a genuine authentication
failure, it decrements the retry counter, and enough of them lock the key.

Why it matters for how many more times you can try: length errors are **free and repeatable**
— you can iterate on the command structure as long as the card keeps rejecting on length,
because you are never reaching the part that charges you. The moment the status changes from
"wrong length" to a crypto-level failure, you have the envelope right and every further
attempt is now spending a real, finite budget.

What to check before the next attempt: the *structure and cipher*, not the key. A wrong
length usually means you are speaking the wrong generation's secure channel — a 3DES
cryptogram where the card wants AES or vice versa, which differ in both length and
construction. Fix the envelope (right cipher, right block layout, right length) while the
attempts are still free; only once the length is accepted should you spend the one attempt
that tests whether the key itself is right.

**13. A 32-byte key for a 3DES channel.**

Almost certainly the **cipher** is wrong — you have been handed the wrong generation's key
material. The length alone tells you because a derived key is always exactly the size its
cipher demands: double-length 3DES is 16 bytes, AES-256 is 32. A 32-byte key is an AES key.
If the card's secure channel is 3DES, that key cannot drive it no matter how you frame the
command — 3DES simply does not take a 32-byte key.

Note what is *not* wrong: the diversifier may be perfectly correct (the service diversified
over the right card serial), and the service may be behaving exactly as designed — it just
implements the *newer* generation's derivation. The mismatch is that this particular card
belongs to an older generation the service was never taught to derive for. The fix is not to
reshape the command; it is to obtain the older generation's key in its own cipher, which may
mean a different derivation mode, a different key label, or a different key source entirely —
and it may not exist at all, if the older scheme was dropped when the infrastructure moved to
AES. Key length is the cheapest possible tell that you are in this situation, long before any
card touches a reader.

---

## Things to try on a real card

In rough order of risk. Everything in the first two groups is free.

**Observe**

1. Read the ATR and decode `T0` by hand. Predict T=0 or T=1 before you send anything, then
   confirm by whether `61 XX` appears.
2. `GET DATA` at tag `66` for card recognition data — GlobalPlatform version and secure
   channel protocol. If it answers `6C XX`, re-issue at the length it names.
3. `GET DATA` at tag `9F7F` for CPLC. Decode the IC serial number and the fabrication
   dates; check they are consistent with the card's issue date.

**Explore**

4. `SELECT` with an empty AID to land on the default applet, then try `READ BINARY`. Note
   the status word. Select a real applet and try again — explain the difference.
5. Scan a FID range inside an applet with `P1=00`, then with `P1=02`. Compare the results
   and work out which dialect the card speaks.
6. Take one file's FCI and decode every tag by hand. Predict its size, then read it and
   check you were right.

**Reason about, don't run**

7. Work out, for a card in front of you, which of the three genuineness mechanisms it
   supports — using only free probes and status words.
8. For each mechanism it supports, write down precisely what a clone could fake and what
   it couldn't.
9. Identify every counter on the card and what decrements it. Then decide which experiments
   you are *not* going to run.

## Questions worth asking me

- "Decode this ATR for me and tell me what I'll be speaking."
- "Here's a trace where the third command fails — is the cause upstream?"
- "What's the difference between `6A82` and `6A86` in practice, with examples?"
- "Given this card only supports X, what security claim can I honestly make?"
- "Which of these probes costs me something if it fails?"
- "Why would a card implement two certificates with different key usages?"
