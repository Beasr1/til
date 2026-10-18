# 2. How NATs behave

## The problem

"Behind a NAT" isn't one situation. Two peers behind two home routers might
connect directly in under a second, while the same software between a phone on
mobile data and a corporate laptop never manages it. The difference comes down to
two independent choices each NAT makes, plus how long it remembers things. You
can't reason about hole punching ([file 03](03-hole-punching.md)) until you can
name those choices.

The vocabulary here is RFC 4787, "NAT Behavioral Requirements for Unicast UDP"
(BCP 127, January 2007). It's about UDP because that's what hole punching mostly
uses; RFC 5382 does the same job for TCP.

## 2.1 Mapping: does my external port depend on who I'm talking to?

Your laptop sends from internal `X:x` to a server `Y1:y1`. The NAT picks external
`X1′:x1′`. Now the same socket sends to a *different* destination `Y2:y2`. Does
the NAT reuse `X1′:x1′`, or allocate a new external port?

RFC 4787 §4.1 names three answers:

| Behaviour | Reuses the mapping when… | In RFC 4787's words |
|---|---|---|
| **Endpoint-Independent Mapping** (EIM) | Always, for any destination | "X1′:x1′ equals X2′:x2′ for all values of Y2:y2" |
| **Address-Dependent Mapping** (ADM) | Only if the destination IP is the same | "X1′:x1′ equals X2′:x2′ if and only if, Y2 equals Y1" |
| **Address and Port-Dependent Mapping** (APDM) | Only if destination IP *and* port are the same | "…if and only if, Y2:y2 equals Y1:y1" |

ADM and APDM are often lumped together as **endpoint-dependent mapping** (EDM).
Tailscale's write-up calls EIM NATs "easy" and EDM NATs "hard", which is the
distinction that matters in practice.

```
  Same internal socket 192.168.1.23:51000, two destinations:

  EIM:   → STUN server 203.0.113.10:3478   uses 198.51.100.7:62001
         → peer        192.0.2.44:41641    uses 198.51.100.7:62001   ← same: predictable
  APDM:  → STUN server 203.0.113.10:3478   uses 198.51.100.7:62001
         → peer        192.0.2.44:41641    uses 198.51.100.7:39874   ← new: unpredictable
```

