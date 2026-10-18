# Real-time Audio — A Course

A from-scratch course on capturing, sending and playing voice live: what audio is in
memory, why the audio thread has rules no other code has, what a codec throws away, how a
receiver turns an unreliable network back into a steady stream, and where the
milliseconds go between a mouth and an ear.

**This is reference learning material.** It's about the standards, the reference
implementations and the general engineering, not about any particular app. Nothing here
is project-specific.

**Assumes:** backend engineering experience; no audio background. For threads, scheduling
and mutexes it builds on [`interview/cs-fundamentals`](../interview/cs-fundamentals/README.md)
files 02 and 04 rather than repeating them. Code examples use Rust crates (cpal, ringbuf,
opus) as concrete reference implementations, but the ideas are language-agnostic. Rust
itself, peer-to-peer connectivity and QUIC are separate courses
([`rust/`](../rust/README.md), [`peer-to-peer/`](../peer-to-peer/README.md), and the QUIC
chapter, [`http/04`](../http/04-quic.md)) and aren't re-taught here.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how, and the problem before the name.
- Every file ends with **Check yourself** questions. Answers are in
  [08-exercises.md](08-exercises.md).
- Numbers come from the spec or the source with a citation. Where something is a rule of
  thumb rather than a standard, the sentence says so.

## The reference implementations and standards

