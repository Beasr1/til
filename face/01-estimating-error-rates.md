# 1. Estimating error rates

Every number in this course is a **proportion measured on a sample**: how many of *n* things
did the thing you were counting. APCER, BPCER, FMR, FNMR — different names, same shape.

That means every one of them is an estimate with an uncertainty attached, and most of the
mistakes people make with biometric results are mistakes about that uncertainty rather than
about biometrics.

This chapter is deliberately metric-agnostic. It never asks whether you're counting attacks
that got through or impostors that matched, because the statistics don't care. Each area
defines its own metrics: presentation attacks in
[`spoof/02`](spoof/02-how-pad-is-scored.md), matching in `match/` when it lands.

## 1.1 The number you measured is not the number you have

You run 500 genuine captures through a system and 21 are rejected.

```
BPCER = 21 / 500 = 4.2%
```

The 4.2% is real, in the sense that it happened. What it is *not* is the rate you'd see on
the next 500, and treating it as one is where the trouble starts. Rerun on a fresh 500 from
the same population and you'd get something else — 3.4%, or 5.0%.

The estimate carries a **confidence interval**, and the interval is what you should quote
when the number is going to be argued about.

> **Teacher's aside.** The instinct is that a big dataset makes this go away. It doesn't
> go away, it shrinks — and it shrinks with `√n`, so a *four times* larger set halves the
> interval. That's why moving from 1,000 to 1,200 samples buys you almost nothing, and why
> people who add a few hundred samples hoping to settle an argument stay stuck in it.

## 1.2 Confidence intervals, and why the obvious formula is wrong here

The textbook interval for a proportion is the **normal approximation**:

```
p ± 1.96 × √( p(1-p) / n )
```

It is taught first, it is easy to compute, and it is unusable at the rates biometrics cares
about. Two failures, both visible immediately:

```
p = 0.004 (2 of 500)     normal 95% CI:  [-0.15%,  0.95%]   ← negative rate
p = 0     (0 of 500)     normal 95% CI:  [ 0.00%,  0.00%]   ← claims certainty
```

A negative error rate is not a rounding artefact, it's the formula being applied outside its
range. It needs `p` far from 0 and 1 and a large `n`; biometric error rates are *specifically*
the case where `p` is near zero.

