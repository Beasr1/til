# 2. Retries and attempt caps

Almost every deployed biometric gate lets a user try again. Almost every evaluation of one
measures a **single attempt**. The gap between those two sentences is where a system's real
error rates live, and it runs in both directions at once: retries make the false-reject
number look worse than you modelled *and* the security number look better than it is.

This chapter is metric-agnostic. It applies wherever a subject can re-present — liveness,
matching, or a quality gate.

## 3.1 A retry is a new sample, not a second look

The tempting model is that attempt two re-examines attempt one. It doesn't. The user moves,
the light changes, they hold the phone differently because the app told them to. Attempt two
is a **fresh capture** drawn from a different, and worse, distribution.

Worse, because of who is taking it. Everyone reaching attempt two failed attempt one, so the
retry population is **selected for difficulty** — unusual faces, bad cameras, dim rooms,
whatever your system finds hard. Its error rate is conditional on having already failed, and
that conditional rate is far above the population rate.

> ⚠️ **The modelling error to avoid.** Taking your scored dataset, applying rule A to get
> "attempt 1 failures", then applying looser rule B to *the same rows* to get "attempt 2
> failures". That measures one image judged twice. It has nothing to say about a second
> capture by a struggling user, and it will be optimistic by a wide margin.

## 3.2 Abandonment is a loss your model never sees

Some users who fail simply leave. They appear in no score distribution, no error rate, and
no confusion matrix — the system never got a second capture to judge.

They are, nevertheless, exactly the outcome the business cares about.

This is why per-capture and per-journey numbers diverge so sharply. Three models of the same
gate, all with a 10% per-capture reject rate and three attempts allowed:

| Model | Assumption | Journey loss |
|---|---|---|
| A | Attempts independent, nobody abandons | **0.10%** |
| B | A, plus 20% abandon after each failure | **2.22%** |
| C | B, plus the retry population is harder (10% → 40% conditional) | **3.66%** |

Model A is `0.10³` — the calculation people reach for. Model C is **37 times larger**, and
it is the one that matches how funnels actually behave.

Note where the loss sits in model B: 200 of the 222 lost users abandoned after their *first*
failure. They never exercised the retry the design was counting on. Adding a fourth attempt
would have helped almost none of them.

> **Teacher's aside.** This is why "we allow three attempts" is not the reassurance it
> sounds like. Retries only help users who take them. If a fifth of your failures leave
> immediately, your effective allowance is much closer to one attempt than three, and the
> way to find out is to measure the funnel rather than reason about the policy.

## 3.3 The same retries are a gift to an attacker

Flip the sign. An attacker is not discouraged by failure and does not abandon. Every retry is
another **independent draw**, and they need exactly one success.

`P(at least one success in k attempts) = 1 − (1 − p)^k`

| Per-attempt APCER | 3 tries | 5 tries | 10 tries | 20 tries |
|---|---|---|---|---|
| 1% | 3.0% | 4.9% | 9.6% | 18.2% |
| 2% | 5.9% | 9.6% | 18.3% | 33.2% |
| 5% | 14.3% | 22.6% | 40.1% | **64.2%** |
| 10% | 27.1% | 41.0% | 65.1% | 87.8% |

Or read as attempts-to-succeed: at 5% per attempt, an attacker reaches **50% cumulative
success in 14 tries** and 90% in 45.

A per-attempt APCER of 5% is a respectable number that many systems would ship. Without a
cap it is a 64% breach rate against anyone patient enough to try twenty times, and twenty
attempts is thirty seconds of work.

**The per-attempt error rate is not the system's error rate.** With unbounded retries the
system's true APCER approaches 100% for any `p > 0`; the only question is how long it takes.

## 3.4 Abstention becomes an attack surface

A detector that cannot judge a sample should **abstain** rather than count as a vote against
it — otherwise every outage becomes a wave of rejections of genuine users. That principle is
right, and it has a consequence that only appears once retries enter the picture. (For how
abstention interacts with combining several detectors, see
[`spoof/12` §12.6](spoof/12-combining-detectors.md).)

If "cannot measure" resolves to *retry*, and retries are unlimited, then:

- the attacker can **induce** abstention cheaply — cover a sensor, move out of range, submit
  a frame the quality gate rejects
- each induced abstention costs them nothing and yields another draw
- so a safety property inverts into a free source of attempts

Abstention is correct. **Unbounded abstention is not.** The fix is not to make the detector
guess; it is to cap and to count.

## 3.5 What to actually do

| Control | Why it works |
|---|---|
| Cap attempts per session | Bounds the geometric series in §2.3 directly |
| Rate-limit per identity, per interval | Stops the cap being reset by starting a new session |
| Count abstentions against the cap | Otherwise §2.4 makes the cap decorative |
| Treat a burst of retries as a signal | A genuine user retrying twice is normal; fifteen is not |
| Escalate rather than repeat | A different challenge is a new question. The same one again is another draw at the same lock |

The last row is the important one and the most often missed. Letting a user retry the
*identical* check is handing out repeated attempts at a fixed threshold. Changing the
challenge — a different randomised prompt — means the previous attempt taught the attacker
little, because the new question wasn't answerable in advance.

> ⚠️ **Escalation leaks.** If a borderline result triggers a *different* flow — a step-up
> challenge, an extra capture — the escalation itself tells an attacker they were close. A
> plain rejection returns one bit; a three-way response returns a direction, which converts
> a black-box attack into a guided search. The same reasoning applies to detailed error
> messages — see [`spoof/09` §9.4](spoof/09-deploying-it.md) — but it is easier to miss
> here, because the escalation is a *feature* designed to help genuine users.

## 3.6 Measuring it

None of the above is visible in a per-capture evaluation. What to instrument:

```
attempts per journey            distribution, not mean
pass rate by attempt number     attempt 2 will be far below attempt 1
abandonment rate by attempt     of those who failed, how many never returned
final journey outcome           passed / lost / abandoned — the number to report
```

Two rules for reporting:

- **Report journey outcome, not capture outcome**, whenever a human is being described.
  Per-capture rates are for tuning a model; per-journey rates are what happened to people.
- **Pass rate by attempt is the diagnostic.** If attempt two passes almost everyone, your
  first attempt is rejecting on transient capture noise and the threshold or the quality gate
  is wrong. If attempt two passes almost nobody, retries are costing users time and delivering
  nothing, and the honest move is to fail faster.

---

## Check yourself

1. Why is applying a looser rule to the same scored images not a model of a second attempt?
2. A gate rejects 10% of captures and allows three attempts. Why is journey loss not 0.1%,
   and which of the two corrections matters more?
3. Your system has a 5% per-attempt APCER and no attempt cap. What is the security claim you
   can honestly make?
4. Why does capping attempts matter more once detectors are allowed to abstain?
5. A user fails, is offered the *same* check again, and passes. What did the second attempt
   demonstrate, and what did it not?
6. Attempt-two pass rate is 95%. Attempt-two pass rate is 20%. What does each tell you to go
   and fix?
7. Why does a step-up challenge on a borderline score leak more than a plain rejection?

(Answers in [`04-exercises.md`](04-exercises.md).)
