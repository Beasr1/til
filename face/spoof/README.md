# Face Presentation Attack Detection — A Course

Also called **anti-spoofing**, **liveness detection**, or **PAD**. Those three names mean
roughly the same thing and file 01 explains why the field settled on the ugliest one.

**Assumes [`face/01`](../01-estimating-error-rates.md)** — confidence intervals, the rule of
three, and what a measured error rate does and doesn't entitle you to claim. File 02 below
defines the PAD metrics; `face/01` is the statistics underneath them. Read it first if you
haven't. [`face/02`](../02-retries-and-attempt-caps.md) matters too if your capture flow lets
users try again, which it almost certainly does.

**This is reference learning material.** It's about the published research and the ISO
standards, not about any particular system. Code references point at open reference
implementations and are relative to each repo's own root.

I wrote this as a teacher, not as a peer:

- I explain things you might already know. Skim if so.
- Analogies before math, *why* before *how*.
- Every file ends with **Check yourself**. Answers are in [`11-exercises.md`](11-exercises.md).
- This field's metrics are a minefield of near-synonyms. File 02 exists entirely to defuse them.

## The reference implementations

Two, rather than the fingerprint course's three, because the open-source landscape here is
thinner than it is for fingerprints — and that thinness is itself a fact worth knowing.

| Project | Approach | Link |
|---|---|---|
| **Silent-Face-Anti-Spoofing** (MiniFASNet) | Patch-based CNN with Fourier-spectrum auxiliary supervision. Small enough for phones. The de facto open baseline. | <https://github.com/minivision-ai/Silent-Face-Anti-Spoofing> |
| **face-antispoof-onnx** | MiniFASNetV2-SE trained on CelebA-Spoof, exported to ONNX with architecture and limitations documented. | <https://github.com/facenox/face-antispoof-onnx> |

Papers worth reading as primary sources:

- **ISO/IEC 30107-3** — the testing and reporting standard. Where APCER/BPCER come from.
- Liu, Jourabloo, Liu (2018), *Learning Deep Models for Face Anti-Spoofing: Binary or
  Auxiliary Supervision* — introduced depth + rPPG supervision. <https://arxiv.org/abs/1803.11097>
- Yu et al. (2020), *Searching Central Difference Convolutional Networks for Face
  Anti-Spoofing* (CDCN) — <https://arxiv.org/abs/2003.04092>
- George & Marcel (2019), *Deep Pixel-wise Binary Supervision* (DeepPixBiS) —
  <https://arxiv.org/abs/1907.04047>
- Zhang et al. (2020), *CelebA-Spoof* — the largest public dataset with rich annotations.
  <https://arxiv.org/abs/2007.12342>

## Reading order

### Part 1 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What a presentation attack is](01-what-a-presentation-attack-is.md) | Use PAI / PAD / bona fide correctly, and name the attack families |
| 2 | [How PAD is scored](02-how-pad-is-scored.md) | ⭐ Read any PAD results table without being misled — APCER, BPCER, ACER, HTER. Assumes [`face/01`](../01-estimating-error-rates.md) |
| 3 | [The cues](03-the-cues.md) | Say *what physical difference* a detector is actually exploiting |

### Part 2 — How systems are built

| # | File | Covers |
|---|------|--------|
| 4 | [Passive, active, and the capture loop](04-passive-active-and-capture.md) | Challenge-response vs single-frame, and what each buys |
| 5 | [The pipeline](05-the-pipeline.md) | detect → crop → classify, and why the crop margin is a model parameter |
| 6 | [MiniFASNet & Silent-Face](06-minifasnet-and-silent-face.md) | The open baseline in detail: patches, Fourier supervision, the scale trick |
| 7 | [Datasets & protocols](07-datasets-and-protocols.md) | CelebA-Spoof, OULU-NPU, Replay-Attack, SiW — and what their protocols test |

### Part 3 — The parts papers skip

| # | File | Covers |
|---|------|--------|
| 8 | [Why it doesn't generalise](08-why-it-does-not-generalise.md) | ⭐ Domain shift, dataset bias, and why 99% intra-dataset means little |
| 9 | [Deploying it](09-deploying-it.md) | Thresholds, false rejects, and measuring on your own traffic |
| 12 | [Combining detectors](12-combining-detectors.md) | ⭐ Score vs decision fusion, where a majority vote fails, abstention |

### Reference

| # | File | |
|---|------|--|
| 10 | [Glossary](10-glossary.md) | Look things up |
| 11 | [Exercises & answers](11-exercises.md) | Questions with worked answers |

## If you're short on time

- **20 minutes:** file 01, then file 02.
- **1 hour:** files 01, 02, 03.
- **Choosing or evaluating a model:** [`face/01`](../01-estimating-error-rates.md), then files 02, 07, 08.
- **Shipping one:** files 02, 05, 09.
- **Running more than one detector:** files 08, 12.
- **Understanding a specific open model:** files 05 → 06.

## The one-paragraph summary of everything

A camera flattens the world to light, and a photograph of a face reflects light too — so
the sensor alone cannot separate them. Presentation attack detection recovers the
difference from **artefacts of the re-capture**: a print has paper texture and no depth, a
screen has moiré and its own pixel grid, a mask has wrong reflectance and no pulse. Early
methods hand-designed those features; modern ones learn them from labelled attacks, which
means they learn *the attacks in their training set* and quite often nothing more. That is
the central problem of the field: intra-dataset accuracy is routinely reported above 99%
and cross-dataset accuracy on the same models collapses to coin-flipping. Anyone
evaluating PAD is really asking one question — **does this generalise to attacks it has
never seen?** — and the honest answer is usually "less than the paper implies."

## How to use me

Ask anything, including:

- "Explain APCER vs BPCER again, but with numbers"
- "Why does the crop margin matter so much?"
- "Walk me through what MiniFASNet actually sees"
- "Is my mental model right? Here's what I think happens…"
- "What would break if I trained on CelebA-Spoof and deployed in a bank lobby?"
