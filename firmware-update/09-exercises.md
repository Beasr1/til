# 9. Exercises

Worked answers to every **Check yourself**, then things to try on a real device.

---

## Chapter 1 — Why firmware updates are hard

**1. The two assumptions application updates make.**

First, **that the old version survives until the new one is complete.** A package manager
unpacks new files alongside the old, verifies, and flips a pointer — nothing is destroyed
until the replacement is known good. A single-region firmware update erases the only copy
first. There is no moment where both exist.

Second, **that you can always try again.** If an installer fails you re-run it, because the
machine that runs the installer is still working. Here the thing that would run the retry
is the thing you just broke. Retry is only possible because of a *separate* component — the
bootloader — that the plan never mentioned and that may or may not exist.

A third, smaller one: application updates can be rolled back from a copy the system kept.
Nobody kept a copy here unless you arranged it (chapter 5).

**2. A clean error, and a dead device.**

The reboot already happened. The host asked the running application to restart into its
bootloader, and the application obeyed. Then something went wrong on the host side —
enumeration timed out, the wrong driver was bound, the serial number did not match — and
the host reported a tidy failure and stopped.

No flash was touched, so nothing is damaged. But the device is now in upgrade mode, which
is not a mode in which it does its job, and it will stay there until it is power-cycled or
told to leave. "Failed safely" and "left the device working" turn out to be different
claims, and only the first one was made.

**3. What "not on its own volition" forces.**

It forces the host to have a **working channel to the running device**, over which the
request to switch modes can be delivered, *before* any of the update machinery matters. If
that channel is unavailable — the device is wedged, the serial port cannot be opened, the
run-time DFU interface is not exposed — the update cannot start at all.

Note the sharp edge: a device that is already misbehaving may be exactly the one that
cannot be told to reboot. The update path depends on the thing it is trying to fix being
healthy enough to cooperate.

**4. Two kinds of bootloader.**

The difference is whether the code that accepts a new image is **inside the region being
replaced**. If the update path lives in the application, replacing the application replaces
the updater, and a failure removes your only means of retrying. If the bootloader is a
separate region that this path cannot write — DFU-mode memory maps mark such a region
read-only (chapter 4) — then the worst case is a broken application and an intact
bootloader, which is recoverable over USB, indefinitely.

The question to ask a vendor: *"Is the bootloader in a region this update can erase or
write? Show me the memory map the device reports."* The answer is in the device's own
interface string descriptor, so it is checkable rather than a matter of trust.

**5. Why "remains in DFU mode" is not enough to rely on.**

Because the specification does not say how the device decides the firmware is corrupt, and
that check is entirely the vendor's. A bootloader that validates only "is the first word a
plausible stack pointer?" will happily jump into a half-written image and hang, and then
nothing is listening on USB at all.

The spec describes the behaviour; it does not guarantee your part implements it well. The
honest position is that this is **untested until you have tested it**: flash a deliberately
truncated image to a device you can afford to lose, power-cycle it, and see whether it
comes back in DFU mode. Until then, plan as though it does not.

---

## Chapter 2 — Getting a device into a programmable state

**1. Class `FE/01/01` while running normally.**

`FEh` is Application Specific, `01h` within that is Device Firmware Upgrade. So the device
is advertising that it can be firmware-upgraded and giving you somewhere to send
`DFU_DETACH` — that is the entire purpose of the run-time interface, which has no
endpoints of its own (DFU 1.1 Table 4.1).

The `01h` protocol byte distinguishes it from `02h`, which is the DFU-*mode* interface
(Table 4.4). Same class and subclass, different position in the lifecycle: `01` means "I
am doing my normal job and can be asked to switch", `02` means "I have switched, send me
firmware".

**2. `bitWillDetach` is set.**

Send `DFU_DETACH`, then **do nothing**. Specifically, do not issue a USB reset: the device
generates the detach–attach sequence on the bus itself, and DFU 1.1 §4.1.3 says the host
must not issue one. Wait for the device to reappear and re-enumerate it.

