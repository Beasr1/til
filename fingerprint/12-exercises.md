# 8. Exercises and answers

Try to answer before reading. Getting one wrong is data, not failure.

---

## File 01 — What a fingerprint is

**1. What is a minutia, and why three numbers not two?**

A point where a ridge ends or splits. Three numbers because position alone is ambiguous —
two minutiae at the same spot with opposite ridge directions are different features.
`θ` also gives every minutia a local coordinate frame, which is what makes rotation-invariant
descriptors possible (file 04).

**2. Which levels does DMD use?**

Level 2 (minutiae) as anchors, and implicitly Level 1–2 texture in the patch appearance
(ridge flow and frequency are visible in a 128×128 patch). It ignores Level 1 *classification*
and Level 3 entirely. That's fine: pattern class is far too coarse to identify anyone, and
Level 3 needs ≥1000 PPI which these datasets don't have.

**3. Why is latent-to-rolled harder? Three reasons.**

Any three of: partial overlap (a latent shows a fraction of the finger); low SNR (faint
ridges, textured backgrounds that mimic ridges); non-linear distortion from smearing;
far fewer reliable minutiae (10–20 vs ~100); unknown and unrecoverable global pose;
overlapping prints from multiple fingers.

**4. 1000 PPI image with `img_ppi: 500`?**

`scale = img_ppi/500 × tar_shape/middle_shape` computes 1.0 instead of 2.0. The crop
samples 128 image pixels around the minutia, but at 1000 PPI those 128 pixels cover half
the physical skin area the network was trained on. Ridges appear twice as wide and you see
half as many of them. Every learned feature is off-distribution and scores collapse.

**5. Core vs delta?**

Core = centre of the innermost curving ridge, the "eye" of a loop or whorl. Delta = a point
where three ridge flows converge in a triangle. Both are singularities of the orientation
field. Loops have one delta, whorls two, plain arches none.

**Bonus (§1.3): what do `1200PPI` / `1106PPI` in the N2N filenames tell you?**

That the source images were captured at various resolutions, so something upstream must
resample them to 500 PPI (or set `img_ppi` correctly per image) before DMD sees them.
Since `img_ppi` is a single global config value, a mixed-resolution dataset must be
normalised during preparation. It's a real gotcha for anyone assembling their own data.

---

## File 02 — Metrics

**1. FAR and FRR without formulas.**

FAR: how often the system lets in someone who shouldn't get in. FRR: how often it turns
away someone who should get in. Tighten security and you annoy legitimate users; relax it
and you let intruders through.

**2. Why flatten the whole score matrix for `roc_curve`?**

Because verification is a *global-threshold* task: one threshold serves all comparisons.
Flattening pools every genuine and impostor pair into one distribution, which is what a
deployed system faces. Per-query ROCs would measure something else (whether scores are
ordered *within* a query) and would hide the failure where two queries produce scores on
incomparable scales. That's exactly the failure score normalisation exists to fix.

**3. Rank-1 = 95%. Good?**

First question: **how big is the gallery?** 95% against 100 prints is unremarkable; 95%
against 100,000 latents-vs-rolled would be extraordinary. Second question: what's the
image type? Rolled-vs-rolled at 95% would be *bad*.

**4. Raise the threshold.**

FAR ↓, FRR ↑, TAR ↓. You move down-and-left along the ROC curve toward the origin.

**5. Why `[:, -k:]` not `[:, :k]`?**

`np.argsort` sorts ascending, so the *highest* scores are at the end. The last `k` columns
are the top-`k` most similar gallery entries.

**6. SD27 pair counts.**

258 × 258 = 66,564 total pairs; 258 genuine (one mate each); 66,306 impostor. The smallest
non-zero FAR measurable is 1/66,306 ≈ 1.5 × 10⁻⁵, so FAR = 0.01% (1e-4) is right at the
edge of resolvable — about 6.6 impostor pairs. Treat TAR@FAR=0.01% on SD27 as noisy.

---

## File 03 — Classical pipeline

**1. Stages, and which DMD replaces.**

Segmentation → orientation field → enhancement → binarisation/thinning → minutiae
extraction → matching. DMD **skips** the middle four (it works on raw patches), **depends
on** minutiae extraction from an external tool, **learns** segmentation as the foreground
mask head, and **rebuilds** matching around a learned descriptor while keeping the classical
consolidation (Hungarian + relaxation).

**2. What is a Gabor filter tuned to?**

A specific spatial frequency and orientation — it's a sinusoid windowed by a Gaussian.
Fingerprint ridges *are* locally parallel stripes of known frequency and orientation, so a
correctly-tuned Gabor passes real ridges and suppresses everything else.

**3. Why orientation mod 180°?**

A ridge has no head or tail — a ridge running "north-east" is identical to one running
"south-west." Only the line's direction matters, not its sign. (Minutia *orientation* θ is
a full 360° quantity, because a ridge *ending* does have a direction. Don't conflate the two.)

**4. Relaxation labeling, using "vote for each other."**

Every candidate minutia pairing starts with a confidence from appearance. Then pairings
vote for each other: a pairing gets a boost from any other pairing whose geometry is
consistent with it (same distance, same relative angles). Correct pairings are all mutually
consistent, so they form a bloc and vote each other up. Wrong pairings are geometrically
random, agree with nobody, and fade.

**5. Why pairs of minutiae for D1/D2/D3?**

