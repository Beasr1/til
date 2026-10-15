# 5. The boot chain and Verified Boot

## The problem

File 04 ended on a cliff: injecting code system-wide means running a modified OS, and
something stops the device from doing that. This chapter is that something. The question it
answers is blunt: **when you power on the phone, what stops it from booting an operating
system an attacker swapped in?** If nothing did, every defence above it — the sandbox,
signing, all of it — would be running on an OS the attacker wrote, and none of it would mean
anything.

The answer is a chain of trust that starts in hardware the attacker can't rewrite.

## 5.1 Each stage verifies the next

Booting isn't one step; it's a relay. Each stage is cryptographically checked by the stage
before it, and the first link is anchored in silicon:

```
┌─────────────────────┐
│ hardware root of     │  an immutable public key / hash fused into the SoC at manufacture.
│ trust (in the SoC)   │  Cannot be changed by software, root included.
└──────────┬──────────┘
           │ verifies
           ▼
┌─────────────────────┐
│ bootloader           │  the first mutable code; its signature is checked against the root
└──────────┬──────────┘
           │ verifies
           ▼
┌─────────────────────┐
│ boot image(s)        │  the kernel + the ramdisk that starts init (file 06 patches here)
└──────────┬──────────┘
           │ verifies
           ▼
┌─────────────────────┐
│ the OS / system      │  protected at read-time by dm-verity (below)
└─────────────────────┘
```

Because each stage refuses to hand control to an unverified next stage, tampering anywhere in
the chain is caught *before* the tampered code runs. The chain is only as trustworthy as its
anchor, which is why the anchor is burned into the chip and not stored anywhere software can
reach.

Two modes you reach *outside* the normal boot are worth naming, since rooting uses them:

- **fastboot** — a low-level mode where the bootloader itself talks to a host over USB and
  can flash partition images. This is where `fastboot flash boot …` happens.
- **recovery** — a separate minimal boot image used for updates and factory reset.

## 5.2 Verified Boot and dm-verity

**Android Verified Boot (AVB)** is AOSP's implementation of that chain. For the read-only
partitions (the system and vendor images), it uses **dm-verity**: a signed Merkle hash tree
covers every block of the partition, and the kernel re-checks a block's hash *each time it is
read*. Change one block of the system image on disk and the mismatch is detected at the
moment that block is read, not merely at boot
([source.android.com/docs/security/features/verifiedboot](https://source.android.com/docs/security/features/verifiedboot/dm-verity)).

The teaching consequence: you cannot quietly edit the system partition of a Verified-Boot
device and have it keep working. That is exactly why modern root is *systemless* — it avoids
touching the verity-protected partitions at all (file 06).

## 5.3 The lock states, by colour

AVB reports the device's trust state at boot as one of four colours. You'll see these on the
warning screen of an unlocked or custom device, and — crucially — hardware can attest to them
(file 10). From AOSP's boot-flow documentation
([source.android.com/docs/security/features/verifiedboot/boot-flow](https://source.android.com/docs/security/features/verifiedboot/boot-flow)):

| State | Bootloader | What it means | User sees |
|---|---|---|---|
| **Green** | locked | Boot image verified against the **OEM** key. The intended, untampered OS | nothing — normal boot |
| **Yellow** | locked | A **custom root of trust** the user enrolled verified the image. The device is locked, but to *the user's* key, not the OEM's | a warning naming the custom key, ~10 seconds, then boots |
| **Orange** | **unlocked** | **No verification is performed at all.** The user has taken responsibility | a warning screen, then boots |
| **Red** | locked | Verification **failed** — corrupt or wrong-key image. (Two variants: dm-verity corruption `eio`, or no valid OS found) | a warning; may refuse to continue |

The one to burn in is **orange = unlocked = unverified**. An unlocked bootloader doesn't check
anything, so it will happily boot a modified OS — which is what rooting needs. The cost of
that freedom is that the device now openly advertises the orange state, and that advertisement
is what later makes root detectable *off* the device (file 10).

## 5.4 Unlocking the bootloader — and why it wipes everything

To get to orange, the user unlocks the bootloader. On modern devices that's two deliberate
steps: enable **OEM unlocking** in Developer options, then, in fastboot, run
`fastboot flashing unlock` (older devices: `fastboot oem unlock`). The device asks for
on-device confirmation — and then **erases all user data**.

AOSP states it plainly: *"For privacy reasons, unlocking the bootloader deletes user data"*,
and *"after unlocking, all data on the device is erased, including both private app data and
shared data"* ([source.android.com/docs/setup/test/running](https://source.android.com/docs/setup/test/running)).
Locking again wipes too.

Why force the wipe? Walk the attack it closes. Your data at rest is protected on the
assumption that only the *trusted* OS — the one Verified Boot vouched for — can decrypt and
read it. If unlocking preserved your data, anyone with brief physical access could unlock the
bootloader, flash an OS of their own that skips the lock screen, and read everything you had.
The wipe severs that path: unlocking throws away the data *before* any untrusted OS can be
booted against it. So the wipe isn't an inconvenience bolted on — it's the hinge that keeps
"unlockable bootloader" from meaning "readable data".

> **Teacher's aside.** This is the cleanest example in the course of security as a *trade*,
> not a wall. Android doesn't forbid you from running your own OS on your own device — that
> would be the manufacturer owning your hardware. It lets you unlock. But it makes unlocking
> (a) wipe your data, so the freedom can't be turned against the previous owner, and (b)
> *visible* and *attestable*, so a bank's server can decline to trust the result. You keep
> control of your device; you just can't both root it and present it as pristine. Hold that
> — it's the exact shape of why file 10 works and file 09 doesn't.

> ⚠️ "Locked bootloader" and "encrypted data" are not the same guarantee, but they're
> entangled: the lock is what makes the encryption's trust assumption (only the verified OS
> runs) hold. Reasoning about one while ignoring the other is how people conclude a wiped
> unlock is pointless.

## 5.5 Where this leaves the attacker

An attacker who wants to run a modified OS has, broadly, two routes:

- **Unlock the bootloader** (the normal path): legitimate, supported on many devices, but it
  flips the state to **orange**, wipes the device, and is fully visible to attestation.
- **Exploit the chain** (the hard path): find a bug in the bootloader or a verification step
  that lets unsigned code run while the state still reports **green**. This is what a
  genuinely dangerous compromise looks like, and it's rare and device-specific.

The entire premise of hardware-backed attestation (file 10) is that the first route is honest
about itself — the hardware reports orange — and the second is hard. Keep both in view: the
common case is a cooperative user who unlocked their own phone, not a broken root of trust.

## Check yourself

1. The boot chain is described as each stage verifying the next. Why must the *first* link
   live in hardware rather than in a file the bootloader reads? What would an attacker do if
   it didn't?
2. dm-verity checks a block's hash every time the block is read, not just once at boot. Why
   is "every read" the right time, and what attack would a boot-time-only check miss?
3. A device boots to a yellow warning screen naming a key. Is the bootloader locked or
   unlocked? Whose key verified the OS, and how does this differ from green?
4. Explain, as an attack scenario, why unlocking the bootloader forces a data wipe. What
   exactly becomes possible if the data survived the unlock?
5. An attacker unlocks a phone to install a modified OS. They succeed. In what two distinct
   ways has the device nonetheless "told on them", and which later chapter turns each into a
   server-side signal?
6. Someone claims their rooted phone is "just as trustworthy as a stock one — I only added
   root, I didn't break anything". In terms of boot state and the verification chain, what
   did adding root necessarily change, assuming they didn't have a bootloader exploit?
