# 4. DfuSe — addresses, erasing, and the memory map

## The problem

DFU 1.1 has a gap in it, and the specification is open about why. §6.1 says firmware images
are "by definition, vendor specific", so "target addresses, record sizes, and all other
information relative to supporting an upgrade are encapsulated within the firmware image
file", and the host "simply slices the firmware image file into N pieces and sends them".

That works if the device knows where everything goes. It does not work if you want to say
*put this at this address*, or *erase this page and nothing else*, or *read me back the
range from here to there*. Plain DFU has no vocabulary for addresses at all.

STMicroelectronics' extension, usually called **DfuSe**, adds one. It is worth learning
even if you never touch that silicon vendor's parts, because it is the most widely
deployed answer to "how do you bolt an address space onto DFU", and the shape of the answer
is instructive.

The authority for the command set below is ST's application note **AN3156, "USB DFU
protocol used in the STM32 bootloader"** (the copy consulted here is Rev 13). It documents
the DFU protocol of the bootloader built into the silicon, and states up front that it
"supports the DFU protocol and requests compliant with the Universal Serial Bus Device
Upgrade Specification for Device Firmware Upgrade Version 1.1, Aug 5, 2004" (§2).

## The trick: `wValue` becomes an opcode

DFU 1.1 uses `wValue` on `DFU_DNLOAD` as a block sequence number and nothing else. DfuSe
overloads it:

> In the DFU download request the command is selected through the `wValue` parameter in the
> USB request structure. If `wValue` = 0 then the data sent by the host after the request
> is a bootloader command code. The first byte is the command code and the other bytes (if
> any) are the data related to this command.
>
> — AN3156, §3

So block 0 is not data; it is a command. And `wValue` = 1 is reserved, which leaves data
blocks starting at 2. That reservation is what makes the address arithmetic work.

### Discovering what a device supports

`DFU_UPLOAD` with `wValue` = 0 is the **Get command**: the device replies with the list of
command codes it implements. AN3156 §4.2 gives the four bytes a compliant bootloader sends:

| Byte | Code | Command |
|---|---|---|
| 1 | `0x00` | Get command |
| 2 | `0x21` | Set Address Pointer |
| 3 | `0x41` | Erase |
| 4 | `0x92` | Read Unprotect |

That is the right first request to send: ask the device what it can do rather than assuming.

### The four download commands

From AN3156 §5:

| Command | Selected by | Payload | Notes |
|---|---|---|---|
| **Write memory** | `wValue` > 1 | the data | 2–2048 bytes for internal flash, RAM or system memory (§5.1) |
| **Set Address Pointer** | `wValue` = 0, first byte `0x21` | 5 bytes: `0x21` then a 32-bit address, LSB first | §5.2 |
| **Erase** | `wValue` = 0, first byte `0x41` | 5 bytes (page address, LSB first) for a page erase, or 1 byte for mass erase | §5.3 |
| **Read Unprotect** | `wValue` = 0, first byte `0x92` | 1 byte | Removes read protection — and erases flash and RAM doing it (§5.4) |
| **Leave DFU mode** | `DFU_DNLOAD` with `wLength` = 0 | none | §5.5 |

And the address arithmetic, which is the same formula for reads and writes (§4.1, §5.1):

```
Address = ((wBlockNum − 2) × wTransferSize) + Address_Pointer
```

Two things fall out of that. Blocks start at 2 because 0 is the command escape and 1 is
reserved. And because the pointer can be moved at any time, an image far larger than
`wBlockNum` could count to is transferred by re-pointing rather than by counting — the wrap
limit from chapter 3 stops being a constraint.

### Nothing happens until you ask for status

This is the rule that catches people, and AN3156 repeats it for every command:

> The Write memory operation is effectively executed only when a `DFU_GETSTATUS` request is
> issued by the host. If the status returned by the device is not `dfuDNBUSY` an error has
> occurred. A second `DFU_GETSTATUS` request is needed to check if the command has been
> correctly executed.
>
> — AN3156, §5.1; the same two sentences appear for Set Address Pointer (§5.2), Erase
> (§5.3) and Read Unprotect (§5.4)

So the pattern for every DfuSe operation is three steps, not one: send the `DFU_DNLOAD`,
send `DFU_GETSTATUS` (which triggers execution, and should answer `dfuDNBUSY`), then send
`DFU_GETSTATUS` again (which reports whether it worked). Skip the first status request and
the command is accepted and never performed.

If the address is wrong or unsupported, the device reports `dfuERROR` with `errTARGET`. If
read protection is active, it reports `dfuERROR` with `errVENDOR` (§5.1, §5.3).

⚠️ **Write protection is silent.** AN3156 §5.1 note 2: "No error is returned when
performing write operations on write-protected sectors", and §5.3 says the same for erase.
The status comes back clean and the flash is unchanged. Note this is a *different*
mechanism from read protection, which does report an error. If you do not read the flash
back and compare, a write-protected part will pass an update and run the old firmware, and
nothing in the protocol will have told you.

