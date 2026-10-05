# 11. Exercises & answers

Answers to every **Check yourself** in the course. Worked, not stated — the reasoning is
the point.

---

## File 01 — What a presentation attack is

**1. Why "presentation attack detection" over "liveness detection"?**

Because most detectors never measure biology. A model that spots moiré has detected a
*screen*, not an *absence of life*. "Liveness" claims more than the evidence supports.

Concrete case where it's the wrong word: a **makeup impersonation attack**. The
presentation genuinely is a living human, so every liveness cue fires correctly and
truthfully — yet it's an attack. "Liveness detection" has no vocabulary for this; "PAD"
does.

**2. Vendor says their product blocks deepfakes. Two questions?**

- *Presentation or injection?* A deepfake on a screen is a replay attack and normal PAD
  cues apply. A deepfake injected past the camera isn't a presentation attack at all, and
  PAD cannot see it.
- *Which PAI species did you test, and what's the per-species APCER?* "Blocks deepfakes" is
  not a measurement.

**3. PAI species, and why averaging misleads.**

A species is a group of attack instruments made the same way. Averaging misleads because
**attackers optimise and averages assume they don't.** If print APCER is 2% and replay is
24%, the average is 13% — but no attacker presents prints. They find the 24% and use it
every time. Report the worst species.

**4. Trained on print and replay. Will it catch a silicone mask?**

No, and say so plainly. A silicone mask has genuine 3D structure, so depth and parallax
cues pass it. Its reflectance can be tuned toward skin. Texture cues are the only ones with
a chance and they weren't trained on this. See file 03 §3.9 — almost the entire cue table
fails on masks. Without evaluation on a mask-containing dataset you have no evidence
either way.

**5. Fusing PAD and match scores into one number — the attack?**

Present a high-quality printed photo of an enrolled user. The *matcher* is very confident
— it's a clear, well-lit image of exactly the right person. If that confidence is averaged
with a mild spoof suspicion, the combined score passes. You've let the attack's own
strength cancel its detection. PAD must be able to veto independently.

---

## File 02 — How PAD is scored

**1. Define APCER and BPCER. Which does a user feel?**

APCER = attacks wrongly accepted ÷ total attacks. BPCER = genuine wrongly rejected ÷ total
genuine. Hook: the first letter is the ground truth of the sample being counted.

The **user** feels BPCER — being blocked. APCER is felt by the business, later, as fraud.

**2. Why is ACER deprecated? When does it mislead?**

Deprecated in ISO/IEC 30107-3:2017 for industry evaluation, though still common in research.
Averaging implies the two errors cost the same; they rarely do. It misleads whenever they're
asymmetric: `APCER 0.1%, BPCER 30%` gives ACER 15%, as does `APCER 30%, BPCER 0.1%`. The
first is unusable, the second is insecure, and they are wildly different systems with an
identical score.

**3. EER 3% — why not 3% in production?**

Because EER picks the threshold *after* seeing the test labels. In deployment you must fix
a threshold in advance, and it won't land on the EER point. It also assumes equal costs,
and real systems deliberately operate away from that.

**4. 30 bona fide, 270 attacks, model rejects everything.**

- Accuracy = 270/300 = **90%**
- APCER = 0/270 = **0%**
- BPCER = 30/30 = **100%**

Perfect security, zero usability, and a 90% headline. Exactly why accuracy is worthless on
imbalanced sets.

**5. "At most 2% of attacks may succeed" — which format?**

`BPCER @ APCER = 2%`. It answers the actual question: at the security level we require,
what fraction of genuine users do we block? Also ask for the per-species breakdown, since
APCER 2% must be the worst species, not the mean.

**6. ACER 5% vs 4% — is the second better?**

Can't tell. The 4% could be `APCER 7% / BPCER 1%` and the 5% could be `APCER 2% / BPCER 8%`
— the "worse" model is over three times more secure. You need both components, the
per-species APCER, and ideally the DET curves, since each single point was chosen by the
authors.

---

## File 03 — The cues

**1. Moiré, in two sentences.**

A screen is a fine grid of pixels and a camera sensor is another fine grid. When you
photograph one grid through the other they interfere, creating a coarse ripple that exists
in neither — a reliable sign you're photographing a screen.

**2. Subsurface scattering — why it matters, what it catches.**

Light enters skin, scatters beneath the surface and exits nearby, giving skin a soft
translucent quality. Paper scatters at the surface and screens emit, so neither reproduces
it. It's the physical basis of texture cues, and it mainly catches **print** attacks —
where a re-captured, surface-scattering medium replaces a translucent one.

