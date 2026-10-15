# 11. Glossary

Reference, not reading. The number in parentheses is the chapter where the term is motivated.

## The central model

| Term | Meaning |
|---|---|
| **Static attack** | Modifying the app *on disk* — decompile, edit, rebuild, re-sign (README, 7) |
| **Dynamic attack** | Modifying the app *in memory* while it runs — instrumentation (README, 8) |

## The platform

| Term | Meaning |
|---|---|
| **Linux kernel** | What Android is built on; the source of the app sandbox (1) |
| **UID** | Linux user ID. Each app gets its own; the kernel enforces isolation between UIDs (1) |
| **App sandbox** | Not a bespoke layer — the Linux UID boundary, one UID per app (1) |
| **SELinux** | Mandatory access control constraining even processes that share a UID, and root itself (1) |
| **Package name** | A string like `com.example.app`. Identity to the user/store; *claimable*, so not what the system trusts (1) |
| **Permission** | A gated door to a *shared* resource (camera, network). Not what isolates one app's private files (1) |
| **`/system`** | Read-only partition holding the OS; Verified-Boot protected (1, 5) |
| **`/data`** | Writable partition holding apps' private dirs and user data (1) |
| **adb / adbd** | Android Debug Bridge; host tool + device daemon. `adb shell` runs as the `shell` user, **UID 2000** — not root, not an app (1) |

## Build and runtime

| Term | Meaning |
|---|---|
| **DEX** | Dalvik Executable — the bytecode format that ships in the APK. High-level, carries names, reverses easily (2) |
| **d8 / r8** | Tools that convert Java bytecode to DEX. r8 also shrinks and obfuscates (2) |
| **APK** | Android Package — a ZIP holding the manifest, DEX, resources, native libs and signatures (2) |
| **`classes.dex`** | The app's compiled code inside the APK (2) |
| **`AndroidManifest.xml`** | The app's declaration (package, components, permissions); stored as binary XML (2) |
| **`lib/<abi>/*.so`** | Native machine-code libraries; do *not* decompile to clean source — a higher reversing bar (2) |
| **`META-INF/`** | Where v1 (JAR) signatures live (2, 3) |
| **Dalvik** | The original runtime that ran DEX; retired after Android 4.4 (2) |
| **ART** | Android Runtime; runs DEX. Default from Android 5.0 (2) |
| **dex2oat** | ART's tool that compiles DEX to native code (2) |
| **AOT** | Ahead-of-time compilation — before running (2) |
| **JIT** | Just-in-time compilation — while running, for hot paths. ART is hybrid JIT/AOT from Android 7.0 (2) |

## Signing

| Term | Meaning |
|---|---|
| **Signing certificate / key** | The developer signs with a private key; the APK carries the certificate. The key is *not* in the APK (3) |
| **v1 (JAR signing)** | Per-file digests in `META-INF/`. Signs contents, not ZIP structure (3) |
| **v2** | Android 7.0. Hashes the whole APK as one blob, in the APK Signing Block (3) |
| **v3** | Android 9. Adds a signing lineage for key rotation (3) |
| **v3.1** | Android 13. A refinement of v3 rotation (3) |
| **v4** | Android 11. A Merkle-tree signature stored as `<name>.apk.idsig`; **requires** a v2/v3 signature; enables incremental install (3) |
| **APK Signing Block** | The region of the ZIP where v2+ signatures live (3) |
| **Signing lineage** | The v3 proof-chain letting an app change keys without becoming a new app (3) |
| **apksigner** | Official tool to apply/verify v1–v4 signatures (3) |
| **zipalign** | Aligns entries to 4-byte boundaries for `mmap`. Must run **before** v2+ signing (3) |

## Process model

| Term | Meaning |
|---|---|
| **Zygote** | The warm parent process: preloads ART + the framework, then forks a child per app launch (4) |
| **fork (not exec)** | Zygote clones itself but keeps its memory image, so children inherit the preloaded framework (4) |
| **Copy-on-write** | Forked pages are shared until written, so one framework copy backs every app (4) |
| **`system_server`** | The first process Zygote forks; hosts most system services (4) |
| **`app_process`** | The executable that *is* Zygote; original Xposed replaced it (4, 8) |

