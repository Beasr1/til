# 7. Glossary

Reference, not reading. Chapter references point at where the term is motivated.

## The wire

| Term | Meaning |
|---|---|
| **APDU** | Application Protocol Data Unit. One command or one response. The only shape of exchange there is (3) |
| **CLA** | Class byte. `0X` = ISO interindustry; `8X`–`EX` = the card's proprietary commands (3) |
| **INS** | Instruction byte. `A4` SELECT, `B0` READ BINARY, `CA` GET DATA, `88` INTERNAL AUTHENTICATE, `82` EXTERNAL AUTHENTICATE, `84` GET CHALLENGE, `20` VERIFY, `2A` PSO |
| **P1, P2** | Parameter bytes. Meaning depends entirely on the instruction |
| **Lc** | Length of the command data you are sending |
| **Le** | Expected length of the response. `0x00` means 256, not zero (3) |
| **SW1 SW2** | The two status bytes every response ends with. `9000` is success (3) |
| **Case 1–4** | Whether an APDU carries command data, expects response data, both, or neither (3) |

## Power-up and transport

| Term | Meaning |
|---|---|
| **ATR** | Answer To Reset. The card's only unsolicited message, describing electrical and protocol capability (2) |
| **TS** | Initial character. `3B` direct convention, `3F` inverse (2) |
| **T0** | Format byte: which interface bytes follow, and how many historical bytes (2) |
| **TA1, TB1, TC1** | Timing and voltage parameters (2) |
| **TD1** | Advertises a transmission protocol. **Absent means T=0** (2) |
| **Historical bytes** | Up to fifteen issuer-defined bytes at the end of the ATR (2) |
| **TCK** | ATR checksum, present when a protocol other than T=0 is indicated (2) |
| **T=0** | Byte-oriented protocol. Cannot return data and status together; uses `61 XX` and `6C XX` (4) |
| **T=1** | Block-oriented protocol. Returns data and status in one response (4) |
| **GET RESPONSE** | `00 C0 00 00 XX` — fetches the data a T=0 card said was waiting (4) |
| **PC/SC** | The host-side reader API implemented by every OS (1) |
| **Pseudo-APDU** | A command the *reader* answers, not the card. Absence proves nothing about the card (1) |
| **ISO/IEC 14443** | The contactless interface standard. Contactless ATRs are synthesised from its ATS (2) |

## Organisation

| Term | Meaning |
|---|---|
| **MF** | Master File. The root of an ISO file tree, FID `3F00` (5) |
| **DF** | Dedicated File. A directory (5) |
| **ADF** | Application DF — a DF selected by AID rather than FID (5) |
| **EF** | Elementary File. A file containing data (5) |
| **FID** | File Identifier. The 2-byte name of an MF, DF or EF (5) |
| **AID** | Application Identifier. A registered 5–16 byte application name (5) |
| **EF.DIR** | Optional EF at `2F00` listing the card's AIDs. Frequently absent (5) |
| **FCI** | File Control Information — what `SELECT` returns about the selected object (5) |
| **FCP / FMD** | File Control Parameters and File Management Data — the two halves an FCI may carry |
| **FDB** | File Descriptor Byte, FCI tag `82`: transparent, record-structured, or a DF (5) |
| **Transparent EF** | A flat byte array. Read with `READ BINARY` (5) |
| **Record EF** | A sequence of records. Read with `READ RECORD` — `READ BINARY` will refuse (5) |
| **SFI** | Short EF Identifier. A 5-bit alias letting some commands address a file without selecting it |
| **BER-TLV** | Tag-Length-Value encoding used throughout card data structures |

## Platform

| Term | Meaning |
|---|---|
| **GlobalPlatform** | The specification for loading, selecting and managing applets on a card (5) |
| **ISD** | Issuer Security Domain — the card manager applet. Implements card content management and little else (5) |
| **JavaCard** | The runtime most GlobalPlatform cards use. Applets are firewalled from each other |
| **Card recognition data** | `GET DATA` tag `66` — OIDs naming the GlobalPlatform version and secure channel protocol (6) |
| **CPLC** | Card Production Life Cycle data, `GET DATA` tag `9F7F` — fabricator, IC type, dates, IC serial number |
| **Security domain** | An on-card entity holding keys on behalf of an issuer or application provider |

