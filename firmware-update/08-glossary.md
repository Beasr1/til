# 8. Glossary

Reference, not reading. Chapter references point at where the term is motivated.

## The shape of the problem

| Term | Meaning |
|---|---|
| **Firmware** | Per DFU 1.1 §1.2: "executable software stored in a write-able, nonvolatile memory on a USB device" (1) |
| **Bootloader** | The small program that runs first and can accept a new application image. Usually not replaceable through the same path it offers (1, 4) |
| **Bricked** | Loosely, any failed update. Usefully, only the case where nothing that can accept a new image is still running (1) |
| **Single-bank** | One program region. The update erases the only copy before writing the new one (5) |
| **Dual-bank / A-B** | Two program regions and a boot pointer. The running image survives the whole write; the commit is the pointer flip (5) |
| **Commit point** | The last moment at which stopping costs nothing — immediately before the first erase on single-bank hardware (5) |
| **Manifestation** | DFU's fourth phase: the device commits the transferred image and re-enumerates (1, 3) |

## DFU 1.1 — the class

| Term | Meaning |
|---|---|
| **DFU** | USB Device Firmware Upgrade, a USB-IF device class specification; revision 1.1 is dated 5 August 2004 (3) |
| **Four phases** | Enumeration, Reconfiguration, Transfer, Manifestation (1) |
| **Run-time DFU interface** | Class `FEh` / subclass `01h` / protocol `01h`, present while the device does its normal job. Marks it upgradable (2) |
| **DFU-mode interface** | The same class and subclass, protocol `02h`. The only interface the device exposes in DFU mode (2) |
| **DFU functional descriptor** | `bDescriptorType 21h`. Carries `bmAttributes`, `wDetachTimeOut`, `wTransferSize`, `bcdDFUVersion` (2) |
| **`bitCanDnload` / `bitCanUpload`** | `bmAttributes` bits 0 and 1: whether the device accepts downloads and offers uploads (2, 3) |
| **`bitManifestationTolerant`** | `bmAttributes` bit 2: whether the device can still talk after committing. If clear, the host must reset it (3) |
| **`bitWillDetach`** | `bmAttributes` bit 3: the device generates the detach–attach itself, and the host must **not** issue a reset (2) |
| **`wDetachTimeOut`** | How long the device waits for the host's USB reset after `DFU_DETACH` before giving up (2) |
| **`wTransferSize`** | The largest control-write block the device will accept (3) |
| **Download** | Host → device. Confusing until you accept the device's point of view (3) |
| **Upload** | Device → host. What makes a backup possible; optional, gated on `bitCanUpload` (3, 5) |
| **Zero-length `DFU_DNLOAD`** | The end-of-download marker. Moves the device into manifestation (3) |
| **`bwPollTimeout`** | Three bytes of the `DFU_GETSTATUS` reply: the minimum milliseconds to wait before asking again. Device-chosen, per operation (3) |
| **`wBlockNum`** | Block sequence number, wrapping to zero from 65,535 — unless an extension repurposes it (3, 4) |
| **`dfuDNBUSY`** | State 4. Programming. Any class request received here stalls the pipe and forces `dfuERROR` (3) |
| **`dfuERROR`** | State 10. Entered by any undefined request; exited only by `DFU_CLRSTATUS` (3) |
| **STALL** | How a DFU device signals an error. Not the answer — the prompt to send `DFU_GETSTATUS` (3) |
| **DFU file suffix** | 16 bytes at end of file: `dwCRC`, `bLength`, `ucDfuSignature`, `bcdDFU`, `idVendor`, `idProduct`, `bcdDevice`. Never sent to the device (6) |
| **`ucDfuSignature`** | The three magic bytes `44h 46h 55h`. A format marker, **not** a cryptographic signature (6) |

### `bStatus` values

| Code | Name | Code | Name |
|---|---|---|---|
| `0x00` | `OK` | `0x08` | `errADDRESS` |
| `0x01` | `errTARGET` | `0x09` | `errNOTDONE` |
| `0x02` | `errFILE` | `0x0A` | `errFIRMWARE` |
| `0x03` | `errWRITE` | `0x0B` | `errVENDOR` |
| `0x04` | `errERASE` | `0x0C` | `errUSBR` |
| `0x05` | `errCHECK_ERASED` | `0x0D` | `errPOR` |
| `0x06` | `errPROG` | `0x0E` | `errUNKNOWN` |
| `0x07` | `errVERIFY` | `0x0F` | `errSTALLEDPKT` |

### States