**3. Monocular-depth-only model: print / replay / mask.**

- **Print** — caught well. A plane where a face should be.
- **Replay** — caught well. Also a plane.
- **Silicone mask** — missed entirely. Real 3D geometry; the depth estimate is correct and
  says "face".

Depth separates flat from volumetric, and a mask is volumetric.

**4. Why rPPG fails on one image; minimum needed?**

Pulse is a *temporal* oscillation — colour changing over time at heart rate. One frame has
no time axis, so there is nothing to measure. Needs a video of several seconds, reasonably
stable lighting and a fairly still subject.

**5. One cue for a web SDK, RGB, single frame.**

**Texture**, most likely via a small learned classifier — it's the only cue that works on
every attack medium from a single RGB frame with no extra hardware.

Accepting: no defence against masks; strong camera-dependence, so it will need per-device
recalibration; and no motion, parallax or rPPG at all. Widening the crop to pick up context
is a cheap addition worth taking alongside it.

**6. Crop 1.5× → 2.5× improves accuracy. Which cue?**

**Context.** The wider crop now includes what surrounds the face — a phone bezel, a hand,
the cut edge of a print, a background at a different focal plane. None of that is visible
in a tight skin-only crop.

---

## File 04 — Passive, active, and the capture loop

**1. What makes an active challenge worth its friction?**

That the response **could not have been prepared in advance** — an unpredictable challenge,
verified against the specific instruction issued, in a tight window, chosen server-side.
"Blink now" checked only for *a blink* buys nothing; a video of blinking passes.

**2. Light challenge — why "active but invisible", what it exploits?**

Active because the system injects a signal it controls and verifies the response; invisible
because the user isn't asked to do anything, so friction stays near-passive.

It exploits **reflectance**. A real 3D face reflects the flash with predictable falloff
across its geometry; an emissive screen barely responds to it; a flat print reflects
uniformly.

**3. SDK picks the challenge client-side and returns pass/fail — what's wrong?**

The attacker controls both ends. A modified client can select a challenge it has already
prepared for, and can simply report success regardless. The challenge must be issued
server-side and the *response* verified server-side; a client-reported verdict is not
evidence.

**4. Two ways the capture loop improves accuracy without improving the model.**

- It **filters the input distribution** — frames that are too dark, blurry or badly framed
  never reach the model, so the model only ever judges frames it can judge.
- It **guides the user into the training distribution** — prompting for framing and
  lighting produces captures closer to what the model saw in training, which is where its
  accuracy actually holds.

**5. One base64 image — which cues, which category gone?**

Available: texture, frequency artefacts (moiré, half-tone), reflectance, monocular depth,
context.

Gone entirely: **everything multi-frame** — motion, parallax, rPPG, and any
challenge–response. Also all hardware cues, unless the sender provides NIR or depth.

**6. Active liveness to stop injection attacks — right?**

No. Injection replaces the frames before any model sees them. If the system asks for a
blink, the injected stream contains a blink. Active PAD raises the cost of *preparing* the
stream but does not detect injection. That needs attestation, secure capture paths and SDK
integrity — a different discipline.

---

## File 05 — The pipeline

**1. Why detect a face when one is guaranteed present?**

Because the classifier needs **coordinates**, not reassurance. It takes a crop centred on a
box at a specific margin. Guaranteeing presence answers the *presence* question and leaves
the *location* question — the one the model actually depends on.

**2. Model expects 2.7×, you give 1.5×. What's removed?**

All **context**: bezel, hand, paper edge, background focal plane. You've left it only skin
texture. Worse, you've moved the input off the distribution it was trained on, so its
scores no longer mean what its threshold was calibrated for.

**3. Box overflows the frame — why clamp rather than pad black?**

These models were never trained on synthetic black borders, so padding introduces a feature
that carries no meaning to the network and can shift scores unpredictably. Clamping the
expansion to what fits, then translating the box back inside, yields a patch of 100% real
pixels — closer to the training distribution, just at a smaller margin than nominal.

**4. Near-identical logits for every image — first hypothesis?**

**You divided by 255 when the model expects raw `[0,255]`.** The classic symptom: the same
few numbers regardless of input. Check colour order (BGR vs RGB) second.

**5. `[0.44, 0.33, 0.23]` — argmax vs 0.5 threshold, which to ship?**

Argmax says class 0 (real, 0.44 is highest). A `p[real] >= 0.5` threshold says **not real**.

