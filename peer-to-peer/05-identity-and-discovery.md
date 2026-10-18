# 5. Identity and discovery: naming a peer by its key

## The problem

In client–server you name the server by a DNS name, the name resolves to an IP,
and a certificate proves the far end is the name you dialled
([http file 05](../http/05-certificates-and-trust.md)). None of that fits a peer.
A laptop's IP changes when it moves from home to café; behind a NAT it's shared
with the whole household; and over the life of one connection, packets may travel
via a relay, then a direct IPv4 path, then IPv6. If "who am I talking to?" were
answered by an address, the answer would change mid-conversation. Tailscale's
write-up puts it bluntly: "once you're dynamically switching paths at runtime,
IP-based security becomes meaningless."

And there's no certificate authority for laptops. Nobody is going to issue a
publicly trusted certificate for "Alice's phone".

The answer every modern system converges on: **name the peer by a public key.**
This chapter covers why that works, how the key gets into the TLS handshake, where
the key comes from, and how you get from a key to an address you can send to.

## 5.1 A key as a name

Give every peer a keypair, generated on the device, the private half never leaving
it. Use the public key as the peer's identifier. iroh's `EndpointId` is exactly
this: "the public half of an Ed25519 keypair" (iroh docs, Endpoints), 32 bytes,
printed as 64 hex characters.

This kind of name is **self-certifying**: to check that the far end *is* the
name, you only need the name. Ask it to prove it holds the matching private key,
and you're done. Compare the three questions from [http file 05](../http/05-certificates-and-trust.md#the-three-questions-and-the-three-errors):

| Question in web TLS | With a key as the name |
|---|---|
| 1. Does the far end hold the private key for this certificate? | Unchanged, and now the whole proof: does it hold the private key for the key I dialled? |
| 2. Does the name on the certificate match the name I asked for? | Trivial: the key *is* the name. Compare bytes |
| 3. Does a chain of signatures reach a root I already trust? | Gone. No chain, no CA, no root store |

The third question disappearing is the big win: no certificate authority, no
expiry, no root store in the container image. It's also where the trust goes
missing, which §5.4 deals with.

What you give up is human-readable names. A key says nothing about who owns it.
Mapping "Alice" to a key is now your application's job.

## 5.2 Raw public keys in TLS

TLS was built around certificates, and a key-as-name system has no certificate to
send. You can wrap the key in a self-signed certificate and teach both sides to
ignore everything in it except the key. libp2p's TLS handshake does this: "the host
key is encoded into the libp2p Public Key Extension, which is carried in a
self-signed certificate" (libp2p TLS spec). TLS also has a cleaner way: **raw
public keys**, RFC 7250 (June 2014).

RFC 7250 adds two TLS extensions, `client_certificate_type` and
`server_certificate_type`, with which the two sides agree to send a bare public
key in place of a certificate. The key is encoded as just the
`SubjectPublicKeyInfo` structure from an X.509 certificate, the algorithm
identifier and the key bytes, and nothing else. TLS 1.3 carries these through:
RFC 8446 §4.4.2 says that if "the RawPublicKey certificate type was negotiated,
then the certificate_list MUST contain no more than one CertificateEntry".

The handshake's proof of possession is unchanged. `CertificateVerify` is still "a
signature over the entire handshake using the private key corresponding to the
public key in the Certificate message" (RFC 8446 §4.4.3). Only the validation
of the key changes: no chain to walk, just a comparison.

RFC 7250 is explicit that this moves a problem rather than solving it:

> The mechanism defined herein only provides authentication when an out-of-band
> mechanism is also used to bind the public key to the entity presenting the key.

Hold onto that sentence. It's the whole of §5.4.

## 5.3 How iroh does it

