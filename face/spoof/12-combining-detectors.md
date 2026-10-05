# 12. Combining detectors

File 08 §8.6 asked *whether* to run more than one detector. This file is the *how* — and the
how is where most of the damage gets done, because a combination rule is chosen once, in a
config file, by whoever is nearest, and then never measured.

Read 08 §8.6 first. It establishes the thing this file assumes: an ensemble is only worth
running if its members fail independently, and adding a member always costs false rejects.

## 12.1 Two families, and what each throws away

There are exactly two places you can combine. The choice matters more than the rule you
pick inside either.

```
decision fusion    each detector applies its own threshold  →  you combine booleans
score fusion       each detector reports a score            →  you combine numbers
                                                               and threshold once
```

**Decision fusion** is what a vote is. It is easy to configure, easy to explain, and it
discards the strength of every opinion. A detector reporting 0.499 and one reporting
0.000001 cast the same ballot.

**Score fusion** keeps that strength. The price is that it needs the scores to mean
comparable things, which — see §12.2 — they usually don't.

> **Teacher's aside.** People reach for a vote because it feels conservative: *"we're not
> trusting any single model."* But a vote is not the cautious option, it is the
> *lossy* one. It reduces each detector to one bit and then reasons about the bits. If
> three detectors are each mildly suspicious — none crossing its own threshold — a vote
> sees three passes and accepts. Score fusion sees the accumulated evidence. Whether that
> matters depends on your data, but "a vote is safer" is not the reason to choose it.

## 12.2 Scores are not probabilities, and rarely share a scale

Most PAD detectors emit something in [0, 1] and call it a probability. Two consequences
bite immediately.

**The same number means different things.** File 09 §9.2 already warns that a threshold
doesn't transfer between models. The same fact makes naive score fusion wrong: averaging
0.7 from one detector with 0.7 from another averages two unrelated quantities.

**Many detectors are near-binary in practice.** A model trained with a softmax over
strongly separated logits saturates: its output piles up at 0 and 1, with little in
between. When that happens the "score" carries almost no more information than the boolean
did, and score fusion degenerates towards a vote whether you intended it or not.

Both problems have the same fix, and it is the standard one from multi-biometric fusion:
**normalise before you combine.** Map each detector's raw score onto a common scale using
that detector's own distribution on genuine data — its rank, its z-score, or a tanh
transform — then fuse the normalised values. Jain, Nandakumar and Ross (2005) compare the
options and find min-max and z-score sensitive to outliers, with tanh more robust; simple
summation of normalised scores performed as well as anything more elaborate.

> ⚠️ **Check for saturation before designing anything clever.** Histogram each detector's
> raw scores on genuine data. If most values sit at the extremes, an elaborate fusion
> formula is arithmetic on two-valued inputs — you have a vote with extra steps, and you
> should either say so or fix the calibration first.

## 12.3 The rule spectrum

Every combination rule sits somewhere on one axis: **how much agreement does rejecting
require?**

| Rule | Rejects when | Character |
|---|---|---|
| OR / any | one detector objects | Most secure, most false rejects |
| Majority / *k*-of-*N* | *k* detectors object | The usual default |
| AND / unanimous | every detector objects | Fewest false rejects, weakest security |

Two things about this table are counterintuitive enough to be worth stating.

**"Unanimous" is the *least* strict setting, not the most.** The word suggests rigour, and
it means the opposite: an attack only has to fool one detector to be admitted. Read the
rule as *"how easily can this reject someone"*, not as *"how demanding does it sound"*.

**Direction is easy to invert in config.** "2 of 3" can mean *two votes needed to accept*
or *two objections needed to reject*, and those are different rules. Whenever you read a
combination setting, restate it as "an attack gets through if …" before believing you've
understood it.

## 12.4 Where a majority vote fails badly

This is the failure mode most worth internalising, because it is structural — it follows
from the definition, so no amount of tuning fixes it.

Suppose an attack is of a type only **one** of your detectors can see. The others have no
cue for it and score it genuine.

```
detector A   sees it     →  objects
detector B   blind to it →  passes
detector C   blind to it →  passes

majority (2 of 3 to reject):   admitted
OR       (1 of 3 to reject):   rejected
```

A *k*-of-*N* vote with *k* ≥ 2 catches **nothing** in this scenario, at any threshold. The
one detector that actually detected the attack is outvoted by two that could not see it.

Now recall from file 08 §8.2 why you run several detectors at all: because they fail on
*different* things. That is the same as saying you expect exactly this scenario. A
majority vote is therefore in tension with the reason the ensemble exists — it is at its
weakest precisely where complementary detection was supposed to pay off.

This does not make votes wrong. It makes them a bet: that attacks are visible to most of
your detectors at once, and that your false-reject budget can't afford OR. Both may be
true. Make the bet knowingly.

> **Teacher's aside.** Score fusion sits between OR and majority here, and its position
> depends on the threshold. Genuine captures usually score near the top of the range, so
> the fused threshold sits high — and one detector scoring near zero can drag the sum below
> it even when the others are confident. At a high operating threshold, summing behaves
> closer to "one strong objection is enough" than to "majority rules". That is a property
> of where your threshold lands, not of the sum, so verify it rather than assuming it.

