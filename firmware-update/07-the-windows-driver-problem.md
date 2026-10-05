# 7. The Windows driver problem

## The problem

Everything in chapters 3 and 4 is USB control transfers on the default pipe. On Linux, a
user-mode process can do those through usbfs once nothing else has claimed the interface;
on macOS, through IOKit on much the same terms. On Windows it cannot, because a USB
interface has to have a **kernel-mode function driver** bound to it before user-mode code
has anything to open.

And the DFU-mode interface has `bInterfaceClass = FEh`, "Application Specific" (DFU 1.1
Table 4.4). Windows ships no in-box driver for that class. So by default the bootloader
enumerates, appears in Device Manager with a warning triangle, and your tool cannot talk to
it at all.

This is a platform problem rather than a protocol problem, and it lands at the worst
possible moment in the sequence — after the device has already rebooted into its
bootloader, which is exactly the stuck state chapter 2 warned about.

## WinUSB, and the two ways to get it

Microsoft's generic answer is **WinUSB** (`winusb.sys`), an in-box driver that exposes a
USB device to user mode. The question is only how it gets bound to your device. There are
two routes, and they differ enormously in what they cost you.

### Route 1 — the device asks for it (WCID / Microsoft OS descriptors)

A device can declare in its own firmware that it wants WinUSB, and Windows will bind it
with no INF, no installer and no user interaction. This is what people mean by a "WCID
device". Microsoft's own definition:

> A WinUSB device is a Universal Serial Bus (USB) device whose firmware defines certain
> Microsoft operating system (OS) feature descriptors that report `WINUSB` as the
> compatible ID.
>
> — Microsoft Learn, *WinUSB Device*

It works through a descriptor at a reserved index:

> Devices that support Microsoft OS Descriptors must store a special USB string descriptor
> in firmware at the fixed string index of `0xEE`. … Its presence indicates that the device
> contains one or more OS feature descriptors. It contains the data that is required to
> retrieve the associated OS feature descriptors. It contains a signature field that
> differentiates the OS string descriptor from other strings that IHVs might choose to
> store at `0xEE`.
>
> — Microsoft Learn, *Microsoft OS Descriptors for USB Devices*

The sequence:

```
 Windows                                   device
   │   GET_DESCRIPTOR(STRING, index 0xEE)    │
   │ ───────────────────────────────────────▶│
   │        MS OS string: signature + bMS_VendorCode
   │ ◀───────────────────────────────────────│
   │                                          │
   │   vendor request, using bMS_VendorCode   │
   │ ───────────────────────────────────────▶│
   │        extended compat ID descriptor:    │
   │        compatibleID = "WINUSB"           │
   │ ◀───────────────────────────────────────│
   │                                          │
   └── matches the in-box Winusb.inf entry for USB\MS_COMP_WINUSB
       → winusb.sys is loaded. No user interaction.
```

The string at `0xEE` carries a vendor code; "the retrieved string descriptor has a
`bMS_VendorCode` field value. The value indicates the vendor code that the USB driver stack
must use to retrieve the extended feature descriptor." Then "the `compatibleID` field of
that section must specify `WINUSB` as the field value". From Windows 8 onwards the in-box
`Winusb.inf` contains an install section referencing the compatible ID `USB\MS_COMP_WINUSB`,
and that is the match.

A device using this route should also supply an **extended properties** feature descriptor
setting `DeviceInterfaceGUID`, because that GUID "is required to find the device from an
application or service, configure the device, and perform I/O operations".

Three traps in this mechanism, all from Microsoft's own documentation and all with real
consequences:

⚠️ **The answer is cached, and a failure is cached too.** Windows asks for the `0xEE`
string once, when the device is first attached, and records the result in the registry under
`HKLM\SYSTEM\CurrentControlSet\Control\UsbFlags\<vvvvpppprrrrr>` in a value named `osvc`.
"If the device doesn't provide a valid response the first time that the operating system
queries it for a Microsoft OS String Descriptor, the operating system makes no further
requests for that descriptor." A firmware fix that adds the descriptor will therefore do
nothing on a machine that has already met that device, until the cached entry is cleared.

⚠️ **The key is per VID/PID/revision.** Which means the bootloader — a *different* product
id, by DFU 1.1's own recommendation (chapter 2) — has its own entry and its own answer.
A device can be perfectly WCID in normal mode and not WCID at all in DFU mode. Probe the
bootloader specifically; do not infer it from the running device.

⚠️ **A non-WCID device must stall the request.** "If a device doesn't contain a valid
string descriptor at index `0xEE`, it must respond with a stall packet." The query happens
"during device enumeration — before the driver for the device loads — which might cause
some devices to malfunction", and Microsoft says such devices simply are not supported.
This is also how you test: ask a device for string `0xEE` yourself and see what it does.

