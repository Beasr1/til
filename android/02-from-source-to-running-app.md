# 2. From source to running app

## The problem this chapter removes

You write an app in Kotlin. Something ends up on the device and runs. Between those two
points sit four or five transformations, and if you don't know what they produce, two later
facts will seem like magic: *why an attacker can read your code back out of a shipped app*
(file 07), and *why editing one file inside the app breaks its signature* (file 03). This
chapter is the pipeline, end to end, with an eye on what each stage leaves recoverable.

## 2.1 Source to bytecode

Android apps are written in **Kotlin** or **Java**. Both compile first to ordinary **Java
bytecode** — the `.class` format the JVM has used for decades. That is an intermediate stop,
not what ships.

A tool called **d8** (or **r8**, its optimising sibling) then converts that Java bytecode
into **DEX** — *Dalvik Executable* — bytecode. DEX is Android's own bytecode format, designed
to be compact and to run on Android's runtime rather than a desktop JVM
([developer.android.com/tools/d8](https://developer.android.com/tools/d8)). All of your
compiled code lands in one or more files named `classes.dex`, `classes2.dex`, and so on.

r8 does one more job worth naming now because file 09 leans on it: **shrinking and
obfuscation**. It strips unused code and can rename classes, methods and fields to
meaningless short names (`a`, `b`, `c`). That renaming is the main thing standing between an
attacker and your original identifiers — and it is cosmetic, not cryptographic.

```
   Kotlin / Java  ──javac/kotlinc──▶  Java bytecode (.class)  ──d8/r8──▶  DEX (classes.dex)
        │                                                                      │
   human-readable                                                   compact register-based
     source                                                          bytecode that ships
```

> **Teacher's aside.** DEX is *bytecode*, not machine code, and it is high-level bytecode:
> it still carries class names, method names, field names and type signatures, because the
> runtime needs them to link things up. That is exactly why a shipped app decompiles back to
> something close to the original source (file 07). Compiling to DEX is not a one-way
> function the way hashing is — it's more like a reversible re-encoding. Obfuscation makes
> the result ugly; it does not make it secret.

## 2.2 What an APK actually is

The build packages everything into an **APK** — an *Android Package*. An APK is not a special
binary format. It is a **ZIP archive** with a known layout. Rename one to `.zip` and your
unzip tool opens it.

Inside:

| Entry | What it is |
|---|---|
| `AndroidManifest.xml` | The app's declaration: package name, components, permissions, minimum OS version. Stored as *binary* XML, not text |
| `classes.dex` (+ `classes2.dex`, …) | Your compiled code, as DEX bytecode |
| `resources.arsc` | Compiled resources table — strings, dimensions, styles |
| `res/` | Resources: layouts, drawables, icons |
| `assets/` | Raw files the app bundles and reads at runtime |
| `lib/<abi>/*.so` | **Native** libraries — actual compiled machine code, per CPU architecture (`arm64-v8a`, `x86_64`, …) |
| `META-INF/` | The signature files (file 03). For the old v1 scheme, this is where signatures live |

Two of those entries matter disproportionately for security:

- **`classes.dex`** is your logic, in a form that reverses easily. If a check is written in
  Kotlin, it's in here, and it's readable.
- **`lib/*.so`** is native code — C, C++, Rust compiled to machine instructions. It does
  *not* decompile back to clean source; reversing it means reading disassembly. That makes
  native code a meaningfully higher bar for an attacker, which is why security-sensitive
  checks are often pushed down into it (file 09).

> ⚠️ The APK being a plain ZIP is not a quaint detail — it is the start of the static attack.
> Anyone can unzip it, swap a file, and zip it back up. The *only* thing that notices is the
> signature (file 03). Without signing, "edit the app" would be as easy as editing any ZIP.

## 2.3 How the device runs DEX: Dalvik, then ART

DEX bytecode doesn't run directly on the CPU — something has to execute it. That something
is the **runtime**, and Android has had two.

**Dalvik** was the original: a virtual machine that interpreted DEX bytecode and, from
Android 2.2, used a **JIT** (just-in-time) compiler to turn hot code into machine code *while
the app ran*. Every launch paid the compilation cost again.

**ART** — the **Android Runtime** — replaced it. ART shipped as an opt-in preview in Android
4.4 (KitKat) and became the default and only runtime from **Android 5.0 (Lollipop)**. AOSP
describes ART as "the managed runtime used by apps and some system services", executing the
DEX format ([source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime)).
The two runtimes share the DEX format, so an app built for Dalvik generally runs on ART.

ART's original trick was **AOT** (ahead-of-time) compilation. At install time it ran a
tool called **dex2oat** that compiled the whole app's DEX to native code once, so launches
were fast — at the cost of slow installs and more storage
([source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime)).

From **Android 7.0 (Nougat)**, ART went **hybrid**: it no longer compiles everything at
install. It interprets the app first, uses a JIT to compile frequently run ("hot") methods,
*profiles* which methods those are, and later — when the device is idle and charging —
compiles just those ahead of time using the profile. Fast installs, fast steady-state, best
of both. (The exact KitKat/Lollipop framing above is the widely documented history; AOSP's
current runtime page states the dex2oat/AOT mechanism but not the per-version timeline, so
treat the version numbers as the community-standard account rather than a single AOSP
sentence.)

The terms, untangled, because they get mixed up constantly:

| Term | What it is |
|---|---|
| **DEX** | The *bytecode format* that ships in the APK. A thing on disk |
| **Dalvik** | The old *runtime* that ran DEX. Retired after Android 4.4 |
| **ART** | The current *runtime* that runs DEX |
| **AOT** | Compiling bytecode → machine code *before* running (ART's `dex2oat`) |
| **JIT** | Compiling bytecode → machine code *while* running, for hot paths |

## 2.4 Why all of this sets up the attacks

Trace the security consequence of each stage:

- Your logic ships as **DEX**, which reverses to readable code — so an attacker can *find*
  your checks (file 07).
- The APK is a **ZIP** — so they can *change* those checks by swapping `classes.dex`.
- ART loads DEX by class and method name — so an in-memory attacker can *intercept* a method
  by name without touching the file at all (file 08). The same names that let ART link your
  code let Frida and LSPosed hook it.
- Native `.so` code is machine code — a higher wall, but still just code on a device the
  attacker owns.

Everything downstream is a consequence of *what form your code takes by the time it's on the
device*. It takes the most reversible form there is, and that is the environment every
defence in file 09 has to operate in.

## Check yourself

1. You rename a shipped `app.apk` to `app.zip` and it opens. What does that single fact
   already tell you about the difficulty of *modifying* an app versus the difficulty of
   *getting the modification accepted*?
2. An attacker wants to read your login logic. It's written in Kotlin. Roughly how much work
   is it for them, and why — in terms of what `classes.dex` actually contains?
3. You move the same login logic into a Rust library compiled to `lib/arm64-v8a/libauth.so`.
   What changes for the attacker, and what does *not*?
4. Someone says "ART compiles the app at install, so the DEX isn't really what runs — you
   can't hook it by method name." Where is the hole in that reasoning, given Android 7.0+
   behaviour?
5. r8 renamed all your classes to `a`, `b`, `c`. A reviewer calls this "encryption of the
   source". Why is that the wrong word, and what has actually been achieved?
