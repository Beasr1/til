# 7. Exercises

Worked answers to every **Check yourself** question, then things to try and questions worth
asking me.

---

## File 01 — What observability is

**1. Instrumentation libraries, no SDK.**

The backend shows nothing from this service. Every API call (start a span, set an attribute, end
it) goes to the no-op implementation, which is the design: libraries can instrument themselves
without forcing an SDK on you. The service behaves exactly as before, with negligible overhead.

One less obvious effect: the global propagator is a separate setting, and if nobody set it either,
it's also a no-op. Then the service doesn't extract incoming `traceparent` headers or inject
outgoing ones, so a trace that passes *through* it breaks there, even if the services on either
side are fully configured.

**2. The span tree.**

```
SERVER    A: handle request
├─ CLIENT    A: database query
└─ CLIENT    A: HTTP call to B
   └─ SERVER    B: handle request
      └─ PRODUCER  B: send message
```

The `CLIENT` span in A and the `SERVER` span in B are two views of the same hop, which is why kinds
exist: the backend pairs them. The consumer that eventually processes the message is a `CONSUMER`
span, parented by or linked to the `PRODUCER` span (chapter 2).

**3. `404` on both sides.**

Only the client span is `Error`. The server answered correctly: the record doesn't exist and it said
so. If server spans counted 4xx as errors, the server's error rate would be driven by its callers'
mistakes and by every probe for a missing record, and an alert on it would fire for things the
server can't fix. The client, though, needed the record and didn't get it, so from its side the
operation failed. The HTTP conventions encode exactly this: 4xx is `Unset` for `SERVER` and `Error`
for `CLIENT`.

**4. `collector:4317`, plaintext Collector, no spans.**

The endpoint has no scheme, so per the OTLP exporter spec the insecure setting decides, and it
defaults to `false`. Go's gRPC exporter adds that "client security is required unless WithInsecure
is used". So the exporter opens a TLS handshake against a server that speaks plaintext gRPC, the
handshake fails, the batch is dropped, and the error goes to the SDK's error handler, which few
people watch. The application never sees any of it.

Fixes, any one of:

- Give the endpoint an explicit scheme: `http://collector:4317`. The spec says `http` "indicates an
  insecure connection and takes precedence over the `insecure` configuration setting". (In Go code,
  use `WithEndpointURL`, which reads the scheme; `WithEndpoint` doesn't take one.)
- Set `OTEL_EXPORTER_OTLP_INSECURE=true`, or `otlptracegrpc.WithInsecure()` in code.
- Enable TLS on the Collector's receiver, so the default is right.

Which is correct depends on the network path, as chapter 1's warning says.

**5. Tracing middleware that returns `500` instead of re-panicking.**

The existing recovery middleware never sees the panic, because it was swallowed first. So the panic
log with its stack trace disappears, the crash counter stops counting, and any alert built on that
counter goes quiet. The response body may change format too, if the old recovery wrote a specific
error shape. The traces look better (the span is marked as an error) while every other signal of a
crash vanishes. A tracing helper must never change how the program behaves; recording and then
re-panicking keeps the existing path intact.

---

## File 02 — Carrying a trace across services and messages

**1. `otelhttp` everywhere, disconnected traces.**

On the client, `otelhttp`'s transport starts a `CLIENT` span and calls the global propagator's
`Inject` to write the span context into the request headers. With no propagator registered, that's
the no-op, so no `traceparent` header is written. On the server, `otelhttp`'s handler calls the
global `Extract`, also a no-op, finds no parent, and starts a root span. Both services record
correct spans in separate traces.

The fix, in both services' startup code:
`otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(propagation.TraceContext{}, propagation.Baggage{}))`.
Both need it: the caller to inject, the callee to extract.

**2. Producer injects correctly, consumer still starts new traces.**

Places the context can be lost between `Inject` and `Extract`:

- **A type conversion on the consumer side.** The broker client's record has headers; the service
  converts it to its own `Message` struct, which has no headers field. Extract then reads an empty
  carrier.
- **Something that republishes.** A retry or dead-letter mechanism, a bridge between clusters, or a
  stream processor builds a new message from the old payload and doesn't copy headers.
