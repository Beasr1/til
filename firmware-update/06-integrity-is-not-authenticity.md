# 6. Integrity is not authenticity

## The problem

You have a firmware image and a checksum. The checksum matches. What have you learned?

You have learned that the bytes you have are the bytes someone computed the checksum over.
You have learned nothing at all about *who* that someone was.

This is the single most common confusion in firmware distribution, and it is easy to fall
into because the check genuinely does something useful — it catches a truncated download, a
bad disk sector, a corrupted transfer — and because the field in the file is very often
called a "signature".

## What DFU actually provides

DFU 1.1 defines a **file suffix**, appended to every downloadable file (Appendix B). Its
purpose is stated in the first sentence:

> The purpose of the DFU suffix is to allow the operating system in general, and the DFU
> operator interface application in particular, to have a-priori knowledge of whether a
> firmware download is likely to complete correctly. In other words, these bytes allow the
> host software to detect and prevent attempts to download incompatible firmware.

The layout, at negative offsets from end of file:

| Offset | Field | Size | Contents |
|---|---|---|---|
| −0 | `dwCRC` | 4 | CRC of the entire file, excluding `dwCRC` itself |
| −4 | `bLength` | 1 | 16 in this revision, including `dwCRC` |
| −5 | `ucDfuSignature` | 3 | `44h 46h 55h` — 'D', 'F', 'U' — appearing in the file in reverse order |
| −8 | `bcdDFU` | 2 | DFU specification number |
| −10 | `idVendor` | 2 | `FFFFh`, or must match the device's vendor ID |
| −12 | `idProduct` | 2 | `FFFFh`, or must match the device's product ID |
| −14 | `bcdDevice` | 2 | `FFFFh`, or a BCD firmware version — informational only |

The CRC is the ordinary CRC-32; the specification's own reference implementation in
Appendix B.1 names the polynomial `0xedb88320` and attributes the algorithm to ANSI X3.66.

And here is the sentence that settles what the whole mechanism means:

> The host application verifies that the bytes occupying the `ucDfuSignature` field contain
> the specified values, and that the CRC over the file matches the `dwCRC` field. If these
> two criteria are passed, then the host can presume that the firmware upgrade file is
> **intact**.
>
> — USB DFU 1.1, Appendix B (emphasis added)

Intact. Not genuine, not authorised, not from anybody in particular. The spec chose the
right word.

> **Teacher's aside.** `ucDfuSignature` is the trap, and it is a vocabulary trap rather
> than a technical one. In this file format, "signature" means what a file-format person
> means by it — a fixed magic value that identifies the format, like `%PDF` or `PK\x03\x04`
> — and not what a cryptographer means by it. They are unrelated ideas that share a word.
> Every occurrence of the word "signature" in the DFU 1.1 document is part of the
> identifier `ucDfuSignature`; there is no other kind in there. When you read a firmware
> format's documentation, check which sense is meant before you conclude anything about
> what it protects.

## The absence is not an oversight, and it is documented

Two points, both checkable.

**DFU 1.1 does not discuss security at all.** Searching the published specification text
for the strings *secure*, *authentic*, *encrypt*, *tamper*, *malicious* and *attack*
returns nothing. It is a 2004 device-class specification for getting bytes into flash
reliably, and it does that; it never claims to do anything else.

**A silicon vendor's DFU implementation says the same thing explicitly.** ST's application
note for its built-in bootloader has a short section headed "Communication safety":

> The communication between host and device is secured by the embedded USB protection
> mechanisms (e.g. CRC checking, acknowledgments). No further protection is performed for
> transferred data or for bootloader specific commands/data.
>
> — AN3156, §2

Read that carefully. "Secured by CRC checking and acknowledgments" means *protected against
transmission errors*. The second sentence says exactly what is missing.

## Vendor containers usually do the same thing again

Device makers commonly wrap the raw image in a container of their own, with a header
carrying magic numbers, a declared length, a version and a CRC32, and sometimes a footer
with more of the same. It is worth knowing what such a container buys you, because at a
glance it looks like more than the DFU suffix and usually is not.

| A header field | What it stops | What it does not stop |
|---|---|---|
| Magic numbers | Feeding the updater an unrelated file | Anyone who knows the magic |
| Declared size matching the file | Truncation | Anyone who can count |
| CRC32 over the payload | Corruption in transit or at rest | Anyone with a CRC-32 routine |
| Version field | Installing the wrong release by accident | Anyone editing two bytes and recomputing the CRC |
| A second CRC, XORed with a constant | A naive edit that misses one of them | Anyone who spends an afternoon with two sample images |

