# 6. MiniFASNet and Silent-Face

The open baseline. If someone hands you a small face anti-spoofing model, there's a good
chance it descends from this lineage.

Repo: <https://github.com/minivision-ai/Silent-Face-Anti-Spoofing>

"Silent" means passive — no challenge, no instructions, one frame. File 04's left-hand
column.

## 6.1 Why this one is worth studying

Not because it's the most accurate — it isn't. Because it's:

- **Small.** The facenox MiniFASNetV2-SE variant reports ~1.8M parameters and a ~600 KB
  INT8 export (its `docs/ARCHITECTURE.md`). Comfortably a phone-class model.
- **Complete.** Detection, crop, model, and inference code in one repo, so the whole
  contract is visible rather than implied.
- **Widely forked.** Its conventions — the 2.7 crop especially — propagate into a lot of
  derivative work, including ONNX re-exports. They propagate *imperfectly*, which §6.5 is
  about and which is the most useful thing in this file.

That last point is the practical reason. You will meet these models in the wild with
partial or wrong documentation, and knowing where the landmines are saves you a day.

## 6.2 The architecture family

**MiniFASNet** is a compact CNN in the MobileNet lineage:

- Depthwise-separable convolutions — the standard trick for cutting parameters
- **Squeeze-and-Excitation** blocks in the SE variants — lets the network reweight channels
  based on global context
- A small classifier head