Ship the **threshold**. It's tunable, comparable across models, and lets you set the
security level deliberately. Argmax hardcodes a decision you can't move, and on a
three-class model it accepts cases where the model is genuinely unsure.

**6. Handed an ONNX and told "128×128 RGB". What else must you ask?**

- Crop expansion factor
- Value range, and whether to divide by 255
- Normalisation constants, if any
- Class order and which index is genuine
- Tensor layout (NCHW/NHWC)
- The resize filter used in training

Any one of those wrong gives you a running model returning confident nonsense.

---

## File 06 — MiniFASNet & Silent-Face

**1. "Silent", and which category?**

Passive — no challenge, no instruction, single frame. File 04's left-hand column.

**2. Why supervise on an FFT spectrum?**

Because the attack artefacts that matter — moiré and printer half-toning — are
**periodic**, which is loud in the frequency domain and subtle in the spatial domain.
Forcing the network to reconstruct the spectrum makes it retain frequency information a
plain binary classifier would discard. More generally: a richer target constrains *how* the
network is allowed to be right, which is the main defence against shortcut learning.

**3. What is `2.7` in `2.7_80x80_MiniFASNetV2`?**

The **crop expansion factor**. Ignore it and you feed the model a crop it never saw in
training — the scores stop corresponding to its calibration, usually without any obvious
symptom.

**4. Three contract values most often wrong, and their symptoms.**

- **Value range** — dividing by 255 when the export expects raw `[0,255]`, or the reverse
  → near-constant output for every input
- **Colour order** — RGB where BGR is expected → silently degraded accuracy, no error, and
  the blame lands on the model
- **Genuine class index** → inverted verdicts with entirely plausible-looking probabilities

The sting is that these genuinely **differ between exports of the same model**, and the
published card for at least one widely-used ONNX re-export contradicts the fixtures shipped
beside it (file 06 §6.5). So the answer isn't a set of values to memorise — it's that you
verify against known-label samples before believing any score.

**5. Ship one of two models alone — what changed?**

The published accuracy was measured on the **ensemble of two models at different crop
scales**, which see different cues. Taking one gives you a weaker system than the number
promises. The README isn't wrong; you changed the system it describes.

**6. Dropout 0.75 — what were the authors fighting?**

**Overfitting.** That's an aggressive rate, and it signals a model that memorises its
training distribution easily — consistent with the field-wide generalisation problem in
file 08.

---

## File 07 — Datasets and protocols

**1. Why does >99% intra-dataset tell you little?**

Same cameras, lighting, printers and screens in train and test. The model may have learned
"this dataset's replay device" rather than "a replay". It demonstrates the model can
separate *these* attacks, not attacks in general.

**2. `C→R`, and why a better proxy?**

Train on CASIA-FASD, test on Replay-Attack. Better because a different dataset means
different cameras, subjects and attack instruments — the same distribution shift you get
when you deploy. It measures generalisation rather than memorisation.

**3. OULU-NPU Protocol 4, and why the Protocol 1 gap matters.**

Protocol 4 holds out **illumination/background, attack instruments, and camera**
simultaneously. The gap between Protocol 1 and Protocol 4 measures how much of the model's
performance came from memorising conditions rather than learning the cue — a small gap
means it generalises; a large one means Protocol 1 was flattering it.

**4. Production selfies that all passed your vendor — what can you compute?**

You can compute **BPCER** — the share of genuine users your model would reject. Useful and
often sobering.

You **cannot** compute APCER: there are no attacks in the set. And note the labels are
"the incumbent accepted these", not "these are provably genuine" — any attack that fooled
the incumbent is sitting in there labelled bona fide.

**5. Why split by subject?**

Otherwise the same person appears in train and test, and the model can score well by
recognising *them* rather than by detecting attacks — **subject leakage**. Results inflate
and don't survive new users.

**6. Threat model includes silicone masks — which dataset, and what if you've only used
CelebA-Spoof?**

**WMCA** — ~80 attack instruments across seven categories, including rigid masks, fake
heads and **flexible silicone** masks, plus depth/IR/thermal channels.

The subtlety: CelebA-Spoof does include a **3D paper mask** class, so "I evaluated on
CelebA-Spoof" isn't quite zero mask evidence. But a paper mask is rigid and matte; a
silicone mask has real geometry and tunable reflectance, and they fail different cues.
Paper-mask results tell you almost nothing about silicone. So the honest answer is: you
have evidence about *paper* masks and none about silicone — say which, rather than
collapsing both into "masks".

