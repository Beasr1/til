# System Design for Interviews — A Course

Assumes [`interview/01`](../01-how-the-loop-works.md) — the shape of the interview loop.

A course on the fundamentals behind the standard system design round: how to scope a vague
prompt, estimate load, and assemble the usual building blocks (load balancers, caches,
replicated and sharded storage, queues) into a design you can defend, plus the low-level
design and machine coding rounds that Indian product companies are widely reported to run.
The level is the basic round for an engineer with one to four years' experience, not
staff-level depth.

**This is reference learning material.** It teaches the ideas, checked against the papers,
standards and documentation they come from; it isn't a set of memorised answers, and it
contains nothing about any particular company's systems. Interview questions appear in every
chapter as a by-product. **The "Asked in interviews" sections come from candidate reports and
prep material, which are anecdotal.** Treat them as "commonly reported", not as any
company's official question bank.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the problem each idea solves comes before the idea.
- Every chapter ends with **Check yourself** questions. Answers are in
  [14-exercises.md](14-exercises.md).
- Where sources disagree (they do, about CAP, about latency numbers, about retry
  arithmetic), both are given, and I say which the evidence supports.
- Code is in Go, with Rust, C++ and JavaScript versions of the key examples in collapsible
  blocks. Every code block was compiled and run; outputs shown are from those runs.

Where another course here already teaches something in depth (Kafka's delivery guarantees,
Postgres pagination and upserts, Go retries and deadlines), this one teaches the
design-level idea and links to the exact section rather than repeating it.

## The one thing to understand first

> **A system design answer is a chain of decisions, each justified by a requirement and paid
> for with a trade-off. The building blocks are nearly the same in every design; what's
> being judged is whether you found the one part of *this* problem that's hard, and whether
> every choice traces back to something the requirements said.**

## Reference sources