## The memory map lives in a string descriptor

Where does a host learn which addresses exist and what may be done to them? DFU 1.1 leaves
a hook for exactly this, in the DFU-mode interface descriptor (Table 4.4, note on
`bAlternateSetting`):

> Alternate settings can be used by an application to access additional memory segments. In
> this case, it is suggested that each alternate setting employ a string descriptor to
> indicate the target memory segment; e.g., EEPROM.

DfuSe takes the suggestion and defines a grammar for that string. The canonical description
is ST document UM0424 §4.3.2; the formulation below is the one implemented by `dfu-util`'s
parser (`src/dfuse_mem.c`), whose comment cites that section, and which is the most
easily checkable statement of it:

```
@<name>/0x<address>/<sectors>*<size><multiplier><type>[,<sectors>*<size><multiplier><type>]…
```

A typical one, as reported by a device with 128 KB of flash split into a bootloader region
and an application region:

```
@Internal Flash  /0x08000000/16*001Ka,112*001Kg
 │                │          │  │   ││
 │                │          │  │   │└─ type: what may be done to these sectors
 │                │          │  │   └── multiplier: B, K or M
 │                │          │  └────── sector size
 │                │          └───────── how many sectors
 │                └──────────────────── the address this run starts at
 └───────────────────────────────────── the region's name
```

The **type letter** is the load-bearing part, and it is not a lookup table — it is three
bits of the letter's ASCII code. `dfu-util` masks with 7 and defines the bits as
`DFUSE_READABLE = 1`, `DFUSE_ERASABLE = 2`, `DFUSE_WRITEABLE = 4`:

| Letter | ASCII | `& 7` | Readable | Erasable | Writeable |
|---|---|---|---|---|---|
| `a` | 0x61 | 1 | ✓ | | |
| `b` | 0x62 | 2 | | ✓ | |
| `c` | 0x63 | 3 | ✓ | ✓ | |
| `d` | 0x64 | 4 | | | ✓ |
| `e` | 0x65 | 5 | ✓ | | ✓ |
| `f` | 0x66 | 6 | | ✓ | ✓ |
| `g` | 0x67 | 7 | ✓ | ✓ | ✓ |

So in the example above, the first 16 KB is `a` — readable, and nothing else. That is the
bootloader.

## "The bootloader region is not writable"

This is the sentence the whole chapter has been building towards, because it is the
difference between a device you can always recover and a device you can destroy.

A region marked `a` can be read out and cannot be erased or written *through this path*.
The bootloader is protecting itself. Which means:

- A bug in your host tool that computes an address 16 KB too low does not overwrite the
  bootloader; it gets `errTARGET`.
- A truncated or corrupt application image leaves you with a broken application and an
  intact bootloader — and the bootloader is the thing that accepts new images. Retry.
- The flash is nevertheless *readable* at that address, which is why a backup taken by
  uploading the whole part comes back containing the bootloader too, and has to be trimmed
  before it can be treated as an application image.

> **Teacher's aside.** People hear "the bootloader can't be overwritten" and take it as a
> promise from the hardware. It is usually not. In this scheme it is a **policy the
> bootloader enforces about itself**, declared in a string descriptor and checked in
> firmware — the same firmware you are trusting to erase the right pages. Hardware write
> protection of the boot sector is a separate, stronger mechanism that some parts offer,
> and the two are easy to confuse because both produce "that write didn't happen". The
> distinction shows up exactly once, in the case that matters: if the bootloader itself is
> the buggy component, only the hardware mechanism still holds. Worth knowing which one is
> standing between you and a dead device.

## What DfuSe gives you, in one table

| Capability | Plain DFU 1.1 | With DfuSe |
|---|---|---|
| Send an image the device knows how to place | yes | yes |
| Write to an address the host chooses | no | Set Address Pointer + Write memory |
| Erase a specific page | no | Erase, `0x41` |
| Read a chosen range back out | only what the device chooses to offer | Set Address Pointer + Read memory |
| Learn the memory layout and its permissions | no | the interface string descriptor |
| Discover which commands exist | no | Get command |

## Check yourself

1. A host sets the address pointer, sends a write block, gets a clean status, and the
   flash is unchanged. Give two distinct explanations consistent with the protocol, and say
   how you would tell them apart.
2. Why must DfuSe data blocks start at `wBlockNum` = 2? Derive it from the two special
   values and the address formula.
3. A device reports `@Internal Flash /0x08000000/4*016Kc,1*064Kg`. How large is the region,
   at what address does the second run start, and what can you do to each run?
4. You upload the whole flash of a part whose map is `16*001Ka,112*001Kg` and save the
   result as a backup. What is wrong with writing that file straight back, and what has to
   happen first?
5. A host implementation sends Erase and then immediately sends the next Erase, checking
   nothing in between. What does AN3156 say happens, and what will the symptom look like
   to whoever debugs it?
6. Argue the case *for* a bootloader marking its own region readable rather than hiding it
   entirely — then give the argument against.
