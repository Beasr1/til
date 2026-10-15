# 6. Root

## The problem — and why it's harder than it sounds

File 01 said the sandbox *is* the Linux UID boundary. The way through a boundary like that
has been the same on Unix since the 1970s: become the one user the boundary doesn't apply to
— **root**, UID 0. So "rooting" an Android phone means arranging for a process to run as, or
escalate to, UID 0.

But files 04 and 05 made that awkward. Running privileged code system-wide means a boot that
will launch it, and the obvious way to install `su` — writing it into the system partition —
is blocked twice over on a modern device: the system partition is mounted read-only, and
**dm-verity** (file 05) detects any change to it at read time, so the device breaks. So the
interesting question isn't "what is root" — it's *how do you hold root without tripping
Verified Boot*. The answer is the central idea of this chapter: **systemless** root.

## 6.1 What root actually is

Strip the mystique. **Root is a UID, not a mode.** A rooted device is one where a path exists
for a chosen process to run as UID 0, which the kernel's discretionary checks don't constrain
— UID 0 can read any UID's files, including every app's `/data/data` (file 01).

In practice "rooting" delivers two things:

- an **`su`** binary — "switch user" — that a process can invoke to escalate to UID 0;
- a **manager** app that decides which apps are allowed to call `su` and prompts you to
  approve each.

One caveat that matters: on modern Android, UID 0 is not the end of it. **SELinux** (file 01)
is a mandatory layer that constrains even root, so a working root solution also has to patch
the SELinux policy to give its root domain the latitude to act. Root that only flips the UID
and ignores SELinux doesn't get far.

## 6.2 Old way vs systemless

| | Old-style (`/system`) root | Systemless root (modern) |
|---|---|---|
| Where `su` goes | written into `/system` (e.g. `/system/xbin/su`) | **never** in `/system`; delivered from the boot ramdisk and `/data` |
| Verified Boot | **broken** — dm-verity sees the modified partition | **intact** — the verity-protected partitions are untouched |
| OTA updates | break, because `/system` no longer matches | survive, because `/system` is pristine |
| Detectable by | a file-presence check for `su` in `/system` | not *that* check — which is the whole point (§6.5) |

The shift from the left column to the right is the single most important development in
Android rooting, and it's what the rest of the chapter is about.

## 6.3 How systemless root works

The reference implementation is **Magisk** (by topjohnwu). Its insight: don't modify the
partition Verified Boot checks. Modify the **boot image's ramdisk** instead, and overlay all
your changes at runtime so the real partitions stay byte-for-byte stock.