The opposite case is the one that catches people. With the bit clear, the device starts a
timer of `wDetachTimeOut` milliseconds and *needs* the host's reset; if it does not arrive
in time the device returns to `appIDLE` and, per Appendix A.2.2, "a subsequent USB reset
will not initiate DFU". You get a device that looks like it ignored you.

**3. Failing because no acknowledgement arrived.**

The device is resetting in the middle of composing the reply. There is no reason to expect
one, and treating its absence as failure turns a successful reboot into a reported error —
after which the tool typically gives up, leaving the device in the bootloader (question 2
of chapter 1).

The correct test is not "did it answer" but "did it come back as the bootloader". Send the
command, ignore a missing reply, and poll for the bootloader's appearance against a
deadline.

**4. Why `idProduct` changes, and what it breaks.**

The spec's reason (§2) is driver binding: "certain operating systems may use only the
vendor and product IDs reported by a device to determine which drivers to load, regardless
of the device class code". Changing `idProduct` guarantees the DFU driver is selected
rather than whatever driver the normal-mode device attracts.

What it breaks is any tool holding `VID:PID` as identity. The device it is tracking
vanishes and an unrelated one appears; there is no connection between them visible in those
two fields. And even if the PID were stable, `VID:PID` identifies a *model*, so two of the
same device on one machine are indistinguishable by it.

**5. Two identical devices, no serial number.**

The tool should **refuse** and say why. There is no per-unit identity available, so
"the one the user meant" is not a question the tool can answer, and the wrong answer is
irreversible — you cannot un-flash a device.

The cost of refusing is a worse experience in a genuinely rare configuration, which the
user can resolve by unplugging one device. The cost of guessing is flashing the wrong unit
with an image intended for another, which may be a different hardware revision. That
asymmetry settles it. Refusing is also the only behaviour you can *test*; "picks the right
one" has no test.

**6. Finding the device by port name.**

First reason: the port name is assigned by the OS at enumeration and is not stable across
one. The bootloader may get a different name — and this is the reason that would survive
even if names were stable, which is the second: **the bootloader is very often not a serial
device at all.** It exposes a DFU interface over the control pipe, with no CDC ACM
interface and therefore no port to open. There is no name to reopen, stable or not.

(A third, on some platforms: one physical device can present as two nodes, so "the port"
was already ambiguous before anything rebooted.)

---

## Chapter 3 — The DFU state machine

**1. Three blocks, request by request.**

```
dfuIDLE
  DFU_DNLOAD (wBlockNum=0, wLength=N)   → dfuDNLOAD-SYNC
  DFU_GETSTATUS                          → dfuDNBUSY, wait bwPollTimeout
  DFU_GETSTATUS                          → dfuDNLOAD-IDLE          (block 1 done)
  DFU_DNLOAD (wBlockNum=1, wLength=N)   → dfuDNLOAD-SYNC
  DFU_GETSTATUS                          → dfuDNBUSY, wait
  DFU_GETSTATUS                          → dfuDNLOAD-IDLE          (block 2 done)
  DFU_DNLOAD (wBlockNum=2, wLength=N)   → dfuDNLOAD-SYNC
  DFU_GETSTATUS                          → dfuDNBUSY, wait
  DFU_GETSTATUS                          → dfuDNLOAD-IDLE          (block 3 done)
  DFU_DNLOAD (wLength=0)                → dfuMANIFEST-SYNC
  DFU_GETSTATUS                          → dfuMANIFEST
```

Ten `DFU_GETSTATUS` requests if the device reports busy once per block — and it may report
busy more than once, in which case the `dfuDNBUSY → dfuDNLOAD-SYNC → dfuDNBUSY` loop
repeats. The count is not fixed; the *pattern* is: poll until the state stops being 4.

**2. `bState = 4`, then an immediate `DFU_GETSTATUS`.**

State 4 is `dfuDNBUSY`, and Appendix A.2.5 is unambiguous: "Receipt of any DFU
class-specific request → Device stalls the control pipe → `dfuERROR`". So the device stalls
and lands in state 10.

