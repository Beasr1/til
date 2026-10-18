# 7. Glossary

Reference, not reading. Skim once, then come back when a spec or a crate's docs throws a
term at you. The file and section where each term is taught are in brackets.

## Digital audio

| Term | Meaning |
|---|---|
| **Sample** | One measurement of the signal's amplitude at one instant, for one channel (01 §1.1) |
| **Sample rate** | Samples per second per channel, in Hz. 48 kHz is the norm for Opus (01 §1.1) |
| **PCM** | Pulse code modulation: audio as equally spaced plain-number samples. "Raw audio" (01 §1.1) |
| **ADC / DAC** | Analogue-to-digital / digital-to-analogue converter: the chips at each end of the pipeline (01 §1.1) |
| **Nyquist frequency** | Half the sample rate: the highest frequency a sampled signal can represent (01 §1.2) |
| **Aliasing** | Frequencies above Nyquist folding down into the representable range as distortion (01 §1.2) |
| **Bit depth** | Bits per sample. Each bit adds about 6 dB of dynamic range (01 §1.3) |
| **Dynamic range** | Ratio between the loudest and quietest representable signal, in dB (01 §1.3) |
| **dBFS** | Decibels relative to full scale. 0 dBFS is the largest representable value (01 §1.3) |
| **Clipping** | A sample that would exceed full scale being flattened to it. Sounds harsh (01 §1.3) |
| **Channel** | One independent signal: mono has one, stereo two (01 §1.4) |
| **Interleaving** | Storing multi-channel audio as L R L R … in one buffer (01 §1.4) |
| **I/O frame** | One sample per channel at one instant. cpal's buffer sizes count these (01 §1.4) |
| **Codec frame** | A chunk of audio a codec encodes as a unit, e.g. 20 ms (01 §1.4, 03 §3.2) |
| **Resampling** | Converting audio from one sample rate to another, with filtering (02 §2.7) |
| **Dither** | Deliberate low-level noise added before reducing bit depth, which trades distortion for benign noise (01 §1.3, via Xiph) |

## Real-time I/O

| Term | Meaning |
|---|---|
| **Callback** | Your function, invoked by the audio system each buffer period to deliver or request samples (02 §2.1) |
| **Buffer period** | Buffer size in frames ÷ sample rate: the callback's deadline (02 §2.2) |
| **Underrun** | Playback needed samples and none were ready. Heard as a click or gap (02 §2.3) |
| **Overrun** | Captured samples arrived with nowhere to go and were lost (02 §2.3) |
| **Glitch** | Any audible artefact from a missed deadline (02 §2.3) |
| **Priority inversion** | A high-priority thread waiting on a lock held by a preempted low-priority thread (02 §2.4) |
| **Lock-free** | A data structure whose operations never wait on another thread to make progress (02 §2.5) |
| **SPSC ring buffer** | Single-producer single-consumer circular FIFO, the standard bridge out of an audio callback (02 §2.5) |
| **Try-lock** | Attempt a lock and return immediately if it's taken; acceptable for optional state only (02 §2.4) |
| **Clock drift** | Two devices' sample clocks running at slightly different real rates (02 §2.7) |
| **ppm** | Parts per million: the unit for clock error. 100 ppm at 48 kHz is 4.8 samples/s (02 §2.7) |
| **Synchronous / asynchronous resampling** | Fixed-ratio conversion vs a ratio that can be adjusted at runtime to track drift (02 §2.7) |
| **Host / Device / Stream** | cpal's three core types: platform audio system, an endpoint, an open flow (02 §2.1) |

## Codecs and Opus

| Term | Meaning |
|---|---|
| **Codec** | Coder/decoder: compresses audio for transport and decompresses it (03) |
| **Lossy** | Discards information the listener is unlikely to perceive; output is not a copy of input (03 §3.3) |
| **Look-ahead** | Extra future audio an encoder must see before it can emit a frame. 6.5 ms for libopus VoIP/audio (03 §3.4) |
| **Algorithmic delay** | Frame size plus look-ahead: the delay inherent in the codec's design (03 §3.4) |
| **Linear prediction (LP/LPC)** | Modelling speech as an excitation passed through a vocal-tract filter. Opus's SILK layer (03 §3.3) |
| **MDCT** | Modified discrete cosine transform: the frequency transform under Opus's CELT layer (03 §3.3) |
| **SILK** | Opus's LP layer, for speech up to wideband, 10–60 ms frames (03 §3.4) |
| **CELT** | Opus's MDCT layer, for music and very low delay, 2.5–20 ms frames (03 §3.4) |
| **Hybrid mode** | SILK below 8 kHz plus CELT above, for SWB/FB speech at 10 or 20 ms (03 §3.4) |
| **NB / MB / WB / SWB / FB** | Narrow-, medium-, wide-, super-wide-, full-band: 4, 6, 8, 12, 20 kHz audio bandwidth (03 §3.4) |
| **TOC byte** | First byte of every Opus packet: configuration, stereo flag, frame count (03 §3.4) |
| **VBR / CVBR / CBR** | Variable, constrained variable, constant bitrate (03 §3.5) |
| **DTX** | Discontinuous transmission: one frame per 400 ms during silence (03 §3.6) |
| **Comfort noise (CNG)** | Low-level synthetic noise played during DTX so silence doesn't sound dead (03 §3.6) |
| **ptime** | Packet duration in ms, as negotiated in SDP. Default 20 for Opus (03 §3.2) |
| **Application (Opus)** | Encoder tuning: `VOIP`, `AUDIO`, `RESTRICTED_LOWDELAY` (03 §3.4) |

