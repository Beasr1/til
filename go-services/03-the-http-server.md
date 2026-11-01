# 3. The HTTP server

## The problem

You set `WriteTimeout: 30 * time.Second` on the server, and believe no request can take
longer than thirty seconds. Then a dependency hangs, and requests run for minutes, holding
database connections and goroutines, while clients have long since been told the
connection failed. The server's timeouts are about **the connection**, not about **the
work**. Go's documentation says so, but the field names suggest otherwise.

## What each server timeout bounds

From the `http.Server` documentation in `server.go`:

| Field | Bounds | Notes from the source |
|---|---|---|
| `ReadHeaderTimeout` | Reading the request line and headers | "the Handler can decide what is considered too slow for the body" |
| `ReadTimeout` | Reading the whole request, body included | "most users will prefer to use ReadHeaderTimeout" |
| `WriteTimeout` | Writing the response | "It is reset whenever a new request's header is read" |
| `IdleTimeout` | Waiting for the next request on a keep-alive connection | Falls back to `ReadTimeout` if zero |
| `MaxHeaderBytes` | Size of the request headers | "It does not limit the size of the request body" |

`ReadHeaderTimeout` is the one every server should set. Without it, a client can open a
connection and send headers one byte at a time for ever, holding a goroutine and a file
descriptor. That's the **Slowloris** attack, and it needs no bandwidth to exhaust a server.

## `WriteTimeout` doesn't stop your handler

A server with `WriteTimeout: 500 * time.Millisecond`, and a handler that takes two seconds
and then reports whether its context was cancelled:

```
client after 2.003s: err=Get "http://127.0.0.1:61042": EOF
handler finished after 2s; ctx.Err()=<nil>
```

Two things to notice:

- **The handler ran to completion.** Its context was never cancelled. Every database
  query and outbound call it made in those two seconds happened, and their results were
  thrown away.
- **The client waited the full two seconds**, then got a bare `EOF` instead of a response.
  `WriteTimeout` set a deadline on the connection. The deadline only took effect when the
  handler finally tried to write, and the write failed. So the client got neither a fast
  failure nor a useful error.

A `WriteTimeout` protects the server from slow *readers*: clients that won't consume a
response. It's not a request timeout.

## Bounding the work

To bound the work, the handler's **context** needs a deadline, and the handler has to pass
that context to everything it calls. Two ways to give it one:

```go
// 1. Wrap handlers. On timeout it responds 503 and cancels the handler's context.
srv.Handler = http.TimeoutHandler(router, 10*time.Second, "request timed out")

// 2. Per handler, when different routes need different budgets.
ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
defer cancel()
```

`TimeoutHandler` runs the inner handler with `context.WithTimeout(r.Context(), dt)` (you
can read this in its `ServeHTTP`), and "if a call runs for longer than its time limit, the
handler responds with a 503 Service Unavailable error". The cancellation only does anything
if the handler's database calls and HTTP calls *use* that context. A helper that makes its
own `context.Background()` (chapter 2) ignores it.

Set `WriteTimeout` a little *longer* than the work deadline, so the handler's own timeout
response gets out before the connection deadline cuts it off.

## When the client goes away

A handler's error path usually ends in a catch-all: anything it doesn't recognise is logged at
Error and answered with a 500. That's right for a bug or a database that's down. Then a client
starts cancelling requests it no longer needs, such as a page that aborts its in-flight fetches
with an `AbortController` when the user navigates away, and the error log fills with lines like
these:

```
level=ERROR msg="request failed" err="list items: context canceled"
level=ERROR msg="request failed" err="get item 42: timeout: context already done: context canceled"
```

Nothing failed. Someone clicked a link. A streaming endpoint, Server-Sent Events for instance,
treats this as normal from its first day, because every stream ends with the client leaving. A
short request whose client leaves is the same event. But an alert that counts errors can't tell the
difference, and the real failures are buried in the noise.

### How the server finds out

[Chapter 1](01-how-a-go-service-runs.md) quoted the rule: an incoming request's context is
cancelled "when the client's connection closes, the request is canceled (with HTTP/2), or when the
ServeHTTP method returns". The first two are a client leaving, and they reach the server
differently.

