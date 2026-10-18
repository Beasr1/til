# Peer-to-Peer Connections — A Course

How two machines on ordinary home and office networks connect to each other
directly, when neither has a public address and both sit behind NATs and
firewalls.

**Assumes [http file 04](../http/04-quic.md)** (QUIC as a transport: streams,
connection IDs and migration, TLS inside the handshake, and RFC 9221 datagrams)
**and [cs-fundamentals file 09](../interview/cs-fundamentals/09-networks-layers-and-tcp.md)**
(IP, ports, TCP and UDP). Certificates and TLS trust are in
[http file 05](../http/05-certificates-and-trust.md), which file 05 here builds on.
None of those is re-taught; this course covers only what peer-to-peer adds.

**This is reference learning material.** It's about the published standards,
research and open-source implementations, not any particular system. Every claim
about a protocol is checked against its RFC; every claim about an implementation
cites a file in a named repository at a named tag; every success-rate figure says
what it measured and how far to trust it.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure that forced each mechanism comes before its name.
- Every chapter ends with **Check yourself** questions. Answers are in
  [09-exercises.md](09-exercises.md).
- Where the field hasn't settled something (how often hole punching works,
  whether real-time media belongs in datagrams) I show the disagreement rather
  than picking a side quietly.

## The one thing to understand first

> **A NAT only lets in packets that answer something its own host sent.
> Client–server works because the client always speaks first. Peer-to-peer works
> by arranging for both peers to speak first at once, and that only succeeds when
> each NAT's choice of external port is predictable. When it isn't, a relay
> forwards the packets, and the relay is safe to use only because the peers
> authenticate each other by public key, not by address.**

Everything in the course is an elaboration of one of those sentences.

## Reference sources

### Standards

