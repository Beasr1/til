# 12. Exercises & answers

Worked answers to every **Check yourself**, then things to try and questions to bring back.
Read an answer only after you've committed to one — the wrong answers here are the ones that
cost afternoons.

---

## File 1 — The Android platform

**1. Game reads the banking file; who denied it, and when?**
The **kernel**, via the Linux UID/permission check. The file is owned by the banking app's
UID with a mode that excludes other UIDs, so the game's `open()` is refused with
"permission denied" — *before* any app code runs. The banking app never even learns the
attempt happened. The wall is in the OS, not in app logic.

**2. Two developers, same package name.**
Not the same app. The package name is just a claimable string; the system tells them apart by
the **signing certificate** (file 03). They collide on install (you can't have two packages
with one name), and neither can update over the other because the certificates differ.

**3. `logcat` works, `cat` of the private file fails — both "me".**
Both run as the `shell` user, UID 2000. The system log buffer is a resource UID 2000 is
permitted to read, so `logcat` works. The app's private file is owned by the *app's* UID with
a mode excluding others; UID 2000 is neither that UID nor root, so the kernel denies the read.
Same identity, different resources, different permissions.

**4. "We granted ourselves `INTERNET`, so now we can read other apps' data."**
Two unrelated mechanisms confused. A permission is a door to a *shared* resource — `INTERNET`
adds your UID to a group the kernel checks for network access. Reading another app's private
files is blocked by the **UID boundary**, which no permission touches. Granting a permission
never widens access to another app's private storage.

**5. Why does root require the read-only system world, when the data is in `/data`?**
Because reading another app's `/data` requires *already* being that app's UID or root. Gaining
root means installing a trusted mechanism to become UID 0 — an `su` path the system honours,
wired into system/boot components. Those live in the read-only, Verified-Boot-protected world
(file 05). You can't drop a file in `/data` to become root; you must modify the protected
system/boot world, which is the hard part. `/data` is the prize, not the lever.

---

## File 2 — From source to running app

**1. `app.apk` renamed to `app.zip` opens.**
Modifying an app is trivial — it's a ZIP. Getting the modification *accepted* is the hard part
(the signature, file 03). The two are decoupled: easy to edit, hard to impersonate.

**2. Reading Kotlin login logic — how much work?**
Little. `classes.dex` carries class, method and field names plus the logic as bytecode; jadx
decompiles it to near-source Java in seconds. Obfuscation makes it uglier but still readable.

**3. Move the logic into a Rust `.so`.**
Changed: it's now machine code, which doesn't decompile to clean source — the attacker must
read disassembly, meaningfully more effort. Unchanged: it still executes on their device, so
it can be disassembled, patched on disk, or hooked at runtime (file 08). A higher bar, same
category.

**4. "ART compiles at install, so you can't hook by method name."**
The hole: from Android 7.0 ART is hybrid — it interprets and JITs and keeps the DEX; it
doesn't fully AOT-compile everything at install. And ART links by class/method name at
runtime regardless, which is exactly what in-memory hooking targets. The compiled image is a
cache, not a names-erasing replacement.

**5. r8 renamed everything to `a`, `b`, `c` — "encryption"?**
Wrong word. Nothing is secret or needs a key to read; the code runs and is fully readable,
just with meaningless names. What's achieved is raised *understanding* cost — you must
re-derive what `a` does — not confidentiality. It's reversible by reading.

---

## File 3 — Signing is identity

**1. One method changed, re-zipped — v1 vs v2.**
v1 lists per-file digests, so the changed file's digest no longer matches — caught — but v1
leaves ZIP structure and metadata unprotected, so some manipulations slip through its gaps.
v2 hashes the whole APK as one blob in the signing block, so *any* byte changed outside that
block fails verification — no structural gaps.

