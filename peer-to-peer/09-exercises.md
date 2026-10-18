# 9. Exercises and answers

Try each question before reading the answer. Getting one wrong is data, not
failure. The answers are worked rather than stated, because the reasoning is the
part that transfers to a network you haven't seen.

---

## File 01 — Foundations

**1. A colleague sends UDP to your public IP, port 4000. Nothing arrives. Two reasons; which does IPv6 remove?**

First, **translation**: the packet reaches your router's public address, and the
router has no mapping for port 4000. Nothing inside has sent from a socket that
the NAT associated with external port 4000, so the router has no idea which of the
devices behind it should get the packet. Second, **filtering**: even if a mapping
existed, a stateful firewall admits only packets that answer a flow an inside host
started, and your colleague's packet answers nothing.

IPv6 removes the first. Your laptop has a globally routable address, so the
packet can be addressed to it directly with no translation. It doesn't remove the
second: a home gateway following RFC 6092 still drops unsolicited inbound
traffic.

**2. Why does a normal HTTPS request work through the same NAT?**

Because your laptop speaks first. Its outbound SYN to the server creates a
**mapping** in the NAT (internal `192.168.1.23:51000` ↔ external
`198.51.100.7:62001`) and **state** in the firewall ("replies from
`203.0.113.10:443` to this mapping are expected"). The server's SYN-ACK arrives at
`198.51.100.7:62001`, matches both, and is translated and delivered. The server
itself needed no such help because it has a public address and accepts
unsolicited connections on 443. Client–server works because exactly one side is
reachable.

**3. Why can't the two peers just send to each other's reflexive addresses whenever they like?**

Each NAT drops packets from the other peer until its own host has sent to that
peer, so whoever sends first loses their packets. That's fine as long as the other
side sends soon afterwards, *while the first side's state is still alive*, and
keeps retrying. So the peers need to agree to start at roughly the same time, and
they need each other's addresses in the first place. Both require a channel that
works before the direct path exists: **signalling** (or rendezvous). It carries
each side's candidate addresses (private, reflexive, relayed), a signal to start,
and in practice the keys each side will use to authenticate.

**4. "We're peer-to-peer, so we don't need infrastructure."**

The roles that remain, and what breaks without each:

| Role | Without it |
|---|---|
| Address discovery (STUN or equivalent) | Peers don't know their own reflexive addresses, so they can't tell each other where to send |
| Signalling / rendezvous | Peers can't exchange candidates or coordinate punching at all |
| Lookup / directory | A peer that changed network can't be found again |
| Relay | The pairs that can't punch (measured at roughly 30% per attempt in the best recent study) never connect |

The servers carry little traffic except the relay, but none of them is optional.

**5. It works every time between two office laptops. Why does that tell you almost nothing?**

Two reasons, either of which is enough. The laptops share a network, so they
connect over private addresses with no NAT crossed; hole punching, relays and
timeouts are never exercised. And even if they went through a NAT, it's one NAT
with one behaviour, which is very likely the generous kind; mobile CGNATs and
strict corporate filters, which are where failures come from, aren't in the test.

A minimal useful setup: one peer on home broadband, the other on a phone hotspot
over mobile data (likely CGNAT). Add a run where UDP is blocked or the system is
forced to relay-only, so you know the fallback works. Then measure in production
what fraction of connections end up relayed.

---

## File 02 — How NATs behave

**1. EIM vs APDM: what port does the peer see?**

Under **EIM**, the NAT reuses the same external port for every destination from
that socket, so the peer sees the same `198.51.100.7:62001` the STUN server
reported. It's predictable, so you can tell the peer in advance. Under **APDM**,
the first packet to the peer's `IP:port` creates a fresh mapping with a port the
NAT chooses, say `:39874`. Nothing you've done so far reveals that number, so you
can't tell the peer where to send.

**2. "Our NAT is symmetric, so it's very secure."**

RFC 4787 §4.1 says the three mapping behaviours "make no difference to the
security properties of the NAT. The security properties are fully determined by
which packets the NAT allows in and which it does not", and that's the
**filtering** behaviour. "Symmetric" in the RFC 3489 sense is
address-and-port-dependent *mapping* (plus strict filtering). The strict filtering
part is what keeps strangers out, and it's available with EIM too (the
"port-restricted cone"). The mapping part adds no protection and breaks hole
punching.

