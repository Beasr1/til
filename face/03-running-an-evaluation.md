# 3. Running an evaluation

The previous chapters are about what to measure. This one is about the process around the
measuring, which is where most wasted effort in biometric work actually goes.

None of it is difficult. All of it is the kind of thing people agree with in principle and
skip under time pressure, and the cost is invisible until you're three weeks into a lead that
was dead on day one.

## 4.1 Pre-register the decision criteria

Before you collect anything, write down the number that would make you **stop**.

```
if the separation is > 5x        build on it
   between 3x and 5x             supporting evidence only, not a gate
   below 3x                      stop
```

Then collect, measure, and do what the note says.

The value isn't rigour for its own sake — it's that a result of 2.06× against a pre-agreed 3×
bar makes stopping a **calculation**, and the same 2.06× without one makes it an **argument**.
Everyone can see a reason to keep going: another capture round, a different band, one more
preprocessing tweak. Pre-registration is what converts "this isn't working" from an opinion
into an observation.

It matters most for the leads you're attached to, which is exactly when nobody wants to write
one.

## 4.2 n = 1 is not a result

A single sample that separates perfectly is not evidence of separation. It is one sample.

The failure mode is specific and recurs: you observe zero overlap between one attack and a
handful of genuine captures, conclude the signal is real, and enable it. Then the next
capture round produces false rejects by the identical mechanism — the one clean sample was a
coincidence, and coincidences at n = 1 are not rare, they're expected.

[Chapter 1 §1.3](01-estimating-error-rates.md) gives the arithmetic: zero failures in *n*
trials bounds the rate at roughly `3/n`. At n = 1 that bound is 300% — the observation
constrains nothing at all.

> **Teacher's aside.** The instinct to trust a clean separation is strong because it *looks*
> like signal, and a scatter plot with no overlap is genuinely persuasive to the eye. What
> the eye can't see is how many ways there were to get that plot by chance. Before enabling
> anything on a handful of samples, ask: how many different signals did I try before this one
> separated? That number is the multiple-comparisons correction nobody applies.

## 4.3 A signal that needs a small margin will not survive implementation

A measurement that separates classes by 1.2× offline will not separate them in production.

Between the two sit a different image pipeline, a different resolution, different
preprocessing, different floating-point behaviour, and a dozen implementation choices each
worth a few percent. In practice a quantity can move 1.3–3.7× on point density and region
choice alone — enough to swallow the whole effect.

The rule this yields: **treat the offline margin as an upper bound on the deployed margin,
and require it to be large.** A 5× separation might survive as 2×. A 1.2× separation will not
survive as anything.

## 4.4 Offline and on-device are different instruments

Related, and worth stating separately because it catches people who already believe §3.3.

Even when both implement the same equation, an offline analysis pipeline and a deployed one
are different instruments: different decoders, colour handling, resize kernels, precision.
Thresholds set from one do not transfer to the other.

> ⚠️ **Never set a deployed threshold from offline numbers.** Calibrate on the pipeline that
> will run in production, on the hardware it will run on. The equation being identical is not
> sufficient — it is the *instrument* that differs, not the maths.

This generalises the note in [`spoof/09`](spoof/09-deploying-it.md) about scores differing
across CPU architectures. Architecture is one instance; the pipeline as a whole is the
category.

## 4.5 Freeze the threshold before you touch the test set

Split your data into **calibration** and **test** before measuring anything. Choose the
threshold on calibration. Then evaluate on test, once.

Sweeping the threshold on the test set and reporting the best result measures how well you
tuned to that specific set, and the number will not reproduce. This is the same error as
selecting a model checkpoint by test-set performance, which is common enough in the published
literature that leaderboard figures should be read with it in mind.

If the data is too small to split, say so and report the number as a fit rather than an
estimate. That is an honest weak claim; a tuned number presented as an evaluation is a
dishonest strong one.

## 4.6 Check that your ground truth varies

An analysis can be inconclusive not because the method failed but because the **reference
didn't move**.

If every sample in your validation set has roughly the same true value, then an estimator
that tracks it perfectly and one that is stuck returning a constant produce identical
results. You cannot tell them apart, and it is easy to read the agreement as success.

Before trusting a comparison against ground truth, plot the ground truth. If it doesn't vary
across your set, the set cannot validate anything and you need a different one.

## 4.7 Label from the artefact, not from memory

Label every sample from the recording itself, at the time, and store the label with the data.

Labelling from recollection — *"that session was a genuine capture, I remember taking it"* —
corrupts analyses in the worst possible way, because a mislabelled sample doesn't look like a
mistake. It looks like a surprising result, and surprising results get investigated,
theorised about, and built on.

Two specific traps worth naming:

- **Paused video is a static photo.** Four captures of a paused replay are not four video
  replays; they're four photographs, and any conclusion about video replay drawn from them is
  unsupported.
- **Per-species labels must be assigned at capture.** [Chapter 1 §1.5](01-estimating-error-rates.md)
  needs the independent unit, and the per-species reporting each area's metrics chapter
  requires needs the species. Retrofitting either from filenames later is miserable and
  error-prone.

## 4.8 Look at the data before you build the harness

The single highest-return habit in this list: **dump the raw samples and look at them** before
building any orchestration around them.

Artefacts that would be obvious in an afternoon of looking at frames routinely surface only
after a probe, a debug pipeline and a parameter sweep have all been built on top of an
assumption about what the data contains.

The related one: **search the literature early**. Finding, at attempt nine, a paper stating
the exact failure mode you've been hitting is a common and avoidable experience. An hour of
searching before the first prototype is worth more than the same hour at any later point.

## 4.9 Keep a corrections file

Write down the claims you made that turned out to be wrong, and why.

This feels like bookkeeping and is worth more than most of the analysis. Three reasons:

- Confidently-asserted wrong claims get **acted on**, and often only caught much later. A
  record makes the blast radius traceable — which conclusions rested on it?
- A wrong claim frequently gets **re-derived** months later by someone who wasn't there.
- The pattern in your own errors is information. Reaching repeatedly for the same kind of
  wrong explanation is a thing you can only notice written down.

The entries that earn their place are the ones where the conclusion was right but the
*mechanism* was wrong. Those look like successes and quietly poison everything built on the
stated reason.

---

## Check yourself

1. What does pre-registering a stopping criterion buy you that deciding at the end doesn't?
2. Why is a perfect separation on one sample not evidence, and what does the rule of three
   say about it?
3. A signal separates classes by 1.2× in offline analysis. What's the honest prediction for
   its deployed behaviour, and why?
4. You implement the identical equation on device and get different numbers. Name two causes
   and say what it means for your threshold.
5. Why is sweeping a threshold on your test set and reporting the best result not an
   evaluation? What is it?
6. Your estimator agrees with ground truth on every validation sample. What would you check
   before concluding it works?
7. Why is a corrections file most valuable for claims where the conclusion turned out to be
   right?

(Answers in [`04-exercises.md`](04-exercises.md).)
