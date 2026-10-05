# 5. One program region, or two

## The problem

Everything so far has assumed that erasing flash and writing it again is a thing you simply
do. This chapter is about the gap between the erase and the write, and about the fact that
whether that gap is dangerous is a property of the *hardware*, decided long before anyone
wrote an updater.

Two devices can implement identical DFU protocols and have completely different risk
profiles, because one has somewhere to put the new image while keeping the old one and the
other does not.

## Dual-bank: the update that is never partially applied

Some parts have two program regions and can be told which one to boot from. Then the
update is:

```
   bank A (running)        bank B (spare)
 ┌──────────────────┐    ┌──────────────────┐
 │  v1  — running   │    │  old / garbage   │   1. write the new image into B
 │                  │ ─▶ │  v2  — written   │   2. read B back, verify
 │                  │    │  v2  — verified  │   3. flip the boot pointer
 └──────────────────┘    └──────────────────┘   4. reset
        still intact throughout every step
```

The property that matters is that **the running image is never destroyed**. If power is
lost at any point before the pointer flips, the device reboots into bank A and runs v1. The
update has not half-happened; it has not happened. The commit is one small atomic write of
a pointer, not a minutes-long erase-and-write of a whole region.

This is the same design as A/B partitions in operating-system updates, and for the same
reason: make the risky part long and reversible, and the irreversible part short.

## Single-bank: erase first, and hope

Many small microcontrollers have one program region. There is nowhere to stage anything.
The sequence is:

```
 ┌──────────────────┐
 │  v1  — running   │   1. erase  ──▶ ┌──────────────────┐
 └──────────────────┘                 │     nothing      │   ← the dangerous window
                                      └──────────────────┘
                                          2. write  ──▶  ┌──────────────────┐
                                                         │  v2 — unverified │
                                                         └──────────────────┘
```

Between step 1 and the end of step 2 the device holds no runnable program. A power cut, a
pulled cable, a host that crashes, a bad image — all of them land in the same place.

What survives that window is a separate bootloader, unwritable through this path
(chapter 4). It is why the failure is "the device does not do its job" rather than "the
device is gone". It is not a small distinction, but it is not a fallback either: the
bootloader can accept a new image, and it has no idea what the old one was.

## So the host has to be the second bank

If the device cannot keep a copy of the old image, someone has to, and the only candidate
is the host. That is what `DFU_UPLOAD` is for, and it is the reason chapter 3 called upload
the quietly important half of the protocol.

The shape of a safe single-bank update, then:

| Phase | What happens | If it fails |
|---|---|---|
| 1. Validate | Parse and check the new image *before* touching anything | Nothing has been touched |
| 2. Back up | Upload the current image; save it; **re-validate what you saved** | Nothing has been touched |
| 3. *(commit point)* | The last moment at which stopping is free | — |
| 4. Erase + write | The dangerous window | → roll back |
| 5. Verify | Upload the written range and compare byte for byte | → roll back |
| 6. Check it works | Confirm the device does its actual job again | → roll back |
| 7. Roll back | Write the backup image back | If this also fails, report the *original* error |

Several things in that table are not obvious. Taking them one at a time.

### The backup has to be validated as a backup

A file you wrote to disk is not a backup. A backup is something you have established you
could put *back*. The cheapest way to establish that is to run the uploaded bytes through
exactly the same parser and validity check you apply to an incoming image, and to fail the
update — before erasing anything — if it does not pass.

This catches real things:

- **The upload returns more than you wanted.** An upload typically hands back the whole
  region the device chose to expose, which may include a bootloader you must not write
  back. The bytes have to be trimmed to the actual image before they are anything.
- **The upload was short or stalled.** A truncated file looks like a file.
- **Your parser and the device disagree about what an image is.** Better to discover that
  while the device still works.

⚠️ Do not assume you know what erased flash reads back as. The universal expectation is
`0xFF`, and on one part observed in practice erased flash read back as `0x00` instead — so
a blank-check written against the usual constant silently classified real data as empty.
Measure it on the part you have, by erasing a page and reading it, rather than importing
a constant from another family.

### There is a commit point, and you should name it

Before the first erase, "cancel" is honest: nothing has changed, the device still works,
you can stop. After the first erase, there is no state to return to, and a cancel button
that claims otherwise is lying about the hardware.

So model it explicitly. Up to the commit point, honour cancellation. After it, the
operation runs to completion and reports what actually happened — success, rolled back, or
failed. The user interface may well keep showing a progress bar, but the cancel affordance
should be gone, because there is no longer an answer it could give that would be true.