**3. EIM + ADF, and ADM + EIF: RFC 3489 names?**

EIM with address-dependent filtering is a **restricted cone**. ADM with
endpoint-independent filtering has **no RFC 3489 name**: the cones all assume EIM,
and "symmetric" assumes port-dependent mapping with strict filtering. That's the
old scheme's basic flaw. It treats NAT behaviour as one list of four types when it
is two independent dimensions of three values each, nine combinations, plus
timers, hairpinning and inconsistency over time. RFC 4787 §3 abandoned it as
"inadequate at describing real-life NAT behavior".

**4. Works for minutes, pause, then silence.**

During the pause no packets used the mapping, and the NAT expired it (or the
firewall forgot the flow). The next packet from the far side arrives at an
external port with no mapping and is silently dropped: NATs don't send errors for
this, so neither side sees anything but silence. If the near side sends first,
the NAT may create a *new* mapping with a different port (rebinding), which the
far side doesn't know about.

RFC 4787 REQ-5 says a UDP mapping "MUST NOT expire in less than two minutes", but
that's a requirement on NAT vendors, not a guarantee about the NAT in front of
your user. Practitioners design for around 30 seconds (Tailscale's write-up; RFC
9308 cites a 2010 study at thirty seconds), and TURN's own RFC warns many NATs
expire "considerably faster". The fix is keepalives well inside that window, which
is why iroh sends one on any path idle for 5 seconds.

**5. Phone behind CGNAT; your analysis says "EIM". Why might punching still fail?**

Several possible reasons. A single STUN answer can't establish EIM at all; you
need two servers at different IPs to see whether the port varies, and even then a
NAT can change strategy under load. The CGN may have strict
address-and-port-dependent filtering, which fails against an EDM peer (file 03
§3.3). And it rations ports per subscriber, which defeats the many-ports birthday
trick.

RFC 6888 would help on two counts if the carrier followed it: **REQ-1** requires a
CGN to meet RFC 4787's behavioural requirements, which include EIM; **REQ-7**
recommends endpoint-independent filtering explicitly because "some games and
peer-to-peer applications require EIF". (REQ-9, PCP support, would let the
application simply ask for a mapping.)

**6. Both peers have IPv6. Why is punching near-certain, and why still necessary?**

Near-certain because there's no translation: each peer's address is the address
the world sees, the same for every destination. That's the best case of EIM, and
both peers know each other's exact address in advance. Still necessary because
the gateways typically run a stateful filter that drops unsolicited inbound
packets (RFC 6092's recommended behaviour, though not mandatory and with a
"transparent mode" allowed), so each side must send to the other before the
other's packets are admitted. It's simultaneous open with nothing left to guess.

---

## File 03 — Hole punching

**1. Why does STUN put the address in the body, and why XOR it?**

The NAT rewrites the packet's IP and UDP headers on the way back in, so the
response's headers carry your *private* address by the time you read them. The
server writes what it saw into the body, which NATs don't normally touch, so you
can read your reflexive address there.

"Normally" is the catch. Some NATs ran generic ALGs that scanned payloads for
their own public IP as 32-bit binary values and rewrote them to the internal
address, trying to help protocols that embed addresses. A plain `MAPPED-ADDRESS`
got rewritten to your private address, which made the answer useless and broke
STUN's integrity check (RFC 8489 §14.2). XORing with the magic cookie means the
public IP never appears in the payload as itself, so the ALG doesn't recognise it.

**2. A's first packet dies at NAT B. What did it achieve, and what if A sends only one?**

