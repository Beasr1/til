# 13. Exercises & answers

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

## File 11 — Key attestation, field by field

**1. Why `attestationApplicationId` is weaker than `verifiedBootState`.**
Both are inside one blob carrying one genuine TEE signature, so the signature doesn't
distinguish them. What distinguishes them is **what the TEE can determine for itself**.

`verifiedBootState` the secure environment *measures*: it participates in boot verification and
knows which key verified the images and whether the bootloader is locked. Nothing in the normal
world is consulted, so nothing in the normal world can change it — it lands in
`hardwareEnforced`.

The TEE has no way to know which app is calling it. It has no view of the package manager, no
notion of UIDs, no access to APK signatures. It is *handed* a package name and some digests by
the Android keystore daemon in the normal world, and it signs them. That lands in
`softwareEnforced`, which AOSP says "can be trusted as long as the device is running an
operating system that complies with the Android Platform Security Model".

So the signature attests that *keystore said this*, not that it is true. A hardware signature
over a software-asserted field proves the software said it.

**2. The verifier that skips `RootOfTrust`.**
Its app-identity check is worth nothing, because the only thing that made
`attestationApplicationId` believable was the conditional in AOSP's definition — "as long as
the device is running an operating system that complies with the Android Platform Security
Model" — and `RootOfTrust` is the *only* evidence in the attestation about whether that
condition holds. Skip it and you are trusting a self-report from an OS you have not established
anything about.

The device state that defeats it: **unlocked bootloader running a modified OS** —
`verifiedBootState` would read `Unverified`, `deviceLocked` false, `verifiedBootKey` 32 zero
bytes. The attacker patches keystore (or anything feeding it) to report your package name and
your signing digest, the TEE faithfully signs that claim, the chain validates to Google's root,
and your check passes. Nothing was forged; the component that fills in the field was simply the
attacker's.

Note the asymmetry with question 1 — this is why the two checks are not independent and not
interchangeable. §11.4's reasoning chain runs one way only: boot state first, then app identity.

**3. Works on test devices, fails on every real phone.**
Almost certainly **Play App Signing**: you are comparing against your **upload** key's
certificate digest, while the APK users install has been re-signed by Google with the **app
signing** key. Your test builds are signed with your own release/upload key, so they match;
nothing from the store ever will.