| Value | State | Value | State |
|---|---|---|---|
| 0 | `appIDLE` | 6 | `dfuMANIFEST-SYNC` |
| 1 | `appDETACH` | 7 | `dfuMANIFEST` |
| 2 | `dfuIDLE` | 8 | `dfuMANIFEST-WAIT-RESET` |
| 3 | `dfuDNLOAD-SYNC` | 9 | `dfuUPLOAD-IDLE` |
| 4 | `dfuDNBUSY` | 10 | `dfuERROR` |
| 5 | `dfuDNLOAD-IDLE` | | |

## DfuSe — the address-space extension

| Term | Meaning |
|---|---|
| **DfuSe** | A vendor extension adding addresses, erase and a memory map to DFU. Documented for one silicon family in application note AN3156 (4) |
| **Get command** | `DFU_UPLOAD` with `wValue` = 0. Returns the command codes the bootloader implements (4) |
| **Set Address Pointer** | `DFU_DNLOAD`, `wValue` = 0, first byte `0x21`, then a 32-bit address LSB first (4) |
| **Erase** | `DFU_DNLOAD`, `wValue` = 0, first byte `0x41`. Five bytes for a page, one byte for a mass erase (4) |
| **Read Unprotect** | First byte `0x92`. Removes read protection and erases flash and RAM in the process (4) |
| **Leave DFU** | A zero-length `DFU_DNLOAD`. The bootloader jumps to the address pointer (4) |
| **The address formula** | `Address = ((wBlockNum − 2) × wTransferSize) + Address_Pointer` (4) |
| **Memory-map string** | The interface string descriptor: `@name/0xaddr/n*size<B\|K\|M><type>,…` (4) |
| **Type letter** | Low three bits of the letter: 1 readable, 2 erasable, 4 writeable. So `a` = read-only, `g` = all three (4) |

## USB plumbing

| Term | Meaning |
|---|---|
| **`iSerialNumber`** | Device-descriptor index of a serial-number string. Optional. The only one of the three obvious identities that survives re-enumeration (2) |
| **`idProduct`** | DFU 1.1 recommends changing it in DFU mode, so a different driver binds. Therefore useless as an identity across the transition (2) |
| **CDC ACM** | USB Communications Device Class, Abstract Control Model. Class `02h` with a paired data interface at `0Ah`. Surfaces as a serial port: `/dev/ttyACM*`, `/dev/cu.usbmodem*`, `COMn` (2) |
| **Re-enumeration** | The device leaving the bus and rejoining as (usually) something else. Invalidates every handle and every port name (2) |
| **Control transfer** | The only transport DFU uses. No endpoints beyond the default pipe (3) |

## Windows

| Term | Meaning |
|---|---|
| **WinUSB** | Microsoft's in-box generic USB function driver, `winusb.sys` (7) |
| **WCID** | Informal name for a device that gets WinUSB bound automatically by declaring Microsoft OS descriptors (7) |
| **MS OS String Descriptor** | The string at index `0xEE`, carrying a signature field and `bMS_VendorCode` (7) |
| **`bMS_VendorCode`** | The vendor request code Windows then uses to fetch the feature descriptors (7) |
| **Extended compat ID** | The OS feature descriptor whose `compatibleID` field says `WINUSB` (7) |
| **Extended properties** | The OS feature descriptor carrying `DeviceInterfaceGUID` and WinUSB power settings (7) |
| **`osvc`** | Registry value under `UsbFlags\<vvvvpppprrrrr>` caching whether a device answered the `0xEE` probe. A failure is cached too (7) |
| **Driver package** | An INF plus a catalog plus binaries, installed into the driver store and matched against hardware ids (7) |
| **Catalog (`.cat`)** | The file the package's signature covers (7) |
| **Attestation signing** | Submitting a driver to Microsoft's portal without Hardware Lab Kit logs; documented for Windows 10 client (7) |
| **Cross-signing** | The pre-1607 model, where a vendor certificate chained to a Microsoft cross-certificate. Now only accepted in the documented exception cases (7) |

## Security vocabulary, used precisely

| Term | Meaning |
|---|---|
| **Integrity** | The bytes are the bytes someone computed the check over. What a CRC gives you (6) |
| **Authenticity** | The bytes came from a particular party. Requires a key; cannot be derived from the data alone (6) |
| **CRC-32** | The check DFU's suffix uses; the specification's reference implementation names polynomial `0xedb88320` (6) |
| **Detached signature** | A signature kept beside the artefact rather than inside it. The way to sign an image whose container has no field for one (6) |
| **Secure boot** | The bootloader verifying a signature before accepting or running an image. A property of the bootloader, not of DFU (6) |
| **Trust boundary** | Where verification actually happens. Host-side and device-side verification defend against different attackers (6) |