| Source | What it settles |
|---|---|
| [RFC 4787 — NAT behavioural requirements for UDP](https://www.rfc-editor.org/rfc/rfc4787.html) | Mapping and filtering vocabulary (EIM, APDF…), timer minimums, hairpinning. BCP 127 |
| [RFC 5128 — State of P2P communication across NATs](https://www.rfc-editor.org/rfc/rfc5128.html) | The techniques catalogued: relaying, hole punching, port prediction |
| [RFC 8489 — STUN](https://www.rfc-editor.org/rfc/rfc8489.html) | Binding requests, XOR-MAPPED-ADDRESS and why it's XORed |
| [RFC 8445 — ICE](https://www.rfc-editor.org/rfc/rfc8445.html) | Candidates, priorities, connectivity checks, nomination |
| [RFC 8656 — TURN](https://www.rfc-editor.org/rfc/rfc8656.html) | Allocations, permissions, channels, and the cost argument for using TURN last |
| [RFC 7250 — Raw public keys in TLS](https://www.rfc-editor.org/rfc/rfc7250.html) | Sending a bare key instead of a certificate, and the out-of-band binding it needs |
| [RFC 6887 — PCP](https://www.rfc-editor.org/rfc/rfc6887.html) | Asking a NAT or firewall for a mapping |
| [RFC 6888 — Carrier-grade NAT requirements](https://www.rfc-editor.org/rfc/rfc6888.html) | What a CGN should do: EIM, EIF, port limits, PCP, unguessable ports |
| [RFC 6092 — IPv6 CPE simple security](https://www.rfc-editor.org/rfc/rfc6092.html) | Why IPv6 still has a firewall to punch through (Informational) |
| [RFC 8827 — WebRTC security architecture](https://www.rfc-editor.org/rfc/rfc8827.html) | How WebRTC binds peers to keys, and what the signalling server is trusted for |
| [draft-bruynooghe-n0-quic-nat-traversal](https://datatracker.ietf.org/doc/draft-bruynooghe-n0-quic-nat-traversal/) | iroh's hole-punching-inside-QUIC protocol, as implemented. Not a standard |

### Measurements and write-ups

| Source | Why it's here |
|---|---|
| Ford, Srisuresh & Kegel, ["Peer-to-Peer Communication Across Network Address Translators"](https://bford.info/pub/net/p2pnat/), USENIX ATC 2005 | The reference description of hole punching, and the first NAT survey (self-selected sample) |
| Halkes & Pouwelse, "UDP NAT and Firewall Puncturing in the Wild", IFIP Networking 2011 ([HAL](https://inria.hal.science/hal-01597969v1)) | Measured hole punching in a deployed P2P network |
| Trautwein, Ihle, Schubotz & Gipp, ["Challenging Tribal Knowledge"](https://arxiv.org/abs/2510.27500), arXiv 2025 | The largest measurement: 4.4 million attempts in libp2p/IPFS |
| David Anderson, ["How NAT traversal works"](https://tailscale.com/blog/how-nat-traversal-works), Tailscale, 2020 | The best long-form explanation; source of the birthday-trick numbers |

### Reference implementation

| Project | Why | Link |
|---|---|---|
| **iroh** v1.3.0 | Open source, QUIC-based, small enough to read; the worked example in file 06 | <https://github.com/n0-computer/iroh> (docs: <https://docs.iroh.computer>) |
| **noq** | iroh's QUIC stack, a Quinn fork with multipath, address discovery and NAT traversal extensions | <https://github.com/n0-computer/noq> |

iroh code references are **relative to the repository root at tag v1.3.0**
(28 September 2026), e.g. `iroh/src/tls/verifier.rs:68`. iroh renamed its core
types and changed its wire protocol several times before 1.0; file 06 §6.1 has
the version table.

## Reading order

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [**Foundations**](01-foundations.md) | ⭐ Prerequisite for everything else. Explain why dialling a peer's IP fails, and name the five steps every P2P system takes |
| 2 | [How NATs behave](02-nat-behaviour.md) | Classify a NAT by mapping and filtering, translate the old "cone" names, and reason about timeouts, CGNAT and IPv6 |
| 3 | [Hole punching](03-hole-punching.md) | Explain STUN and simultaneous open, predict which NAT pairs can't punch, quote success rates honestly, and read ICE |
| 4 | [Relays](04-relays.md) | Compare TURN, DERP and iroh's relay, and say exactly what "the relay can't read it" does and doesn't promise |
| 5 | [Identity and discovery](05-identity-and-discovery.md) | Explain public-key addressing and raw public keys in TLS, and trace where the trust actually lives |
| 6 | [Worked example: iroh](06-iroh-end-to-end.md) | Follow one implementation from `bind()` to a direct path, and compare it with Tailscale and WebRTC |
| 7 | [Real-time traffic over a P2P path](07-real-time-over-p2p.md) | Predict how streams and datagrams behave as the path moves between relay and direct |

### Reference

| # | File | |
|---|------|--|
| 8 | [Glossary](08-glossary.md) | Look things up |
| 9 | [Exercises & answers](09-exercises.md) | Worked answers to every Check yourself, things to try, and questions for going deeper |

## If you're short on time

- **20 minutes:** file 01, then the pair table in file 03 §3.3.
- **"Will direct connections work for my users?"** File 02 §2.1–2.3, then file 03
  §3.3 and §3.5. The last is the honest state of the numbers.
- **Designing a P2P feature:** files 01, 04 and 05. Relays and trust are the
  parts people under-plan.
- **Evaluating iroh specifically:** file 01, then file 06, dipping into 05 §5.3
  for the TLS details.
- **Building voice or video on a P2P connection:** file 07, after the QUIC
  chapter's datagram section.

## The one-paragraph summary of everything

A machine on a home or office network has a private address that means nothing
outside, and a NAT in front of it that translates its packets to a shared public
address and, together with a firewall, admits only replies to flows the inside
host started. Client–server never notices because the client always speaks
first. Two peers can't both wait to be spoken to, so they **hole punch**: each
learns its public-facing **reflexive address** from a server outside its NAT
(STUN, or QUIC Address Discovery), they swap addresses over a **signalling**
channel, and both send to each other at once so each NAT sees an outbound flow and
admits the reply. That works when NATs use **endpoint-independent mapping** (the
same external port for every destination), and fails when one side gives every
destination a new port and the other filters strictly. Old "cone/symmetric" names
blur exactly this distinction, which is why RFC 4787 replaced them. Measured
success is around 70% per attempt in the largest recent study, with vendor claims
nearer 90%; nobody gets close enough to 100% to skip a **relay** (TURN, Tailscale's
DERP, iroh's DERP-derived relay), which forwards packets for the rest and doubles as
the signalling channel. Relays see only ciphertext *if* peers authenticate each
other end to end, and with addresses changing mid-connection the only stable thing
to authenticate is a **public key**. So peers are named by keys, TLS carries a raw
public key instead of a certificate (RFC 7250), and the trust that a CA used to
provide moves to whatever told you the peer's key. iroh puts it together by
connecting over the relay first, hole punching inside the live QUIC connection,
and running relay and direct paths side by side with QUIC multipath, always
preferring direct. Real-time traffic has to be designed for that moving path: a
datagram on a TCP-based relay is delayed rather than dropped.

## Scope, and what changed from the original brief

The brief listed identity and discovery as one topic and the iroh walkthrough as
another; both stayed. Four judgement calls:

- **Real-time traffic got its own short chapter (07)** rather than a section of
  file 06. It answers a different question (designing your protocol rather than
  understanding iroh) and the QUIC chapter already covers the transport side, so
  what remains is short.
- **NAT port-mapping protocols (UPnP, NAT-PMP, PCP) are a section of file 02,
  not a chapter.** They're opportunistic: often disabled, and they don't help with
  carrier-grade NAT, so no design can rest on them.
- **TCP hole punching is a short section, not a chapter.** Every system in the
  course punches over UDP. It stays because the 2025 measurement found TCP and
  QUIC equally successful, which overturns a common assumption.
- **The relay chapter corrects the brief's framing.** "Relays forward encrypted
  packets they cannot read" is true of DERP and iroh *because of the end-to-end
  protocol*, not because of what a relay is. TURN forwards whatever it's given,
  and every relay still sees metadata. File 04 §4.6 makes that distinction.

Not covered: QUIC itself (http file 04), DHT-based peer *discovery* in large
overlays like BitTorrent or IPFS beyond publishing one peer's address, onion
routing, and NAT64/DNS64 transition mechanisms. Each would be its own chapter.

## How to use me

Ask things like:

- "My users on one mobile network never connect directly. Walk me through how to
  find out why."
- "Draw the packets for hole punching between these two NAT types."
- "Is this success-rate claim measured or estimated? What does it count?"
- "Where would a man-in-the-middle sit in this design, and what stops them?"
- "Should this feature use iroh, WebRTC, or a VPN like Tailscale? What am I
  trading?"
- "Explain this part of iroh's source line by line."
- "Is my mental model right? Here's what I think happens when the direct path
  dies…"