**2. Re-signed with a new key — three things it can't do.**
(a) Update over the original — update continuity requires the same key/lineage, so a different
cert is rejected. (b) Read the original's `/data` — it's a different install with its own UID
directory (data ownership). (c) Claim signature-level permissions defined by same-cert apps.
Each from a different binding in §3.3.

**3. Lost signing key; users can't update.**
v3's signing **lineage** (key rotation) was meant to prevent exactly this — but it requires
the developer to have set up rotation *in advance*, while they still held the old key, so the
new key carries a proof it's endorsed by the old. Lost outright with no prior lineage means no
such proof, so updates are rejected.

**4. `apksigner` then `zipalign`, installs fail verification.**
`zipalign` ran *after* signing and shifted bytes the v2+ whole-file signature already covered,
invalidating it. Fix the order: build → zipalign → sign.

**5. "Signed, so it can't be tampered with."**
True half: it can't *masquerade* as the original — can't update over it, inherit its data or
permissions, or match a pinned certificate. False half: it can absolutely be modified and run
as a *separate, re-signed copy* on a device the attacker controls. Signing bounds
impersonation, not modification.

**6. Re-signed copy talks to your server; device sig-check passed.**
The server can compare the presented certificate to the known-good one it (or Google Play)
distributes — the thing a device-side check structurally can't be trusted to do, because that
check runs inside the attacker's process and can be hooked or edited out (file 09). The server
is the one party the attacker doesn't control, so the check must live there.

---

## File 4 — Zygote and the process model

**1. Why is launch fast, and what two savings?**
Zygote preloads ART and the framework once, then forks a child per launch. Time: no per-launch
framework load. Memory: copy-on-write shares the framework's pages, so one physical copy backs
every app.

**2. What breaks if Zygote exec'd in the child?**
`exec()` replaces the child's memory image, discarding the preloaded framework. You'd pay the
full load every launch and lose COW sharing — exactly the two savings above. It'd be a cold
runtime start each time.

**3. Code in Zygote before it forks — who's affected?**
Every app process forked afterwards, i.e. all of them, because each is a copy of the template.
Contrast attaching to one running app, which affects only that process. Template injection is
global and persistent; per-process attach is local and ephemeral.

**4. COW reassuring for code integrity but not a boundary.**
Reassuring: framework code pages are shared read-only and consistent, and a stray write gets
a private copy rather than corrupting neighbours. Not a boundary: COW is a memory-sharing
optimisation, not access control — anything that can already write to the process (root,
injection) is not stopped by it.

