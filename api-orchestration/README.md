# API Orchestration — A Course

A course on the engines that chain API calls together. It covers:
- turning a graph of steps into a run;
- joining branches again, and knowing what a join may find missing;
- passing data from one step to the next;
- retrying without repeating side effects;
- surviving a crash, or deciding not to;
- recording each run without slowing it, and showing it live;
- refusing to be turned against their own network;
- describing their APIs so that two codebases agree;
- AI agents that are starting to build the workflows themselves.

**This is reference learning material.** It's about the general problem,
not any one product. Products appear as examples of a design choice
(Airflow, Temporal, Kestra, n8n, Windmill, KrakenD), and every claim about
one is cited to its own documentation.

Assumes [go-services](../go-services/README.md) files 01, 02, 04 and 05:
how a Go service runs, the HTTP client, deadlines and retries, and fanning
out. Files 03, 06, 08 and 09 are linked where they add depth. Chapters link back
to them rather than re-teaching them.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the fix.
- Code is in Go. Each core example also has Rust, C++ and JavaScript
  versions in collapsibles, and every example is run before it goes in.
- Every chapter ends with **Check yourself** questions. Answers are in
  [11-exercises.md](11-exercises.md).

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
| [OMG BPMN 2.0.2](https://www.omg.org/spec/BPMN/2.0.2/PDF) | The join gateways, and the inclusive gateway's path-based rule for when an OR-join may fire |
| [Workflow Patterns: control-flow](http://www.workflowpatterns.com/patterns/control/) | Named kinds of join (WCP-3, 5, 7, 9, 37, 38) and how engines rate against them |
| [OASIS WS-BPEL 2.0](https://docs.oasis-open.org/wsbpel/2.0/OS/wsbpel-v2.0-OS.html) | Join conditions over link status, and dead-path elimination in an acyclic graph |
| [van der Aalst, Desel and Kindler, 2002](http://www.workflowpatterns.com/documentation/documents/EPK02.pdf) / [Wynn et al., 2005](http://www.workflowpatterns.com/documentation/documents/resetnets.pdf) | Why the OR-join is non-local, the vicious circle, and what a general one costs |
| [Camunda 7.24](https://docs.camunda.org/manual/7.24/reference/bpmn20/gateways/inclusive-gateway/) / [Camunda 8](https://docs.camunda.io/docs/components/modeler/bpmn/inclusive-gateways/): inclusive gateway | Two engines' OR-join rules, one documented as departing from BPMN |
| [JLS SE 21 ch. 16](https://docs.oracle.com/javase/specs/jls/se21/html/jls-16.html) / [§14.22](https://docs.oracle.com/javase/specs/jls/se21/html/jls-14.html#jls-14.22) / [go vet: unreachable](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/unreachable) | Definite assignment and unreachable code: the compiler analyses that a join's may/must analysis copies |
| [Cook, 1971](https://www.cs.toronto.edu/~sacook/homepage/1971.pdf) / [Clay Institute: P vs NP](https://www.claymath.org/millennium/p-vs-np/) | Why an exact analysis of joins is out of reach |
| [Temporal: retry policies](https://docs.temporal.io/encyclopedia/retry-policies) | Retrying an activity; a per-attempt timeout vs a bound on the whole sequence |
| [Kestra: server components](https://kestra.io/docs/architecture/server-components) | The moving parts of a job-style engine |
| [n8n: Respond to Webhook](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.respondtowebhook/) | A workflow that answers the request that started it |
| [KrakenD: sequential proxy](https://www.krakend.io/docs/endpoints/sequential-proxy/) | Request-style composition inside an API gateway |
| [AWS Step Functions: workflow types](https://docs.aws.amazon.com/step-functions/latest/dg/choosing-workflow-type.html) | Standard vs Express: the job-style or request-style choice, made per state machine |
| [WHATWG HTML: server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html) | The event stream format, reconnection, and `Last-Event-ID` |
| [nginx: ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) | Proxy buffering, `X-Accel-Buffering`, and the read timeout a quiet stream has to beat |
| [JSONata](https://docs.jsonata.org) / [jsonata-js](https://github.com/jsonata-js/jsonata) / [gnata](https://github.com/recolabs/gnata) | The expression language, its reference implementation, and a Go one |
| [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) | Calling URLs you don't control |
| [OpenAPI](https://spec.openapis.org/oas/latest.html) / [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) / [openapi-typescript](https://openapi-ts.dev) | One description of an API, generated into both codebases |
| [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html) / [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901.html) / [RFC 6902](https://www.rfc-editor.org/rfc/rfc6902.html) / [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) | Problem details, JSON Pointer, JSON Patch, conditional requests, and which 5xx says whose fault |
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
| 9 | [Joins: a step with more than one way in](09-joins-a-step-with-more-than-one-way-in.md) | Name the kinds of join in BPMN's, the Workflow Patterns' and Airflow's terms, say why an OR-join needs no look-ahead in a DAG, mark a join's required edges, and work out before any run what its mapping may find missing. Read after 2, and §9.4 and §9.6 after 3 |
| 3 | [Mapping data between steps](03-mapping-data-between-steps.md) | Build a request from earlier results with JSONata, put limits on an expression someone else wrote, and explain why preview and run must share an evaluator |
| 4 | [Retries, idempotency and durability](04-retries-idempotency-and-durability.md) | Retry a POST without doing it twice, honour `Retry-After`, and choose how much of a run should survive a crash |
| 8 | [Recording a run, and following it live](08-recording-a-run-and-following-it-live.md) | Record each step without slowing the run, keep secrets and cut-up bodies out of the record, stream a run's progress past proxies that buffer and time out, and say whose fault a failed run was. Read after 4 |

### Part 2 — Living with other people's input

| # | File | After this you can… |
|---|------|---------------------|
| 5 | [Calling URLs you don't control](05-calling-urls-you-dont-control.md) | ⭐ Explain why checking a URL string doesn't stop SSRF, refuse private addresses at connection time in four languages, and keep data placed in a URL from changing what it names |
| 6 | [Describing APIs](06-describing-apis.md) | Generate both sides of an API from one OpenAPI file, return errors that point at the offending field, and stop two editors overwriting each other |

### Part 3 — Agents

| # | File | After this you can… |
|---|------|---------------------|
| 7 | [Agents that build workflows](07-agents-that-build-workflows.md) | Explain the draft, validate, publish pattern, design tools an agent can repair from, and say which risks are solved and which aren't |

### Reference

| # | File | |
|---|------|---|
| 10 | [Glossary](10-glossary.md) | Terms, one line each |
| 11 | [Exercises & answers](11-exercises.md) | Worked answers, things to try, and questions to ask |

Read file 01 first: every other file assumes its vocabulary and its data
document. File 04 also assumes 02. Files 03, 05 and 06 need only 01, so read
them in any order. File 07 leans on most of the others, 06 above all. File 08
was added later and takes the next number, but it builds on 02's scheduler and
04's durability levels, so it's listed in Part 1, after 04. File 09 came later
still and builds on 02 §2.6's rules, so it's listed straight after 02; its
§9.4 and §9.6 also use 03 §3.3 on what a mapping does with a missing step.

## If you're short on time

- **New to workflow engines:** file 01, all of it.
- **Letting users type URLs into your product:** file 05, all of it.
- **A retried step did its side effect twice:** file 04.
- **A step after a branch never runs, or runs without the data it needs:** file 09, §9.3 and §9.4.
- **Building an editor with a live preview:** file 03.
- **A live run view that updates only at the end, or drops after a minute:** file 08, §8.6.
- **Deciding what a run record may store:** file 08, §8.3.
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
That rule is an OR-join, which in general must look ahead along every path
and, with cycles, has no agreed meaning; in a DAG where every edge is
resolved, dead-path elimination turns it into a count. An edge marked
required is a gate for the scheduler and a promise to the join's mapping, and
an analysis built like a compiler's definite assignment tells the author,
before any run, which steps that mapping may find missing. It's sound but
incomplete, and being exact would mean solving satisfiability.

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

Record a run a step at a time, so a crash keeps what was done and a running
run shows progress, but never on the scheduler's goroutine: saving inline
stretched a 576 ms run to 882 ms against a 100 ms database, and a writer
goroutine fed by a channel with room for every record left it at 576. Redact
credentials by name, knowing it misses secrets in bodies and paths, and keep a
large body as a string preview cut where a character starts, because cut JSON
isn't JSON. Failing interrupted runs at start-up is safe only if two processes
never overlap, and Kubernetes's default rolling update overlaps them. Follow a
run with server-sent events: numbered events kept in memory, replayed after
`Last-Event-ID`, a comment to keep a quiet stream alive, and a 204 once the
client has everything. Go's `WriteTimeout` cuts the stream unless the response
clears it; nginx holds it to the end when it can't track the body's length,
unless the response says `X-Accel-Buffering: no`, and closes it after 60 s of
silence. `EventSource` can't send a bearer token and gives up for good on a
502, so a browser app may read the stream with `fetch()`. A slow client should
cost a position in the log, not memory or the run's time. And a request-style
run answers 502, 504 or 500 to say whose fault a failure was.

When users type the URLs, checking the string fails to DNS rebinding,
redirects and alternative IP spellings, which are all time-of-check to
time-of-use gaps. The check belongs at connection time, on the resolved
address, for every hop, in each language's own hook, and some hooks have
holes: reqwest's resolver skips IP literals, and `tls.connect` ignores Node's
`blockList`. When data fills part of a fixed URL, escaping keeps it in place
only until a proxy decodes the path: nginx turned an escaped `..%2Fadmin`
into `/admin`, so a path value that's a dot segment or holds a slash is
refused, not escaped.

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
- "Which steps can this join's mapping rely on, and which may be missing?"
- "Step `crm` failed after posting. What does a retry do now?"
- "Our live run view freezes behind the load balancer. Which layer is holding it, and what do we change?"
- "Here's a step's recorded request and response. What in them could still be a secret?"
- "Is this JSONata expression safe to run on input I don't control?"
- "Can a user point this HTTP client at our cloud metadata endpoint?"
- "Generate both sides of this endpoint from OpenAPI and show me what drifts if I edit one."
- "What tools would an agent need to build this workflow from a written spec?"
