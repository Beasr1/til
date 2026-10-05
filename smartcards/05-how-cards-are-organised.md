# 5. How cards are organised

## The problem

A card holds a certificate, a photograph, a demographic record, a key, a counter. The
terminal needs a way to name the one it wants. Two quite different answers exist, both in
wide use, and a card can be one, the other, or both at once — which is why a terminal that
assumes the wrong one gets "file not found" for a file that is definitely there.

## Model one: the ISO 7816-4 file tree

A filesystem, with the vocabulary changed.

| Term | Is | Analogy |
|---|---|---|
| **MF** — Master File | The root, file identifier `3F00` | `/` |
| **DF** — Dedicated File | A directory; may contain EFs and other DFs | a folder |
| **ADF** — Application DF | A DF selected by a long name rather than a number | a folder with a global name |
| **EF** — Elementary File | A file with bytes in it. A leaf | a file |
| **FID** — File Identifier | The 2-byte name of any of the above | a filename |
| **AID** — Application Identifier | A registered 5–16 byte name for an application | a reverse-DNS package name |

```
MF (3F00)
├── EF 2F00            (EF.DIR — the list of applications, if present)
├── DF 0100
│    └── EF 0101
└── ADF  A0 00 00 02 47 10 01
     ├── EF 0201
     └── EF 0203
```

### Selection is stateful, and that is the whole trap

The card remembers a **current DF**. `SELECT` by FID is interpreted relative to it, and the
`P1` byte says how:

| `P1` | Means |
|---|---|
| `00` | Select by file identifier |
| `01` | Select a child DF |
| `02` | Select an EF under the current DF |
| `03` | Select the parent |
| `04` | Select by DF name (an AID) |
| `08`/`09` | Select by path, from the MF or from the current DF |

Two consequences that account for a great deal of wasted time:

- **The same FID resolves differently depending on where you are.** A sequence that works
  from the MF fails from inside an application.
- **"File not found" usually means "not found *here*".** `6A82` is a statement about the
  current DF, not about the card.

And a subtler one: not every card supports every `P1` mode. A card that answers `6A86`
(incorrect P1/P2) to `P1=02` may answer `9000` to the same file with `P1=00`. Software
that hardcodes one mode will report an empty card.

### FCI — the card describing what you selected

`SELECT` with `P2=0x00` asks for **File Control Information**. This is the card telling you
about the object you just selected, and it is how you stop guessing. The standard form is a
`6F` template of BER-TLV:

```
6F 13 81 02 01 13 82 01 01 83 02 00 07 8A 01 05 8C 03 03 D2 00
      │        │        │        │        └─ 8C security attributes
      │        │        │        └─ 8A life-cycle status
      │        │        └─ 83 the file's own FID
      │        └─ 82 file descriptor byte: 01 = transparent EF
      └─ 81 total file size = 0x0113 = 275 bytes
```

| Tag | Carries |
|---|---|
| `80` | Data size, excluding structural information |
| `81` | Total file size |
| `82` | File descriptor byte — transparent, linear-fixed, record-structured, or a DF |
| `83` | The file identifier |
| `84` | DF name (the AID), when you selected an application |
| `88` | Short EF identifier |
| `8A` | Life-cycle status |

The size tags are what let a reader size its read loop instead of groping to end-of-file.
The descriptor byte is what tells you `READ BINARY` is the wrong command — a
record-structured file needs `READ RECORD`, and will refuse the other.

> ⚠️ Issuers may use **proprietary FCI formats**. A card is free to answer with a
> vendor-defined template instead of `6F`, carrying the same information in a different
> shape. A parser that only knows the standard tags gets nothing at all from such a card
> and silently reports every file as "size unknown". If FCI parsing yields nothing,
> examine the raw bytes before concluding the card is uncooperative.

### EF.DIR

An optional EF at `2F00` listing the AIDs present, so a terminal can enumerate
applications without knowing them in advance. Useful when it exists — and plenty of cards
don't have one, which quietly defeats any discovery routine built on it.

## Model two: the GlobalPlatform applet model

Here the card is a small runtime — usually a JavaCard — hosting independent **applets**,
each with its own AID, its own data, and its own firewalled memory. GlobalPlatform
standardises how they are loaded, selected and managed.

Before you select an applet, you are talking to the **Issuer Security Domain** — the card
manager. It implements card content management and very little else: `SELECT` by AID and
`GET DATA`, essentially.

The file commands — `READ BINARY`, `SELECT` by FID — are implemented **by applets**, not by
the platform. So on a pure GlobalPlatform card, before any selection:

```
→ 00 A4 00 00 02 3F 00 00     select the MF
← 6A 86                        incorrect P1/P2

→ 00 B0 00 00 10               read 16 bytes
← 6D 00                        instruction not supported
```

Both become available the moment an applet is selected. `6D00` on `READ BINARY` is the
signature of this model: the instruction isn't missing, there is simply nobody to
implement it yet.

An applet may present an ISO file tree *inside* itself. So the two models compose:

```
card manager (GlobalPlatform)      SELECT by AID · GET DATA
├── applet  A0 00 00 02 47 10 01
├── applet  A0 00 …  01 01          ← is itself DF 0200
│     └── EF 0201, EF 0203, …
└── applet  A0 00 …  01 04
      └── EF 0001, EF 0005, EF 0006, …
```

> **Teacher's aside.** Look at the middle applet. Selecting its AID does not put you
> *above* a file tree — it puts you **inside a specific DF**, and that DF's identity is
> reported in tag `83` of its own FCI. If that DF is `0200`, then `0100` is a *sibling*,
> and no amount of selecting from here will reach it: `6A82`, every time, correctly. Code
> written as "select the application, then descend into `DF 0100`" encodes an assumption
> about where the AID lands, and on a card where it lands elsewhere the failure is
> indistinguishable from a missing file. Read the FCI's tag `83` and you know exactly where
> you are.

## Finding out what's actually there

When the documentation and the card disagree, enumerate. Select an application, then walk
a range of FIDs, recording which answer `9000` and what their FCIs say. A few hundred
`SELECT`s take seconds and replace every assumption with an observation.

Three things make the difference between a useful scan and a misleading one:

- **Use the selection mode the card accepts** — try `P1=00` as well as `P1=02`.
- **Scan inside applications, not only from the MF.** On a GlobalPlatform card the MF may
  hold nothing at all.
- **Complete the T=0 exchanges** (chapter 4), or every FCI comes back empty and the map has
  no sizes or file types in it.

## Check yourself

1. Selecting an application's AID succeeds. Selecting `DF 0100` immediately afterwards
   returns `6A82`. Give two different explanations, and say which byte of which response
   distinguishes them.
2. Why is `6D00` on `READ BINARY` evidence about the card's *model* rather than its
   capabilities?
3. You get an FCI back but your parser extracts nothing. What are two possible causes, and
   how would you tell them apart from the raw bytes?
4. A FID scan across `0001–02FF` reports zero files on a card you know holds several. Name
   three distinct causes.
5. An applet's FCI reports `83 02 02 00`. What does that tell you about which files you can
   reach from here, and which you cannot?
