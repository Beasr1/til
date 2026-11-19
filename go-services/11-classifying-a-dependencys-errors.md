# 11. Classifying a dependency's errors

## The problem

Some requests can't go ahead until another service says yes: a credit check before a billable
operation, a fraud score before a payment, a quota check before an upload. Sometimes that service
says no. Sometimes it doesn't answer at all. Code that handles both in one `if err != nil` branch
has to give them the same answer, and whichever answer it picks is wrong for one of them.

It shows up in three ordinary ways:

- A client helper logs any failure and returns `nil, nil`. The gate sees "no error" and lets
  everyone through, including the accounts the dependency was refusing.
- A dependency reports a business outcome, "not found" or "over the limit", with a 500. The caller
  treats it as an outage: it pages someone, or lets the request through, or bills it.
- A circuit breaker decides what counts as an outage by searching error messages for words like
  "unavailable", and trips because the caller's own wrapping added the word.

This chapter is about classifying a dependency's failures in one place, and about the Go error
handling that makes the classification survive wrapping. It assumes
[chapter 2](02-the-http-client.md) (response bodies) and [chapter 4](04-deadlines-and-retries.md)
(retry only what can succeed). Outputs are from Go 1.24.4.

## Fail open, fail closed

When a gate can't get an answer from the dependency, it has to do something with the request.

| Choice | What happens | What it costs |
|---|---|---|
| **Fail closed** | Refuse the request | Availability: a dependency outage becomes your outage |
| **Fail open** | Let it through without the check | Whatever the check protected: unbilled usage, an unscreened payment |

Neither is right in general. An authorisation check fails closed, because letting strangers in
during an outage is worse than refusing everyone. A usage meter often fails open, because refusing
every customer during a billing outage costs more than a few minutes of unbilled use. It's a product
decision, made per gate, and it should be written down where the gate is.

The choice only exists when there's **no answer**. When the dependency answered "no", there's
nothing to choose: the rejection is the answer. A gate that fails open on a rejection isn't being
resilient. It's ignoring the dependency.

