# 4. Exercises & answers

Worked answers to the **Check yourself** questions in this folder's chapters. Each area has
its own exercises file — presentation attacks in
[`spoof/11-exercises.md`](spoof/11-exercises.md).

---

## File 01 — Estimating error rates

**1. Zero failures in 200 samples. Strongest honest claim?**

By the rule of three, the 95% upper bound is `3/200` = **1.5%**. So: *"we are 95% confident
the true rate is below 1.5%"*.

It's a bound rather than a value because zero observed events contains no information about
where inside `[0, 1.5%]` the truth sits. A rate of 0.001% and a rate of 1.4% both very
plausibly produce zero failures in 200 draws. The data rules out *large* rates and says
nothing else.

The practical version: if your requirement is "below 1%", 200 clean samples has not met it.

**2. Why does the normal approximation go negative, and when should you use it?**

It's symmetric around `p` by construction — `p ± something` — with no awareness that a
proportion is bounded at zero. When `p` is small the half-width exceeds `p` and the lower
bound crosses into negative territory.

That's a symptom of using it where its assumptions fail. It needs `np` and `n(1-p)` both
comfortably above ~10, i.e. `p` far from the boundaries. Biometric error rates are precisely
the case where `p` sits near zero, so the approximation is wrong exactly where the field
works. Use Wilson.

**3. 8,000 frames from 40 recordings — what is *n*?**

**40.** The recording is what was independently sampled; frames within one are near-copies of
each other and carry almost no additional information.

Using *n* = 8,000 shrinks the interval by roughly `√(8000/40)` ≈ **14×**. A genuine ±5pp
interval gets reported as ±0.35pp — precision that doesn't exist. Every downstream comparison
inherits the error, and differences that are pure noise start looking significant.

**4. A scores 4.2%, B scores 4.9% on the same 500. What do you need?**

- **The intervals.** At *n* = 500 both are around ±1.8pp and overlap almost completely.
- **The operating point.** Both rates are meaningless without the threshold that produced
  them, and comparable only if the *other* error rate was held equal. Otherwise A may simply
  be accepting more.
- **The independent unit.** 500 what? If it's 500 captures from 50 people, *n* = 50.
- **Whether it's paired.** Same samples through both systems allows a paired comparison,
  which is more sensitive than comparing two intervals — overlapping intervals do not settle
  it either way.

Absent all four, "A is better" is a reading of noise.

**5. Distinguishing 2% from 3% — what sample size?**

You need intervals narrow enough not to overlap, so roughly ±0.5pp on each — around
**3,000–4,000 samples per system**, and more if the samples are correlated.

More than people expect because precision costs **quadratically**: halving an interval is
four times the data. The instinct that a low rate is easy to measure is backwards — to
observe a rare event at all you need many trials, and to bound it you need many more.

**6. Interval for "the worst rate across 19 categories"?**

Wilson describes a **single** proportion. A maximum across categories is a different
statistic: it's driven by whichever category happened to be worst, and it inherits
uncertainty both from each category's own sample size and from which categories you have.

**Bootstrap it.** Resample with replacement and recompute the maximum, 1,000 times, taking
the 2.5th and 97.5th percentiles. Resample **both** the samples within categories *and the
categories themselves* — otherwise you've measured uncertainty about these 19 while
implicitly claiming they're the whole world.

**7. Why DET rather than ROC, and what does a straight line imply?**

ROC plots on linear axes, which crushes the low-error region — where every usable system
lives — into a corner. DET plots **both error rates** on a **normal deviate** scale, which
expands that corner across the plot.

A straight DET line implies the genuine and impostor (or bona fide and attack) score
distributions are approximately **normal with similar variance**. That's why it's a useful
default view: the common case is a straight line, so a kink or a curve is immediately visible
and means something structural — a subpopulation behaving differently, or a score
distribution that isn't unimodal.

---

## File 02 — Retries and attempt caps

**1. Why isn't a looser rule on the same images a model of a second attempt?**

Because it re-judges **one capture** under two rules. A real second attempt is a *different
photograph*, taken by someone who has just been told they failed — different pose, light,
distance, and often more tension.

It's also drawn from a worse population. Everyone at attempt two failed attempt one, so
they're selected for whatever your system finds hard. The same-image model misses both
effects and is optimistic on both counts.

**2. 10% per capture, three attempts — why isn't loss 0.1%?**

`0.1³` assumes attempts are independent *and* that everyone takes all three. Both are wrong.

- **Abandonment**: a fifth of failures leave without retrying. This is the bigger correction
  — it takes 0.10% to 2.22%, and 200 of those 222 lost users quit after their *first* failure.
- **Correlation**: the retry population is harder, so attempt two's rate is well above 10%.
  This takes it to 3.66%.

Abandonment dominates, which is counterintuitive — people expect the statistical correction
to matter more than the behavioural one.

**3. 5% per-attempt APCER, no cap. What can you claim?**

Almost nothing reassuring. `1 − 0.95^k`: three tries gives 14.3%, ten gives 40.1%, twenty
gives **64.2%**. Fifty percent cumulative success arrives at fourteen attempts.

The honest statement is *"5% per attempt, and unbounded attempts, so the system's APCER
approaches 100% for a patient attacker."* With no cap the per-attempt figure isn't a security
property at all — it only sets how long the attack takes.