iroh switched its TLS authentication to raw public keys in version 0.34.0 (March
2025, per the repo's `CHANGELOG.md`). At tag v1.3.0 the module says so in its first
lines (`iroh/src/tls.rs:1-6`):

```rust
//! TLS configuration for iroh.
//!
//! Currently there is one mechanism available:
//! - Raw Public Keys, using the TLS extension described in [RFC 7250]
```

The client-side check is short enough to read whole
(`iroh/src/tls/verifier.rs:33-75`, trimmed):

```rust
fn verify_server_cert(&self, end_entity, intermediates, server_name, ..) {
    // 1. Recover the EndpointId we meant to dial from the TLS server name
    let ServerName::DnsName(dns_name) = server_name else { return Err(..) };
    let Some(remote_peer_id) = super::name::decode(dns_name.as_ref()) else {
        return Err(CertificateError::NotValidForName);
    };
    // 2. A raw public key has no chain: any intermediates are an error
    if !intermediates.is_empty() { return Err(CertificateError::UnknownIssuer); }
    // 3. The key presented must be exactly the key we dialled
    let end_entity_as_spki = SubjectPublicKeyInfoDer::from(end_entity.as_ref());
    let remote_public_spki = public_key_to_spki(&ED25519, remote_peer_id.as_bytes());
    if remote_public_spki != end_entity_as_spki {
        return Err(CertificateError::UnknownIssuer);
    }
    Ok(ServerCertVerified::assertion())
}
```

Three things to notice:

- **The expected key travels as the TLS server name.** iroh encodes the
  `EndpointId` as a pseudo-hostname, `<base32 key>.iroh.invalid`
  (`iroh/src/tls/name.rs:21-26`), and the verifier decodes it back. `.invalid` is
  the RFC 2606 TLD that's guaranteed never to resolve. Per the comment at the top
  of `name.rs`, iroh sent this name in SNI up to v1.0.3 and stopped in 1.1.0; it's
  now used only locally, by the verifier and to separate 0-RTT session tickets per
  peer.
- **The proof is TLS 1.3's own.** The signature check is
  `verify_tls13_signature_with_raw_key`; TLS 1.2 is refused outright.
- **The accepting side authenticates but doesn't authorise.** The client
  verifier checks for no intermediates and then, in its own comment, "Beyond
  checking for no intermediates, we don't check the client certificate. The
  actual signatures are already verified - this ensures authentication."
  (`verifier.rs:137-138`). The crate docs spell out the consequence: "When
  accepting connections the peer's EndpointId is authenticated. However it is up
  to the application to decide if a particular peer is allowed to connect or not"
  (`iroh/src/lib.rs:93-94`).

> ⚠️ **Authenticated is not authorised.** Any device on the internet can generate
> a key and connect to your endpoint, and the handshake will succeed: it proves
> the client holds *some* key, and tells you which. If your accept loop doesn't
> check that key against an allow-list, a database, or some capability, you've
> built a server that accepts everyone with perfect cryptographic certainty about
> who they are. And note the error naming in the verifier above: dialling the
> wrong key fails as `UnknownIssuer`, which in web TLS means a missing root. In
> iroh it means "not the peer you asked for". Don't let the log message send you
> looking for a root store.

## 5.4 Where the key comes from

RFC 7250's warning, applied: a key proves "this is the holder of key K". It's
only "this is Alice" if you got K from something that knew it was Alice's. Every
system has that something, and it's where the trust actually lives.

| System | Where you get the peer's key | So you're trusting… |
|---|---|---|
| iroh | Whatever gave you the `EndpointId`: a QR code, a pasted ticket, your own server's database | That channel |
| Tailscale | The coordination server: a node "downloads a list of public keys and addresses in its domain" (Pennarun, "How Tailscale works") | Tailscale's control plane (or your Headscale) |
| WebRTC | A `fingerprint` of the peer's DTLS certificate inside the SDP offer/answer, delivered by the app's signalling server | The signalling server, unless an identity provider is used |

WebRTC's security architecture (RFC 8827) states the WebRTC case plainly. Each
side's signalling message carries "A 'fingerprint' attribute binding the
communication to a key pair", and the resulting assurance is that the parties know
"the signaling server claims that the call is from Alice to Bob. This level of
security is provided merely by having the fingerprint in the message and having
that message received securely from the signaling server." A malicious signalling
server could swap in its own fingerprint and sit in the middle.

> **Teacher's aside.** Public-key addressing is often described as removing the
> need to trust anyone. It doesn't; **it relocates the trust to whoever told you
> the key.** What it removes is the need to trust anything *on the data path*:
> relays, NATs, the Wi-Fi in the café, the DNS server that told you which relay
> to use. Once you hold the right key, nothing between you and the peer can
> impersonate it. But the moment you got the key is now the single point where an
> attacker can win, so design that moment deliberately: scan a QR code in person,
> fetch keys from your own authenticated API, show users a short fingerprint to
> compare. A key pasted from an unauthenticated web form is a certificate signed
> by nobody.

## 5.5 Discovery: from a key to an address

The key tells you who; you still need to know where. A peer's addresses change
constantly (new network, new NAT mapping, new home relay), so something has to
map key → current addresses. iroh calls this **address lookup** (it was called
"discovery" until version 0.96, when the `Discovery` trait was renamed
`AddressLookup`), and offers three mechanisms.

### Signed DNS records (pkarr)

**Pkarr**, "public-key addressable resource records", lets the holder of an
Ed25519 key publish DNS records *signed by that key*. iroh's address lookup page
describes the scheme:

1. Each endpoint builds a small DNS packet with its home relay URL (and optionally
   direct addresses), signs it with its secret key, and HTTP-PUTs it to a pkarr
   server. iroh's default is `iroh-dns-server` run by n0 at `dns.iroh.link`.
2. A peer looking it up does an ordinary DNS query:
   `_iroh.<z32-endpoint-id>.dns.iroh.link TXT`, where the key is z-base-32
   encoded (52 characters).
3. The TXT records come back as `relay=<url>` and `addr=<ip:port> …`.

How much the signature protects you depends on how you look the record up. A
client that fetches the signed packet itself (pkarr over HTTPS, or from the DHT)
verifies the signature and rejects forgeries: `SignedPacket::from_relay_payload`
fails with a `Verify` error (`iroh/src/address_lookup/pkarr.rs:651-652`). A plain
DNS TXT lookup returns only the `key=value` strings, with no signature to check,
so a compromised or spoofed DNS answer could send you to the wrong relay or
addresses. In both cases the damage stops at availability. A wrong answer can make
the connection slow or fail, and it tells the attacker who you're trying to reach,
but it can't make anyone else pass the TLS handshake as the key you dialled. What
leaves
your network is limited too: iroh's security page says the published record
contains "the device's public key and which relay it is using", and that
"Internal IP addresses are not published by default."

Notice how little needs publishing. With a home relay URL, a peer can reach you
through the relay, and direct addresses are exchanged *inside* the encrypted
connection afterwards ([file 06](06-iroh-end-to-end.md)). iroh's mDNS page makes
the point: "Publishing IP addresses to a global address lookup is not required for
iroh's hole punching."

### The Mainline DHT

The same signed pkarr packets can be published to the BitTorrent Mainline DHT
instead of a server, which removes the dependence on n0's DNS server at the cost
of slower, less predictable lookups. Opt-in in iroh.

### Local network: mDNS

On one LAN, peers can find each other with no server at all by multicasting
announcements (UDP 5353). In iroh 1.x this is optional, in two crates with
different trade-offs (`iroh-mdns-address-lookup`, which adapts announcement
frequency, and `iroh-mdns-peer-lookup`, standard query/response), and doesn't work
in browsers. It matters for the case hole punching handles worst: two devices
behind the same NAT that doesn't hairpin ([file 02 §2.4](02-nat-behaviour.md#24-two-more-properties-that-bite)).

| Mechanism | Needs | Finds | Good for |
|---|---|---|---|
| DNS / pkarr | A pkarr-aware DNS server | Home relay (+ optional addresses) | The default; works anywhere with DNS |
| Mainline DHT | DHT participation | Same records | Avoiding a central lookup server |
| mDNS | Same local network | Local addresses | LAN-only, offline, same-NAT peers |
| A ticket | A way to pass a string | Whatever was current when it was made | Bootstrapping, one-off sharing |

## 5.6 Tickets

Sometimes there's no server at all: two people want to connect and can pass a
string (a QR code, a chat message). A **ticket** packs everything needed into
one token. From iroh's tickets page, the format is:

1. an ASCII prefix naming the ticket type (`endpoint`, `blob`, `doc`, …),
2. followed by a lowercase base32 payload of
   [postcard](https://postcard.jamesmunns.com/wire-format.html)-encoded data.

An endpoint ticket encodes an `EndpointAddr`, which is
`{ id: EndpointId, addrs: BTreeSet<TransportAddr> }`: the key, plus relay URLs and
direct `ip:port` addresses. Application tickets add their own data, such as a blob
hash or a document ID with a capability.

| Property | Consequence |
|---|---|
| Contains the key | Whoever made the ticket is your trust root for that peer (§5.4) |
| Contains IP addresses | "Sharing a ticket means sharing your IP address with whoever receives it" (iroh docs) |
| Reusable | "tickets are not single-use tokens": anyone holding one can use it repeatedly |
| Goes stale | Addresses and relays change; the key doesn't |
| May carry secrets | An application ticket granting write access is a credential |

iroh's own advice is to treat tickets as bootstrapping: "If you have *any* means of
coordinating (a database, server, or gossip protocol), we recommend you work with
`EndpointID`s directly instead of tickets." Store the key; look up the addresses
fresh each time.

## Check yourself

1. Why does a key-as-name system need no certificate authority, and what exactly
   does the TLS handshake prove when the peer presents a raw public key?
2. Your iroh endpoint accepts a connection and the handshake succeeds. A
   colleague concludes "it's one of our devices". What has actually been
   established, and what code is missing?
3. Someone gives you an iroh ticket in a public chat channel, claiming it's from
   your colleague. Connecting succeeds and every byte is end-to-end encrypted.
   Explain why that tells you nothing about whether you're talking to your
   colleague, and where in the system the weak point is.
4. The DNS server at `dns.iroh.link` is compromised. What can the attacker do to
   iroh connections that use DNS address lookup, and what can't they do? Why the
   difference?
5. WebRTC and iroh both authenticate peers by key. Compare where each gets the
   peer's key from, and explain what a malicious signalling server can do in
   WebRTC that a malicious relay can't do in iroh.
6. Your app stores tickets in its database so users can reconnect to each other
   later. Months later, many reconnections are slow or fail. Explain why, and what
   you should have stored instead.

(Answers in [09-exercises.md](09-exercises.md).)
