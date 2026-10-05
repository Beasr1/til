# 2. The ATR — the card's first and only unsolicited message

## The problem

The reader has just applied power to a chip it knows nothing about. How fast can it be
clocked? Does it speak the byte-oriented protocol or the block-oriented one? How much
voltage does it want? None of that can be negotiated by asking, because asking requires
already knowing how to ask.

So the card speaks first. Exactly once, on reset, before any command, it emits the
**Answer To Reset** — a short byte string describing its own electrical and protocol
capabilities. It is the only thing a card ever says unprompted. ISO/IEC 7816-3 defines it.

## The shape

```
TS  T0  [TA1 TB1 TC1 TD1]  [TA2 …]  …  historical bytes  [TCK]
```

**`TS`** — the initial character, which also tells the reader the bit convention. `3B` is
direct convention, `3F` is inverse. Modern cards are almost always `3B`.

**`T0`** — the format byte, and the one worth being able to read:

```
T0 = 0x7A
     ├─ high nibble 0x7 = 0b0111 : TA1, TB1, TC1 present; TD1 absent
     └─ low  nibble 0xA = 10     : ten historical bytes follow
```

The high nibble is a **presence bitmap** for the next four interface bytes, in the order
TA, TB, TC, TD. The low nibble is a count of historical bytes at the end.

**`TA1`/`TB1`/`TC1`** — clock rate conversion, programming voltage, extra guard time.
Timing parameters; rarely interesting unless something is failing electrically.

**`TD1`** — and this is the important one. If present, its low nibble names a
**transmission protocol** the card supports, and its high nibble is another presence
bitmap for a further round of interface bytes.

**Historical bytes** — up to fifteen bytes whose meaning is largely up to the card issuer.
Sometimes they encode a card-capabilities structure defined in 7816-4; often they are
simply an issuer string. Do not build logic on them without the issuer's documentation.

**`TCK`** — a checksum, present when any protocol other than T=0 is indicated.

## Reading two real ATRs

A card with no `TD1`:

```
3B 7A 95 00 00 80 65 A2 01 31 01 3D 72 D6 41
│  │  │  │  │  └──────────────────────────┘
│  │  │  │  │   ten historical bytes
│  │  │  │  └─ TC1 = 00
│  │  │  └──── TB1 = 00
│  │  └─────── TA1 = 95
│  └────────── T0  = 7A → TA1,TB1,TC1 present, no TD1, 10 historical bytes
└───────────── TS  = 3B, direct convention
```

No `TD1` means **no protocol was ever advertised**, and the default applies: **T=0**.
There is no checksum, which is consistent.

A card with `TD1`:

```
3B B8 97 00 81 31 FE 45 FF FF 14 82 30 50 23 00 F1
│  │  │  │  │  └─ TD2… and onward
│  │  │  │  └──── TD1 = 81 → high nibble 8: TD2 present; low nibble 1: T=1
│  │  │  └─────── TB1 = 00
│  │  └────────── TA1 = 97
│  └───────────── T0  = B8 → TA1,TB1,TD1 present, 8 historical bytes
└──────────────── TS  = 3B
```

`TD1 = 0x81` advertises **T=1**, and the trailing `F1` is the checksum.

That single nibble is the difference between the two cards behaving completely differently
under identical software — see chapter 4, which is entirely about the consequences.

## The synthesised contactless ATR

A contactless card **never emits an ATR**. The ATR is an artefact of the contact
interface — the chip producing it as the reader applies voltage — and a card energised by
an RF field goes through the ISO 14443 anti-collision and select sequence instead,
answering with an **ATS**.

So PC/SC fabricates one, from the ATS, so that host software has something of the familiar
shape to consume. The result has a fixed opening:

```
3B 8N 80 01 <historical bytes from the ATS> <TCK>
```

A real one, decoded:

```
3B 89 80 01 FF FF 00 00 30 50 23 00 00 4B
│  │  │  │  └───────────────────────┘  └─ TCK
│  │  │  │   nine historical bytes, from the ATS
│  │  │  └─ 0x01
│  │  └──── 0x80
│  └─────── T0 = 0x89 → only TD1 present; nine historical bytes
└────────── TS = 3B
```

In a real ATR those two bytes would be `TD1` and `TD2`: `0x80` meaning "another interface
byte follows, protocol T=0 coding" and `0x01` meaning "no more, T=1". Here they are better
read as a **literal PC/SC convention** for "contactless, ISO 14443-4" — nothing was
negotiated, because there was no power-up to negotiate during.

**That `3B 8x 80 01` opening is the giveaway**, and a contact card cannot produce it. In
those positions a contact card carries its own interface bytes — `TA1`, `TB1`, `TC1`,
describing the clock division and voltage it actually negotiated. Compare a contact card
from the same product family:

```
contactless   3B 89 80 01 FF FF 00 00 30 50 23 00 00 4B
contact       3B B8 97 00 81 31 FE 45 FF FF 14 82 30 50 23 00 F1
                    └──┴──┘ TA1, TB1 — real negotiated parameters
```

Note that both carry `30 50 23 00` among the historical bytes. That is issuer data, so it
survives across interfaces; the interface bytes do not, because they describe the link
rather than the card.