It created state in A's own NAT: a mapping (if one didn't exist) and a filter
entry admitting packets from B's address. That's what lets B's packet in moments
later.

If A sends exactly one packet and stops, B's packet still gets in to A, as long as
it arrives before A's state expires. But B's own NAT only opens for A once B has
sent, and A's single packet arrived before that and was dropped. So unless A
sends again, B never hears from A: the path is open in one direction only. And if
B was slow, A's state may have expired too. That's why implementations retry for
several seconds on every candidate pair and consider a path up only once both
directions have been confirmed.

**3. A: EIM + ADF; B: APDM. Trace it. Then make A's filtering APDF.**

B learned its reflexive address `192.0.2.44:41641` from a STUN server and told A.
When B sends to A's reflexive address, NAT B creates a *new* mapping toward A,
say `:50312`. That packet arrives at NAT A from `192.0.2.44:50312`. A has been
sending to `192.0.2.44:41641`. Under **address-dependent filtering** only the IP
matters, and A has sent to `192.0.2.44`, so the packet is admitted. A's software
sees the true source port, `50312`, records it as a peer-reflexive candidate, and
replies there. NAT B has state allowing A's address in (B sent to it), so the
reply gets through. Direct path established.

Make A's filtering **address-and-port-dependent** and the first step fails: A has
sent to `:41641`, not `:50312`, so B's packet is dropped. A's packets to `:41641`
reach NAT B on the STUN mapping, whose filter admits only the STUN server, so
they're dropped too. Neither side can learn anything new. This is the case RFC
4787's REQ-8 justification describes as requiring a relay. The birthday trick
(§3.4) can sometimes rescue it.

**4. STUN from a helper socket, data from the main socket.**

A NAT mapping belongs to one internal `IP:port`. The helper socket's STUN request
creates a mapping for the *helper's* port, and that's the reflexive address your
service advertises. The data socket has its own mapping with a different external
port, even behind a perfectly EIM NAT. The peer sends to the helper's mapping and
reaches the wrong socket, or nothing.

Why some users still connect: when the *peer's* NAT filters leniently (EIF or
ADF), packets from your data socket, arriving from its real external port, are
admitted. The peer learns that port as a peer-reflexive candidate and replies to
it. It fails when the peer's side filters by address and port. The fix is to run
STUN, hole punching and data on one bound UDP socket, and demultiplex replies
(STUN messages are identifiable by the magic cookie `0x2112A442`).

**5. "Direct connections succeed 9 times out of 10." What to ask?**

- **Nine out of ten what?** Devices, network configurations, connection
  attempts, or user sessions? Ford 2005 counted NAT devices; Trautwein et al.
  counted attempts.
- **Conditional on what?** Does the figure exclude cases where the relay or
  address discovery failed first? (The 2025 study's figure does, about 29% of its
  data.)
- **Sampled from whom?** Volunteers, the vendor's own users, a lab? Volunteer
  samples skew towards easier networks, and the 2025 authors say theirs may
  overestimate success.
- **Measured or estimated?** Tailscale's "over 90%" is explicitly an estimate;
  iroh's "roughly 9 out of 10" has no published methodology.

The best-measured figure I know of is **70% ± 7.1%** for the hole-punching stage
of libp2p's DCUtR across 4.4 million attempts (Trautwein et al., 2025),
conditional on a relayed connection and address discovery having already
succeeded, from volunteer clients.

**6. Why prefer host over relay, and when does a relayed pair still win?**

The type preferences make the priority formula sort host pairs first (direct, no
NAT crossing, lowest latency), then peer- and server-reflexive pairs (direct
through NATs), and relayed pairs last, because relaying costs the operator
bandwidth and adds latency. ICE nominates the highest-priority pair that passes
its checks, so a relay is used only if nothing direct works.

A relayed pair is nominated when every direct pair fails its checks. For example,
both peers are behind endpoint-dependent NATs, so no reflexive address is valid
for the peer; or one peer is on a corporate network that blocks all outbound UDP
except to its TURN server over TLS on port 443. Each peer has host and reflexive
candidates, but no check on those pairs ever gets a response.

