# 2. Exercises & answers

Every "Check yourself" question from this folder's own chapters, with worked answers. Each
area — `dsa/`, `cs-fundamentals/`, `system-design/` — has its own exercises file.

---

## File 1 — How the interview loop works

**1. Aced DSA, average in design, rejected for a senior role. Unfair?**
No. The loop takes readings of *different* things, and a senior role is defined by the
ones design measures — breaking down an ambiguous system, choosing trade-offs, defending
them. A strong DSA reading tells the panel nothing about those; one can't stand in for the
other. At a junior level the weighting is different, and the same scores might well pass.

**2. Twenty minutes left, working but tangled code. Refactor or test?**
Neither, first: make sure it runs end to end with the main flows demonstrable, because
non-running code is the one result that's hard to recover from. Then spend the time where
the *review* will hit — the reviewer is going to ask for an extension. Pull the part most
likely to change (pricing rules, scheduling strategy, storage) behind an interface so the
extension is a new type rather than an edit to five `if` chains. A couple of tests that
prove the main flow are worth more than coverage. The review tells you the round was never
"finish the spec"; it was "is this code cheap to change".

**3. An AI-permitted round: what will the problems look like?**
Problems where the AI's first answer isn't enough: an existing multi-file codebase to read,
a bug to find, a feature to add without breaking things, ambiguous requirements to pin down.
That's what Canva, Meta and Google's pilot describe. A LeetCode-style problem is a poor
choice because any capable model solves it on the first try — the round would measure
whether you can paste, which tells the interviewer nothing. The signal has to come from
decomposition, direction and verification.

**4. "Google now allows AI in interviews." Three questions before you believe it for Bangalore.**
- *Where's it from?* The report came from an internal document via Business Insider, not a
  Google announcement.
- *What's the scope?* A pilot, select US teams, early and mid-career roles, second half of
  2026.
- *Which round?* One code-comprehension round — not the whole loop.
None of that reaches a Bangalore interview next month. Prepare for the traditional loop and
treat an AI round as a pleasant surprise.

**5. "70% of Flipkart interviews ask about LRU cache." What's wrong with that as evidence?**
Flipkart doesn't publish its questions, so the denominator — all Flipkart interviews — is
unknowable. The figure can only come from candidate write-ups, which are self-selected
(people who post), skewed by recency and team, and frequently copied between prep sites.
A precise percentage on top of that is invented precision. The honest form is "LRU cache
appears often in candidate reports", which is still useful — it tells you to know it — but
it's a pattern, not a measurement.

**6. Do AI-assisted rounds make fundamentals more or less important?**
*Less:* if the tool writes the code, recall of algorithms matters less, and the job looks
more like specifying and reviewing.
*More:* reviewing *is* the job now, and you can only catch the wrong data structure, the
hidden O(n²) or the race condition in generated code if you'd recognise it in your own.
The tool removes typing, not judgement.
The evidence in §1.6 supports "more". Every company that describes its AI round grades
verification of the output. Pichai is widely reported as giving "strong computer science
fundamentals" as the reason for in-person rounds, though I couldn't trace where he said it. And every AI round replaces one round; it
doesn't replace the DSA and design rounds.

---

## Questions worth asking me

- "Find three candidate reports from the last six months for [company] in India and
  summarise the loop. Label what's consistent and what isn't."
- "Has any Indian product company announced an AI-permitted round since October 2026?"
- "Give me a machine-coding problem at Flipkart SDE-2 level, 90 minutes, and review my code
  like the evaluator would."
- "Interview me as an Amazon Bar Raiser on 'Ownership' and 'Disagree and Commit'."
- "What does a hiring committee actually read, and how should that change how I talk in
  each round?"
