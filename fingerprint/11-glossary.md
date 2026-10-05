# 7. Glossary

Reference, not reading. Skim once, then come back when a paper throws a term at you.

## Fingerprint anatomy

| Term | Meaning |
|---|---|
| **Ridge** | A raised line of skin. The dark lines in a print. |
| **Valley** | The gap between ridges. Light lines. |
| **Minutia** (pl. *minutiae*) | A point where a ridge ends or splits. Stored as `(x, y, θ)`. |
| **Ridge ending** | A ridge that stops. |
| **Bifurcation** | A ridge that splits in two. |
| **Core** | Centre of the innermost curving ridge. A singular point. |
| **Delta** | Point where three ridge flows meet. A singular point. |
| **Singular point** | Core or delta — where the orientation field is discontinuous. |
| **Loop / Whorl / Arch** | The three Level-1 global pattern classes. |
| **Pore** | Sweat gland opening on a ridge crest. Level 3, needs ≥1000 PPI. |
| **Level 1 / 2 / 3** | Global pattern / minutiae / pores & ridge contours. |
| **Orientation field** | Local ridge direction at every point. Defined mod 180°. |
| **Ridge frequency** | Ridges per unit length. ~1/9 per pixel at 500 PPI. |
| **Pose** | A print's global centre and rotation. |

## Image types

| Term | Meaning |
|---|---|
| **Rolled** | Nail-to-nail roll. Largest area, best quality. |
| **Plain / slap** | Straight press. Smaller area, good quality. |
| **Latent** | Lifted from a surface. Partial, noisy, distorted. The hard case. |
| **PPI** | Pixels per inch. 500 is standard. |
| **Foreground / background** | Skin region vs. everything else. |
| **Segmentation** | Deciding which pixels are foreground. |
| **Enhancement** | Cleaning up the image so ridges are clearer. |
| **Contactless / touchless** | Captured without pressing the finger on anything. No fixed PPI. |
| **Finger photo** | A contactless print from a phone camera. Same thing, informal name. |
| **CL2CL** | Contactless probe, contactless gallery. No domain gap; the easier task. |
| **CL2CB** | Contactless probe, **contact** gallery. The one that matters — legacy galleries are contact. |
| **Ridge dilation** | Contactless ridges are wider than contact ones — no pressure flattens the skin. |
| **Out-of-plane rotation** | Roll / pitch / yaw of the finger. Impossible on a platen, unavoidable with a camera. |
| **Unwarping** | Inverting the 3D→2D projection so a contactless print resembles a contact one. |
| **Shape-from-texture** | Recovering 3D surface orientation from how ridge spacing changes across the image. |

## Matching

| Term | Meaning |
|---|---|
| **Query / search / probe** | The print you're trying to identify. |
| **Gallery / reference / enrolled** | The database you search against. |
| **Genuine / mated pair** | Two prints from the same finger. |
| **Impostor / non-mated pair** | Two prints from different fingers. |
| **Verification (1:1)** | "Is this Alice?" One comparison, threshold, yes/no. |
| **Identification (1:N)** | "Who is this?" Search a gallery, return a ranked list. |
| **Score matrix** | N_query × N_gallery table of similarity scores. |
| **Target matrix** | Same shape, 1 = genuine, 0 = impostor. Ground truth. |
| **Threshold** | The score cut-off for accept/reject in verification. |

## Metrics

| Term | Meaning |
|---|---|
| **FAR** | False Accept Rate — fraction of impostor pairs wrongly accepted. |
| **FRR** | False Reject Rate — fraction of genuine pairs wrongly rejected. |
| **TAR** | True Accept Rate = 1 − FRR. |
| **FMR / FNMR** | ISO names for FAR / FRR at the algorithm level. |
| **TAR@FAR=x** | TAR when the threshold is set so FAR equals x. The headline number. |
| **ROC curve** | TAR vs FAR as the threshold sweeps. |
| **DET curve** | FNMR vs FMR, log-log. Preferred in biometrics. |
| **EER** | Equal Error Rate — where FAR = FRR. |
| **Rank-1 / Rank-k** | Fraction of queries whose true mate is at position 1 / in the top k. |
| **CMC curve** | Rank-k accuracy plotted against k. |
| **Closed-set identification** | The probe is assumed to be in the gallery; only its rank matters. |
| **Open-set identification** | The probe may not be enrolled at all; wrongly returning a candidate is an error. |
| **FPIR** | False Positive Identification Rate — non-mated searches that wrongly return a candidate. |
| **FNIR** | False Negative Identification Rate — mated searches where the true mate isn't in the candidate list. |
| **Recall@k** | Fraction of probes whose true mate is in the top k. What you tune a shortlist size from. |

## Descriptors and learning

| Term | Meaning |
|---|---|
| **Descriptor / embedding** | A vector summarising appearance, so similar things → nearby vectors. |
| **Local descriptor** | One per minutia. Variable count per print. |
| **Fixed-length descriptor** | One per finger, constant size. Fast to search, needs alignment. |
| **Cosine similarity** | `(a·b)/(‖a‖‖b‖)`. Direction-only similarity, in [−1, 1]. |
| **MCC** | Minutia Cylinder-Code. The hand-designed local descriptor DMD learns a version of. |
| **DeepPrint** | 2019 paper: one learned fixed-length vector per fingerprint. |
| **DMD** | Dense Minutia Descriptor — this course's subject. |
| **FDD / FLARE** | Fixed-length Dense Descriptor, and the framework around it. Same lab. |
| **Dense descriptor** | One that keeps spatial structure instead of pooling it away. |
| **Validity / foreground mask** | Per-cell flag for "is this part of the descriptor real?" |
| **CosFace / ArcFace** | Margin-based softmax losses for training embeddings. |
| **Center loss** | Adds an explicit pull toward a learned per-class centroid. DeepPrint's alternative to CosFace. |
| **Auxiliary loss** | A secondary training objective used only to shape the shared representation. |
| **Position encoding** | Added signal telling a spatially-blind layer where it is. |
| **Global average pooling** | Collapsing a feature map's spatial axes to one value per channel. Destroys location. |
| **Minutia map** | A fixed-size tensor encoding a variable-length minutiae list: Gaussian blobs spread over orientation layers. |
| **STN** | Spatial Transformer Network — a differentiable warp whose parameters are predicted by a small net. |
| **Soft-argmax** | Classify into bins, then take a probability-weighted average of bin centres. Trainable *and* continuous. |
| **Pose** | A print's global centre and rotation, `(x, y, θ)`. |