**4. Why does capping matter more once detectors can abstain?**

Abstention is the right response to an unmeasurable sample — otherwise every outage rejects
genuine users. But if abstention resolves to *retry*, an attacker can **induce** it cheaply:
obscure a sensor, submit a frame the quality gate refuses.

Each induced abstention is free and yields another draw. So a safety property becomes a
free source of attempts. Abstention stays correct; it just has to be counted against the cap.

**5. User fails, retries the same check, passes. What was demonstrated?**

That they passed the check **once**, on the second of two draws at a fixed threshold. That is
weaker evidence than passing first time, and quantifiably so: at a 5% per-attempt failure
rate, one draw passes 95% of the time and two draws pass 99.75% of the time. The bar the
subject actually cleared got lower.

What it did *not* demonstrate is anything new. The same check re-run asks the same question,
so the attacker learns from the failure and tries again. A *different* randomised challenge
would have been a new question, not a second attempt at the old one.

**6. Attempt-two pass rate of 95% vs 20%?**

- **95%** — attempt one is rejecting on transient capture noise. Nearly everyone it turns
  away is fine. Fix the threshold or the quality gate; you're adding friction and catching
  nothing.
- **20%** — retries deliver almost nothing. The people failing are failing for a persistent
  reason: their device, face, or environment. Retrying costs them time and mostly ends in the
  same place. Fail faster, or route them somewhere that can actually help.

**7. Why does a step-up challenge leak more than a plain rejection?**

A rejection returns one bit: *no*. A three-way response — accept / escalate / reject —
returns a **direction**: you were close. An attacker iterating on a spoof can use that as a
gradient, turning a black-box search into a guided one.

Harder to spot than a verbose error message because the escalation is a *feature*, designed
to reduce false rejects for genuine users. It does. It also leaks.

---

## File 03 — Running an evaluation

**1. What does pre-registering a stopping criterion buy you?**

It converts stopping from an **argument** into a **calculation**. A result of 2.06× against a
pre-agreed 3× bar is a decision anyone can read off. The same 2.06× with no bar is a debate,
and there is always a reason to keep going — another capture round, a different parameter.

It matters most for leads you're invested in, which is exactly when nobody wants to write one.

**2. Why isn't perfect separation on one sample evidence?**

Because n = 1 constrains nothing. The rule of three puts the bound at `3/1` = 300%, i.e. the
observation excludes nothing at all.

There's a second effect: you likely tried several candidate signals before this one separated.
The chance that *some* signal separates one sample cleanly is high, and no multiple-comparison
correction is being applied. A clean scatter plot is persuasive to the eye precisely because
the eye can't see how many ways there were to get it by luck.

**3. A signal separating 1.2× offline — deployed prediction?**

It will not separate. Between offline and deployed sit a different decoder, resize kernel,
colour handling, precision and preprocessing, each worth a few percent; the same quantity can
move 1.3–3.7× on point density and region choice alone.

Treat the offline margin as an **upper bound**. 5× might survive as 2×. 1.2× survives as
nothing.

**4. Identical equation, different numbers on device. Two causes, and the implication?**

Any two of: different image decoder, different resize kernel or interpolation, different
colour space handling, different float precision or fused operations, a different crop.

The implication is that they are **different instruments**, so the threshold does not
transfer. Calibrate on the pipeline that will run in production, on the hardware it runs on.
"Same equation" is not sufficient.

**5. Sweeping a threshold on the test set — what is it?**

A **fit**, not an evaluation. You've measured how well a threshold can be tuned to that
specific set, which is optimistic and won't reproduce. It's the same error as picking a model
checkpoint by test-set score.

Choose the threshold on a calibration split, freeze it, then touch test once. If the data is
too small to split, report it as a fit and say so — an honest weak claim beats a dishonest
strong one.

**6. Estimator agrees with ground truth on every sample. What to check?**

**That the ground truth varied.** If every sample has roughly the same true value, an
estimator that tracks it perfectly and one stuck returning a constant produce identical
results. The agreement measures nothing.

Plot the reference before trusting any comparison against it. A flat reference means you need
a different validation set, not a better estimator.

**7. Why is a corrections file most valuable for right-conclusion-wrong-mechanism?**

Because those look like successes, so nothing prompts a re-examination — and everything built
on the stated *reason* inherits the error silently.

If you conclude "burst capture is essential" because you believe stacking raises the peak
score, and the real mechanism is that stacking lowers the noise floor, the conclusion holds
but every prediction you derive from the wrong mechanism is wrong. A plainly wrong conclusion
gets caught by the next measurement. A right conclusion with a wrong mechanism doesn't.

---

## Questions worth asking me

- "Recompute this results table with Wilson intervals and tell me which comparisons survive."
- "My test set is video. Walk me through choosing the independent unit."
- "Show me a bootstrap of worst-species error, step by step, on toy numbers."
- "How large a test set do I need to demonstrate a rate below X%?"
- "Sketch what my DET curve would look like if one subgroup performed much worse."
- "Is my mental model right? Here's what I think a confidence interval means: …"
- "Model my funnel: here's my per-capture reject rate and attempt policy."
- "What attempt cap do I need for a given per-attempt APCER and target?"
- "Critique my evaluation plan before I collect anything."
