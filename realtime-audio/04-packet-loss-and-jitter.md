# 4. Packet loss and jitter

## The problem

The sender emits a packet every 20 ms like a metronome. The receiver's speaker needs 20 ms
of audio every 20 ms, also like a metronome. The network between them is not a metronome:
packets arrive bunched up, spread out, out of order, twice, or never. If you play each
packet the moment it arrives, the listener hears gaps whenever the network hesitates. This
chapter is about turning an unreliable, uneven stream back into a steady one, and about
what to play when a piece is simply missing.

Assumes [file 01](01-foundations.md) and [file 03](03-codecs-and-opus.md) (codec frames,
DTX). Transport choice itself (UDP, QUIC datagrams, NAT traversal) belongs to the
[`peer-to-peer/`](../peer-to-peer/README.md) course and the QUIC chapter,
[`http/04`](../http/04-quic.md); this chapter only needs "packets, unreliable".

## 4.1 Why packets arrive unevenly

ITU-T G.114 Annex A.3 states it plainly: "Packetized transmission systems exhibit variable
delay (jitter) in packet delivery time. This is caused by the fact that different packets
carrying speech samples of the same telephone conversation may encounter varying queue
lengths, or different routes through the network."

The usual causes, from your machine outward:

| Cause | What happens |
|---|---|
| Sender scheduling | Your encode task wakes a few ms late because the OS ran something else |
| Wi-Fi | The radio retries frames at the link layer, and shares airtime with every other device |
| Router queues | A burst of someone else's traffic (a download, a video) fills a queue your packet has to wait in |
| Route changes | Packets take a different path mid-call with a different base delay |
| Receiver scheduling | Your receive task is late to read the socket, so several packets appear "at once" |

```
sent      │ 1   │ 2   │ 3   │ 4   │ 5   │ 6   │      one every 20 ms
          0    20    40    60    80   100   120 ms

arrived        1      2           4 3   5          6      (3 overtaken by 4; 6 late)
          0    20    40    60    80   100   120   140 ms
               └─ base delay ─┘
```

**Jitter** is variation in delay, not delay itself. A path with a constant 150 ms delay has
zero jitter and is easy to play smoothly. A path with 20 ms average delay that
occasionally spikes to 80 ms is much harder.

## 4.2 Why retransmission (mostly) doesn't help

The backend reflex is "if a packet's lost, resend it": that's what TCP does. For live
audio, it usually doesn't work, and with TCP it makes things worse.

1. **The deadline.** A lost packet is detected, at the earliest, when a later packet
   arrives without it. A retransmission then needs a round trip. If the playout point
   passes before the copy arrives, the copy is useless: that moment has already been
   played as concealment (§4.6).
2. **Head-of-line blocking (TCP).** TCP delivers bytes in order. One lost segment holds
   back *every* later packet until the retransmission arrives, so a single loss becomes a
   stall for the whole stream, followed by a burst. A voice stream over TCP turns each
   loss into a freeze.

> **Teacher's aside.** "Retransmission is useless for real-time audio" is the slogan, and
> it's the right default, but the truth is narrower, and knowing exactly how narrow
> matters. Retransmission is useless **when one round trip takes longer than the slack
> your jitter buffer gives you.** On a short path with a 20 ms round trip and a 60 ms
> jitter buffer, a selective re-request can land in time. WebRTC's NetEq tracks lost
> packets and generates NACK (negative acknowledgement) lists for exactly this reason
> ([NetEq design doc](https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_coding/neteq/g3doc/index.md),
> "Other responsibilities"). What's always wrong is *reliable in-order* delivery, TCP's
> contract, because it lets one late packet delay all the others. So the precise
> statement is: **never block the stream on a retransmission; a late copy may be used if
> it beats the playout deadline.**

## 4.3 Sequence numbers and timestamps

