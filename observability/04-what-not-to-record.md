# 4. What not to record

## The problem

The primary database that holds customer records has access controls, an owner, a retention
policy and probably an auditor. The tracing backend that holds a copy of half those records, in
span attributes and log lines, usually has none of that: every engineer can search it, it's kept
for weeks, and it may be a third-party service. Nobody decided to put personal data there. An
instrumentation library recorded the client's IP and user agent by default, someone logged a
request body to debug a failure, and an error message embedded the national id number it couldn't
validate.

The same libraries that leak data also blow up cost. A metric labelled with a raw URL path creates
a new time series for every id in every URL, until the dashboard times out or the bill arrives.

Both problems have the same root: **telemetry is a data store**, and it needs the same decisions
about what goes in as any other.

## Telemetry is a data store

The security guidance is consistent on this. The [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
lists data that "should usually not be recorded directly in the logs, but instead should be
removed, masked, sanitized, hashed, or encrypted", including:

| Category | Examples |
|---|---|
| Credentials and secrets | Authentication passwords, access tokens, database connection strings, encryption keys |
| Session handles | Session identification values ("consider replacing with a hashed value if needed to track session specific events") |
| Personal data | "Sensitive personal data and some forms of personally identifiable information (PII)" |
| Financial data | Bank account or payment card holder data |
| Data above the log store's classification | "Data of a higher security classification than the logging system is allowed to store" |
| Data you may not hold | Information illegal to collect, or that the user opted out of |

[OWASP ASVS 5.0](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x25-V16-Security-Logging-and-Error-Handling.md)
(chapter V16) explains why: if sensitive data is logged "without consideration, the logs themselves
become classified and therefore subject to encryption requirements, stricter retention policies,
and potential disclosure during audits." Requirement 16.2.5 allows some data "only ... by being
hashed or masked, either in full or partially." (In the older ASVS 4.0 this material was chapter
V7; references to "ASVS V7" are to that version.)