**HTTP/1.1** has no message for cancelling a request, so a client that gives up closes the
connection. Any side "MAY close the transport connection at any time", and the same section of
[RFC 9112 (§9.5)](https://www.rfc-editor.org/rfc/rfc9112#section-9.5) asks implementations to
"constantly monitor open connections for a received closure signal". Go's server does it with a
one-byte **background read** that waits on the connection while the handler runs. From
[`server.go`](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L2140-L2147):
"Start background read, which detects when a client has closed its connection while a request
handler is still running." When the client closes, that read fails, and
[`handleReadErrorLocked`](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L813-L821)
cancels the connection's context, the parent of the request's.

**HTTP/2** carries many requests on one connection, so a client cancels one by resetting its
stream. From [RFC 9113 §6.4](https://www.rfc-editor.org/rfc/rfc9113#section-6.4): "RST_STREAM is
sent to request cancellation of a stream". The server's
[`processResetStream`](https://github.com/golang/go/blob/go1.27.3/src/net/http/internal/http2/server.go#L1553-L1570)
cancels that stream's context and leaves the connection open.

Measured on Go 1.27.3, and the same on 1.26.9, with a handler that waits up to two seconds on its
context:

```
Go client, 100 ms timeout           client: context deadline exceeded   handler: context canceled after 100ms
Node 24 fetch, abort() at 100 ms    client: AbortError                  handler: context canceled
HTTP/2 client, cancel at 100 ms     client: context canceled            handler: context canceled after 100ms
  the next HTTP/2 request           200 OK over the same connection
```

Look at the first line. The client hit its own timeout, and the handler saw a cancellation. A
timeout, a closed tab and a navigation all arrive as a closed connection or a reset stream, so the
server learns that the client left, never why.

Three things about the background read are worth knowing:

- **It starts only once the request body has been read.** The comment quoted above goes on: "When
  the request has a body, we start the background read only after the entire body has been
  consumed."
  A handler that hasn't read its body to the end can't see the client leave. Measured, with the
  client gone at 100 ms: POST handlers that ignored a 5-byte body, or read 2 bytes of it, ran their
  full second with their contexts intact. Ones that read it all, or decoded it with
  `json.Decoder`, stopped at 100 ms.
- **`ReadTimeout` can't cancel a handler that has read its request.** `startBackgroundRead` clears
  the read deadline before starting the read
  ([L741](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L741)). Measured with
  `ReadTimeout: 200ms`: a GET handler, and a POST handler that had read its body, each ran a full
  second with its context intact, and the clients got `200 OK`. A handler still reading the body
  when `ReadTimeout` passes is different. The read fails with `i/o timeout`, and a failed read
  cancels the context the same way
  ([L860-L862](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L860-L862);
  measured), so a stalled upload looks like a client that left.
- **A pipelined request hides a departure.** If the next request's first byte arrives while the
  handler runs, the background read takes it and stops without cancelling (since Go 1.11), so a
  client that then leaves goes unnoticed until a write fails
  ([L748-L772](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L748-L772);
  measured, the handler ran its full 300 ms). As the comment there says, "almost nothing uses
  HTTP/1.x pipelining".

### Behind a proxy

Behind nginx there's one more hop, and nginx's default passes the departure on.
[`proxy_ignore_client_abort`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ignore_client_abort)
defaults to `off`, and "Determines whether the connection with a proxied server should be closed
when a client closes the connection without waiting for a response." So when the client closes,
nginx closes its own connection to your server, and your server sees the HTTP/1.1 case above. An
HTTP/2 client's stream reset takes the same path inside nginx
([`ngx_http_v2.c`](https://github.com/nginx/nginx/blob/release-1.30.6/src/http/v2/ngx_http_v2.c#L2117-L2141)).

nginx logs the event at `info`, "client prematurely closed connection, so upstream connection is
closed too"
([`ngx_http_upstream.c`](https://github.com/nginx/nginx/blob/release-1.30.6/src/http/ngx_http_upstream.c#L1539-L1544)),
and records the request with status **499**. HTTP defines no such code, and the documentation of
nginx's core, log, proxy and upstream modules doesn't mention it. The source defines it, as
`NGX_HTTP_CLIENT_CLOSED_REQUEST`, and says why
([`ngx_http_request.h`](https://github.com/nginx/nginx/blob/release-1.30.6/src/http/ngx_http_request.h#L132-L138)):

> HTTP does not define the code for the case when a client closed the connection while we are
> processing its request so we introduce own code to log such situation when a client has closed
> the connection before we even try to send the HTTP header to it

So a 499 in nginx's access log and `context canceled` in your server's log are one event, seen
from each side. Two things break the chain: `proxy_ignore_client_abort on`, and a response that
nginx is caching. In both, nginx keeps reading from your server after the client has gone, so the
work runs to completion and your server never learns.

### Telling it apart from a failure

The fix goes in the catch-all. It asks whether this request's client has gone, not whether the
error mentions cancellation:

```go
switch {
// ... the errors you recognise, mapped to 404, 409, 422 and so on
default:
    if errors.Is(r.Context().Err(), context.Canceled) {
        slog.Info("request cancelled by the client", "err", err)
    } else {
        slog.Error("request failed", "err", err)
    }
    writeError(w, http.StatusInternalServerError)
}
```

Why not `errors.Is(err, context.Canceled)`? Because the error is a symptom, and each layer reports
the cancellation its own way. pgx, a PostgreSQL driver, reports a query started on a context that's
already done as the second log line above: the context's error, properly wrapped, behind the word
"timeout" ([`errors.go`](https://github.com/jackc/pgx/blob/v5.11.0/pgconn/errors.go#L194-L231)).
Other errors don't wrap it at all. Measured, the two checks disagree in both directions:

| Situation | `errors.Is(err, context.Canceled)` | `r.Context().Err()` |
|---|---|---|
| The client leaves while the handler writes a large response | `false`: the write fails with `broken pipe` or `connection reset by peer` | `context canceled` |
| Two requests share one load through `singleflight`, and the first request's client leaves | `true` for both | `context canceled` for the first. `nil` for the second, whose client was still waiting and got a 500 |

The second row is the dangerous one. `singleflight` makes a duplicate caller wait for the original
and receive "the same results"
([`singleflight.go`](https://github.com/golang/sync/blob/v0.24.0/singleflight/singleflight.go#L86-L91)),
so when the shared load runs on the first request's context, that client's departure becomes every
caller's error. Checked by the error, the second request's 500 is logged at Info, as a client's
choice. It's a bug: the shared load needs a context of its own, as background work does in
[chapter 10](10-routing-and-frameworks.md). Checked by the request's context, it's logged at Error.

Checking for `Canceled` in particular, rather than for any error on the context, matters under
`TimeoutHandler`, which puts its deadline on the context the inner handler gets. Measured inside
one: a client that left first gave the handler `context canceled`, and a deadline that passed
first gave it `context deadline exceeded`.

The check is only as good as the list of things that cancel `r.Context()`. With net/http alone,
while the handler runs, that's the client leaving, plus the stalled upload above. Add a canceller
of your own, such as a `BaseContext` you cancel at shutdown, and it cancels every request's context
too (measured), so the Info line blames the client. Give your own cancellations a cause with
`context.WithCancelCause`. Then `context.Cause(r.Context())` returns that cause, while a client
that left still gives `context.Canceled` (measured, both).

> ⚠️ **Write the error response anyway.** If the client has gone, the 500 reaches no one and costs
> nothing. If it hasn't, it gets a real error: with the `BaseContext` cancelled, the
> still-connected client got the 500 (measured). A handler that sees the cancellation and returns
> without writing sends `200 OK` with an empty body instead
> ([`finishRequest`](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L1714-L1719);
> measured: `200 OK, 0 bytes`), a success that never happened.

### Cancelled isn't timed out

`context.DeadlineExceeded` looks like `Canceled`'s twin, but in a handler it points the other way,
at your own server. net/http never adds a deadline to a request's context. It derives it with `WithCancel`, for
[HTTP/1.1](https://github.com/golang/go/blob/go1.27.3/src/net/http/server.go#L1089) and
[HTTP/2](https://github.com/golang/go/blob/go1.27.3/src/net/http/internal/http2/server.go#L2087)
alike, and a client's timeout reaches you as a cancellation (the first measurement above). So every
`DeadlineExceeded` in a handler comes from a budget your server set, through `TimeoutHandler` or a
`context.WithTimeout` of its own ("Bounding the work", above). That's a problem on your side, a
dependency too slow or a budget too tight:

| What ended the work | What the handler sees (measured) | Log at | Respond with |
|---|---|---|---|
| The client left | `r.Context().Err()` is `context.Canceled` | Info | The 500 anyway |
| A deadline the server set | Under `TimeoutHandler`, `r.Context().Err()` is `DeadlineExceeded`. Under the handler's own `WithTimeout` it's `nil`, and the error wraps `DeadlineExceeded` | Warn or Error | 503, as `TimeoutHandler` sends, or 504 |
| The work failed | `r.Context().Err()` is `nil` | Error | 500 |

gRPC has a status code for each case, and its tooling draws the same lines. go-grpc-middleware's
logging interceptor, by default
([`options.go`](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/v2.3.4/interceptors/logging/options.go#L105-L121)),
logs a call that ended `Canceled` at Info, alongside `OK` and `NotFound`; `DeadlineExceeded` at
Warn; and `Unknown`, `Internal` and any code it doesn't list at Error. Google's error model maps
them to HTTP
([`code.proto`](https://github.com/googleapis/googleapis/blob/ebd1d23ac613b177828dad42ad8dfb13ba498279/google/rpc/code.proto#L38-L67)):
`CANCELLED`, "The operation was cancelled, typically by the caller", is "499 Client Closed
Request", the code nginx uses, and `DEADLINE_EXCEEDED` is "504 Gateway Timeout".

## Graceful shutdown

On `SIGTERM`, a rolling deploy wants the old process to stop accepting work and finish
what it has. `Server.Shutdown` does the HTTP half:

> Shutdown works by first closing all open listeners, then closing all idle connections,
> and then waiting indefinitely for connections to return to idle and then shut down. If
> the provided context expires before the shutdown is complete, Shutdown returns the
> context's error.

What it does *not* wait for:

| Not covered by `Shutdown` | What to do |
|---|---|
| Goroutines a handler started and didn't wait for (background publishes, async logging) | Track them in a `sync.WaitGroup` and wait on it after `Shutdown` returns |
| Message consumers running alongside the server | Cancel their context *first*, so no new messages are taken, then wait for the in-flight one |
| Hijacked connections (WebSockets) | "Shutdown does not attempt to close nor wait for hijacked connections" |
| The orchestrator's grace period | The process is killed when it expires, whatever is still running |

And a detail that's easy to get wrong: after calling `Shutdown`, `ListenAndServe` returns
`ErrServerClosed` *immediately*. If `main` treats that return as "done" and exits, it kills
the process while `Shutdown` is still draining connections. Return from `main` only after
`Shutdown` itself has returned.

```go
<-quit                                   // SIGTERM
consumerCancel()                         // stop taking new messages
ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()
_ = srv.Shutdown(ctx)                    // drain HTTP
background.Wait()                        // drain fire-and-forget work
```

Pick the shutdown timeout below the orchestrator's grace period, Kubernetes'
`terminationGracePeriodSeconds` for instance, so the process finishes on its own terms
instead of being killed.

## Checking dependencies at startup and in probes

A common early design: on startup, call every dependency's health endpoint (database,
message broker, three upstream services) and refuse to start if any fails. It feels like
safety. In practice it couples your service's *availability* to every dependency's, so when
one upstream has a bad minute, every new pod of yours fails to start, a rolling deploy stalls,
and an autoscaler can't add capacity at exactly the moment it's needed. Many services end up
removing these checks.

Orchestrators separate the questions, and it helps to keep them separate in code. From the
Kubernetes
[probe documentation](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/):

| Probe | Failing means | Should check |
|---|---|---|
| **Liveness** | The kubelet restarts the container | Only that *this process* can make progress, e.g. it isn't deadlocked. Never a dependency |
| **Readiness** | The pod is removed from Service endpoints, so it gets no traffic | That this instance can serve requests now. A hard dependency it can't work without can count |
| **Startup** | Liveness and readiness wait until it passes | That initialisation finished |

The documentation's warning: "Incorrect implementation of liveness probes can lead to
cascading failures." A liveness probe that checks the database restarts every pod when the
database is slow. That adds load and restarts to an outage that wasn't yours. Readiness is the
right place for "I can't serve without X", and even there, be honest about which dependencies
are *hard*. A service that can still answer most requests without an upstream should stay ready
and degrade the requests that need it.

> **Teacher's aside.** People think of timeouts as protecting the *client* from waiting.
> Server-side, they mostly protect the *server* from holding resources for work nobody
> will read. That's why the work deadline, propagated through context, matters more than
> any field on `http.Server`. A cancelled request that keeps running a five-second
> database query is a load amplifier during exactly the incident that caused the
> cancellations.

## Check yourself

1. Which server timeout defends against Slowloris, and why doesn't `WriteTimeout`?
2. A handler calls a dependency that hangs for 90 seconds. The server has
   `WriteTimeout: 30s` and nothing else. What does the client see, when, and what's the
   handler doing at 60 seconds?
3. You wrap the router in `http.TimeoutHandler(…, 10*time.Second, …)`, but slow requests
   still hold database connections for 90 seconds. Give the most likely reason.
4. `main` calls `srv.Shutdown(ctx)` in a signal-handling goroutine and returns when
   `ListenAndServe` returns. In-flight requests are cut off during deploys. Explain.
5. A liveness probe calls the database and fails if the query takes over a second. The
   database has a slow five minutes. Describe what happens to the service's pods, and what the
   probe should check instead.
6. A page aborts its in-flight fetches when the user navigates away, and your server's error
   log fills with `context canceled`. A colleague proposes logging at Info any error for which
   `errors.Is(err, context.Canceled)` is true. Describe a request this would log at Info that
   is a real failure, and a departed client it would still log at Error. What should the check
   look at instead, and why should the handler still write the 500?