Because *relative* quantities are invariant to global rotation and translation. The distance
between two minutiae doesn't change if you rotate the whole print; their absolute positions
do. Using pairwise relations means you never have to estimate the alignment, which is the
thing you can't reliably estimate on a latent.

**6. Why top-k rather than all?**

On a partial latent, most minutiae have no true correspondent — the Hungarian algorithm
still assigns them *something*, and those forced pairings have meaningless low scores.
Averaging them in dilutes the real signal. Setting `n_pair = 50` on a print with 12
minutiae would average 12 real scores with 38 zeros (padding), driving every score toward
zero and destroying the ranking.

**7. What does DMD depend on, and what risk?**

Minutiae from VeriFinger (commercial) or FDD. The risk: reported accuracy is a property of
*DMD + that extractor*, not DMD alone. Swap in a weaker extractor and the numbers drop
without DMD changing at all. It also makes exact reproduction hard for anyone without a
VeriFinger licence — and harder still because the public FLARE repo ships no minutiae
extractor, so the FDD alternative isn't actually obtainable. Substitutes: file 06 §6.12.

**8. Why greedy isn't good enough.**

Because picking a pair doesn't merely gain you its value — it **forecloses a row and a
column**. Greedy evaluates cells in isolation, so it can't see the damage a pick does to the
options that remain.

The worked 3×3 in file 03: greedy takes `(q1,g1) = 0.92`, the single largest cell in the
table, and totals **1.92**. But `g1` was the *only* good option `q2` had, so consuming it
strands `q2` on 0.25. The optimum declines the best-looking cell entirely — `q1→g2` (0.88),
`q2→g1` (0.85), `q3→g3` (0.35) — and totals **2.08**.

**9. The invariant.**

*Subtracting a constant from an entire row (or column) of the cost matrix does not change
which assignment is optimal.*

True because every valid assignment uses **exactly one cell from each row and each column**.
So subtracting `c` from row *i* reduces the total of *every* possible assignment by the same
`c` — all the totals shift together and their relative ordering is untouched. That's what
lets the algorithm reshape the matrix to manufacture zeros without corrupting the answer.

**10. Why pad with 2?**

Because `2` is the **maximum possible cost** in this formulation. `S ∈ [-1,1]` and
`cost = 1 − S`, so real costs live in `[0,2]`. Padding at the ceiling makes dummy pairings
maximally unattractive: the solver will route around them and only assign a real minutia to
padding when it genuinely has nowhere else to go.

`0` would be catastrophic — padding would look like a *perfect* match and the solver would
prefer dummies to real minutiae. `999` would work correctly but is worse practice: it's an
arbitrary magic number rather than a value derived from the score range, and it distorts any
downstream statistic computed over the cost matrix. `2` is the principled choice.

**11. Dropping one-to-one.**

Impostor scores would inflate badly, and the metric would lose its meaning.

Without the constraint, every query minutia independently grabs whichever gallery minutia
looks most similar — and nothing stops fifty of them all choosing the *same* attractive
gallery minutia. Since impostor prints still contain ordinary ridge endings and bifurcations
that resemble each other locally, a non-mated pair could rack up a high total from a handful
of generically-typical gallery minutiae being matched over and over.

The one-to-one rule is what forces the score to reflect a **coherent set of distinct
correspondences** rather than a count of locally similar-looking points. It's the difference
between evidence and a popularity contest.

**12. Why the decaying λ values don't matter.**

Because the update mixes each pair's own λ with an *average* of its neighbours' λ, and every
λ starts below 1. Nothing in the formula can grow; the whole population drifts downward. That
is a property of the arithmetic, not a signal about the match.

Watch the **ratio** — the separation between well-supported and unsupported pairs. In the
worked example λ(p1)/λ(p3) goes 2.51 → 3.17 → 3.84 → 4.47 while every individual value falls.
Separation is the only thing that matters, because a matcher's scores are meaningful solely
in their ordering (file 02 §2.1). It's also why `n_rel = 5` rather than 500: the *ranking*
converges quickly, and further iterations only shrink every number toward zero.

**13. Why the invariants are computed between pairs of pairs.**

Because absolute positions are not comparable between two prints. `q1` is at (100,100) and
its true correspondent `g2` is at (200,150) — the same physical point of skin, different
coordinates, because the finger was placed differently. Comparing those numbers directly is
meaningless until you know the transform, and **not knowing the transform is the entire
problem** (file 09 §9.1).

Relationships *between* two pairs sidestep it. The distance from `q1` to `q2` (40 px) equals
the distance from `g2` to `g1` (40.00 px) regardless of where or how the finger was placed,
because rotation and translation preserve distances. Same for relative orientation and radial
angle. So you get a consistency check that requires no alignment — which is precisely how DMD
avoids ever estimating global pose.

If you used absolute positions, every genuine pair would look inconsistent the moment the two
prints were captured at different rotations, which is always.

**14. Ranking by `λ/S` while summing `λ`.**

The question it asks is: **"which pairs gained the most support from their neighbours?"** —
not "which pairs look best?" Dividing the post-relaxation confidence by the original
descriptor similarity normalises away the head start, leaving how much the geometry
corroborated each pair.

