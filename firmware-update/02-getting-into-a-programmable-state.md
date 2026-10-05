# 2. Getting a device into a programmable state

## The problem

Chapter 1 left us with an instruction we cannot carry out: "tell the device to become a
programmer". There is no standard button for that. The device is busy being a reader, or a
sensor, and the code that would accept a new image is either dormant or absent.

Worse, when it *does* become a programmer, it usually stops being the device you were
talking to. It disappears from the bus and something else appears in its place — with a
different product id, a different interface set, and quite possibly a different driver
bound to it by the host OS. Your handle is dead and your enumeration is stale.

So this chapter is about two things: how to ask, and how to keep hold of the device's
identity across an event that changes almost everything about it.

## Two ways to ask

### The standard way: a run-time DFU interface

DFU 1.1 has an answer built in. A device that supports DFU exposes, *while running
normally*, an extra interface that does nothing except mark the device as upgradable and
give the host somewhere to send the request:

| Field | Value | From |
|---|---|---|
| `bInterfaceClass` | `FEh` — Application Specific | DFU 1.1 Table 4.1 |
| `bInterfaceSubClass` | `01h` — Device Firmware Upgrade | DFU 1.1 Table 4.1 |
| `bInterfaceProtocol` | `01h` — runtime protocol | DFU 1.1 Table 4.1 |
| `bNumEndpoints` | `00h` — control pipe only | DFU 1.1 Table 4.1 |

Alongside it sits the **DFU functional descriptor** (`bDescriptorType 21h`, DFU 1.1
Table 4.2), which is the device advertising what it can do: `bmAttributes` with
`bitCanDnload` (bit 0), `bitCanUpload` (bit 1), `bitManifestationTolerant` (bit 2) and
`bitWillDetach` (bit 3); `wDetachTimeOut`; `wTransferSize`; `bcdDFUVersion`.

The handshake is then three steps (DFU 1.1 §5):

1. The host sends **`DFU_DETACH`** to that interface, with a `wTimeout` in milliseconds.
2. The host issues a **USB reset**.
3. The device comes back exposing the DFU-mode descriptor set.

`bitWillDetach` decides who does step 2. If it is set, the device generates the
detach–attach sequence itself and *the host must not issue a reset*. If it is clear, the
device starts a timer of `wDetachTimeOut` milliseconds and waits for the host's reset; if
the timer expires first, the device gives up and goes back to normal (DFU 1.1 §5.1 and
Appendix A.2.2).

### The other way: a vendor command over whatever channel exists

Plenty of devices have no run-time DFU interface at all. Their bootloader is entered by a
reset under particular conditions — a boot pin, a magic value left in RAM, a mode the
silicon vendor's system bootloader enters when the application does not run. For those,
`DFU_DETACH` is not merely unimplemented; it is meaningless. ST says so plainly about its
own built-in bootloader:

> The Detach request is not meaningful in the case of the bootloader. The bootloader is
> started by a system reset depending on the boot mode configuration settings, which means
> that no other application is running at that time.
>
> — AN3156, §2

So the application has to be told some other way, over whatever channel it already speaks.
A very common one is a **CDC ACM virtual serial port**: a USB class interface
(`bInterfaceClass 02h`, "Communications and CDC Control", with its paired data interface
at `0Ah`, "CDC-Data" — USB-IF defined class codes) that the host OS surfaces as an ordinary
serial device. On Linux that is typically `/dev/ttyACM0`, on macOS a `/dev/cu.usbmodem…`
node, on Windows a `COMn` port.

The application listens on that port for a vendor-defined command, and on receiving it
writes a magic value somewhere the bootloader will look — a known RAM address, a backup
register — and resets itself. The bootloader starts, sees the magic, and stays in upgrade
mode instead of jumping to the application.

Three consequences of that mechanism, none obvious:

- **The device may never acknowledge the command.** It is rebooting while composing its
  reply. Treat a missing response as normal, not as a failure.
- **The magic is volatile.** If it lives in RAM, a power cycle clears it — which is a
  feature (an unplug always escapes upgrade mode) and a trap (you cannot reset the device
  to "retry" and expect it still to be in the bootloader).
- **Nothing in the standard describes any of this.** The framing, the command codes, the
  magic value and its address are all the vendor's. The only part DFU covers is what
  happens *after* the bootloader is running.

> **Teacher's aside.** People read DFU 1.1 and expect `DFU_DETACH` to be how you always
> start. In practice there are two quite different populations. Devices whose *application*
> implements DFU use the standard detach; devices that rely on a *separate bootloader*
> reached by reset use something proprietary, and the standard only picks up once the
> bootloader has enumerated. Knowing which you have tells you what to look for on the bus:
> a run-time DFU interface (class `FE`/`01`/`01`) on the device as it normally appears, or
> nothing at all.

## The bootloader is a different USB device

Whichever route you took, the device that comes back is not the one that left. DFU 1.1
recommends this deliberately:

> to ensure that only the DFU driver is loaded, it is considered necessary to change the
> `idProduct` field of the device when it enumerates the DFU descriptor set. This ensures
> that the DFU driver will be loaded in cases where the operating system simply matches the
> vendor ID and product ID to a specific driver.
>
> — USB DFU 1.1, §2

The spec declines to say *how* to change it: "adding one, setting the high bit, or using
FFFFh are all valid possibilities. Vendors may use any scheme that they choose."