## Boot and Verified Boot

| Term | Meaning |
|---|---|
| **Bootloader** | The first mutable code; verifies the boot image. Can be locked or unlocked (5) |
| **fastboot** | Low-level mode where the bootloader flashes partition images over USB (5) |
| **recovery** | A separate minimal boot image for updates and factory reset (5) |
| **Root of trust** | The immutable key/hash fused into the SoC; the anchor of the boot chain (5) |
| **AVB** | Android Verified Boot — AOSP's chain-of-trust implementation (5) |
| **dm-verity** | A signed Merkle hash tree over a read-only partition, re-checked on every block read (5) |
| **Green / Yellow / Orange / Red** | AVB boot states: OEM-verified / custom-key-verified / **unlocked (unverified)** / verification failed (5) |
| **OEM unlocking** | The Developer-options toggle that must be on before the bootloader can be unlocked (5) |
| **`fastboot flashing unlock`** | Unlocks the bootloader (→ orange) and **wipes user data** (5) |

## Root

| Term | Meaning |
|---|---|
| **Root** | A process running as UID 0. A UID, not a mode (6) |
| **su** | "Switch user" binary that escalates a chosen process to UID 0 (6) |
| **Systemless root** | Root that patches the boot ramdisk and overlays changes at runtime, leaving `/system` pristine so Verified Boot still passes (6) |
| **Magisk** | The reference systemless-root implementation (6) |
| **magiskinit** | Replaces `init` as the first program; sets up Magisk, patches sepolicy, then runs the real `init` (6) |
| **Magic mount / bind mount** | How Magisk makes module files appear in place without writing to read-only partitions (6) |
| **`boot.img` / `init_boot.img`** | Where the ramdisk lives. `init_boot` for devices that *launched* on Android 13+; `boot` otherwise (6) |
| **`ro.product.first_api_level`** | Records the launch API level. ≥ 33 (Android 13) ⇒ ramdisk in `init_boot` (6) |
| **Zygisk** | Magisk running module code inside Zygote and app processes. Not a hiding feature by itself (6) |
| **DenyList** | Per-app list; Magisk unmounts its changes for listed apps so they see a stock system. Replaced MagiskHide (6) |
| **resetprop** | Magisk tool that rewrites system properties a detector might read (6, 9) |

## Static tools

| Term | Meaning |
|---|---|
| **jadx** | Decompiles DEX to readable Java. For *reading*; doesn't round-trip (7) |
| **apktool** | Decodes an APK to smali + resources and rebuilds it. For *editing* (7) |
| **smali / baksmali** | Assembler / disassembler for DEX bytecode; smali is DEX's readable textual form (7) |
| **`const/4 vA, 0xN`** | smali: load a small literal into register vA. `0x0` = false, `0x1` = true (7) |
| **`if-eqz` / `if-nez`** | smali: branch if a register is zero / non-zero. Flipping one inverts a test (7) |
| **Repackaging** | The static attack: decompile → edit → rebuild → re-sign → install (7) |

## Dynamic tools

| Term | Meaning |
|---|---|
| **Frida** | Dynamic instrumentation toolkit; injects a JS engine and hooks Java + native code in memory (8) |
| **frida-server** | On-device daemon; needs **root**; exposes frida-core over TCP on `localhost:27042` (8) |
| **frida-gadget** | A shared library embedded in the app; no root, but the APK is repackaged (and re-signed) (8) |
| **GumJS** | The JavaScript engine Frida injects; your hook scripts run in it (8) |
| **Interceptor** | Frida's API for hooking native functions (onEnter/onLeave) (8) |
| **27042** | Frida's documented default TCP port; a common thing anti-Frida checks probe (8) |
| **Xposed** | Original hooking framework (rovo89); replaced `app_process` to load hooks into Zygote. Deprecated on newer Android (8) |
| **LSPosed** | Modern ART-only Xposed reimplementation; a Riru/Zygisk module; hooks via LSPlant; persistent, system-wide (8) |
| **Riru / Zygisk** | Two substrates for loading a module's native code into Zygote (8) |
| **LSPlant** | The ART method-hooking engine LSPosed uses (8) |
| **Shizuku** | A **privilege bridge** (not a hooking framework): relays an app's own Binder calls at root or ADB-shell privilege (8) |
| **Sui** | A Magisk-module variant of Shizuku for rooted devices (8) |
| **Binder** | Android's IPC; tells the server the caller's UID/PID, which is how Shizuku works (8) |
| **TracerPid** | Field in `/proc/self/status`; non-zero means something is `ptrace`-ing the process — a Frida/debugger signal (8, 9) |