And where GDPR applies, [Article 5(1)](https://gdpr-info.eu/art-5-gdpr/) requires personal data to
be "adequate, relevant and limited to what is necessary in relation to the purposes for which they
are processed" (data minimisation, 5(1)(c)) and "kept in a form which permits identification of
data subjects for no longer than is necessary" (storage limitation, 5(1)(e)). A span attribute
holding a national id number, retained for 30 days in a backend every engineer can query, is hard
to justify against either.

Everything said here about logs applies to span attributes too. They're the same data in a
different shape.

## Where sensitive data gets in

| Source | Example | What the conventions say |
|---|---|---|
| HTTP instrumentation defaults | `client.address`, `user_agent.original` on server spans | Both "Recommended" in the [HTTP semantic conventions](https://opentelemetry.io/docs/specs/semconv/http/http-spans/), so most instrumentations record them |
| URLs | Query strings with tokens; ids in paths | `url.full` "MUST NOT contain credentials"; values for a short list of signature keys (`X-Amz-Signature`, `sig`, …) SHOULD be replaced with `REDACTED`; "Sensitive content provided in `url.path` SHOULD be scrubbed when instrumentations can identify it" |
| Database instrumentation | `db.query.text` | [Recommended](https://opentelemetry.io/docs/specs/semconv/database/database-spans/), but "Non-parameterized query text SHOULD NOT be collected by default" unless literals are redacted. Many libraries also offer to record connection details |
| Your own attributes | `user.id`, a national id, an email, a document number | Your decision entirely |
| Log lines | A request or response body; the first 100 bytes of an uploaded document | Your decision entirely |
| Error messages | `fmt.Errorf("invalid id %s", id)`, later recorded on the span and logged | Nobody reviews error strings as data |
| Baggage | A user id put in baggage "for correlation" | Baggage is propagated in headers to every downstream service, third parties included |

Note what the conventions do and don't do. They redact a short, fixed list of well-known signature
parameters. They can't know that your `?ref=` parameter carries a customer reference, or that path
segment three is a passport number. That's yours to enforce.

## Enforce an allowlist in the export pipeline

You could try to fix each source: configure each instrumentation library, review each log call. You
should, but it isn't enough on its own, because the next library upgrade adds an attribute and the
next engineer adds a log line. The control that survives change is an **allowlist in the export
path**: every span leaving the process (or the Collector) keeps only the attributes on a list, and
everything else is dropped.

Two design points make it work:

- **Allowlist, not blocklist.** A blocklist fails open: the attribute nobody thought of goes through.
  An allowlist fails closed: the new attribute is dropped until someone decides it's safe, which is
  a visible, cheap failure.
- **Key the list by instrumentation scope.** Every span records the library that created it (chapter
  1). The HTTP server instrumentation, the database instrumentation and your own business spans emit
  completely different attributes; one global list would have to be the union of all of them. Per
  scope, the HTTP list can say `http.request.method`, `http.route`, `http.response.status_code` and
  nothing else.

You can't do this in a span processor's `OnEnd`, because in Go it receives a `ReadOnlySpan`
([`span_processor.go`](https://github.com/open-telemetry/opentelemetry-go/blob/main/sdk/trace/span_processor.go)).
Wrap the exporter instead, and hand it filtered views of each span:

```go
type allowlistExporter struct {
    next  sdktrace.SpanExporter
    allow map[string]map[attribute.Key]bool // scope name -> allowed keys
}

type filteredSpan struct {
    sdktrace.ReadOnlySpan // embedding the interface keeps every other method
    attrs []attribute.KeyValue
}

func (f filteredSpan) Attributes() []attribute.KeyValue { return f.attrs }

func (e *allowlistExporter) ExportSpans(ctx context.Context, spans []sdktrace.ReadOnlySpan) error {
    out := make([]sdktrace.ReadOnlySpan, len(spans))
    for i, s := range spans {
        allowed := e.allow[s.InstrumentationScope().Name] // unknown scope: nil map, nothing allowed
        var kept []attribute.KeyValue
        for _, kv := range s.Attributes() {
            if allowed[kv.Key] {
                kept = append(kept, kv)
            }
        }
        out[i] = filteredSpan{ReadOnlySpan: s, attrs: kept}
    }
    return e.next.ExportSpans(ctx, out)
}

func (e *allowlistExporter) Shutdown(ctx context.Context) error { return e.next.Shutdown(ctx) }
```

A complete version also filters span events (an exception event carries the error message) and
resource attributes, and fails at startup if two registered lists claim the same scope.

The alternative, or the second layer, is the Collector. Its contrib
[redaction processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/redactionprocessor/README.md)
"deletes span, log, and metric datapoint attributes that don't match a list of allowed attributes",
and "The list is designed to fail closed. If allowed_keys is empty, no attributes are allowed". It
can also mask values matching regular expressions (`blocked_values`). Its stability is beta for
traces and alpha for logs and metrics.

```yaml
processors:
  redaction:
    allow_all_keys: false
    allowed_keys: [http.request.method, http.route, http.response.status_code, db.system.name]
    blocked_values: ["\\b\\d{13,16}\\b"] # example: mask long digit runs, such as card numbers
```

| | In the SDK (exporter wrapper) | In the Collector |
|---|---|---|
| Data leaves the process | Already filtered | Unfiltered, as far as the Collector |
| Who owns the policy | Each service, or a shared library | One central config |
| Per-scope lists | Easy: the span carries its scope | Possible with conditions, less direct |
| A service that skips the library | Leaks | Still filtered |

Doing both is reasonable: the SDK filter so data never leaves the process, the Collector so a
service that forgot the shared library is still covered.

## Identifiers: surrogates and masking

You still need to find "the trace for this customer's failed request". The fix is to record
something that identifies the *transaction* without being the *person*:

- **A stable surrogate.** Your own transaction or request id, which means nothing outside your
  system. Support looks up the customer in the primary database, gets the transaction id, and
  searches telemetry by that.
- **A keyed hash** when you need to correlate by the identifier itself across events. Use HMAC with
  a secret key, not a bare hash: identifiers like phone or national id numbers come from a space
  small enough to hash every possible value, so an unkeyed hash is reversible by enumeration.
- **Masking**, for humans who need to recognise a value. Mask the part that carries the information.
  Many identifiers start with a constant or low-entropy prefix (a country code, a year of birth, a
  bank code), so masking the first N characters hides almost nothing. Keep at most a short suffix,
  hide at least half the value, and hide more as the value gets shorter.

```go
func Surrogate(key []byte, id string) string {
    mac := hmac.New(sha256.New, key)
    mac.Write([]byte(id))
    return hex.EncodeToString(mac.Sum(nil)[:12]) // 96 bits: plenty to correlate, nothing to read
}
```

<details><summary>Rust</summary>

```rust
use hmac::{Hmac, KeyInit, Mac}; // KeyInit provides new_from_slice in current hmac releases
use sha2::Sha256;

fn surrogate(key: &[u8], id: &str) -> String {
    let mut mac = Hmac::<Sha256>::new_from_slice(key).expect("HMAC takes any key length");
    mac.update(id.as_bytes());
    hex::encode(&mac.finalize().into_bytes()[..12])
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <openssl/evp.h>
#include <openssl/hmac.h>

std::string Surrogate(const std::string& key, const std::string& id) {
  unsigned char out[EVP_MAX_MD_SIZE];
  unsigned int len = 0;
  HMAC(EVP_sha256(), key.data(), static_cast<int>(key.size()),
       reinterpret_cast<const unsigned char*>(id.data()), id.size(), out, &len);
  static const char* hex = "0123456789abcdef";
  std::string s;
  for (int i = 0; i < 12; ++i) { s += hex[out[i] >> 4]; s += hex[out[i] & 0xf]; }
  return s;
}
```

</details>

<details><summary>JavaScript</summary>

```js
const { createHmac } = require('node:crypto');

function surrogate(key, id) {
  return createHmac('sha256', key).update(id).digest('hex').slice(0, 24);
}
```

</details>

## Redact by value, everywhere it's copied

Field-level redaction ("mask the `nationalId` field") catches the field. It misses every copy: the
error message that quotes the value, the nested JSON payload logged on failure, the note attached to
an event sent to another service, the downstream request body. Data spreads by being copied, so
redaction has to follow the value, not the field name.

The robust pattern is **redaction by value**: when a sensitive value enters the system (a request
field marked sensitive, a secret loaded from config), register it with the request's redactor, and
run every outgoing log line, span attribute, error string and event payload through it, replacing
any occurrence. It's more work than a field mask, and it's the only version that catches the copy
nobody anticipated. The same reasoning applies across services: a note you attach to an event sent
to another team's ledger is now in *their* logs and *their* retention, under *their* access rules.

## Diagnostics: gate them, don't commit them

A bug that only reproduces in a shared environment tempts people into "add logging, merge, deploy,
look, revert". Each round puts throwaway code through review and deployment, and the logging added
under pressure is exactly the kind that dumps payloads. A typical example logs the first 100
characters of an uploaded document with `s[:100]`, which leaks document content and, on a shorter
string, panics: the [Go spec](https://go.dev/ref/spec#Slice_expressions) says "If the indices are out
of range at run time, a run-time panic occurs."

Build the diagnostics in instead, behind a level you can change at run time, and make them record
shapes and sizes rather than contents:

```go
var logLevel = new(slog.LevelVar) // "The zero LevelVar corresponds to LevelInfo"
var logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: logLevel}))

// An admin endpoint or flag calls logLevel.Set(slog.LevelDebug) while you investigate.

logger.DebugContext(ctx, "document received",
    "bytes", len(doc), "content_type", ct, "pages", pages) // what it is, not what it says
```

## Metric labels: bounded by route templates

Prometheus's [naming guide](https://prometheus.io/docs/practices/naming/#labels) states the cost
model: "every unique combination of key-value label pairs represents a new time series", and "Do not
use labels to store dimensions with high cardinality (many different label values), such as user
IDs, email addresses, or other unbounded sets of values."

A `path` label holding `/orders/8f3a…/items/12` is an unbounded set: one series per order per item.
The bounded version is the **route template**, `/orders/{id}/items/{item}`. The HTTP semantic
conventions define exactly this as `http.route`: it "MUST be low-cardinality and include all static
path segments", and "Instrumentation MUST NOT default to using URI path as a `{target}`" in span
names.

On the server side the router already knows the template. Since Go 1.23, `http.Request.Pattern` is
"the [ServeMux] pattern that matched the request" ([`request.go`](https://github.com/golang/go/blob/go1.25.0/src/net/http/request.go#L325-L327)):

```go
func countRequests(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        next.ServeHTTP(w, r)
        route := r.Pattern // e.g. "GET /orders/{id}"; empty if no pattern matched
        if route == "" {
            route = "unmatched" // never fall back to r.URL.Path
        }
        requests.WithLabelValues(route).Inc()
    })
}
```

It reads `Pattern` after the handler returns because the mux sets it during routing, by assigning
to the request it was given: `h, r.Pattern, r.pat, r.matches = mux.findHandler(r)`
([`server.go`](https://github.com/golang/go/blob/go1.25.0/src/net/http/server.go#L2847-L2862)).
That's fine while this middleware and the mux share one `*http.Request`. But any middleware in
between that replaces the request, with `r.WithContext(ctx)` (tracing middleware does exactly this,
to put the server span in the context), hands the mux a copy. The mux sets `Pattern` on the copy, and
this middleware reads an empty string on every request.

Outbound calls are harder: your HTTP client doesn't know the remote service's routes. The usual fix
is a path normaliser in the instrumented client transport, an **ordered** list of rewrite rules that
turn variable segments into the same placeholders the API documentation uses. Two properties matter.
Rules are order-dependent: a specific rule (`/documents/{type}/{number}`) must run before a general
one that would otherwise eat its segments. And every rule must be **idempotent**: applied to an
already-normalised path, it must leave it alone, or a path that passes through two layers gets
mangled. Test the rule list with a table of real paths and expected templates.

<details><summary>Rust</summary>

With `axum`, the `MatchedPath` extractor holds the route template.

```rust
use axum::{extract::MatchedPath, http::Request, middleware::Next, response::Response};

async fn count_requests(req: Request<axum::body::Body>, next: Next) -> Response {
    let route = req.extensions().get::<MatchedPath>()
        .map(|p| p.as_str().to_owned())
        .unwrap_or_else(|| "unmatched".into());
    let res = next.run(req).await;
    metrics::counter!("http_requests_total", "route" => route).increment(1);
    res
}
```

</details>

<details><summary>C++</summary>

C++ HTTP frameworks vary; the principle is to label with the route string you registered, never the
request path.

```cpp
server.route("/orders/{id}", [&](const Request& req) {
  requests_total.Add({{"route", "/orders/{id}"}}).Increment(); // the template, not req.path
  return HandleOrder(req);
});
```

</details>

<details><summary>JavaScript</summary>

In Express, `req.route.path` is the matched template, available after routing.

```js
app.use((req, res, next) => {
  res.on('finish', () => {
    const route = req.route ? req.baseUrl + req.route.path : 'unmatched';
    requestsTotal.inc({ route });
  });
  next();
});
```

</details>

> **Teacher's aside.** People think of telemetry as exhaust: whatever the system gives off, and the
> more the better. Treat it as a product with a schema instead. Every attribute and label is a
> column someone will query, pay to store and have to justify keeping. The default answer to "should
> we record this?" is no, and an allowlist is just that default written down where the code can
> enforce it.

## Check yourself

1. A service's tracing backend is readable by all engineers and keeps 30 days. A product manager asks
   for the customer's national id on every span "so support can find traces". What do you record
   instead, and what does support's workflow become?
2. You add a blocklist to the exporter that drops `client.address` and `user_agent.original`. Six
   months later a library upgrade starts recording `http.request.header.authorization`. What happens
   with the blocklist, and what would have happened with a per-scope allowlist?
3. Why can't you implement the allowlist as a span processor that deletes attributes in `OnEnd`, and
   what does wrapping the exporter do differently?
4. A customer identifier is 14 characters: a four-character issuer code that's the same for every
   customer, a two-digit registration year, then eight random digits. A masking rule hides the first
   six characters and shows the rest. What does a reader learn, and what would you change?
5. Your validation code returns `fmt.Errorf("invalid id %q", id)`. The handler records the error on
   the span, logs it, and includes it in an event sent to a billing service. Field-level redaction on
   the request's `id` field is enabled. Where does the id end up?
6. A metrics middleware labels a request counter with `r.Pattern`. It worked in a test with just the
   mux. In production, with tracing middleware added between it and the mux, every request is counted
   as `unmatched`. What changed, and what would have happened if the fallback had been `r.URL.Path`?
