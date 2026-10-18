# 3. Codecs, frames and Opus

## The problem

Raw 48 kHz mono speech is 768 kbit/s (file 01 §1.5). That's already a big ask of a
phone's uplink on a weak cell, it's multiplied by every participant you receive, and the
more bytes you send, the more queueing delay and loss you invite (file 04). A **codec**
(coder/decoder) shrinks the stream by an order of magnitude or more. The price is that it
has to see a chunk of audio before it can compress it, which costs latency, and that it
throws information away, which you need to understand before you test it.

Assumes [file 01](01-foundations.md).

## 3.1 Counting bits: payload and overhead

Start with the payload. At Opus's recommended "sweet spots" for 20 ms frames
([RFC 6716 §2.1.1](https://www.rfc-editor.org/rfc/rfc6716.html#section-2.1.1), repeated in
[RFC 7587 §3.1.1](https://www.rfc-editor.org/rfc/rfc7587.html#section-3.1.1)):

| Configuration | Opus sweet spot | Raw PCM equivalent (16-bit mono) | Ratio |
|---|---|---|---|
| NB speech (8 kHz) | 8–12 kbit/s | 128 kbit/s | ~11–16× |
| WB speech (16 kHz) | 16–20 kbit/s | 256 kbit/s | ~13–16× |
| FB speech (48 kHz) | 28–40 kbit/s | 768 kbit/s | ~19–27× |

Now the part people forget: **every packet carries headers**, and voice packets are tiny.
An Opus packet at 32 kbit/s with 20 ms frames is 32,000 × 0.020 / 8 = **80 bytes**. The
headers in front of it:

| Header | Size | Source |
|---|---|---|
| IPv4 | 20 bytes (minimum) | RFC 791 |
| IPv6 | 40 bytes | RFC 8200 |
| UDP | 8 bytes | RFC 768 |
| RTP (if used) | 12 bytes (fixed part) | [RFC 3550 §5.1](https://www.rfc-editor.org/rfc/rfc3550.html#section-5.1) |

IPv4 + UDP + RTP is 40 bytes on every packet, half again the size of the 80-byte payload.
Encryption and any tunnelling add more. That's why frame size matters for bandwidth, not
just latency.

## 3.2 Frames: why a codec needs a chunk

A codec can't compress one sample at a time. Compression works by finding structure (a
pitch that repeats, a spectral shape that holds steady), and structure only exists across
a stretch of time. So the encoder collects a **codec frame**, typically 20 ms, and
compresses it as a unit.

That has a direct latency cost. ITU-T G.114 Annex A.2 lays it out: "The coded speech frame
is not generated until all speech samples in the input block have been collected by the
encoder. Thus, there is a delay of one frame before processing can begin. In addition,
many coders also look into the succeeding frame to improve compression efficiency. The
length of this advance look is known as the **look-ahead** time of the coder."

Frame size is a four-way trade. With 40 bytes of headers per packet:

| Frame | Samples at 48 kHz | Packets/s | Header overhead (40 B/pkt) | Audio lost per lost packet |
|---|---|---|---|---|
| 2.5 ms | 120 | 400 | 128 kbit/s | 2.5 ms |
| 5 ms | 240 | 200 | 64 kbit/s | 5 ms |
| 10 ms | 480 | 100 | 32 kbit/s | 10 ms |
| **20 ms** | **960** | **50** | **16 kbit/s** | **20 ms** |
| 40 ms | 1,920 | 25 | 8 kbit/s | 40 ms |
| 60 ms | 2,880 | 16.7 | 5.3 kbit/s | 60 ms |

RFC 6716 §2.1.4 summarises it: fewer packets per second "reduces the bitrate, since it
reduces the overhead from IP, UDP, and RTP headers. However, it increases latency and
sensitivity to packet losses … Increasing the frame duration also slightly improves coding
efficiency, but the gain becomes small for frame sizes above 20 ms. For this reason, 20 ms
frames are a good choice for most applications." RFC 7587 makes 20 ms the default `ptime`
(packet duration) when none is negotiated.

That 960 is the magic number from file 01: one 20 ms Opus frame at 48 kHz.

## 3.3 What a speech codec throws away

A lossless compressor (zip, FLAC) gives back exactly the bytes it was given, and gets
maybe 2× on audio. Voice codecs get 20× because they are **lossy**: they keep what a
listener perceives and discard the rest. Opus combines two layers that do this in
different ways ([RFC 6716 §1](https://www.rfc-editor.org/rfc/rfc6716.html#section-1)):
"in speech, linear prediction techniques (such as Code-Excited Linear Prediction, or CELP)
code low frequencies more efficiently than transform (e.g., MDCT) domain techniques, while
the situation is reversed for music and higher speech frequencies."

**Linear prediction (the SILK layer)** uses a model of how speech is *made*. A voice is a
buzz or hiss from the throat (the excitation), shaped by the mouth and nose acting as a
filter. Instead of sending the waveform, the encoder sends the filter's shape (a handful
of coefficients, updated every few milliseconds), the pitch of the buzz, and a coarse
description of what's left over. The decoder rebuilds a waveform that sounds the same
without matching it sample for sample. The SILK decoder in RFC 6716 §4.2 "runs a decoded
excitation signal through adaptive long-term and short-term prediction synthesis filters":
long-term is pitch, short-term is the vocal-tract shape.

**Transform coding (the CELT layer)** turns each frame into frequencies and codes them in
bands that "(roughly) follow the Bark scale, i.e., the scale of the ear's critical bands"
(§4.3). Each band's **energy** is coded explicitly, and only then the finer detail of its
shape, so the overall tone colour survives even when the detail is coarse.

What gets discarded, in both:

| Discarded | Why the ear doesn't mind (much) |
|---|---|
| Frequencies above the chosen bandwidth | Speech intelligibility lives low; wideband (to 8 kHz) already sounds natural |
| Exact waveform shape and phase | The ear is largely insensitive to phase within a sound; it hears spectrum and timing |
| Fine detail that louder nearby sounds mask | **Noise shaping** pushes coding error to where the signal hides it (SILK uses "look-ahead for noise shaping estimation", RFC 6716 §2) |
| Precision in quiet or steady stretches | VBR spends fewer bits there (§3.5) |

> **Teacher's aside.** People assume a lossy codec is a slightly blurry copy of the input,
> so a decoded signal should be "close" sample by sample. It isn't, and it was never meant
> to be. A parametric speech codec can output a waveform that looks quite different on a
> plot and sounds identical. Two practical consequences. First, **your tests can't compare
> decoded audio to the input with a tight sample-wise tolerance**: they need either loose,
> energy-level checks or a perceptual metric. Second, the decoded output is also **shifted
> in time** by the codec's delay (§3.4), so even a perfect codec fails a naive diff unless
> you align first. libopus exposes that delay through `OPUS_GET_LOOKAHEAD` precisely so you
> can "skip on from the start of the decoder's output to provide time aligned input and
> output" ([encoder CTLs](https://opus-codec.org/docs/opus_api-1.5/group__opus__encoderctls.html)).

## 3.4 Opus in one page

Opus is specified by [RFC 6716](https://www.rfc-editor.org/rfc/rfc6716.html) (2012). An
unusual feature: "the primary normative part of this specification is provided by the
source code in Appendix A. Only the decoder portion of this software is normative." The
encoder is free to improve, and it has: [libopus](https://opus-codec.org/) is on 1.6 at
the time of writing.

**Scale.** "The Opus codec scales from 6 kbit/s narrowband mono speech to 510 kbit/s
fullband stereo music, with algorithmic delays ranging from 5 ms to 65.2 ms" (§2).

**Bandwidths** (§2, Table 1):

| Name | Audio bandwidth | Effective sample rate |
|---|---|---|
| NB (narrowband) | 4 kHz | 8 kHz |
| MB (medium-band) | 6 kHz | 12 kHz |
| WB (wideband) | 8 kHz | 16 kHz |
| SWB (super-wideband) | 12 kHz | 24 kHz |
| FB (fullband) | 20 kHz | 48 kHz |

**Modes.** Every Opus packet starts with a one-byte table of contents whose top five bits
pick one of 32 configurations (§3.1, Table 2). They group into three modes:

| Mode | Bandwidths | Frame sizes | Used for |
|---|---|---|---|
| SILK-only | NB, MB, WB | 10, 20, 40, 60 ms | "low bitrate connections with an audio bandwidth of WB or less" |
| Hybrid (SILK + CELT) | SWB, FB | 10, 20 ms | "SWB or FB speech at medium bitrates". SILK codes up to 8 kHz, CELT codes above |
| CELT-only | NB, WB, SWB, FB | 2.5, 5, 10, 20 ms | "very low delay speech transmission as well as music" |

You don't pick the mode directly. You set the application type, bitrate and so on, and the
encoder chooses per frame. The decoder handles any mode, any bandwidth, at any of its
supported output rates: "any Opus decoder operating at 8 kHz can decode an FB Opus frame,
and any Opus decoder operating at 48 kHz can decode an NB frame" (§2.1.3).

**Frames and packets.** Frames of 2.5, 5, 10, 20, 40 or 60 ms; several frames can be
combined into one packet of up to 120 ms (§2.1.4). The packet framing "is not
self-delimiting": it relies on the transport (UDP, RTP, a length prefix) to say how long
the packet is (§3).

**Application types** in libopus
([encoder API](https://opus-codec.org/docs/opus_api-1.5/group__opus__encoder.html)):

| Application | libopus description |
|---|---|
| `VOIP` | "best quality at a given bitrate for voice signals. It enhances the input signal by high-pass filtering and emphasizing formants and harmonics" |
| `AUDIO` | "best quality at a given bitrate for most non-voice signals like music" |
| `RESTRICTED_LOWDELAY` | "disables the speech-optimized mode in exchange for slightly reduced delay" |

**Algorithmic delay.** The look-ahead in libopus is computed in `OPUS_GET_LOOKAHEAD`
([`src/opus_encoder.c`](https://github.com/xiph/opus/blob/main/src/opus_encoder.c)) as
`Fs/400` (2.5 ms) plus a further `Fs/250` (4 ms) of delay compensation unless the
application is restricted low-delay. So:

| Application | Look-ahead | With 20 ms frames |
|---|---|---|
| `VOIP` / `AUDIO` | 6.5 ms | 26.5 ms algorithmic delay |
| `RESTRICTED_LOWDELAY` | 2.5 ms | 22.5 ms |

RFC 6716 §2 explains where those numbers come from: SILK "requires an additional 5 ms
look-ahead for noise shaping estimation" plus up to 1.5 ms for resampling, while CELT
needs 2.5 ms for its overlapping windows, which "cannot be reduced in the encoder because
it is needed for the MDCT overlap". The 26.5 ms figure you'll see quoted for Opus is this
default: 20 ms frame plus 6.5 ms look-ahead. It's the encoder's *algorithmic* delay only;
encode/decode compute time and everything else in file 05's budget come on top.

> ⚠️ **Opus has a short list of legal frame sizes and input rates.** The encoder accepts
> input at 8, 12, 16, 24 or 48 kHz only, and frames of exactly 120, 240, 480, 960, 1,920 or
> 2,880 samples per channel at 48 kHz (libopus encoder API). Hand it 1,024 samples, or
> 44.1 kHz audio, and you get an error, not a best effort. Accumulate exact frames via the
> ring buffer (file 02 §2.6) and resample 44.1 kHz devices first (file 02 §2.7). Also note
> "frames shorter than 10 ms prevent use of LPC or hybrid encoding modes": 2.5 and 5 ms
> frames force CELT-only, which also means no in-band FEC (file 04 §4.7).

## 3.5 VBR, constrained VBR and CBR

**Variable bitrate (VBR)** lets the encoder spend more bits on hard frames (onsets,
fricatives) and fewer on easy ones (steady vowels, silence). It's Opus's default and, per
RFC 7587 §3.1.2, "for the majority of voice transmission applications, VBR is the best
choice."

| Mode | libopus setting | Behaviour |
|---|---|---|
| Unconstrained VBR | `VBR=1`, `VBR_CONSTRAINT=0` | Bits follow content freely |
| **Constrained VBR** (default) | `VBR=1`, `VBR_CONSTRAINT=1` | Varies per frame but behaves like a bit reservoir; "creates a maximum of one frame of buffering delay" |
| Hard CBR | `VBR=0` | Every packet the same size |

Why ever use CBR? RFC 6716 §2.1.8 gives two reasons: a transport that only supports fixed
sizes, or encryption of "highly constrained" or "highly sensitive" speech. Packet sizes in
VBR leak information about *what was said*, even when the content is encrypted;
[RFC 6562](https://www.rfc-editor.org/rfc/rfc6562.html) covers when that matters. For a
general voice app, VBR is the normal choice; be aware the leak exists.

## 3.6 DTX: sending nothing during silence

In a conversation each person spends much of the time listening, so a large share of each
direction's audio is silence or background noise. **Discontinuous transmission (DTX)**
exploits that: "When DTX is enabled, only one
frame is encoded every 400 milliseconds" during silence or background noise (RFC 6716
§2.1.9). The receiver fills the gaps with comfort noise from the decoder's concealment
logic.

Points that matter in practice:

| Point | Source |
|---|---|
| DTX "is only applicable to the LPC layer" (SILK) | [libopus encoder CTLs](https://opus-codec.org/docs/opus_api-1.5/group__opus__encoderctls.html) |
| The receiver must tell DTX gaps from loss. It does so with sequence numbers: during DTX, sequence numbers stay contiguous while timestamps jump | RFC 7587 §3.1.3, citing RFC 3551 §4.1 (file 04 §4.3) |
| DTX "will have a slightly lower speech or audio quality than continuous transmission", so continuous is "RECOMMENDED unless constraints on available network bandwidth are severe" | RFC 7587 §3.1.3 |
| DTX changes the jitter buffer's life: the first packet after silence is a natural moment to resize it | RFC 3551 §4.1, file 04 §4.5 |

## 3.7 Opus from Rust

Two bindings to libopus are common: [`opus`](https://docs.rs/opus/latest/opus/) (0.4 at
the time of writing) and [`audiopus`](https://docs.rs/audiopus/latest/audiopus/) (0.2).
Both wrap the C library, so the behaviour, legal sizes and CTLs above apply unchanged. The
`opus` crate's shape:

```rust
use opus::{Application, Channels, Decoder, Encoder};

let mut enc = Encoder::new(48_000, Channels::Mono, Application::Voip)?;
enc.set_bitrate(opus::Bitrate::Bits(32_000))?;
enc.set_inband_fec(true)?;          // file 04 §4.7
enc.set_packet_loss_perc(10)?;      // FEC does nothing at the default of 0 (file 04 §4.7)

let mut packet = [0u8; 4000];       // libopus suggests 4000 bytes as a safe max
let n = enc.encode_float(&pcm_960, &mut packet)?;   // exactly 960 samples, mono, 48 kHz
// send &packet[..n]

let mut dec = Decoder::new(48_000, Channels::Mono)?;
let mut out = [0f32; 5760];         // room for the 120 ms maximum at 48 kHz
let got = dec.decode_float(&packet[..n], &mut out, false)?;  // samples per channel
```

For a lost packet, the `opus` crate's docs say: "To represent packet loss, pass an empty
slice `&[]`." File 04 §4.6 covers how many samples to ask for when you do.

---

## Check yourself

1. At 24 kbit/s with 10 ms frames over IPv6/UDP/RTP, what fraction of the bits on the wire
   are Opus payload? Repeat for 20 ms frames. What does this tell you about choosing a
   small frame for latency?
2. Explain why 2.5 ms Opus frames give lower latency but can't use the SILK layer, and
   name one loss-resilience feature you give up as a result.
3. A teammate writes a regression test: encode a WAV through Opus, decode it, and assert
   every sample is within 0.01 of the original. It fails. Give two independent reasons
   that have nothing to do with a bug.
4. During DTX the sender transmits one frame every 400 ms. The receiver sees a 400 ms gap
   in arrivals. How does it decide whether this is DTX or a burst of loss, and what does it
   play in each case?
5. You've measured that your app's mouth-to-ear delay is 30 ms too high and you want to
   claw it back inside the codec. List every Opus-side lever you have, what each saves, and
   what each costs.
6. Your capture device is 44.1 kHz stereo. List every transformation between the cpal
   callback and `encode_float`, in order, and say which one is most often done wrong.

(Answers in [08-exercises.md](08-exercises.md).)
