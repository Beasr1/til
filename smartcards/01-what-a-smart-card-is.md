# 1. What a smart card is

## The problem that made them

You want to give a million people a physical token that proves who they are. The obvious
design is a card with an identifier written on it. Every such design fails the same way:
anything a verifier can read, an attacker can read too, and then write onto a card of
their own. A magnetic stripe is a perfect example — it holds a few hundred bytes, any
reader can read them, and copying them takes seconds.

The problem isn't storage. It's that **possession of readable data proves nothing**.

The fix is to stop reading the secret at all. Put a small computer on the card, give it a
secret it will never emit, and let it *use* that secret on demand — sign this challenge,
decrypt this blob. A verifier never learns the secret; it only observes that whoever is
holding the card can do something only the secret makes possible.

That is a smart card: not a storage medium with a chip attached, but a **tamper-resistant
computer that computes with a key it will not disclose**.

> **Teacher's aside.** Almost every confusion about smart cards traces back to thinking of
> them as storage. "Can I read the key off it?" — no, and if you could, the format would be
> pointless. "Why is reading this file so slow?" — because you're talking to a
> microcontroller over a serial line at a few hundred kilobits, not to flash. "Why does it
> need a PIN to sign?" — because the PIN is what authorises the chip to use the key, not
> what decrypts a file for you.

## The parts

```
┌─────────────┐   USB    ┌────────────┐   ISO 7816   ┌───────────┐
│ your code   │ ───────▶ │   reader   │ ───────────▶ │  the card │
│  (PC/SC)    │ ◀─────── │  (dumb)    │ ◀─────────── │  (a CPU)  │
└─────────────┘          └────────────┘              └───────────┘
```

**The card** is a microcontroller with its own CPU, ROM, EEPROM and often a crypto
co-processor, sealed in plastic. It has no clock and no power of its own: the reader
supplies both. It never initiates anything. It answers.

**The reader** is a pipe. It powers the chip, clocks it, converts between USB and the
card's electrical interface, and moves bytes. It does not understand what it is carrying.
This matters more than it sounds — it means every question you want answered is *your*
software's to ask.

**PC/SC** is the host-side API that every operating system implements for talking to
readers. It gives you a list of readers, a way to connect to the card in one, and a
`Transmit` call. Almost nothing else. The intelligence is all in what you transmit.

### One thing the reader answers on its own

Readers implement a handful of *pseudo-APDUs* — commands that look like card commands but
are intercepted and answered by the reader. The common one asks a contactless reader for
the card's anti-collision identifier.

Because these are the reader's own feature, a reader that doesn't implement one answers
"class not supported". That failure says nothing whatsoever about the card. It is a very
easy mistake to read such a response as a property of the card in front of you.

## Contact and contactless

**Contact** cards have the gold pad you can see. The reader physically touches it,
supplies power, and holds a stable connection for as long as the card is seated.

**Contactless** cards are powered by the reader's RF field through an antenna in the card
body, following ISO/IEC 14443. Everything above the physical layer is very nearly the
same — the same commands, the same responses.

Two practical consequences follow from the physics, and both bite:

- **The session lives only while the card is in the field.** A hand moving a few
  millimetres ends it. A card in a contact slot is mechanically held; a card hovering over
  an antenna is not.
- **Contactless cards have no ATR of their own** (chapter 2), because the ATR is a
  power-up artefact of the contact interface. PC/SC synthesises one so that host software
  has something of the expected shape to look at.

A "combi" or dual-interface reader exposes each interface as a **separate reader name** in
PC/SC, so one physical device appears as two. Software that connects to "the first reader"
will sometimes connect to the empty slot of a device whose card is in the other one.

## What a card can do

The instruction set is small and it is worth knowing the shape of it before the detail:

| Family | Examples | What it's for |
|---|---|---|
| Selection | `SELECT` | Point at a file or an application |
| Reading | `READ BINARY`, `READ RECORD` | Get bytes out of a file |
| Writing | `UPDATE BINARY`, `WRITE BINARY` | Put bytes in, if permitted |
| Authentication | `VERIFY`, `INTERNAL AUTHENTICATE`, `EXTERNAL AUTHENTICATE`, `GET CHALLENGE` | Prove something to the card, or the card to you |
| Security operations | `PSO: COMPUTE DIGITAL SIGNATURE`, `PSO: DECIPHER` | Use a private key |
| Data objects | `GET DATA`, `PUT DATA` | Read or write a tagged value that isn't in a file |

ISO/IEC 7816-4 defines the first four families; 7816-8 defines the security operations.
Cards may implement a subset, and may add proprietary commands of their own — a point
chapter 3 returns to, because it is a common source of "this card doesn't support X" when
it does.

## Check yourself

1. A colleague proposes storing a signed token on the card and having verifiers read it
   back. What attack does this design not survive, and what does a smart card do instead?
2. Your reader returns "class not supported" to a command. Name two entirely different
   causes, one of which has nothing to do with the card.
3. Why does a contactless card need PC/SC to invent an ATR for it?
4. You connect to "the first reader" and get "no card present", but you can see a card in
   the slot. What is the most likely cause on a dual-interface device?
5. A card takes 400 ms to return a 2 kB file. Your colleague says the card's flash is
   slow. What's a better explanation?
