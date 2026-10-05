# Running a Go Service in Production — A Course

A course on the parts of a Go HTTP service that look finished in development and fail
slowly in production: the HTTP client's connection pool, what the server's timeouts
really bound, how deadlines and retries interact, copies that aren't copies, time zones
in minimal images, worker pools that mislabel or drop results, and dependency bumps that
change your toolchain or quietly pin an old version.

**This is reference learning material.** Everything here is general Go, checked against
the standard library source and documentation for Go 1.26. Where behaviour matters and
the docs leave room for doubt, I ran a small program and show its output. The motivating
failures are ordinary: a service leaking a connection per request, handlers that kept
running long after clients had given up, a "copy" whose edits leaked into stored data,
dates off by one for four hours a day, and a security patch that broke the image build.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the fix.
- Every chapter ends with **Check yourself** questions. Answers are in
  [10-exercises.md](10-exercises.md).
- Measurements are labelled as measurements, and computed figures as computed.

## The one thing to understand first

> **Most of these failures come from an object or a setting whose zero value or default
> does something reasonable for one request and something harmful for a million.** A
> fresh `Transport` works for one call and leaks over a day. A `WriteTimeout` looks like a
> request timeout and bounds only the write. `context.Background()` is fine in `main` and
> cuts every cancellation path when used in a helper. A struct assignment looks like a
> copy and shares its maps.

The fix each time is to know what the default really does, by reading its documentation or
running it, rather than what its name suggests.

## Reference implementations