Why this is the property that decides hole punching: you learn your reflexive
address by asking a server ([file 03 §3.1](03-hole-punching.md#31-stun-asking-whats-my-address)).
Under EIM, the address the server saw is the address your peer will see too, so
you can tell the peer where to send. Under EDM, the server saw one port and the
peer will see another that nobody has told you about. RFC 4787 makes EIM a hard
requirement for exactly this reason:

> REQ-1: A NAT MUST have an "Endpoint-Independent Mapping" behavior.
>
> Justification: In order for UNSAF methods to work, REQ-1 needs to be met.
> Failure to meet REQ-1 will force the use of a UDP relay, which is very often
> impractical.

("UNSAF" is UNilateral Self-Address Fixing: learning your own address from
outside, which is what STUN does.)

## 2.2 Filtering: who may send to my mapping?

Once a mapping exists, which inbound packets may use it? RFC 4787 §5 again names
three:

| Behaviour | Admits packets from `Z:z` if the inside host has sent to… |
|---|---|
| **Endpoint-Independent Filtering** (EIF) | Anyone at all. Any packet to the mapped address gets in |
| **Address-Dependent Filtering** (ADF) | Z's IP address, on any port |
| **Address and Port-Dependent Filtering** (APDF) | Exactly `Z:z` |

RFC 4787 doesn't mandate one. REQ-8 recommends EIF "if application transparency
is most important" and ADF "if a more stringent filtering behavior is most
important". APDF isn't recommended at all, and the justification says why in
terms that will matter in [file 03](03-hole-punching.md):

> When the endpoints use ICE, if NAT-A uses Address and Port-Dependent Filtering,
> connectivity will require a UDP relay. However, if NAT-A uses
> Endpoint-Independent Filtering or Address-Dependent Filtering, ICE will
> ultimately find connectivity without requiring a UDP relay.

Filtering is the easier obstacle. A filter only asks "have you sent to them?",
and hole punching answers it by sending. Mapping is the hard one, because no
amount of sending tells you a port the NAT hasn't revealed.

> **Teacher's aside.** People assume a NAT that changes ports per destination is
> the "secure" kind. RFC 4787 §4.1 says flatly that it isn't: "these three
> possible choices make no difference to the security properties of the NAT. The
> security properties are fully determined by which packets the NAT allows in and
> which it does not. This is determined by the filtering behavior." Mapping
> decides whether peer-to-peer works; filtering decides what gets in. A vendor
> that ships endpoint-dependent mapping "for security" bought no security and
> broke hole punching. The common real-world combination, EIM with strict
> filtering, gets you both.

## 2.3 The old names, and where they mislead

You'll meet four older names everywhere: in Stack Overflow answers, in game
consoles' network test screens, and in libraries. They come from RFC 3489, the
original STUN (2003), which tried to classify NATs into four types:

| RFC 3489 name | Mapping | Filtering | Notes |
|---|---|---|---|
| **Full cone** | EIM | EIF | Anyone can send to the mapped address |
| **Restricted cone** | EIM | ADF | Must have sent to that IP |
| **Port-restricted cone** | EIM | APDF | Must have sent to that IP and port |
| **Symmetric** | APDM | APDF | New mapping per destination; only that destination may reply |

Every "cone" is EIM; the cone names differ only in filtering. "Symmetric" is the
only one with endpoint-dependent mapping. That table is the translation, and it
also shows the problems with the old scheme:

1. **It collapses two independent dimensions into one list.** A NAT with
   address-dependent mapping, or with APDM and lenient filtering, has no name.
   RFC 4787 §3: the terminology "has been the source of much confusion, as it has
   proven inadequate at describing real-life NAT behavior."
2. **Real NATs don't hold still.** Many preserve your source port until there's a
   collision and then switch strategy, or behave differently under load. RFC 5389
   (STUN's 2008 revision) dropped the classification algorithm because it "was
   found to be faulty, as many NATs did not fit cleanly into the types defined
   there". RFC 4787's REQ-11, "A NAT MUST have deterministic behavior", exists
   because plenty don't.
3. **"Full cone" gets used for any easy NAT.** Tailscale's David Anderson notes
   that most of the internet calls easy NATs full cone "when these days they're
   much more likely to be Port-Restricted Cone". The difference is filtering,
   which hole punching handles anyway, so the label is wrong in a way that
   happens not to matter, which is how wrong labels survive.
4. **The names are UDP-only.** RFC 3489 classified UDP behaviour. A NAT can treat
   TCP quite differently (RFC 5382).

> ⚠️ **A "NAT type" test result is a snapshot.** Tools that print "your NAT type:
> port-restricted cone" ran a few probes against a couple of servers at one
> moment. The same NAT may allocate ports differently once its table fills up, or
> behave differently for IPv6, or for a second device. Treat the result as one
> observation, not a property of the network.

## 2.4 Two more properties that bite

**Port preservation.** Some NATs try to keep your internal source port as the
external port when it's free. It makes NATs look predictable in a lab and then
change behaviour when two devices on the same network use the same port. RFC
4787 §4.2.1 discusses it and doesn't require it.

**Hairpinning.** Two devices behind the *same* NAT, each knowing only the other's
reflexive address. A packet from one to the NAT's own public address has to turn
around inside the NAT and go back in. RFC 4787 REQ-9 says a NAT "MUST support
'Hairpinning'". Measured support was poor: Ford, Srisuresh and Kegel's 2005
survey (more in [file 03 §3.5](03-hole-punching.md#35-how-often-does-it-work)) found
UDP hairpinning in 80 of 335 reported NATs, 24%, in a self-selected sample.
Tailscale's 2020 write-up still describes "a great many otherwise well-behaved NAT
devices" without it. That's why peers exchange their *private* addresses as well
as their reflexive ones: two machines on the same network should find each other
directly, without relying on the NAT to hairpin.

## 2.5 Timeouts and keepalives

A mapping lives as long as packets keep using it. Go quiet and the NAT deletes the
entry; the next inbound packet hits nothing and is dropped, and the peer on the
other side sees silence rather than an error.

What the standard says (RFC 4787 §4.3):

| Requirement | Text |
|---|---|
| REQ-5 | "A NAT UDP mapping timer MUST NOT expire in less than two minutes", with an exception for well-known destination ports (0–1023) |
| REQ-5c | "A default value of five minutes or more for the NAT UDP mapping timer is RECOMMENDED" |
| REQ-6 | Outbound packets MUST refresh the mapping. Inbound refresh is optional, because it would let an outsider hold a mapping open forever |

What practitioners design for is much shorter. Tailscale's 2020 write-up: "a
common value for UDP is 30 seconds". RFC 9308 §3.2, the QUIC applicability
document, cites a 2010 study suggesting a binding can expire "after just thirty
seconds of inactivity" (see [http file 04](../http/04-quic.md#connection-ids-and-migration)).
Even TURN's own spec warns, in RFC 8656 §9, that "many NATs deployed at the time of
publication expire their UDP bindings considerably faster" than five minutes.

So implementations keep paths warm far more often than the RFC minimum would
suggest. iroh, as one data point, sends a keepalive on any path idle for 5
seconds and gives up on a path after 15 seconds of silence
(`iroh/src/socket.rs:109` and `:117` at tag v1.3.0), and its deployment guide
asks network administrators for a UDP idle timeout of at least 30 seconds.

Two consequences follow:

- **Keepalives cost power on phones.** A packet every few seconds keeps the radio
  awake. Peer-to-peer apps on mobile trade connection freshness against battery,
  and often let idle connections die and reconnect instead.
- **Mappings can change under you.** A NAT that drops and recreates a mapping
  gives the flow a new external port: **NAT rebinding**. TCP connections die when
  this happens. QUIC survives it, because connection IDs rather than the 4-tuple
  identify the connection ([http file 04](../http/04-quic.md#connection-ids-and-migration)),
  which is one reason modern peer-to-peer stacks are built on QUIC.

## 2.6 Asking the NAT directly

If the NAT would just open a port when asked, none of the guessing would be
necessary. Three protocols exist for that:

| Protocol | From | Notes |
|---|---|---|
| **UPnP IGD** | UPnP Forum, late 1990s | XML/SOAP over HTTP on the LAN. Widely implemented, widely disabled |
| **NAT-PMP** | Apple; RFC 6886 (2013) | Small binary UDP protocol |
| **PCP**, Port Control Protocol | RFC 6887 (2013) | NAT-PMP's successor. Designed to also work with carrier-grade NATs and IPv6 firewalls |

RFC 6887 §1 describes the purpose: PCP "provides a mechanism to control how
incoming packets are forwarded by upstream devices such as … NAT44 … and IPv6 and
IPv4 firewall devices".

When it works, it's the best outcome: a real inbound mapping, no punching needed.
The catch is availability. Tailscale's write-up explains that these protocols
"might be disabled by default … might have been disabled by policy. Disabling by
policy is fairly common because UPnP suffered from a number of high-profile
vulnerabilities." And it only helps with the NAT you can talk to: your home
router's UPnP does nothing about a carrier-grade NAT upstream. So systems try it
opportunistically and never depend on it. iroh asks the gateway via UPnP, NAT-PMP
and PCP by default, and documents how to turn that off.

## 2.7 Carrier-grade NAT: two NATs in a row

There aren't enough IPv4 addresses for every household to have one, so many ISPs,
and nearly all mobile networks, run a NAT of their own. Your home router gets a
"public" address that is in fact private to the ISP, often from `100.64.0.0/10`,
the block RFC 6598 reserved for this. The ISP's **carrier-grade NAT** (CGNAT, or
CGN) translates again on the way out.

```
  laptop ──► home router NAT ──► ISP CGN ──► internet
  192.168.1.23   100.64.12.9        198.51.100.7
                 (RFC 6598 space)   (shared by many customers)
```

What this does to everything in this chapter:

- **Your reflexive address is the outermost NAT's.** A STUN server sees the CGN's
  address and port. That's correct (it's what a peer must send to), but your
  router's admin page now shows a different "public" address from the one the
  world sees.
- **Both NATs must cooperate.** Hole punching through two layers works only if
  both have predictable mappings. One endpoint-dependent layer anywhere in the
  path makes the whole path endpoint-dependent.
- **The CGN's behaviour isn't yours to configure.** RFC 6888 (BCP 127's companion
  for CGNs) requires a CGN to meet RFC 4787's requirements, including EIM (its
  REQ-1), recommends endpoint-independent filtering (REQ-7, "a stronger form of
  REQ-8 from [RFC4787]", because "some games and peer-to-peer applications require
  EIF"), and says a CGN "MUST implement a protocol giving subscribers explicit
  control over NAT mappings", which "SHOULD be the Port Control Protocol" (REQ-9).
  Whether a particular carrier does any of that is another matter.
- **Ports are rationed.** RFC 6888 REQ-4: a CGN "MUST support limiting the number
  of external ports … assigned per subscriber". Opening many sockets to improve
  your odds (the trick in [file 03 §3.4](03-hole-punching.md#34-the-birthday-trick))
  can hit that limit.
- **Ports are deliberately unpredictable.** RFC 6888 REQ-15: a CGN's port
  allocation "SHOULD make it hard for attackers to guess port numbers". That's a
  good security property and it's also exactly what defeats older port-prediction
  tricks.

## 2.8 IPv6: no translation, still a firewall

IPv6 has enough addresses that every device can have a globally routable one.
No translation means no mapping problem: the address a device sees on its
interface is the address the world sees, and it's the same for every destination.
For hole punching, IPv6 behaves like the best case of EIM.

What doesn't go away is filtering. RFC 6092 (January 2011), "Recommended Simple
Security Capabilities in Customer Premises Equipment (CPE) for Providing
Residential IPv6 Internet Service", recommends that home gateways run a stateful
filter much like an IPv4 NAT's filtering half, with the same UDP timer rules (its
REC-14: not less than two minutes, default five). It's an Informational document,
not a standard. It says itself that "conformance with it is not required", and
its REC-49 even allows a fully open "transparent mode" as the default. So IPv6
behaviour varies by vendor, but the common case is a filter that drops unsolicited
inbound packets, and peers still need to send to each other first. Tailscale's
write-up makes the same point: even "in an IPv6-only world … we're still stuck
with stateful firewalls".

The practical consequence: when both peers have IPv6, hole punching is close to
guaranteed to work, because the hard part (predicting a mapping) is gone. That's
why implementations try IPv6 paths and often prefer them. iroh's default path
selector gives IPv6 a small RTT advantage over IPv4 ([file 06 §6.5](06-iroh-end-to-end.md#65-choosing-and-switching-paths)).

## 2.9 Summary table

| Property | Values | Effect on peer-to-peer |
|---|---|---|
| Mapping | EIM / ADM / APDM | EIM: reflexive address is reusable. EDM: it isn't, and punching needs luck or a relay |
| Filtering | EIF / ADF / APDF | Overcome by sending first. Matters when the *other* side is EDM ([file 03 §3.3](03-hole-punching.md#33-why-it-fails-for-some-pairs)) |
| Mapping timeout | RFC: ≥ 2 min. Practice: design for ~30 s | Silent path death without keepalives |
| Hairpinning | Supported or not | Peers on one network must also try private addresses |
| Port mapping protocols | UPnP / NAT-PMP / PCP, often off | Opportunistic win; never rely on it |
| CGNAT | Present or not | Two layers; outermost decides your reflexive address; ports rationed |
| IPv6 | Available or not | No mapping problem; filtering remains |

## Check yourself

1. Your laptop sends from one UDP socket to a STUN server and then to a peer.
   Under EIM and under APDM, what external port does the peer see, and can you
   predict it in each case?
2. A colleague says "our office NAT is symmetric, so it's very secure". Using
   RFC 4787's own reasoning, explain what's wrong with that, and what property
   actually determines what gets in.
3. A NAT has endpoint-independent mapping and address-dependent filtering. Which
   RFC 3489 name fits it? Now take a NAT with address-dependent mapping and
   endpoint-independent filtering. What's its RFC 3489 name, and what does that
   tell you about the old scheme?
4. A peer-to-peer connection works for a few minutes, then a long pause in the
   conversation, then nothing gets through, with no error on either side.
   Explain mechanically what happened, and why RFC 4787's two-minute minimum
   doesn't protect you.
5. Your phone on mobile data is behind a carrier-grade NAT. You run a STUN check
   and your home-style router analysis says "EIM". Why might hole punching still
   fail, and what two CGN requirements in RFC 6888 would help if the carrier
   followed them?
6. Two peers both have working IPv6. Explain why hole punching is near-certain to
   succeed, and why it's still necessary at all.

(Answers in [09-exercises.md](09-exercises.md).)
