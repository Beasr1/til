# 4. Zygote and the process model

## The problem that made it

Every Android app runs on ART, and ART needs the whole application framework loaded before a
single line of your code runs — thousands of classes, the resource tables, the graphics
stack. If each app launch started a fresh runtime and loaded all of that from scratch, every
tap on an icon would stall for a noticeable beat, and every running app would hold its own
private copy of framework code that is byte-for-byte identical to every other app's.

That's wasteful twice over: slow to start, and duplicated in memory. Android's fix to both is
a single design decision, and that decision happens to hand code injection its best foothold
on the platform. Worth understanding properly.

## 4.1 One warm parent, many forks

At boot, `init` starts a process called **Zygote**. Zygote boots ART, then **preloads** the
common framework classes and shared resources into its own memory — the one-time expensive
work. Then it does nothing: it sits on a UNIX domain socket and waits
([AOSP `ZygoteInit`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)).

When you launch an app, the system doesn't start a process from zero. It asks Zygote to
**fork** — to clone itself. The child is an exact copy of the warm, framework-loaded parent,
produced in a moment. The child then **specialises**: it sets itself to the app's UID
(file 01), drops the privileges it doesn't need, loads the app's APK, and calls into the
app's entry point.

```
          init
            │ starts
            ▼
        ┌────────┐   preloads ART + framework classes + resources, then waits
        │ Zygote │◀──────────────────────────────────────────────────────┐
        └────────┘                                                         │
            │ fork()                                                       │ fork() again
            ▼                                                              │   per app launch
   ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
   │  system_server   │        │   app process A  │        │   app process B  │
   │ (first fork)     │        │  UID = A, loads  │        │  UID = B, loads  │
   │                  │        │  A's classes.dex │        │  B's classes.dex │
   └──────────────────┘        └──────────────────┘        └──────────────────┘
        all three already contain the preloaded framework, shared copy-on-write
```

The very first thing Zygote forks is **`system_server`** — the process that hosts most of the
system services, including the one that decides when to launch apps. After that, every app
process you ever run is a fork of Zygote.

## 4.2 Why forking (and not exec) is the whole point

Here's the part that's unusual if your background is server-side Linux. The normal way to
start a new program on Linux is **fork + exec**: fork the current process, then immediately
`exec()` to *replace* the child's memory with a fresh program image. The child keeps nothing
of the parent.

Zygote forks but deliberately does **not** exec. The child keeps the parent's entire memory
image — which is the whole point, because that image is the preloaded framework. Two
consequences:

- **Launch is cheap**, because the framework is already there in the fork.
- **Framework memory is shared** via **copy-on-write**: `fork()` doesn't physically duplicate
  pages, it maps the child onto the parent's pages and only makes a private copy when someone
  *writes* to one. So one physical copy of the framework's read-only code backs every app at
  once.

> **Teacher's aside.** The mental model to carry is *every app is a photocopy of one warm
> template*. That explains the speed and the memory sharing — but now ask the security
> question it begs: if every app is a copy of one process, what happens to anything you can
> get *into that process before it forks*? It ends up in every copy. Zygote is a single
> chokepoint that every app on the device passes through at birth. Control the template and
> you control every app made from it. That is not a bug; it's the direct shadow of a design
> chosen for performance.

## 4.3 The gift to code injection

Put the chokepoint together with file 02's fact that ART links code by name, and the attack
surface for *dynamic* attacks (file 08) is laid out in advance:

- **Inject once, affect everything.** Code that runs in Zygote — or in the fork-and-specialise
  path each child runs — runs inside *every* app process. A hook installed there is
  automatically system-wide and persistent across launches. This is precisely how the Xposed
  family and LSPosed get their reach (file 08), and how Magisk's **Zygisk** runs module code
  inside app processes (file 06): both operate at the Zygote boundary.
- **Per-process injection is the fallback.** You don't have to work at Zygote. You can also
  attach to one already-running app process and rewrite it in place — which is Frida's usual
  mode (file 08). That affects only the process you attached to, which is why it's
  per-target rather than global.

The difference between those two is exactly the difference between a persistent framework
(hook the template, catch all apps forever) and an interactive tool (attach to this app, this
run).

## 4.4 What stops just anyone doing this

If Zygote injection is so powerful, why isn't every app doing it to every other app? Because
getting code into Zygote means modifying a **system** process — or the `app_process`
executable that is Zygote, or the boot ramdisk that launches it. All of that lives in the
read-only, Verified-Boot-protected system world (files 01, 05). An ordinary app, sealed in
its UID, cannot touch any of it.

So Zygote-based hooking frameworks require **root** (file 06), and root requires a boot chain
that will run modified system code (file 05). The chokepoint is powerful *and* gated — which
is why the rest of the course spends so long on how that gate is opened (root) and how a
server can tell whether it has been (attestation, file 10).

> ⚠️ Don't conclude "apps can hook each other". By default they can't — the UID boundary and
> the read-only system partition both stand in the way. Zygote injection is a capability of
> the *platform owner* (root), not of a peer app. The threat model for the whole course is a
> user who has rooted their *own* device, not app-on-app attack.

## Check yourself

1. Launching an app is fast even though ART must have the entire framework loaded first.
   Explain how the Zygote design makes that possible, naming the two distinct savings it
   gives (time and memory).
2. On a normal Linux server, starting a program is fork **then** exec. Zygote forks but
   skips the exec. What would break about Android's design if Zygote *did* exec in the child?
3. An attacker gets a few lines of code running inside Zygote before it forks. Which app
   processes are affected, and why — contrast this with attaching to a single running app.
4. Copy-on-write means the framework's code pages are shared across all apps until someone
   writes to one. Why is that detail reassuring for *code* integrity but not a security
   boundary you'd rely on?
5. Zygote injection is extremely powerful, yet a normal app can't do it. Name the two
   platform protections that stand between an app and the Zygote process, and say which later
   chapter is about removing each.