Where to get the right value: Play Console → Setup → App signing → the SHA-256 fingerprint
under *App signing key certificate*. (And if you ship to users through more than one route,
you may legitimately need several digests — Google's matcher accepts *any* expected digest
present in the candidate's set.)

**4. The relay attack against the nine checks.**
The attacker intercepts `generateAttestedKey()` on a rooted phone and relays it to a clean,
locked device, returning that device's genuine response.

Honest passes — nothing forged, nothing extracted:

| Check | Why it passes honestly |
|---|---|
| 1 path to Google's root | the chain was issued by real secure hardware |
| 2 revocation | that device's key isn't revoked; it was never leaked |
| 3 RKP validity | freshly provisioned, well within validity |
| 4 extension parse | a single, genuine extension on the genuine leaf |
| 5 challenge | **your** nonce was relayed and embedded by the remote TEE |
| 6 `origin = GENERATED` | genuinely generated inside the remote TEE |
| 7 security level | genuinely `TrustedEnvironment` or `StrongBox` |
| 8 `RootOfTrust` | the remote device really is locked and `Verified` |

Eight of nine pass without a single lie, which is Quarkslab's point: "We do not attack the
secure hardware, we do not extract keys, and we do not need a leaked keybox."

Only check 9 sees anything wrong — the `attestationApplicationId` belongs to a different app
(and a different device). That makes 9 **structural** rather than cosmetic: it is not a bonus
hardening step on top of device integrity, it is the only check that **binds the attestation to
the asker**. Without it, device integrity is being proven about *some* device, which is a
statement with no subject.

**5. "Attestation verified: strict app ID" without comparing anything.**
Two configurations, both from Google's own library:

(a) `Verifier`'s `expectedAttestationApplicationId` constructor parameter is left at its default
of `null` (`Verifier.kt:155`). `STRICT` is
`expectedId?.isSatisfiedBy(actualId) ?: true` (`ConstraintConfig.kt:428–431`) — a null expected
value short-circuits to **true**.

(b) An `AttestationApplicationId` *is* supplied but has empty `packages` **and** empty
`signatures`. `isSatisfiedBy` only tests the lists that are non-empty (`Extension.kt:667–678`),
so with both empty it falls through to `return true`.

Both log success having compared nothing. The general lesson is worth more than the two cases:
a check whose "no expectation configured" branch returns *pass* will eventually be deployed with
no expectation configured. Fail closed — assert at startup that the expected value is present
and non-empty.

**6. Expired root trusted, RKP validity enforced.**
Not a contradiction, because the two statements are about different positions in the chain.

The root is the **trust anchor**, and RFC 5280 §6.1 excludes the anchor from the certification
path ([`pki/01` §1.6](../pki/01-what-a-certificate-is.md)) — so no path check, expiry included,
applies to it. Its validity dates were a statement by an issuer, and an anchor has no issuer
above it to make that statement meaningfully. Google is therefore free to say "keep trusting
chains to `f92009e853b6b045` regardless of validity period", and Android applies the same
instinct to TLS anchors.

RKP intermediates are **in** the path, so check 2 applies to them normally. More than that,
their short life is the *mechanism*: Google calls the shorter expiration "an important
consideration of the threat model… intended to allow for quicker responses to compromises".
Ignoring expiry there would discard the containment property the design was bought with.

**7. A leaked factory keybox.**
With the private key, the attacker signs their own leaf containing whatever `KeyDescription`
they like, chaining to Google's real root.

| Check | Catches it? |
|---|---|
| 1 path validation | **no** — the chain is cryptographically valid to the real root |
| 5 challenge | **no** — they embed your fresh nonce; they control the contents |
| 6, 7, 8 `origin`, security level, `RootOfTrust` | **no** — all forgeable, because they are writing the extension themselves. This is the one case where `hardwareEnforced` lies, because the hardware's *key* has escaped the hardware |
| 9 app ID | **no** — same reason |
| **2 revocation** | **yes, once Google lists the serial** — the only check that can, and only after the leak is known |

So the honest position is that a leaked keybox defeats content checks entirely; the defence is
revocation, and it is reactive.

Requiring `REMOTELY_PROVISIONED` (`provider/KeyAttestationCertPath.kt:167`) buys you a
structural improvement rather than a cryptographic one: RKP keys are per-app, rotated, valid
around two months, and never exist outside the device's secure environment — so there is no
factory keybox to leak, the blast radius of any single compromise is one app briefly rather than
~100k devices for years, and Google can simply stop provisioning compromised software. Google's
framing is that attestation key leaks "don't apply to attestation keys certified by the newer
Remote Key Provisioning (RKP) mechanism". On devices launching with Android 16 it's the only
mechanism anyway.

**8. An SDK wrapped by third parties, each with their own signing key.**
**What to collect from each client:** the **SHA-256 digest of the signing certificate of the APK
as installed** — which for a Play-distributed app is the app signing certificate from their Play
Console, not their upload certificate — plus the **package name** of their app, and a minimum
`version` if you want a downgrade floor. Then configure the expected set so that any one of your
clients' (package, digest) pairs satisfies the check; Google's matcher is already "any expected
digest appears in the candidate's set".

**Why a chain check is the wrong mechanism for that part:** app signing certificates are
self-signed. There is no issuer CA, nothing above them, and no path to build
([`pki/03` §3.4](../pki/03-four-chains-compared.md)). The control is a **pinned digest**. The
chain check in this system is a different chain entirely — the attestation chain, terminating
at Google's attestation root — and it says nothing about who wrote the app except through the
digests carried in this field.

**What breaks on rotation:** every attestation from that client's updated app reports the new
digest and fails your check, locking out their whole user base at once. You need to accept old
and new digests through the overlap, which means (a) clients must tell you *before* they rotate,
and (b) your expected-digest set has to be configuration you can change without a deploy. Note
that v3 signing lineage ([file 03](03-signing-is-identity.md) §3.2) lets the *device* accept the
update — it does nothing for *your server*, which only ever sees the new digest and has no idea
it was endorsed by the old key.

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
- **Dump an attestation chain and read the leaf.** Generate a Keystore key with
  `setAttestationChallenge`, call `KeyStore.getCertificateChain`, write the certificates out and
  run `openssl x509 -text -noout` on the leaf. Confirm `CN=Android Keystore Key`, serial `1`,
  and the extension at `1.3.6.1.4.1.11129.2.1.17` (file 11 §11.1).
- **Find which list each field is in.** Parse that extension and print the tags present in
  `softwareEnforced` versus `hardwareEnforced`. Locate `attestationApplicationId` in one and
  `rootOfTrust` in the other — §11.4's whole argument, on your own device.
- **Watch the boot state change the meaning.** Repeat the dump on a locked device and on one
  with an unlocked bootloader. Compare `verifiedBootState`, `deviceLocked` and `verifiedBootKey`
  (32 zero bytes when unlocked), then re-read §11.4 and say what the second device's
  `attestationApplicationId` is worth (files 05, 11).
- **Confirm the digest you'd actually compare.** `keytool -printcert -jarfile your.aab` for your
  upload certificate, then the SHA-256 under *App signing key certificate* in the Play Console.
  Note that they differ — that difference is file 11 §11.5's five-minute bug.
- **Fetch both roots and the status list.** `curl https://android.googleapis.com/attestation/root`
  and `.../attestation/status`. Note there are two roots with different key algorithms, and that
  the status list contains only the non-normal entries (file 11 §11.2, §11.8).

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
- "Here's our attestation verification code — walk the nine checks in file 11 §11.7 and tell me
  which ones we're not doing."
- "Which fields in this attestation dump may I believe, and what does each one depend on?"
- "We ship an SDK into clients' apps. Design the app-identity check across all of them, and tell
  me what happens when one client rotates their signing key."
- "Key attestation or Play Integrity for our case? Argue it on the GMS dependency alone, then on
  effort."
- "Could the relay attack work against us as currently configured? Show me where it gets in"
