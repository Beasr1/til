# 7. Static attack: reverse engineering and repackaging

## The problem this realises

Files 02 and 03 set it up: the app ships as a ZIP full of reversible DEX bytecode, and the
only thing guarding it is a signature the attacker can't forge but *can* replace with their
own. This chapter is the attack that follows directly — the canonical **static** attack:
take the APK off disk, read it, change it, put it back. No root required, no device
compromise. Just a file and a toolchain.

The reason to study it in a defender's course: an assessment finding of the form *"the app
can be decompiled, modified and recompiled to bypass control X"* is describing exactly this
pipeline, and you can't reason about the fix without knowing the steps.

## 7.1 The pipeline, and the two tools that do different jobs

```
  app.apk ──jadx──▶  readable Java      (READ: understand the app, find the check)
     │
     └────apktool d──▶ smali + resources (EDIT: change the check — smali rebuilds reliably)
                           │
                           │ edit the .smali
                           ▼
                      apktool b ──▶ unsigned app.apk
                                        │
                                        ▼
                                   zipalign ──▶ apksigner (attacker's key) ──▶ install
```

The single most useful thing to understand here is **why there are two decompilers**, because
newcomers waste days on this:

| Tool | Produces | Use it to | Round-trips? |
|---|---|---|---|
| **jadx** | Approximate **Java** source | *Read* and understand — it's close to the original and pleasant to follow | **No.** jadx's Java usually won't recompile |
| **apktool** | **smali** (DEX assembly) + decoded resources + manifest | *Edit* — smali is a faithful 1:1 view of the bytecode and rebuilds cleanly | **Yes** |

([jadx](https://github.com/skylot/jadx); [apktool](https://apktool.org)). So the professional
flow is: read in Java with jadx to find *what* to change and *where*, then make the actual
change in smali and rebuild with apktool. Trying to edit jadx's Java and recompile it is the
beginner trap.

**smali** (and its inverse **baksmali**) is the assembler/disassembler for DEX bytecode
([smali/baksmali, JesusFreke](https://github.com/JesusFreke/smali)). apktool uses it under the
hood. smali is to DEX what assembly is to machine code: a readable, exact, reversible textual
form of the same instructions.

## 7.2 A worked example: neutering a boolean check

Take a conceptual control — an app that refuses to run in a modified environment. In Kotlin
it might read, roughly:

```kotlin
if (integrity.isCompromised()) {
    finish()      // bail out
}
```

The attacker's goal is to make `isCompromised()` always report `false`. They open the app in
jadx, find the class, and confirm the method's shape. Then they decode with apktool and open
the corresponding `.smali` file. The method disassembles to something like:

```smali
.method public isCompromised()Z        # ()Z  → takes nothing, returns a boolean
    .registers 2
    # ... real logic that inspects the environment ...
    # ... and leaves 1 (true) or 0 (false) in v0 ...
    return v0
.end method
```

A few smali facts make the edit legible
([Dalvik bytecode reference](https://source.android.com/docs/core/runtime/dalvik-bytecode)):

- The `Z` at the end of `isCompromised()Z` is the return type — `Z` means boolean (`V` void,
  `I` int, and so on).
- Registers are `v0`, `v1`, … (locals) and `p0`, `p1`, … (parameters; `p0` is `this`).
- A boolean is just an int: **`0` is false, `1` is true**.
- `const/4 v0, 0x0` loads the literal `0` into `v0`. `return v0` returns it.

So the attacker deletes the real logic and replaces the body with two instructions:

```smali
.method public isCompromised()Z
    .registers 2
    const/4 v0, 0x0     # load 0  (= false)
    return v0           # always return false
.end method
```

The method now returns `false` unconditionally. The `if` in the Kotlin never fires. The
control is gone — not disabled by a flag, *removed from the program*. (The equivalent move at
a call site is to flip the branch — swap an `if-eqz` for an `if-nez` so the test's outcome is
inverted — but zeroing the source method is the cleaner illustration.)

Then: `apktool b` to rebuild, `zipalign`, and `apksigner` with a key the attacker generated.
Install, and the app runs believing its environment is clean.

> **Teacher's aside.** Notice what made this surgical: the shipped artifact still carried the
> method name `isCompromised` and its exact logic as editable instructions. Nothing was
> secret — obfuscation (file 09) would have renamed the method to `a` and made it *harder to
> find*, but once found it's edited the same way. This is the concrete meaning of file 02's
> claim that compiling to DEX isn't a one-way function. A control written as app code is a
> control the attacker can read and delete, because you shipped them the code.

## 7.3 Why re-signing is unavoidable — and what that costs the attacker

The moment apktool rebuilds the APK, the original signature is gone (the bytes changed, so
file 03's v2+ whole-file hash no longer matches). The rebuilt APK is **unsigned**, and
Android refuses to install an unsigned APK. So the attacker *must* re-sign — and they don't
have your key, so they sign with their own.

This is where file 03's containment bites, and it's worth stating as the balance sheet of the
whole static attack:

| The attacker got | The attacker paid |
|---|---|
| A working modified app that runs the way they want | A **different signing certificate** — so it's a different app identity |
| The on-device control removed entirely | It can't update over the real app, can't touch its data, can't claim its signature permissions (file 03) |
| No root, no device modification needed | The new certificate won't match what a **server** expects (file 10) |

So for *"I want to run a cracked copy on my own phone"*, the static attack basically wins, and
there's little the app alone can do about it. For *"I want the modified app to be accepted by
the backend as the genuine one"*, it loses at the server — provided the server actually checks
app identity rather than trusting the client. That gap is the entire argument for file 10.

> ⚠️ Two classic pipeline failures, both from file 03's rules. First: installing the rebuilt
> APK fails with a no-certificate error — you forgot to re-sign. Second: it fails signature
> verification after signing — you ran `zipalign` *after* `apksigner` instead of before, so
> the alignment changed bytes the signature already covered. Build → align → sign, in that
> order.

## 7.4 The defender's reading

What this chapter should change about how you think:

- A control implemented purely as app logic is **readable and removable** by a static
  attacker. Writing it in native code (file 02) raises the effort; it doesn't change the
  category.
- The signature is what keeps the modified copy from *impersonating* the original — but only a
  party that checks the signature can benefit, and the attacker's own device won't.
- Therefore the question a defender should ask of any anti-tamper control is not "can it be
  bypassed on the device" (yes, always, eventually) but "**does anything the attacker doesn't
  control get to see the result**" — which is the pivot into detection (file 09) and
  attestation (file 10).

## Check yourself

1. You want to *understand* an unfamiliar app's licensing logic, then *change* it. Which tool
   do you use for each task, and why is using the wrong one for the second task a dead end?
2. In the smali `.method public isValid()Z`, what does the trailing `Z` mean, and what two
   instructions would force the method to always return `true`?
3. An attacker edits one smali instruction, rebuilds, and the install fails before the app
   even starts — "no certificates". What step did they skip, and why is it not optional?
4. They fix that, re-sign with their own key, and it installs and runs. The app's backend
   still rejects it. Which property changed that the backend can detect, and why couldn't the
   attacker have avoided changing it?
5. A teammate proposes defeating repackaging by "encrypting the APK". Given the APK is a ZIP
   the OS must be able to open and ART must be able to run, what's the fundamental problem
   with that idea, and what is the realistic goal of obfuscation instead?
6. Rank these by how much they slow a static attacker, and say why each is only a *delay*: (a)
   renaming methods with r8, (b) moving the check into a native `.so`, (c) adding a second,
   duplicate check elsewhere in the DEX.