The host must now send `DFU_CLRSTATUS`, which is the only exit from `dfuERROR` (§6.1.3),
and it returns the device to `dfuIDLE` — **not** to where the download was. The download is
over; it has to be restarted. That is a real cost for what looks like an innocuous extra
poll, and it is what `bwPollTimeout` exists to prevent.

**3. A STALL partway through.**

Because in DFU a stall is a *notification*, not a result. §6.1: "If the device detects an
error, it signals the host by issuing a STALL handshake on the control endpoint. The host
then sends a DFU class-specific request, called `DFU_GETSTATUS`, on the control endpoint to
determine the nature of the problem."

So you send `DFU_GETSTATUS` and read `bStatus`, which distinguishes cases that need
completely different responses — `errWRITE` and `errERASE` say the flash is unhappy;
`errTARGET` and `errADDRESS` say your request was wrong; `errVERIFY` says it wrote
something other than what you sent. A tool that reports "stalled" throws all of that away.
Then clear the error with `DFU_CLRSTATUS` before doing anything else.

**4. `bitCanUpload = 0`.**

You cannot read the existing firmware out, so:

- **No backup.** The host-as-second-bank strategy of chapter 5 is unavailable, and with it
  rollback — there is nothing to roll back *to*.
- **No read-back verification.** You cannot compare what is on the device with what you
  sent, so you are trusting status words, which chapter 4 shows can be clean while the
  flash is unchanged.

