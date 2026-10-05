# 9. Deploying it

Papers stop at the metric. This is the part after.

## 9.1 The gap between a model and a system

A PAD model gives you a score. A system has to decide, explain, retry, and not collapse
under real traffic. Most of the distance between "the model works" and "the product works"
is in this file, and almost none of it is machine learning.

The single most useful reframing: **your production false-reject rate is a business
number, not a model number.** It is set by the model, the threshold, the capture loop, the
device mix and the user population together. Optimising the model alone moves it less than
you'd expect.

## 9.2 Choosing a threshold

You cannot pick a threshold from a paper. You pick it from your own data and your own
appetite for the two errors.

The procedure:

1. Decide the security requirement first, in the form file 02 §2.7 recommends — *"no more
   than X% of attacks may succeed"*. This is a business decision, not an engineering one.
2. Collect labelled data from your own pipeline. Both classes. See §9.6 if you can't.
3. Sweep the threshold, plot BPCER against APCER.
4. Pick the threshold meeting the APCER requirement, and read off the BPCER.
5. **Show that BPCER to whoever owns conversion**, before launch, not after.

Step 5 is the one that gets skipped and the one that causes the argument. A 10% false
reject rate is a fine engineering result and a catastrophic onboarding funnel. Better to
have that conversation with a number than with a support queue.

> ⚠️ **Thresholds don't transfer across architectures.** Scores from two models are not
> comparable even when both are "probabilities" — they're outputs of different softmaxes
> over differently-scaled logits. A 0.5 that works on one model means nothing on another.
> Recalibrate per model, always.

## 9.3 Report everything, decide once

A PAD service should return **more than a boolean**. Useful response content:

| Field | Why |
|---|---|
| Overall verdict | The decision |
| Per-model scores and raw logits | Lets a caller apply a different rule without a second request |
| Which threshold was applied | Makes the verdict reproducible later |
| The face box used, and its source | Most "wrong answer" reports are actually wrong-crop reports |
| Capture quality measurements | Distinguishes "attack" from "unusable frame" |
| Timings | Capacity planning |

The reason is practical: when someone reports that a real user was blocked, "false" tells
you nothing. The box, the scores and the quality flags tell you whether the detector picked
a background face, whether the frame was too dark to judge, or whether the model genuinely
disagreed.

It also lets the caller own its own policy. A consumer that wants to be stricter for
high-value transactions can be, without you shipping a second endpoint.

## 9.4 Keep the verdicts separate

Three different questions get collapsed into one boolean far too often:

```
no face found          → a capture problem. Ask the user to reframe
frame not judgeable    → a quality problem. Ask for more light
face judged an attack  → a security decision. Do not coach the user
```

They need different user-facing behaviour and different alerting. A spike in "no face" is
a detector or SDK problem. A spike in "attack" is either an incident or a regression. If
they share a counter you can't tell which is happening.

> **Teacher's aside.** Note the asymmetry in the third row. For the first two you should
> tell the user exactly what to fix. For the third, you shouldn't — detailed feedback on
> *why* an attack was detected is a free tuning signal for the attacker. "Please try again"
> is the correct amount of information.

## 9.5 What to monitor

| Signal | What a change means |
|---|---|
| Reject rate overall | Regression, or an actual attack campaign |
| Reject rate **by device model** | Nearly always a domain-shift or camera problem |
| Reject rate by hour | Lighting. Evening spikes are a capture problem, not fraud |
| No-face rate | Detector threshold, or an SDK/camera change |
| Score distribution | The early warning. Shifts before the reject rate does |
| Retry-then-succeed rate | High means your gate is annoying rather than protective |

**Score distribution is the one to instrument first.** A threshold turns a continuous
signal into a boolean and throws away the evidence that something is drifting. Histogram
the raw scores and you'll see a population shift while the reject rate is still flat.

**Retry-then-succeed** is the underrated one. If most rejected users pass on the second or
third attempt, you are not blocking attacks — you are adding friction to genuine users and
then letting them through anyway. That pattern means the threshold is wrong, or the capture
loop is submitting frames it shouldn't.

## 9.6 When you only have genuine data

Common situation: plenty of real captures, no labelled attacks.

**What you can measure:** BPCER. Run the model over genuine captures and count rejections.
Real, useful, and often sobering.

**What you cannot measure:** APCER. Nothing in genuine-only data says whether attacks get
through.

**What you must not conclude:** that a model with a low reject rate is good. A model that
accepts everything scores perfectly on genuine-only data.

This asymmetry has a sharp consequence when comparing models. Genuine-only data can tell
you model A rejects fewer real users than model B. It cannot tell you whether B was
rejecting them *because it catches more attacks*. Dropping B on that evidence is a security
change made on non-security data — which may still be the right call, but should be made
knowingly and stated plainly.

Getting even a small attack set is worth disproportionate effort: print twenty photos, take
twenty screen replays on three devices, label them by species. A few hundred samples
gathered in an afternoon converts guesswork into a measurement.

## 9.7 Operational realities

**Model files belong in the image.** Fetching weights at runtime means a network dependency
in your startup path and no guarantee which version is running. Vendor them; the image is
bigger and the deployment is reproducible.

**Configuration should be remountable.** Thresholds are the thing you will change most and
should be the thing that requires least ceremony to change. A config file mounted into the
container beats a rebuild.

**Echo the active configuration on a health endpoint.** When a tuning change appears not to
have taken effect, the first question is which config is actually loaded. Answering that
from outside the container saves hours.

**Fail closed, and know what your defaults are.** If a model file is missing or a config
mount fails, decide deliberately whether the service refuses to start or runs on baked-in
defaults. Both are defensible; silently running defaults while everyone believes the
mounted config is live is not.

**Scores can differ across CPU architectures.** Floating-point associativity differs, so
the same model on the same image can produce slightly different logits on arm64 and amd64.
Usually small — but a sample sitting either side of a threshold flips. Calibrate on the
architecture you deploy on, and be suspicious of a threshold within a hair of a cluster of
samples.

---

## Check yourself

1. Why can't you take a threshold from a paper?
2. A user reports being wrongly blocked. Which fields in the response do you need to
   diagnose it, and why isn't the boolean enough?
3. Why should "no face", "too dark" and "looks like an attack" be distinguishable, and why
   should only two of them get detailed user feedback?
4. What does a high retry-then-succeed rate tell you?
5. You have genuine-only data. Model A rejects 4% and model B rejects 19%. What can you
   conclude, and what can't you?
6. Why instrument the score distribution rather than only the reject rate?
7. The same image scores 0.173 on one server and 0.180 on another, same model. Plausible
   cause, and what does it imply for threshold choice?

(Answers in [`11-exercises.md`](11-exercises.md).)
