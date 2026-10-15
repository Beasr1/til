# 8. Dynamic attack: instrumentation

## The problem with the static attack

File 07's repackaging works, but it leaves a trace that file 03 catches: the modified APK is
a different file, so it carries a different signature. Any party that checks the certificate
can tell.

So ask the dual question. What if you *don't* touch the file at all — leave the APK
byte-for-byte genuine, signature valid — and instead change the program while it **runs**,
in memory? Now every on-disk check passes (the file really is pristine), every signature
verifies, and the behaviour is still whatever the attacker wants. That is **dynamic
instrumentation**, and it's the other half of the course's central split.

This chapter covers the two instrumentation frameworks you must know — **Frida** and the
**Xposed/LSPosed** family — and then one tool, **Shizuku**, that is constantly mistaken for a
third and is mechanically nothing like them.

## 8.1 Frida: attach, script, rewrite

**Frida** is a dynamic instrumentation toolkit. You write a script — in JavaScript — that
says "when method X is called, do Y instead", Frida injects that script into the target
process, and from then on the process runs your logic at those points
([frida.re/docs](https://frida.re/docs/home/)). It can:

- **Hook Java/Kotlin methods** through its Java bridge — replace a method's implementation,
  change its arguments, or override its return value. (Forcing file 07's `isCompromised()` to
  return `false` is one line of Frida, with no file edit.)
- **Hook native functions** in `.so` libraries through its `Interceptor` API —
  run code on entry and exit, read and change registers and arguments.

So Frida reaches both the DEX/ART side and the native side of file 02's APK.

### The piece that runs your script: GumJS

Frida injects a JavaScript engine, **GumJS**, into the target process; your hooks execute
inside it. Frida's core "packages up GumJS into a shared library that it injects into existing
software, and provides a two-way communication channel for talking to your scripts"
([frida.re/docs/frida-core](https://frida.re/docs/home/)). That channel is why Frida is
*interactive*: you can watch values stream out and send commands back, live.

### Two ways to get Frida into the process

This is the detail defenders must get right, because the two modes have completely different
prerequisites — and they map straight onto the static/dynamic split.

| | **frida-server** | **frida-gadget** |
|---|---|---|
| What it is | A daemon running **on the device** | A shared library **embedded in the app** |
| How it injects | Attaches to a running process (on Android, via `ptrace`-style injection) | The app loads it itself at startup, like any `.so` |
| Needs root? | **Yes** — injecting into another app's process requires it | **No** — it's inside the app already |
| Needs the APK modified? | No — the APK stays genuine | **Yes** — you repackaged the app to add the library (file 07), so it's re-signed |
| Default endpoint | frida-core over TCP on **`localhost:27042`** | listen mode on **`127.0.0.1:27042`** |

([frida.re/docs/modes](https://frida.re/docs/modes/), [frida.re/docs/gadget](https://frida.re/docs/gadget/).)

> **Teacher's aside.** Look at the "needs root" and "needs APK modified" rows together. You
> cannot escape the two-class model *even inside a single tool*. `frida-server` is pure
> **dynamic**: the file stays clean, but you paid with **root** (and therefore an unlocked
> bootloader, file 05). `frida-gadget` avoids root, but only by **statically** repackaging
> the app to embed the library — which re-signs it (file 03) and makes it a modified file
> again. Clean file *or* clean device — you don't get both. That dilemma is the attacker's,
> and file 10 is built to make them pay for it on whichever side they chose.

On the port: **27042** is the documented default for both, and it's a common thing anti-Frida
checks look for. Frida also opens additional per-session ports; their exact numbers aren't
something I'd state as fixed — treat 27042 as the one reliable default and anything beyond it
as version-dependent.

## 8.2 Xposed and LSPosed: hook once, everywhere, forever

Frida is a scalpel you hold in your hand during a session. The **Xposed** family is a
framework you *install*, and it hooks persistently and system-wide — because it works at the
**Zygote** chokepoint (file 04).

The original **Xposed** (by rovo89) replaced `app_process` — the executable that *is* Zygote
— with a modified version that loaded a hook bridge into Zygote at boot. Since every app is
forked from Zygote (file 04), every app inherited the hooks. Modules registered hooks on
specific methods, and the framework applied them to all apps. Original Xposed stopped keeping
pace with newer Android versions.

**LSPosed** is the modern, ART-only reimplementation. It runs as a **Riru** or **Zygisk**
module — so it needs Magisk/root (file 06) — and uses an ART hooking engine (**LSPlant**) to
rewrite method entry points
([LSPosed](https://github.com/LSPosed/LSPosed); the OWASP MASTG describes it as "an extended
version of Zygote that exports APIs for running Java code when a new process is started"
[MASTG](https://mas.owasp.org/MASTG/tools/android/MASTG-TOOL-0149/)). (Riru and Zygisk are
two substrates for getting a module's native code loaded into Zygote; which one isn't
load-bearing here — both put the module inside the process-specialisation path.) Once a method
is hooked, a module can run code before or after it, change its arguments, replace its return
value, or swap the implementation entirely — and because it's all in memory, you undo it by
disabling the module and rebooting.

Frida versus the Xposed family, since you'll choose between them in any analysis:

| | **Frida** | **Xposed / LSPosed** |
|---|---|---|
| Form | Scripts (JavaScript), injected per session | Installed modules (compiled), loaded at boot |
| Reach | The process(es) you attach to | Every app, via Zygote — system-wide |
| Persistence | Ephemeral — gone when you detach | Persistent across launches and reboots |
| Best for | Exploration, live experiments, one-off bypasses | Permanent behaviour changes, set-and-forget |
| Root | `frida-server` needs it; `frida-gadget` doesn't | Needs it (Magisk module) |

## 8.3 Shizuku is not one of these

Here's the tool everyone mis-files. **Shizuku** keeps appearing in the same breath as Frida
and LSPosed, and it belongs in a different category entirely. It does **not** hook, inject
into, or modify another app's code. It is a **privilege bridge**.

The problem Shizuku solves: some system operations need more privilege than an app's UID has,
but less than you'd want to hand out root for — and historically apps got them by shelling out
to `su` and parsing text, which is slow and fragile. Shizuku replaces that
([github.com/RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)).

How it works:

```
  ordinary app ──Binder call──▶  Shizuku service  ──Binder──▶  system_server
  (its own UID)                  (runs as root UID 0,          (checks the CALLER's
                                  or ADB shell UID 2000)        uid/pid, sees Shizuku's)
```

- You start the Shizuku service once, with **root** or — without root — by running it over
  **ADB** (Android 11+ can do this on-device via wireless debugging). With root it runs as
  **UID 0**; via ADB it runs as the **shell** identity, **UID 2000** (file 01).
- An app then makes its privileged request to Shizuku over **Binder** (Android's IPC). Shizuku
  relays it to the system server. Because Binder tells the system server the *caller's* UID,
  and the caller is now Shizuku, the operation runs with Shizuku's privileges, not the app's.

That's the whole mechanism, and notice what's absent: there is no injection, no hooking, no
rewriting of any process. Shizuku lets an app *ask a privileged helper to perform system
operations on its behalf*. It never reaches inside another app. (Sui is a Magisk-module
variant that provides the same bridge on rooted devices.)

> ⚠️ Filing Shizuku with Frida and LSPosed gives you a wrong threat model. The instrumentation
> frameworks **change a target process from the inside**; Shizuku **performs privileged system
> calls for the app that asked**, with the user's explicit consent via ADB or root. "An app
> uses Shizuku" is not "an app is hooking other apps". And Shizuku-over-ADB is *not* root — it
> is bounded by whatever UID 2000 is allowed to do, which is far less than UID 0.

The three tools, classified by what they actually do:

| Tool | Category | Mechanism | Reaches into other apps' code? | Root needed? |
|---|---|---|---|---|
| **Frida** | Instrumentation | Inject GumJS, hook Java + native in memory | **Yes** | `frida-server` yes; `frida-gadget` no (but repackages) |
| **LSPosed** | Instrumentation | Hook ART methods via Zygote, persistently | **Yes** | Yes (Magisk module) |
| **Shizuku** | **Privilege bridge** | Relay the app's own Binder calls at higher privilege | **No** | No (ADB start) or yes (root start) |

## 8.4 Why dynamic attacks matter to the defender

The reason this chapter sits where it does: dynamic instrumentation **defeats every on-disk
defence by construction.** A signature check (file 03), a DEX checksum, a "has my file been
modified" self-test — all of them pass, because the file genuinely hasn't been modified. The
lie is in memory, installed after loading.

And the hook can target the *defence itself*. A root check, a Frida-detector, an integrity
test — each is a method in the app, and a method is exactly what Frida and LSPosed exist to
rewrite. This is why file 09 is honest that on-device detection is an arms race: the detector
runs in the same process as the thing instrumenting it, below which the attacker sits. The
only prerequisite the attacker still can't wish away is the one underneath it all — root, and
therefore an unlocked bootloader — which is what file 10 reports from hardware the hooks can't
reach.

## Check yourself

1. An app verifies its own signature at startup and refuses to run if it's wrong. A Frida
   user bypasses the whole control without the self-check ever failing. Explain how both
   things are true at once.
2. `frida-server` and `frida-gadget` achieve the same hooking. State the prerequisite each
   one needs that the other doesn't, and classify each as more "static" or more "dynamic" in
   the course's sense.
3. Why does hooking via LSPosed reach *every* app on the device while a given Frida session
   typically affects only one process? Answer in terms of file 04.
4. A detector scans for an open TCP port 27042 and concludes "no Frida present". Give two
   different reasons this conclusion is unsafe.
5. A colleague says "we should block Shizuku — it's a hooking tool like Frida". Correct the
   category error: what does Shizuku actually do, and what can it *not* do that Frida can?
6. Shizuku started over ADB runs as UID 2000, not UID 0. Give one concrete thing it therefore
   cannot do that a root-started Shizuku could, tying back to file 01.
7. Stand back: dynamic instrumentation beats every on-disk integrity check, yet it still has
   one hard prerequisite on most devices. Name it, trace it back through two earlier chapters,
   and say why that's the thread file 10 pulls on.
