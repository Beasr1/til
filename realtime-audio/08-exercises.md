# 8. Exercises and answers

Try to answer before reading. Getting one wrong is data, not failure. Answers are worked,
because the reasoning is the part you'll need when the numbers change.

---

## File 01 — Foundations

**1. An `&[f32]` of length 1,024 from a stereo device at 48 kHz.**

Stereo interleaves two samples per I/O frame, so 1,024 samples is 512 frames, and
512 / 48,000 s = **10.67 ms**.

If you treat it as 1,024 frames of mono, three things go wrong at once. You think you have
21.3 ms, so every duration, timestamp and buffer-size calculation is off by two. The
alternating L, R, L, R values are treated as consecutive moments in one signal, so played
back as mono at 48 kHz the audio takes twice as long, an octave lower. And your ring
buffer fills twice as fast as you planned, so it overruns. This is the "samples versus
frames" confusion from §1.4, and it's why every length should be divided by the channel
count before it becomes time.

**2. Downsampling 48 kHz to 16 kHz with `step_by(3)`.**

Keeping every third sample is sampling at 16 kHz with **no anti-aliasing filter**. The new
Nyquist frequency is 8 kHz. Everything between 8 and 24 kHz in the original doesn't
disappear: it folds back below 8 kHz. A component at 10 kHz (the hiss of an "s", say)
lands at 16 − 10 = **6 kHz**, where it's now an inharmonic tone mixed into the speech
band. The output contains energy at frequencies that bear no relation to the original
sound. A real resampler low-pass filters below 8 kHz first, then decimates.

**3. Two `f32` voices peaking at 0.7, summed.**

The sum can peak at 1.4 when the peaks coincide. In `f32` arithmetic that's harmless: 1.4
is a perfectly good float, and gain applied later could bring it back. It becomes a
problem **at the first step that imposes a fixed range**:

- Converting to `i16` (for a codec's integer API, a file, or the network): values beyond
  full scale must be clamped (clipping) or, if you forget the clamp, may wrap.
- Handing it to the output device: the DAC has a fixed range, so the driver or hardware
  clips it.
- Feeding Opus's float API: libopus's header says samples beyond ±1.0 "are supported but
  will be clipped by decoders using the integer API", so the far end may clip it.

That's why `f32` gives you headroom *during* processing, not a licence to exceed full
scale at the output. File 06 §6.2 covers the fix.

**4. Why Opus stops at 20 kHz, and what 20–24 kHz is for.**

RFC 6716 §2: "Opus never codes audio above 20 kHz, as that is the generally accepted upper
limit of human hearing." Bits spent above it buy nothing a listener can hear.

At the ADC, the band between 20 kHz and the 24 kHz Nyquist frequency is the anti-aliasing
filter's **transition band**. A real filter can't go from passing everything to blocking
everything instantly. Xiph's primer explains that at 44.1 or 48 kHz the filter "has to be
extremely sharp to avoid cutting any audible frequencies below 20kHz but still not allow
frequencies above the Nyquist to leak forward." The 4 kHz gap is the room it gets to roll
off.

**5. Halving the codec frame from 20 ms to 10 ms.**

You save 10 ms of waiting to fill a frame. Under G.114's conservative planning formula
(2 × frame + look-ahead, file 05 §5.2) the planning figure drops by 20 ms, but the real
saving is the 10 ms of collection plus whatever processing time actually shrinks.

What it costs (file 03 §3.2): packets per second double from 50 to 100, so header overhead
on IPv4/UDP/RTP goes from 16 to 32 kbit/s, which is comparable to the speech payload
itself. Coding efficiency drops slightly. You keep SILK and FEC (10 ms is still allowed for
both), and each lost packet now loses only 10 ms of audio, which is a small benefit. More
packets also means more per-packet CPU and twice as many chances for scheduling hiccups.

---

## File 02 — Real-time audio threads

**1. 480-frame callbacks with a once-a-minute 15 ms spike.**

480 frames at 48 kHz is a 10 ms period, so every callback must finish within 10 ms. The
normal 0.3 ms is fine. Once a minute the hash-map resize takes 15 ms, overshooting the
deadline by 5 ms: that buffer isn't ready in time, the device underruns, and the listener
hears **a click (or a short gap) about once a minute**. Depending on how the audio system
recovers, the overshoot may cost one buffer or two.

The average doesn't help because the deadline is per buffer, not per minute. Bencina's
point exactly: code optimised "for throughput instead of latency" is judged by its worst
case here. A 0.3 ms average with a 15 ms worst case is a 15 ms algorithm as far as the
audio thread is concerned.

**2. "The critical section is only three lines."**

The length of the critical section isn't what makes the callback wait. The callback waits
for **whoever holds the lock to release it**, and that depends on how that thread is
scheduled. Suppose the network thread takes the mutex, and before it executes the three
lines it's preempted by a medium-priority thread, or it page-faults. The audio thread
wakes, tries the lock, finds it held, and blocks (on Linux, typically a futex wait in the
kernel). It can't run again until the low-priority holder is scheduled, finishes, and
releases, and then the scheduler must wake the audio thread. None of that is bounded. This
is **priority inversion** (cs-fundamentals 04 §4.8), and the three lines are irrelevant to
it.

**3. Why SPSC is safe without a lock, and what two producers break.**

The producer is the only writer of the write index, and the consumer the only writer of
the read index, so neither index ever has two writers racing. The producer stores the
sample into the slot, then publishes the new write index with a **release** store. The
consumer loads the write index with an **acquire** load, which guarantees it also sees the
sample written before the release. No read-modify-write is ever contended, so no lock is
needed.

With two producers, both can load the same write index (say 5), both write their sample
into slot 5, one overwriting the other, and both store 6. One sample is silently lost, and
if their stores interleave differently the index can even skip a slot that was never
written, so the consumer reads garbage. Fixing that needs compare-and-swap loops (a
multi-producer queue), which reintroduces retries: you're back to "how long will this
take?"

**4. 512-frame callbacks feeding a 960-sample encoder.**

Taking one channel as mono, occupancy after each callback, and what the consumer does:

| After callback | Pushed | Occupancy | Consumer |
|---|---|---|---|
| 1 | 512 | 512 | < 960, waits |
| 2 | 512 | 1,024 | Pops 960 → **first packet**; 64 left |
| 3 | 512 | 576 | Waits |
| 4 | 512 | 1,088 | Pops 960 → second packet; 128 left |

The first packet can go out after the second callback, about 21.3 ms after capture began,
plus up to one polling interval if the consumer polls. The leftover keeps growing by 64
each pair of callbacks until a callback fills two frames at once; over time the consumer
sends exactly 50 packets per second of device time, just not evenly spaced relative to
callbacks. That uneven spacing is a small source of send-side jitter, which the receiver's
jitter buffer absorbs.

**5. Fine for 25 minutes, then clicks that get worse; restart fixes it.**

**Clock drift.** A buffer between two clocks (capture ring to encoder, or more likely the
receiver's playout path, where sender and receiver clocks differ) has been slowly filling
or draining since the call started. After 25 minutes it hits a limit. From then on, it
overflows or underflows repeatedly at the boundary, and as the drift continues the events
become more frequent. Restarting resets occupancy to the middle, buying another 25 minutes.
At 100 ppm the drift is 360 ms per hour (file 02 §2.7), so a buffer with ~150 ms of
headroom lasts about 25 minutes.

Fixes, crudest first: drop or insert samples (or packets) when occupancy leaves a band,
which clicks each time; do the same only during silence; time-stretch speech by whole
pitch periods (NetEq's approach); or adaptive resampling steered by a control loop on
buffer occupancy, which is inaudible.

**6. Why not resample by "my clock vs the sender's nominal 48 kHz"?**

Because the receiver can't measure that ratio. It has no reference for "a true second"
other than its own clock, and it doesn't know the sender's actual rate either. Both clocks
claim exactly 48 kHz. The only observable is **relative**: packets (in sender time) arrive
slightly faster or slower than the playback device (in receiver time) consumes them. That
shows up as a slow trend in buffer occupancy, buried under network jitter. So the correct
design is a slow, filtered control loop: watch occupancy (or, as NetEq does, measure
arrivals in playout ticks) and nudge the resampling ratio to hold it at target.

---

## File 03 — Codecs, frames and Opus

**1. Payload fraction at 24 kbit/s over IPv6/UDP/RTP.**

Headers: 40 + 8 + 12 = 60 bytes per packet.

| Frame | Payload bytes (24,000 × t / 8) | On the wire | Payload fraction | Header bitrate |
|---|---|---|---|---|
| 10 ms | 30 | 90 | **33%** | 60 × 8 × 100 = 48 kbit/s |
| 20 ms | 60 | 120 | **50%** | 60 × 8 × 50 = 24 kbit/s |

At 10 ms, two thirds of what you send is headers. Shrinking the frame for latency is a real
option, but on the wire it can cost more than the speech itself. Halve the frame and the
header bitrate doubles while the payload stays the same.

**2. 2.5 ms frames.**

Latency falls because the encoder waits only 2.5 ms to fill a frame, and in
`RESTRICTED_LOWDELAY` the look-ahead is 2.5 ms rather than 6.5 ms. But frames shorter than
10 ms "prevent use of LPC or hybrid encoding modes" (libopus encoder docs), so the encoder
is CELT-only. You lose **in-band FEC** (LBRR frames are SILK; `decide_fec()` returns 0 for
CELT-only) and **DTX** ("only applicable to the LPC layer"). You also pay heavily in
headers: 400 packets per second.

**3. Why the sample-by-sample regression test fails.**

Two independent reasons, neither a bug:

- **Lossy coding doesn't preserve the waveform** (§3.3). SILK resynthesises speech from a
  model; CELT quantises band shapes. The output sounds right and can differ by far more
  than 0.01 per sample. The `VOIP` application even high-pass filters and "emphasizes
  formants and harmonics" on purpose.
- **The output is time-shifted** by the codec's look-ahead: 6.5 ms, 312 samples at 48 kHz
  (`OPUS_GET_LOOKAHEAD`). Even a perfect codec would fail an unaligned comparison.

A better test aligns by the look-ahead, then compares something perceptually meaningful
(per-band energy within a tolerance, or a perceptual quality metric), or simply checks
that decoding round-trips without errors and that levels are sane.

**4. DTX versus a burst of loss.**

Sequence numbers. During DTX, the sender's next packet has sequence number +1 even though
its timestamp has jumped by 400 ms (19,200 at 48 kHz). A loss leaves a gap in sequence
numbers. RFC 7587 §3.1.3 says exactly this, citing RFC 3551 §4.1.

In the moment, though, the receiver can't tell yet: when the gap starts, the next packet
hasn't arrived. It does the same thing either way: it asks the decoder to conceal each
missing frame. After a DTX frame, the decoder's concealment produces comfort noise (RFC
7587: "a frame loss concealment unit in the Opus decoder, which generates a comfort noise
signal"). The distinction matters afterwards: a DTX gap must **not** be counted as loss,
fed into the loss estimate you give the encoder for FEC, or treated as a delay spike by
the jitter buffer.

**5. Clawing back 30 ms inside the codec.**

| Lever | Saves | Costs |
|---|---|---|
| Frame 20 → 10 ms | 10 ms of collection | Double packet rate and header bitrate; slight efficiency loss |
| Frame → 5 or 2.5 ms | 15–17.5 ms | CELT-only: no FEC, no DTX, worse low-bitrate speech, 200–400 pkt/s |
| `RESTRICTED_LOWDELAY` | 4 ms of look-ahead | No SILK/hybrid speech modes, so worse speech at low bitrates and no FEC |
| Disable FEC | Lets the jitter buffer be one frame shallower (file 04 §4.7) | Isolated losses fall back to PLC |
| Lower complexity | Some compute time | Quality at a given bitrate |

Even combining 10 ms frames, low-delay mode and no FEC lands around 24 ms, and gives up
the features that make speech robust. The honest answer: the codec is rarely the right
place to find 30 ms. The jitter buffer and device buffers (file 05 §5.2) usually are.

**6. 44.1 kHz stereo device to `encode_float`.**

In order:

1. **Format conversion**, if the device delivers `i16` or `u16`: scale to `f32` (in the
   callback is fine: a multiply).
2. **Downmix** interleaved stereo to mono: take one channel, or average the two (in the
   callback is fine).
3. **Push** into the SPSC ring buffer.
4. **Resample** 44.1 → 48 kHz on the consumer side, with a proper (synchronous) resampler,
   or an asynchronous one if you also correct drift.
5. **Accumulate** exactly 960-sample frames.
6. **Encode** with `encode_float`.

The one most often done wrong is step 4. Either it's skipped and the encoder is told the
audio is 48 kHz, so everything plays 48,000 / 44,100 ≈ 8.8% fast and about 1.5 semitones
sharp, and the sender produces fewer real seconds of audio than it claims, so the receiver
slowly starves; or it's done by dropping or duplicating samples, which aliases and clicks.

---

## File 04 — Packet loss and jitter

**1. Sequence 100, 101, 103, 102, 102, 105 (104 never arrives).**

| Slot | What the buffer hands the decoder |
|---|---|
| 100 | Packet 100, decoded normally |
| 101 | Packet 101 |
| 102 | Packet 102. It arrived after 103 but before its playout time, so it was inserted in order |
| — | The second 102 was discarded on insert: duplicate (RFC 7587 §4.1 "MUST discard the others") |
| 103 | Packet 103 |
| 104 | Missing. If 105 is already buffered and FEC is in use: decode **105 with `fec = true`** for exactly one frame, recovering 104 from its LBRR data (falls back to PLC if 105 has none). Otherwise: decode an **empty packet** (PLC) for exactly one frame |
| 105 | Packet 105, decoded normally |

The decoder is called exactly once per slot, in order, which keeps its state continuous.

**2. NACK with RTT 25 ms and 180 ms, jitter buffer 60 ms.**

The earliest you can notice packet N is missing is when N+1 arrives, about 20 ms after N
should have. A retransmission then takes a round trip.

- RTT 25 ms: the copy arrives about 20 + 25 = 45 ms after N was due to arrive. With a 60 ms
  target, packets wait up to ~60 ms before playout, so the copy usually beats the deadline,
  barely. Worth implementing, as NetEq does, especially for bursty paths. Network jitter
  eats into that margin, so expect only partial recovery.
- RTT 180 ms: the copy arrives about 200 ms after N was due. It would need a jitter buffer
  of over 200 ms, which blows the latency budget on its own. Useless: rely on FEC and PLC.

**3. Every packet "late" after about 20 minutes; after 2 minutes in another call.**

Plain integer comparison on the 16-bit sequence number. When the counter wraps from
65,535 to 0, every new packet looks older than the last one played, so the jitter buffer
discards it as late, and the call goes silent.

Why 20 minutes: at 50 packets per second, the counter covers its full range in about 21.8
minutes. Why 2 minutes elsewhere: RFC 3550 requires a random starting value, so a call that
started at 59,500 wraps after 6,036 packets, about 2 minutes. With DTX the packet rate drops
in silence and the wrap comes later. The fix is wrapping comparison (`b.wrapping_sub(a) as
i16`), and the test is starting at 65,530 (file 05 Q6).

**4. Target = mean interarrival jitter `J`.**

`J` is a smoothed **mean absolute change** in transit time between consecutive packets. A
jitter buffer has to cover the delay of nearly every packet relative to the fastest, and
network delay is long-tailed: most packets are quick, a few are very late. Setting the
target at a mean means a meaningful fraction of packets, the tail, always arrive after
their playout time, even on a perfectly stable network whose statistics never change. That
steady trickle of late packets is concealment the buffer *caused*. NetEq targets a high
quantile (0.95) of relative delay instead, which is the right shape of answer.

**5. FEC enabled, 5% loss, zero FEC recoveries.**

Sender side:

- Expected loss percentage left at the default of 0: libopus's `decide_fec()` returns 0.
- The encoder is in CELT-only mode: frames under 10 ms, `RESTRICTED_LOWDELAY`, or an
  application or bitrate that makes it choose CELT.
- Bitrate below the per-bandwidth FEC threshold (with loss at or under 5%, the encoder
  won't reduce bandwidth to make room).
- `OPUS_SET_INBAND_FEC` set to 2 with music-like input: FEC "does not necessarily switch to
  SILK if we have music".

Receiver side:

- The receiver never calls the decoder with `fec = true`; it always does PLC.
- The jitter buffer is too shallow: packet N+1 hasn't arrived when N must be played, so
  there's nothing to recover from.
- Losses are bursty: N and N+1 are lost together, so N's redundancy is gone too.
- The counter is wired to the wrong branch (it happens).

**6. Resizing at talkspurt starts, and the two-minute talker.**

In the silence before a talkspurt there's nothing to damage: shortening or lengthening
silence is inaudible. RFC 3551 §4.1 names the start of a talkspurt (flagged by the marker
bit) as the moment that "MAY be used to adjust the playout delay".

If someone talks continuously while the network worsens, there's no silence to use, so
waiting would mean a steadily rising late-packet rate. The buffer must resize **during
speech** by time-stretching: NetEq's preemptive expand plays decoded audio slower (adding
pitch periods) to grow the buffer, and acceleration removes them to shrink it. Done on
periodic, voiced audio, this is usually inaudible; done crudely, it's a jump.

---

## File 05 — Latency, echo, testing

**1. "G.114 says anything over 150 ms is unacceptable."**

The current edition (05/2003, with amendments) doesn't say that. It says two things in §4:
it is "recommended to not exceed a one-way delay of 400 ms for general network planning",
and if mouth-to-ear delay "can be kept below" 150 ms, "most applications, both speech and
non-speech, will experience essentially transparent interactivity". 150 ms is where delay
stops being noticeable, not where it becomes unacceptable. For speech, the current edition
replaces bands with the E-model (G.107), plotted as a smooth curve of rating against delay.
The 0–150 / 150–400 / 400+ bands come from earlier editions, which the 2003 introduction
says caused confusion. Its own Appendix II calls ~225 ms on a long inter-regional call
"still quite satisfactory for the vast majority of users".

**2. 210 ms measured; 120 ms in-app; 25 ms RTT/2. Where are 65 ms?**

Most likely in what your instrumentation can't see:

- **Device and driver buffering** on both ends, below the cpal callback: OS mixers, driver
  buffers, and especially wireless headsets.
- **Asymmetric network paths**: RTT/2 assumes both directions take equal time.
- **Unmeasured gaps** between instrumented stages, such as polling intervals or a send
  queue.

To prove it: run the acoustic loopback on one machine with no network to isolate device
plus pipeline latency; compare cpal's `capture` and `callback` instants on input (and the
output equivalents) to see the driver-side share; swap a Bluetooth headset for a wired
one and re-measure. Each step separates one hypothesis.

**3. A hears their own voice, only when talking to B.**

B's device. A's voice plays from B's speaker into B's microphone and comes back. A hears
it, B doesn't. First recommendation: **B wears headphones**. Second: enable an echo
canceller (platform voice processing or a library AEC) on B's side.

**4. Speaker/mic clock drift and the echo canceller.**

An AEC subtracts a filtered copy of the speaker signal (the **reference**, on the output
device's clock) from the microphone signal (on the input device's clock). Its adaptive
filter learns the echo path as a fixed delay and shape. With 100 ppm of drift, the two
timelines slide apart by about 4.8 samples per second, so the echo moves relative to the
reference and the learned filter is always going stale. The canceller has to keep
re-converging, and residual echo leaks through while it does.

Without an AEC, nothing compares those two streams sample by sample. Capture and playback
each run on their own clock through their own buffers, so the drift is invisible.

**5. `netem loss 5%` showed big FEC gains; production didn't.**

`loss 5%` (or `loss random 5%`) drops packets independently. Opus in-band FEC carries
packet N−1 inside packet N, so it can only recover **isolated** losses. Real networks lose
packets in bursts (a Wi-Fi fade, a queue overflow), and in a burst N and N+1 go together.
The test overestimated FEC because its loss model was kinder than reality. Change the test
to a bursty model (netem's `gemodel`, or a Gilbert-Elliott shim, file 05 §5.5), and if
possible replay loss traces captured from real users.

**6. Catching the plain-`<` sequence comparison bug.**

Start the sender's sequence numbers at 65,530 in a unit or integration test, send twenty
packets through the jitter buffer, include one reordering across the wrap (65,535 arriving
after 0), and assert that nothing is discarded as late and the output order is correct.

It runs in milliseconds, and it's deterministic. A real call hits the wrap after up to
21.8 minutes depending on the random start value, so a half-hour soak test catches it only
if you remember to run one, and a short manual test almost never does.

---

## File 06 — Mixing, PTT and VAD

**1. Three voices peaking at 0.6.**

Clipping is **possible, not guaranteed**. The worst case sum is 1.8, but that needs all
three peaks at the same instant with the same sign. Independent speech mostly adds in
power (RMS grows about √3 ≈ 1.7×) and rarely aligns peaks, but over a long call it will
sometimes happen. You need a limiter or soft clip.

Three synchronised copies of the same recording add sample by sample: every value
triples, so the peak is exactly 1.8, and clipping is **guaranteed** wherever the original
exceeds 1/3.

**2. Dividing by the number of participants.**

In a six-person call where one person is talking, the mix is that person's voice × 1/6,
which is 20·log₁₀(1/6) ≈ −15.6 dB: much quieter than the same person in a two-person call.
Each new joiner makes the current speaker quieter still, and the listener turns the volume
up, which raises noise from everyone else. The gain depends on who's *connected*, not on
the signal. A limiter only acts when the actual sum approaches full scale, so a lone
talker stays at natural loudness.

**3. Why each stream needs its own jitter buffer and decoder.**

Packets are compressed bitstreams; adding bytes of two Opus packets is meaningless. Mixing
needs PCM, so you must decode first. And each stream is independent in every way that
matters to those components: its own sequence and timestamp spaces (and SSRC), its own
loss pattern, its own network delay, its own sender clock drifting at its own rate. The
decoder is stateful too: concealing a loss in stream A needs A's history. One shared
jitter buffer or decoder would mix up all of those.

**4. Push-to-talk pause of 3 seconds.**

- **Sequence number**: last + 1. No packets were sent, so none are missing.
- **Timestamp**: advanced by the full elapsed sampling time, about 3 s × 48,000 = 144,000
  plus the usual 960 for the frame, because RFC 3550 advances the timestamp "regardless of
  whether the block is transmitted … or dropped as silent".
- **Marker bit**: 1, marking the first packet of a talkspurt (RFC 3551 §4.1).

The receiver concludes: no loss (contiguous sequence), 3 seconds of intended silence
(timestamp jump), and a new talkspurt has begun, which is a good moment to re-anchor or
resize the jitter buffer's playout delay.

**5. VAD clipping first syllables of quiet speakers.**

- **Lower the threshold or pick a less aggressive mode.** Cost: more false positives, so
  more background noise transmitted.
- **Pre-roll**: keep the last ~100–200 ms of capture and send it when the VAD opens. Cost:
  a burst of packets at each onset, or permanently higher latency if you play it in real
  time.
- Alternatively, per-user input gain or automatic gain control before the VAD, or offering
  push-to-talk to users the VAD fails.

**6. User C on speakers with a VAD gate and no AEC.**

The other participants' speech plays from C's speakers and reaches C's microphone as
echo. When they speak quietly, that echo is below C's VAD threshold, so the gate stays
shut and nothing is sent. When they speak loudly, the echo crosses the threshold, the VAD
classifies it as speech, opens, and transmits it back, so they hear themselves a round
trip later. The VAD isn't an echo control; it's a level-dependent gate that echo can
trip. Fix: headphones for C, or an echo canceller before the VAD.

---

## Things to try

1. **Hear an underrun.** Write a cpal output callback that plays a sine tone, then add a
   `thread::sleep` that exceeds the buffer period once a second. Listen, then vary the
   buffer size.
2. **Measure drift.** Run capture from one device and playback to another through a ring
   buffer with no correction, log occupancy every second for an hour, and fit a line. The
   slope is your devices' relative clock error.
3. **Hear aliasing.** Downsample a recording with `step_by(3)` and with a proper resampler,
   and listen to both on headphones. Sibilants show it first.
4. **Opus round trip.** Encode a speech file at 6, 12, 24 and 48 kbit/s, decode, and listen.
   Then do it again with 10% of frames replaced by PLC, and again with FEC on.
5. **Jitter-buffer replay.** Record arrival times and sequence numbers from a real call
   over Wi-Fi, then replay them through fixed buffers of 20, 40, 60 and 100 ms and count
   late packets for each. Plot the trade-off curve for your own network.
6. **The clap test.** Measure mouth-to-ear for your app with a phone recording both ends.
   Repeat with a wired headset and with Bluetooth.

---

## Questions worth asking me

Some prompts, if you're not sure where to poke:

- "Walk me through what NetEq does on one `GetAudio` call when the next packet is missing
  but the one after has arrived."
- "Show me a minimal SPSC ring buffer in Rust with the exact atomic orderings, and explain
  each one."
- "How would I design the control loop for adaptive resampling: what's the setpoint, the
  gain, and how do I keep jitter from making it oscillate?"
- "What's the E-model actually computing? Show me R for 100, 200 and 300 ms."
- "How does Opus's SILK layer represent a 20 ms frame of voiced speech, field by field?"
- "Compare RED (RFC 2198) with Opus in-band FEC and with DRED."
- "Design the wire format for my packets if I'm not using RTP: what do I keep, and why?"
- "How do platform voice-processing modes (macOS, Windows, Android) differ from WebRTC's
  AEC3?"
- "What would change if I sent audio over QUIC datagrams rather than UDP?"
- "Is my mental model right? Here's what I think happens to a packet that arrives 70 ms
  late: …"
