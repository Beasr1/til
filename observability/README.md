# Observability — A Course

A course on making a distributed system explain itself, and on the ways that quietly fails:
traces that stop at every service or queue, logs that can't be joined to the trace they belong
to, telemetry that leaks personal data or costs a fortune in metric series, an exporter that
can't reach its Collector, and an audit trail that went empty months before anyone looked.

**This is reference learning material.** It teaches OpenTelemetry and the practices around it in
general terms, checked against the W3C Trace Context Recommendation, the OpenTelemetry
specification and semantic conventions, the OpenTelemetry Go, Rust, C++ and JavaScript sources,
and the OWASP and Prometheus guidance. Where a convention is still marked Development, or where
common practice and the conventions disagree, the chapter says so. Code is in Go, with Rust, C++
and JavaScript versions in collapsible blocks.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the fix.
- Every chapter ends with **Check yourself** questions. Answers are in
  [07-exercises.md](07-exercises.md).

Related, not required: [`go-services/`](../go-services/README.md) for the HTTP client, server and
context behaviour the Go examples rely on, and [`kafka/`](../kafka/README.md) for how topics,
headers and consumers work.

## The one thing to understand first

> **Instrumentation is a second program running alongside yours, and its defaults do nothing.**
> The propagator is a no-op until you register one. An exporter given an endpoint with no scheme
> tries TLS. A log call without a context carries no trace id. An attribute an instrumentation
> library records by default goes to the backend unless something removes it. None of these
> produce an error in the application. So every claim about telemetry, "the trace is connected",
> "logs are correlated", "no personal data is exported", gets verified in the backend, not by
> reading the code.

## Reference implementations

