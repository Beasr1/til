# 1. The Android platform, from zero

If you've shipped Android apps, skim — but §1.2 is the load-bearing idea for the whole
course, so don't skip it. If you've only ever typed `adb` a few times, this is the chapter
that makes every later one make sense.

## The problem that made the design

You have one device. On it: a banking app, a game, a note-taking app you downloaded from a
stranger. The banking app holds a session token in a file. What stops the game — or the
app from the stranger — reading that file?

On a desktop, nothing much does by default: every program you run runs as *you*, with your
permissions, and can read everything you can read. That model is fine when you install only
software you trust. It is catastrophic for a phone, where you install fifty things a year
from fifty authors and expect them not to rob each other.

So Android needed isolation between apps to be the *default*, enforced by something an app
cannot talk its way around. It did not invent a new mechanism for this. It reached for the
oldest one in the building.

## 1.2 An app is a Linux user

Android runs on the **Linux kernel**. Everything above it — the Java-like runtime, the apps,
the home screen — is ordinary Linux user space. And Linux already has rock-solid isolation
between *users*: the kernel will not let one user's process read another user's files unless
the permission bits say so. That is a 1970s operating-system feature, battle-tested for
decades.

Android's move: **give every app its own Linux user ID (UID)**, assigned at install time. The
banking app is one user; the game is another. The kernel then enforces the wall between them
for free, with the same code that keeps users apart on a server.

> **Teacher's aside.** This is the single most important sentence in the course: *the app
> sandbox is not an Android security layer bolted on top — it is the Linux user boundary,
> with one UID per app.* When you later read "the app runs in a sandbox", translate it in
> your head to "the app runs as its own user and the kernel stops it stepping outside".
> Everything the attacker does in this course is, at bottom, an attempt to get out of that
> user boundary or to become a user (root) that the boundary doesn't apply to.

Concretely, each app gets a private data directory — `/data/data/<package>` — owned by its
UID, with permissions that exclude every other UID. App A literally cannot `open()` app B's
files; the kernel returns "permission denied" before any app code gets a say. AOSP documents
this as the **Application Sandbox** ([source.android.com/docs/security/app-sandbox](https://source.android.com/docs/security/app-sandbox)).

Since Android has layered a second mechanism on top: **SELinux**, a mandatory access
control system that constrains even processes that *do* share a UID, and constrains root
itself. The UID boundary is the discretionary layer; SELinux is the mandatory one. For this
course the UID boundary is the one to hold in mind — it's what repackaging and root are
really about.

## 1.3 So what *is* "an app", precisely?

Three things travel together, and keeping them distinct will save you later:

| Thing | What it is | Set by |
|---|---|---|
| **Package name** | A unique string like `com.example.wallet`. The app's identity *to the user and the store* | The developer, in the manifest |
| **Signing certificate** | The key the APK is signed with. The app's identity *to the system* (file 03) | The developer's signing key |
| **UID** | The Linux user the app's processes run as. The app's identity *to the kernel* | The system, at install |

The package name is just a label — anyone can *claim* `com.example.wallet` by typing it into
their manifest. What the system actually trusts is the **signing certificate** (file 03).
That distinction is the entire reason repackaging attacks are a thing and the entire reason
they're contained, so flag it now.

An app at rest is an **APK** file (file 02). An app running is one or more **processes**
(file 04), executing the app's code under the app's UID.

## 1.4 Permissions: the sandbox has doors

If every app were sealed in its own UID with no way out, nothing could use the camera, the
network, or your contacts — all of which are shared resources owned by the system or by
other apps. So the sandbox has controlled doors: **permissions**.

An app declares the permissions it wants in its manifest. Some are granted automatically
(the `INTERNET` permission, for instance); the dangerous ones (camera, location, contacts)
require the user to agree at runtime. Under the hood many permissions map to **group IDs**:
being granted network access means your UID is added to a group the kernel checks. So even
permissions, at the bottom, resolve to the same Linux primitives
([developer.android.com/guide/topics/permissions](https://developer.android.com/guide/topics/permissions/overview)).

The teaching point: permissions are about reaching *shared* resources. They are not what
stops app A reading app B's private files — the UID boundary does that, with no permission
involved on either side.

## 1.5 The shape of the system: two worlds

One more split to put in place now, because the boot and root chapters live on it.

```
┌──────────────────────────────────────────────┐
│  system partition(s)  — the OS itself         │   read-only in normal operation,
│  /system, /vendor, the framework, ART, …      │   shipped by the manufacturer, verified
├──────────────────────────────────────────────┤
│  data partition  — /data                      │   writable: every app's private dir,
│  /data/data/<package>, user files, settings   │   user files, settings
└──────────────────────────────────────────────┘
```

The OS lives on partitions mounted **read-only** and protected by Verified Boot (file 05).
Your apps and your data live on `/data`, which is writable and encrypted. This separation is
why a factory reset (wiping `/data`) leaves a working phone, and why gaining write access to
the read-only system world is exactly what "rooting" has to achieve (file 06).

## 1.6 adb, and the user you are when you use it

You already know a few `adb` commands, so name the tool properly. **adb** — the Android Debug
Bridge — is a host-side program that talks to a daemon (`adbd`) on the device. `adb shell`
drops you into a shell *on the device*.

The part worth internalising: that shell does **not** run as root, and it does **not** run as
any app. It runs as a dedicated user called `shell`, **UID 2000**, which sits above an app's
privileges but well below root. It can do developer things — list packages, read logs,
install APKs, poke at many system services — but it still cannot read an app's private
`/data/data` files, because UID 2000 is not UID of the app and not root.

> ⚠️ "I can `adb shell` into the device" is not "I have root". A great deal of confusion —
> and one whole tool, Shizuku (file 08) — lives in the gap between the `shell` user's powers
> and root's. When you read a command working over `adb` but failing when an app tries it,
> the reason is almost always this UID difference.

## 1.7 What the rest of the course attacks

With the platform in place, here's the map of where each later chapter pushes:

- Repackaging (file 07) forges the **package** but cannot forge the **signature** (file 03).
- Root (file 06) escapes the **UID boundary** by becoming UID 0.
- Instrumentation (file 08) works *inside* the app's own UID and process, rewriting it in
  memory.
- Verified Boot and attestation (files 05, 10) move the question off the device entirely,
  because none of the on-device walls hold once the attacker is root.

## Check yourself

1. The game app tries to read the banking app's session-token file and gets "permission
   denied". Which mechanism denied it, and at what point in the attempt — before or after
   the banking app's code could react?
2. Two developers both put `com.example.wallet` as the package name in their manifests. Does
   the system treat their apps as the same app? What *does* it use to tell them apart?
3. You can read an app's log output over `adb logcat`, but an attempt to `cat` that same
   app's `/data/data/<package>/files/secret` over `adb shell` fails. Why does one work and
   the other not, given both are "me on the device"?
4. A colleague says "we granted ourselves the `INTERNET` permission, so now we can read other
   apps' data too." Untangle the two different things they've confused.
5. Why does gaining *root* require getting at the read-only system world specifically, rather
   than just writing to `/data` where everything interesting seems to live?
