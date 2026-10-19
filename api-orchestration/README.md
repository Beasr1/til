# API Orchestration — A Course

A course on the engines that chain API calls together. It covers:
- turning a graph of steps into a run;
- passing data from one step to the next;
- retrying without repeating side effects;
- surviving a crash, or deciding not to;
- refusing to be turned against their own network;
- describing their APIs so that two codebases agree;
- AI agents that are starting to build the workflows themselves.

**This is reference learning material.** It's about the general problem,
not any one product. Products appear as examples of a design choice
(Airflow, Temporal, Kestra, n8n, Windmill, KrakenD), and every claim about
one is cited to its own documentation.

Assumes [go-services](../go-services/README.md) files 01, 02, 04 and 05:
how a Go service runs, the HTTP client, deadlines and retries, and fanning
out. Files 06, 08 and 09 are linked where they add depth. Chapters link back
to them rather than re-teaching them.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the fix.
- Code is in Go. Each core example also has Rust, C++ and JavaScript
  versions in collapsibles, and every example is run before it goes in.
- Every chapter ends with **Check yourself** questions. Answers are in
  [09-exercises.md](09-exercises.md).

## The one thing to understand first

> **A workflow engine is a scheduler, a data store and an HTTP client that
> other people program.** Each of the three is ordinary on its own. The
> hard part is that the people writing workflows aren't the people running
> the engine. They type the URLs, write the expressions and choose the
> retries. So every limit the engine enforces is a limit on someone else's
> code: on time, on recursion, on where requests may go, on what gets
> retried.

## Reference implementations