---

## File 08 — Why it doesn't generalise

**1. Domain shift, and the axis that matters most?**

The training and deployment distributions differ — camera, lighting, attack media,
subjects, framing, compression.

The biggest is usually the **camera**, and it's unintuitive because you think of it as
"just a camera". A phone hands you the output of an ISP doing denoising, sharpening and
tone mapping — operations that alter precisely the micro-texture statistics the model
relies on. Two phones photographing the same print produce different texture signatures.

**2. Shortcut learning, for a non-ML person. Why isn't it cheating?**

If every attack photo in a dataset was taken in the same room, a model can score perfectly
by learning "that room" instead of "that's a photo". It looks brilliant in testing and
fails everywhere else.

It isn't cheating because the model has no concept of intent — it minimises the loss by the
easiest available route. The dataset made the shortcut available; the loss made it optimal.
The fault is in the data and the objective, not the network.

**3. How does depth/FFT supervision help?**

A binary label can be satisfied by *any* correlated feature, including shortcuts. A depth
map or FFT spectrum target can only be satisfied by features that genuinely encode geometry
or frequency — background colour doesn't help you reconstruct a depth map. It constrains the
solution space to the physics you care about.

**4. Why doesn't more data solve it as it does for recognition?**

Two reasons. **Attacks are adversarial and open-ended** — invented by people who read your
papers, so any dataset is a snapshot of yesterday's attacks; there is no "enough". And
**collection bias compounds**: a large set gathered by one team with one protocol is large
*and* correlated, so it doesn't buy diversity proportional to its size.

**5. 26%, 19%, 4% → majority vote gives 13%. How is that worse than the best?**

Because the two weak models **outvote the strong one**. On any capture A and B both
dislike, the majority is already reached against it regardless of C's opinion. Voting
doesn't average quality — it accumulates the union of the weak members' false rejects. An
ensemble only helps when members fail *independently*; here A and B fail on overlapping
sets and drag the result toward themselves.

**6. 1000 genuine captures, no attacks.**

- **Can compute:** BPCER — the false-reject rate.
- **Supports:** tuning to reduce user friction; comparing models on how many real users
  they block; spotting device- or lighting-correlated failures.
- **Must not conclude:** that a low-rejection model is *good*. A model that accepts
  everything scores perfectly. Any decision to drop a model for rejecting more users is a
  security change made on non-security data, and should be stated as such.

---

## File 09 — Deploying it

**1. Why can't you take a threshold from a paper?**

It was calibrated on a different population, different cameras and different attacks. Score
scales aren't comparable across models or even across training runs, and the paper's
operating point reflects its authors' cost assumptions, not yours.

**2. User wrongly blocked — which fields, and why isn't the boolean enough?**

You need the **face box and its source** (did the detector pick a background face, or the
right one?), the **per-model scores** (unanimous or one model dissenting?), the **quality
measurements** (was the frame too dark to judge?) and the **threshold applied** (is this
verdict even reproducible now?).

"false" tells you the outcome and nothing about the cause, so you can't distinguish a
detector error from a quality problem from a genuine model disagreement.

**3. Why distinguish the three, and why detailed feedback on only two?**

They have different causes and need different responses: no-face is a framing/SDK problem,
unjudgeable is a lighting problem, attack is a security decision. Sharing one counter means
you can't tell a detector regression from an attack campaign.

Detailed feedback on the first two helps the user fix something they legitimately control.
Detailed feedback on the third is a **free tuning signal for an attacker** — telling them
*which* cue fired lets them iterate. "Please try again" is the right amount.

**4. High retry-then-succeed rate?**

Your gate is adding friction rather than providing protection. If most rejected users pass
on attempt two or three, you're not blocking attacks — you're annoying genuine users and
then admitting them anyway. Either the threshold is wrong or the capture loop is submitting
frames it should have filtered.

**5. Genuine-only; A rejects 4%, B rejects 19%.**

**Can conclude:** A blocks far fewer genuine users. That's a real usability difference.

**Cannot conclude:** that A is better. B's extra rejections might be it catching attacks A
misses — and with no attacks in the data, nothing distinguishes "B is over-sensitive" from
"B is more secure". Choosing A on this evidence is defensible but is a security decision
made without security data.

**6. Why instrument the score distribution?**

A threshold collapses a continuous signal into a boolean and discards the evidence of
drift. A histogram of raw scores shows a population shifting *before* it crosses the
threshold in enough volume to move the reject rate — it's the early warning, where the
reject rate is the alarm.

