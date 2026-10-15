# 9. Defences and the arms race

## The problem — and the uncomfortable answer up front

Files 06–08 showed that root and instrumentation get under every wall the platform builds for
an app. So apps try to defend *themselves*: detect the bad environment, detect the tampering,
and refuse to run. This chapter is the catalogue of those defences — and it is honest about
their ceiling, because pretending otherwise is how teams waste quarters.

The ceiling, stated now so nothing below is a surprise:

> **Every defence in this chapter runs *inside* the process on a device the attacker controls.
> The attacker is therefore positioned *below* it and can edit its code, its inputs, or its
> outputs. None of these is a security boundary. They raise cost and buy time — that is their
> whole, real, limited value.**

Keep that frame. The techniques are worth doing; they are not worth *trusting*.

## 9.1 Root detection

The app looks for evidence that the device is rooted. Common signals, and how each is beaten
([OWASP MASTG anti-tampering](https://mas.owasp.org/MASTG/)):

| Signal the app checks | How root defeats it |
|---|---|
| An `su` binary on common paths | Systemless root never puts `su` in `/system` (file 06); the path is empty |
| Known superuser/Magisk manager package installed | Manager apps can be hidden or renamed |
| `test-keys` in the build tag, insecure props (`ro.debuggable`) | Magisk's `resetprop` can rewrite the property the app reads |
| Magisk artefacts — `/data/adb`, its sockets | The DenyList unmounts and hides them for the checking app (file 06) |
| Anything at all | The detection method is itself a method — hook it to return "not rooted" (file 08) |

The last row is the general defeat: whatever the check reads, it reads *through APIs root sits
below*. You're asking the compromised system to incriminate itself.

## 9.2 Instrumentation detection

The app looks for a Frida/LSPosed-shaped environment. Typical signals:

| Signal | How it's beaten |
|---|---|
| TCP port **27042** open (file 08) | Run `frida-server`/gadget on a non-default port |
| Strings like `frida`, `gum-js-loop`, `gadget` in the process's memory maps (`/proc/self/maps`) | Rename the artefacts; strip the strings from a custom build |
| `TracerPid` non-zero in `/proc/self/status` (something is `ptrace`-ing you) | Clear or hook the read; attach by means that don't show there |
| Known injected library names loaded | Rename; load differently |

Same structural problem: the detector is a function, and a function is what the instrumentation
framework exists to rewrite. A Frida script's first job is often to neutralise the very
detection the app runs.

## 9.3 Integrity and signature self-checks

These target the **static** attack (file 07) specifically, and they're the most *worthwhile*
of the on-device controls because they catch the cheapest attack:

- **Signature self-check.** At runtime the app asks the system for its own signing certificate
  and compares it to a value baked into the code. A repackaged app was re-signed with the
  attacker's key (file 03), so the certificate differs — caught.
- **DEX checksum.** The app hashes its own `classes.dex` (or the whole APK) and compares to an
  embedded expected hash. A static edit changes the bytes — caught.

Why they still lose:

- Against a **static** attacker: they can edit the *expected* value too, or edit out the
  comparison (it's just more smali, file 07).
- Against a **dynamic** attacker: the file is unmodified, so the check passes honestly — and
  if it didn't, they'd hook the API that returns the signature (e.g. the package-info call) to
  hand back the original (file 08).

> **Teacher's aside.** There's a subtle self-defeating property to shipping detection logic in
> the client: **it tells the attacker exactly what you check.** Every signal you test is, to a
> reverse engineer reading your DEX (file 07), a checklist of what to spoof. A client-side
> check is simultaneously a defence and a disclosure of that defence. This is why the better
> architectures move the *decision* server-side and keep only *signal-gathering* on the client
> — and why the signal that can't be spoofed has to come from below the attacker (file 10).

## 9.4 Native checks, and why they're only a bigger speed bump

A recurring instinct: move the sensitive check from Kotlin into a native `.so` (file 02),
because native code doesn't decompile to clean source and native hooking is more effort than
Java hooking. That instinct is *correct about the cost* and *wrong about the category*:

- It genuinely raises the bar. Reading disassembly is slower than reading jadx's Java; Frida's
  `Interceptor` on a native function is more fiddly than its Java bridge.
- It changes nothing fundamental. The `.so` still runs on the attacker's device, so it can
  still be disassembled, patched on disk, or hooked in memory. Commercial RASP (runtime app
  self-protection) products bundle native checks like these, and they too are bypassed — they
  just cost the attacker more.

The honest summary: native checks convert a one-hour bypass into a one-day bypass. For
deterring casual and scripted attackers that can be worth real money. It is not a boundary.

## 9.5 Obfuscation

Obfuscation makes the code *harder to understand*, not harder to run:

- **Identifier renaming** (r8/ProGuard): `isCompromised` becomes `a`
  ([developer.android.com/build/shrink-code](https://developer.android.com/build/shrink-code)).
- **String encryption**: literal strings (URLs, error messages that reveal logic) are
  decrypted at runtime instead of sitting in plain sight.
- **Control-flow flattening / packing**: the structure is mangled, or the real DEX is
  unpacked at runtime by a loader.

All of it raises the effort to *locate* a target (file 07's "find the check" step). None of it
changes that the code must ultimately execute in a form the CPU runs, which means the attacker
can observe and reach that form. Obfuscation is not encryption: the device has to be able to
run the code, so the device — and the attacker holding it — can get at it.

## 9.6 The false-positive cost nobody budgets for

One failure mode that hurts defenders more than attackers: **over-aggressive detection flags
real users.** People run custom ROMs, legitimately rooted phones, unusual-but-benign
devices. Tuning root/instrumentation detection to catch more attackers inevitably catches
more of these, and they are paying customers, not adversaries. The attacker, meanwhile, is
the one person who *can* make their device look clean. So aggressive on-device detection tends
to tax exactly the wrong population. Budget the support cost and the churn, not just the
"attacks blocked" number.

> ⚠️ Don't let a detection control *leak why it failed*. If the app shows "root detected" or
> behaves differently per signal, you've handed the attacker a precise oracle to iterate
> against (flip one thing, see if the message changes). Fail the same way for every cause,
> and preferably fail *late* and server-side so the attacker can't locally A/B their bypass.

## 9.7 How to think about all of this

The whole chapter reduces to one decision rule:

- On-device defences are **economic**: they raise the attacker's cost and the defender's
  support cost, both. Spend on them in proportion to how much casual/scripted abuse you face,
  and no further.
- They are **not** a boundary. Do not gate anything that must be *correct* against a determined
  attacker on their own device behind an on-device check alone.
- The one thing that actually holds is a statement made by **hardware the attacker can't
  rewrite**, verified by a **server they don't own**. That's file 10, and the reason it holds
  is precisely the reason nothing in this chapter does: it is not running inside the process
  the attacker controls.

## Check yourself

1. State the one structural property shared by root detection, Frida detection, and DEX
   checksums that guarantees each is bypassable on a controlled device. Why does adding a
   fourth check not fix it?
2. A DEX-checksum self-test catches a repackaged app but is useless against a Frida user.
   Explain why the same control succeeds against one attack class and is irrelevant to the
   other.
3. "We moved the licence check into native code, so now it's secure." Rewrite that claim into
   one that's actually true, in terms of attacker cost rather than impossibility.
4. Why is shipping a long list of client-side detections partly self-defeating? What does the
   attacker gain simply by reading them?
5. A team tightens root detection and sees support tickets spike from users on custom ROMs,
   while sophisticated fraud is unchanged. Explain the asymmetry: why did tightening hit the
   wrong group?
6. Given everything here is bypassable, make the positive case for doing *some* of it anyway —
   and state precisely where the line is between "worth it as deterrence" and "mistaking it
   for a boundary".