That last row is worth dwelling on. Obfuscating a checksum — masking it with a fixed key,
splitting it, hiding it at an odd offset — raises the cost of forging a container from
*trivial* to *mildly annoying*, exactly once, for the first person who bothers. It buys
nothing against the second. If you find yourself reasoning about how hard a container
format is to reverse-engineer, you have already left the domain where the format is
providing security.

**Every field in the table is a function of the payload that anyone holding the payload can
compute.** That is the definition of integrity and the boundary of it. Nothing derived
purely from the data can tell you who produced the data.

## What authenticity would actually require

A signature, in the cryptographic sense: a value over the image that can only be produced
with a private key, and checked with the corresponding public key. The check is
asymmetric — verifying proves the signer had the key, and verifying does not let you sign.

Which raises the question the rest of this chapter is about: **who checks it, and against
what key?**

```
   build ──▶ signed image ──▶ distribution ──▶ host tool ──▶ device
                                                   │            │
                                        check here ┘            └ or check here
```

| Checked by | Protects against | Does not protect against |
|---|---|---|
| **The host tool**, before sending | A tampered file on disk, a compromised mirror, an operator with the wrong build | Anyone who can run a *different* host tool against the device |
| **The bootloader**, before accepting or before jumping | Every host, including hostile ones and the vendor's own tool | Nothing much — this is the strong position, if the key is protected |

The second row is what is usually meant by *secure boot*. It is a property of the
bootloader, not of DFU: the DFU protocol has no place to put a signature and no request
that means "verify this". Some silicon vendors ship bootloaders and toolchains that add
signed and encrypted image support on top; whether the specific part in front of you does
is a question for its reference manual, and the answer is frequently no for the plain
in-silicon bootloader.

⚠️ Host-side verification and device-side verification are not two strengths of the same
control. They defend against different attackers. If the device accepts any image with a
valid CRC, then *any* software that can reach the bus can reflash it, and your host-side
signature check is a policy your own tool follows and nobody else is bound by.

## Where the trust boundary actually ends up

So: the format has no signature and the device checks nothing. You still have to ship
updates. Where does the trust live now?

It moves, entirely, to **everything upstream of the bytes arriving at the device** — and it
is worth writing that chain down, because each link is a place someone can be wrong.

| Link | The question | A way to answer it |
|---|---|---|
| Where did the image come from? | Was it served by us, over a channel that authenticates the server? | TLS to a service you control, with a pinned or properly validated chain |
| Was it the image we meant? | Does it match a hash we published out of band? | A signed manifest listing image hashes — the signature you could not put in the container, put beside it |
| Who may trigger an update? | Is this caller allowed to flash this device? | Authorisation at the API, and a server-side policy that can be turned off centrally |
| What is doing the flashing? | Is the updater on this machine the one we shipped? | Sign the updater binary; verify it before invoking it |
| Which device is it allowed to flash? | Have we pinned the target? | Match on the device's own serial number (chapter 2), refuse when ambiguous |

Note the second row in particular. If the container format cannot hold a signature, you are
not obliged to abandon signing — you are obliged to move it. A detached signature over a
manifest of image hashes gives you the same cryptographic property in a different file. The
device still cannot check it, so this defends the host and the channel rather than the
device, but that is a real and worthwhile boundary and it is much better than none.

And the honest statement of what you have then bought, which belongs in your own
documentation rather than being quietly omitted:

> Any software running with sufficient privilege on this machine, and anyone with physical
> access to the device, can write arbitrary firmware to it. Our controls make that hard to
> do *by accident*, and hard to do *through our tooling*. They do not make it hard to do
> deliberately with other tooling.

## Check yourself

1. A firmware container has a magic number, a length, and a CRC32 that is XORed with a
   constant before being stored. An attacker wants to install their own image. What must
   they do, and roughly what does it cost them?
2. DFU 1.1 says a host that checks `ucDfuSignature` and `dwCRC` "can presume that the
   firmware upgrade file is intact". Rewrite that sentence to say precisely what has and
   has not been established, for someone who will quote you in a security review.
3. Your updater verifies a detached signature over the image before sending it, and the
   device accepts anything with a valid CRC. Name one attacker this stops and one it does
   not, and say which of the two you would expect a security reviewer to ask about first.
4. `idVendor` and `idProduct` in the DFU suffix may be `FFFFh`. What does that mean, and
   what has to be true elsewhere for it to be safe?
5. Someone proposes moving the version field in the container to a different offset "so
   people can't tamper with it". Explain, in terms of the integrity/authenticity
   distinction, why this is not a security measure — and then name the one thing it
   genuinely costs an attacker.
6. Your device's bootloader verifies a signature before jumping to the application. Where
   does the public key live, and what is the new question you now have to answer that you
   did not have before?