Fail open is a fallback, and fallbacks have a known weakness. Jacob Gabrielson, in the AWS
Builders' Library's
[Avoiding fallback in distributed systems](https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/):
"fallback logic is hard to test" and "It is easy to develop fallback strategies that rarely trigger
in production." The Google SRE book's chapter on
[cascading failures](https://sre.google/sre-book/addressing-cascading-failures/) says the same of
degraded modes: "the code path you never use is the code path that (often) doesn't work." So the
fail-open path needs a counter and a log line every time it runs, and a test.

## Three classes, not two

"Did it fail?" is the wrong question. The useful one is "did the dependency give us an answer?"

| Class | What it means | Examples | The gate | Retry? |
|---|---|---|---|---|
| **Rejected** | It answered, and the answer is no | Over the limit, account not found, account inactive | Fail closed, with a client error that says why | No. The SRE book: "Don't retry permanent errors or malformed requests in a client, because neither will ever succeed" |
| **Unavailable** | No answer: it couldn't be reached, or couldn't do the work | Timeout, connection refused, a 502 or 503 page from a proxy | The gate's policy: open or closed, counted either way | Yes, within the request's budget ([chapter 4](04-deadlines-and-retries.md)) |
| **Unknown** | An answer you can't read | An error code you've never seen, a body that won't parse | A deliberate default, chosen once | Follows the default |

The third row is the one people leave out, and it's where the bugs live. Any response the
classifier can't read has to go somewhere, and if nobody chose where, it goes wherever the code
happens to fall through to.

Which default to give unknowns is a real trade-off, and teams choose differently:

| Unknown defaults to | When the dependency adds a new rejection code | When it adds a new transient code |
|---|---|---|
| Unavailable (fail open) | Silently ignored until you update the mapping | Handled correctly |
| Rejected (fail closed) | Handled correctly | Customers refused until you update the mapping |
| The status class: 4xx rejected, 5xx unavailable | Correct if the dependency uses 4xx for rejections | Correct if it uses 5xx for outages |

The third row follows [RFC 9110 §15](https://www.rfc-editor.org/rfc/rfc9110.html#section-15): "a
client MUST understand the class of any status code, as indicated by the first digit, and treat an
unrecognized status code as being equivalent to the x00 status code of that class." It's only as
good as the dependency's use of status codes, which is the next section. Whichever you pick, count
every unknown and log its status and code, so the mapping gets updated within days rather than
discovered in an incident.

## The status code is a hint

[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5) defines the classes: a 4xx
"indicates that the client seems to have erred", and a 5xx
"[indicates that the server is aware that it has erred or is incapable of performing the requested method](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6)".
A dependency that follows it sends a business rejection as a 4xx and an outage as a 5xx, and
classifying by status would almost work.

In practice the status lies often enough that it can't be the classifier:

- **A 500 for "not found".** Some services answer an unknown identifier with 500 and an
  explanatory message. A gateway in front of another service may relabel every upstream failure
  as 500, so a 400 from the far end arrives as a 500 with a code that still says "not found".
- **A 200 that failed.** A service that wraps a vendor call returns `200` with `"success": false`
  when the vendor call failed. A caller that checks only the status reads it as a clean negative
  result.
- **402 means nothing in particular.** "The 402 (Payment Required) status code is reserved for
  future use" ([§15.5.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.3)). Using it
  for "over your spending limit" is a reasonable convention, but a client that doesn't know your
  convention only knows it's a 4xx.

The reliable signal is the dependency's own machine-readable error code. [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457.html)
(problem details, which obsoletes RFC 7807) standardises one: a `type` member, a URI identifying the
problem type, and "Consumers MUST use the "type" URI (after resolution, if necessary) as the problem
type's primary identifier." It also says "Consumers SHOULD NOT parse the "detail" member for
information", the human-readable text. Plenty of services don't use RFC 9457 and send a code field
of their own instead, and the rule carries over: decide on the code, keep the status as a
tie-breaker, and match message text only as a last resort for a dependency that gives you nothing
else.

> ⚠️ **Copy error codes from the provider's contract, never from memory.** A constant table typed
> from memory swapped two codes, so "account inactive" was treated as "over the limit" and the real
> limit code fell through to the fail-open default. Nothing errored. Put a comment on the table
> naming the provider's published catalogue and its version, and test every row.

## One adapter, one function

The classification lives in one function, in the package that talks to the dependency. Nothing
else in the service inspects that dependency's status codes or messages. First, an error type that
records what the classifier needs:

```go
type Kind int

const (
    KindTransport Kind = iota + 1 // no response: refused, reset, timed out
    KindStatus                    // a non-2xx response we could read
    KindDecode                    // a non-2xx response we couldn't read
)

type UpstreamError struct {
    Kind    Kind
    Service string
    Method  string
    Path    string // URL path only: query strings can carry personal data
    Status  int    // 0 when no response arrived
    Code    string // the provider's machine-readable code, if any
    Body    []byte // a bounded copy, for logs
    Err     error  // the cause, for transport errors
}
```

Then the classifier and its classes:

```go
type Class int

const (
    ClassUnknown     Class = iota // the zero value: a path that forgot to classify lands here
    ClassRejected                 // it answered no: fail closed
    ClassUnavailable              // no answer: apply the gate's policy
)

// Codes from the billing service's published error catalogue, v3. Source of truth: their
// contract, not memory. Every entry has a test case.
var rejectCodes = map[string]bool{
    "LIMIT_EXCEEDED":    true,
    "ACCOUNT_NOT_FOUND": true,
    "ACCOUNT_INACTIVE":  true,
}

func Classify(err error) Class {
    var ue *UpstreamError
    if !errors.As(err, &ue) {
        return ClassUnknown
    }
    switch {
    case ue.Kind == KindTransport:
        return ClassUnavailable
    case rejectCodes[ue.Code]:
        return ClassRejected // whatever the status: a 500 with this code is still a no
    case ue.Code == "" && ue.Status >= 500:
        return ClassUnavailable // an error page with no code of theirs: an outage
    }
    return ClassUnknown // a code we don't know, or a body we couldn't read
}
```

Making `ClassUnknown` the zero value is deliberate, for [chapter 1](01-how-a-go-service-runs.md)'s
reason: a `Class` that nobody set should mean "we don't know", not "fine".

The gate then reads like its policy:

```go
func (g *Gate) Allow(ctx context.Context, account string) error {
    err := g.billing.Reserve(ctx, account)
    if err == nil {
        return nil
    }
    switch billing.Classify(err) {
    case billing.ClassRejected:
        return ErrNotAllowed // the handler maps this to a 4xx that names the reason
    case billing.ClassUnavailable:
        g.failedOpen.Inc() // policy for this gate: fail open, and count it
        g.log.Warn("billing unavailable, allowing request", "err", err)
        return nil
    default:
        g.unknown.Inc()
        g.log.Error("unclassified billing error", "err", err)
        return g.onUnknown(err) // the default, decided once
    }
}
```

<details><summary>Rust</summary>

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug, PartialEq)]
enum Class { Unknown, Rejected, Unavailable }

