# Android App Security — A Course

How an Android app is isolated, signed, booted and attested — and how every one of those
defences is attacked. From the Linux sandbox underneath a single app up to hardware-backed
attestation a server can trust about a device it has never seen.

**This is reference learning material.** It is about the platform and the public toolchain —
AOSP, the Android developer docs, and each tool's own documentation — not about any
particular app you might be hardening. Nothing here is specific to one product.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- *Why* before *how*, and the mental model before the mechanism.
- Every file ends with **Check yourself** questions. Answers are in [`13-exercises.md`](13-exercises.md).
- Android has twenty years of jargon — `init_boot`, Zygote, DEX, Zygisk, AVB — that everyone
  pretends is obvious. None of it is.

## The one thing to understand first

Every attack in this course, and every defence against it, is one of exactly two kinds.

> **STATIC** — change the app *on disk*. Decompile the APK, edit it, rebuild it, re-sign it,
> install the altered copy. The running process is honest; the file it ran from was not.
>
> **DYNAMIC** — leave the app on disk alone and change it *in memory*, while it runs. Attach
> to the live process and rewrite its functions, its return values, its data. The file is
> pristine; the process is lying.

Hold those two words. Almost every term that follows slots under one of them:

| | Attacks | The defence that answers it |
|---|---|---|
| **Static** | decompile → edit → rebuild → re-sign (file 07) | signature = identity, DEX checksums (files 03, 09) |
| **Dynamic** | Frida, Xposed/LSPosed (file 08) | instrumentation detection (file 09) |
| **Both, from below** | root gives either one more power (file 06) | root detection, and ultimately attestation (files 09, 10) |

And the sting the whole course builds towards: **on a device the attacker fully controls,
every on-device defence is bypassable.** The only thing that survives is a statement made by
hardware the attacker can't rewrite and checked by a server they don't own (file 10).

## The primary sources

Everything here is checkable against one of these. Where I say "AOSP says" or "the docs
say", this is which.