## Security

| Term | Meaning |
|---|---|
| **Diversification** | Deriving a per-card key from a master key and a card-unique value, so one card's key reveals nothing about another's (6) |
| **KDD** | Key Diversification Data — the card-unique input to that derivation (6) |
| **HSM** | Hardware Security Module. Holds the master key and performs the derivation (6) |
| **GET CHALLENGE** | Asks the card for a random value, so an authentication cannot be replayed (6) |
| **EXTERNAL AUTHENTICATE** | The **terminal** proving itself to the card (6) |
| **INTERNAL AUTHENTICATE** | The **card** proving itself to the terminal, asymmetrically (6) |
| **Cryptogram** | A MAC over the exchanged challenges, proving possession of the key (6) |
| **SCP01 / SCP02 / SCP03** | GlobalPlatform Secure Channel Protocols. 3DES, 3DES with a counter, AES-CMAC respectively (6) |
| **Secure messaging** | MACing and optionally encrypting APDUs over an established session (6) |
| **PSO** | Perform Security Operation, `INS 2A` — the ISO 7816-8 command for signing, deciphering and similar |
| **Trust anchor** | A CA certificate obtained out of band, against which card certificates are checked. Without one, verification is circular (6) |
| **Retry counter** | The count of remaining attempts for a credential. `63 CX` reports it — and decrements it (3) |

## eID credentials and access control

Introduced in chapter 6 alongside the two worked national schemes. The distinction that
matters is **access** (prove you hold the card) versus **use** (prove you are the holder).

| Term | Meaning |
|---|---|
| **Access credential** | Printed on the card, so anyone holding it has it. Opens a channel to the chip. MRZ, CAN (6) |
| **Use credential** | Known only to the holder, delivered off-card. Authorises a key to act. PIN, signature PIN (6) |
| **MRZ** | Machine Readable Zone — the fixed-width lines printed on a document. Also usable as a PACE/BAC password (6) |
| **CAN** | Card Access Number. A short number printed on the card face, used as a PACE password (6) |
| **PACE** | Password Authenticated Connection Establishment. Opens a secure channel using one of four passwords — MRZ, CAN, PIN or PUK (6) |
| **BAC** | Basic Access Control. PACE's predecessor, keyed on the MRZ only (6) |
| **Active Authentication** | ICAO's asymmetric chip-genuineness proof: the chip signs the terminal's challenge. Not PIN-gated (6) |
| **Chip Authentication** | The EAC mechanism that proves the chip genuine *and* yields a secure channel. Not PIN-gated (6) |
| **Inspection System** | Terminal role that may use the CAN or MRZ — border control, document checking (6) |
| **Authentication Terminal** | Terminal role that uses the holder's PIN to read personal data (6) |
| **PUK** | PIN Unblock Key. Restores a blocked PIN. Some schemes replace it with a biometric reset (6) |
| **SAM** | Secure Access Module. A second smart card in the terminal holding symmetric keys, doing locally what an HSM would do centrally (6) |
| **Transport PIN** | A one-time PIN shipped to the holder, valid once, replaced with one of their choosing on first use (6) |

## Status words worth knowing by sight

| SW | Meaning |
|---|---|
| `90 00` | Success |
| `61 XX` | T=0: `XX` bytes waiting — send GET RESPONSE |
| `6C XX` | T=0: wrong `Le` — re-send with `XX` |
| `62 82` | End of file before `Le` — a **warning**; the data is valid |
| `63 CX` | Verification failed, `X` retries left — **costs a try** |
| `69 82` | Security status not satisfied — authenticate first |
| `69 85` | Conditions of use not satisfied — often nothing is selected |
| `6A 82` | File not found — often "not under what's currently selected" |
| `6A 86` | Incorrect P1/P2 — often the wrong selection mode |
| `6D 00` | Instruction not supported — often no applet selected |
| `6E 00` | Class not supported — often try `CLA=80` |
| `63 00` | Verification failed — after authentication, the key didn't match |