**7. 0.173 vs 0.180, same model, two servers.**

Almost certainly **different CPU architectures** (arm64 vs amd64). Floating-point
operations aren't associative, so different instruction orderings give slightly different
results.

Implication: calibrate on the architecture you deploy on, and treat any threshold sitting
within ~0.01 of a cluster of samples as not meaningfully pinned — those samples will flip
between environments.

---

## File 12 — Combining detectors

**1. What does a vote have that a sum doesn't, and vice versa?**

Neither has more information — they *use* different parts of it. The vote uses each
detector's own threshold, which encodes that detector's calibration; the sum discards those
thresholds and uses one. The sum keeps **magnitude**, which the vote throws away: 0.499 and
0.000001 are the same ballot to a vote and very different numbers to a sum.

The practical version: a vote can express "detector A's own notion of borderline". A sum
can express "three mild suspicions add up to one strong one". You can't have both from one
rule.

**2. Why is "unanimous" the least strict setting?**

Because the rule governs *rejection*. Unanimous means every detector must object before
anyone is turned away, so an attack only has to fool **one** detector to be admitted. The
word implies rigour; the mechanics are the opposite. Read any combination rule as "an
attack gets through if …" before trusting your intuition about it.

**3. Attack visible to one of three; what does 2-of-3 do?**

Nothing. It admits the attack, at any threshold. The one detector that saw it is outvoted
by two that had no cue for it.

The tension: file 08 §8.2 says you run several detectors *because they fail on different
things*. That is a statement that complementary detection is expected — and a majority vote
is at its weakest exactly there. The vote is a bet that attacks are visible to most members
at once. Sometimes a good bet, but it should be made explicitly.

**4. One detector stuck returning "genuine". Unanimous rule?**

The rule requires every detector to object. One that never objects makes that condition
unsatisfiable, so the ensemble **stops rejecting anything at all** — not degraded, disabled.
Every health check passes, because the stuck detector is returning 200s promptly.

The signal that catches it is the **reject rate falling**. Most alerting watches for it
rising, which is why this failure can run for a long time.

**5. Four configured, one down: "two-thirds of responders" vs "at least three agree".**

- *Two-thirds of responders* adapts: 3 responding needs 2. The rule's character is
  unchanged.
- *At least three* does not adapt: 3 responding now needs all 3. Losing a detector has
  silently made the rule unanimous.

A fixed count gets proportionally stricter as members drop out. A fraction holds its shape.
Neither is wrong, but only one of them is what most people think they configured.

**6. Three fusion rules all report 0% worst-species APCER.**

You have learned that none of them lets through anything **in this corpus**, and that the
corpus **cannot rank them**. You have not learned that they are equally good, and you
certainly haven't learned any of them is perfect.

What follows: your attack set is exhausted, so any choice between the rules is now being
made on the false-reject side or on grounds the data doesn't speak to. Say which. And treat
each 0% as bounded by sample size — a species with 80 samples and no successes still has a
95% upper bound near 4%.

**7. A rejects 4%, B rejects 6% — why isn't the pair ~10%?**

Because their false rejects are **correlated**. Hard captures — dark, blurred, off-angle,
occluded — are hard for every detector, so the samples A rejects are disproportionately the
ones B rejects too. The union is smaller than the sum.

The error is in the **pessimistic** direction: treating members as independent overestimates
the ensemble's false-reject rate. Which sounds harmless, until you reject a design on a
number that was never real. Compute the union from the joint data, on the same rows.

---

## Questions worth asking me

- "Explain APCER vs BPCER again with a confusion matrix and real numbers."
- "Walk me through what MiniFASNet actually sees at 2.7× versus 4×."
- "Draw the tensor shapes through a MiniFASNet forward pass as a table."
- "Show me a toy numeric example of moiré — two grids and their beat frequency."
- "What would break if I trained on CelebA-Spoof and deployed in a dim bank lobby?"
- "Design an evaluation set for my deployment. What do I collect and how do I split it?"
- "Is my mental model right? Here's what I think happens: …"
- "Given genuine-only data, what's the strongest honest claim I can make?"
- "Sketch a response schema for a PAD API that lets callers apply their own policy."
- "How would I detect, from production telemetry alone, that my model has drifted?"
- "My three detectors disagree on a capture. Walk me through what each rule does with it."
- "Show me a worked example where a majority vote and a score sum disagree."
- "How do I tell whether my detectors' scores are saturated, and what do I do if they are?"