Mechanically, patching the boot image installs a replacement early-init program. Per Magisk's
own documentation, **`magiskinit` replaces `init` as the first program to run**: it mounts
early partitions, injects Magisk's services, patches the SELinux policy, and only then runs
the original `init`
([Magisk docs/details.md](https://github.com/topjohnwu/Magisk/blob/master/docs/details.md)).
From there Magisk mounts a `tmpfs`, and **"magic mounts"** its modules — which live in
`/data/adb/modules` — into place using **bind mounts**, so files appear where the system
expects them without a single byte being written to the read-only partition.

```
stock:        [ boot ramdisk: init ] ──▶ init ──▶ Zygote ──▶ apps
                                              (verity-protected /system mounted read-only)

systemless:   [ boot ramdisk: magiskinit ] ──▶ sets up Magisk, patches sepolicy,
                     │                            bind-mounts modules from /data/adb
                     └──▶ runs original init ──▶ Zygote ──▶ apps
                                              (/system still pristine — verity still passes)
```

The user-visible setup is: unlock the bootloader (file 05 — this is the step that wipes the
device and flips the state to **orange**), extract the stock boot image, let the Magisk app
patch it, and flash the patched image back. That's the general mechanism; the exact
per-device dance is a project runbook, not something this course prescribes.

### Which image: `boot` or `init_boot`

One detail trips everyone, and it comes straight from a platform change. For most of
Android's life the generic ramdisk lived in **`boot.img`**, so that's what you patched. But
**"for devices launching with Android 13, the generic ramdisk is removed from the `boot`
image"** and placed in a separate **`init_boot`** image
([AOSP generic-boot](https://source.android.com/docs/core/architecture/partitions/generic-boot)).

So the rule:

| Device launched on… | Ramdisk lives in | Patch and flash |
|---|---|---|
| Android 12 or earlier (even if later updated) | `boot.img` | `fastboot flash boot …` |
| Android 13 (API 33) or later | `init_boot.img` | `fastboot flash init_boot …` |

The deciding factor is the **launch** version, not the current one — a phone that shipped on
Android 12 and updated to 13 keeps its ramdisk in `boot`. The property that records the launch
version is **`ro.product.first_api_level`**; a value of **33** or higher (Android 13 = API 33)
means the ramdisk is in `init_boot`. AOSP phrases the rule in version terms; the
`first_api_level` property is the standard way to detect it in the field.

## 6.4 Zygisk and the DenyList

Two Magisk features connect this chapter to files 04 and 09.

**Zygisk** is Magisk running code inside the **Zygote** process (file 04) and the app
processes forked from it. Recall why that's powerful: Zygote is the one parent every app
descends from, so a Zygisk module can run inside any app. This is the root-level equivalent
of the hooking reach that file 08's frameworks use. Note that enabling Zygisk does **not** by
itself hide root — it's a code-loading mechanism, not a concealment one.

**The DenyList** is the concealment mechanism. It's a per-app list; for each process on it,
Magisk **reverts and unmounts** its modifications so that process sees a stock-looking
system. It replaced the older MagiskHide. The intent: let a chosen app (a bank, say) run as
if the device were unmodified.

## 6.5 Why this defeats *naive* root detection — and why it doesn't end the story

Now the payoff, and it points straight at files 09 and 10.

A naive root check asks on-device questions: *is there an `su` binary in `/system`? is a known
superuser manager installed? is the build signed with test-keys?* Systemless root answers all
of those the "clean" way — `su` isn't in `/system`, and the DenyList unmounts Magisk's traces
for the checking app — so the check comes back negative on a device that is very much rooted.

> **Teacher's aside.** The lesson generalises past Magisk. A root check runs *as an app, on
> the device, inside the environment root controls*. It is asking the compromised system to
> describe itself. Whatever signals your check reads, root is positioned below you and can
> edit what you see — hide a file, unmount a mount, hook the API you query (file 08). This is
> not a flaw in one check to be fixed with a better check; it's structural. Any purely
> on-device root detection is a speed bump, valuable only against lazy attackers. The only
> observer root can't edit the answer for is one *below* root — the hardware — reporting to
> someone *off* the device — a server (file 10).

> ⚠️ Treating the DenyList as a guarantee is the mistake. It hides Magisk's *mounts* from
> listed apps, but it cannot change what the **hardware** says about the boot state. The
> bootloader is still unlocked; Verified Boot still reports **orange** (file 05); and Key
> Attestation / Play Integrity read that from below Magisk (file 10). DenyList can win the
> on-device argument and still lose the attested one.

## Check yourself

1. "Rooting broke dm-verity" is true of one rooting style and false of the other. Explain the
   difference in terms of *which partition* each style modifies, and why only one trips
   Verified Boot.
2. Magisk never writes to `/system`, yet files it provides appear where the system expects
   them. By what mechanism, and why does that leave the verity-protected partition's hashes
   unchanged?
3. You're told a device "launched on Android 12 and was updated to Android 14". Which image
   holds the ramdisk to patch, and which property would you read to be sure rather than
   guessing?
4. Enabling Zygisk gives a module code execution inside any app, but a reviewer says "so
   Zygisk hides root". Correct them: what does Zygisk do, and what separate feature does the
   hiding?
5. An app checks for `/system/xbin/su` and a known superuser package, finds neither, and
   concludes the device is clean. Give two independent reasons that conclusion can be wrong
   on a systemlessly-rooted device.
6. The DenyList makes a bank app see a stock-looking system, yet the bank's server still
   declines the device. What is the server learning that the on-device DenyList cannot
   suppress, and where does that information originate?