Shapes below are for **one specific derivative** — the MiniFASNetV2-SE trained on
CelebA-Spoof and published by
[facenox/face-antispoof-onnx](https://github.com/facenox/face-antispoof-onnx), per its own
`docs/ARCHITECTURE.md`. It is *not* the upstream minivision checkpoint, and the two differ
in more than weights. Treat this as one worked example of the family, not a spec for it:

| Stage | Output | Blocks |
|---|---|---|
| Conv1 3×3 stride 2, then depthwise Conv2 | 64×64×32 | — |
| Stage 1 | 32×32×64 | 4 residual |
| Stage 2 | 16×16×128 | 6 residual |
| Stage 3 | 8×8×128 | 2 residual |
| Head | `FC 128 → 2` | dropout 0.75 |

Two things stand out.

**Dropout 0.75** is aggressive — and note it differs sharply from the upstream
minivision MiniFASNetV2, which is constructed with `drop_p=0.2`. That is a real divergence
between two models sharing a name, and a heavy dropout rate signals authors fighting
overfitting, which is the central problem in this field (file 08).

**The input is small.** 80×80 or 128×128, versus 224×224 for a typical ImageNet backbone.
That's deliberate: this model is looking at *texture statistics*, not at semantic content.
It doesn't need to know that's a nose.

## 6.3 The multi-scale ensemble

The original repo ships **two models at different crop scales** and combines them.

| Model | Input | Crop expansion |
|---|---|---|
| MiniFASNetV2 | 80×80 | 2.7× |
| MiniFASNetV1SE | 80×80 | 4.0× (some releases 1.0×) |

This is file 05 §5.3 made concrete. The two models see genuinely different evidence — one
gets a face with modest context, the other gets a wide field including bezels and hands.
Summing their outputs is a cheap ensemble over *cue types*, not just over random seeds.

> **Teacher's aside.** This is the part most re-exports drop. People extract a single
> checkpoint, ship it alone, and inherit an accuracy figure that was measured on the pair.
> If you're using one MiniFASNet in isolation, you have something weaker than the number
> in the README, and the README is not lying — you changed the system.

## 6.4 Fourier supervision — the interesting idea

During training, the network has an auxiliary branch that predicts the **FFT magnitude
spectrum** of the input patch, supervised against the actual FFT.

Why this works, connecting to file 03 §3.3: replay attacks carry moiré, prints carry
half-tone dots. Both are **periodic**, and periodic structure is loud in the frequency
domain and subtle in the spatial domain. Asking the network to reconstruct the spectrum
forces its features to retain frequency information a plain binary classifier would happily
discard.

It's the same family of idea as depth supervision in Liu et al. (2018): **a richer training
target than a binary label produces features that generalise better.** A binary label says
*that* these differ; an auxiliary map says *how*.

The branch is training-time only. It does not appear in the exported inference graph — so
if you inspect an ONNX export and find no FFT anywhere, nothing is missing.

## 6.5 The inference contract — and why you must verify it

This is where integrations go wrong, and the honest lesson is not a table of values. **The
contract differs between exports of "the same" model, and published model cards are
sometimes wrong.**

Two documented positions, in genuine conflict:

| Property | Upstream minivision / HF model card | Verified against acceptance fixtures |
|---|---|---|
| Input size | 80×80 | 80×80 |
| Crop expansion | 2.7× | 2.7× |
| Colour order | BGR | BGR |
| Value range | `pixel / 255` → `[0,1]` | **`[0,255]`, no division** |
| Class order | `[live, print, replay]` | `[print, live, replay]` |
| **Genuine index** | **0** | **1** |

The [HuggingFace card for the ONNX re-export](https://huggingface.co/garciafido/minifasnet-v2-anti-spoofing-onnx)
states `pixel / 255` and live at index 0. An integration guide shipped alongside that same
ONNX file states the opposite on both points and warns explicitly that it contradicts the
card.

**The fixtures settle it for that file.** The bundle ships golden 80×80 BGR patches with
published expected logits. For the patch labelled *live*, the logits are
`[-5.32344, 6.84359, -1.52467]` — the maximum is at **index 1**, and the accompanying
metadata records `live_score: 0.99976` and `argmax: 1`. Feeding raw `[0,255]` reproduces
those logits to within `1e-4`; dividing by 255 first collapses every input to
approximately `[0.0003, 0.006, 0.994]` regardless of content.

So for that export, the card is wrong and the fixtures are right. The upstream PyTorch
checkpoint is a *different artefact* and may genuinely use the card's convention — the two
should not be assumed interchangeable just because they share a name.

> **Teacher's aside — the actual lesson.** Do not trust a model card. Do not trust a blog
> post, and do not trust this file either. For any PAD model you integrate, get samples
> with known labels and verify that the model agrees with them before you believe a single
> score. Two things make this non-optional here: a wrong colour order or scaling produces
> **no error**, just quietly worse numbers; and a wrong class index produces confident,
> plausible probabilities that are exactly inverted. Both look like a working integration.
>
> If a model ships acceptance fixtures, they outrank every prose description of it,
> including its own. If it ships none, the first thing to build is a handful.

### The diagnostic you should keep

Two checks, worth writing as tests rather than doing once:

1. **The constant-output check.** Feed several very different images. If the outputs barely
   move, your scaling is wrong. Assert the collapse signature deliberately so nobody
   reintroduces it.
2. **The known-label check.** One confirmed genuine sample and one confirmed attack. If
   the verdicts are inverted, your genuine-class index is wrong — this is the failure that
   never announces itself.

The `2.7` in filenames like `2.7_80x80_MiniFASNetV2` **is** the crop expansion, and that
one is consistent across sources. The filename is documentation; read it.

## 6.6 Reading its numbers honestly

The repo reports strong results on its own test data. File 08 covers why that means less
than it appears, but two specifics for this model:

**It was trained on a particular attack distribution.** Prints and screen replays,
captured on particular devices. Silicone masks were not in scope.

**The published figure is for the ensemble.** See §6.3.

**Its own documentation says the threshold needs calibrating.** Re-exports of this model
carry warnings to that effect, and the default 0.5 is a starting point, not a tuned value.
Anyone who ships the default without measuring on their own traffic is using a number
somebody else chose for a different population.

## 6.7 Where it sits

| | MiniFASNet | CDCN | DeepPixBiS |
|---|---|---|---|
| Size | ~1.8M params (SE variant) | Larger | Medium |
| Supervision | Binary + FFT | Depth map | Per-pixel binary |
| Input | 80×80 | 256×256 | 224×224 |
| Runs on phone | Yes, comfortably | Not really | Marginally |
| Open weights | Yes | Reference impls | Reference impls |

MiniFASNet is the *deployable* one, not the *best* one. That's a legitimate position — a
mediocre model you can actually run at the edge, inside a good capture loop, often beats a
strong model that needs a server round-trip per frame.

---

## Check yourself

1. What does "silent" mean here, and which of file 04's categories does it place this in?
2. Why supervise on an FFT spectrum rather than just the binary label?
3. What is the `2.7` in `2.7_80x80_MiniFASNetV2`, and what breaks if you ignore it?
4. Name the three inference-contract values most likely to be got wrong, and the symptom
   of each.
5. You extract one of the two shipped models and ship it alone. What have you changed
   about the accuracy you were promised?
6. Dropout is 0.75. What does that tell you about what the authors were fighting?

(Answers in [`11-exercises.md`](11-exercises.md).)