Right for latents because of what's reliable on a latent and what isn't. Appearance is the
unreliable signal — that's the premise the whole architecture rests on (file 05 §5.2).
Geometry is comparatively trustworthy: a set of minutiae either does or doesn't sit in a
consistent rigid arrangement, and smudging doesn't manufacture one. So when the two sources
disagree, this ranking prefers the pair geometry vouches for over the pair that merely
photographed well.

In the worked example it swaps out A and E (high `S`, mediocre corroboration) for B, D and F
(unremarkable `S`, nearly untouched by relaxation). The resulting score is *lower* in
absolute terms, which is fine — impostor pairs, having no consistent geometry anywhere, lose
much more under this ranking than genuine pairs do, so the gap widens even as both numbers
fall.

---

## File 04 — Descriptors

**1. Define descriptor without "vector" or "embedding."**

A compact numeric summary of what something looks like, built so that two summaries of the
same thing come out nearly identical and summaries of different things come out clearly
different.

**2. Local vs fixed-length trade-off.**

Fixed-length is fast (one dot product, indexable) but needs a complete, well-aligned print.
Local is slow (N₁ × N₂ plus consolidation) but survives partial, unaligned, messy prints.
Latents are partial, unaligned and messy — so local.

**3. What does global average pooling destroy?**

Spatial location. After pooling you know *what* textures are present but not *where*.
DMD refuses because location is exactly what lets it apply a per-cell validity mask and
compare only the jointly-visible regions.

**4. Why does a per-cell mask require spatial structure?**

A mask entry has to *refer* to something. If each descriptor dimension corresponds to a
known physical sub-region, you can mark that sub-region invalid. After pooling, every
dimension is a mixture of all regions — the contribution of the invalid area is smeared
irreversibly across all of them, and there's nothing left to mask.

**5. The two branches; why mask from texture?**

Minutiae branch: supervised with an auxiliary minutiae-map head, so it encodes the
arrangement of nearby minutiae (MCC-like information, learned). Texture branch: encodes
ridge flow, frequency, general appearance. They fail in different situations, so
concatenating is robust. The mask comes off the texture branch because "is this region
real skin?" is a question about texture presence, not about minutiae.

**6. Why is MCC rotation-invariant, and what does DMD do?**

MCC builds its cylinder in a local frame whose x-axis is the minutia's own orientation, so
rotating the whole print rotates the frame identically and the descriptor is unchanged.
DMD does the same thing physically: it rotates the *image patch* by θ before the CNN sees
it, so the network always views the neighbourhood from the minutia's own point of view.

**7. What does CosFace solve that softmax doesn't?**

Plain softmax only needs training classes to be *separable* — it will happily leave
same-class embeddings spread out as long as a boundary exists. At test time you compare
identities never seen in training, so you need same-identity embeddings genuinely *tight*
and different-identity embeddings genuinely *far*. CosFace adds an angular margin that
forces exactly that, producing embeddings whose cosine similarity is meaningful for
unseen identities.

---

## File 05 — DMD

**1. Three sentences.** See §5.1. Key beats: (a) rotated patch per minutia → CNN → 3D
feature block + validity mask; (b) mask-weighted cosine over all minutia pairs; (c)
Hungarian assignment + relaxation labeling + top-k mean.

**2. What does the minutia-anchored frame remove, and not remove?**

Removes: global rotation and translation. You never estimate the alignment.
Does **not** remove: scale (fixed externally via PPI), non-linear distortion (mitigated by
using small patches), or the need to figure out *which* minutia corresponds to which —
that's still the assignment problem.

**3. Shape trace.**

`1×128×128` → layer0 (/2) → `64×64×64` → layer1 → `64×64×64` → layer2 (/2) →
`128×32×32` → layer3 (/2) → `256×16×16` → layer4 (/2) → `512×8×8` → `embedding` (1×1 convs)
→ `6×8×8` = 384. Same on the texture side → 384. Concatenated = 768. The 8×8 is
128 ÷ 16, from the four stride-2 stages (layer0, layer2, layer3, layer4).

**4. Masked cosine; role of `m₁ᵢ·m₂ᵢ` in each place.**

```
sim = Σ m₁ᵢm₂ᵢf₁ᵢf₂ᵢ  /  ( √(Σ m₁ᵢm₂ᵢf₁ᵢ²) · √(Σ m₁ᵢm₂ᵢf₂ᵢ²) )
```

Numerator: restricts the dot product to cells valid in **both** patches — invalid cells
contribute nothing rather than contributing noise. Denominator: recomputes both norms over
that same restricted region, so the similarity is a *fair* cosine over the overlap and isn't
artificially depressed just because the overlap is small.

**5. Why score normalisation if the denominator already handles mask size?**

Because those are two different problems. The denominator makes the similarity *fair* for
a small overlap — but "fair" means a 3-cell overlap can legitimately hit 0.95 by chance.
Fairness isn't reliability. Score normalisation multiplies by `√(n₁₂/N_mean)` to encode
*confidence*: a high score backed by a large overlap is worth more than the same score
backed by almost nothing. Without it, tiny-overlap flukes pollute the top of the impostor
distribution and wreck TAR@FAR.

**6. Why sort by `efficiency` rather than `lambda_t`?**

`efficiency = λ_after / λ_before` measures how much *corroboration from surrounding
geometry* a pairing gained. On a latent, appearance similarity is unreliable but geometric
consistency is comparatively trustworthy, so you'd rather keep the pairings that the
geometry endorsed than the ones that merely looked good in isolation. (It then sums the
λ values, not the efficiencies — efficiency is only the ranking key.)

