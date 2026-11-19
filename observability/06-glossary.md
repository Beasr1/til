# 6. Glossary

Reference, not reading. Each term links to the chapter that introduces it.

| Term | Meaning | Chapter |
|---|---|---|
| **API (OpenTelemetry)** | The interfaces instrumentation calls. A no-op until an SDK is installed | [1](01-what-observability-is.md) |
| **Allowlist (attributes)** | The set of attribute keys allowed to leave the process or Collector; everything else is dropped. Fails closed | [4](04-what-not-to-record.md) |
| **Attribute** | A key-value pair on a span, log record, metric point or resource | [1](01-what-observability-is.md) |
| **Audit trail** | Append-only record of who did what, to what, when, and with what outcome. Not telemetry | [5](05-audit-trails.md) |
| **Baggage** | Key-value pairs propagated alongside the trace context to every downstream service | [2](02-carrying-a-trace-across-services.md), [4](04-what-not-to-record.md) |
| **Cardinality** | The number of distinct values (or label combinations) a metric has. Each combination is a separate time series | [4](04-what-not-to-record.md) |
| **Carrier** | The thing a propagator writes into and reads from: HTTP headers, message headers, a map | [2](02-carrying-a-trace-across-services.md) |
| **Collector** | A standalone OpenTelemetry process that receives, processes and forwards telemetry | [1](01-what-observability-is.md) |
| **Context** | What holds the current span inside a process: `context.Context` in Go, an implicit per-task or per-thread context elsewhere | [1](01-what-observability-is.md) |
| **Creation context** | The span context a producer attaches to a message, in the messaging semantic conventions | [2](02-carrying-a-trace-across-services.md) |
| **Exporter** | SDK component that serialises and sends telemetry, usually over OTLP | [1](01-what-observability-is.md) |
| **Extract** | Read span context from a carrier into a context. Never throws; an unparseable value leaves no parent | [2](02-carrying-a-trace-across-services.md) |
| **Inject** | Write the current span context into a carrier | [2](02-carrying-a-trace-across-services.md) |
| **Instrumentation scope** | Name (and version) of the library that created a span. The natural key for per-library attribute policy | [1](01-what-observability-is.md), [4](04-what-not-to-record.md) |
| **Keyed hash (HMAC)** | A hash computed with a secret key, so a small identifier space can't be reversed by hashing every candidate | [4](04-what-not-to-record.md) |
| **Link** | A reference from one span to another span that isn't its parent, possibly in another trace | [2](02-carrying-a-trace-across-services.md) |
| **Log bridge** | An adapter that turns an existing logger's records into OpenTelemetry log records | [3](03-logs-that-know-their-trace.md) |
| **OTLP** | OpenTelemetry Protocol, over gRPC (default port 4317) or HTTP (default port 4318) | [1](01-what-observability-is.md) |
| **Outbox** | Writing an event in the same database transaction as the change it describes, then publishing it asynchronously | [5](05-audit-trails.md) |
| **Propagator** | Code that injects and extracts span context (and baggage) in a given format. The global one is a no-op by default | [2](02-carrying-a-trace-across-services.md) |
| **Remote span context** | A span context that arrived from another process | [1](01-what-observability-is.md), [2](02-carrying-a-trace-across-services.md) |
| **Resource** | Attributes describing the process emitting telemetry, such as `service.name` | [1](01-what-observability-is.md) |
| **Route template** | The pattern a request matched (`/orders/{id}`), as opposed to its concrete path. `http.route` in the conventions | [4](04-what-not-to-record.md) |
| **SDK** | The implementation behind the API: real spans, sampling, processors, exporters | [1](01-what-observability-is.md) |
| **Semantic conventions** | OpenTelemetry's agreed names and meanings for attributes, span names and statuses | [1](01-what-observability-is.md) |
| **Span** | One timed operation, with a name, kind, status, attributes and events | [1](01-what-observability-is.md) |
| **Span context** | The part of a span that propagates: trace id, span id, trace flags, trace state | [1](01-what-observability-is.md) |
| **Span kind** | `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER` or `INTERNAL` | [1](01-what-observability-is.md) |
| **Span processor** | SDK hook called on span start and end; the batch processor queues spans for export | [1](01-what-observability-is.md) |
| **Span status** | `Unset`, `Ok` or `Error` | [1](01-what-observability-is.md) |
| **Surrogate** | An identifier that stands for a person or record without revealing it, such as a transaction id | [4](04-what-not-to-record.md) |
| **Telemetry** | Data a system emits about its own behaviour: metrics, logs, traces | [1](01-what-observability-is.md) |
| **Trace** | All spans sharing one trace id; a tree of operations | [1](01-what-observability-is.md) |
| **Trace flags** | One byte in `traceparent`; bit 0 is "sampled" | [2](02-carrying-a-trace-across-services.md) |
| **`traceparent`** | The W3C header carrying version, trace id, parent span id and trace flags | [2](02-carrying-a-trace-across-services.md) |
| **`tracestate`** | The W3C header carrying vendor-specific trace data | [2](02-carrying-a-trace-across-services.md) |
