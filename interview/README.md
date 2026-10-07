# Software Engineering Interviews — A Course

Preparing for software engineering interviews in India, done as learning rather than
cramming. The material is the computer science that interviews probe — algorithms, how
machines and networks work, how systems are designed — taught properly. The interview
questions fall out as a by-product: each chapter ends with the ones its material prepares
you for.

**This is reference learning material.** It's about the field and how hiring in it works,
not about any one company's process. Where a company has published its process I cite it;
where the only evidence is candidate reports, I say so.

Same teaching contract as the other courses in this repo:

- I explain things you might already know. Skim if so.
- Intuition before formalism, *why* before *how*.
- Every file ends with **Check yourself** questions, answered in that area's exercises file.
- Code is **polyglot**: Go inline, with Rust, C++ and JavaScript alongside in collapsible
  blocks. Learn across all four; in the interview itself, pick one and stay in it.

## Why three areas

Interview preparation isn't one subject. It's three courses that a candidate draws on in
different rounds, and plenty of people need only one — someone strong at algorithms but
moving into a senior role might read only system design. So by the rule in
[`../CLAUDE.md`](../CLAUDE.md), each is its own course with its own numbering.

| Area | Question it answers | Rounds it prepares you for |
|---|---|---|
| [**`dsa/`**](dsa/README.md) | Given a problem, which structure and which pattern, and what does it cost? | Online assessment, DSA/coding rounds |
| [**`cs-fundamentals/`**](cs-fundamentals/README.md) | What is actually happening under my code — in the OS, the database, the network? And how do I model it in objects? | CS fundamentals, parts of every technical round |
| [**`system-design/`**](system-design/README.md) | How do I design something many people use, and defend the trade-offs? | HLD, LLD, machine coding |

Each area's README opens by naming what it assumes. They assume this folder's chapter 01
and nothing from each other.

## This folder's own chapters

| # | File | Read it | Covers |
|---|------|---------|--------|
| 1 | [How the interview loop works](01-how-the-loop-works.md) | **Prerequisite** — before any area | The rounds and what each measures; service vs product vs big tech; the machine-coding round; what AI has and hasn't changed; behavioural rounds; how to weigh interview advice |
| 2 | [Exercises & answers](02-exercises.md) | Reference | Worked answers for chapter 01 |

Chapters added later take 03 onward; the exercises file moves to stay last.

## Reference sources

The primary sources for chapter 01. Each area has its own table for its subject.

| Source | What it's good for |
|---|---|
| [Amazon — SDE II interview prep](https://www.amazon.jobs/content/en/how-we-hire/sde-ii-interview-prep) | The most detailed official loop description any big company publishes |
| [Amazon — Leadership Principles](https://www.aboutamazon.com/about-us/leadership-principles) | The 16 principles behavioural rounds are scored against |
| [Atlassian — engineering interviews](https://www.atlassian.com/company/careers/resources/interviewing/engineering) | A full official loop including values and hiring committee |
| [Meta — onsite prep](https://www.metacareers.com/swe-prep-onsite/) | Official round types and timings |
| [Canva — "Yes, you can use AI in our interviews"](https://www.canva.dev/blog/engineering/yes-you-can-use-ai-in-our-interviews/) | The clearest public description of an AI-assisted round |
| [Karat — Engineering interview trends 2026](https://karat.com/engineering-interview-trends-2026/) | Survey data on AI policy (vendor, read critically) |
| [LeetCode Discuss — interview experiences](https://leetcode.com/discuss/interview-experience/) | Candidate reports. Anecdotal, but the only source for most Indian companies |

## Reading order

| Step | Read | After this you can… |
|---|---|---|
| 1 | [`01-how-the-loop-works`](01-how-the-loop-works.md) | Say which rounds you'll face, what each measures, and where AI-assisted rounds actually exist |
| 2 | [`dsa/`](dsa/README.md) | Recognise the pattern in a new problem, solve it, and justify the complexity — out loud |
| 3 | [`cs-fundamentals/`](cs-fundamentals/README.md) | Explain what the OS, database and network are doing under a request, and model a domain in objects |
| 4 | [`system-design/`](system-design/README.md) | Run a design round end to end, and build a machine-coding solution that survives the extension |

Areas 2–4 can be read in any order, or in parallel. DSA first is the usual choice because
the online assessment gates everything else.

## If you're short on time

- **Interview in two weeks, 1–4 years' experience, Indian product company:** chapter 01 §1.3–1.4,
  then `dsa/` foundations and the pattern chapters in its short path, then `system-design/`'s
  LLD and machine-coding chapter.
- **Fresher, campus or service-company hiring:** chapter 01 §1.3 and §1.5, `dsa/`, then
  `cs-fundamentals/` start to finish — fundamentals are weighted most for you.
- **Senior role, design-heavy loop:** chapter 01 §1.5–1.7, `system-design/` in full, and
  `dsa/`'s short path to stay sharp.
- **"Has AI changed all this?":** chapter 01 §1.6. Twenty minutes.

## The one-paragraph summary

An interview loop is several noisy readings of different abilities — solving problems
(DSA), understanding the machine (fundamentals), designing systems (LLD/HLD), building
working code fast (machine coding, the round most specific to Indian product companies),
and working with people (behavioural) — combined into one decision. Which readings you get
depends on the company type and your level: freshers are measured on DSA and fundamentals,
1–4 years adds building and design, seniors are measured mainly on design and judgement. AI
has pushed companies two ways at once — tighter proctoring and a return to in-person
rounds, and, at a few companies (Canva, Meta, Google's US pilot, parts of Amazon), rounds
that permit AI and grade how you direct and verify it. Neither has replaced the loop; as of
October 2026 no Indian product company has announced an AI-permitted round. Both directions
reward the same thing: knowing the material well enough to judge code, whoever wrote it.

## How to use me

The notes are a starting point. Questions worth asking:

- "Give me a problem from `dsa/` chapter N I haven't seen, and interview me on it — don't
  give hints until I ask."
- "Run a 45-minute system design mock: design a notification service. Push back on my
  choices the way an interviewer would."
- "Here's my machine-coding solution. Extend the requirements the way a reviewer would and
  tell me where my design makes the extension expensive."
- "Ask me five STAR questions mapped to Amazon's Leadership Principles and critique my
  answers."
- "What's changed in interview formats since October 2026? Check chapter 01 §1.6 against
  current sources."