### Route 2 — you ship a driver package

If the firmware does not answer the `0xEE` probe, and you cannot change the firmware
(which, note the circularity, is the problem you are trying to solve), then WinUSB has to
be bound by an **INF driver package** that matches the device's hardware id and pulls in
the in-box WinUSB sections. Microsoft describes this as the pre-Windows-8 path: "Before
Windows 8, to load `Winusb.sys` as the function driver, you needed to provide a custom
INF."

A driver package is a different kind of artefact from an application, and it is worth
being explicit about the differences because they are what makes this expensive:

| | Application | Driver package |
|---|---|---|
| Unit | an executable | an INF plus a catalog (`.cat`) plus any binaries |
| Installed by | copying, or an installer | the OS driver store, via `pnputil` or setup APIs, as administrator |
| Bound to | nothing | specific hardware ids, matched at enumeration |
| Signature covers | the executable | the whole package, via the catalog |
| Signed by | you, with a code-signing certificate | see below — not just you |

## Driver signing is not code signing

This is the part that surprises people who have shipped signed applications and assume a
driver is the same job with a different file extension.

> Starting with Windows 10, version 1607, Windows will not load any new kernel-mode drivers
> which are not signed by the Dev Portal. To get your driver signed, first Register for the
> Windows Hardware Dev Center program. Note that an EV code signing certificate is required
> to establish a dashboard account.
>
> — Microsoft Learn, *Driver Signing Policy*

Unpack that:

- **Your own signature is not sufficient.** The package has to be *submitted to Microsoft*
  and comes back bearing Microsoft's signature. You are asking a third party for something,
  on their schedule.
- **An EV certificate is required to open the account** — not merely a standard
  code-signing certificate. EV issuance involves organisational validation and, typically,
  hardware-token delivery, which takes time and is not something a developer can arrange in
  an afternoon.
- **There are two submission routes.** Full certification means running the Hardware Lab
  Kit and submitting logs. **Attestation signing** skips HLK and is documented for Windows
  10 client systems — much lighter, and usually what a WinUSB INF needs.

The documented exceptions are worth knowing, because they explain why something works on
your bench and not on a customer's machine. Cross-signed drivers are still permitted if:

> - The PC was upgraded from an earlier release of Windows to Windows 10, version 1607.
> - Secure Boot is off in the BIOS.
> - Drivers was signed with an end-entity certificate issued prior to July 29th 2015 that
>   chains to a supported cross-signed CA.

A development machine with Secure Boot disabled, or one upgraded rather than clean
installed, will accept a self-signed test package quite happily. A clean-installed customer
machine with Secure Boot on will not. Test on the second kind.

> **Teacher's aside.** The asymmetry to internalise: **WCID is a firmware decision with
> almost no cost, and its absence is an organisational cost you pay forever.** Sixteen
> bytes of string descriptor at index `0xEE`, plus a compat-ID response, and Windows binds
> the driver silently on every machine. Without them you need a certificate with an
> organisational validation process, an account with Microsoft, a submission for every
> package revision, an installer that runs as administrator, and a support burden for every
> machine where the binding did not take. If you have any influence over the firmware of a
> device that will ever need a vendor-class interface — including its bootloader — spend it
> here. It is the highest-leverage change in this entire course.

## What the signing policy actually governs

Read the previous section and you would conclude that shipping a vendor-class driver
requires a certificate with organisational validation, an account with Microsoft, and a
submission per package. People do conclude that, budget for it, and lose weeks.

Then you meet a shipping product whose installer drops a **self-signed** WinUSB package,
and Windows binds it on a stock machine with `Status: OK`. Both things cannot be true, so
one of them is being read too broadly.

It is the first. Look again at what the policy says: *"Windows will not load any new
**kernel-mode drivers** which are not signed by the Dev Portal."* The object of that
sentence is a kernel binary — a `.sys` that will execute in ring 0. Code integrity is
being enforced on code.

A WinUSB INF contains no code. Read one and the payload is two lines:

```ini
[USB_Install]
Include = winusb.inf
Needs   = WINUSB.NT
```

It installs nothing. It is a note saying *"bind the WinUSB you already have to this
hardware id"*, and `WinUSB.sys` is Microsoft's own, in-box, signed by them, already loaded
on the machine for other devices. There is no new kernel-mode driver in the transaction, so
the policy that governs new kernel-mode drivers has nothing to act on.

What *is* still required is that Windows trust the **driver package** — the INF plus the
catalog (`.cat`) that hashes it. That is a Plug and Play trust decision, not a code
integrity one, and it is satisfied by the signing certificate being present in the
machine's certificate stores: `TrustedPublisher` so the package installs without asking,
and `Root` as well when the certificate is self-signed and is therefore its own authority.

So a driver package is three things, and it is worth holding them apart:

| Part | What it is | Who must trust it |
|---|---|---|
| `.inf` | the rules — which driver binds which hardware id | nobody; it is data |
| `.cat` | a signature over the INF and its files | Windows, at install |
| the `.sys` | the actual driver code | Windows, at load — **this** is what the 1607 policy governs |

A WinUSB package has no third row. That is the whole reason it escapes.

> **Teacher's aside.** This is a good habit to build generally: when a rule and an
> observation disagree, re-read the rule for its *object* before concluding the observation
> is a fluke. "Drivers must be signed by Microsoft" and "kernel-mode driver binaries must
> be signed by Microsoft" are different sentences, and only one of them is the policy.

### The failure is a dialog, which matters more than it sounds

Install a package whose catalog is unsigned, or signed by a certificate the machine does
not trust, and Windows does not refuse. It asks — *"Windows can't verify the publisher of
this driver software"* — and a human clicking through gets a working driver.

That is why this defect hides. Someone tests by hand, clicks the box, sees the device work,
and reports success. Then the same package goes out inside a silent installer, where there
is no desktop and no human, and it fails on every machine.

The lesson generalises past drivers: **when a trust failure degrades to a prompt, testing
interactively cannot detect it.** Any install path meant to run unattended has to be tested
unattended, because the interactive path is not a weaker version of the same test — it is a
different test that passes for a different reason.

### What is not settled here

The observation above comes from a Windows 11 machine on which a self-signed WinUSB package
installed and bound with no test signing. What was **not** recorded on that machine is
whether Secure Boot was enabled. The cross-signing exceptions quoted earlier explicitly
include *"Secure Boot is off in the BIOS"*, so a machine with it disabled would accept a
self-signed package for a reason that has nothing to do with the argument above, and the
observation would not support the conclusion.

The reconciliation offered here — that WinUSB packages ship no kernel binary and so fall
outside the kernel-mode policy — is consistent with the documents and with the observation,
but one measurement would separate it from the alternative:

```powershell
Confirm-SecureBootUEFI
```

`True` on a machine that accepted a self-signed WinUSB package supports the reading above.
`False` means the machine was an exception case and the question is still open. Until
someone runs it on the machine where the package bound, treat this section as the
better-supported of two readings rather than as settled.

## Practical rules

**Match narrowly.** An INF that binds WinUSB should name exactly the bootloader's
`VID`/`PID` and nothing else. A broad match can capture the device in its *normal* mode,
where it needs a class driver to do its actual job, and the symptom — the device enumerates
and is silently useless — looks nothing like a driver problem.

**A device already owned by a class driver is not free to take.** If the normal-mode device
is claimed by an in-box class driver, you cannot simply bind WinUSB to it without breaking
the function that driver provides. This is a real constraint on "just use libusb for
everything" designs, and it is another argument for keeping the vendor interface on a
distinct product id that exists only during an update.

**Clean up your own history.** An INF from a previous scheme stays in the driver store and
keeps binding. If you stop needing WinUSB on a device, or you change which id it matches,
the installer has to remove the old package — otherwise machines updated from the old
version behave differently from machines installed fresh, for reasons nobody will guess.

**Fail with a message that names the cause.** The symptom of a missing driver is a timeout
waiting for a device that is right there on the bus. Nothing about "device did not
re-enumerate in DFU mode" suggests "install a driver". On Windows specifically, when the
bootloader is not found after a reboot that appeared to succeed, say so in those words and
point at the fix — and, per chapter 2, put the device back before giving up.

## Check yourself

1. Why does a DFU-mode interface need a driver on Windows but not, in the same way, on
   Linux? Answer in terms of what each platform requires before user-mode code can issue a
   control transfer.
2. A vendor ships a firmware update adding Microsoft OS descriptors. On a fresh machine
   the device binds WinUSB automatically; on a developer's machine it still does not.
   Explain, and say what has to be done.
3. Your device answers the `0xEE` probe in normal mode. What can you conclude about its
   bootloader, and why?
4. A colleague says "we already sign our installer with an EV certificate, so the driver
   is covered". Signing the installer does not sign the driver package — but if the package
   is a WinUSB INF, their instinct is closer to right than the policy makes it sound.
   Untangle both halves.
5. Your INF matches on vendor id alone. Describe a concrete failure this causes that does
   not look like a driver problem at all.
6. You must support a bootloader that is not WCID, on machines you do not administer.
   Lay out the options with their costs, and say which you would recommend and why.
7. A driver package installs correctly when you run the installer by hand, and fails on
   every machine when the same installer runs from a deployment tool. Nothing about the
   package changed. What is the most likely cause, and what does it say about how the
   install path should have been tested?
8. Name the three parts of a driver package and say, for each, who has to trust it and at
   what moment. Then explain why a package binding WinUSB is exempt from the rule that
   catches most people.