## Transport, loss and jitter

| Term | Meaning |
|---|---|
| **RTP** | Real-time Transport Protocol, RFC 3550: the reference header for media packets (04 §4.3) |
| **Sequence number** | 16-bit per-packet counter: order, loss and duplicate detection (04 §4.3) |
| **Timestamp (RTP)** | 32-bit sampling instant of the packet's first sample, in media clock units; 48 kHz for Opus (04 §4.3) |
| **SSRC** | 32-bit identifier of a stream's source (04 §4.3) |
| **Marker bit** | For audio, flags the first packet of a talkspurt (04 §4.3, 06 §6.3) |
| **Talkspurt** | A continuous run of transmitted speech between silences (04 §4.3) |
| **Wraparound** | A fixed-width counter overflowing back to zero. Every ~22 min for RTP sequence numbers at 50 pkt/s (04 §4.3) |
| **Jitter** | Variation in packet delay (04 §4.1) |
| **Interarrival jitter (J)** | RFC 3550's smoothed mean of transit-time changes, gain 1/16 (04 §4.3) |
| **Jitter buffer** (de-jitter buffer, playout buffer) | Receiver-side queue that delays packets to a fixed playout schedule (04 §4.4) |
| **Playout time** | When a packet's audio must be played: timestamp plus the buffer's target delay (04 §4.4) |
| **Late packet** | Arrived after its playout time; treated as lost (04 §4.4) |
| **Adaptive jitter buffer** | One whose target delay follows network conditions (04 §4.4) |
| **NetEq** | WebRTC's adaptive jitter buffer and concealment engine (04 §4.5) |
| **Accelerate / preemptive expand** | NetEq's time-compression and time-stretching to shrink or grow the buffer (04 §4.5) |
| **Head-of-line blocking** | One missing packet holding back all later ones, as in TCP (04 §4.2) |
| **NACK** | Negative acknowledgement: the receiver asks for a specific missing packet (04 §4.2) |
| **PLC** | Packet loss concealment: decoder-generated substitute audio for a missing frame (04 §4.6) |
| **FEC** | Forward error correction: redundancy sent in advance (04 §4.7) |
| **LBRR** | Low Bit-Rate Redundancy: Opus's in-band FEC frames, carrying the previous frame at lower quality (04 §4.7) |
| **DRED** | Deep REDundancy: libopus 1.5's optional ML-based long-range redundancy (04 §4.7) |
| **Burst loss / Gilbert-Elliott** | Loss that clusters; the two-state model used to simulate it (05 §5.5) |

## Latency, echo and talk control

| Term | Meaning |
|---|---|
| **Mouth-to-ear delay** | One-way end-to-end delay from speaking to hearing (01 §1.8, 05 §5.1) |
| **Latency budget** | A target total delay, divided between pipeline stages (01 §1.8, 05 §5.2) |
| **G.114** | ITU-T recommendation on one-way transmission time (05 §5.1) |
| **E-model** | ITU-T G.107's transmission rating model, giving R from delay and impairments (05 §5.1) |
| **Acoustic echo** | Far-end audio from a speaker re-captured by the mic and sent back (05 §5.4) |
| **AEC** | Acoustic echo cancellation (05 §5.4) |
| **Double-talk** | Both ends speaking at once; the hard case for AEC (05 §5.4) |
| **Feedback (howl)** | Echo loop with gain above one, growing until it saturates (05 §5.4) |
| **netem** | Linux network emulator qdisc for adding delay, loss, duplication and reordering (05 §5.5) |
| **Gain** | Multiplier on amplitude; in dB, 20·log₁₀(g) (06 §6.1) |
| **Headroom** | Margin between typical levels and full scale (06 §6.2) |
| **Limiter** | Signal-dependent gain reduction that keeps peaks below a ceiling (06 §6.2) |
| **SFU / MCU** | Industry labels for a forwarding server vs a mixing server (06 §6.2) |
| **Sidetone** | Deliberately playing a talker's own voice back to them with minimal delay (06 §6.2) |
| **PTT** | Push-to-talk (06 §6.3) |
| **VAD** | Voice activity detection (06 §6.4) |
| **Hangover** | Continuing to transmit briefly after speech ends, so word endings aren't cut (06 §6.4) |
| **Pre-roll** | Buffered audio from just before a transmit trigger, sent so onsets aren't lost (06 §6.3) |
