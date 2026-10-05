# 1. Why firmware updates are hard

## The problem

You have a USB device in the field — a reader, a sensor, a printer — and there is a bug in
the program running inside it. No change to your application fixes it, because the bug is
not in your application. It is in code stored in the device's own flash memory, which
starts before any driver loads and which you have no way to reach.

Fixing it means replacing that code. And the only thing that can write to the device's
flash is the device itself.

That is the whole difficulty in one sentence: **the device has to cooperate in its own
replacement**. There is no external authority you can appeal to. You ask the running
program to stand aside, and if it doesn't — or if it stands aside and then something goes
wrong — you have no second channel.

## Firmware is not software

The distinction matters because almost every instinct you have from updating software is
wrong here.

| | Software on the host | Firmware in the device |
|---|---|---|
| Runs on | the computer's CPU | the device's own microcontroller |
| Stored in | the host's disk | the device's flash, a few tens or hundreds of KB |
| Replaced by | an installer; the old files are still on disk until the new ones land | erasing the only copy, then writing a new one |
| If it goes wrong | reinstall, or roll back to the previous package | the device may not come back at all |
| Who can write it | any process with permission | only the device, from inside |

The third row is the one to sit with. A package manager writes new files alongside the old
ones and flips a pointer at the end; nothing is destroyed until the new thing is known to
be complete. A single-region firmware update does the opposite. It erases first and writes
second, and between those two moments the device holds no working program.

## The device has to change what it is

The USB Device Firmware Upgrade specification is unusually candid about this. Its overview
says that because a device cannot do both jobs at once, its normal activity must stop for
the duration — and then, memorably:

> a printer is *not* a printer while it is undergoing a firmware upgrade; it is a PROM
> programmer.
>
> — USB DFU 1.1, §2 Overview

The same section adds the constraint that shapes everything in chapter 2:

> a device that supports DFU is not capable of changing its mode of operation on its own
> volition. External (human or host operating system) intervention is required.

So the update is not one operation. It is at least three: persuade the device to become a
programmer, program it, and persuade it to become a device again. DFU 1.1 names four
phases — Enumeration, Reconfiguration, Transfer, Manifestation — and the two on the
outside are where most of the trouble lives.

```
  normal device            programmer              normal device
 ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
 │ does its job │───────▶│ erases and   │───────▶│ does its job │
 │              │ reboot │ writes flash │ reboot │  (new code)  │
 └──────────────┘        └──────────────┘        └──────────────┘
        ▲                       │                       │
        └── if we never get ────┘                       │
            here, the device is stuck in the middle     │
                                                        ▼
                                          ... or does nothing at all
```

## What actually goes wrong

Worth enumerating, because the defences in later chapters each exist for one of these.

| Failure | When | Consequence |
|---|---|---|
| Power lost or cable pulled mid-write | during Transfer | flash holds half an image |
| The image is valid but wrong for this device | before or during Transfer | a device that boots into nonsense |
| The image is corrupt in transit or on disk | before Transfer | same |
| The device reboots into the programmer and never leaves | Reconfiguration or Manifestation | device present on the bus, useless for its job |
| The host cannot find the device in its programmer form | Reconfiguration | update aborts *after* the device has already stopped being a device |
| The write succeeds and the new firmware is broken | after Manifestation | worst case — everything reported success |

That fifth row deserves a name, because it is the one people don't design for. The device
has already rebooted. Nothing has been erased, so no flash is damaged. But the device is
sitting in its programmer mode waiting for a host that has given up, and it will stay
there until something resets it. The update "failed safely" and the device is still not
doing its job.

> **Teacher's aside.** "Bricked" is used loosely to mean any failed update, and that
> obscures the only distinction that matters: **is there still something running that can
> accept a new image?** A device with a separate, unwritable bootloader is never bricked by
> a bad application image — you can always try again, because the thing that accepts images
> was never at risk. A device where the update path is part of the code being replaced can
> genuinely die, and then recovery means physical access: a jumper, a boot pin, a
> programmer clipped to the board. Ask which kind you have *before* you write anything. It
> changes what a failure costs by several orders of magnitude.

## The last-resort property

Follow that aside into the standard and you find it written down. In the DFU 1.1 state
tables, several states list the same pair of outcomes for a reset:

> USB reset or power on reset and firmware is valid → Re-enumeration. Revert to
> application firmware. → **appIDLE**
>
> USB reset or power on reset and firmware is corrupt → Re-enumeration. Remain in DFU mode
> awaiting recovery attempt by the host. → **dfuERROR**
>
> — USB DFU 1.1, Appendix A.2.3 (dfuIDLE), and repeated in A.2.4–A.2.6

A device built to that rule cannot lose its ability to be reprogrammed. It decides, on
every reset, whether its application is fit to run, and if not it stays where a host can
reach it.

⚠️ Read "firmware is valid" carefully. The spec does not say how the device decides — that
is entirely vendor-specific, and a device that only checks "is the first word a plausible
stack pointer?" will happily jump into a half-written image. Whether a particular
bootloader really comes back on its own is something to **test on a device you are willing
to lose**, not something to infer from the fact that the specification describes the
behaviour.

## Why bother at all

Because the alternatives are worse. Four reasons that apply to any device fleet:

- **Bugs.** A timeout, a dropped transfer, a failure under load. If the bug is in the
  device, only new firmware fixes it.
- **Compatibility.** New host operating systems, new USB stacks, new peripherals the
  device has to interoperate with. Old firmware meets conditions it was never tested
  against.
- **Security.** A device sitting in the path of sensitive data is part of your attack
  surface. A vulnerability inside it can only be closed by replacing it.
- **Fleet consistency.** Supporting several firmware versions at once means every bug
  report starts with "which version?". Convergence removes a variable from every
  investigation you will ever do.

And the reason to automate it rather than have someone visit each device: a manual
flashing tool operated by a human at each machine scales linearly with the number of
machines, and offers no backup, no verification and no rollback unless the person thought
to arrange them.

## Check yourself

1. A colleague proposes shipping firmware updates the way you ship application updates:
   download the new image, write it, done. Name the two properties of application updates
   that this plan silently assumes and that firmware updates do not have.
2. An update fails with a clean error before a single byte of flash has been erased, and
   the device is nevertheless unusable afterwards. Explain how, mechanically.
3. DFU 1.1 says a device cannot enter upgrade mode "on its own volition". What does that
   force the host software to be able to do, and what happens to the update if it cannot?
4. Two devices both have "a bootloader". On one, a failed write is always recoverable over
   USB; on the other it may need a jumper and a bench. What is the difference between them,
   and which question would you ask a vendor to find out which you have?
5. The DFU state tables say a device whose firmware is corrupt should "remain in DFU mode
   awaiting recovery attempt by the host". Why is it still unsafe to rely on that when
   planning a fleet-wide update?
