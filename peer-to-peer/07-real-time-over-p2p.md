# 7. Real-time traffic over a peer-to-peer path

## The problem

[http file 04](../http/04-quic.md#datagrams-when-you-dont-want-reliability) covers
the choice between QUIC streams and RFC 9221 datagrams on a connection whose path
is a path: UDP from one address to another. A peer-to-peer connection doesn't
have one path. It starts on a relay that runs over TCP, moves to direct UDP, may
add IPv6, and falls back to the relay when the direct path dies
([file 06 §6.6](06-iroh-end-to-end.md#66-the-whole-lifecycle)). Each of those
changes what "unreliable but timely" actually delivers. This chapter is only about
what peer-to-peer adds; read the QUIC chapter's section first.

## 7.1 The same datagram on three kinds of path

| Path | What a lost packet becomes | What else changes |
|---|---|---|
| **Direct UDP** | A lost datagram, gone. QUIC doesn't retransmit it (RFC 9221 §5.2) | Nothing: this is the case datagrams were designed for |
| **Relay over TCP** (iroh's relay, DERP) | Not lost on the relay leg: TCP retransmits it, and **every later packet waits behind it**, datagrams and stream data alike | Higher RTT from the detour; rate limiting throttles reads, so excess traffic queues rather than drops ([file 04 §4.7](04-relays.md#47-the-costs-of-relaying)) |
| **Relay over UDP** (TURN on UDP) | A lost datagram, as with direct | Higher RTT; the 36-byte or 4-byte TURN framing overhead ([file 04 §4.2](04-relays.md#42-turn)) |

And moving between them:

| Event | Effect on real-time traffic |
|---|---|
| Relay → direct upgrade | RTT drops in a step, often by tens of milliseconds. A jitter buffer sized for the relay is now too big, adding latency until it adapts |
| Direct → relay fallback | RTT jumps; a burst of loss while the dead path times out (iroh: up to 15 s of silence before a path is abandoned, though traffic moves to another path sooner once the selector notices) |
| New path added | With QUIC multipath each path has its own RTT and congestion state, so a fresh path starts cautious and ramps up |

> **Teacher's aside.** "Datagrams are unreliable" is shorthand for "*QUIC* won't
> retransmit them". It says nothing about the layers underneath. On a relay that
> runs over TCP, the datagram rides inside a TCP stream that retransmits
> everything and delivers it in order, so a single loss on the relay leg delays
> every datagram behind it, which is precisely the head-of-line blocking you chose
> datagrams to avoid. The datagram API still works; its timing promise doesn't.
> On a peer-to-peer connection, "unreliable" is a property of the *current path*,
> and the current path can change while you're mid-call.

## 7.2 Size: the smallest path wins

A QUIC datagram "cannot be fragmented" (RFC 9221 §5): it must fit in one packet
on the path it's sent on. A peer-to-peer connection's packets may take any of
several paths, and you don't control which, so a datagram size that works on the
direct Wi-Fi path may not fit after a switch to a path with a smaller MTU.

The safe floor is QUIC's own: "QUIC MUST NOT be used if the network path cannot
support a maximum datagram size of at least 1200 bytes" (RFC 9000 §14). Every path
QUIC uses can carry a 1200-byte UDP payload. Keep each message comfortably under
that after QUIC's own header and frame overhead, and it will fit on every path the
connection might move to. Anything larger belongs on a stream, or needs your own
fragmentation, which reintroduces the "lose one piece, lose the whole message"
problem datagrams don't have.

## 7.3 What iroh recommends, and why it's contested

iroh's "Using QUIC" guide takes a firm line:

> We often see users reaching for the QUIC datagram extension when implementing
> real-time protocols. Doing this is in most cases misguided. QUIC datagram
> sending still interacts with QUIC's congestion controller and thus are also
> acknowledged.

Its recommendation is many short-lived streams that you stop or reset when their
contents go stale, citing Media over QUIC (MoQ), which uses "one QUIC stream for
each frame" and has the receiver stop streams that are "too old".

The QUIC chapter's own aside ([http file 04](../http/04-quic.md#choosing-between-them))
shows the field hasn't settled this: MoQ's draft supports both, and small,
independent, frequent updates (audio frames, game positions) remain the clearest
datagram case. iroh's warning is best read narrowly. If you expect QUIC datagrams
to behave like raw UDP, sending as fast as you like with nothing pushing back,
you'll be surprised, because they're congestion-controlled. That's true on every
path, and it's the point of RFC 9221, not a flaw.

The peer-to-peer twist strengthens iroh's case a little. On the relay path neither
approach gets true loss-tolerance, and streams at least give you per-frame
cancellation, which still helps when the relay leg is backed up.

## 7.4 Choosing, for a peer-to-peer path

| Traffic | Use | Peer-to-peer consideration |
|---|---|---|
| Control messages, chat, file chunks | Streams | Unaffected by path type; only latency changes |
| Small, independent, frequent real-time updates (voice frames, game state) | Datagrams, each well under ~1200 bytes | Expect them to be delayed, not dropped, while on a TCP relay; drop stale ones yourself on receipt |
| Video frames | One stream per frame (or group), reset when stale | Works on every path type; frames can exceed one packet |
| Anything during the first seconds of a call | Assume the relay | The direct path may not exist yet ([file 06 §6.3](06-iroh-end-to-end.md#63-dialling-a-real-connection-over-the-relay-immediately)) |

> ⚠️ **Timestamp your real-time messages.** On a direct path, stale data mostly
> just disappears. On a TCP relay leg it arrives, late and in order, after a
> stall. A receiver that plays everything it gets will play a burst of stale audio
> after every relay hiccup. Put a send timestamp or sequence number in each message
> and discard what's too old, whichever transport mechanism you chose.

## Check yourself

1. A voice call over iroh sends audio in QUIC datagrams. It sounds fine for most
   users, but users on one corporate network report occasional half-second
   freezes followed by a burst of fast audio. Explain the mechanism.
2. Why is 1200 bytes the right ceiling to design datagram sizes around on a
   peer-to-peer connection, rather than the MTU you measured on your test network?
3. Your app's jitter buffer is tuned during the first ten seconds of each call
   and then fixed. Why is that a bad idea specifically for peer-to-peer
   connections?
4. iroh's docs call datagrams for real-time protocols "in most cases misguided",
   while the QUIC chapter treats small real-time updates as the clearest datagram
   case. Reconcile the two positions: what is each one right about?

(Answers in [09-exercises.md](09-exercises.md).)