---

## File 04 — Relays

**1. Allocation but no permission: what happens?**

The TURN server receives the peer's datagrams on the relayed address, checks the
allocation's permission list, finds no entry for the peer's IP, and "silently
discard[s]" them (RFC 8656 §3.3). Permissions exist so a TURN allocation doesn't
become an open inbound port into a protected network: RFC 8656 says they were
added "to ease concerns amongst enterprise IT administrators that TURN could be
used to bypass corporate firewall security", and they "mimic the
address-restricted filtering mechanism of NATs". The client must name who may
reach it, just as a NAT only admits peers the host has sent to.

**2. Why is relay last in ICE, yet first in Tailscale and iroh?**

RFC 8656 §1: relaying "comes at a high cost to the provider of the TURN server
since the server typically needs a high-bandwidth connection to the Internet", so
"it is best to use a TURN server only when a direct communication path cannot be
found". ICE encodes that with type preference 0.

Tailscale and iroh start on the relay for a different reason: every peer is
*already* connected to its home relay, so a connection through it costs one round
trip and works on any network. The relay is also their signalling channel, so the
candidate exchange happens over the very connection the data is flowing on. Then
they upgrade to a direct path within seconds, so the relay carries only the start
of most connections. They pay a little relay bandwidth per connection to get
immediate connectivity and never wait on ICE.

**3. Why run a relay over HTTPS on TCP 443?**

It gains reachability. Some corporate and public networks block outbound UDP, or
allow only web traffic through a proxy; TCP 443 looks like browsing and gets
through almost everywhere that browsing does. It costs real-time behaviour: TCP
delivers in order and retransmits, so one lost packet on the relay leg stalls
everything behind it (head-of-line blocking, the problem QUIC was built to escape),
and the QUIC congestion controller now runs on top of TCP's. For a call, that's
freezes followed by bursts. The design accepts this because the relay path is
meant to be temporary.

**4. An iroh relay without proof of key possession.**

An attacker could connect claiming a victim's `EndpointId`, and the relay would
forward packets meant for the victim to the attacker, or alternate unpredictably
between the two. What the attacker gets is ciphertext encrypted for the victim's
key, which they can't decrypt. They also can't complete a TLS handshake as the
victim, because that needs the victim's private key. So: **denial of service and
traffic analysis, but no reading and no impersonation.** The handshake in
`iroh-relay/src/protos/handshake.rs` closes the DoS by making the client sign a
challenge (or TLS-exported keying material) with the key it claims.

**5. "The relay can't read our traffic, so relays are a solved privacy problem."**

What a relay can still learn or do:

1. **Metadata.** iroh's docs list it: "The identities of the devices connected to
   it, which pairs are exchanging traffic, when, and how much", plus the IP
   addresses connecting.
2. **Drop, delay or throttle** anything, selectively.
3. **Sit in the key exchange.** If peers don't verify each other's keys, the relay
   can terminate encryption on both sides and read everything.

"Can't read" holds only if **each peer verifies, in the handshake, that the far
end holds the key it expected, and the expectation came from a source the relay
doesn't control.**

---

## File 05 — Identity and discovery

**1. Why no CA, and what does the handshake prove with a raw public key?**