This shape is worth knowing because it is often the **only** reliable way to tell which
interface you are on. Chapter 1 noted that the reader's UID pseudo-APDU is a reader
feature: a reader that doesn't implement it answers "class not supported", which looks
exactly like a contact card and is nothing of the sort. The ATR shape settles it when the
probe cannot.

## What the ATR does not tell you

This is the part people get wrong, so it gets its own section.

The ATR describes **electrical and protocol capability**. It does not describe:

- who issued the card
- what applications are on it
- which version of a card scheme it belongs to
- what data it holds

Every system that maps "ATR → card model" does so through a **lookup table someone
maintained by hand**. That table is a claim about the world, not a fact read from the card.
Two consequences follow:

- A card whose ATR isn't in the table is unidentifiable, even if it works perfectly.
- A card whose ATR *is* in the table is identified as whatever the table says — including
  when the table is wrong, or when one ATR is shared by several card models.

ATRs are frequently shared across an entire manufacturing batch or product line, because
they describe the silicon and its configuration. Two cards with different applications,
different issuers and different data can have byte-identical ATRs.

> **Teacher's aside.** If you need to know *what* a card is, ask the card: select its
> applications, read its capability data objects, look at what files exist. The ATR tells
> you how to *talk* to it, and that's all it was ever designed to do. Treating it as an
> identity is the single most common design error in card software, and it fails silently —
> you get a confident wrong answer rather than an error.

## One card model, five ATRs

The section above is about one ATR covering many cards. The reverse happens too, and it
catches people just as often.

The UAE's Emirates ID terminal toolkit ships a configuration file that has to list the
ATRs of live cards so the middleware can recognise one. It needs **five entries** — for a
single card programme:

```
[UAECard]
NUMBER = 5
ATR1 = 3B6A00008065A20130013D72D641
ATR2 = 3B6A00008065A20131013D72D641
ATR3 = 3B7A9500008065A20130013D72D641
ATR4 = 3B7A9500008065A20131013D72D641
ATR5 = 3B8A80018065A20131013D72D641A5
```

You have already decoded one of these. ATR 4 is the "card with no `TD1`" worked through
in §Reading two real ATRs above, byte for byte.

Apply the same rules to all five and the pattern is clear:

| ATR | `T0` | interface bytes | hist. | protocols | TCK |
|---|---|---|---|---|---|
| 1 | `6A` | `TB1=00`, `TC1=00` | 10 | T=0 | none |
| 2 | `6A` | `TB1=00`, `TC1=00` | 10 | T=0 | none |
| 3 | `7A` | `TA1=95`, `TB1=00`, `TC1=00` | 10 | T=0 | none |
| 4 | `7A` | `TA1=95`, `TB1=00`, `TC1=00` | 10 | T=0 | none |
| 5 | `8A` | `TD1=80`, `TD2=01` | 10 | T=0, then **T=1** | `A5` |

Three things vary, and none of them is *what the card can do for you*:

- **`TA1`.** ATRs 1–2 omit it, so the card runs at the default divider. ATRs 3–4 carry
  `TA1=95`, requesting faster clock-rate/bit-rate parameters. Same chip, different speed
  negotiation.
- **One historical byte.** All five carry ten, all beginning `80` — the ISO/IEC 7816-4
  category indicator marking what follows as compact-TLV data objects — and all ten are
  identical bar one: the fifth byte is `30` in ATRs 1 and 3, `31` in ATRs 2, 4 and 5. An
  issuer's own versioning field, sitting inside bytes the standard leaves proprietary.
- **Protocol support, and therefore the checksum.** Only ATR 5 has a `TD1`, indicating T=0
  and then T=1. Because T=1 is offered, a `TCK` becomes mandatory — the trailing `A5`,
  which is the XOR of every byte from `T0` to the last historical byte. It checks out.

> **Teacher's aside.** `NUMBER = 5` is the whole lesson in one line. Someone had to count.
> The ATR is a description of an electrical and protocol configuration, so it changes when
> the silicon, the mask, the speed parameters or the personalisation profile changes — none
> of which need alter a single command the card answers. A table keyed on the exact ATR
> string therefore needs a row per *configuration*, not per card model, and it acquires new
> rows without anybody telling you. That is the failure mode: the sixth configuration ships,
> the table has five rows, and perfectly good cards start reading as unknown.

Note also what is *identical* across all five: the historical bytes are ten long and begin
`80` in every case. If you must key on something, key on the stable, meaningful part and
treat the rest as noise — or better, take the advice above and ask the card.

## Check yourself

1. From `T0 = 0x6A`, say exactly which interface bytes follow and how many historical
   bytes there are.
2. A card's ATR contains no `TD1`. Which transmission protocol will you be speaking, and
   how do you know without asking the card?
3. Why does the contactless ATR have to be synthesised, and what gives it away as
   synthesised?
4. Two cards from different issuers, holding different applications, have identical ATRs.
   Is that a manufacturing fault? Explain.
5. A version-detection routine reads the ATR, looks it up in a table, and returns "version
   2". Describe two distinct ways that answer can be wrong while the code is working
   exactly as written.
6. A single card programme needs five ATRs listed in its terminal software. Nothing about
   what the card *does* differs between them. Name three things that can differ, and say
   which of them a `TCK` byte tells you about.
7. You are handed `3B8A80018065A20131013D72D641A5`. Work out the protocols on offer, then
   verify the checksum by hand. What would have been wrong with this ATR if the final byte
   were absent?