| Source | What it settles |
|---|---|
| [W3C Trace Context](https://www.w3.org/TR/trace-context/) (Recommendation, 23 Nov 2021) | `traceparent` and `tracestate` format, invalid values, trace-id randomness, privacy rules |
| [OTel propagators API](https://opentelemetry.io/docs/specs/otel/context/api-propagators/) | Inject and Extract, and that the API's global propagator is a no-op by default |
| [OTel Go](https://pkg.go.dev/go.opentelemetry.io/otel) / [`otel/trace`](https://pkg.go.dev/go.opentelemetry.io/otel/trace) / [`autoprop`](https://pkg.go.dev/go.opentelemetry.io/contrib/propagators/autoprop) | `SetTextMapPropagator`, `ContextWithRemoteSpanContext`, links, the no-op default in Go |
| [`otelhttp`](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp) | Go HTTP client and server instrumentation; uses the global propagator unless told otherwise |
| [OTel messaging spans](https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/) | Creation context, producer and consumer span kinds, parent versus link |
| [`tracing` `Span`](https://docs.rs/tracing/latest/tracing/struct.Span.html) | Why an entered guard across `.await` gives wrong traces, and `Instrument` |
| [OTel Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/) | `TraceId`, `SpanId` and the other log record fields |
| [`otelslog`](https://pkg.go.dev/go.opentelemetry.io/contrib/bridges/otelslog) and [`log/slog`](https://pkg.go.dev/log/slog) | The slog bridge; context-free versus `...Context` methods; wrapping and source locations |
| [zap `options.go`](https://github.com/uber-go/zap/blob/v1.27.0/options.go) | `AddCaller`, and `AddCallerSkip` being additive |
| [OTLP exporter configuration](https://opentelemetry.io/docs/specs/otel/protocol/exporter/) | Endpoint scheme versus the insecure flag, default ports and protocol |
| [OTel HTTP spans](https://opentelemetry.io/docs/specs/semconv/http/http-spans/) | Span status by status code and side, `http.route`, URL redaction rules |
| [OTel database spans](https://opentelemetry.io/docs/specs/semconv/database/database-spans/) | When query text should and shouldn't be recorded |
| [Collector redaction processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/redactionprocessor/README.md) | A fail-closed attribute allowlist in the Collector |
| [Prometheus naming](https://prometheus.io/docs/practices/naming/#labels) | Every label combination is a series; no unbounded label values |
| [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) | Data to exclude, "when, where, who and what", audit trails kept separate, detecting that logging stopped |
| [OWASP ASVS 5.0 V16](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x25-V16-Security-Logging-and-Error-Handling.md) | Security logging requirements; sensitive data in logs; log protection (V7 in ASVS 4.0) |
| [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/) | Data minimisation and storage limitation |
| [NIST SP 800-92](https://csrc.nist.gov/pubs/sp/800/92/final) (2006) | The long-standing guide to log management |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What observability is, and how OpenTelemetry models it](01-what-observability-is.md) | ⭐ Explain metrics, logs and traces; spans, span context and kinds; the API, SDK, exporter and Collector; configure OTLP TLS to match the receiver; and set span status correctly for HTTP, including panics. **Prerequisite for everything else** |

### Part 1 — Connecting the signals

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Carrying a trace across services and messages](02-carrying-a-trace-across-services.md) | ⭐ Read a `traceparent`, register a propagator, inject and extract in transport wrappers, trace a consumer so malformed messages still show up, choose between parent and link, keep context across goroutines and async tasks, and judge a derived trace id for a long flow |
| 3 | [Logs that know their trace](03-logs-that-know-their-trace.md) | Put trace and span ids on every log record, ship logs over OTLP, keep the pipeline from logging into itself, and fix caller annotations through a wrapper |

### Part 2 — Governing what you record

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [What not to record](04-what-not-to-record.md) | ⭐ Find where personal data enters telemetry, enforce a per-scope attribute allowlist in the SDK or Collector, replace identifiers with surrogates, redact by value, gate diagnostics, and keep metric labels bounded with route templates |
| 5 | [Audit trails](05-audit-trails.md) | Separate audit from telemetry, choose the few events that belong, lift audit out of handlers, redact by allowlist, decide between best effort and an outbox, and alert when the trail goes silent |

### Reference

| # | File | |
|---|------|---|
| 6 | [Glossary](06-glossary.md) | Terms, each linked to the chapter that introduces it |
| 7 | [Exercises & answers](07-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. Chapter 2 builds on its span context and propagation vocabulary, and chapter
3 assumes chapter 2's point that context must reach every call. Chapters 4 and 5 can be read in
either order after chapter 1; chapter 5 reuses chapter 4's allowlist idea and its route-template
section. Not yet written, and belonging here: sampling (head, tail, and what parent-based sampling
does to a trace), metrics instruments and histograms, and running the Collector.

## If you're short on time

- **New to tracing or OpenTelemetry:** chapter 1, all of it.
- **Every service is instrumented but traces stop at each boundary:** chapter 2, "Propagators: the
  piece that's off by default", then "Inject and extract inside the transport wrapper".
- **Traces stop at a queue, or poison messages are invisible:** chapter 2, "Messages: producer and
  consumer spans".
- **The application runs but no telemetry arrives:** chapter 1, "Getting telemetry out: the
  exporter".
- **You can't get from a trace to its logs:** chapter 3, "Every log call needs the context".
- **Reviewing telemetry for privacy, or the metrics bill jumped:** chapter 4.
- **Building or reviewing an audit log:** chapter 5.

## The one-paragraph summary of everything

Telemetry comes as metrics (aggregates keyed by labels), logs (events) and traces (trees of timed
spans sharing a trace id), and you need all three plus a way to move between them. A span's
context, its trace id, span id and flags, is what travels: inside a process in a context, across
processes in the W3C `traceparent` header. OpenTelemetry's API is a no-op until an SDK is installed,
and its global propagator is a no-op until you register one, so services wrapped in HTTP
instrumentation still produce disconnected traces until `SetTextMapPropagator` is called. Inject
and extract inside the transport wrappers (HTTP client, producer, consumer), not at call sites, and
make sure every internal message type carries headers. Open a consumer span before deserialising, so
a malformed message is an errored span attached to its producer. The messaging conventions default
to links from consumer to producer and allow parenting only for single messages; batches need links.
In async Rust, attach spans with `.instrument`, never an entered guard across `.await`; in Go, give
background goroutines `context.WithoutCancel(ctx)` rather than `context.Background()`. A trace id
derived from a business flow's id can join a long flow into one trace, but W3C prefers random ids and
forbids sensitive data in `traceparent`, so derive it with a keyed hash if at all, and verify every
"connected" claim by checking parent span ids in the backend. Logs correlate only if each record
carries the trace and span id, which in Go means every log call takes the request's context:
`slog.Info` logs with `context.Background()`. Ship logs over the same OTLP pipeline, keep the
pipeline's own errors out of it, and skip wrapper frames exactly once, since zap's `AddCallerSkip`
adds up. OTLP over gRPC chooses TLS from the endpoint's scheme, or from an insecure flag that
defaults to false when there's no scheme; match the receiver deliberately. HTTP server spans are
errors only on 5xx, client spans on 4xx and 5xx, and tracing middleware records a panic and
re-panics. Telemetry is a data store: instrumentation records client addresses, user agents and
query text by default, so enforce a fail-closed, per-scope attribute allowlist in the SDK or the
Collector, record surrogates or keyed hashes instead of identifiers, mask the high-entropy part,
redact by value wherever data is copied, and gate diagnostics by level instead of committing debug
logs. Metric labels must be bounded: label by route template, never by raw path. An audit trail is
not telemetry: keep it to a few "who did what" events, enforce coverage outside the handlers, redact
by allowlist instead of skipping sensitive actions, decide between best effort and an outbox on
purpose, and alert on silence, because a deleted publisher produces no errors.

## How to use me

Ask me things like:

- "Here's our tracing bootstrap. What's missing for traces to cross services and topics?"
- "Trace this request through our services. Where would the context be dropped?"
- "Should this consumer parent or link its spans?"
- "Why don't these log lines have trace ids?"
- "Draft an attribute allowlist for these instrumentation libraries."
- "Which of our metric labels are unbounded?"
- "Review this audit middleware: coverage, redaction, and what happens when publishing fails."