- **The consumer's propagator.** The consumer service never registered one, so its `Extract` is the
  no-op even though the header is present.
- **The carrier adapter.** The consumer's adapter looks up the wrong key, or a case-sensitive key
  that some layer changed.

The way to find out which: log or inspect the headers at each step, and check parent span ids in the
backend.

**3. Parse first, span on success.**

The backend shows nothing for the bad message. Successful messages produce spans; the poison pill
produces none, on every retry, so the one thing stalling the partition is invisible. You might see
consumer lag rising in metrics with no trace to explain it.

Move the span start before parsing: extract the parent from the headers, start the `CONSUMER` span,
then parse. On failure, record the error and set the status to `Error` before dead-lettering. Each
attempt is now an error span, attached to the producer's trace (or linked to it), carrying the parse
error.

**4. A batch span with "the producer as its parent".**

A span has exactly one parent, and the batch has 200 producers' creation contexts. Choosing one
misattributes the other 199 and silently drops their correlation. The messaging conventions say the
span "SHOULD link to the message's creation context" for each message it accounts for, and reserve
parenting for "single messages scenarios". So: one span with 200 links. If the colleague wants a
nested waterfall per message, process each message in its own child span and parent that one on its
message's creation context.

**5. `span.enter()` held across awaits.**

The guard keeps the span entered on the current thread until it drops. When the task awaits, it
yields, still holding the guard, and the executor polls other tasks on the same worker thread. Those
tasks find this span current, so their spans and events nest under it. That's the mis-nesting in the
traces. The `tracing` docs say exactly this produces "incorrect traces". Use `.instrument(span)`,
which enters the span only while this future is being polled, or `#[instrument]`.

**6. Trace id derived from the raw bytes of a session token.**

Every service that receives `traceparent`, including any third party you call with instrumented
clients, learns the token's bytes (or most of them). If the token authenticates the session, that's
a leaked credential; at minimum it's a stable identifier that lets them link every request in the
session. W3C Trace Context §6 forbids putting sensitive information in `traceparent`.

Change: derive the trace id from an HMAC of the token with a secret key, so the trace id reveals
nothing; or don't derive it at all, and instead record a non-sensitive flow id as an attribute or
link each request's root span to the first. The costs of keeping the derived id: downstream services
can still tell that requests belong to one session (same trace id); you deviate from the spec's
preference for random trace ids; tools that assume one trace per request show very long traces; and
you now have a key to manage.

---

## File 03 — Logs that know their trace

**1. A third of records have no trace id.**

Log calls made without the request's context. In Go: `logger.Info` instead of `logger.InfoContext`
(which logs with `context.Background()`), helper functions that log but take no `ctx`, goroutines
started with a fresh context, or a wrapper logger whose methods don't accept a context at all. Some
of the third will be legitimate (startup, background jobs with no request). Grep for the
context-free method names and for helpers with a logger but no `ctx` parameter.

**2. Joining logs to spans by timestamp.**

Under load, dozens of requests are in flight in the same service in the same millisecond, so a
timestamp and a service name match many spans. Clocks also differ between where the log was stamped
and where the span was, and log pipelines may record the time they *observed* a line rather than when
it was written. The job will attach lines to the wrong request, or to several, and nobody can tell
which joins are wrong. Only an id written at the moment of the call identifies the request.

**3. Tens of thousands of "export failed" lines after a Collector outage.**

The exporter failed, and logged the failure. That log went through the bridged logger, so it became
a log record queued for export, through the failing exporter, which failed again and logged again.
Each retry and each batch multiplied it, and records emitted during shutdown added more. When the
Collector came back, the queue drained into the backend.

Cut it where the pipeline's diagnostics enter the pipeline: route the SDK's internal errors
(`otel.SetErrorHandler`, `otel.SetLogger` in Go) to a plain stderr logger, and in Rust filter the
exporter's transport crates (`tonic`, `hyper`, `h2`, `reqwest`) out of the OTel layer, as the
official example does.

**4. Caller annotations, before and after.**

