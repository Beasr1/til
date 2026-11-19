# 3. Logs that know their trace

## The problem

You find the slow trace. You want the log lines from that request, the ones that say *why* it
was slow. The logs are in a different tool, written to stdout and scraped, with no field in
common with the trace except a timestamp. You search a window of a few hundred milliseconds
across the service and get forty requests' worth of interleaved lines.

Logs become useful in a distributed system when every log line carries the id of the trace and
span it was written in. Then "show me the logs for this trace" is a filter, not a guess. This
chapter is about getting those ids onto every line, getting the lines to the same place as the
traces, and two smaller traps that come with wrapping a logger.

## What correlation needs

The [OpenTelemetry Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
(status: Stable) gives every log record top-level `TraceId`, `SpanId` and `TraceFlags` fields,
alongside `Timestamp`, `SeverityNumber`, `Body`, `Attributes` and the rest. `TraceId` is the
"Request trace ID as defined in" W3C Trace Context; `SpanId` "Can be set for logs that are part
of a particular processing span", and "If SpanId is present TraceId SHOULD be also present".
Both are optional, which is the problem in one word: nothing forces them to be there.

There are two ways to get them there:

| Approach | How | What the backend can do |
|---|---|---|
| **Stamp ids as fields** | Your logger reads the current span and adds `trace_id` and `span_id` to each line | Search by trace id. Any log tool works |
| **Emit OTel log records** | A **log bridge** turns your logger's records into OTel log records with real `TraceId`/`SpanId`, exported over OTLP | Link logs and spans natively: a "logs" tab on every span |

Both depend on the same thing: at the moment of the log call, the logger must be able to find
the current span. In Go, that means it must be given the context.

## Every log call needs the context

Go's `log/slog` has two forms of every method. Here is what the context-free one does, from
[`log/slog/logger.go`](https://github.com/golang/go/blob/go1.25.0/src/log/slog/logger.go#L208-L215):

```go
func (l *Logger) Info(msg string, args ...any) {
    l.log(context.Background(), LevelInfo, msg, args...)
}

func (l *Logger) InfoContext(ctx context.Context, msg string, args ...any) {
    l.log(ctx, LevelInfo, msg, args...)
}
```

The OpenTelemetry bridge for slog, [`otelslog`](https://pkg.go.dev/go.opentelemetry.io/contrib/bridges/otelslog),
passes that context straight to the OTel logger's `Emit`, and the SDK fills the trace fields from
it: `sc := trace.SpanContextFromContext(ctx)`, then `traceID: sc.TraceID()`, `spanID:
sc.SpanID()` ([`sdk/log/logger.go`](https://github.com/open-telemetry/opentelemetry-go/blob/main/sdk/log/logger.go)).
So `slog.Info` inside a traced handler produces a record with an all-zero trace id. It's exported,
it's in the backend, and it's attached to nothing.

```go
import "go.opentelemetry.io/contrib/bridges/otelslog"

// At startup, after installing a LoggerProvider with an OTLP exporter.
logger := otelslog.NewLogger("orders") // uses the global LoggerProvider by default

func (h *Handler) GetOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    logger.InfoContext(ctx, "loading order", "order_id", id) // carries TraceId and SpanId
    logger.Info("loading order", "order_id", id)            // carries neither
}
```

This is why "add trace correlation" turns into a diff that touches every logging call in the
service: each call site needs a context, and helper functions that log need a `ctx` parameter
they didn't have. A wrapper logger whose methods don't take a context can't correlate at all,
however clever its internals. Make the context-taking form the only one your wrapper offers.

<details><summary>Rust</summary>

`tracing` events inside an instrumented span already know their span. To stamp the OpenTelemetry
ids as fields (useful for plain-text logs), read them through `tracing-opentelemetry`:

```rust
use opentelemetry::trace::TraceContextExt;
use tracing_opentelemetry::OpenTelemetrySpanExt;

let cx = tracing::Span::current().context();
let sc = cx.span().span_context().clone();
tracing::info!(trace_id = %sc.trace_id(), span_id = %sc.span_id(), "loading order");
```

For OTLP log records, the `opentelemetry-appender-tracing` crate bridges `tracing` events to the
OTel logs SDK. Check which version links records to the active span, since this has changed
between releases.

</details>

<details><summary>C++</summary>

```cpp
auto sc = opentelemetry::trace::Tracer::GetCurrentSpan()->GetContext();
char trace_id[32], span_id[16];
sc.trace_id().ToLowerBase16(trace_id);
sc.span_id().ToLowerBase16(span_id);
log_info("loading order", {{"trace_id", std::string(trace_id, 32)},
                           {"span_id", std::string(span_id, 16)}});
```

</details>

<details><summary>JavaScript</summary>

```js
const { trace } = require('@opentelemetry/api');

function logInfo(msg, fields = {}) {
  const sc = trace.getActiveSpan()?.spanContext();
  console.log(JSON.stringify({ level: 'info', msg, trace_id: sc?.traceId, span_id: sc?.spanId, ...fields }));
}
```

</details>

## Ship logs over the same pipeline

The common first attempt keeps logs on stdout and has an agent scrape the container output. It
works for reading logs. It's awkward for correlation, because the scraper sees text: either every
line already has the ids stamped in a parseable field, or there is nothing to join on.

Exporting logs over OTLP, through the same exporter and Collector as traces, means the ids arrive
as structured fields, the resource attributes (`service.name` and friends) match the spans exactly,
and the Collector's processing (chapter 4) applies to both. It's not free:

| | Scrape stdout | Export over OTLP |
|---|---|---|
| Correlation | Only if ids are stamped in the text | Native `TraceId`/`SpanId` |
| Process crashes | Lines already written survive | Records still in the batch queue are lost |
| Collector down | Lines sit in the container log until scraped | Queue fills, then records are dropped |
| Resource attributes | Added by the scraper, may not match the spans | Same resource as the spans |

Many teams do both: OTLP for correlation, and stdout kept for the last words of a crashing process.

## The pipeline must not log into itself

Once logs go through the OTel pipeline, the pipeline's own logs become a hazard. The exporter is a
component like any other. When it fails (the Collector is unreachable, the TLS mismatch from
chapter 1) it logs the failure. If that log goes through the same bridge, it becomes a record to
export, through the exporter that's failing, which logs another failure. A network blip turns into
thousands of lines, and lines emitted while the provider shuts down add their own noise.

The official Rust example for the OTLP exporter carries the warning in its setup code
([`basic-otlp/src/main.rs`](https://github.com/open-telemetry/opentelemetry-rust/blob/main/opentelemetry-otlp/examples/basic-otlp/src/main.rs)):
"To prevent a telemetry-induced-telemetry loop, OpenTelemetry's own internal logging is properly
suppressed. However, logs emitted by external components (such as reqwest, tonic, etc.) are not
suppressed", so it turns off `hyper`, `tonic`, `h2` and `reqwest` for the OTel layer while the
stdout layer still shows them.

The rule: **the telemetry pipeline's own diagnostics go to stdout or stderr, never back into the
pipeline.** In Go, the SDK reports its errors through the handler set by `otel.SetErrorHandler`
and its internal logger set by `otel.SetLogger`; point both at a plain stderr logger, not at the
bridged one. In Rust, filter the exporter's transport crates out of the OTel layer as the example
does.

## Caller annotations through a wrapper

Most services don't call the logging library directly. They call a small internal package that
wraps it, to fix the format, add standard fields, or hide the library choice. That wrapper adds a
stack frame, and loggers that annotate the caller find the wrong one.

Zap's `AddCaller` annotates "each message with the filename, line number, and function name of
zap's caller". Zap's caller is your wrapper. Every line in every service then reports
`logger/logger.go:42`. Zap's answer is in the next function in
[`options.go`](https://github.com/uber-go/zap/blob/v1.27.0/options.go#L104-L112):

```go
// AddCallerSkip increases the number of callers skipped by caller annotation
// (as enabled by the AddCaller option). When building wrappers around the
// Logger and SugaredLogger, supplying this Option prevents zap from always
// reporting the wrapper code as the caller.
func AddCallerSkip(skip int) Option {
    return optionFunc(func(log *Logger) {
        log.callerSkip += skip
    })
}
```

Note the `+=`. Skips add up. If your wrapper appends `AddCallerSkip(1)` and a caller also passes
`AddCallerSkip(1)` because they wrapped it again, the total is 2, which is right for two wrappers
and wrong if the caller was compensating for a skip that was missing. Put the skip in exactly one
place: the constructor of the wrapper that adds the frame.

```go
func New(base *zap.Logger) *Logger {
    return &Logger{z: base.WithOptions(zap.AddCaller(), zap.AddCallerSkip(1))}
}

func (l *Logger) Info(ctx context.Context, msg string, fields ...zap.Field) {
    l.z.Info(msg, append(fields, traceFields(ctx)...)...) // one extra frame: this method
}
```

`log/slog` has the same issue and a different mechanism. Its package docs, under "Wrapping output
methods", say a wrapper "will report the source file as mylog.go, not main.go" unless it obtains
the caller's program counter itself and passes it to `NewRecord`; the package's `wrapping` example
shows how with `runtime.Callers`.

> **Teacher's aside.** Correlation is a property of the *call site*, not of the logging backend.
> No exporter, Collector or vendor feature can attach a trace id to a log line written with a
> context that doesn't contain one. That's why the work is a tedious sweep through every logging
> call rather than a configuration change, and why the cheapest time to do it is before a service
> has a thousand log calls. The design choice that makes it stick is a logger interface whose
> every method takes a context.

## Check yourself

1. A service exports logs through `otelslog`. In the backend, about a third of its log records
   have no trace id, even though every request is traced. What are you going to find when you
   look at the code that wrote those records?
2. Someone proposes adding a background job that periodically joins log lines to spans by matching
   timestamps and service name. Why will it mis-attribute lines under load?
3. The Collector goes down for ten minutes. Afterwards, the log backend shows tens of thousands of
   lines from your services saying an export failed. Trace the loop that produced them, and say
   where you'd cut it.
4. Every log line from every service reports its caller as the shared logging package. One service
   works around it by passing its own `zap.AddCallerSkip(1)` into the package's constructor. Months
   later the shared package is fixed to add `AddCallerSkip(1)` itself, and that one service's lines
   now report a frame inside its HTTP framework. Explain both symptoms.
5. A team decides to keep scraping stdout only, and to stamp `trace_id` on every line. What do they
   gain over OTLP export, and what can they still not do?