| Source | What it settles |
|---|---|
| [`net/http` Transport](https://pkg.go.dev/net/http#Transport) | Reuse, `DefaultTransport`'s settings, idle limits, `ForceAttemptHTTP2` |
| [`net/http` Server](https://pkg.go.dev/net/http#Server) | What each timeout bounds, `Shutdown`, `TimeoutHandler` |
| [`context`](https://pkg.go.dev/context) | Cancellation, deadlines, `WithoutCancel` (Go 1.21) |
| [`time.LoadLocation`](https://pkg.go.dev/time#LoadLocation) and [`time/tzdata`](https://pkg.go.dev/time/tzdata) | Where zone data comes from, and embedding it |
| [IANA tz database](https://github.com/eggert/tz) | Whether a zone has daylight saving |
| [Go toolchains](https://go.dev/doc/toolchain) / [release policy](https://go.dev/doc/devel/release#policy) | The `go` line as a minimum, `GOTOOLCHAIN`, support windows |
| [Go spec: channel types](https://go.dev/ref/spec#Channel_types) / [Go 1.25 release notes](https://go.dev/doc/go1.25) / [GODEBUG](https://go.dev/doc/godebug) | Channel blocking; container-aware `GOMAXPROCS` and how its default follows the `go` line |
| [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup) | Fan-out with shared cancellation and limits |
| [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) | Liveness, readiness and startup, and the cascading-failure warning |
| [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) / [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) | Backoff variants and retry practice, by Marc Brooker |
| [Go modules reference](https://go.dev/ref/mod) | What a version-less `replace` replaces; `-mod=readonly` |
| [govulncheck tutorial](https://go.dev/doc/tutorial/govulncheck) | Reachability: imported but not called |
| [POSIX crontab](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/crontab.html) | Day-of-month OR day-of-week; weekday range |

## Reading order

### Part 0 — Foundations

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [How a Go service runs](01-how-a-go-service-runs.md) | ⭐ Explain goroutines and leaks, channel blocking, the server's goroutine-per-connection model, `context`, and why zero-value config often means "no limit". **Prerequisite for everything else** |

### Part 1 — Talking to other services

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [The HTTP client](02-the-http-client.md) | ⭐ Spot a per-request Transport, and build a client that pools, times out and proxies correctly |
| 3 | [The HTTP server](03-the-http-server.md) | Say what each server timeout bounds, put a real deadline on handler work, shut down without dropping requests, and keep dependencies out of liveness probes |
| 4 | [Deadlines and retries](04-deadlines-and-retries.md) | ⭐ Make a retry budget fit inside a request's deadline, and retry only what can succeed |
| 5 | [Fanning out](05-fanning-out.md) | Replace a hand-counted WaitGroup fan-out with `errgroup`, and bound its concurrency |
| 9 | [Worker pools](09-worker-pools.md) | Pair batch results with their inputs by key, make a pool report partial failure, and size deadlines for queued work. Read after 5 |

### Part 2 — Inside the process

| # | File | After this you can… |
|---|------|---------------------|
| 6 | [Copies that share memory](06-copies-that-share-memory.md) | Predict which fields a struct copy shares, and pick a cloning strategy |
| 7 | [Time in a container](07-time-in-a-container.md) | Make time-zone handling work in `scratch`, judge identifiers built from timestamps, and test a hand-parsed cron schedule |
| 8 | [When a dependency moves your toolchain](08-when-a-dependency-moves-your-toolchain.md) | Explain why a dependency bump broke the image build, pin a patched transitive dependency without a hidden downgrade, and judge a scanner finding |

### Reference

| # | File | |
|---|------|---|
| 10 | [Exercises & answers](10-exercises.md) | Worked answers, things to try, and questions to ask |

Read chapter 1 first. Then read 2 → 3 → 4 in order: chapter 4 builds on the context discussion
in 2 and the server timeouts in 3. Chapter 5 needs chapter 1's channel rules. Chapter 9 was
added later and takes the next number, but belongs straight after 5, which it builds on, so it's
listed there. Chapters 6, 7 and 8 are independent of each other and of Part 1. Not yet written, and belonging here: structured
logging and what not to log, metrics and cardinality, database connection pools, and profiling
with `pprof`.

## If you're short on time

- **New to running Go services:** chapter 1, all of it.
- **10 minutes:** chapter 2's measurement table, then chapter 3's "`WriteTimeout` doesn't
  stop your handler".
- **A service's goroutines or file descriptors climb steadily:** chapter 2.
- **Writing a retry helper, or reviewing one:** chapter 4, "Add up the worst case" and the
  two small bugs after "Backoff".
- **Requests to a fan-out endpoint hang until timeout:** chapter 5.
- **Batch results are attached to the wrong records, or failures vanish:** chapter 9.
- **Dates are off by one in production only:** chapter 7.
- **A security bump broke the Docker build, or you're pinning a CVE fix:** chapter 8.

## The one-paragraph summary of everything

A Go service is goroutines. The HTTP server starts one per connection, so handlers run
concurrently. A goroutine blocked for ever is never collected, a channel send blocks once the
buffer is full, cancellation reaches only code that receives the request's `context`, and a
zero-valued config field often means "no limit". Go's HTTP client keeps its connection pool in the `Transport`, and the docs say plainly that
Transports "should be reused instead of created as needed". One per request opened 200
connections for 200 calls and left 600 goroutines alive, because a zero-value Transport has
no idle timeout. It also drops `DefaultTransport`'s proxy support, dial and handshake
timeouts, so clone the default once and share it. Bodies must be read and closed for
connections to be reused. On the server, `ReadHeaderTimeout` is the one to always set.
`WriteTimeout` bounds only the response write: a handler under it ran to completion with
its context never cancelled, and the client waited the whole time and got `EOF`. Real
deadlines come from the handler's context, via `TimeoutHandler` or `context.WithTimeout`,
and only stop work that's passed that context. Helpers that build their own background
contexts break the chain. `Shutdown` drains HTTP connections but not the goroutines handlers
started, and `main` must wait for it. Retries must fit inside the caller's deadline, target
only transient failures, use backoff with jitter, and live at one layer, since three retries
at each of five layers is 243 attempts. Assigning a struct copies its slice headers and map
references, so writes through the "copy" reach the original, and `maps.Clone` is shallow.
Prefer functions that return new values over functions that mutate their input. A hand-rolled
fan-out with a `WaitGroup` and buffered channels hangs and leaks when a buffer size falls one
behind the number of senders. `errgroup` removes the counting and cancels the rest on the first
error. Liveness probes must never check dependencies.
`time.LoadLocation` needs the IANA database at run time, and neither `alpine` nor `scratch`
provides it. Embed it with `time/tzdata` and fail at startup rather than falling back to
UTC. A fixed offset is exact only for zones without daylight saving. Timestamp-plus-random
identifiers repeat yearly if the year is missing, and collide by the birthday bound. Since
Go 1.21 the `go` line is a minimum that dependencies can raise, and the official images set
`GOTOOLCHAIN=local`, so a dependency bump can require a new base image. To force a patched
transitive dependency, raise a requirement with `go get`: a version-less `replace` replaces all
versions, so it silently downgrades the day something needs newer (measured: `v1.80.0 =>
v1.79.3`), and it does nothing in a library. Scanners match versions; `govulncheck` tells you
whether the vulnerable code is called. Hand-parsed cron schedules need table tests: POSIX ORs
day-of-month with day-of-week, and a parser that treats `0` as "0 onwards" runs an hourly job
every minute. Worker pools that append results under a mutex return them in completion order,
so pairing `results[i]` with `inputs[i]` mislabels data; write to `results[i]` or carry the
key. A pool that logs an error and returns `nil` makes its caller commit failures, and one
deadline shared by queued work fails healthy tasks that waited too long, because a child
context can never outlive its parent.

## How to use me

Ask me things like:

- "Review this HTTP client constructor for production use."
- "What's the end-to-end timeout budget for this endpoint, and where is it enforced?"
- "Is this retry loop safe to put in front of a payment API?"
- "Does this function mutate its input? Show me how the caller could be affected."
- "What does this binary need from the OS that a `scratch` image won't have?"
- "We're bumping this dependency. What else in the repo has to move with it?"