Before: `AddCaller` reports "zap's caller", which is the shared package's wrapper method, so every
line points at the shared package. After: the service passed `AddCallerSkip(1)`, and the shared
package now adds its own `AddCallerSkip(1)`. Zap's implementation is `log.callerSkip += skip`, so the
total is 2. Zap skips the wrapper frame *and* the service's own call site, and reports whoever called
that: for a handler, the HTTP framework's code. The fix is to delete the service's local
compensation. Skips belong in exactly one place, the constructor of the wrapper that adds the frame.

**5. Scrape stdout only, with `trace_id` stamped on every line.**

They gain: lines already written survive a crash (nothing waits in an in-process batch queue), no
logs SDK dependency, and search by trace id works in any log tool. They still can't get native
linking (a logs view on each span) unless the backend parses the field into a real trace id; the
resource attributes the scraper adds may not match the spans' `service.name`; and any Collector-side
processing, such as the attribute allowlist in chapter 4, applies only if the scraped logs pass
through the Collector. And they still have to do the sweep: stamping only works where the log call
has the context.

---

## File 04 — What not to record

**1. The national id "so support can find traces".**

Record a surrogate: the transaction or request id, which already identifies the operation. If
support needs to find every operation for one person across transactions, record a keyed hash
(HMAC) of the identifier, never the identifier or an unkeyed hash of it. Support's workflow becomes:
look the customer up in the primary system, which has proper access control, get their transaction
ids (or compute the HMAC through an access-controlled internal tool), and search telemetry by those.
That keeps the national id in one controlled store instead of copying it into a widely readable
30-day one, which is what data minimisation asks for.

**2. Blocklist versus allowlist after a library upgrade.**

The blocklist doesn't know `http.request.header.authorization`, so the authorisation header, a
credential, is exported on every span for six months, readable by everyone with backend access.
OWASP lists access tokens among the data that must not be logged. A per-scope allowlist for the HTTP
instrumentation would never have included that key, so it would have been dropped from the first
span. The cost of the allowlist is the mirror image: a new attribute someone actually wants is
missing until they add it to the list. That failure is visible and harmless.

**3. Why not delete attributes in `OnEnd`.**

In Go, `OnEnd` receives a `ReadOnlySpan`: there are no setters, the span has ended, and the same span
is handed to every registered processor, so mutating it would be visible to all of them even if you
could. Wrapping the exporter lets you build a filtered *view* of each span (embed the
`ReadOnlySpan`, override `Attributes()`) at the moment it leaves the process. The original span is
untouched, other exporters see whatever their own policy allows, and the filter sits exactly at the
boundary you care about.

**4. Hiding the first six characters.**

The hidden part is the four-character issuer code, identical for everyone, and a two-digit year with
only a few dozen plausible values. The shown part is the eight random digits, which are the entire
identifying content. A reader learns the identifier, up to a few dozen guesses. The rule masks the
part with no information and shows the part with all of it. Mask the random portion instead, show at
most a short suffix, and hide at least half the value; or show a surrogate and no part of the
identifier at all.

**5. An error string carrying the id.**

Into the span (as an exception event's message), the tracing backend, the log line and the log
backend, the billing event's payload, the billing service's storage and its logs, and every retry of
each. Field-level redaction covered only the request's `id` field, not copies of its value inside
other strings. Fix it twice: don't put the value in the error (say what's wrong with it, and attach a
surrogate), and redact by value at the sinks, so the copy nobody anticipated is caught too.

**6. `r.Pattern` and tracing middleware.**

The tracing middleware calls `r.WithContext(ctx)` to put the server span in the context, and passes
the *copy* to the mux. The mux sets `Pattern` on the copy it received. The metrics middleware, outside
both, still holds the original request, whose `Pattern` is empty, so every request falls through to
`unmatched`. In the test there was no layer in between, so the mux and the metrics middleware shared
one request. Fix it by placing the metrics middleware inside the tracing layer (or per route), or
have the inner layer report the matched pattern back through a holder in the context.

With `r.URL.Path` as the fallback, every distinct path would become a new series: one per id in the
URL. Prometheus warns against exactly that. Memory and query cost grow without bound, dashboards slow
down, and the ids in the paths are now in the metrics backend too.

---

## File 05 — Audit trails

**1. Every state transition to the audit topic.**

