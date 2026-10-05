# Smart Cards — A Course

How a contact smart card actually works, from the first byte it emits on power-up to
proving it holds a secret nobody can copy.

**This is reference learning material.** It's about the standards — ISO/IEC 7816,
GlobalPlatform, PC/SC — not about any particular system you might be building. Nothing
here is project-specific.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- *Why* before *how*: each chapter opens with the problem that made the design necessary.
- Every file ends with **Check yourself** questions. Answers are in
  [`08-exercises.md`](08-exercises.md).
- Smart cards have forty years of jargon that everyone pretends is obvious. None of it is.

## The primary sources

Everything here is checkable against one of these. Where I say "the standard says", this
is which.

| Standard | Covers |
|---|---|
| **ISO/IEC 7816-3** | Electrical interface, power-up, the ATR, the T=0 and T=1 transmission protocols |
| **ISO/IEC 7816-4** | The file system, APDU wire format, the interindustry commands, status words, FCI |
| **ISO/IEC 7816-8** | Commands for security operations — signing, internal/external authentication |
| **ISO/IEC 7816-9** | File management and life-cycle |
| **GlobalPlatform Card Specification** (and its amendments) | The applet model, Issuer Security Domain, card content management, the Secure Channel Protocols. SCP03 is defined in Amendment D |
| **PC/SC Specification** (PC/SC Workgroup) | The host-side reader API every OS implements; Part 3 covers the synthesised ATR for contactless cards |
| **ICAO Doc 9303** (Part 11) | Security mechanisms for machine readable travel documents and eID cards: BAC, PACE, Active Authentication, Chip Authentication. Cited in chapter 6 |
| **BSI TR-03110** | Advanced security mechanisms for MRTDs and eIDAS tokens — the PACE password types and the terminal roles that may use each. Cited in chapter 6 |

ISO standards are paywalled. The 7816-4 command set and status words are widely mirrored;
GlobalPlatform publishes its specifications free at <https://globalplatform.org/specs-library/>.
ICAO Doc 9303 is free from <https://www.icao.int> and BSI TR-03110 from
<https://www.bsi.bund.de>.

## Reading order

Read 1–4 in order. They build.

| # | File | After this you can… |
|---|---|---|
| 1 | [What a smart card is](01-what-a-smart-card-is.md) | Say why a card is not a USB stick, and what a reader does and doesn't do |
| 2 | [The ATR](02-the-atr.md) | Read an ATR byte by byte, know which transmission protocol you're about to speak, and see why one card programme can need five ATRs |
| 3 | [APDUs and status words](03-apdus-and-status-words.md) | Write any command by hand and interpret what comes back |
| 4 | [T=0 and T=1](04-transmission-protocols.md) | Explain why the same command works on one card and appears to fail on another |
| 5 | [How cards are organised](05-how-cards-are-organised.md) | Navigate a card's files, or recognise that it hasn't got any |
| 6 | [Proving a card is genuine](06-proving-genuineness.md) | Explain what a clone can and cannot fake, which direction of a handshake proves it, why one scheme can need two different mechanisms, and which credential actually gates which — worked through two real national eID schemes |

Reference, not reading:

| # | File | |
|---|---|---|
| 7 | [Glossary](07-glossary.md) | Every abbreviation, in one place |
| 8 | [Exercises](08-exercises.md) | Worked answers to every **Check yourself**, plus things to try |

### If you're short on time

| You want | Read |
|---|---|
| To debug a card that "isn't responding" | 4, then 3 §status words |
| To understand card security claims | 6, then 2 §what the ATR doesn't tell you |
| To read someone's APDU trace | 3, then 5 |
| To work out which credential a card is asking you for | 6 §which credential gates which mechanism, then 6 §two national eID schemes |
| The whole model in twenty minutes | 1, then the summary below |

## The one-paragraph summary of everything

A smart card is a tamper-resistant computer that will *use* a secret without ever
revealing it — that property, not storage, is the whole reason the format exists. On
power-up it emits an **ATR** describing its capabilities, including which transmission
protocol it speaks: **T=1** returns data and a status together, while **T=0** cannot, and
instead answers `61 XX` ("come and fetch it") or `6C XX` ("wrong length, use this"), which
software that assumes T=1 misreads as failure. Thereafter every exchange is an **APDU** —
`CLA INS P1 P2` plus optional data and an expected length — answered by two status bytes,
most of which describe the card's *current state* rather than a permanent property, so the
same command can fail and then succeed. Cards organise their contents either as an ISO
**file tree** (MF at the root, DFs as directories, EFs as files, selection being stateful)
or as **GlobalPlatform applets** selected by AID, where the file commands only exist
*inside* an applet — and many real cards are the second while looking like the first.
Finally, because anything readable can be copied, genuineness can only be established by
making the card **prove possession of a secret**: symmetrically, through a challenge and a
diversified key an HSM re-derives, or asymmetrically, by having the card sign a challenge
you chose. Both are only meaningful if the certificate involved chains to a trust anchor
you obtained somewhere other than from the card. And the credential that unlocks any of
this is set by **what the key's output claims**, not by the algorithm: a key proving the
chip is genuine is gated on holding the card — often a number printed on its face — while
a key signing on the holder's behalf is gated on the holder's PIN.

## How to use me

These notes are a starting point. Good questions to bring back:

- "Walk me through this ATR byte by byte" — paste one.
- "Here's an APDU trace; what's the card telling me?"
- "Why would `READ BINARY` return `6D00` here but `9000` two commands later?"
- "If a card returns `9000` to `EXTERNAL AUTHENTICATE`, what exactly has been proven?"
- "What could a cloner fake in this handshake, and what couldn't they?"
- "Which of these status words costs me something if I retry?"
- "This card wants a credential — which kind, and where would a holder get it?"
- "Compare how two schemes I name solve the same genuineness problem."