It also constrains what else the device may show. In DFU mode the device exposes only a
device descriptor, one configuration, one interface (plus alternate settings) and the
functional descriptor — "these are the only descriptors that the device may expose after
reconfiguration. The reason is to prevent any other device drivers from being loaded"
(DFU 1.1 §4.2). The interface's `bInterfaceProtocol` changes from `01h` to `02h`
(Table 4.4).

```
      normal mode                              DFU mode
 ┌───────────────────────┐             ┌─────────────────────┐
 │ VID:PID  1234:0002    │             │ VID:PID  1234:0001  │  ← different PID
 │ iSerial  A1B2C3D4     │  ═reboot═▶  │ iSerial  A1B2C3D4   │  ← same serial
 │ class    FE/01/01     │             │ class    FE/01/02   │
 │ (+ the device's real  │             │ (and nothing else)  │
 │  interfaces)          │             │                     │
 └───────────────────────┘             └─────────────────────┘
```

## So what do you hold on to?

You now need to find "the same device" on the other side of a re-enumeration. There are
three candidate identities, and two of them are wrong.

| Identity | Survives the reboot? | Why it fails |
|---|---|---|
| Port name (`COM7`, `/dev/ttyACM0`, `/dev/cu.usbmodem1234`) | **No** | It is assigned by the OS at enumeration and is not stable across one. The DFU-mode device may not have a serial port at all |
| `VID:PID` | **No, by design** | The spec advises the PID to change, precisely so a different driver binds. And two identical devices on one machine share it anyway |
| `iSerialNumber` | **Yes, when present** | It is a property of the unit, carried in the device descriptor in both modes |

`iSerialNumber` is a field of the USB device descriptor — it appears at offset 16 of the
DFU-mode device descriptor in DFU 1.1 Table 4.3, as an "Index of string descriptor". It is
the only one of the three that describes *this unit* rather than this model or this moment.

Microsoft's guidance for device designers makes the practical consequence explicit: when a
device supplies a serial number, Windows treats the device instance id as unique and it
stays stable across ports; when it does not, Windows composes the instance id from the
parent hub and port, so "if a device is moved to a new parent (plugged into a different
port, plugged into a USB hub, etc.), that id changes and the device appears as a new
device". Microsoft's recommendation for devices that cannot supply a unique value is to
supply none at all — set `iSerialNumber` to `0x00` rather than repeat one across units.

⚠️ A serial number is *optional*, and not every vendor populates it uniquely. Before
building identity on it, read it off two units of the same model and check they differ. If
they don't, you have no per-unit identity over USB and an update tool must refuse to
proceed whenever more than one candidate device is attached — guessing is the one outcome
you cannot recover from, because flashing the wrong device is not undoable.

Two smaller traps that follow from the same "identity is not the handle" idea:

- **One device can appear as several nodes.** On macOS a USB serial device typically shows
  up twice, as `/dev/cu.*` and `/dev/tty.*`. Deduplicate by serial number, or a single
  device looks like two and a "refuse if ambiguous" rule fires on nothing.
- **Enumeration is not instantaneous or infallible.** The bootloader takes real time to
  appear, and host enumeration APIs can transiently fail to describe a device that is
  plainly attached. Poll to a deadline rather than asking once.

As an illustration of the time scales involved: on one STM32-based USB reader, the
bootloader appeared roughly 1.4–3.5 s after the reset command across repeated runs, and
the device took about 4.8–7.3 s to come back as its normal self afterwards. Those are
measurements from one device, not a specification figure — the point is only that the
right order of magnitude is *seconds*, they vary between units of the same model, and any
constant you hard-code should be logged against the real elapsed time so it can be
corrected from the field rather than from one bench.

## The state you can get stuck in

Put the two halves together and a specific failure appears, worth naming because the
defences later in the course are partly aimed at it.

The host sends the reboot command. The device obeys — it is now in its bootloader. The host
then fails to find it: the wrong driver is bound (chapter 7), the serial number is missing,
the enumeration timeout is too short. The host reports a clean failure and gives up.

Nothing has been erased. No flash is at risk. And yet the device is sitting in upgrade mode
and has stopped doing its job, and will stay there until it is power-cycled or something
tells it to leave.

⚠️ If your update tool can reach the bootloader, it should also be able to tell the
bootloader to *leave* — and it should do so on the error path, not just the happy path. An
update that aborts after the reboot but before any write should put the device back where
it found it. Otherwise "the update failed safely" and "the device works" are two different
statements.

## Check yourself

1. A device exposes an interface with class/subclass/protocol `FE/01/01` while doing its
   normal job. What does that tell you, and what does the `01` in the last position
   distinguish it from?
2. `bitWillDetach` is set in a device's DFU functional descriptor. Describe exactly what
   the host must do, and what it must *not* do, after sending `DFU_DETACH`.
3. Your tool opens the device's serial port, sends the vendor reboot command, and reports
   failure because no acknowledgement arrived. What is wrong with that logic?
4. DFU 1.1 recommends changing `idProduct` in DFU mode. Give the reason the spec gives,
   and then explain why that recommendation breaks a tool that tracks devices by `VID:PID`.
5. You have two identical devices attached and the model supplies no serial number. Argue
   for what the update tool should do, and say what the cost of the alternative is.
6. A colleague's tool finds the device by port name, sends the reboot command, then
   reopens the same port name to talk to the bootloader. Name two independent reasons this
   fails, one of which would still apply if port names were stable.