A CA exists to vouch that a key belongs to a name. When the name *is* the key,
there's nothing to vouch for: comparing the presented key with the one you dialled
answers "is this the name I asked for?" directly. The handshake proves the
remaining part: through `CertificateVerify`, the peer signed the whole handshake
transcript, including fresh randomness from both sides, with the private key
matching the presented public key. So the far end holds that private key *now*
(the signature can't be replayed), and the session keys are bound to that
handshake. Combined with iroh's comparison, that means "the holder of the key I
dialled is on the other end of this encrypted channel".

**2. The handshake succeeded, so "it's one of our devices"?**

What's established: the remote end holds the private key for some specific
`EndpointId`, which you now know, and all traffic is encrypted between you and it.
What isn't: that this key belongs to anyone you know. iroh's accept side
authenticates every client and authorises none. Its verifier says "Beyond checking
for no intermediates, we don't check the client certificate", and the crate docs
say it's "up to the application to decide if a particular peer is allowed to
connect". The missing code is an authorisation check in the accept path: read the
connection's remote `EndpointId`, compare it with an allow-list, database or
capability, and close the connection if it isn't allowed.

**3. A ticket from a public chat channel "from your colleague".**

The connection proves you're talking to the holder of the key *in the ticket*, and
anyone could have made that ticket with their own key and posted it under your
colleague's name. Encryption to an attacker's key is perfectly strong and perfectly
useless. The weak point is the out-of-band binding RFC 7250 warns about: how the key
reached you. Get keys through a channel that authenticates the sender (in person,
from your own authenticated server, or by comparing a fingerprint over a channel
you trust).

**4. `dns.iroh.link` is compromised.**

The attacker can answer lookups with wrong relay URLs or addresses, or with none.
Plain DNS TXT lookups carry no signature, so the client can't detect it (pkarr
lookups over HTTPS verify the signed packet and would reject a forgery). The
effect is that connections fail or slow down while iroh tries bad addresses, and
the attacker learns which `EndpointId`s people are trying to reach.

What they can't do is impersonate a peer. Whatever address you're sent to, the
TLS handshake still demands proof of the private key for the `EndpointId` you
dialled, and the attacker doesn't have it. The difference is that address lookup
only answers *where*; the handshake answers *who*, and it never trusts the lookup
for that.

**5. WebRTC vs iroh: where the key comes from.**

In WebRTC each side's DTLS certificate fingerprint travels inside the SDP, through
the application's signalling server. RFC 8827 says the resulting assurance is that
"the signaling server claims that the call is from Alice to Bob". A malicious
signalling server can replace each side's fingerprint with its own, then relay
media between two DTLS sessions it terminates: a man in the middle, unless an
identity provider binds the fingerprints to identities.