| Source | What it settles |
|---|---|
| Kleppmann & Riccomini, *Designing Data-Intensive Applications*, 2nd ed., O'Reilly, March 2026 ([announcement](https://martin.kleppmann.com/2026/03/24/designing-data-intensive-applications-2e.html)) | The standard deep reference for replication, sharding and consistency |
| [Google SRE book](https://sre.google/sre-book/table-of-contents/) (free online), chs. 3, 4, 21, 22 | SLIs, SLOs, SLAs, error budgets, percentiles, overload, cascading failures, retry budgets |
| DeCandia et al., [*Dynamo*](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf), SOSP 2007 | Consistent hashing with virtual nodes, quorums (3,2,2), sloppy quorum, hinted handoff, vector clocks |
| Karger et al., [*Consistent Hashing and Random Trees*](https://cs.brown.edu/courses/csci2950-u/f09/papers/chash97stoc.pdf), STOC 1997 | The definition of consistent hashing |
| Brewer, [PODC 2000 keynote](https://people.eecs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf) and [*CAP Twelve Years Later*](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/), 2012 | CAP as stated, and as its author later qualified it |
| Gilbert & Lynch, [*Brewer's Conjecture…*](https://www.comp.nus.edu.sg/~gilbert/pubs/BrewersConjecture-SigAct.pdf), SIGACT News 2002 | What was actually proved, with its definitions |
| Abadi, [*Consistency Tradeoffs in Modern Distributed Database System Design*](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf), IEEE Computer 2012 | PACELC and its system classifications |
| Kleppmann, [*Please stop calling databases CP or AP*](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html), 2015 | Why the CP/AP labels don't fit real systems |
| Nishtala et al., [*Scaling Memcache at Facebook*](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf), NSDI 2013 | Look-aside caching, delete-on-write, leases against stampedes |
| Brooker, [*Exponential Backoff And Jitter*](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/), 2015 | Backoff variants and their formulas |
| Fowler, [*CircuitBreaker*](https://martinfowler.com/bliki/CircuitBreaker.html), after Nygard's *Release It!* (2nd ed., 2018) | Circuit breaker states |
| Dean & Barroso, [*The Tail at Scale*](https://www.barroso.org/publications/TheTailAtScale.pdf), CACM 2013 | Fan-out and tail latency; hedged requests |
| [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110), [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111), [RFC 6585](https://www.rfc-editor.org/rfc/rfc6585) | Safe and idempotent methods, redirects, caching directives, `429` |
| Dean's "Numbers Everyone Should Know" ([LADIS 2009 slides](https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf)) and [napkin-math](https://github.com/sirupsen/napkin-math) | Latency numbers: the originals, and measured modern ones |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | ⭐ Explain latency, throughput, p99, availability, SLOs and Little's Law, trace one request through a typical system, and run the seven-step method. **Prerequisite for everything else** |

### Part 1 — Building blocks

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Back-of-envelope estimation](02-estimation.md) | Turn users into QPS and storage, use latency numbers as ratios, and let the numbers decide the design |
| 3 | [Scaling a service](03-scaling-a-service.md) | Make a service stateless, choose L4 or L7 and a balancing algorithm, and say what breaks when you scale out |
| 4 | [Caching](04-caching.md) | ⭐ Choose a caching pattern, explain the stale-set race and stampedes, write an LRU cache, and use a CDN safely |
| 5 | [Data storage](05-data-storage.md) | ⭐ Choose a data model by access pattern, compare replication shapes, pick a shard key, and implement consistent hashing |
| 6 | [Consistency and CAP](06-consistency-and-cap.md) | ⭐ Say what CAP proved and what it didn't, use PACELC, and state consistency per operation |
| 7 | [Asynchronous work](07-asynchronous-work.md) | Choose a queue or a log, design idempotent consumers, and explain the outbox and dead-letter queues |
| 8 | [Reliability patterns](08-reliability-patterns.md) | Implement a token bucket, size timeouts and retries, and explain circuit breakers and load shedding |
| 9 | [API design](09-api-design.md) | Choose REST, gRPC or GraphQL, paginate with cursors, make writes safe to retry with idempotency keys, and version an API |

### Part 2 — Putting it together

| # | File | After this you can… |
|---|------|---------------------|
| 10 | [Worked designs](10-worked-designs.md) | ⭐ Run the method end to end on a URL shortener, a rate limiter, chat, and a news feed |
| 11 | [Low-level design and machine coding](11-lld-and-machine-coding.md) | ⭐ Build an extensible program to a spec in 90 minutes, apply SOLID where it changes code, and see how Go, Rust, C++ and JavaScript express the same design |
| 12 | [System design in the AI era](12-the-ai-era.md) | Say honestly what has and hasn't changed in interviews, and design around an LLM as a dependency, including RAG at a glance |

### Reference

| # | File | |
|---|------|---|
| 13 | [Glossary](13-glossary.md) | Terms, each linked to its chapter |
| 14 | [Exercises & answers](14-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first, then 2. Chapters 3 to 9 can be read in any order, with two
dependencies: 6 needs 5's replication section, and 10 draws on all of them. Chapter 11
stands alone after chapter 1 (its LRU and rate limiter examples live in 4 and 8). Chapter 12
needs 4, 7 and 8 for its second half.

Not yet written, and belonging here: consensus (Raft) at interview depth, distributed
transactions (two-phase commit, sagas), search systems, object storage and media pipelines,
observability, and more worked designs (payments, ride matching, notifications). The
questions at the end of file 14 are the honest list.

## If you're short on time

- **Interview this week, 2 hours:** file 01 (all), file 02 §2.3–§2.4, then file 10 §10.1 and
  §10.4. That's the method, the arithmetic, and two complete designs.
- **Machine coding round coming up:** file 11 top to bottom, then build the parking lot
  yourself in your language without looking, then file 04 §4.4 (LRU).
- **You'll be asked about consistency and can't explain CAP cleanly:** file 06 §6.3, §6.4
  and the table in §6.7.
- **The deep-dive topics that come up most:** file 04 §4.5–§4.6 (cache races and
  stampedes), file 05 §5.5 (consistent hashing), file 08 §8.1 (rate limiting), file 09 §9.4
  (idempotency keys).

## The one-paragraph summary of everything

A system design round tests whether you can turn a vague prompt into a defensible design,
so the method matters more than any component: requirements (functional, and the
non-functional ones that actually drive architecture), estimates, API, data model,
high-level design, one or two deep dives, trade-offs. Estimation decides which world you're
in. A million requests a day is about twelve a second, and the latency ladder (memory,
SSD, in-region network, disk, cross-continent, each far slower than the last) is
remembered as ratios, because the famous numbers are dated and even their author revised
them. Measure latency in percentiles, because fan-out turns one server's rare slow request
into a common slow page. App servers scale horizontally once they're stateless, behind L4
balancers (per connection) or L7 (per request, which matters for HTTP/2 and gRPC), choosing
servers by round robin, least connections or two random choices, with active and passive
health checks. Caches cut database load but are copies: cache-aside with delete-on-write is
the default, stale sets and stampedes are the classic failures (fixed with leases,
coalescing and early expiry), every entry needs a TTL, and a cold cache can take the
database down. Databases are chosen by access pattern rather than by "SQL or NoSQL", then
replicated (single-leader for almost everything, with lag that breaks read-your-writes;
multi-leader with conflicts; leaderless with quorums where R + W > N overlaps but still
isn't linearizable) and sharded by a key chosen from the main query, with consistent hashing
moving about 1/N of keys when servers change instead of nearly all. CAP's proof covers one
register and linearizability during message loss; its "pick two" slogan was disowned by its
author, Abadi's PACELC adds the everyday latency-versus-consistency trade-off, and Kleppmann
argues the CP/AP labels don't fit real systems, so state consistency per operation instead.
Slow or optional work goes through queues or logs with at-least-once delivery, idempotent
consumers, an outbox to avoid dual writes, and dead-letter queues for permanent failures.
Reliability comes from refusing work early: rate limits (token bucket, sliding window,
shared and usually failing open), timeouts from propagated deadlines, retries at one layer
with jittered backoff and a budget, circuit breakers, bulkheads and load shedding. APIs are
REST at the edge and often gRPC inside, paginate with cursors, make writes retryable with
idempotency keys, and change only compatibly. Machine coding rounds, widely reported at
Indian product companies though documented only by candidates, reward working code whose
second requirement lands as a new type rather than six edits. And an LLM is a slow,
expensive, rate-limited, sometimes-wrong dependency that the same fundamentals handle; the
evidence says coding rounds have changed for AI, not system design rounds.

## How to use me

Ask me things like:

- "Interview me on a URL shortener. Interrupt like a real interviewer, then tell me what I
  missed."
- "Here's my design for a chat app. What's the weakest part, and what would you ask next?"
- "Check my estimate for this system. Which assumption is doing the most work?"
- "Is it right to call this database AP? Make my answer precise."
- "Review my machine coding solution: where does the next requirement hurt?"
- "Explain consistent hashing again, but start from the problem."
- "What would change in this design at 100× the load?"