**7. Three things DMD doesn't do.**

Any three of: extract minutiae; enhance images; estimate global pose; classify Level-1
patterns; ship training code; run fast.

---

## File 06 — Code

**1. What's a "sample"?**

A single minutia — not an image. `dump_dataset_mnteval.py` emits one entry per minutia, so
a 500-image dataset with ~60 minutiae each yields ~30,000 samples. Extraction is slow
because the CNN runs once per minutia, not once per image.

**2. Why isn't `forward` called at inference?**

`extract_feat` calls `get_embedding` directly. `get_embedding` omits the `minu_map` head —
an expensive path with two deconvolutions producing a 6×128×128 output — which is only
needed for the training loss. It also omits the `feat_lst`/`minu_lst` splits.

**3. `mask1.repeat(1,1,ndim_feat)`.**

`flatten(1)` on `[C,H,W]` is channel-major, so the 768-vector is
`[ch0's 64 cells, ch1's 64 cells, …, ch11's 64 cells]`. Tiling the 64-value mask 12 times
produces exactly that layout, so element *i* of the mask lines up with element *i* of the
feature. It's 12 rather than 6 at the call site because `calculate_scores` passes
`ndim_feat=self.ndim_feat*2` — the feature is the concatenation of two 6-channel branches.

**4. Binary path derivation.**

`n12` = count of cells valid in both. `bmm(m₁f₁, m₂f₂ᵀ)` = count of jointly-valid cells
where both bits are 1. `bmm(m₁(1−f₁), m₂(1−f₂)ᵀ)` = count where both are 0. Subtracting
both agreement counts from the total leaves the **disagreements**: `d12`. So `d12/n12` is
the fraction disagreeing, in [0,1]; `1 − 2·(that)` maps 0 → +1 and 1 → −1. It's Hamming
similarity rescaled to match the cosine path's range.

**5. Why `NaN` padding, not `0`?**

Because 0 is a legal value for a feature, a mask, or a similarity. NaN is not, so it's an
unambiguous "this slot isn't real." The code then uses `torch.isnan` to count true minutiae
per item (`n1_batch`, `n2_batch`), to substitute prohibitive cost 2 in the assignment
matrix, and to zero out padded scores — all of which would be impossible if padding looked
like data.

**6. Two missing files after `git clone`.**