| Source | Covers |
|---|---|
| **AOSP** — [source.android.com](https://source.android.com) | The platform: the runtime (ART), Verified Boot (AVB), boot partitions, APK signing, key attestation |
| **Android developers** — [developer.android.com](https://developer.android.com) | The app-facing contract: Play Integrity verdicts, the SafetyNet turndown, `apksigner` |
| **Magisk** — [github.com/topjohnwu/Magisk](https://github.com/topjohnwu/Magisk) | Systemless root, `magiskinit`, Zygisk, the DenyList |
| **Frida** — [frida.re/docs](https://frida.re/docs/home/) | `frida-server` vs `frida-gadget`, GumJS, hooking |
| **LSPosed / Xposed** — [github.com/LSPosed](https://github.com/LSPosed/LSPosed) | Runtime method hooking on ART |
| **Shizuku** — [github.com/RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku) | The privileged-process + Binder bridge |
| **Key attestation verifier** — [github.com/android/keyattestation](https://github.com/android/keyattestation) | Google's own Kotlin verifier. Cited by `file:line` in file 11 for what the checks actually are, and what its defaults don't check |
| **apktool / smali / jadx** | [apktool](https://apktool.org), [smali/baksmali](https://github.com/JesusFreke/smali), [jadx](https://github.com/skylot/jadx) — the repackaging pipeline |

AOSP and the developer docs are free and authoritative. The tools are open source; their
READMEs and docs are the reference, not the forum posts that mirror them.

## Reading order

### Part 1 — The platform from zero (read these first, in order)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [The Android platform](01-the-android-platform.md) | Say what "an app" is, and why one app can't read another's files |
| 2 | [From source to running app](02-from-source-to-running-app.md) | Explain Kotlin → DEX → APK → ART, and name everything inside an APK |
| 3 | [Signing is identity](03-signing-is-identity.md) | Explain why re-signing an app makes it a *different* app — the crux of the whole course |
| 4 | [Zygote and the process model](04-zygote-and-the-process-model.md) | Say why every app is a fork of one parent, and why that is a gift to code injection |
| 5 | [The boot chain and Verified Boot](05-the-boot-chain-and-verified-boot.md) | Explain the lock states, and why unlocking the bootloader wipes the device |

### Part 2 — The attacks

| # | File | Covers |
|---|------|--------|
| 6 | [Root](06-root.md) | `su`, systemless vs `/system` modification, Magisk's boot/`init_boot` patch, Zygisk, DenyList |
| 7 | [Static: reverse engineering and repackaging](07-static-attack-repackaging.md) | jadx to read, apktool + smali to edit, rebuild, re-sign — one worked example |
| 8 | [**Dynamic: instrumentation**](08-dynamic-attack-instrumentation.md) | ⭐ Frida, Xposed/LSPosed, and why Shizuku is *not* one of them |

### Part 3 — The defences, and what finally holds

| # | File | Covers |
|---|------|--------|
| 9 | [Defences and the arms race](09-defences-and-the-arms-race.md) | Root/instrumentation detection, integrity checks, obfuscation — and why each is beatable |
| 10 | [**Hardware-backed attestation**](10-hardware-backed-attestation.md) | ⭐ The TEE, Key Attestation, Play Integrity, server-side verification — the one thing that holds |
| 11 | [**Key attestation, field by field**](11-key-attestation-in-depth.md) | ⭐ The chain, the extension, hardware- vs software-enforced fields, the server-side check order, revocation — and what the app-signing digest really proves |

### Reference

| # | File | |
|---|------|--|
| 12 | [Glossary](12-glossary.md) | Every abbreviation, in one place |
| 13 | [Exercises & answers](13-exercises.md) | Worked answers to every **Check yourself**, plus things to try |

## If you're short on time

| You want | Read |
|---|---|
| The whole model in twenty minutes | the summary below, then file 03 |
| To understand why "just check if the device is rooted" fails | 6, then 9 |
| To understand a repackaging finding ("the app can be rebuilt and bypassed") | 3, then 7 |
| To know what a Frida hook can and can't do | 4, then 8 |
| To design a defence that actually holds | 10 (and accept that 9 alone never does) |
| To actually *implement* attestation verification | 10, then 11 §11.7 |
| To know whether attestation proves your app wasn't repackaged | 11 §11.9 |

## The one-paragraph summary of everything

An Android app is a process running under its own **Linux user ID**, which is the whole
sandbox: the kernel stops one UID reading another's files, so isolation is a 1970s operating
-system feature, not an Android invention. The app ships as an **APK** — a ZIP holding a
manifest, compiled **DEX** bytecode, resources and native `.so` libraries — which the
device runs on **ART**, compiling the DEX to machine code. The APK is **signed**, and the
signature is the app's *identity*: the system keys permissions and update-eligibility to the
signing certificate, so an attacker who edits the app must re-sign it with their own key,
which makes it — to the system — a different app that can't update or inherit the original's
data. That is the defence against **static** attacks (decompile, edit, rebuild, re-sign). But
an attacker who **roots** the device defeats the premise: root is just UID 0, and modern
root (**Magisk**) is *systemless* — it patches the boot ramdisk, never the read-only system
partition, so it survives updates and hides from naive checks. With root, **dynamic** attacks
open up: **Frida** injects an agent that rewrites functions in memory; **Xposed/LSPosed**
hook every app persistently through **Zygote**, the single parent process every app is forked
from. On-device defences — root detection, Frida-port checks, DEX checksums, obfuscation —
all run *inside* the very process the attacker controls, so each is bypassable by editing it
out or hooking it to lie. The escape from that trap is to stop trusting the device: the
**bootloader** enforces **Verified Boot**, and a locked device with a certified OS can make a
**hardware-backed** statement — a **Key Attestation** certificate chain, or a **Play
Integrity** verdict — that a **server** verifies, because the signing key lives in a
**TEE** the attacker can't reach even with root. That, and only that, holds against a client
the attacker owns.

## How to use me

These notes are a starting point. Good questions to bring back:

- "Walk me through what's inside this APK I just unzipped."
- "An assessment says our app 'can be decompiled, modified and recompiled'. What exactly is the risk, and what stops it?"
- "If the bootloader is locked, can the user still run Frida? Why or why not?"
- "Here's a root check in our code. How would someone bypass it, and is it worth having?"
- "What does `MEETS_DEVICE_INTEGRITY` actually prove, and what doesn't it?"
- "Why can't I just check for root on-device and be done with it?"
- "SafetyNet is gone — what replaced it and what changed about the guarantee?"