| Source | What it settles |
|---|---|
| [RFC 6716 — Opus](https://www.rfc-editor.org/rfc/rfc6716.html) | SILK/CELT/Hybrid modes, bandwidths, frame sizes, bitrate range, FEC, DTX, PLC, clock drift (§4.4.1) |
| [RFC 7587 — RTP payload for Opus](https://www.rfc-editor.org/rfc/rfc7587.html) | 48 kHz timestamp clock, duplicate handling, FEC and DTX on the wire, `ptime` default |
| [RFC 3550 — RTP](https://www.rfc-editor.org/rfc/rfc3550.html) | Sequence numbers, timestamps, interarrival jitter formula, sequence validation (App. A.1) |
| [RFC 3551 — RTP audio/video profile](https://www.rfc-editor.org/rfc/rfc3551.html) | §4.1: marker bit on talkspurts, adjusting playout delay |
| [ITU-T G.114 — One-way transmission time](https://www.itu.int/rec/T-REC-G.114) | The 150 ms and 400 ms figures, what they actually mean, codec and jitter-buffer delay planning |
| [libopus API docs](https://opus-codec.org/docs/) and [source](https://github.com/xiph/opus) | Legal frame sizes and rates, encoder CTLs, look-ahead, when FEC actually engages |
| [WebRTC NetEq](https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_coding/neteq/g3doc/index.md) | The reference adaptive jitter buffer: operations, delay estimation, drift |
| [Bencina, *Real-time audio programming 101*](http://www.rossbencina.com/code/real-time-audio-programming-101-time-waits-for-nothing) | The rules for audio callbacks, and why |
| [Xiph, *A Digital Media Primer for Geeks*](https://wiki.xiph.org/Videos/A_Digital_Media_Primer_For_Geeks) and [24/192](https://people.xiph.org/~xiphmont/demo/neil-young.html) | Sampling, Nyquist, aliasing, bit depth, sample formats, interleaving |
| [cpal](https://docs.rs/cpal) | Cross-platform audio I/O in Rust: hosts, devices, streams, callbacks |
| [ringbuf](https://docs.rs/ringbuf) | Lock-free SPSC ring buffer in Rust |
| [opus](https://docs.rs/opus) / [audiopus](https://docs.rs/audiopus) | Rust bindings to libopus |
| [rubato](https://docs.rs/rubato) | Synchronous and asynchronous (drift-tracking) resampling in Rust |
| [`tc-netem(8)`](https://man7.org/linux/man-pages/man8/tc-netem.8.html) | Linux network impairment for testing |

## Reading order

### Part 1 — Foundations (read first)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations: what voice looks like to a computer](01-foundations.md) | Picture any audio buffer in memory, name every stage from mic to speaker, and reason about latency as a budget. **Prerequisite for everything else** |

### Part 2 — One machine: the real-time side

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Real-time audio threads](02-realtime-audio-threads.md) | Write a callback that never glitches, bridge it to async code with a ring buffer, and recognise and fix clock drift |
| 3 | [Codecs, frames and Opus](03-codecs-and-opus.md) | Work out bitrates and overhead, choose a frame size, configure Opus, and know what it throws away |

### Part 3 — Two machines: the network side

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Packet loss and jitter](04-packet-loss-and-jitter.md) | Design packet headers and a jitter buffer, and use PLC and Opus FEC correctly |
| 5 | [Latency, echo and testing](05-latency-echo-and-testing.md) | Quote G.114 accurately, account for every millisecond, handle echo, and test under realistic bad networks |

### Part 4 — Stretch

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Mixing, push-to-talk and VAD](06-mixing-and-talk-control.md) | Combine several voices without clipping, and decide when to transmit at all |

### Reference

| # | File | |
|---|------|--|
| 7 | [Glossary](07-glossary.md) | Look things up |
| 8 | [Exercises & answers](08-exercises.md) | Worked answers to every Check yourself, things to try, questions worth asking |

## If you're short on time

- **30 minutes:** file 01, then file 04 §4.4 (the jitter buffer).
- **Before writing the audio-thread code:** files 01 → 02. Read §2.4 twice.
- **Before choosing codec and network settings:** files 01 → 03 → 04.
- **Before a design review or a "why is it laggy / echoey" meeting:** file 01 §1.8, then
  file 05.

## The one-paragraph summary of everything

Audio is a list of numbers sampled at a fixed rate, and the hardware consumes and
produces them on a schedule that waits for nothing, so the code on the audio thread must
never block, lock, allocate or do I/O; it hands samples to and from the rest of the
program through lock-free single-producer single-consumer ring buffers. Raw speech is far
too big to send, so a codec (Opus, at 20 ms frames by default) compresses it about twenty-
fold by modelling what the ear hears rather than preserving the waveform, at the cost of
one frame plus 6.5 ms of look-ahead. The network delivers those packets unevenly, out of
order or not at all, and retransmission usually can't beat the playout deadline, so each
packet carries a sequence number and a timestamp, and the receiver deliberately delays
playback in a jitter buffer just long enough that nearly all packets arrive in time,
treating late ones as lost and filling gaps with concealment or forward error correction.
Every buffer in the chain trades glitches for latency; the total, mouth to ear, should
stay under about 150 ms for conversation to feel transparent (G.114), and you control more
of it than the network does. Two clocks never agree, so every buffer between them needs a
way to absorb drift; and once delay is low, echo from the other person's speaker becomes
the next problem, for which headphones remain the most reliable fix.

## How the scope was set

The brief for this course asked for six topics; a few judgement calls changed its shape.

| Decision | Why |
|---|---|
| **Clock drift is split across two chapters** | Device-to-device drift is a ring-buffer and resampling problem (file 02 §2.7); sender-to-receiver drift is absorbed by the jitter buffer (file 04 §4.5). Same physics, different fixes. Teaching them together would put the jitter-buffer fix before the jitter buffer exists |
| **"Retransmission is useless" is qualified** | It's the right default but not the whole truth: WebRTC's NetEq does NACK. The accurate rule is about round-trip time versus jitter-buffer slack (file 04 §4.2) |
| **G.114 is taught as it reads now, with the popular version shown as superseded** | The widely quoted 0–150 / 150–400 / 400+ bands come from editions before 2003. The current text says something narrower (file 05 §5.1) |
| **Simulation moved into file 05, and the stretch chapter was written** | Testing belongs with measurement. Mixing and talk control (file 06) fit in a short chapter and answer questions a voice app meets quickly |
| **Out of scope** | Signalling and session negotiation (SDP), RTCP reports, SRTP/encryption, NAT traversal, congestion control and bitrate adaptation beyond a mention, video and lip sync. Each is either covered by the networking courses or a course of its own |

## How to use me

Ask anything, including:

- "Explain X again but simpler."
- "Draw the timeline of one packet from mic to speaker with real numbers for my settings."
- "Why does NetEq target a quantile instead of the mean?"
- "Check my callback: here's the code. Does anything in it allocate or lock?"
- "Is my mental model right? Here's what I think happens when…"
- "What would change if I used 10 ms frames instead of 20?"

I'll add files as we go if a topic earns its own page.
