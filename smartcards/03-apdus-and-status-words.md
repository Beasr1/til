# 3. APDUs and status words

## The problem

The card answers, never initiates. So the entire protocol reduces to one shape: you send a
command, it sends a response. ISO/IEC 7816-4 names these **APDUs** — Application Protocol
Data Units — and defines a wire format flexible enough to express "read forty bytes from
here", "check this PIN" and "sign this" without the card and the terminal agreeing on
anything else in advance.

## The command

```
CLA INS P1 P2 [Lc <data…>] [Le]
 │   │   │  │   │             └─ expected response length
 │   │   │  │   └─ length of the data you're sending
 │   │   └──┴─ parameters; meaning depends entirely on INS
 │   └─ instruction
 └─ class
```

Only the first four bytes are always present. Whether `Lc`, data and `Le` appear gives the
four **cases**:

| Case | Shape | Example |
|---|---|---|
| 1 | `CLA INS P1 P2` | a command with no input and no output |
| 2 | `CLA INS P1 P2 Le` | read something |
| 3 | `CLA INS P1 P2 Lc data` | write something |
| 4 | `CLA INS P1 P2 Lc data Le` | send something, get something back |

Worked, byte by byte:

```
00 A4 00 00 02 02 03 00
│  │  │  │  │  └───┘ └─ Le = 00 → "up to 256 bytes back"
│  │  │  │  └─ Lc = 02 → two bytes of data follow
│  │  └──┴─ P1=00 "select by file identifier", P2=00 "return the FCI"
│  └─ INS A4 = SELECT
└─ CLA 00 = interindustry (ISO standard) command
```

A case-4 command, then: it sends two bytes and expects data back.

### `Le = 0x00` means 256, not zero

There is no way to ask for zero bytes, so `0x00` is defined as the maximum for a short
APDU — 256. Extended-length APDUs exist for longer transfers, but not every card or reader
supports them, and T=0 (chapter 4) does not.

## `CLA` is not decoration

The class byte says which *command set* you are speaking. The values that matter:

| `CLA` | Meaning |
|---|---|
| `0X` | Interindustry — the ISO 7816 standard command set |
| `8X`, `9X`, `AX`–`EX` | **Proprietary** — the card's own commands |
| bits within the low nibble | Secure messaging indicator, logical channel number |

A card may implement the *same instruction number* under a proprietary class with
different behaviour, or implement an instruction **only** under a proprietary class.

> **Teacher's aside.** This is worth internalising because of how the failure presents. Ask
> for an instruction under `CLA=00` that the card only offers under `CLA=80`, and you get
> "class not supported" — which reads exactly like "this card can't do that". It can; you
> asked in the wrong language. Before concluding a card lacks a capability, try the
> proprietary class. The difference between "not supported" and "not authorised yet" is
> the difference between abandoning an approach and getting a PIN.

## The response

```
[data…] SW1 SW2
```

Always two status bytes at the end, sometimes data before them. `90 00` is success.
Everything else is the card telling you something about its state.

## Status words, and what they actually mean

The standard defines a structured space: `6X` for warnings and errors, `9X` for success,
with `61`, `62`, `63`, `9000` carrying information rather than failure.

| SW1 SW2 | Standard meaning | What it usually means in practice |
|---|---|---|
| `90 00` | Success | — |
| `61 XX` | `XX` bytes available | Not an error — chapter 4 |
| `6C XX` | Wrong `Le`, use `XX` | Not an error — chapter 4 |
| `62 82` | End of file before `Le` bytes read | **Warning.** The data returned is valid |
| `63 CX` | Verification failed, `X` retries remain | **This one costs you a try** |
| `69 82` | Security status not satisfied | "Authenticate first" — the command exists |
| `69 85` | Conditions of use not satisfied | Often: nothing is selected |
| `6A 82` | File or application not found | Often: it exists, but not under what's selected |
| `6A 86` | Incorrect `P1`/`P2` | Often: the card wants a different selection mode |
| `6D 00` | Instruction not supported | Often: no application is selected yet |
| `6E 00` | Class not supported | Often: try the proprietary class |
| `63 00` | Verification failed | After an authentication step: the key didn't match |

**Read the right-hand column twice.** Most status words describe the card's *current
state*, not a permanent property. The same APDU that returns `6D00` before you select an
application returns `9000` after. A file that answers `6A82` may be perfectly present
somewhere you haven't navigated to.

This is why "the card doesn't support it" is almost always premature. The card supports a
command *in a context*, and the status word is usually telling you the context is wrong.

### The one to be careful with

`63 CX` and `63 00` are different from all the others: they mean something was **checked
and rejected**, and on most cards that decrements a counter. PIN counters typically allow
three to five attempts before the credential locks, and administrative counters may have
no recovery path at all.

So when probing an unfamiliar card, sort your experiments:

- **Free**: `SELECT`, `READ BINARY`, `GET DATA`, `GET CHALLENGE`. Nothing is verified, so
  nothing can be decremented.
- **Not free**: `VERIFY`, `EXTERNAL AUTHENTICATE`, anything presenting a credential — even
  an empty or malformed one, on some implementations.

Exhaust the free probes before spending a try, and be sure of your inputs before you spend
one at all.

## Check yourself

1. Write the APDU that reads 16 bytes from offset 0 of the currently selected file. Which
   case is it?
2. A command returns `62 82`. Your code treats any `SW != 9000` as failure and discards
   the response. What have you just thrown away?
3. `00 88 00 00 08 …` returns `6E00`; `80 88 00 00 08 …` returns `6982`. What have you
   learned from each, and what should you do next?
4. Why is `6D00` on `READ BINARY` more likely to be a selection problem than a missing
   feature?
5. You are exploring an unfamiliar card and want to know whether a PIN is required for an
   operation. Design a probe that answers the question without decrementing anything, and
   say what each possible response would tell you.