In iroh, you already hold the peer's `EndpointId` before contacting any relay, and
the relay forwards packets *addressed by* that key. It never supplies the key.
A malicious relay can drop packets, but it can't substitute a key the dialler
checks against its own copy. (If your iroh app fetched `EndpointId`s from your own
server, that server would be in exactly the signalling server's position. The trust
moves; it doesn't vanish.)

**6. Stored tickets, months later.**

A ticket captures the peer's relay URL and direct addresses *as they were when it
was made*. Months later the peer has changed networks, NAT mappings, perhaps home
relay, so iroh tries dead addresses first and then times out or falls back slowly.
And if the peer didn't persist its secret key, it now has a new `EndpointId` and
the ticket points at an identity that no longer exists. Store the `EndpointId` (and
have peers persist their keys) and let address lookup resolve current addresses at
connect time, as iroh's docs recommend.

---

## File 06 — Worked example: iroh

**1. A 2024 tutorial using `NodeAddr` and a STUN server.**

Differences in 1.x: `NodeAddr`/`NodeId` are `EndpointAddr`/`EndpointId` (renamed in
0.94); there's no STUN (removed in 0.94, replaced by QUIC Address Discovery);
`Discovery` is `AddressLookup` (0.96); hole punching is the QNT extension inside
QUIC with multipath (0.96); the QUIC stack is noq (0.97). A 1.x endpoint can't talk
to a 0.3x endpoint: 0.96 was explicitly wire-breaking, and the hole-punching
protocol itself changed.

**2. The app restarts and every saved contact breaks.**

By default `bind()` generates a **new random keypair** on every start, so the
endpoint's `EndpointId` changed, and everyone who saved the old one is dialling an
identity that no longer exists. Relays and DNS are fine; they're just answering for
the new key. The fix is to generate the `SecretKey` once, store it securely, and
load it at every start.

**3. Why can `connect()` return before any punching?**

Because the QUIC handshake ran entirely over the relay: the dialler's packets went
to the acceptor's home relay, addressed by `EndpointId`, and were forwarded. TLS
verified the key and ALPN agreed the protocol, so the connection is complete. At
that moment the relay carries all traffic. The QUIC stack sees the relay path as
an ordinary IP path to an IPv6 Unique Local Address (n0 QNT draft §3.1). Punching
happens afterwards, inside the connection, and adds direct paths alongside.

**4. Why does the server send `ADD_ADDRESS` but only the client `REACH_OUT`?**

RFC 9000 §9 says "Clients are responsible for initiating all migrations"; QUIC
servers don't normally start using new paths. QNT keeps that asymmetry for
*control*: the client decides when a round starts (`REACH_OUT`, carrying its own
candidates), and the server, which can't initiate, contributes its candidates in
advance with `ADD_ADDRESS`. Having one side drive the rounds avoids both sides
starting competing rounds, the same reason ICE has a single controlling agent.
What QNT does add is server-side path validation, so both sides can send probes:
punching needs packets from both ends.

**5. IPv4 at 4 ms vs IPv6 at 5 ms, then IPv4 at 9 ms.**

The default selector subtracts a 3 ms bias from IPv6 RTTs and, within the primary
tier, switches only if the best candidate is at least 5 ms better than the current
path (`biased_rtt_path_selector.rs:19, 23, 179`).

- Current IPv4: biased 4 ms. IPv6: 5 − 3 = 2 ms. Switch only if 2 + 5 ≤ 4, i.e.
  7 ≤ 4, which is false. **Stays on IPv4.** IPv6 is "better" by 2 ms of biased RTT,
  inside the stickiness margin.
- IPv4 rises to 9 ms: is 2 + 5 ≤ 9? Yes. **Switches to IPv6.**

The margin exists so jitter doesn't make the connection flap between paths, each
switch costing a congestion-control ramp.

**6. Connect-then-upgrade vs find-then-connect, on a strict corporate network.**

**Sooner:** connect-then-upgrade. iroh's first round trip goes over the relay,
which is reachable on TCP 443 from almost anywhere, so data flows after one
relayed handshake. ICE must gather candidates and run checks, and on a strict
network the direct checks all time out before the relayed pair is nominated, which
can take seconds.

**Infrastructure load:** for *this* user, about the same: both end up relaying the
whole transfer because nothing direct works. Across all users, ICE puts less on the
relays, because pairs that can go direct never touch TURN at all, whereas iroh
relays the first moments of every connection. That's the trade the teacher's aside
in §6.3 describes.

---

## File 07 — Real-time traffic over a peer-to-peer path

**1. Half-second freezes then a burst, on one corporate network.**

That network almost certainly blocks direct UDP, so those calls stay on the relay
path, which runs over TCP 443. When a packet on the relay leg is lost, TCP holds
every later byte until the retransmission arrives, a round trip or more later, so
the audio datagrams queue up (the freeze). Then they're all delivered at once (the
burst), and a naïve receiver plays them as they come. Relay rate limiting produces
the same pattern by throttling reads. The datagrams were never "lost", which is
exactly the problem: on this path they were delayed. The fix is receiver-side:
timestamp each frame, drop any that arrive too late to play, and let the jitter
buffer absorb what it can.

**2. Why design for 1200 bytes, not your measured MTU?**

A QUIC datagram can't be fragmented, so it must fit in one packet on whichever
path carries it, and on a peer-to-peer connection that path changes (relay,
direct IPv4, IPv6, mobile data) without your involvement. Your measurement applies
to one path on one network. RFC 9000 §14 guarantees every path QUIC uses supports
a maximum datagram size of at least 1200 bytes, so messages sized comfortably
under that (after QUIC overhead) fit on any path the connection might move to.

**3. A jitter buffer tuned in the first ten seconds, then fixed.**