## Algorithms

| Term | Meaning |
|---|---|
| **Gabor filter** | Sinusoid × Gaussian. Responds to stripes of a given frequency & orientation. |
| **Thinning / skeletonisation** | Eroding ridges to 1-pixel-wide lines. |
| **Crossing number** | Neighbour-transition count used to classify skeleton pixels into minutiae. |
| **Hough transform** | Voting scheme for finding a global alignment. |
| **Hungarian algorithm / LSA** | Optimal one-to-one assignment minimising total cost. O(n³). |
| **Relaxation labeling** | Iteratively refining confidences using mutual geometric consistency. |
| **TPS** | Thin Plate Spline. Smooth warp defined by control-point displacements. |
| **LSA-R / LSAR** | This repo's name for LSA + Relaxation. |

## Templates, formats, quality

| Term | Meaning |
|---|---|
| **Template** | Ambiguous: a standard minutiae record, a proprietary SDK blob, or a learned embedding. |
| **ISO/IEC 19794-2** | The classic finger-minutiae interchange format. INCITS 378 is the US equivalent. |
| **ISO/IEC 39794-2** | Successor generation of interchange formats; extensible (ASN.1/XML). |
| **ANSI/NIST-ITL 1** | Transaction format for exchanging biometric data. Type-4/13/14 images, Type-9 minutiae. |
| **EBTS** | The FBI's profile of ANSI/NIST-ITL. |
| **NFIQ 2** | NIST Fingerprint Image Quality v2. 0–100, higher is better. Standardised as ISO/IEC 29794-4. |
| **WSQ** | Wavelet Scalar Quantization — the FBI's compression spec for 500 PPI fingerprint images. |
| **ISO/IEC 24745** | Biometric information protection: irreversibility, unlinkability, renewability. |
| **ISO/IEC 19795** | Biometric performance testing and reporting. |
| **MINEX / PFT / FpVTE / ELFT** | NIST evaluations: template interoperability / 1:1 / 1:N / latents. |
| **Cancelable biometrics** | Storing a revocable, non-invertible transform of the biometric instead of the biometric. |
| **Slot** | The key a template is stored under, typically (subject, finger position, source). |

## Repo-specific

| Term | Where | Meaning |
|---|---|---|
| `pose_2d` | dataloader | The anchor minutia `(x, y, θ)` for a patch. Not a global pose. |
| `ndim_feat` | config | Channels per branch. 6. Final descriptor = `2 × 6 × 8 × 8` = 768. |
| `tar_shape` | config | Patch size fed to the network, `[128, 128]`. |
| `middle_shape` | config | Intermediate size in the PPI scale calculation. |
| `n12` | scoring | Effective overlap: `Σ m₁ᵢ · m₂ᵢ`. |
| `N_mean` | scoring | Normalisation constant. Passed as 5 at the call site. |
| `f2f_type` | scoring | Image-type pair for mask thresholds. `(2,1)` = latent query, rolled gallery. |
| `efficiency` | scoring | `λ_after_relaxation / λ_before`. The sort key for pair selection. |
| `n_pair` | scoring | How many top pairs to average. Adaptive, 4–12. |
| `minu_map` | model | Auxiliary minutiae-map head. Training only. |
| `NoMP` | config | Training statistic (mean minutiae per print). Unused at inference. |
| `fptools` | dependency | External helper lib: `uni_io`, `fp_verifinger`. Gitignored. |
| `search` vs `query` | everywhere | The same thing. `image/query/` on disk, `search/` in the feature cache. |
| `FDD` | FLARE | Fixed-length Dense Descriptor. The model class — same code as DMD's, bigger input. |
| `GRIDNET4` | FLARE | The voting-based pose model. FingerNet-derived, U-Net decoder + dense Hough voting. |
| `claSum` / `claMax` | FLARE | Soft-argmax variants: expectation over all bins / over near-peak bins only. |
| `FastCartoonTexture` | FLARE | TV decomposition separating smooth illumination from oscillatory ridge texture. |
| `coarse_center` | FLARE | Silent fallback centre estimate (blur + morph open + weighted centroid) when no pose file exists. |
| `subject` / `impression` | flx | **Their** usage: a distinct *fingerprint* / one capture of it. Not a person. |
| `DeepPrint_[Loc][Tex][Minu]_<N>` | flx | Variant naming: localization net / texture branch / minutia branch / embedding dims. |

## Datasets

| Name | What |
|---|---|
| **NIST SD4** | 2000 rolled pairs. |
| **NIST SD14** | 27,000 rolled pairs. DMD's training set. |
| **NIST SD27** | 258 latents + mates. The latent benchmark. |
| **N2N** | Nail-to-Nail challenge data; has a latent subset. |
| **FVC 200x** | Fingerprint Verification Competition sets. Plain prints. |