Costs: the audit store is built for low volume and long retention, so you pay years of storage for
data with a useful life of days; the handful of records an auditor needs are buried among thousands;
any personal data in those transitions is now retained for years in a second place; and the audit
pipeline, sized for a few events per transaction, has to carry telemetry volume. Transitions belong
in telemetry, as spans and logs, linked to the audit records by trace id.

**2. Removing "create user" and "change password" from the audit.**

The trail now omits account creation and credential changes, the actions an investigator is most
likely to ask about. They should have replaced "store the whole body" with a per-action allowlist:
"user created" records the new user's id, role, organisation, the actor and the outcome. The password
was never on the list, so there's nothing to strip and nothing to drop.

**3. The test that would have failed.**

A route-coverage test: enumerate every registered route that changes state (`POST`, `PUT`, `PATCH`,
`DELETE`, plus sensitive reads such as document downloads) and assert that each one has an audit
mapping or an explicit exemption with a written reason. New routes without a mapping fail the build,
so a rewrite can't drop the trail silently.

**4. In-memory queue: how events are lost, and a design that can't diverge.**

Lost when: the process dies with events queued (crash, out-of-memory kill, a deploy that doesn't
drain); the queue is full and the enqueue drops; a publish fails and isn't retried, or the broker is
down longer than the retries last; shutdown doesn't wait for the publisher.

The design: an **outbox**. Insert the audit record into an outbox table in the same database
transaction as the action. Both commit or neither does. A relay reads the outbox, publishes with
retries, and marks rows sent. The action and its record can't diverge, and the publish can't be lost,
only delayed. The alternative is a synchronous write that fails the action when the audit write
fails, which is a policy choice with a latency and availability cost.

**5. WARN-level failures and an empty trail.**

In the case that opened the chapter, nothing failed: the publishing code had been deleted. Logging
failures at ERROR, with a counter, catches only calls that are made and fail, so it wouldn't have
fired. Alerting on silence (zero `document.signed` events for hours while signing traffic continues)
catches the absence regardless of its cause. You want both: ERROR plus a counter tells you *which*
event failed; the silence alert tells you that something stopped.

---

## Things to try

- Run two small Go services that call each other through `otelhttp`, exporting to a local Collector
  with a stdout or debug exporter. Don't register a propagator. Confirm two traces per request. Add
  `otel.SetTextMapPropagator`, and check that the server span's parent span id equals the client
  span's span id.
- Write a unit test with `tracetest.NewSpanRecorder` that publishes through your producer wrapper,
  consumes through your consumer wrapper, and asserts the consumer span's `Parent().SpanID()` equals
  the producer span's `SpanContext().SpanID()`. Then delete the headers field from your internal
  message type and watch it fail.
- Feed the consumer a malformed message before and after moving the span start above the parse step.
  Compare what the backend shows.
- In Rust, on a multi-threaded Tokio runtime, hold `span.enter()` across an `.await` in many
  concurrent tasks and look at the nesting. Then switch to `.instrument(span)`.
- Log the same message with `slog.Info` and `slog.InfoContext` through `otelslog` with a stdout log
  exporter, and compare the trace id fields.
- Wrap zap in a one-method logger, log through it with and without `AddCallerSkip(1)`, and read the
  `caller` field each time.
- Put a counter labelled with the raw path behind a load generator that requests `/orders/{random}`,
  and watch the series count on the Prometheus `/metrics` page. Repeat with `r.Pattern`.
- Run a Collector with the redaction processor and an empty `allowed_keys`, and confirm that every
  attribute disappears. That's what "fails closed" looks like.

## Questions worth asking me

- "Here's our tracing setup code. Will context cross service and message boundaries, and how would
  we prove it?"
- "Where in this service could a goroutine or task start an orphan trace?"
- "Should our consumer spans be parented or linked, given how we batch?"
- "Which of our log calls can't be correlated, and what would the sweep look like?"
- "What attributes do our instrumentation libraries emit by default? Draft a per-scope allowlist."
- "Which identifiers appear in our telemetry, and what would a surrogate look like for each?"
- "Which of our metrics have a label with unbounded values?"
- "Which events belong in our audit trail, and which are really telemetry?"
- "If our audit publisher stopped tomorrow, how long until someone noticed?"