## 12.5 The single point of failure in an AND rule

A detector can fail in a way that is worse than being wrong: it can get *stuck*, returning
the same confident "genuine" for everything. A stale model file, a mis-parsed response, a
default value where an error should have been.

Under a unanimous rule, one stuck-open detector means the condition "every detector
objects" can never be satisfied. The rule does not degrade — it **stops rejecting
anything**, silently, while every dashboard shows a healthy service returning 200s.

The tell is counterintuitive and worth putting in an alert: the reject rate falls. Most
monitoring watches for it rising.

Votes degrade more gracefully here: a stuck member loses one voice rather than disabling
the rule. That is a genuine argument for a vote, and a better one than "it feels safer".

## 12.6 Abstention is a third state

The most common bug in multi-detector systems is treating two different things as one:

```
"I judged this and it looks like an attack"   →  a vote against
"I could not judge this"                      →  no vote at all
```

A detector that times out, errors, or returns a malformed payload has **abstained**. If
your combination logic counts that as an objection, every outage becomes a wave of
rejections of genuine users — and it will look like an attack campaign, because that is
what a spike in rejections looks like.

The correct handling is to shrink the denominator, not to add a vote:

```
4 configured, 3 responded, rule is "majority of responders"
→ 2 of 3, not 2 of 4
```

Two consequences fall out of that, and both surprise people:

- **Whether a rule tightens or loosens under an outage depends on how it's written.** A
  rule stated as a *fraction* of responders adapts. A rule stated as a fixed *count* gets
  proportionally stricter as detectors drop out — with four configured and a count of
  three, losing one means the remaining three must be unanimous.
- **You need a floor.** "Majority of responders" with one responder is that detector
  deciding alone. Decide the minimum number of opinions you'll accept a verdict on, and
  what happens below it.

## 12.7 Evaluating a combination rule

Rules must be compared at a **matched operating point**, exactly as models are (file 02
§2.7). Comparing a rule at 3% BPCER against another at 8% tells you nothing.

The procedure:

1. Fix a BPCER budget.
2. Tune each candidate rule — including its per-detector thresholds — so it lands on that
   budget.
3. Compare worst-species APCER at that budget.
4. Report the dual too: the lowest BPCER at which each rule reaches your APCER target.

Two traps sit in step 3.

> ⚠️ **Saturation.** If every candidate rule reaches 0% APCER on your corpus, the corpus
> cannot rank them — you are choosing on the false-reject side alone and should say so.
> This is common and easy to miss, because a table of zeros reads like success. It means
> your attack set is exhausted, not that your rules are perfect. File 02 §2.2's per-species
> reporting makes this visible; a pooled average hides it.

> ⚠️ **Worst-species across mixed attack types.** The per-species rule says report the
> maximum. That is right for reporting a *system*. It is misleading for comparing
> *members*, because a detector's worst species is often one where every member struggles
> — 3D masks, say — so one number condemns a detector that may be strong on the class your
> traffic actually contains. When choosing between members, break APCER out by attack
> family and look at the whole table.

Finally, a modelling error that costs real money: **you cannot derive an ensemble's BPCER
from its members' individual rates.** Their false rejects are correlated — usually heavily,
because hard captures are hard for everyone — so multiplying or adding per-member rates
overestimates the union, often by a wide margin. Compute it from the joint data, on the
same rows.

## 12.8 Adding a detector: the honest test

Combining sounds additive and is not. Before a detector earns a place:

| Question | How to answer it |
|---|---|
| What does it cost? | Its contribution to ensemble BPCER, measured on the joint data |
| What does it catch that the others miss? | Count attacks *it alone* rejects, at matched operating points |
| Does it survive the others being wrong? | Re-run with each other member disabled |

The middle row is the one that decides it, and the one you cannot answer without labelled
attacks (file 09 §9.6). A detector whose catches are a strict subset of another's is not
adding coverage — it is a second opinion on a question already answered, and it is charging
you false rejects for it.

That may still be worth paying. Independence from a single supplier, or a model you can
retrain yourself, are real reasons. They are just not *detection* reasons, and the
distinction should survive into whatever document records the decision.

---

## Check yourself

1. A vote and a score sum are given the same three detectors and the same data. What
   information does the vote have that the sum doesn't, and what does the sum have that the
   vote doesn't?
2. Why is a "unanimous" rule the least strict setting rather than the most?
3. An attack type is visible to exactly one of your three detectors. What does a 2-of-3
   majority do, and why is that in tension with the reason for running three?
4. One detector gets stuck returning "genuine" for everything. Trace what happens under a
   unanimous rule, and say which monitoring signal would catch it.
5. Four detectors are configured; one is down. Explain how the rule behaves differently
   when it is written as "two-thirds of responders" versus "at least three must agree".
6. You compare three fusion rules and all three report 0% worst-species APCER. What have
   you learned, and what have you not?
7. Detector A rejects 4% of genuine captures and B rejects 6%. Why can't you conclude the
   pair rejects about 10%, and which direction is the error in?

(Answers in [`11-exercises.md`](11-exercises.md).)