What you do instead: get the previous image from the vendor and keep it, so a rollback is
at least possible from a known-good file (accepting that it is the released image, not
*this unit's* image, which differs if anything per-device was ever written). Failing that,
be much more conservative about when you update at all — schedule it where physical
recovery is possible, and verify the device's function afterwards, since that is the only
check you have left.

**5. `bitManifestationTolerant = 0`, `bitWillDetach = 0`.**

After the zero-length `DFU_DNLOAD` and its `DFU_GETSTATUS`, the device enters
`dfuMANIFEST`, completes programming, and moves to `dfuMANIFEST-WAIT-RESET` (state 8). With
`bitWillDetach` clear it will not detach itself, so **the host must issue a USB reset**
(§7).

If it doesn't, the device sits in state 8 indefinitely. The symptom is a device that has
been written successfully and never comes back — an update that reports "writing complete"
and then hangs on "waiting for device". Easy to misdiagnose as a slow or failed write when
the write was fine and nobody told it to restart.

**6. The `wBlockNum` wrap.**

65,536 blocks × 1024 bytes = 64 MiB before the counter wraps. Far beyond the flash of the
microcontrollers this class is usually applied to, so it is rarely a practical limit.

It may not be a limit at all, because an extension can give `wBlockNum` a different
meaning. In DfuSe (chapter 4) the block number is not a running counter but an *index into
a window*: the address is `((wBlockNum − 2) × wTransferSize) + Address_Pointer`, and the
pointer can be moved between transfers. The host re-points instead of counting on, so the
wrap never arrives.

---

## Chapter 4 — DfuSe

**1. Clean status, unchanged flash.**

Explanation one, and the likely one: **the sector is write-protected.** AN3156 §5.1 note 2
says "No error is returned when performing write operations on write-protected sectors",
and §5.3 says the same for erase. The device accepted the request, declined to act, and
reported success.

Explanation two: **the command never executed.** DfuSe commands run only when
`DFU_GETSTATUS` is issued, and the host must see `dfuDNBUSY` and then poll again. A host
that skips the status request gets a clean result from a command that was never performed.

Telling them apart: look at what the device reported. If you never saw `dfuDNBUSY`, it is
the second. If you did, and a second status said OK, it is the first — confirm by trying to
*erase* the same page and reading it back; a write-protected sector comes back with its
original contents.

**2. Why data blocks start at 2.**

`wValue` = 0 is the command escape: "If `wValue` = 0 then the data sent by the host after
the request is a bootloader command code" (AN3156 §3). `wValue` = 1 is reserved. So the
first value available for data is 2.

The address formula then has to subtract that offset so the first data block lands on the
pointer itself: `Address = ((wBlockNum − 2) × wTransferSize) + Address_Pointer`. With
`wBlockNum` = 2, the address is exactly `Address_Pointer`. The `− 2` is not arbitrary; it
is the two reserved values.

**3. `@Internal Flash /0x08000000/4*016Kc,1*064Kg`.**

Four sectors of 16 KB = 64 KB, then one of 64 KB. Total 128 KB.

The second run starts where the first ends: `0x08000000 + 4 × 16384 = 0x08010000`.

Types: `c` is 0x63, `& 7` = 3 = readable + erasable — so the first 64 KB can be read and
erased but **not written**. `g` is 0x67, `& 7` = 7 = readable + erasable + writeable. So an
image intended for `0x08000000` cannot be written through this path at all; only the region
from `0x08010000` is a valid target.

**4. Writing a full-flash upload straight back.**

The uploaded file contains the bootloader as well as the application, because the
bootloader's region is marked readable (`a`) and the upload returns the whole range. Writing
it back would attempt to write the bootloader region — which the device will refuse
(`errTARGET`), or, worse on a part with different protections, might not.

Before it is a backup it has to be **trimmed to the application image** — cut to the size
the image's own header declares, or to the bounds of the writeable region from the memory
map — and then **re-parsed and validated** as if it were an incoming image (chapter 5). A
file that will not parse is not a rollback; it is a file.

**5. Erase, then immediately Erase again.**

The first Erase was accepted but never executed, because AN3156 §5.3 says it "is
effectively executed only when a `DFU_GETSTATUS` request is issued by the host". So the
page was not erased. The second Erase overwrites the pending command with another pending
command.

The symptom: the update reports success on every step and the device runs the old firmware,
or runs a mixture — pages that happened to be erased by some other path hold new data and
the rest hold old. If verification is also skipped, this ships. If verification is present,
it fails with a mismatch at an offset that makes no sense until someone notices the missing
status requests.

**6. Bootloader region readable — for and against.**

**For:** it makes a full-image backup possible with a single upload and no special
handling, it lets tooling read and report the bootloader's version, and it lets you prove
after an incident that the bootloader was not touched. It also costs nothing to implement:
read is the default state of flash.

**Against:** it hands anyone with USB access a copy of the bootloader, which is the code
that enforces every protection in this chapter. Reading it is the first step in finding a
flaw in it, and if the bootloader ever verifies signatures, its logic and any embedded
public key are now public.

The usual resolution is that the bootloader is not secret and should not be relied on to be
— its protections should hold against an attacker who has read it. If they do not, marking
the region unreadable buys delay, not safety.

---

## Chapter 5 — One program region, or two

**1. Power lost at the equivalent point.**

Dual-bank, power lost after the pointer write and before the reset: the pointer is in
nonvolatile memory, so on the next power-up the device boots bank B and runs the new
firmware. The update completed; the reset merely happened later than intended. Both images
are intact either way.

Single-bank, at the equivalent point — that is, at the commit — the equivalent point is
*the first erase*, and power lost there leaves the device with a partially erased region
and no program. It reboots into the bootloader, if there is one. The two designs are not
the same operation with different risk; the irreversible moment is at opposite ends.

**2. A written file that is not a backup.**

One: **it contains more than the image.** An upload returns the range the device chose,
typically the whole region, bootloader included. Written straight back it targets addresses
the device will refuse. The file exists and is useless.

Two: **it is truncated.** The upload ended early — a stall, a short read treated as EOF, a
device that stopped responding. You have a file of plausible size that decodes to nothing
coherent, and you will find out at the moment you need it.

Both pass "did the file get written?". Neither passes "parse this file with the same
validator you use on incoming images", which is why that check has to run before the erase.

**3. The commit point.**

It is immediately before the first erase — the last instant at which the device still holds
a complete working program.

Before it: honour cancellation, unwind, leave the device as you found it (including telling
the bootloader to leave, per chapter 2). After it: the operation runs to completion and
reports what happened — succeeded, rolled back, or failed. There is no third answer that is
true, so the cancel affordance should be gone.

Checking only at the start is not enough because the interval between start and commit can
be *seconds*, dominated by waiting for the bootloader to enumerate. A user who clicks
cancel during that wait has made a perfectly reasonable request that the tool could honour
for free, and a tool that only looked at the beginning will erase anyway. Re-check
immediately before committing.

**4. Success everywhere, old firmware running.**

Write protection. AN3156 §5.1 note 2: no error is returned for a write to a write-protected
sector. Every status is clean and the flash is unchanged.

The step that catches it is **read-back verification** — uploading the written range and
comparing byte for byte. Nothing at the protocol level distinguishes this case; only the
actual contents do. (A secondary catch is the post-update function check, if the old
firmware is broken in a way the check exercises — but that is luck, not a control.)

**5. Which error to show.**

Show the **original** failure: the one that explains what happened. "Verification failed at
offset 0x3C00" tells the recipient what went wrong; "rollback failed" tells them only that
a recovery they did not know had started did not work.

The rollback failure must not be discarded — it changes what the device's state is, and
therefore what the operator should do next. Log it, and surface it as a secondary fact
("the previous firmware could not be restored") alongside the headline error. What you must
not do is let it *replace* the original, which is the easy bug: an error path that returns
the last error it saw.

**6. Counting endpoints instead of comparing them.**

A device that exposes two functional endpoints loses one when the update breaks it, and at
the same moment an unrelated device is attached — or a second, similar device that was
already present is re-enumerated. Count before: 2. Count after: 2. Check passes; the device
is broken.

Compare the **set of identities** — names, or per-endpoint identifiers — against a baseline
captured before the update, and report which specific ones are missing. That also makes the
failure message useful instead of "health check failed".

And keep a third state: if the subsystem you check through is itself unavailable, that is
"cannot tell", not "broken". Rolling back a good update because a host service was not
running is a self-inflicted outage.

---

## Chapter 6 — Integrity is not authenticity

**1. Magic, length, masked CRC32.**

They build the image they want, set the magic (public — it is in every released file),
set the length to the real length, compute CRC-32 over the payload, XOR it with the
constant, and write it in. The container is valid.

Cost: finding the constant, which is one XOR of the stored word against a CRC they compute
themselves over one legitimate image, confirmed against a second. An afternoon, once, and
it is then worth nothing forever — including to everyone they tell. There is no key, so
there is nothing they had to steal.

**2. Restating the DFU suffix guarantee for a reviewer.**

*Established:* the file is complete and uncorrupted with respect to the CRC-32 stored in
it, and it is a file of the DFU format (the three magic bytes are present); and, where
`idVendor` / `idProduct` are not `FFFFh`, it declares itself intended for this vendor and
product.

*Not established:* who produced the file, whether they were authorised to, whether the
declared vendor and product are true, or whether the contents are the firmware anyone
intended. Both the CRC and the magic bytes are functions of public information that any
holder of the file can compute, so they distinguish a *corrupted* file from an intact one
and cannot distinguish a *hostile* file from a genuine one.

The specification's own word is "intact", and it is the right one.

**3. Host-side signature, device accepts any valid CRC.**

**Stops:** someone who tampers with the image file after we published it — a compromised
mirror, a modified file on the customer's disk, an operator who substituted a build. Our
tool refuses to send it. That is a real and common threat and worth the control.

**Does not stop:** anyone who runs *different* software against the device. The
manufacturer's own flashing utility, `dfu-util`, fifty lines of libusb — none of them are
bound by our policy. Physical access, or any privileged process on the machine, is enough.

Which a reviewer asks about first: the second. "What stops someone flashing this device
with something else entirely?" is the question, and the honest answer is "nothing at the
device". Lead with that rather than being led to it.

**4. `idVendor` / `idProduct` = `FFFFh`.**

`FFFFh` means "do not enforce this match" — the suffix carries no vendor or product
constraint, and the file may be sent to any device. The spec's stated reason is to keep a
standard file format, or to carry a `bcdDevice` version for information, without imposing a
match.

For that to be safe, the targeting has to come from somewhere else: the distribution
channel only ever offering a given image for a given model, the update tool checking the
attached device against a manifest, or the device itself rejecting images it cannot run. If
nothing does any of those, a wildcard suffix means the only thing standing between a device
and an image built for a different product is the operator picking the right file.

**5. Moving the version field.**

Because nothing about the field's *location* involves a secret. An attacker reads two
released images, diffs them, sees which bytes track the version, and edits those. The
format's layout is information they can derive from public artefacts, so hiding it changes
their effort and not their capability — which is the whole integrity/authenticity
distinction restated: a check anyone can recompute protects against accident, not intent.

The one thing it genuinely costs: **time, once, for the first person.** That is not nothing
— it may deter casual meddling — but it is a speed bump, it never regenerates, and it
should never appear in a risk assessment as a control.

**6. The bootloader verifies a signature.**

The public key has to be somewhere the bootloader can read and an attacker cannot change —
in practice, inside the bootloader image itself, in a locked region of flash, or in
one-time-programmable fuses. If it lives anywhere writable through the update path, the
attacker replaces the key and then signs whatever they like.

The new question: **key management.** Who holds the private key, where does the signing
happen, what happens when someone leaves, and — the hard one — what is the plan if the key
is compromised or simply lost? A device that will only accept images signed by a key you no
longer have is a device that can never be updated again. Rotation has to be designed in
before the first unit ships, because after that it is a firmware update away, and the
firmware update is the thing that needs the key.

---

## Chapter 7 — The Windows driver problem

**1. Why Windows needs a driver and Linux does not, in the same way.**

On Linux, the kernel's USB core exposes every device through usbfs, and a user-mode process
can claim an interface that no kernel driver has bound and issue control transfers directly.
The permission question is a filesystem one. macOS is similar in spirit through IOKit.

On Windows, user-mode code has no path to a USB interface except through a kernel-mode
function driver that has been bound to it, and there is no generic "let me at it" device
node. So a device whose interface class has no in-box driver is genuinely unreachable until
one is bound — which is the whole of this chapter's problem.

**2. Binds on a fresh machine, not on the developer's.**

The developer's machine has met that device before, when its firmware had no MS OS
descriptor. Windows asked for string `0xEE` once, got no valid answer, and cached that fact
in `HKLM\SYSTEM\CurrentControlSet\Control\UsbFlags\<vvvvpppprrrrr>` under `osvc`. Microsoft's
documentation is explicit: after an invalid first response "the operating system makes no
further requests for that descriptor".

The fix is to clear that cached entry (delete the `UsbFlags` key for that VID/PID/revision)
and re-attach the device so Windows probes again. Which is also a warning for the field: a
firmware fix that adds WCID support will not take effect on machines that already know the
device, so the rollout needs to account for that rather than assuming new firmware means
new behaviour.

**3. What normal-mode WCID tells you about the bootloader.**

Almost nothing. The cache key includes the product id, and the bootloader has a *different*
product id — DFU 1.1 recommends exactly that (§2). It is a different USB device with its
own descriptors, its own firmware, and its own answer to the `0xEE` probe.

In practice the two are often written by different people at different times: the
application is the vendor's own code, the bootloader may be the silicon vendor's stock
one. Probe the bootloader itself — put the device into DFU mode and request string `0xEE`
from it — and treat the result as the only evidence.

**4. "We already sign our installer with an EV certificate."**

An Authenticode signature on an executable and a signed driver package are different
artefacts under different policies. Microsoft's driver signing policy: "Starting with
Windows 10, version 1607, Windows will not load any new kernel-mode drivers which are not
signed by the Dev Portal." Your signature is not the one Windows is looking for — the
package must be *submitted to Microsoft* and returned bearing Microsoft's signature.

The EV certificate is not the deliverable either; it is the entry requirement: "an EV code
signing certificate is required to establish a dashboard account." Having one means you can
open the account. You still need the account, a submission per package revision, and either
Hardware Lab Kit logs or the lighter attestation-signing route.

**5. An INF matching on vendor id alone.**

It binds WinUSB to *every* device from that vendor, including the reader in its normal
mode. That device then has a generic USB driver where it needed its class driver, so it
enumerates fine, appears in Device Manager without a warning, and simply does not do its
job — it is not recognised as the kind of device it is, and the software that looks for it
finds nothing.

Nobody debugs that as a driver problem, because the driver installed successfully and the
device is present. It reads as "the device stopped being detected after we shipped the
update tool", and the connection to an INF that was only supposed to matter during firmware
updates is several hours away.

**6. A non-WCID bootloader, on machines you do not administer.**

| Option | Cost |
|---|---|
| **Fix the firmware** to answer the `0xEE` probe | The right answer, and circular: it needs a firmware update on devices that cannot currently be updated on Windows. Viable for future production, not for the installed base |
| **Ship a signed driver package** in your installer | EV certificate, Dev Center account, a submission per revision, an installer that runs elevated, and cleanup logic when the package changes. Real but bounded |
| **Ask the user to bind the driver by hand** | Free for you, unacceptable for anyone else, and the failure mode is a half-updated fleet |
| **Do not support Windows updates** | Honest, and sometimes right if the fleet is small — but the device is still left in its bootloader when someone tries, so you must at minimum detect Windows and refuse *before* rebooting the device |

Recommend the second for the installed base and the first for everything built afterwards,
and treat them as parallel rather than alternatives. Whichever you pick, add the last
sentence of option four: refuse early, before the reboot, rather than discovering the
problem when the device is already in a mode it cannot leave.

---

## Things to try on a real device

In rough order of risk. Everything in the first group writes nothing.

**Observe**

1. Enumerate a device and dump its descriptors. Find `iSerialNumber` and read the string.
   Do it for two units of the same model and confirm they differ.
2. Look for a run-time DFU interface (class `FE`, subclass `01`, protocol `01`) and, if
   present, decode the functional descriptor's `bmAttributes` bit by bit. Predict whether
   the host or the device will handle the detach.
3. Request string descriptor index `0xEE` and see whether you get a Microsoft OS string
   descriptor or a stall. Do it for the normal-mode device *and* the bootloader.

**Explore — no writes**

4. Put a device into DFU mode and read its interface string descriptor. Parse the memory
   map by hand: work out every region's address, size and permission letter, and predict
   which addresses a write would be refused at.
5. Send `DFU_GETSTATUS` and decode all six bytes, including `bwPollTimeout`. Then send
   `DFU_GETSTATE` and check the two agree.
6. Deliberately send a request that has no transition from the current state — say
   `DFU_CLRSTATUS` from `dfuDNLOAD-SYNC`. Confirm the stall, confirm `bState` becomes 10,
   and confirm nothing else works until you clear it.
7. Upload the whole readable region and look at it in a hex editor. Find the boundary
   between the bootloader and the application. Find the image's own header if it has one.
8. Erase one page of a *scratch* region if the part has one, then read it back and record
   what erased flash actually reads as on this part. Do not assume `0xFF`.

**Reason about, don't run**

9. For a device in front of you, write down where the commit point would be and what your
   tool would do on each side of it.
10. Work out what an attacker with physical access could write to this device, and which of
    your controls — if any — would be in their way. Write the honest sentence from chapter 6
    for your own product.
11. Identify every wait in your update path and what happens if it expires. For each one,
    say what state the device is left in and whether anything puts it back.

## Questions worth asking me

- "Here's a device's descriptor dump — what update path does it support, and what will it
  cost me?"
- "Decode this memory-map string and tell me where I'm allowed to write."
- "Walk me through what happens if power is lost at each step of this sequence."
- "My update reports success and the device runs the old firmware. Where do I look?"
- "What exactly does this container's checksum prove, and what would I have to add to make
  it prove authorship?"
- "This has to work on Windows machines we don't administer. What are my options, in order
  of how much they cost?"
- "Which of these waits should be a constant, and which should come from the device?"