Use the **Wilson score interval** instead ([Wilson,
1927](https://www.jstor.org/stable/2276774)). It is barely harder to compute, it never leaves
[0, 1], and it behaves sensibly at zero:

```
        p + z²/2n  ±  z √( p(1-p)/n + z²/4n² )
CI  =   ─────────────────────────────────────
                    1 + z²/n
```

Same two cases, done properly:

```
p = 0.004 (2 of 500)     Wilson 95% CI:  [0.11%, 1.45%]
p = 0     (0 of 500)     Wilson 95% CI:  [0.00%, 0.76%]
```

The second is the important one. Zero observed failures does **not** mean a zero rate — it
means the rate is small enough that 500 samples couldn't find one.

> ⚠️ Most spreadsheet and library defaults give you the normal approximation. If an interval
> you're shown includes a negative rate, or is exactly zero-width at zero, it's the wrong
> formula and every conclusion resting on it is soft.

## 1.3 The rule of three

Zero failures is common enough in biometric testing to deserve its own shortcut. If you
observe **0 events in n trials**, the 95% upper bound is approximately:

```
upper bound ≈ 3 / n
```

Known as the **rule of three**, from [Hanley and Lippman-Hand
(1983)](https://jhanley.biostat.mcgill.ca/c607/ch08/zero_numerator.pdf), who asked the
question this chapter is really about: *if nothing goes wrong, is everything all right?*

| Samples with 0 failures | You may claim the rate is below |
|---|---|
| 30 | 10% |
| 100 | 3% |
| 300 | 1% |
| 1,000 | 0.3% |
| 3,000 | 0.1% |

Read the table the unflattering way round. **A perfect score on 100 samples is consistent
with a 3% error rate.** If your requirement is "under 1%", 100 clean samples has not
demonstrated it and no amount of rephrasing makes it so.

This is the single most useful line of statistics in evaluation work, because "we tested it
and it caught everything" is a sentence people say constantly, and it means something
precise and usually modest.

A worked case, at the sample sizes early-stage investigations actually run at:

```
0 misses in  11 captures   →  rate could be as high as 27%
0 misses in  40 captures   →                          7.5%
0 misses in  59 captures   →                          5%
0 misses in 300 captures   →                          1%
```

Eleven clean captures feels like a working detector. It is consistent with **one in four**
attacks succeeding. Certification programmes run several hundred presentations for exactly
this reason — not bureaucratic thoroughness, but the arithmetic above.

## 1.4 How many samples do you need?

Invert the question: how tight an interval do you want, at roughly what rate?

| Target rate | Samples for ±1pp | Samples for ±0.5pp |
|---|---|---|
| ~1% | ~400 | ~1,500 |
| ~5% | ~1,800 | ~7,300 |
| ~10% | ~3,500 | ~13,800 |

Two things fall out, and both change how you plan a test set.

**Precision costs quadratically.** Halving the interval is four times the data. Decide the
precision you actually need before collecting, because "as much as we can get" converges on
the wrong amount in both directions.

**Rarer events need more data, not less.** People assume a low error rate is easier to
measure. The opposite: to *observe* a 0.1% rate at all you need thousands of samples, and to
bound it you need more.

## 1.5 Your samples are usually not independent

Every formula above assumes *n* independent observations. Biometric data routinely violates
this, and the violation always runs the same direction: it makes your intervals look tighter
than they are.

| What you counted | What you actually have |
|---|---|
| 3,000 video frames | Maybe 20 recordings. Adjacent frames are nearly identical |
| 500 captures, 50 people | 50 subjects. One person's captures resemble each other |
| 1,200 attempts | Many are retries by users who already failed once |

The unit of independence is the thing that was **sampled**, not the thing that was **scored**.
Ten thousand frames from twenty videos is a sample of twenty, and the interval should be
computed on twenty.

A related trap: a metric computed **frame-wise** over video inflates the apparent sample
size by roughly the frames-per-capture ratio, often 10× or more, for free. The metric looks
better-supported than it is and the interval shrinks by `√10`.

> **Teacher's aside.** This is the most common inflated number in biometric reporting, and
> it usually isn't dishonest — it's a genuine confusion between "how much data did I score"
> and "how many independent things did I observe". When you read *n* = 14,000 attack frames
> in a results table, the question to ask is: **how many distinct presentations?** If the
> answer is 19, the effective *n* is 19, and every interval in that table is far too tight.

The practical fixes, in order of preference: sample one frame per recording; or aggregate to
one score per recording; or bootstrap by **resampling recordings**, not frames (§1.7).

## 1.6 Comparing two rates

Two systems, same test set: A rejects 4.2%, B rejects 4.9%. Is A better?

Three things to check before answering.

**Do the intervals overlap?** With *n* = 500 each, both intervals are roughly ±1.8pp and
overlap heavily. The measurement cannot distinguish them.

**Were they measured at matched operating points?** An error rate means nothing without the
threshold that produced it. Any system can post a lower false-reject rate by accepting more —
so comparing two rates only works when the *other* error is held equal. This is why the
literature reports one rate **at** a fixed value of the other, and why a bare "4.2%" in a
table is not a result.

**Are the comparisons paired?** Run on the *same* samples, a paired test is far more
sensitive than comparing two independent intervals, because it cancels the difficulty of the
individual samples. Overlapping intervals do not prove two systems are equivalent when the
data is paired — a paired comparison can still separate them.

> ⚠️ The most common version of this error: sweeping a threshold until each system hits its
> best-looking number, then comparing those. That compares two different operating points and
> tells you nothing. Fix one error rate, compare the other.

## 1.7 Bootstrapping, for when the formula doesn't apply

Wilson covers a single proportion. It doesn't cover the quantities you actually care about —
"the worst rate across nineteen categories", "the rate after this fusion rule", "the
difference between two systems on paired data".

The **bootstrap** handles all of them, and needs no formula:

```
repeat 1,000 times:
    resample your data with replacement   ← at the unit of independence (§1.5)
    recompute the statistic
report the 2.5th and 97.5th percentiles of those 1,000 values
```

Two rules keep it honest:

- **Resample the independent unit.** Recordings, not frames. Subjects, not captures. Getting
  this wrong reproduces §1.5's error with more arithmetic on top.
- **Resample categories too**, if your statistic is a maximum across categories. A
  worst-of-nineteen is dominated by which nineteen you happened to have.

## 1.8 DET curves, and why biometrics prefers them

A threshold is arbitrary, so performance is reported as a curve over all thresholds. Two
conventions exist and biometrics almost always uses the second.

| Curve | Axes | Character |
|---|---|---|
| ROC | true-accept vs false-accept, linear | Everything good is crushed into one corner |
| DET | both **error** rates, on a **normal deviate** scale | Spreads the low-error region out |

The **DET curve** ([Martin et al.,
1997](https://www.isca-archive.org/eurospeech_1997/martin97b_eurospeech.html)) plots the two
error rates against each other with both axes warped by the normal quantile function. Two
consequences make it the right default here:

- The interesting region — both errors small — occupies most of the plot instead of a
  pixel in the corner.
- If both score distributions are roughly normal, the curve comes out approximately
  **straight**, so systems are easy to compare by eye and a kink means something real.

The single-number summaries drawn from these curves — EER, AUC — are convenient and lossy;
each area's own metrics chapter covers what they hide.

## 1.9 What to report

A defensible result gives all four:

```
the rate                  4.2%
the interval              [2.8%, 6.3%] Wilson 95%
the counts                21 / 500
the operating point       at the threshold where the other error was 1%
```

The counts matter more than they look. `21/500` lets a reader recompute anything, spot that
your *n* is smaller than they assumed, and notice when 4.2% is really 21 samples of which 15
came from one recording session.

---

## Check yourself

1. You measure 0 failures in 200 samples. What is the strongest claim you can honestly make
   about the true rate, and what makes it a bound rather than a value?
2. Why does the normal approximation produce a negative lower bound at low rates, and what
   does that tell you about when to use it?
3. Your test set is 8,000 frames drawn from 40 recordings. What is *n*, and what happens to
   your confidence interval if you use the wrong one?
4. System A scores 4.2% and system B scores 4.9% on the same 500 samples. List everything
   you need to know before saying A is better.
5. You want to distinguish a 2% rate from a 3% rate with confidence. Roughly what sample size
   are you looking at, and why is that more than people expect?
6. You need an interval for "the worst rate across 19 attack categories". Why doesn't Wilson
   answer this, and what would you do instead?
7. Why does biometrics plot DET rather than ROC, and what does a straight DET curve imply
   about the underlying score distributions?

(Answers in [`04-exercises.md`](04-exercises.md).)