#[derive(Debug)]
enum Kind { Transport, Status, Decode }

#[derive(Debug)]
struct UpstreamError {
    kind: Kind,
    status: u16,                                // 0 when no response arrived
    code: Option<String>,                       // the provider's machine-readable code
    body: Vec<u8>,                              // a bounded copy, for logs
    source: Option<Box<dyn Error + Send + Sync>>,
}

impl fmt::Display for UpstreamError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "upstream {:?} error, status {}", self.kind, self.status)
    }
}

impl Error for UpstreamError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        self.source.as_deref().map(|e| e as &(dyn Error + 'static))
    }
}

const REJECT_CODES: &[&str] = &["LIMIT_EXCEEDED", "ACCOUNT_NOT_FOUND", "ACCOUNT_INACTIVE"];

fn classify(err: &(dyn Error + 'static)) -> Class {
    // Walk the source() chain: the analogue of errors.As. Rust's chain is a list, not a tree.
    let mut cur = Some(err);
    while let Some(e) = cur {
        if let Some(ue) = e.downcast_ref::<UpstreamError>() {
            return match (&ue.kind, ue.code.as_deref()) {
                (Kind::Transport, _) => Class::Unavailable,
                (_, Some(c)) if REJECT_CODES.contains(&c) => Class::Rejected,
                (Kind::Status, None) if ue.status >= 500 => Class::Unavailable,
                _ => Class::Unknown,
            };
        }
        cur = e.source();
    }
    Class::Unknown
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <exception>
#include <set>
#include <stdexcept>
#include <string>

enum class Class { Unknown, Rejected, Unavailable };
enum class Kind { Transport, Status, Decode };

struct UpstreamError : std::runtime_error {
    Kind kind;
    int status;          // 0 when no response arrived
    std::string code;    // the provider's machine-readable code
    std::string body;    // a bounded copy, for logs
    UpstreamError(Kind k, int s, std::string c, std::string b)
        : std::runtime_error("upstream error"), kind(k), status(s),
          code(std::move(c)), body(std::move(b)) {}
};

Class classify(const std::exception& e) {
    static const std::set<std::string> reject{
        "LIMIT_EXCEEDED", "ACCOUNT_NOT_FOUND", "ACCOUNT_INACTIVE"};
    if (auto* ue = dynamic_cast<const UpstreamError*>(&e)) {
        if (ue->kind == Kind::Transport) return Class::Unavailable;
        if (reject.count(ue->code)) return Class::Rejected;
        if (ue->code.empty() && ue->status >= 500) return Class::Unavailable;
        return Class::Unknown;
    }
    // Wrapped with std::throw_with_nested? Look inside: the analogue of errors.As.
    try {
        std::rethrow_if_nested(e);
    } catch (const std::exception& inner) {
        return classify(inner);
    }
    return Class::Unknown;
}
```

</details>

<details><summary>JavaScript</summary>

```js
class UpstreamError extends Error {
  constructor({ kind, status = 0, code = "", body = "", cause }) {
    super(`upstream ${kind} error, status ${status}`, { cause });
    Object.assign(this, { kind, status, code, body });
  }
}

const REJECT_CODES = new Set(["LIMIT_EXCEEDED", "ACCOUNT_NOT_FOUND", "ACCOUNT_INACTIVE"]);

function classify(err) {
  // Walk the cause chain: the analogue of errors.As.
  for (let e = err; e; e = e.cause) {
    if (e instanceof UpstreamError) {
      if (e.kind === "transport") return "unavailable";
      if (REJECT_CODES.has(e.code)) return "rejected";
      if (!e.code && e.status >= 500) return "unavailable";
      return "unknown";
    }
  }
  return "unknown";
}
```

</details>

The tests are a table with one row per response someone has actually seen, including the awkward
ones:

```go
func TestClassify(t *testing.T) {
    limit := &UpstreamError{Kind: KindStatus, Status: 402, Code: "LIMIT_EXCEEDED"}
    cases := []struct {
        name string
        err  error
        want Class
    }{
        {"over the limit", limit, ClassRejected},
        {"not found, sent as 500", &UpstreamError{Kind: KindStatus, Status: 500, Code: "ACCOUNT_NOT_FOUND"}, ClassRejected},
        {"proxy error page", &UpstreamError{Kind: KindStatus, Status: 502}, ClassUnavailable},
        {"timeout", &UpstreamError{Kind: KindTransport, Err: context.DeadlineExceeded}, ClassUnavailable},
        {"wrapped by a caller", fmt.Errorf("reserve: %w", limit), ClassRejected},
        {"a code we've never seen", &UpstreamError{Kind: KindStatus, Status: 409, Code: "NEW_THING"}, ClassUnknown},
        {"4xx we couldn't parse", &UpstreamError{Kind: KindDecode, Status: 402}, ClassUnknown},
    }
    for _, tc := range cases {
        if got := Classify(tc.err); got != tc.want {
            t.Errorf("%s: got %v, want %v", tc.name, got, tc.want)
        }
    }
}
```

When the dependency changes its behaviour, the change is one row in this table and one line in the
classifier. Whether a "not found" is billed, or logged at warning, is a separate decision made by
the caller from the class, so it can change without touching the classification.

## Act on the owner's verdict

A balance service started allowing accounts to go below zero, down to a floor set per account. A
caller still read `available_balance` from the success response and refused the request when it
was negative, so accounts inside their allowed overdraft were blocked. The owner had already said
yes. The caller had re-derived the decision from metadata, using the old rule.

The service that owns a policy, a balance, a quota or an entitlement, makes the decision, and
callers act on its explicit verdict: a success, or an error with a rejection code. Fields in the
response are for display and logging. Re-deriving the verdict copies the policy into every caller,
and the copies drift the first time the owner's rule changes.

## Verdicts aren't errors, and absent isn't zero

The classifier keeps "no" apart from "couldn't ask". Two more distinctions sit next to it, and
each has a Go shape:

| Keep apart | Because | In Go |
|---|---|---|
| "No" and "couldn't ask" | One fails closed, the other is policy | `Classify(err)` returns `ClassRejected` or `ClassUnavailable` |
| A negative outcome and an error | "No match" is a normal result, and the caller should carry on | A field in the result, with `err == nil` |
| "Not reported" and zero | Zero is a measurement | `*float64`, and a nullable column |

The second row bites in loops. A matcher returned a valid result *and* a sentinel error meaning
"no candidate matched". Its caller treated any error as fatal, so of several images it only ever
tried the first. If a function's normal outcomes include "no", put the "no" in the result and keep
`error` for "couldn't find out". Where a sentinel is unavoidable, every call site has to check it
with `errors.Is` before treating the error as fatal.

The third row bites in storage. A dependency that doesn't report a score is not reporting a score of
zero, but a `float64` field can only say zero, and `0.0000` on a dashboard looks like a measurement
that happened. With a pointer, the absence survives: `encoding/json` says "A nil pointer encodes as
the null JSON value."

## Transient means retryable, not terminal

The classes decide more than the gate.

**A state machine.** A multi-step workflow treated any error during a step as terminal: it marked
the job failed and refunded the reservation. A few milliseconds of skew between a cache and a
database produced a transient error, and a paid job was destroyed. Unavailable errors should leave
the step retryable ([chapter 4](04-deadlines-and-retries.md)). Only a rejection, or an exhausted
retry budget, should reach a terminal state with side effects.

**A circuit breaker.** A breaker exists to stop calling a dependency that is down. A rejection means
the dependency is working perfectly, so it shouldn't count towards tripping. With
[sony/gobreaker](https://github.com/sony/gobreaker), that's the `IsSuccessful` setting, which "is
called with the error returned from a request. If `IsSuccessful` returns true, the error is counted
as a success." Its default "returns false for all non-nil errors"
([README](https://github.com/sony/gobreaker/blob/v1.0.0/README.md)), so out of the box a burst of
legitimate rejections trips the breaker. Plug in the classifier:

```go
cb := gobreaker.NewCircuitBreaker[*Reservation](gobreaker.Settings{
    Name: "billing",
    IsSuccessful: func(err error) bool {
        return err == nil || billing.Classify(err) == billing.ClassRejected
    },
})
```

v2 also has `IsExcluded`: an excluded error is "neither counted as a success nor as a failure",
and its doc comment suggests it for context cancellations
([`v2/gobreaker.go:76-82`](https://github.com/sony/gobreaker/blob/v2.4.0/v2/gobreaker.go#L76-L82)). A
caller that gave up says nothing about the dependency's health.

> ⚠️ **Never classify by searching the message.** A breaker counted an error as an outage if
> `err.Error()` contained "unavailable" or "500". The service's own HTTP client wrapped every 5xx
> as "service unavailable: …", and the dependency sent some rejections as 500s, so every rejection
> matched and the breaker opened on a healthy dependency. Someone had already removed "500" from
> the list for the same reason. Messages are written for people, and every layer that wraps an
> error adds words to them. Classify on typed fields.

## Matching errors through wrapping

Everything above depends on the classifier finding the `*UpstreamError` however many layers have
wrapped it. That's what Go's `errors` package is for, and string comparison is what it replaces.

### Strings break under `%w`

A handler turned "no rows" into a 404 with `err.Error() == "no rows in result set"`. It worked
until a repository layer wrapped the error with context. Measured:

```go
err := fmt.Errorf("load contract: %w", ErrNoRows)
fmt.Println(err.Error() == "no rows in result set") // false
fmt.Println(errors.Is(err, ErrNoRows))               // true
```

The [`errors` documentation](https://pkg.go.dev/errors) explains the mechanism. An error wraps
another if it has an `Unwrap() error` or `Unwrap() []error` method, "Successive unwrapping of an
error creates a tree", and `Is` and `As` "inspect an error's tree by examining first the error
itself followed by the tree of each of its children in turn (pre-order, depth-first traversal)."

| To ask | Use | Not | Why |
|---|---|---|---|
| Is this, or does it wrap, a known value? | `errors.Is(err, ErrNoRows)` | `err == ErrNoRows`, or comparing `err.Error()` | `==` sees only the outer error. The message changes whenever anyone wraps it |
| Is this, or does it wrap, a known type? | `errors.As(err, &ue)` | `err.(*UpstreamError)` | A type assertion sees only the outer error |

Measured, on an `*UpstreamError` wrapped once by `fmt.Errorf("check balance: %w", …)`:

```
type assertion: false errors.As: true billing
```

Wrapping is a promise as well as a convenience. The Go blog's
[Working with Errors in Go 1.13](https://go.dev/blog/go1.13-errors), by Damien Neil and Jonathan
Amsterdam: "wrapping an error makes that error part of your API." Wrap the errors you want callers
to match. Don't wrap a database driver's error into a public function's result if you don't want
callers depending on the driver.

### One error, two identities

A caller wants to ask two different questions of the same failure: "did the billing call fail?"
and "was it a timeout?". Since [Go 1.20](https://go.dev/doc/go1.20#errors), "An error `e` can wrap
more than one error by providing an `Unwrap` method that returns a `[]error`", and `errors.Is` and
`errors.As` "have been updated to inspect multiply wrapped errors." So the upstream error can unwrap
to both a sentinel for the dependency and the root cause:

```go
var ErrBilling = errors.New("billing")

func (e *UpstreamError) Error() string {
    return fmt.Sprintf("%s %s %s: status %d %s: %v", e.Service, e.Method, e.Path, e.Status, e.Code, e.Err)
}

func (e *UpstreamError) Unwrap() []error {
    if e.Err == nil {
        return []error{ErrBilling}
    }
    return []error{ErrBilling, e.Err}
}
```

Measured, on a transport error wrapping `context.DeadlineExceeded`, wrapped once more by the caller:

```
Is ErrBilling: true Is DeadlineExceeded: true
```

The `nil` check isn't decoration. The `errors` documentation: "It is invalid for an Unwrap method
to return an []error containing a nil error value."

The fields earn their place in the log line. `Kind` says where the call failed, which separates "we
built a bad request" from "the network dropped it" from "they said no" from "they sent something we
can't read". The path without its query, and a body capped at a few kilobytes, mean the error can be
logged without copying personal data or a megabyte of HTML into the logs.

## The body is read once

`resp.Body` isn't a value you can look at twice. The [`Response`](https://pkg.go.dev/net/http#Response)
documentation: "The response body is streamed on demand as the Body field is read." Once something
has read it, it's gone.

A client logged the body of every error response and then decoded it for the error code:

```go
msg, _ := io.ReadAll(resp.Body)            // for the log line
log.Printf("billing error %d: %s", resp.StatusCode, msg)
err := json.NewDecoder(resp.Body).Decode(&p) // for the code
```

Measured, against a server that sent a 402 with a valid JSON code:

```
first read 58 bytes; second decode: EOF, code=""
```

The decode failed, so the code fell back to a generic error with no status, which the classifier
treated as unavailable. Under a fail-open policy, every rejection was allowed. Two fixes, and the
second matters as much as the first:

```go
const maxErrBody = 4 << 10

func readError(resp *http.Response, service string) error {
    body, _ := io.ReadAll(io.LimitReader(resp.Body, maxErrBody)) // read once, bounded
    ue := &UpstreamError{
        Kind: KindStatus, Service: service, Status: resp.StatusCode, Body: body,
        Method: resp.Request.Method, Path: resp.Request.URL.Path,
    }
    var p struct {
        Error struct{ Code string } `json:"error"`
    }
    if json.Unmarshal(body, &p) != nil || p.Error.Code == "" {
        ue.Kind = KindDecode // the status is kept: a 4xx we can't read is still a 4xx
        return ue
    }
    ue.Code = p.Error.Code
    return ue
}
```

Read into bytes once, then log and parse from the bytes. And make every fallback keep the
upstream's status, so a failure to parse degrades to "unknown 402", which the classifier can still
handle deliberately, instead of to "something went wrong", which it can't.

<details><summary>Rust</summary>

```rust
// reqwest: text() takes `self`, so reading the body twice doesn't compile.
let status = resp.status().as_u16();
let body = resp.text().await.unwrap_or_default(); // read once
let body = &body[..body.len().min(4096)];          // bounded (cut at a char boundary in real code)
let code = serde_json::from_str::<serde_json::Value>(body)
    .ok()
    .and_then(|v| v["error"]["code"].as_str().map(str::to_owned));
// status is kept whether or not the code parsed
```

</details>

<details><summary>C++</summary>

```cpp
// Whatever the HTTP library, the body arrives as a stream. Read it into a string once.
std::string body(std::istreambuf_iterator<char>(stream), {});
if (body.size() > 4096) body.resize(4096);       // bounded
auto code = parse_error_code(body);              // returns std::optional<std::string>
throw UpstreamError(code ? Kind::Status : Kind::Decode, status, code.value_or(""), body);
```

</details>

<details><summary>JavaScript</summary>

```js
// fetch: a body can be consumed once; a second res.json() after res.text() rejects.
const text = (await res.text()).slice(0, 4096); // read once, bounded
let code = "";
try { code = JSON.parse(text)?.error?.code ?? ""; } catch {}
throw new UpstreamError({ kind: code ? "status" : "decode", status: res.status, code, body: text });
```

</details>

> **Teacher's aside.** People frame this as "fail open or fail closed?", as if that were the
> decision. It's the second decision. The first is classification: which failures even have a
> choice. A rejection has no choice, a transient failure has a policy, and an unreadable response
> needs a default someone picked on purpose. Get the classification wrong and the policy doesn't
> matter, because the permissive branch runs for the wrong errors. That's why the classifier sits in
> one function with a test for every response you've seen: it's the place where a misread error
> quietly turns into the permissive path.

## Check yourself

1. A billing client logs any error and returns `nil, nil`. Accounts over their limit are being
   served. Explain mechanically why, and why "return the error instead" isn't the whole fix.
2. A dependency answers an unknown account with `500` and the code `ACCOUNT_NOT_FOUND`. Your
   classifier uses the status class alone, and the gate fails open on unavailable. What happens to
   the request, to the breaker, and to your on-call rota? What should the classifier key on?
3. A breaker counts an error as a failure when `err.Error()` contains "unavailable". The service's
   client wraps every 5xx as `fmt.Errorf("service unavailable: %w", err)`. The dependency starts
   sending some rejections as 500s. Describe what happens, and what the breaker should count instead.
4. A client logs an error body with `io.ReadAll(resp.Body)`, then decodes `resp.Body` for the code,
   and falls back to a generic error with no status when decoding fails. The dependency sends a
   402 with a valid code. What does a fail-open gate do, and what are the two fixes?
5. Your log line says "context deadline exceeded", but `errors.Is(err, context.DeadlineExceeded)` is
   false for errors from your client. Give a likely reason, and show how one error can answer both
   "did the billing call fail?" and "was it a timeout?".
6. The balance service now allows balances down to a per-account floor below zero, and your gate
   refuses when `available_balance < 0` in a success response. What breaks, and what's the
   principle? Separately: the service adds a new rejection code tomorrow. What happens if unknowns
   default to fail open, and what if they default to fail closed? What makes either default safe
   to live with?
