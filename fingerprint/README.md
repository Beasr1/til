# Fingerprint Matching & Template Generation — A Course

A from-scratch course on fingerprint recognition, built around three open reference
implementations that between them cover the whole design space.

**This is reference learning material.** It's about the published research and the
standards, not about any particular system you might be building. Nothing here is
project-specific.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- I use analogies before math, and *why* before *how*.
- Every file ends with **Check yourself** questions. Answers are in `12-exercises.md`.
- There are no stupid questions. This field has 50 years of jargon that everyone pretends
  is obvious and none of it is.

## The three reference implementations

They are three different answers to the same question, which is what makes reading all
three worth far more than reading any one.

| Project | Approach | Link |
|---|---|---|
| **DMD** — Dense Minutia Descriptor | A **local dense** descriptor: one per minutia. Built for latent prints. Tsinghua, IJCB 2024 + journal 2025. | <https://github.com/Yu-Yy/DMD> |
| **FLARE / FDD** | A **fixed-length dense** descriptor for the whole finger, plus learned pose alignment and enhancement. Same lab, next generation. WIFS 2024 / TIFS 2026. | <https://github.com/Yu-Yy/FLARE> |
| **flx** — DeepPrint reimplementation | A **fixed-length flat** embedding: the whole finger as one 512-dim vector. Includes training code and the best benchmark suite of the three. BIOSIG 2023. | <https://github.com/tim-rohwedder/fixed-length-fingerprint-extractors> |

Code references throughout are **relative to each repo's own root** (e.g.
`evaluate_mnt.py:166` means line 166 of that file in the DMD repo), so they work wherever
you have things checked out.

Papers, if you want the primary sources:

- DMD (IJCB 2024) — <https://arxiv.org/abs/2405.01199>
- DMD journal extension (2025) — <https://arxiv.org/abs/2507.15297>
- DeepPrint (2019) — <https://arxiv.org/abs/1909.09901>
- FLARE (TIFS 2026) — <https://ieeexplore.ieee.org/abstract/document/11367048>
- FDD (WIFS 2024) — <https://ieeexplore.ieee.org/abstract/document/10810702>

## Reading order

### Part 1 — Foundations (read these first, in order)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What a fingerprint is](01-what-is-a-fingerprint.md) | Talk about ridges, minutiae, cores, latents, PPI without guessing |
| 2 | [How matching is scored](02-how-matching-is-scored.md) | Read any biometrics results table — FAR, TAR, Rank-1, CMC, DET |
| 3 | [The classical pipeline](03-classical-pipeline.md) | Understand the 40-year-old pipeline everything else reacts against |
| 4 | [Descriptors](04-descriptors.md) | Understand local vs fixed-length, and what "dense" means |

### Part 2 — The three approaches

| # | File | Covers |
|---|------|--------|
| 5 | [DMD — the idea](05-dmd-the-idea.md) | Local dense descriptors, per-minutia patches, masked cosine |
| 6 | [DMD — code walkthrough](06-dmd-code-walkthrough.md) | Every file, both scoring paths, gotchas, how to run it |
| 7 | [FLARE / FDD](07-flare-fdd.md) | Pose estimation (voting + regression), alignment, fixed-length dense matching |
| 8 | [DeepPrint / flx](08-deepprint-flx.md) | Spatial transformers, center loss, minutia maps, open-set benchmarks, training |
| 9 | [**All three, side by side**](09-three-way-comparison.md) | ⭐ The synthesis. Choosing between them; cascaded systems |

### Part 3 — The parts papers skip

| # | File | Covers |
|---|------|--------|
| 10 | [Templates, formats, quality](10-templates-and-quality.md) | ISO 19794-2 / 39794-2, ANSI/NIST-ITL, NFIQ 2, WSQ, template protection |
| 13 | [Touchless & contactless](13-touchless-and-contactless.md) | Why touchless ≠ latent, the normalization front-end, unwarping, CL2CB |

(File 13 is numbered after the reference files because it was added later — renumbering would
break the cross-references in every chapter. Read it as part of Part 3.)

### Reference

| # | File | |
|---|------|--|
| 11 | [Glossary](11-glossary.md) | Look things up |
| 12 | [Exercises & answers](12-exercises.md) | ~90 questions with worked answers, plus things to try |

## If you're short on time

- **30 minutes:** file 01, then file 09.
- **2 hours:** files 01, 02, 04, 09.
- **Focused on extraction & matching mechanics:** files 04 → 05 → 07 → 08 → 09.
- **Focused on building something real:** files 02, 09, 10.
- **Working with phone-camera / touchless capture:** files 01, 09, then 13.

## The one-paragraph summary of everything

A fingerprint is a pattern of skin ridges. The classic way to match two of them is to find
the **minutiae** — points where ridges end or split — and check whether two prints share a
lot of minutiae in the same relative arrangement. That works well on clean prints and
badly on **latents** (crime-scene smudges). Modern methods attach a *learned descriptor*
to the image so even partial, noisy prints can be scored. The three repos here differ in
one decision: **when do you work out which part of print A corresponds to which part of
print B?** DMD defers it to match time and solves an assignment problem per comparison —
robust, slow. FLARE resolves it once at extraction time by aligning every print to a
canonical pose — a good balance. DeepPrint dissolves the question entirely by pooling away
all spatial structure into a single vector — fast, small, but fragile on partial prints.
Cost and robustness trade against each other along exactly that axis, and real systems
cascade a fast method for shortlisting with a slow one for re-ranking.

## How to use me

Ask anything, including:

- "Explain X again but simpler"
- "Walk me through `lsar_score_torchB` line by line"
- "Draw the tensor shapes through FDD as a table"
- "Why does FLARE do it that way and DeepPrint the other way?"
- "Is my mental model right? Here's what I think happens…"
- "Show me a toy numeric example of relaxation labeling"

I'll add files here as we go if a topic earns its own page.