**5. Two protections between an app and Zygote.**
The UID boundary (an app is its own UID and can't touch a system process) and the read-only,
Verified-Boot-protected system/boot world (can't modify `app_process` or the ramdisk that
launches Zygote). File 06 (root) removes the first; file 05 (unlock/Verified Boot) is what
must be defeated for the second.

---

## File 5 — The boot chain and Verified Boot

**1. Why the first link in hardware?**
If the anchor were a file the bootloader reads, an attacker could replace both the file and
the code it vouches for, and the chain would happily validate the forgery. An immutable
hardware key can't be swapped, giving a fixed point the attacker can't move the root of trust
to.

**2. dm-verity on every read, not just at boot.**
An attacker can modify a block after boot, or a block never read during boot. Checking on
every read catches tampering whenever the data is actually used; a boot-time-only check would
miss both cases.

**3. Yellow screen naming a key.**
Locked bootloader, verified against the *user's* enrolled custom root of trust, not the OEM's.
Green is OEM-key-verified with no warning; yellow is custom-key-verified and shows a warning
identifying that key.

**4. Why unlocking forces a wipe.**
If data survived, someone with brief physical access could unlock, flash an OS that bypasses
the lock screen or decryption policy, and read the previous owner's data. Data-at-rest
protection assumes only the verified OS runs; unlocking removes that assumption, so the data
must be destroyed first. The wipe severs the path.

**5. Attacker unlocks successfully — two ways the device told on them.**
(a) The boot state is now **orange** (unlocked/unverified), shown at boot and attestable. (b)
The running system/boot image no longer matches the OEM-signed one. File 10 turns the first
into Play Integrity's device verdict and the second into Key Attestation's extension.

**6. "I only added root, didn't break anything."**
Without a bootloader exploit, running modified system/boot code required unlocking — so the
boot state is now orange/unverified and the chain no longer vouches for the OS. The attestable
trust state necessarily changed, even if the UI is identical.

---

## File 6 — Root

**1. "Rooting broke dm-verity" — true for which style?**
True for old-style `/system` root: it writes to the verity-protected system partition, so the
hash mismatches and verification breaks. False for systemless root: it patches only the boot
ramdisk and leaves `/system` pristine, so verity still passes. The difference is which
partition is modified.

**2. Files appear without writing `/system`.**
Bind / "magic" mounts overlay files from `/data/adb/modules` onto the paths the system
expects, in the live mount namespace. The underlying read-only partition's bytes — and so its
dm-verity hashes — are untouched.

**3. Launched on 12, updated to 14 — which image?**
`boot.img`: the *launch* version decides, and it launched on Android 12. Read
`ro.product.first_api_level` to confirm (< 33 ⇒ `boot`). The current OS version is a
distractor.

**4. "Zygisk hides root."**
No. Zygisk runs module code inside Zygote/app processes — a code-loading reach, not
concealment. The hiding is the **DenyList**, which unmounts Magisk's changes for listed apps.
Two separate features.

**5. No `su` in `/system`, no manager app ⇒ clean?**
Wrong because (a) systemless root never puts `su` in `/system`, so that path is legitimately
empty on a rooted device; (b) the DenyList hides Magisk's artefacts from the checking app; and
(c) the check itself can be hooked to report "clean" (file 08).

**6. DenyList fools the app; the server still declines.**
The server reads the hardware-attested boot/lock state (orange/unlocked) via Key Attestation /
Play Integrity, which originates in the **TEE**, below Magisk (file 10). DenyList can hide
mounts in the normal world; it can't change what the TEE reports.

---

## File 7 — Static attack

**1. Understand, then change — which tool for each?**
jadx to understand (readable Java). apktool/smali to change (it round-trips reliably). Using
jadx output for the change is a dead end — its Java usually won't recompile.

**2. `isValid()Z` — the `Z`, and forcing `true`.**
`Z` is the boolean return type. Force true: `const/4 v0, 0x1` then `return v0`.

**3. Install fails "no certificates".**
Re-signing was skipped. apktool's rebuild dropped the original signature, and Android refuses
an unsigned APK. Not optional — every installable APK must be signed.

**4. Re-signed, installs, backend rejects.**
The signing certificate changed (the attacker's key, not the original), and the backend checks
app identity server-side. Unavoidable: they don't hold the original private key (file 03), so
any rebuild must be signed with a different one.

**5. "Encrypt the APK."**
The OS must open the ZIP and ART must run the DEX, so whatever decrypts it ships with the app
and runs on the attacker's device — they can recover the decrypted form. The realistic goal of
obfuscation is to raise the cost/time of *understanding*, not to achieve secrecy.

**6. Rank by delay: (a) r8 rename, (b) native `.so`, (c) duplicate check.**
(b) > (a) > (c). Native code forces disassembly plus native hooking — the biggest bar.
Renaming forces re-deriving identifiers — moderate. A duplicate check is almost nothing: the
attacker finds and neutralises both the same way, linear extra work. All are delays because
the code still runs on the attacker's device and can be read, patched or hooked.

---

## File 8 — Dynamic attack

**1. Self sig-check passes, yet bypassed.**
The check reads the on-disk APK, which really is unmodified — the dynamic attack changes memory
after loading, not the file — so the check honestly passes while behaviour is subverted in
memory. The file and the running process have diverged.

**2. frida-server vs frida-gadget prerequisites.**
frida-server needs **root** (to inject into another process); frida-gadget needs the APK
**repackaged** to embed the library (and thus re-signed). server is the dynamic path (clean
file, rooted device); gadget is the static path (modified file, no root).

**3. Why LSPosed reaches every app but a Frida session one.**
LSPosed hooks at the Zygote boundary (file 04), and every app is forked from Zygote, so all
inherit the hooks. A Frida session attaches to a specific process, so only that one changes.

**4. Port 27042 absent ⇒ no Frida?**
Unsafe: Frida can run on a non-default port, and an embedded frida-gadget needn't expose that
port at all; the detector can also be hooked. Absence of the default port proves nothing.

**5. "Block Shizuku — it's a hooking tool like Frida."**
Category error. Shizuku is a privilege bridge: it relays the app's *own* Binder calls to the
system server at elevated privilege. It does not inject into or hook other apps, so it can't
rewrite another process's code the way Frida can.

**6. Shizuku at UID 2000 vs UID 0.**
Over ADB it runs as the shell user (2000), bounded by what that user may do — it still can't
read another app's private `/data/data` files (that needs the app's UID or root). A
root-started Shizuku (UID 0) could. Straight back to file 01's UID boundary.

**7. Dynamic beats on-disk checks but needs one prerequisite.**
Root (for frida-server / LSPosed), which needs an unlocked bootloader (files 05–06), which
flips the boot state to orange. That orange state is what hardware attestation reports
(file 10) — the signal that survives when on-device checks don't.

---

## File 9 — Defences and the arms race

**1. The shared structural property.**
All of them run inside the process, on a device the attacker controls, below which the
attacker sits and can edit code, inputs or outputs. A fourth check shares that property, so
it's bypassed the same way — adding checks is linear work, not a category change.

**2. DEX checksum: catches repackaging, useless vs Frida.**
Repackaging changes the file's bytes, so the checksum fails. Frida leaves the file unchanged
and alters memory, so the checksum honestly passes. One control, opposite relevance to the two
attack classes.

**3. "Native = secure", rewritten truthfully.**
"Moving the check into native code raises the attacker's cost — disassembly plus native
hooking instead of reading Java — turning a quick bypass into a slower one. It does not make
the check unbreakable on a device the attacker controls."

**4. Why shipping detections is partly self-defeating.**
The attacker reads your DEX (file 07) and gets a precise checklist of what you test. Each
shipped check both defends and discloses the defence, so they spoof exactly those signals.

**5. Tightening hits the wrong group.**
Legitimate users on custom ROMs or benignly-rooted devices trip the detections and churn,
while the sophisticated attacker is the one person who *can* make a device look clean. So
tightening taxes honest-but-unusual users more than adversaries.

**6. The positive case, and the line.**
Worth it as deterrence: it blocks casual and scripted abuse cheaply and raises cost for the
rest. The line: use on-device checks to raise cost and gather signals, never to *gate*
anything that must be correct against a determined attacker on their own device — that needs
off-device verification (file 10).

---

## File 10 — Hardware-backed attestation

**1. Why root can't extract a TEE key or forge the chain.**
Root is UID 0 in the *normal world*; the TEE is a separate, hardware-isolated world the
normal-world OS (and so root) cannot enter. The key never leaves the TEE and the chain is
signed inside it, so root can neither read the key nor forge a chain to the trusted root.

**2. Verifying the token client-side.**
The verifier runs in the attacker's process, so they hook it to return "valid" (files 08–09)
or fabricate the result outright. The guarantee depended on an off-device verifier the attacker
can't touch; moving it on-device hands them the check. Tokens replay or forge at will.

**3. Two things the server must check about the chain.**
(a) It verifies to a known Google/OEM root obtained out of band — stops a forged or self-signed
chain. (b) The embedded challenge matches the fresh nonce the server issued — stops replay of
an old genuine attestation. (Then read the extension for boot/lock/patch state.)

**4. `MEETS_BASIC_INTEGRITY` but not `MEETS_DEVICE_INTEGRITY`.**
The bootloader is likely unlocked and/or the device isn't certified. Basic integrity tolerates
an unlocked bootloader; device integrity (especially Android 13+) requires a locked, verified
boot. In file 05's terms: device integrity ≈ green/locked; basic tolerates orange.

**5. `appRecognitionVerdict` = `UNRECOGNIZED_VERSION`.**
Most likely repackaging (file 07): the app was rebuilt and re-signed with the attacker's key,
so the certificate/package doesn't match Google Play's record. The app's own device-side
signature check could be hooked or edited out (files 08–09); the server's check runs where the
attacker can't reach and compares against Play's authoritative record.

**6. `ctsProfileMatch` → `MEETS_DEVICE_INTEGRITY`: what tightened?**
Play Integrity ties the device/strong verdict to hardware-backed verified boot (especially
Android 13+), whereas SafetyNet's evaluation had a more software-based path that was easier to
spoof — e.g. with root concealment. The tightening is against software-only spoofing of the
certified-device signal.

**7. Two legitimate users turned away.**
(a) Devices without Google Mobile Services — de-Googled, pure-AOSP, some regions — can't
produce a verdict at all. (b) An honest power user running a legitimately unlocked/custom ROM
fails device integrity. Inherent, not a bug: the control *defines* trust as genuine + locked +
Google-attested, so any honest device outside that definition fails by design.

**8. The two properties, and removing either.**
(1) The signing key lives in hardware root can't reach; (2) the verdict is checked by a server
the attacker doesn't control. Remove (1) and it's a software-only signal that root spoofs
(file 09). Remove (2) and it's a client-side check the attacker hooks (file 08). Either removal
collapses it back into a file-09 defence.

---

## Things to try

Safe on your own device or an emulator, against your own builds — no third-party target:

- **Dissect an APK you built.** Rename it `.zip`, unzip, and list the entries. Find
  `classes.dex`, `AndroidManifest.xml`, `resources.arsc`, `lib/<abi>/`. This makes file 02
  concrete in two minutes.
- **See the signature.** Run `apksigner verify -v your.apk` and read which schemes (v1/v2/v3)
  verified. Change one byte and watch verification fail.
- **Watch DEX reverse.** Open your own debug APK in jadx-gui. Then rebuild with r8/ProGuard
  minification on and open it again — compare how much harder it is to read, and confirm the
  logic is still all there (file 02, file 09).
- **Round-trip with apktool.** `apktool d`, change a string resource, `apktool b`, then try to
  install without signing — meet the "no certificates" error (file 07). Re-sign and it
  installs.
- **Predict the boot partition.** `adb shell getprop ro.product.first_api_level` and decide
  whether that device's ramdisk is in `boot` or `init_boot` (file 06).
- **Find the debugger signal.** On an emulator, attach a debugger and
  `adb shell cat /proc/<pid>/status | grep TracerPid` — see the non-zero value a detector
  would read (files 08–09).
- **Read a real verdict.** Wire Play Integrity (or a Key Attestation request) into a test
  build and inspect the returned verdict / certificate chain on *your own* locked and then
  unlocked device — watch the device-integrity label change (files 05, 10).

## Questions worth asking me

- "Walk me through the smali for this method, and show what forcing it to return X does."
- "Here's a root check we ship — rank its signals by how hard each is to bypass, and say which
  are worth keeping."
- "Our server got a Play Integrity verdict with these fields — what may it conclude, and what
  is it over-reading?"
- "Design a control that resists a rooted client. Where does the trust have to live, and why
  can't it live in the app?"
- "Explain the frida-gadget-vs-server choice purely in terms of the static/dynamic split."
- "We moved a check into native code and it still got bypassed — was that predictable? What
  did we actually buy?"
- "Why did unlocking the bootloader wipe the device, and what exactly would be at risk if it
  didn't?"
