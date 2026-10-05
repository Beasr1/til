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
4. Why should `WriteTimeout` be a bit longer than the `TimeoutHandler` duration rather than
   shorter?
5. `main` calls `srv.Shutdown(ctx)` in a signal-handling goroutine and returns when
   `ListenAndServe` returns. In-flight requests are cut off during deploys. Explain.
6. A liveness probe calls the database and fails if the query takes over a second. The
   database has a slow five minutes. Describe what happens to the service's pods, and what the
   probe should check instead.
