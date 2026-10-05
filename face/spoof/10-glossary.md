# 10. Glossary

Reference, not reading. Skim once, then come back when a paper throws a term at you.

## The standard's vocabulary

| Term | Meaning |
|---|---|
| **Presentation attack (PA)** | Presenting something to a biometric sensor to subvert it. ISO/IEC 30107-1 |
| **PAD** | Presentation Attack Detection. The mechanism that spots one |
| **PAI** | Presentation Attack Instrument — the physical artefact. The printed photo, the phone, the mask |
| **PAI species** | A group of instruments made the same way. "A4 matte prints at 600dpi" is one species |
| **Bona fide presentation** | A genuine attempt by a genuine user. Preferred over "real" or "live" |
| **Liveness detection** | Informal synonym for PAD. Overclaims — most detectors measure artefacts, not biology |
| **Anti-spoofing** | Informal synonym for PAD. The most common name in papers |
| **Injection attack** | Bypassing the sensor entirely — virtual camera, patched SDK. **Not** a presentation attack, and PAD does not stop it |

## Metrics

| Term | Meaning |
|---|---|
| **APCER** | Attack Presentation Classification Error Rate. Attacks wrongly accepted ÷ total attacks. Report the **worst PAI species** |
| **BPCER** | Bona fide Presentation Classification Error Rate. Genuine users wrongly rejected ÷ total genuine |
| **ACER** | (APCER + BPCER) / 2. **Deprecated** in ISO/IEC 30107-3:2017 for industry evaluation — implies the two errors cost the same. Still common in research papers |
| **HTER** | Half Total Error Rate, (FAR + FRR) / 2. Older naming, same shape as ACER. Standard in cross-dataset work |
| **EER** | Equal Error Rate — where APCER = BPCER. Threshold chosen with hindsight, so it flatters |
| **BPCER @ APCER=x%** | The useful format. "Hold attacks at x%; this is the user cost" |
| **DET curve** | BPCER against APCER across all thresholds. The only honest way to compare two models |
| **FAR / FRR** | False Accept / False Reject Rate. Older, and in face *matching* they mean something different — check which problem you're in |

## Attack types

| Term | Meaning |
|---|---|
| **Print attack** | A photo on paper or card |
| **Cut-photo attack** | A print with the eyes cut out, worn — defeats naive blink detection |
| **Replay attack** | A screen showing a photo or video. The most common real-world attack |
| **Mask attack** | 3D artefact: paper, resin, silicone. Silicone defeats depth cues and is the expensive end |
| **Makeup attack** | A real face altered to impersonate. All liveness cues fire correctly |
| **Partial attack** | Real face plus occlusion — glasses, tape, prosthetics |
| **Deepfake** | Synthetic face. A *replay* attack if displayed; an *injection* attack if fed past the camera |

## Cues and physics

| Term | Meaning |
|---|---|
| **Moiré** | Interference pattern from photographing one pixel grid with another. The signature replay cue |
| **Half-toning** | The dot pattern a printer uses to fake continuous tone. Periodic, so visible in the spectrum |
| **Subsurface scattering** | Light entering skin, bouncing beneath the surface, and exiting nearby. Why skin looks soft and paper doesn't |
| **Specular highlight** | A mirror-like reflection. Small and moving on skin; large and flat on glossy paper |
| **rPPG** | Remote photoplethysmography — recovering pulse from tiny colour oscillations in a video of a face |
| **Parallax** | Near points shifting more than far points under motion. Zero on a flat surface |
| **NIR** | Near-infrared. Screens emit almost nothing in NIR, so a replay looks black |

## Pipeline

| Term | Meaning |
|---|---|
| **Crop expansion / scale factor** | How far the face box is enlarged before cropping. **Part of the model's contract**, not a tuning knob |
| **YuNet** | Lightweight face detector, shipped in OpenCV as `cv2.FaceDetectorYN`. Returns a box plus five landmarks |
| **NCHW / NHWC** | Tensor layouts. ONNX conventionally NCHW; TFLite NHWC |
| **Logit** | Unnormalised network output, before softmax |
| **Softmax** | Turns logits into probabilities summing to 1 |
| **Auxiliary supervision** | Training against a richer target than the label — a depth map, an FFT spectrum. Defends against shortcut learning |
| **Passive / active PAD** | Whether the user is asked to do something. See file 04 |
| **Challenge–response** | Active check where an unpredictable instruction is issued and the specific response verified |

## Models

| Term | Meaning |
|---|---|
| **MiniFASNet** | Compact CNN from the Silent-Face project. The common open baseline |
| **Silent-Face-Anti-Spoofing** | minivision-ai's repo. "Silent" = passive |
| **CDCN** | Central Difference Convolutional Network. Depth-supervised |
| **DeepPixBiS** | Deep Pixel-wise Binary Supervision. Per-pixel rather than per-image labels |
| **SE block** | Squeeze-and-Excitation. Reweights channels using global context |
| **Depthwise-separable convolution** | Factorised convolution; the MobileNet parameter-saving trick |

## Datasets

| Term | Meaning |
|---|---|
| **Replay-Attack** | Idiap, 2012. The classic small benchmark. Source of the HTER convention |
| **CASIA-FASD** | 2012. Print, cut-photo and replay at several qualities |
| **OULU-NPU** | 2017. 55 subjects, 4950 videos. Four protocols isolating environment, PAI species, camera, and all three |
| **SiW** | 2018. Pose, illumination and expression variation |
| **CelebA-Spoof** | 2020. Largest public set — 625,537 images, 10,177 subjects, 43 attributes |
| **WMCA** | 2019. 72 identities, 1941 videos, colour+depth+IR+thermal. ~80 PAIs incl. flexible silicone masks — rare among public sets |
| **CASIA-SURF** | 2019. Multi-modal RGB + depth + IR |

## Evaluation

| Term | Meaning |
|---|---|
| **Intra-dataset** | Train and test on one dataset, split by subject. Numbers are high and mean little |
| **Cross-dataset** | Train on A, test on B. Written `C→R`. The best public proxy for deployment |
| **Unseen-attack protocol** | Hold out an entire attack type from training |
| **Domain shift** | Train and deployment distributions differ — camera, lighting, subjects, compression |
| **Shortcut learning** | Model latches onto an incidental correlation (background, device) instead of the real cue |
| **Subject leakage** | Same person in train and test. Inflates results; always split by subject |
