# Firmware Update Over USB — A Course

How you replace the program running inside a USB device, from persuading it to stop being
that device to proving the new code is actually on it — and what the standards do and do
not promise you along the way.

**This is reference learning material.** It is about the standards — USB DFU 1.1, one
silicon vendor's DfuSe extension, USB CDC, Microsoft's OS descriptors and driver signing
policy — not about any particular device or product. Nothing here is project-specific.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- *Why* before *how*: each chapter opens with the problem that made the design necessary.
- Every file ends with **Check yourself** questions. Answers are in
  [`09-exercises.md`](09-exercises.md).
- Where a measurement appears, it is labelled as one observed device. None of them are
  specification figures.

## The primary sources

Everything here is checkable against one of these. Where I say "the standard says", this
is which.

| Source | Covers |
|---|---|
| **USB Device Firmware Upgrade specification, revision 1.1** (USB-IF, 5 Aug 2004) — [PDF](https://www.usb.org/sites/default/files/DFU_1.1.pdf) | The class: four phases, the seven requests, the eleven states, the functional descriptor, status polling, upload and download, the file suffix. Chapters 1–3, 6 |
| **AN3156, "USB DFU protocol used in the STM32 bootloader"** (STMicroelectronics) — [PDF](https://www.st.com/resource/en/application_note/cd00264379-usb-dfu-protocol-used-in-the-stm32-bootloader-stmicroelectronics.pdf) | The DfuSe command set layered on DFU: Get, Set Address Pointer `0x21`, Erase `0x41`, Read Unprotect `0x92`, Leave; the address formula; what write protection does and does not report. Chapter 4 |
| **UM0424 §4.3.2** (STMicroelectronics) | The memory-map interface string syntax. See the note below — I worked from `dfu-util`'s parser, which cites this section |
| **`dfu-util`**, `src/dfuse_mem.c` / `dfuse_mem.h` — [project](https://dfu-util.sourceforge.net/) | The reference open-source host implementation. The memory-map grammar and the readable/erasable/writeable bit values in chapter 4 are read off this code |
| **USB-IF defined class codes** — [usb.org](https://www.usb.org/defined-class-codes) | `02h` Communications and CDC Control, `0Ah` CDC-Data, `FEh` Application Specific (which is DFU's class). Chapters 2, 7 |
| **USB CDC and its PSTN subclass** (USB-IF) | The virtual serial port a device commonly uses to receive a vendor reboot command. Chapter 2 |
| **Microsoft OS Descriptors for USB Devices**, **Microsoft OS 1.0 Descriptors Specification**, **WinUSB Device** — [Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/microsoft-defined-usb-descriptors) | String index `0xEE`, `bMS_VendorCode`, the `WINUSB` compatible ID, the `osvc` cache. Chapter 7 |
| **Driver Signing Policy** — [Microsoft Learn](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/kernel-mode-code-signing-policy--windows-vista-and-later-) | The Windows 10 1607 rule, EV certificates, attestation signing, and the three documented exceptions. Chapter 7 |

DFU 1.1 is free from usb.org. ST's application notes are free from st.com. Microsoft's
pages are public. The one source I could not obtain directly is **UM0424**, so chapter 4's
memory-map grammar is stated as `dfu-util` implements it, with its own citation to that
section noted — treat the syntax as well-evidenced and the section reference as
second-hand. Likewise, DfuSe is documented here for one silicon vendor's bootloader;
whether a particular part behaves identically is a question for its own reference manual.

## Reading order

Read 1–3 in order. They build. Chapters 4–7 can be taken in any order after that, though
5 reads better after 4.

| # | File | After this you can… |
|---|---|---|
| 1 | [Why firmware updates are hard](01-why-firmware-updates-are-hard.md) | Say what makes this different from every other kind of update, and ask the one question that decides how bad a failure can get |
| 2 | [Getting a device into a programmable state](02-getting-into-a-programmable-state.md) | Get a device into its bootloader by either route, and hold on to its identity across a re-enumeration that changes almost everything else |
| 3 | [The DFU state machine](03-the-dfu-state-machine.md) | Drive a download or an upload by hand, read a status reply byte by byte, and diagnose a stuck device from its state number |
| 4 | [DfuSe and the memory map](04-dfuse-and-the-memory-map.md) | Address a specific region, erase a page, decode a device's memory map, and explain what "the bootloader region is not writable" really rests on |
| 5 | [One program region, or two](05-single-bank-and-dual-bank.md) | Design an update that survives a power cut on hardware that has nowhere to stage anything — and say exactly what that costs |
| 6 | [Integrity is not authenticity](06-integrity-is-not-authenticity.md) | State precisely what a checksum proves, where the trust boundary really sits, and what you would have to add to move it |
| 7 | [The Windows driver problem](07-the-windows-driver-problem.md) | Explain why a device that works on Linux is unreachable on Windows, price the ways out, and separate what the signing policy actually governs from what people think it does |

Reference, not reading:

| # | File | |
|---|---|---|
| 8 | [Glossary](08-glossary.md) | Every term, request, state and status code, in one place |
| 9 | [Exercises](09-exercises.md) | Worked answers to every **Check yourself**, plus things to try |

### If you're short on time

| You want | Read |
|---|---|
| To decide whether a device can be updated safely at all | 1, then 5 |
| To debug an update that hangs or reports nonsense | 3, then 4 §nothing happens until you ask for status |
| To answer "is this update secure?" in a review | 6, then the last two rows of its trust-boundary table |
| To work out why it works on your bench and not on a customer's Windows machine | 7, then 2 §what do you hold on to |
| The whole model in twenty minutes | 1, then the summary below |

## The one-paragraph summary of everything

Replacing a device's firmware means the device must cooperate in its own replacement, and
it cannot decide to do so by itself — **USB DFU 1.1** is a protocol for asking, and it
says outright that "a printer is *not* a printer while it is undergoing a firmware
upgrade". The device reboots into a bootloader that is, deliberately, a **different USB
device** with a different product id, so the only identity that survives the transition is
the unit's **serial number**; a port name and a `VID:PID` both break, the first because the
OS reassigns it and the second because the spec tells vendors to change it. Once there, the
protocol is **host-driven and closed-loop**: every block is followed by `DFU_GETSTATUS`,
whose reply carries the device's state and a `bwPollTimeout` telling you how long to wait,
and almost every bug is a host that went on without asking. Plain DFU has no notion of an
address, so extensions like **DfuSe** overload `wValue` to carry commands — set address
`0x21`, erase `0x41` — and publish a **memory map in an interface string descriptor** whose
type letters are three permission bits, which is where "the bootloader region is not
writable" actually lives: as a policy the bootloader declares about itself, not a guarantee
from the silicon. Whether a failed write is survivable is decided by the hardware: **dual
bank** keeps the running image intact until a pointer flips, while **single bank** erases
the only copy, so the host must become the second bank by uploading the old image first —
and a backup you have not re-parsed is not a backup. Verify by **reading the flash back**,
because at least one bootloader returns a clean status for a write to a protected sector
and changes nothing. Finally, none of this is security: the DFU suffix's CRC lets a host
"presume that the firmware upgrade file is **intact**", the field called `ucDfuSignature`
is three magic bytes and not a cryptographic signature, and the whole 2004 specification
never once uses the words *secure*, *authentic* or *tamper* — so when the format has no
signature the trust boundary moves entirely to the channel, the manifest and the privilege
of whoever can call the updater. On Windows, add one more obstacle: the DFU interface is
vendor-class, there is no in-box driver, and either the firmware asks for WinUSB through
the **`0xEE` Microsoft OS descriptor** — free, silent, and cached forever including the
failures — or you ship a **driver package**, which needs Microsoft's signature, an EV
certificate to get an account, and a submission per revision.

## How to use me

These notes are a starting point. Good questions to bring back:

- "Here's a descriptor dump — what update path does this device support?"
- "Decode this memory-map string and tell me which addresses I can write."
- "Walk me through what happens if power is lost at each step of this sequence."
- "My update reports success and the device is running the old firmware. Where do I look?"
- "What does this container's checksum actually prove, and what would make it prove more?"
- "This has to work on Windows machines we don't administer — what are my options, in
  order of cost?"
- "Which of these timeouts should be a constant, and which should come from the device?"
- "Where is the commit point in this design, and what does cancel mean on each side of it?"