## Defences

| Term | Meaning |
|---|---|
| **Root detection** | On-device checks for `su`, manager apps, props, Magisk artefacts. Bypassable by hiding/hooking (9) |
| **Instrumentation detection** | On-device checks for the Frida/LSPosed footprint (port, strings, TracerPid). Bypassable (9) |
| **Signature self-check** | App compares its own cert to an embedded value; catches repackaging. Bypassable by hooking the package-info API (9) |
| **DEX checksum** | App hashes its own code; catches static edits, irrelevant to dynamic ones (9) |
| **RASP** | Runtime app self-protection — bundled native anti-tamper/anti-instrumentation. Raises cost, not a boundary (9) |
| **Obfuscation** | Renaming, string encryption, control-flow flattening, packing. Raises *understanding* cost; not encryption (9) |

## Attestation

| Term | Meaning |
|---|---|
| **TEE** | Trusted Execution Environment — a hardware-isolated world on the SoC; unreachable from the normal world, root included (10) |
| **TrustZone** | ARM's hardware mechanism underpinning most TEEs (10) |
| **StrongBox** | A second KeyMint instance in a dedicated tamper-resistant chip; stronger isolation than the TEE (10) |
| **Keystore** | Android's key store; hardware-backed keys live in and never leave the TEE/StrongBox (10) |
| **KeyMint / Keymaster** | The TEE component guarding Keystore keys. Renamed Keymaster → KeyMint in Android 12 (10) |
| **Key Attestation** | The TEE issues a certificate chain vouching for a key *and* the device state (boot/lock/patch). Since Android 7.0 (10) |
| **Attestation extension** | The field in the leaf certificate carrying the attested properties (10) |
| **ATTESTATION_CHALLENGE** | The verifier's fresh nonce, bound into the attestation to stop replay (10) |
| **Remote Key Provisioning** | Attestation signing keys fetched from Google at runtime instead of factory batches; per-app, rotated (10) |
| **Play Integrity API** | Google's managed verdict service; the token is verified **server-side** (10) |
| **`MEETS_DEVICE_INTEGRITY`** | Genuine certified device; on Android 13+, hardware-backed locked-bootloader proof (10) |
| **`MEETS_BASIC_INTEGRITY`** | Basic checks pass; bootloader may be unlocked (10) |
| **`MEETS_STRONG_INTEGRITY`** | Genuine, certified, recently patched (10) |
| **`appRecognitionVerdict`** | `PLAY_RECOGNIZED` / `UNRECOGNIZED_VERSION` / `UNEVALUATED` — is this Google Play's unmodified binary? (10) |
| **`appLicensingVerdict`** | `LICENSED` / `UNLICENSED` / `UNEVALUATED` — obtained from Play? (10) |
| **SafetyNet Attestation** | The predecessor API. Deprecated 2022, **fully turned down January 2025** (10) |
| **`ctsProfileMatch` / `basicIntegrity`** | SafetyNet's two verdicts; roughly the ancestors of `MEETS_DEVICE_INTEGRITY` / `MEETS_BASIC_INTEGRITY` (10) |
| **CTS** | Compatibility Test Suite — the certification a device matches for `ctsProfileMatch` / device integrity (10) |
| **GMS** | Google Mobile Services. Play Integrity depends on it, so GMS-less devices can't produce a verdict (10) |
