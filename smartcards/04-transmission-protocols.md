# 4. T=0 and T=1 — why the same code works on one card and not another

## The problem

Chapter 3 described a clean model: send a command, receive data followed by a status word.
That model is **T=1**. It is not how every card works, and the other protocol —
**T=0** — cannot express it at all.

The distinction is invisible in your source code. It surfaces as a card that appears to
refuse commands it plainly supports, and it is the single most common reason working
software fails against a card it has not met before.

## T=1 is block-oriented

Each direction sends a framed block with a length. A response block carries the data and
the two status bytes together. Nothing more needs saying, which is why it is the model
everyone carries in their head.

## T=0 is byte-oriented, and cannot do that

T=0 predates T=1 and is much simpler electrically. Its constraint is that a response is
**either** data **or** a status word, never both in one exchange. The card therefore uses
two status words as a negotiation:

| Status | The card is saying | You must |
|---|---|---|
| `61 XX` | "I have `XX` bytes of response for you" | Send `GET RESPONSE` — `00 C0 00 00 XX` |
| `6C XX` | "Your `Le` was wrong; use `XX`" | Re-send **the same command** with `Le = XX` |

**Neither is an error.** Both mean *success, with a follow-up*.

A worked exchange, where the card corrects the length:

```
→ 80 CA 00 66 00          asking with Le = 0 (256)
← 6C 4E                   "no — ask for 0x4E"
→ 80 CA 00 66 4E          same command, corrected length
← 66 4C 73 4A …  90 00    78 bytes, success
```

And one where the data waits behind a second command:

```
→ 00 A4 04 00 0C A0 00 …  SELECT an application
← 61 21                   "33 bytes waiting"
→ 00 C0 00 00 21          GET RESPONSE
← 85 10 00 00 …  90 00    the FCI
```

A card may chain: the response to `GET RESPONSE` can itself end in `61 XX`, and you loop.

## What goes wrong when software assumes T=1

Every one of these is a real failure mode, and none of them looks like the cause:

| Where the assumption lives | How it presents |
|---|---|
| A capability probe | "Feature not found" — the data was there, behind `61 XX` |
| A `SELECT` whose FCI you parse | Every file has "unknown size"; readers fall back to reading until EOF |
| A read loop using `Le = 0x00` | The reader or driver refuses the 256-byte form outright, as a transport error with no status word at all |
| A multi-step sequence | A *later* command fails with `69 85` or `6A 82`, because an earlier step silently didn't complete |

The last row is the nastiest. The failure surfaces several commands downstream of its
cause, pointing at the wrong thing entirely.

> **Teacher's aside.** The reason this bug survives so long in a codebase is that it is
> invisible until a T=0 card arrives, and T=0 cards are usually the *older* ones. So the
> code is written and tested against current cards, ships, works for years, and then fails
> on exactly the population least likely to be in anyone's test drawer. If you maintain
> card software, the cheapest insurance is to handle `61 XX` and `6C XX` in **one** place
> that every command goes through — see below, because that is where it actually goes
> wrong.

## Handle it once

The `61`/`6C` logic is four lines. The danger is not its difficulty but its duplication:
codebases accumulate several transmit helpers — one for the pretty-printing debug path,
one for the fast path, one inside a discovery tool — and each one independently forgets.
Every such omission produces a *different* symptom, so they don't get fixed together.

The rule: exactly one function talks to the reader. Everything else calls it.

```
send(apdu):
    r = transmit(apdu)
    if r.sw1 == 0x6C:                       # wrong length
        r = transmit(apdu with Le = r.sw2)
    while r.sw1 == 0x61:                    # more data waiting
        r += transmit(00 C0 00 00 r.sw2)
    return r
```

## Two more T=0 consequences

**`Le = 0x00` is risky.** It means 256, and some card–reader–driver combinations refuse it
while accepting an explicit smaller length. Asking for a length you actually want costs
nothing, because a card that disagrees answers `6C XX` and the loop above corrects it.

**Extended-length APDUs don't exist.** Transfers longer than 255/256 bytes must be chunked
by the caller, typically by reading a file in offsets.

## Check yourself

1. A card answers `6C 4E`. Write the exact next APDU you send, given the original was
   `80 CA 00 66 00`.
2. Why can't a T=0 card return data and a status word in the same exchange? Answer in
   terms of what the protocol carries, not history.
3. Your capability probe reports "not supported" on one card and works on another, with
   identical code. The ATRs differ in one nibble. Which nibble, and what is happening?
4. A `SELECT` succeeds, and the `READ BINARY` two commands later returns `69 85`. Give a
   T=0 explanation in which the `SELECT` is not the problem.
5. Why is duplicating the `61`/`6C` handling across several transmit helpers worse than
   having no handling at all in exactly one of them?
