# 8. Glossary

Reference, not reading. Each entry points to where the term is taught.

## Addresses and NATs

| Term | Meaning | Taught in |
|---|---|---|
| **Private address** | An address from the RFC 1918 blocks (`10/8`, `172.16/12`, `192.168/16`), never routed on the public internet | 01 §1.1 |
| **Shared address space** | `100.64.0.0/10`, reserved by RFC 6598 for the link between an ISP's CGN and customers | 01 §1.1, 02 §2.7 |
| **NAT** | A device that rewrites addresses and ports so many internal hosts share one external address | 01 §1.2 |
| **Mapping** (or binding) | A NAT's record that internal `IP:port` X currently uses external `IP:port` X′ | 01 §1.2 |
| **Reflexive address** | The external `IP:port` a NAT gave you, as seen from outside. "Server-reflexive" if learned from a server | 01 §1.2, 03 §3.1 |
| **Firewall / filtering** | Deciding which packets may pass. In RFC 4787, a NAT's filtering behaviour | 01 §1.3, 02 §2.2 |
| **Stateful firewall** | A filter that admits inbound packets matching flows its hosts started | 01 §1.3 |
| **4-tuple** | Source IP, source port, destination IP, destination port | 01 §1.6 |
| **EIM** | Endpoint-Independent Mapping: same external port for every destination. Required by RFC 4787 REQ-1 | 02 §2.1 |
| **ADM / APDM** | Address-Dependent / Address and Port-Dependent Mapping: a new external port per destination IP, or per IP and port | 02 §2.1 |
| **EDM** | Endpoint-dependent mapping: ADM or APDM. Tailscale's "hard NAT" | 02 §2.1 |
| **EIF / ADF / APDF** | Endpoint-Independent / Address-Dependent / Address and Port-Dependent Filtering | 02 §2.2 |
| **Full cone, restricted cone, port-restricted cone, symmetric** | RFC 3489's obsolete NAT types. Cones are EIM with EIF, ADF or APDF; symmetric is APDM | 02 §2.3 |
| **Easy / hard NAT** | Tailscale's informal names for EIM and EDM NATs | 02 §2.1 |
| **Port preservation** | A NAT reusing the internal source port as the external port when free | 02 §2.4 |
| **Hairpinning** | A NAT looping a packet sent to its own public address back to an internal host. Required by RFC 4787 REQ-9, often missing | 02 §2.4 |
| **Mapping timeout** | How long an idle mapping survives. RFC 4787: ≥ 2 min; practice: design for ~30 s | 02 §2.5 |
| **Keepalive** | A packet sent only to stop a mapping or path from expiring | 02 §2.5 |
| **NAT rebinding** | A NAT giving an existing flow a new external port, often after a timeout | 02 §2.5 |
| **UPnP IGD / NAT-PMP / PCP** | Protocols for asking a gateway to open a port. PCP is RFC 6887 | 02 §2.6 |
| **CGNAT / CGN** | Carrier-grade NAT, run by the ISP; makes a second layer of NAT | 02 §2.7 |
| **IPv6 simple security** | RFC 6092's recommended stateful filter for home IPv6 gateways | 02 §2.8 |

## Traversal

| Term | Meaning | Taught in |
|---|---|---|
| **NAT traversal** | Every technique for connecting through NATs and firewalls | 01 §1.6 |
| **Signalling / rendezvous** | The channel peers use to exchange candidates before a direct path exists | 01 §1.5 |
| **STUN** | RFC 8489. A protocol for learning your reflexive address and keeping mappings alive; also the format of ICE checks | 03 §3.1 |
| **Binding request** | STUN's one method: "what address did this come from?" | 03 §3.1 |
| **XOR-MAPPED-ADDRESS** | The STUN attribute carrying your reflexive address, XORed to survive NAT ALGs | 03 §3.1 |
| **ALG** | Application Layer Gateway: NAT logic that rewrites addresses inside payloads | 03 §3.1 |
| **QUIC Address Discovery (QAD)** | An IETF draft that does STUN's job inside QUIC with an `OBSERVED_ADDRESS` frame | 03 §3.1, 06 §6.4 |
| **Hole punching** | Both peers sending to each other so each NAT creates the state the other's packets need | 03 §3.2 |
| **Simultaneous open** | Both sides initiating the same flow at once. For TCP, crossing SYNs | 03 §3.2, §3.6 |
| **Birthday trick** | Opening many ports on one side and probing randomly from the other to beat an EDM NAT | 03 §3.4 |
| **Port prediction** | Guessing the next port a NAT will allocate; defeated by randomised allocation | 03 §3.4 |
| **ICE** | RFC 8445. The procedure for gathering, exchanging, pairing and checking candidates | 03 §3.7 |
| **Candidate** | An address a peer might be reached at: host, server-reflexive, peer-reflexive or relayed | 03 §3.7 |
| **Peer-reflexive candidate** | An address discovered because a check arrived from it unexpectedly | 03 §3.3, §3.7 |
| **Connectivity check** | A STUN Binding request sent on a candidate pair; tests and punches at once | 03 §3.7 |
| **Controlling / controlled agent** | The ICE agent that nominates the pair to use, and the one that waits | 03 §3.7 |
| **Nomination** | The controlling agent choosing the pair; RFC 8445 deprecated "aggressive" nomination | 03 §3.7 |
| **Trickle ICE** | RFC 8838. Sending candidates as they're found | 03 §3.7 |
| **DCUtR** | libp2p's "Direct Connection Upgrade through Relay" hole-punching protocol | 03 §3.5 |
| **QNT** | "QUIC NAT traversal": the drafts (Seemann's; n0's variant) that move hole punching into QUIC | 06 §6.4 |
| **ADD_ADDRESS / REACH_OUT** | n0 QNT frames: the server's candidates, and the client starting a punching round | 06 §6.4 |
| **NAT probe** | In n0 QNT, a QUIC packet with a `PATH_CHALLENGE` sent to a candidate | 06 §6.4 |