`fptools/` (clone from <https://github.com/youngjetduan/fptools>) and
`logs/<version>/best_model.pth.tar` (download separately). Both are in `.gitignore`.

**7. Three Mac blockers.**

Hardcoded `torch.device(f"cuda:{...}")` with no CPU/MPS fallback; `torch_linear_assignment`
is a CUDA extension compiled from source; `os.system('rm -rf ...')` and
`os.popen("stty size")` are POSIX/TTY-dependent (the latter also breaks on redirected
output). Plus the Python 3.8 / torch 1.10.1 pins.

**8. Crash without `-e`.**

`is_load=args.extract`, so without `-e` the model and dataset are never constructed and the
script goes straight to `calculate_scores()`. If features aren't already cached on disk,
`MatchDataset` finds empty `search/` and `gallery/` folders and it fails. Other strong
candidates: `prefix` still set to `/path/to/TEST_DATA`, or `genuine_pairs.txt` missing
(note the plural — the README says `genuine_pair.txt`, the code says `genuine_pairs.txt`).

**9. Why match scores are the wrong diagnostic for a swapped extractor.**

Because a wrong angle convention **degrades scores without ever raising an error**, and a
degraded score distribution is indistinguishable from "this extractor is just worse on my
data" or "DMD is mediocre here." The signal is confounded — you can't separate a coordinate
bug from a genuine quality difference by looking at the output of the whole pipeline.

Inspect the **patches** instead, cropped with `fast_tps_distortion`. Correct anchoring means
the ridge flows along **+x in every patch**, because that's the entire purpose of the minutia
frame (file 05 §5.4). This is a direct observation of the thing you're unsure about rather
than an inference from four stages downstream, and it takes about ten minutes. General
lesson: **debug at the stage where the invariant is stated, not where the symptom appears.**

**10. Degrees, counter-clockwise, at θ = 90°.**

Two errors compound. The sign flip means you rotate by −90° instead of +90°, so the patch is
off by 180°. The unit error means the code reads "90" as 90 *radians* ≈ 14.3 full turns
≈ 5.2° after wrapping — so in practice the patch is cropped at an essentially arbitrary
rotation that varies unpredictably with θ.

Nothing warns you because θ is only ever consumed as an argument to trig functions building
a rotation matrix. Every float is a valid angle. There is no range check to fail, no shape
mismatch, no NaN — the pipeline runs to completion and emits a score matrix that is simply
worse than it should be.

**11. Endings fine, bifurcations wrong.**

The two extractors disagree on what a **bifurcation's direction means**. SourceAFIS points an
ending toward the ridge and a bifurcation toward the *split side*; another extractor may use
a different rule (e.g. the bisector of the two branches, or the opposite direction). Since
the ending convention agrees, endings anchor correctly and bifurcations carry a systematic
offset.

Roughly **half** your descriptors are affected — bifurcations and endings occur in broadly
comparable numbers. That's the worst case for detection: enough correct descriptors that the
system still half-works and looks plausible, enough corrupted ones to lose real accuracy.
Hence the advice to inspect patches from the two types *separately*; pooling them hides
exactly this.

---

## File 07 — FLARE / FDD

**1. Why no Hungarian, no relaxation?**

Because alignment already established correspondence. Once both prints are warped into
the same canonical pose frame, cell (i,j) of the query *is* cell (i,j) of the gallery.
There's no assignment problem left to solve, and no set of candidate pairings whose
geometric consistency needs checking.

**2. What does moving correspondence to extraction time gain and cost?**

Gains: correspondence is resolved once per image instead of once per comparison, so
matching becomes a single matmul; templates become fixed-length and therefore indexable;
storage drops ~25× vs DMD.
Costs: you now need a pose estimator (a whole extra model and labelled pose data), and
pose becomes a single point of failure with no recovery path. DMD has no such point.

**3. Learned Hough voting, and why it degrades gracefully.**

Every foreground pixel independently votes for where the finger's centre is and how it's
rotated, based on local ridge structure; the votes are aggregated. With only 20% of a
finger you get 20% as many votes, but they still point at the right answer — the estimate
gets noisier, not wrong. A network that regresses pose from a globally-pooled feature has
no such property: feed it a fragment and the pooled feature is simply out of distribution.

**4. Why average cos and sin, not the angle?**

Because angles wrap. If the network is split between 179° and −179° — 2° apart — the
arithmetic mean is 0°, which is 180° wrong. Averaging the unit vectors and recovering the
angle with `atan2` gives ≈180°, which is right. The mean of angles is only meaningful on
the circle, not on the real line.

**5. What is `claSum`, and why classify-then-expect?**

Predict a probability distribution over discretised bins, then take the probability-
weighted average of the bin centres — soft-argmax. Classification into bins gives a
well-conditioned, easy-to-optimise loss surface and naturally represents uncertainty
(and multi-modality), while the expectation step recovers a continuous output so you're
not stuck at bin resolution. Direct coordinate regression trains worse and gives you no
uncertainty signal.

**6. FDD vs DMD — every difference, traced to config.**

The network class is literally the same code. Differences:
`tar_shape` 256 vs 128 → 16×16 = 256 cells vs 8×8 = 64 → 3072 floats vs 768, and mask
256 vs 64. `middle_shape` 512 vs 128 → scale 0.5 vs 1.0, so FDD covers a large area at
coarse resolution and DMD a small area at fine resolution. And the *input* differs: whole
aligned finger vs one minutia patch, which is what makes FDD one-descriptor-per-image
and DMD N-per-image. (One code difference: FDD's `get_embedding` is `@torch.no_grad()`
and still computes the unused `minu_map`.)

**7. The silent pose fallback.**

If the pose `.txt` is missing, `Descdataset` prints "Do not use the pose" and proceeds
with `coarse_center` for the centre and **θ = 0** for rotation. Descriptors are then
extracted from unrotated images, so two impressions of the same finger at different
rotations produce completely different descriptors. Symptom: extraction succeeds, no
errors, and accuracy is near chance. First thing to check when FLARE results look wrong.

**8. Binarised template sizes.**

FDD: 3072 feature bits + 256 mask bits = 3328 bits ≈ **416 bytes**.
DMD at 100 minutiae: 100 × (768 + 64) bits = 83,200 bits ≈ **10.4 KB**.
Ratio ≈ **25×**. (Same ratio as the float case, since both scale linearly — DMD's
advantage isn't size, it's robustness.)

---

## File 08 — DeepPrint / flx

**1. What does `AvgPool2d(8)` cost?**

It collapses the 8×8 spatial grid to a single value per channel, so all location
information is destroyed. The cost is permanent: no spatial structure means no per-region
validity mask, and no graceful degradation when part of the print is missing.

**2. Why can't DeepPrint have a mask?**

A mask entry has to *refer* to a region. After global pooling, every output dimension is a
mixture of all regions — the contribution of an occluded area is smeared irreversibly
across all dimensions. There is nothing left that corresponds to "the top-left of the
print," so nothing to mark invalid. (File 04 §4.4.)

**3. Why zero-initialise the STN's final layer?**

Zero weights and zero bias make it output `(θ, tx, ty) = (0, 0, 0)`, so
`cos θ = 1, sin θ = 0` and the affine matrix is the identity — the STN starts as a no-op
and learns to deviate. Without it, step 1 applies a random warp, the embedding network
trains on garbage, and the two components chase each other into divergence.

**4. Why predict 3 numbers, not 6?**

Six parameters is a general affine transform, which includes shear, scale anisotropy, and
reflection — all meaningless or actively harmful for a fingerprint. Constraining the
output to rotation + translation encodes the correct prior about what variation actually
occurs between two impressions of the same finger, which makes the problem far easier
to learn and impossible to solve degenerately.

**5. Center loss vs CosFace.**

Both solve: plain softmax only makes training classes *separable*, but test-time
identities were never seen in training, so you need same-identity embeddings genuinely
tight and different-identity ones genuinely far.
Center loss adds a *separate* term pulling each embedding toward a learned per-class
centroid — extra parameters, an extra weight to tune (0.125 here).
CosFace instead *modifies the softmax*: it subtracts an angular margin from the correct
class's logit, forcing separation with room to spare. No extra parameters, and it operates
natively on the hypersphere where you'll actually compare things. Margin losses generally
won; center loss here is a period detail.

**6. `create_minutia_map`.**

Allocate an `(H, W, n_layers)` tensor where layer *k* represents ridge direction
`k · 2π / n_layers`. For each minutia, stamp a Gaussian blob at its `(x, y)`, and split
that blob's energy across layers by a softmax over the angular distance between the
minutia's orientation and each layer's orientation (taking the shorter arc around the
circle). Result: a fixed-size, dense, differentiable soft 3D histogram over (x, y, θ) —
which is exactly what you need to regress a variable-length unordered set with a CNN.

**7. Why clamp negative similarities to zero?**

Cosine similarity ranges over [−1, 1], but a negative value would mean "this fingerprint
is the *opposite* of that one," which is not a meaningful relation — fingerprints don't
have opposites. All negative values encode the same information ("not similar"), so
clamping removes a meaningless tail from the impostor distribution without discarding
anything.

**8. Open-set identification; FPIR and FNIR.**

Closed-set assumes the probe *is* enrolled, so the only question is its rank. Open-set
allows that the probe is a stranger, and the correct answer may be "no match." FPIR is
the rate at which non-mated searches wrongly return a candidate; FNIR is the rate at which
mated searches fail to return the true mate above threshold. Rank-1 can't capture this
because it's only defined for probes that *have* a mate — it says nothing about how often
you'd wrongly accuse a stranger, which in most deployments is the error that matters most.

**9. Why is argsort + two cumsums enough for the whole DET curve?**

Sorting all scores once puts every possible threshold in order. Walking that sorted list,
the cumulative count of mated scores below the cut is exactly the false-non-match count,
and the remaining non-mated count above the cut is exactly the false-match count. So two
cumulative sums give FMR and FNMR at *every* threshold simultaneously, in O(n log n)
total, instead of re-scanning the arrays once per candidate threshold.

**10. Recommended embedding size?**

512 for the texture embedding. Performance saturates around there — below it you lose
accuracy, above it you pay storage and compute for nothing. It's a useful defensible
default when you need to pick a number.

---

## File 09 — Three-way comparison

**1. The organising question.**

*Given two fingerprints, how do you know which part of A corresponds to which part of B?*
DMD: resolve it at match time (Hungarian assignment over minutiae). FLARE: resolve it at
extraction time (align both to a canonical pose). DeepPrint: dissolve it (pool away all
spatial structure so there's nothing to correspond).

**2. Is the robustness/efficiency trade fundamental?**

Largely yes, and for a concrete reason: robustness to partial prints requires knowing
*which parts are present*, which requires retaining spatial structure, which makes the
representation larger and comparison more involved. Efficiency comes from discarding
exactly that information. It isn't an accident of these three designs — FDD is the proof
that you can sit in the middle (keep a coarse spatial grid and a mask, still get one
matmul), but you can't have DeepPrint's 4 KB *and* DMD's latent performance.

**3. The five shared things.**

Two branches (texture + minutiae) concatenated; a minutiae-map auxiliary head used only in
training; an identity-classification loss plus a tightening term (CosFace or center loss);
L2-normalised embeddings compared by cosine; a binary quantisation path for scale
(DMD and FDD; flx omits it).

**4. Why 6 channels at 128×128 for the minutiae map?**

The channels are **orientation buckets** — each layer represents one ridge direction, and
a minutia's Gaussian blob is distributed across layers by angular proximity. Six buckets
over the circle is 60° apart, which is enough angular resolution to be informative without
making the target sparse and hard to regress. All three repos converged on it because it
traces back to the same source (Cao & Jain, *End-to-End Latent Fingerprint Search*).

**5. Three alignment strategies compared.**

| | Supervision | Interpretable | Failure mode |
|---|---|---|---|
| DMD (minutia frame) | none | n/a — implicit | localised; one bad minutia hurts one descriptor |
| FLARE (learned pose) | **labelled poses** | ✅ inspectable `.txt` | global and catastrophic, but detectable |
| DeepPrint (STN) | none — identity loss only | ❌ internal | global and **silent** |

**6. 10M rolled gallery, latent query.**

Cascade. Stage 1: a fixed-length method (FDD, since latents are partial and you want the
mask's graceful degradation) with an ANN index over the 10M gallery → shortlist of a few
hundred. Stage 2: DMD's local matching with Hungarian + relaxation to re-rank the
shortlist → ordered candidates for an examiner.
Shortlist size comes from Stage 1's **recall@k** curve, not its Rank-1: you pick the
smallest k at which the true mate is almost always inside, because a Stage 1 miss can
never be recovered by Stage 2.

**7. 5,000 plain prints, sub-100ms, CPU.**

DeepPrint-style flat embedding. The gallery is small and the prints are complete and
decent quality, so you don't need the mask or local matching; 5,000 dot products against
1024-dim vectors is a trivial matmul that runs in well under a millisecond on CPU. DMD is
out because per-pair Hungarian solves on CPU won't hit the latency budget; FDD is
defensible but you'd be paying for a pose estimator and 3× the template size to buy
robustness the input quality doesn't require.

**8. Which repo to learn training from?**

**flx.** It's the only one of the three that ships training code (`model_training.py`),
the loss definitions, the augmentation pipeline, *and* — crucially — the code to build
minutiae-map ground truth (`flx/data/minutia_map.py`), which all three architectures need.
It's also the only one that's properly packaged and unit-tested.

---

## File 10 — Templates and quality

**1. Three meanings of "template."**

(a) A standardised minutiae record (ISO/IEC 19794-2, INCITS 378) — **interoperable**.
(b) A proprietary SDK template — readable only by that vendor's matcher.
(c) A learned embedding — comparable only within one specific trained model.

**2. Why can't you compare embeddings across models?**

The embedding space is arbitrary: training only constrains *relative* geometry within one
model — which points are near which. Two training runs converge on different, unrelated
coordinate systems, so dimension 37 means nothing in common between them. Nothing in the
objective ties either to a shared external reference. Same architecture, different weights,
incomparable outputs.

**3. Retaining source images: for and against.**

For: algorithms improve and you can re-extract; format migrations become batch jobs rather
than re-enrolment campaigns; learned embeddings are model-locked so a model upgrade
otherwise invalidates everything; human examiners need images.
Against: storage cost, and privacy/data-minimisation obligations that may require you not
to retain raw biometrics at all. It should be a deliberate decision, not a default.

**4. Why not JPEG?**

Ridges are high-frequency structure — precisely what block-based DCT compression damages.
JPEG blocking artifacts can both destroy real minutiae and create false ones. **WSQ** was
designed specifically for 500 PPI fingerprints (~15:1), and JPEG 2000 is used at 1000 PPI.

**5. What is NFIQ 2's definition of quality?**

**Predicted matcher performance** — it's trained so the score correlates with the false
non-match rate you'd actually get from that sample. That makes it actionable: a low score
is a concrete prediction that this capture will cause a problem, which justifies a
recapture. "Looks clear to me" is not actionable.

**6. "Quality is 2, that's bad."**

Ask **which NFIQ version**. NFIQ 1 was 1–5 with **1 = best**, so 2 is near-excellent.
NFIQ 2 is 0–100 with **100 = best**, so 2 is nearly unusable. The scales are inverted and
mixing them up inverts your entire quality logic.

**7. ISO/IEC 24745 properties; why encryption isn't enough.**

Irreversibility (can't reconstruct the biometric from stored data), unlinkability
(templates of the same finger in two systems can't be cross-matched), renewability
(you can issue a fresh template and revoke the old one).
Database encryption satisfies none of them: the template is decrypted to be used, and at
that moment it's a fully reversible biometric again. Encryption protects the storage
medium; template protection protects the biometric itself.

**8. Why are embeddings easier to protect cryptographically?**

Because matching is a single dot product, and dot products are computable under
homomorphic encryption. Minutiae matching needs an assignment algorithm plus iterative
relaxation with data-dependent control flow — not practically computable on encrypted data.
So the fixed-length family admits privacy-preserving matching in a way the local-descriptor
family does not.

**9. The forgotten field.**

**Extractor and model version** on every stored template. Without it you can't tell which
rows need re-extraction after a model upgrade, and you can't detect embeddings from
different model versions being silently compared against each other. Add it before you
need it — backfilling is guesswork.

**10. What does MINEX test?**

Whether a template generated by vendor A's extractor matches correctly in vendor B's
matcher — i.e. real interoperability of standardised minutiae records. The question is
only meaningful for standard formats: proprietary templates and learned embeddings are
definitionally single-vendor / single-model, so there's nothing to test.

---

## File 13 — Touchless and contactless

**1. Which repo is not built for latents?**

**flx / DeepPrint.** It global-average-pools the feature map into a single vector, which
destroys all spatial structure. A flat embedding has no way to represent "this region is
missing," so a partial print produces an embedding that is confidently wrong rather than
usefully uncertain. File 09's table says it directly: *weakest at partial / latent prints*.
DMD is the latent specialist; FDD sits in between.

**2. "Too good for this matcher" — rewritten.**

*"We wouldn't use DMD behind a live scanner because it costs ≈333 KB per print and a Hungarian
solve per comparison, and the images are clean enough that we don't need that robustness."*
Quality never disqualifies a matcher — DMD's gallery is rolled prints, so it runs on clean
images every comparison. What disqualifies it is paying for robustness you aren't using.

**3. Why unknown scale breaks DMD specifically.**

The quantity that stops being meaningful is **patch extent in millimetres of skin**. DMD's whole
design rests on a 128-px patch covering ~6.5 mm ≈ 13–14 ridge periods (file 05 §5.4) — large
enough to contain neighbouring minutiae, small enough that distortion inside it is nearly rigid.
That equivalence holds only at 500 PPI. The config value that lies is **`img_ppi`**: it's typed
in by hand, never measured, and feeds `self.scale` in `dataloader_densemnt.py:38`. With a camera
there is no correct value to type — the scale differs between two photos taken seconds apart.

**4. Why three alignment parameters aren't enough.**

FLARE predicts `(x, y, θ)`; DeepPrint's STN predicts `(θ, tx, ty)` and builds a **rigid affine**
matrix. Both parameterise in-plane motion only, because a platen constrains the finger to a
plane. Unrepresented: **pitch and yaw** (tilting the finger toward or away from the lens, which
produces perspective foreshortening no affine transform can express) and **distance** (scale).
A model that cannot express the transformation that occurred cannot invert it.

**5. Domain gap vs. noise.**

Take a genuine pair — same finger, contact gallery image and contactless probe. With noise, some
rigid transform still overlays them and the residual is random. Here **no rigid transform, and no
single global scale, overlays them**: ridge dilation is non-uniform, strongest at the centre of
the finger pad and weakest at the curving edges. The minutiae are in genuinely different relative
positions. You're not looking at a corrupted copy of the gallery image, you're looking at a
different projection of the same 3D object — which is why the fix is a preprocessing stage that
inverts the projection, not a more robust matcher.

**6. Why the FDD mask is worth more on touchless.**

Because touchless degradation is **spatially uneven** in a way contact degradation isn't. On a
platen the whole contacting region is in focus and evenly lit — quality varies smoothly, and
mostly at the foreground boundary. With a camera, one part of the finger is sharp and well lit
while another is motion-blurred, shadowed, specular, or curving out of view, and *which* part
changes shot to shot. A per-cell validity mask can say "exclude this region from the similarity";
a flat embedding has no vocabulary for it and silently averages the garbage in.

**7. 10M contact rolled gallery, phone-photo probes.**

```
probe photo → SEGMENT → ENHANCE → SCALE (ridge-frequency PPI estimate) → UNWARP
            → Stage 1: fixed-length embedding + ANN over 10M → top ~500
            → Stage 2: local descriptor + Hungarian + relaxation → ranked shortlist
```

The cascade of file 09 §9.5 is unchanged — that's the part that didn't change, because it's
driven by gallery size, not by capture modality. What changed is everything upstream: four
preprocessing stages that don't exist in the contact pipeline, and models that must be finetuned
on paired contactless/contact data because the gallery is contact (CL2CB, not CL2CL).

**The error budget will be dominated by the front-end, not the matcher** — specifically by scale
estimation and unwarping, since an error there corrupts every downstream stage and neither
matcher has any parameter that can compensate. Set the shortlist size from recall@k as before,
but expect to need a *larger* k than the contact case: Stage 1 is a flat embedding, and it's
seeing residual domain gap that Stage 2's local descriptors tolerate better.

---

## Things to actually try

Roughly in order of effort. You don't need a GPU for the first four.

1. **Draw the pipeline from memory.** Five stages, what's on disk between each. If you can
   do this, you understand the repo.

2. **Trace one minutia by hand.** Pick a fictional minutia at (300, 450, 37°) in a 500×500
   image. What patch gets cropped? What shape comes out of the network? How many floats
   land in the pickle?

3. **Reimplement `calculate_score_torchB` on CPU** with small random tensors. Verify: (a)
   an all-ones mask gives plain cosine similarity; (b) a descriptor compared to itself
   scores 1.0; (c) zeroing half of `mask1` changes the score but not to zero.

4. **Write the metrics yourself.** Make up a 10×10 score matrix and a target matrix.
   Compute Rank-1 and TAR@FAR=10% by hand, then check against
   `rank1_general` / `TAR_flatten`.

5. **Predict an ablation, then reason it out.** What happens to Rank-1 if you (a) disable
   the mask entirely, (b) disable relaxation, (c) fix `n_pair = 12` always? Write your
   prediction *first*, then justify it from the code.

6. **Diff the two model zoos.** Open DMD's `models/model_zoo.py` and FLARE's side by side
   and confirm for yourself that the `DMD` and `FDD` classes are the same code. Then list
   every behavioural difference and trace each to a config value. (Answer: file 07 §7.7.)

7. **Diff the two matchers.** Put DMD's `calculate_score_torchB` next to FLARE's
   `calculate_score` and line them up term by term. Then list what FLARE *removed* and,
   for each, say why alignment made it unnecessary.

8. **Implement `create_minutia_map` yourself**, from `flx/data/minutia_map.py`. Feed it
   four minutiae at known angles and visualise the layers. This is the single most
   reusable piece of code across all three repos, and writing it makes the
   variable-length-set-to-fixed-tensor problem concrete.

9. **Write the angle-averaging bug, then fix it.** Average 179° and −179° arithmetically,
   then via cos/sin + atan2. Confirm the first gives 0° and the second gives 180°.
   Two minutes, and you will never make that mistake again.

10. **Estimate a real system.** For a 5M-print gallery, compute the storage and per-query
    match cost for each of the three approaches (float and binary). Then design a
    cascade and justify your shortlist size.

11. **Read flx's benchmarks properly** — `verification.py` and `identification.py`. Then
    reimplement `threshold_for_fmr` and the one-sort EER on toy data. This is the most
    transferable thing in the course; every biometric system needs these.

12. **Write the comparison yourself.** Without looking at file 09, fill in a blank
    three-column table: input unit, alignment strategy, template size, matching algorithm,
    best-at, weakest-at. Then check. This is the real test of whether it stuck.

---

## Questions worth asking me

Some prompts, if you're not sure where to poke:

- "Walk me through `lsar_score_torchB` line by line."
- "Draw the tensor shapes through FDD / DeepPrint as a table."
- "Why 6 channels? What would break at 64?"
- "Show me a toy numeric example of relaxation labeling with 3 minutiae."
- "Walk me through `dense_hough_voting4` — how do the votes actually aggregate?"
- "Show me center loss and CosFace as equations, side by side."
- "What would I change to make each of these run on CPU?"
- "How would I plug in a different minutiae extractor / pose estimator?"
- "If I wanted to train FDD myself, what's missing from the repo and where would I get it?"
- "Sketch the schema for storing templates from all three, with versioning."
- "Is my mental model right? Here's what I think happens: …"