This matters more than it sounds because the wait to enter the bootloader can be long
(chapter 2: seconds, variable). A cancel arriving during that wait is perfectly legitimate
and must be honoured; the same cancel arriving forty milliseconds later must not be. Check
for it immediately before committing, not only when the operation began.

### Verify by reading back, not by reading a status word

Chapter 4 supplied the reason: at least one widely deployed bootloader returns a clean
status for a write to a write-protected sector and does nothing. A protocol-level success
means the device accepted the request. It does not mean the flash changed.

The only check that actually establishes the new image is on the device is to read the
written range back and compare it byte for byte with what you sent. Log the offset of the
first mismatch when it fails; "verification failed" alone tells the next person nothing,
while "first mismatch at offset 0x3C00" distinguishes a bad page from a wrong address from
an off-by-one in the length.

### "The device came back" is not "the device works"

The USB device re-enumerating proves the microcontroller is running *something*. It does
not prove the thing it is running does its job. A reader whose card-reading firmware is
broken can still appear on the bus.

So the final check should exercise the function, not the presence — and it should compare
against a baseline taken *before* the update, not against an absolute. Two rules that fall
out of doing this for real:

- **Compare identities, not counts.** If the device provides two functional endpoints and
  the machine has other similar devices attached, a count that stays the same hides one
  disappearing and something else arriving. Compare the set of names or identifiers.
- **"I cannot tell" is not "it is broken".** If the host subsystem you check through is
  itself unavailable — the service is not running, permissions are missing — that is an
  absence of evidence, not evidence of failure. Distinguish the two, or you will roll back
  perfectly good updates on machines that were merely configured differently.

### Rollback is best-effort, and the first error is the one to report

If the write or the verify fails, the recovery is to flash the backup. Two judgements
worth making in advance.

Rollback generally should *not* re-verify as strictly as the update does. Getting the old
firmware onto the part matters more than proving it; a rollback that refuses to finish
because verification is imperfect leaves the device in the worst available state.

And if the rollback fails too, the error you report is the **original** one. That is the
error that explains what happened. The rollback failure is a second fact, worth logging,
but reporting it as the headline tells the recipient only that recovery failed, not what
they were recovering from.

> **Teacher's aside.** The general principle underneath this chapter: **the unit of safety
> is the last moment at which the old state still exists.** Dual-bank hardware pushes that
> moment to the very end, so almost the whole operation is reversible. Single-bank hardware
> puts it at the very beginning, so almost none of it is — and the only way to buy any of
> it back is to copy the old state somewhere else first. Everything else here (validate
> before you erase, verify by reading, roll back on failure) is downstream of that one
> question: *where is the last copy of the working system, and when does it stop existing?*

## The cost of the single-bank design

Worth stating plainly, because it is the argument to hand a hardware team choosing a part.

| Cost | Detail |
|---|---|
| Time | The image crosses the wire at least three times: out for the backup, in for the write, out again for the verify. A rollback adds a fourth |
| Reboots | Each of those phases may need its own entry to and exit from the bootloader, so the device resets several times on the *happy* path |
| A place to keep backups | Per-device files, somewhere durable, that are not swept up by log rotation or removed by a package upgrade — and which need a retention policy, or they accumulate forever |
| An irreversible window | Which no amount of host-side care removes. It only gets shorter |
| Exclusivity | While the device is rebooting it is unavailable, so anything else on the machine that might be *writing* through it has to be stopped or refused first. A reader that loses power mid-read has failed a read; one that loses power mid-write may have left the thing it was writing to in a state nobody can predict |

That last row is a design decision in itself: the requirement is usually not "lock the
device" but "do not reboot it while something is writing through it", which is a narrower
and much cheaper thing to enforce.

## Check yourself

1. A device has two program banks and a boot pointer. Power is lost immediately after the
   pointer write and before the reset. What is the state of the device? Now answer the same
   question for a single-bank device, at the equivalent point.
2. Your updater uploads the old image, writes it to a file, and proceeds. Give two ways
   that file can be useless as a backup, both of which would pass a "did the file get
   written?" check.
3. Where exactly is the commit point, and what should a cancel request do on each side of
   it? Explain why checking for cancellation when the operation *starts* is not enough.
4. The write reports success at every step and the device is running the old firmware
   afterwards. Which specific protocol behaviour from chapter 4 explains this, and which
   step of the safe sequence would have caught it?
5. A rollback fails after a failed write. Argue for which of the two errors the user should
   be shown, and what should happen to the other one.
6. A post-update health check counts the device's functional endpoints and finds the same
   number as before, so it reports success. Construct a scenario where that is wrong, and
   say what the check should have compared instead.