## Relays

| Term | Meaning | Taught in |
|---|---|---|
| **Relay** | A public server that forwards packets between peers that can't connect directly | 04 |
| **TURN** | RFC 8656. The IETF relay protocol, an extension of STUN | 04 §4.2 |
| **Allocation / relayed transport address** | A TURN client's reserved `IP:port` on the server | 04 §4.2 |
| **Permission** | TURN's per-peer-IP filter on an allocation; lasts 5 minutes | 04 §4.2 |
| **Channel / ChannelData** | TURN's low-overhead framing: 4-byte header instead of 36 bytes | 04 §4.2 |
| **DERP** | Tailscale's key-addressed relay over HTTPS. Expanded as "Designated Encrypted Relay for Packets" (current docs) or "Detoured Encrypted Routing Protocol" (2020 blog) | 04 §4.3 |
| **Peer relay** | A Tailscale relay run on your own node, tried before DERP | 04 §4.3 |
| **Home relay** | The relay an endpoint stays connected to and publishes as its contact point | 04 §4.4, 06 §6.2 |
| **iroh relay** | iroh's DERP-derived relay over HTTPS/WebSocket, addressed by `EndpointId` | 04 §4.4 |

## Identity and discovery

| Term | Meaning | Taught in |
|---|---|---|
| **Public-key addressing** | Naming a peer by its public key rather than an IP | 01 §1.6, 05 §5.1 |
| **Self-certifying name** | A name you can verify using only the name: here, a public key | 05 §5.1 |
| **Raw public key (RPK)** | RFC 7250. Sending a bare `SubjectPublicKeyInfo` in TLS instead of a certificate | 05 §5.2 |
| **SubjectPublicKeyInfo (SPKI)** | The X.509 structure holding an algorithm identifier and key bytes | 05 §5.2 |
| **EndpointId** | iroh's peer identifier: a 32-byte Ed25519 public key. `NodeId` before 0.94 | 05 §5.1 |
| **EndpointAddr** | iroh's `{ EndpointId, set of relay URLs and direct addresses }`. `NodeAddr` before 0.94 | 05 §5.6 |
| **Address lookup** | iroh's name for resolving an `EndpointId` to addresses. "Discovery" before 0.96 | 05 §5.5 |
| **Pkarr** | Public-key addressable resource records: DNS records signed by an Ed25519 key | 05 §5.5 |
| **Mainline DHT** | BitTorrent's distributed hash table, usable for publishing pkarr records | 05 §5.5 |
| **mDNS** | Multicast DNS: finding peers on the local network with no server | 05 §5.5 |
| **Ticket** | iroh's shareable string: type prefix + base32 postcard-encoded `EndpointAddr` (+ app data) | 05 §5.6 |
| **Fingerprint (WebRTC)** | A hash of a peer's DTLS certificate, sent in SDP via signalling | 05 §5.4 |

## QUIC terms used here

Taught properly in [http file 04](../http/04-quic.md); listed for lookup.

| Term | Meaning | Taught in |
|---|---|---|
| **Path** | One 4-tuple a QUIC connection sends on | 06 §6.5 |
| **QUIC multipath** | An IETF draft letting one connection use several paths at once, each with its own RTT and congestion state | 06 §6.5 |
| **PATH_CHALLENGE / PATH_RESPONSE** | QUIC's path validation frames, reused as NAT probes | 06 §6.4 |
| **noq** | n0's fork of the Quinn QUIC implementation, used by iroh since 0.97 | 06 §6.1 |
| **DATAGRAM frame** | RFC 9221. Unreliable, congestion-controlled, unfragmentable QUIC data | 07 |