To repair the stream, the receiver needs two facts about each packet: **where it goes in
the order**, and **which instant of audio it holds**. RTP
([RFC 3550](https://www.rfc-editor.org/rfc/rfc3550.html)) is the standard design for
carrying those. Even if you never send RTP, it's the reference design, and copying its
fields saves you from rediscovering its corner cases.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
                       (RFC 3550 §5.1)
```

| Field | Size | What it's for |
|---|---|---|
| **Sequence number** | 16 bits | "increments by one for each RTP data packet sent, and may be used by the receiver to detect packet loss and to restore packet sequence" |
| **Timestamp** | 32 bits | "the sampling instant of the first octet in the RTP data packet", in units of the media clock |
| **SSRC** | 32 bits | Identifies the stream's source, so streams from several senders can be told apart |
| **Marker (M)** | 1 bit | For audio with silence suppression, set on "the first packet of a talkspurt" ([RFC 3551 §4.1](https://www.rfc-editor.org/rfc/rfc3551.html#section-4.1)) |
| **Payload type (PT)** | 7 bits | Which codec; for Opus, "assigned dynamically" (RFC 7587 §4.1) |

### Why you need both numbers

They answer different questions, and DTX (file 03 §3.6) is where the difference shows:

| What the receiver sees | Sequence numbers | Timestamps | Meaning |
|---|---|---|---|
| Normal | +1 | +960 | Next 20 ms frame |
| Loss | +2 (one missing) | +1,920 | A packet vanished: conceal it |
| DTX / silence | +1 | +19,200 (400 ms) | Sender deliberately sent nothing: play comfort noise, not "loss" |
| Duplicate | Same as before | Same | Discard. RFC 7587 §4.1: the receiver "MUST provide at most one of those payloads to the Opus decoder" |
| Reordered | Lower than the highest seen | Lower | Slot it into place if it's not too late |

RFC 3550 is explicit that the timestamp follows the *sampling clock*, not whether anything
was sent: if an application reads 160-sample blocks, "the timestamp would be increased by
160 for each such block, regardless of whether the block is transmitted in a packet or
dropped as silent." For Opus specifically, RFC 7587 §4.1 fixes the timestamp clock at
48 kHz "for all modes of Opus and all sampling rates", counted per channel, so a 20 ms
frame always advances it by 960 even if you encoded at 16 kHz.

Two more RTP details worth copying:

- **Random starting values.** Both start values "SHOULD be random (unpredictable)", which
  makes known-plaintext attacks on encryption harder. It also means your code must handle
  wraparound from the first minute.
- **Wraparound.** A 16-bit sequence number at 50 packets/s wraps every 65,536 / 50 ≈ 1,311
  s, about **22 minutes**. Compare sequence numbers with wrapping arithmetic: treat
  `b.wrapping_sub(a) as i16` as "how far ahead b is". RFC 3550 Appendix A.1 gives the
  reference algorithm, with constants "based on a maximum misordering time of 2 seconds
  at 50 packets/second and a maximum dropout of 1 minute" (`MAX_MISORDER = 100`,
  `MAX_DROPOUT = 3000`).

> ⚠️ **Plain `<` on sequence numbers breaks at minute 22**, or in the first second if the
> random start is near 65,535. Packet 3 looks "older" than packet 65,534, the jitter buffer
> throws away every new packet as late, and the call goes silent. Test with a start value
> of 65,530.

### Measuring jitter the RTP way

RFC 3550 §6.4.1 defines the jitter statistic receivers report. For packets *i* and *j*
with send timestamps *S* and arrival times *R* (both in timestamp units):

```
D(i,j) = (Rj − Ri) − (Sj − Si)          how much more (or less) the gap grew in transit
J(i)   = J(i−1) + (|D(i−1,i)| − J(i−1)) / 16
```

It's an exponentially smoothed mean of the absolute change in transit time, with gain 1/16.
Useful for a dashboard or a log line. It's not, on its own, a good jitter-buffer target,
because a mean says nothing about the occasional spike that causes a late packet (§4.5).

## 4.4 The jitter buffer

The fix for uneven arrival is to **wait on purpose**. The receiver holds each packet until
a fixed time after it was sent, so packets that arrive early wait longer and packets that
arrive late wait less, and they all come out evenly spaced.

The analogy is an airport baggage carousel run on a delay. Bags come off the plane at
irregular intervals; if the belt only starts once enough time has passed for even the slow
bags to arrive, passengers see a steady procession. The cost is that everyone waits for the
slowest bag you planned for.

G.114 Annex A.3 describes it the same way: the buffer "rearranges the timely order of the
packets, and is sized to take into account a certain range of network delay variation,
effectively delaying all packets to correspond to the delay of the packet with the longest
tolerable transit time. If the delivery time of a packet exceeds the length of the receive
buffer, then this packet 'arrives too late' … and will be discarded."

```
playout time of packet  =  its timestamp  +  a fixed offset (the buffer's target delay)

arrival delay:   │■■■■       pkt1 waits a lot
                 │■■■■■■■■   pkt2 waits a little
                 │■■         pkt3 waits a lot
                 │■■■■■■■■■■■■■■  pkt4: arrives after its playout time → LATE, discarded
                 └──────────┬────────
                       target delay
```

So **a late packet is a lost packet**. That's the core trade, and it's a dial:

| Deeper buffer | Shallower buffer |
|---|---|
| Fewer packets arrive too late | More late packets → more concealment |
| More mouth-to-ear delay, on every packet, always | Less delay |
| Tolerates spikes | Snappy conversation, choppy audio when the network wobbles |

G.114 gives the planning rule for what the buffer costs: "it is recommended to assume
that a de-jitter buffer adds one half of its peak delay to the mean network delay", with
the example that a buffer for a 50 ms delay-variation range "will introduce 25 ms
additional delay, on average."

### What the buffer must handle

| Event | Rule |
|---|---|
| In-order arrival | Insert; play at its time |
| **Reordering** | Insert in timestamp order. Harmless if it arrives before its playout time |
| **Duplicate** | Already have this sequence number → drop |
| **Late** | Its playout time has passed → drop and count it. It was already concealed |
| **Gap at playout time** | Nothing for this slot → ask the decoder to conceal (§4.6), or use FEC from the next packet if it's here (§4.7) |
| **Far-future packet** | Sequence number jumps by thousands → a sender restart or garbage; follow RFC 3550 A.1 rather than inserting it |
| Buffer full | Something upstream is badly wrong; NetEq's answer is to "discard all the existing packets (this should be rare)" |

### Fixed versus adaptive

| | Fixed | Adaptive |
|---|---|---|
| Target delay | Chosen once (say 60 ms) | Re-estimated continuously from observed arrivals |
| Good network | Wastes delay | Shrinks toward the minimum that works |
| Bad network | Constant late-packet loss | Grows to cover the spikes |
| Complexity | A sorted queue and a clock | Needs a delay estimator *and* a way to change depth without audible damage |
| Clock drift (file 02 §2.7) | Slowly overflows or starves | Absorbed as part of the adaptation |

The hard part of an adaptive buffer isn't estimating the target. It's **moving to it**. To
shrink by 20 ms you must play 20 ms of audio in less than 20 ms; to grow, you must stretch.
Jumping (dropping or inserting a frame) is audible mid-word. That's why RFC 3551 notes that
the start of a talkspurt "MAY be used to adjust the playout delay": silence is the cheap
place to resize.

## 4.5 NetEq: the reference adaptive jitter buffer

WebRTC's **NetEq** is the most widely deployed open-source adaptive jitter buffer and the
best one to read. From its
[design doc](https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_coding/neteq/g3doc/index.md):
"NetEq is the audio jitter buffer and packet loss concealer … the buffering delay is
continuously optimized based on the network conditions."

Its API has two calls. `InsertPacket` puts a network packet in the buffer and updates the
delay statistics. `GetAudio` "pulls 10 ms of audio from NetEq for playout" and, each time,
chooses one of six operations:

| Operation | What NetEq does |
|---|---|
| **Normal** | Plays audio decoded from a packet |
| **Acceleration** | Plays a decoded packet faster: shrinks the buffer |
| **Preemptive expand** | Plays a decoded packet slower: grows the buffer |
| **Expand** | Packet loss concealment, "generated by NetEq or the decoder" |
| **Merge** | Stitches concealed audio back onto real decoded audio after a loss |
| **Comfort noise** | Fills DTX silences |

Acceleration and preemptive expand are the time-stretching that lets it resize mid-speech
without jumps.

**How the target is chosen.** In the current source (`delay_manager.h` and
`underrun_optimizer.cc` under `modules/audio_coding/neteq/`, read at the time of writing;
these are tunables and change between versions):

| Parameter | Default | Meaning |
|---|---|---|
| Starting target | 80 ms | Before any statistics exist |
| Histogram buckets | 20 ms each, up to 2,000 ms | Arrival delay is binned |
| Quantile | 0.95 | Target = the delay that 95% of recent packets beat |
| Forget factor | 0.983 | How fast old observations fade |
| Resample interval | 500 ms | The histogram is fed the *worst* delay in each 500 ms window |

The delay being measured is **relative arrival delay**: how late a packet is compared with
the best-case packet in recent history, computed in the playout's own 10 ms ticks. The
design doc notes that measuring in `GetAudio` ticks means "clock drift between the sender
and receiver can be accounted for." So NetEq doesn't treat drift as a separate problem: a
fast sender just looks like packets arriving early, and acceleration shrinks the buffer
back.

The design lesson to take, whether or not you use NetEq: **target a high quantile of
recent delay, not the mean**, let old history fade, and change depth by stretching rather
than jumping.

## 4.6 Packet loss concealment (PLC)

When a packet's slot comes up empty, you still have to hand the speaker 20 ms of
something. Silence makes a hard-edged gap. **Packet loss concealment** invents a plausible
continuation instead.

RFC 6716 §4.4 makes it "an optional decoder-side feature that SHOULD be included when
receiving from an unreliable channel", and describes the reference implementation: "In
CELT mode, the PLC finds a periodicity in the decoded signal and repeats the windowed
waveform using the pitch offset … In SILK mode, the PLC uses LPC extrapolation from the
previous frame." In other words, it keeps the last pitch and vocal-tract shape going and
lets them fade.

How you call it, per the [libopus decoder API](https://opus-codec.org/docs/opus_api-1.5/group__opus__decoder.html):

- Pass a null/empty packet ("Use a NULL pointer to indicate packet loss"). In the Rust
  `opus` crate, an empty slice `&[]`.
- Ask for **exactly** the missing duration: "In the case of PLC (data==NULL) or FEC
  (decode_fec=1), then frame_size needs to be exactly the duration of audio that is
  missing, otherwise the decoder will not be in the optimal state to decode the next
  incoming packet. For the PLC and FEC cases, frame_size must be a multiple of 2.5 ms."
- Do it **in order**. The decoder is stateful; concealing a gap keeps its internal state
  continuous for the next real packet.

> ⚠️ **Skipping the decoder call for a lost packet is a bug even if you play silence.** The
> next real packet is then decoded against state from two frames ago, which produces a
> glitch on recovery as well as the gap. Always call the decoder once per missing frame.

PLC works well for single lost frames and degrades over long gaps, because it extrapolates
from older and older audio. Extended concealment sounds buzzy or robotic, which is why
implementations fade it out over a burst. How long before it should fade to silence is an
implementation choice, not a standard. libopus 1.5 added an optional neural PLC ("Deep
PLC"), enabled at compile time and selected with decoder complexity 5 or more
([Opus 1.5 release notes](https://opus-codec.org/demo/opus-1.5/)), which aims to conceal
longer gaps more naturally.

## 4.7 Opus in-band FEC

Concealment guesses. **Forward error correction (FEC)** sends real redundancy in advance so
the receiver doesn't have to guess.

Opus's in-band FEC (RFC 6716 §2.1.7): "Packets that are determined to contain perceptually
important speech information, such as onsets or transients, are encoded again at a lower
bitrate and this re-encoded information is added to a subsequent packet." In the bitstream
these are **LBRR** (Low Bit-Rate Redundancy) frames, "reduced-bitrate encodings of previous
SILK frames" (§4.2.2). RFC 7587 §3.3 puts it in packet terms: FEC "adds redundant
information about the previous packet (N−1) to the current output packet N."

```
packet:      N−1                  N                    N+1
           ┌──────────┐        ┌──────────┬───────┐  ┌──────────┬───────┐
           │ frame N−1│        │ frame N  │ lbrr  │  │ frame N+1│ lbrr  │
           │          │        │          │ (N−1) │  │          │  (N)  │
           └──────────┘        └──────────┴───────┘  └──────────┴───────┘
                                   LOST ✗               arrives ✓
                                                         → decode its LBRR to recover N,
                                                           then decode it normally for N+1
```

**Sender side**, and the conditions are stricter than the API names suggest. In libopus's
`decide_fec()` ([`src/opus_encoder.c`](https://github.com/xiph/opus/blob/main/src/opus_encoder.c)),
FEC is off if in-band FEC isn't enabled, **or the expected packet-loss percentage is 0, or
the mode is CELT-only**. It's also off if the bitrate is below a per-bandwidth threshold,
and when loss is above 5% the encoder will lower the audio bandwidth to make room for it.

| To get FEC you need | Why |
|---|---|
| `set_inband_fec(true)` | Off by default (`OPUS_SET_INBAND_FEC`: "0: Disable inband FEC (default)") |
| `set_packet_loss_perc(n)` with n > 0 | Default is 0, and `decide_fec()` returns 0 when it's 0 |
| SILK or Hybrid mode | LBRR is a SILK feature. Frames under 10 ms force CELT-only (file 03 §3.4) |
| Enough bitrate | Below the threshold, the encoder declines |

Feeding the encoder a realistic loss estimate (from receiver reports, if your protocol
sends them) is how you get FEC to switch on when it's needed.

**Receiver side**: when packet N is missing at playout and N+1 is already in the jitter
buffer, call the decoder on packet N+1 with `fec = true` and exactly one frame's duration;
this recovers N from the LBRR data ("If no such data is available, the frame is decoded as
if it were lost", so it degrades to PLC safely). Then decode N+1 normally. RFC 7587 §3.3:
"In order to use the FEC data, the jitter buffer needs to provide access to payloads with
the FEC data."

That last sentence has a latency consequence: **FEC only helps if N+1 has arrived by the
time N must be played**, so the jitter buffer must be at least one frame deeper than it
otherwise would be. FEC also only covers *isolated* losses; lose N and N+1 together and
N's redundancy went down with N+1.

**DRED.** libopus 1.5 added Deep REDundancy, which can carry "up to 1 second" of
low-quality redundancy in each packet for "12-32 kb/s overhead", to survive burst loss. At
the 1.5 release it was described as not yet standardised and carrying an experimental
version number, so check its status before relying on interoperability
([Opus 1.5 release notes](https://opus-codec.org/demo/opus-1.5/)).

---

## Check yourself

1. Packets arrive with sequence numbers 100, 101, 103, 102, 102, 105, and the jitter buffer
   plays one per 20 ms tick. Assume 102 arrived before its playout time and 104 never came.
   For each slot from 100 to 105, what does the buffer hand to the decoder, and how?
2. Your round trip is 25 ms and your jitter buffer target is 60 ms. Is a NACK-based
   retransmission worth implementing? Now the round trip is 180 ms. Explain both answers
   with the deadline arithmetic.
3. A call has been running for about 20 minutes when every incoming packet starts being
   discarded as "late". What's the bug, why did it take 20 minutes to appear, and why does
   the same bug show up after 2 minutes in a different call?
4. You set the jitter buffer's target to the mean of RFC 3550's interarrival jitter `J`.
   Why will this produce a steady trickle of late packets even on a stable network?
5. FEC is enabled on the sender, the network drops 5% of packets, yet the receiver's
   recovered-by-FEC counter stays at zero. List every reason this could be, on both ends.
6. Explain why an adaptive jitter buffer should prefer to resize at the start of a
   talkspurt, and what it must do instead if someone talks continuously for two minutes
   while the network gets worse.

(Answers in [08-exercises.md](08-exercises.md).)