| Source | What it settles |
|---|---|
| [Airflow: DAGs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html) | Trigger rules, and how skips cascade |
| [Temporal: retry policies](https://docs.temporal.io/encyclopedia/retry-policies) | Retrying an activity; a per-attempt timeout vs a bound on the whole sequence |
| [Kestra: server components](https://kestra.io/docs/architecture/server-components) | The moving parts of a job-style engine |
| [n8n: Respond to Webhook](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.respondtowebhook/) | A workflow that answers the request that started it |
| [KrakenD: sequential proxy](https://www.krakend.io/docs/endpoints/sequential-proxy/) | Request-style composition inside an API gateway |
| [AWS Step Functions: workflow types](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html) | Standard vs Express: the job-style or request-style choice, made per state machine |
| [JSONata](https://docs.jsonata.org) / [jsonata-js](https://github.com/jsonata-js/jsonata) / [gnata](https://github.com/recolabs/gnata) | The expression language, its reference implementation, and a Go one |
| [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) | Calling URLs you don't control |
| [OpenAPI](https://spec.openapis.org/oas/latest.html) / [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) / [openapi-typescript](https://openapi-ts.dev) | One description of an API, generated into both codebases |
| [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) / [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901.html) / [RFC 6902](https://www.rfc-editor.org/rfc/rfc6902.html) / [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) | Problem details, JSON Pointer, JSON Patch, conditional requests |
| [Model Context Protocol, 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog) | How agents discover and call tools |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | ⭐ Tell a job-style engine from request-style composition, use this course's vocabulary, and place the common products on that spectrum. **Prerequisite for everything else** |

### Part 1 — Running a workflow

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Graphs and scheduling](02-graphs-and-scheduling.md) | ⭐ Reject a cyclic workflow with a useful error, write a scheduler that owns a run's state, and say what happens to a step whose parent was skipped |
| 3 | [Mapping data between steps](03-mapping-data-between-steps.md) | Build a request from earlier results with JSONata, put limits on an expression someone else wrote, and explain why preview and run must share an evaluator |
| 4 | [Retries, idempotency and durability](04-retries-idempotency-and-durability.md) | Retry a POST without doing it twice, honour `Retry-After`, and choose how much of a run should survive a crash |

### Part 2 — Living with other people's input

| # | File | After this you can… |
|---|------|---------------------|
| 5 | [Calling URLs you don't control](05-calling-urls-you-dont-control.md) | ⭐ Explain why checking a URL string doesn't stop SSRF, and refuse private addresses at connection time in four languages |
| 6 | [Describing APIs](06-describing-apis.md) | Generate both sides of an API from one OpenAPI file, return errors that point at the offending field, and stop two editors overwriting each other |

### Part 3 — Agents

| # | File | After this you can… |
|---|------|---------------------|
| 7 | [Agents that build workflows](07-agents-that-build-workflows.md) | Explain the draft, validate, publish pattern, design tools an agent can repair from, and say which risks are solved and which aren't |

### Reference

| # | File | |
|---|------|---|
| 8 | [Glossary](08-glossary.md) | Terms, one line each |
| 9 | [Exercises & answers](09-exercises.md) | Worked answers, things to try, and questions to ask |

Read file 01 first: every other file assumes its vocabulary and its data
document. File 04 also assumes 02. Files 03, 05 and 06 need only 01, so read
them in any order. File 07 leans on most of the others, 06 above all.

## If you're short on time

- **New to workflow engines:** file 01, all of it.
- **Letting users type URLs into your product:** file 05, all of it.
- **A retried step did its side effect twice:** file 04.
- **Building an editor with a live preview:** file 03.
- **Adding an AI builder to a product:** file 07, then 06.

## The one-paragraph summary of everything

An orchestration engine runs workflow definitions: it validates them,
schedules steps, maps data between them, applies failure policy and records
each run. The question that decides its design is whether anyone is holding
a connection open for the answer. If they are, the caller's deadline
dominates: fail fast, keep retries few and short, and hold state in memory.
If not, durability dominates and every step's result is written down.

Validation and scheduling are one algorithm. Kahn's topological sort rejects
a cycle and names the steps on or behind it, and the scheduler is Kahn's
queue with network calls in it. One goroutine owns the run's state, and step
goroutines only report back. Fan-in is keyed by step id, never by arrival
order. A skipped parent cascades unless the join's rule says otherwise:
Airflow's `none_failed_min_one_success` is the usual join after a branch.

Mappings are expressions over a data document: the input plus each finished
ancestor's response. JSONata is a reference implementation plus a family of
ports that agree on the portable test suite and disagree just outside it, so
preview and run must use the same evaluator. Its guardrails bound time and
depth but not memory. A backtracking regex can run straight past a timeout,
where RE2 can't. And expressions are untrusted code: n8n's CVE-2025-68613
turned one into remote code execution.

Retrying a step means its earlier side effects may already have happened.
Durable execution gives exactly-once recording, not exactly-once effects.
Only the upstream recognising a repeat makes a retry safe, either through a
key it honours (`Idempotency-Key` is still an expired draft) or a natural key
it upserts on. `Retry-After` arrives as seconds or as an HTTP-date, and a
per-attempt timeout is not a bound on the whole retry sequence.

When users type the URLs, checking the string fails to DNS rebinding,
redirects and alternative IP spellings, which are all time-of-check to
time-of-use gaps. The check belongs at connection time, on the resolved
address, for every hop, in each language's own hook, and some hooks have
holes: reqwest's resolver skips IP literals, and `tls.connect` ignores Node's
`blockList`.

One OpenAPI description can generate both codebases, but generated code is
two compile-time promises about your own code, not a check on the bytes that
arrive. Errors should be RFC 9457 problem details with JSON Pointers, edits
should be JSON Patches, and concurrent edits should be guarded by `If-Match`.

Agents can draft workflows from a catalogue, repair them from pointer errors
and test them with a dry-run. But a published workflow holds all three of
the Rule of Two's properties on every run, so the person who clicks publish
is the control the whole design leans on.

## How to use me

Ask me things like:

- "Is this workflow a DAG? If not, which steps are in the cycle?"
- "Step `crm` failed after posting. What does a retry do now?"
- "Is this JSONata expression safe to run on input I don't control?"
- "Can a user point this HTTP client at our cloud metadata endpoint?"
- "Generate both sides of this endpoint from OpenAPI and show me what drifts if I edit one."
- "What tools would an agent need to build this workflow from a written spec?"