The first seconds of a peer-to-peer call are usually on the relay: higher RTT,
more jitter, maybe TCP stalls. The buffer grows to match. Then the connection
upgrades to a direct path with much lower, steadier latency, and the oversized
buffer adds pure delay for the rest of the call. If the direct path later dies,
the relay's jitter returns while the buffer may have been shrunk for the direct
path. Peer-to-peer connections change path mid-call by design, so buffers must
keep adapting.

**4. Reconcile iroh's "misguided" with the QUIC chapter's datagram case.**

iroh is right that QUIC datagrams aren't raw UDP. They're congestion-controlled and
acknowledged, so a protocol designed to blast UDP won't behave as designed. On a
TCP relay path they don't even give you loss tolerance. And streams that are reset
when stale handle large frames and give per-frame cancellation, which is why MoQ
leans on them.

The QUIC chapter is right that for **small, independent, frequent** updates (an
audio frame, a game position), each of which is worthless once a newer one exists
and fits in one packet, datagrams avoid per-stream overhead and never waste capacity
retransmitting stale data on a direct path. RFC 9221 §2 names exactly those uses.
The deciding questions are size (does it fit in one packet?) and staleness (is a
late copy worth anything?), and the peer-to-peer twist adds a third: what happens
to it on the relay path?

---

## Things to try

1. **See your own NAT's mapping behaviour.** Write a short Go program with
   `github.com/pion/stun` that sends Binding requests from *one* UDP socket to two
   STUN servers at different IPs and prints both `XOR-MAPPED-ADDRESS` results. Same
   port: endpoint-independent mapping. Different: endpoint-dependent. Run it at
   home, on a phone hotspot and on office Wi-Fi. If you use Tailscale,
   `tailscale netcheck` reports the same thing as `MappingVariesByDestIP`.
2. **Watch a path upgrade.** Install `dumbpipe` (built on iroh). Run
   `dumbpipe listen` on one machine and `dumbpipe connect <ticket>` on another
   network, with logging turned up, and look for the moment traffic moves from the
   relay to a direct path. Then block UDP on one side and confirm it stays relayed
   but works.
3. **Decode a ticket.** Paste a ticket into <https://ticket.iroh.computer> and
   identify the `EndpointId`, relay URL and direct addresses. Notice your IP
   addresses are in it.
4. **Look up a peer in DNS.** For an endpoint using the default address lookup,
   z-base-32 encode its `EndpointId` and run
   `dig TXT _iroh.<z32-id>.dns.iroh.link`. Compare what's published with what the
   ticket contained.
5. **Read the verifier.** Read `iroh/src/tls/verifier.rs` at tag v1.3.0 (under 250
   lines) and find the exact line where a connection to the wrong peer fails.
   Then find where the *accept* side would need your authorisation check.
6. **Do the birthday arithmetic yourself.** Compute `1 − (1 − k/65536)^n` for
   k = 256 and n = 174, 1024, 2048, and check Tailscale's table. Then work out
   roughly why two hard NATs need ~170,000 probes for 99.9%.

---

## Questions worth asking me

- "Walk me through what ICE does, packet by packet, for two peers behind one EIM
  and one EDM NAT."
- "How would I build the signalling channel for a WebRTC app, and what does it
  need to protect?"
- "What changes if I put my own relay in front of iroh's? What do I gain in
  privacy and what do I take on operationally?"
- "Explain QUIC multipath's congestion control per path. What happens to
  in-flight data when the selected path dies?"
- "How does libp2p's DCUtR compare with iroh's QNT, step by step?"
- "How would you design authorisation on top of iroh: allow-lists, capability
  tokens, or something in the ALPN?"
- "What does a carrier-grade NAT's port allocation look like in practice, and how
  would I measure whether a mobile network is EDM?"
- "Is there a peer-to-peer design that hides IP addresses from the peer, and what
  does it cost?"
- "Show me the TLS 1.3 handshake for a raw public key exchange, message by
  message."
