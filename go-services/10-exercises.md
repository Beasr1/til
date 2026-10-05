# 10. Exercises

Worked answers to every **Check yourself** question, then things to try and questions
worth asking me.

---

## File 01 — How a Go service runs

**1. A goroutine waiting on `<-results` after the handler returns.**

It blocks for ever. Nothing will send, and nothing can stop a goroutine from outside. It keeps
its stack and everything it references (the request's data, perhaps a large decoded body), and
because running goroutines are garbage-collection roots, none of that is ever freed. One per
request is a leak that grows with traffic.

**2. Ten senders, `make(chan int, 9)`, reader starts late.**

The first nine sends fill the buffer and succeed. The tenth blocks, because the buffer is full
and nobody is receiving. If the reader is waiting for "all ten have finished" before it reads,
it waits for ever, because the tenth can't finish until someone reads. That's a deadlock. If
it's the whole program, the runtime reports "all goroutines are asleep". In a server, where
other goroutines are still running, nothing reports it at all.

**3. `h.count++` from two requests.**

A data race. `++` is a read, an add and a write, and two goroutines interleaving them can both
read 5 and both write 6, losing an increment. With maps it's worse: concurrent writes can crash
the process. Find it with `go test -race`, which only catches races on paths the tests actually
run concurrently, so write a test that calls the handler from several goroutines. Fix it with
`sync/atomic` for a counter or a mutex for anything larger.

**4. CPU limit 2 on a 32-core node, `go 1.24` in `go.mod`, Go 1.26 toolchain.**

`GOMAXPROCS` is 32. The container-aware default (`containermaxprocs`) is a GODEBUG setting, and
GODEBUG defaults are "amended to match the Go version listed in `go.mod`", so a module declaring
1.24 keeps the 1.24 behaviour of counting logical CPUs. With 32 runnable threads under a quota
of 2 CPUs, the process gets throttled by the CPU bandwidth limit, often showing up as latency
spikes. Bumping the `go` line to 1.25 or later makes the default 2.

**5. What cancels `r.Context()`, and how a query escapes it.**

The client's connection closing, the request being cancelled (with HTTP/2), and the handler's
`ServeHTTP` returning. A query keeps running anyway if it was started with a different context:
`context.Background()` inside a repository helper, a method that takes no context and calls
`Query` instead of `QueryContext`, or a goroutine started with a fresh context.

**6. A `kafka.Writer` built as a struct literal.**

Read the documentation of every field you didn't set, because each zero is a decision you didn't
make. For kafka-go's `Writer`, the zero `RequiredAcks` means don't wait for the broker, the zero
`Balancer` is round-robin and ignores keys, and the zero `BatchTimeout` flushes every second. "It
worked in staging" isn't evidence, because these settings matter only under failure (a broker
restart), ordering-sensitive traffic, or load, and staging rarely has any of the three.

---

## File 02 — The HTTP client

**1. A new Transport per call, 50 calls a second, after a day.**

Every call opens a fresh TCP (and TLS) connection, which then sits idle for ever in a pool
nobody reuses. That's about 4.3 million connections a day, each with roughly two client
goroutines. Long before then you'd see the process's open file descriptors climb steadily
until it hits its limit ("too many open files"), goroutine count and memory rising in a
straight line, and the upstream's connection count rising too. Latency is higher from the
start, because every call pays a handshake.

**2. Why the garbage collector doesn't clean up.**

An idle pooled connection has live goroutines (the read and write loops) that reference it
and its Transport. Running goroutines are roots for the garbage collector, so everything
they reference stays reachable. And a zero-value Transport has no `IdleConnTimeout`, so
nothing ever tells those goroutines to exit.

**3. One shared `&http.Transport{}`.**

Fixed: connections are reused, and the leak stops. Remaining: no proxy from the environment,
no dial or TLS-handshake timeouts, no idle timeout (idle connections to hosts you stop
calling stay open), and HTTP/2 disabled the moment you add a custom dialler or TLS config.
Clone `http.DefaultTransport` instead.

**4. A 500 with the body left unread and unclosed.**

The connection can't go back to the pool, because the Transport doesn't know whether the
rest of the body is still coming. It stays tied to the unclosed body until that's garbage
collected, and the next request opens a new connection. Under a burst of upstream errors,
the error path is exactly where connections leak.

**5. A helper with its own 30-second background context.**

The outbound call carries on for up to 30 seconds after the user has gone, using a
connection, a goroutine and upstream capacity for a response nobody will read. Cancellation
of the inbound request can't reach it. The helper should take a `ctx context.Context` as its
first parameter and derive any per-call timeout from it, for example
`context.WithTimeout(ctx, 30*time.Second)`, which is then bounded by whichever deadline is
sooner.

---

## File 03 — The HTTP server

**1. Slowloris.**

`ReadHeaderTimeout` (or `ReadTimeout`). Slowloris trickles the request *headers*, so the
server never reaches the handler and never writes anything. `WriteTimeout` governs writing
the response and never comes into play. A server with no read timeouts holds the connection
for as long as the attacker keeps trickling.

**2. A 90-second hang under `WriteTimeout: 30s`.**

At 60 seconds the handler is still blocked on the dependency. Its context isn't cancelled
(unless the client disconnected, which cancels it for HTTP/1.1). At 90 seconds the
dependency answers, the handler writes, the write fails because the connection's write
deadline passed long ago, and the client, which has waited the whole 90 seconds, gets a
connection error such as `EOF`. Nothing told it at 30 seconds.

**3. `TimeoutHandler` set, slow requests still hold database connections.**

The handler's database calls aren't using `r.Context()`. `TimeoutHandler` cancels the
context it passes to the handler, and that only stops work that's listening to it. A query
issued with `context.Background()`, or through a repository method that takes no context,
runs to completion regardless.

**4. `WriteTimeout` longer than the `TimeoutHandler` duration.**

When the work deadline fires, `TimeoutHandler` writes a 503 to the client. If the
connection's write deadline has already passed, that write fails too, and the client gets a
bare connection error instead of a clear 503. With `WriteTimeout` slightly longer, the
handler's timeout response always has time to get out.

**5. In-flight requests cut off during deploys.**

`Shutdown` makes `ListenAndServe` return `ErrServerClosed` immediately. `main` returns as
soon as `ListenAndServe` does, and the process exits, while `Shutdown`, running in the other
goroutine, is still waiting for active requests to finish. Those requests die with the
process. `main` must block until `Shutdown` itself returns.

**6. A liveness probe that queries the database.**

The database slows down, so the probe's query passes one second and the probe fails on every
pod at about the same time. After the failure threshold, the kubelet restarts every container.
Restarting pods drop their in-flight requests, start cold, and reconnect to the database all at
once, adding load to a database that was already slow. While they're down, the remaining pods
take all the traffic and fail their probes too. A slow database has become a full outage, caused
by your own probe. Liveness should check only that the process itself can make progress, for
example that the HTTP server answers a trivial handler. A database dependency, if it's hard,
belongs in readiness, which removes the pod from traffic without restarting it.

---

## File 04 — Deadlines and retries

**1. Tab closed after one second, helper using `context.Background()`.**

The handler's own context is cancelled, so anything it does directly with `r.Context()`
stops. The helper's outbound call doesn't. It keeps running for up to its own timeout, and if
it's wrapped in a retry loop that was also given a background context, every remaining
attempt and backoff runs too. The server-side work for a request nobody is waiting for can
last minutes.

**2. The worst case from the configuration table.**

Five attempts at up to 30 seconds each is 150 seconds. Backoff between them (200 ms, 400 ms,
800 ms, 1.6 s if the first wait is already doubled) adds about 3 seconds. The 60-second
elapsed cap is only checked after a failure, so it stops the loop after the second or third
attempt has *finished*, not when the cap passes. Realistically the handler runs 60–90
seconds, and longer if the cap check is missing. The client receives nothing at 30 seconds,
because `WriteTimeout` doesn't stop the handler, and a connection error when the handler
finally tries to write.

**3. Retrying 404s during a burst of new-record lookups.**

Every lookup for a record that doesn't exist yet becomes five requests, spaced out by
backoff. A burst of N such lookups sends 5N requests to the upstream, and each inbound
request is held open for the whole backoff sequence, tying up goroutines and connections. A
`404` is a definitive answer. Retrying it only multiplies load, unless the record is
genuinely expected to appear within the retry window, which is a different design problem.

**4. Exponential backoff alone versus full jitter.**

A thousand clients that failed at instant t all wait the same 100 ms, retry at t+100 ms
together, fail together, wait 200 ms, and retry together again. The dependency sees spikes of
a thousand requests with silence between them, which is the worst load shape for recovery.
Full jitter makes each client wait a random time between zero and the backoff value, so the
retries spread evenly across the window and the dependency sees a steady, lower rate.

**5. Three layers each retrying three times.**

3 × 3 × 3 = 27 attempts can reach the database for one click. Keep retries at one layer,
usually the client library or service closest to the database, which can tell whether the
operation is safe to repeat and whether the error is transient. Make the gateway and the
outer service fail fast, or let them retry only connection failures that never reached a
server.

**6. Retrying a timed-out payment `POST`.**

Only if the server deduplicates requests by an **idempotency key** that the client generated
before the first attempt and sends unchanged on every retry, so that a second arrival returns
the first one's result instead of charging again. Without it, a timeout says nothing about
whether the payment happened: the request may have succeeded and only the response was lost.
The alternative is to query the payment's status by your own reference before deciding.

---

## File 05 — Fanning out

**1. Why nobody reads until `done`, and why the buffer is critical.**

The main goroutine waits on `done`, and only drains the channels once it's closed. `done` is
closed only after `wg.Wait()` returns, which needs every worker to have returned. So every
worker's send must complete with no reader, which is possible only while the buffer has room.
The buffer has to hold every send, so its size must equal the number of senders.

**2. Four workers, buffer three, no deadline.**

Three sends fill the buffer. The fourth blocks for ever, so its worker never calls `wg.Done()`,
`wg.Wait()` never returns, and `done` never closes. The handler waits on a `select` that will
never fire, and the client eventually times out on its side. An hour later every such request
has left at least two goroutines behind (the stuck worker and the waiter), plus the handler's
own goroutine and whatever it holds. Goroutine count and memory climb in a straight line with
traffic to that endpoint.

**3. Why merges, and what catches it.**

The correct state requires three numbers to agree (the `Add` count and two buffer sizes) and the
number of `go` statements, spread over several lines. Two branches can each change some of them
correctly, and git merges the lines without conflict. Review catches it only if the reviewer
checks that the numbers still agree after the merge, which nobody does reliably. What catches it
structurally is not having the numbers: `errgroup`, or channels sized from `len(sources)` in one
place.

**4. A fails at 10 ms while B is querying.**

A's error cancels the group's context. If B's query was started with that context (`QueryContext(ctx,
…)`), the driver sees the cancellation and asks the database to cancel the query, and B returns
`context.Canceled`. `g.Wait()` returns A's error once B has returned. B must have received the
group's `ctx`, not the outer one or a background context, and passed it into the query call.

**5. One goroutine per item, unbounded.**

A request with 10,000 items starts 10,000 goroutines, each opening a database query or HTTP call
at once. That exhausts the connection pool (everything else queues behind it), may overwhelm the
dependency, and spikes memory. The fix is `g.SetLimit(n)` with a sensible `n`, plus a limit on
request size.

**6. When not to fail on the first error.**

When partial results are more useful than none: a dashboard of independent panels, a search
across optional sources, enrichment that's nice to have. Then each task records its own result
*and* error in its own slot and returns `nil` to the group, so the group never cancels, and the
caller decides afterwards what to show and what to report as degraded.

---

## File 06 — Copies that share memory

**1. Which assignments affect `a`?**

`b.Name = "x"`: no, strings are values. `b.Photo = nil`: no, it reassigns `b`'s own slice
header. `b.Photo[0] = 0`: yes, it writes into the shared backing array. `b.Attrs["k"] = "v"`:
yes, the map is shared. `b.Attrs = nil`: no, it reassigns `b`'s own map reference.

**2. Reassigning versus writing into the slice.**

A slice field holds a header pointing at a backing array. Reassigning `cp.Snapshot` replaces
the copy's header with one pointing at new memory, so the original's header still points at
the old bytes, untouched. `copy(cp.Snapshot[…], …)` follows the copy's header to the backing
array, which is the same array the original points at, and changes the bytes there. Same
field, different operation, opposite outcome.

**3. `maps.Clone` and a write two levels down.**

The original changes too. `maps.Clone` copies the top-level map's entries, and the value under
`"doc"` is a reference to a nested map, so both top-level maps now point at the same nested
map. For a decoded JSON document use a real deep copy: a JSON marshal/unmarshal round trip,
a recursive clone function over `map[string]any` and `[]any`, or, better, change the code so
it builds a new document instead of editing the old one.

**4. Deep-copying a struct with a `sync.Mutex` and an unexported cache.**

The library documents that "unexported field values are not copied". `sync.Mutex`'s
internals are unexported, so the copy gets a zero-value mutex, unlocked whatever the
original's state. That happens to be what you'd want, but by accident. The unexported cache
field comes back zero-valued, so the copy behaves like a fresh, empty instance, which may break
invariants the type relies on. (`go vet`'s copylocks check also flags copying a struct that
contains a mutex.)

**5. Making `watermark` unable to affect callers.**

Change it from "modify the document you're given" to "return a new document":
`func watermark(doc map[string]any) map[string]any`, which builds and returns a new map and
treats `doc` as read-only, copying only the parts it changes on the way down. Callers keep
their original by default and use the return value when they want the watermarked one.

---

## File 07 — Time in a container

**1. Laptop and `golang:` image fine, `scratch` wrong.**

On the laptop LoadLocation finds the operating system's database (the second location). In a
`golang:` image, `$GOROOT/lib/time/zoneinfo.zip` exists because the whole toolchain is
installed (the third location). A `scratch` image has no OS files and no `$GOROOT`, and if the
program doesn't import `time/tzdata` there's nothing left, so the load fails and the code falls
back to UTC.

**2. Why only part of each day, and which part at UTC−5.**

The date differs only when UTC and the local zone are on different calendar days. For UTC+4
that's 20:00–24:00 UTC, when local time has passed midnight. For UTC−5 it's the other way
round: from 00:00 to 05:00 UTC, UTC has already moved to tomorrow while local time is still
on today, so the UTC fallback dates records a day *late*.

**3. When `FixedZone` is exactly right.**

When the zone has had a single UTC offset with no daylight saving across every date you'll
handle, past and future as far as you know. Check the zone's entry in the IANA tz source
(the region file, `asia`, `europe` and so on): a `-` in the RULES column and no offset changes
in the period you care about. Also consider that governments change zones, and the IANA
database tracks it while your constant doesn't.

**4. Two ways `MMDDhhmmss` plus four random digits collides.**

Two records in the same second draw the same four digits (the birthday problem). And two
records exactly one year apart in the same second draw the same digits, because the format
has no year, so its timestamps repeat every year.

**5. 200 a second, unique.**

Using the approximation, 200 × 199 / 20,000 ≈ 1.99, so the probability of at least one
collision in a given second is about 1 − e<sup>−1.99</sup> ≈ 86%. Collisions would be routine.
Change the scheme: more random digits (the chance falls with the square of the suffix space),
a database sequence or counter for the suffix, or a standard identifier such as a UUIDv7, which
is time-ordered and has 74 random bits. Keep a unique constraint either way.

**6. `0 * * * *` with a forty-minute job, running continuously.**

First candidate: the parser treats a single value as the start of a range, so the minute field
`0` matches 0–59 and every minute is a match (measured on such a parser: next runs 12:01,
12:02, 12:03). Each run lasts forty minutes and the next match is the minute after it ends,
so the job never stops. Second candidate: steps or ranges mis-parsed so the minute set is
wider than written, with the same effect. A table-driven test that, for a fixed start time,
asserts the next three run times of a few canonical expressions (`0 * * * *` must give
13:00, 14:00, 15:00 from 12:00:30) catches both, and any day-of-week mistake too if
`0 0 1 * 1` is among them.

---

## File 08 — When a dependency moves your toolchain

**1. It builds on a laptop with Go 1.24.**

The default `GOTOOLCHAIN=auto` sees that the module requires 1.25, downloads a 1.25 toolchain
and re-runs the command with it, printing a "switching to go 1.25.x" message that's easy to
miss.

**2. It fails `FROM golang:1.24-alpine`.**

The official image sets `ENV GOTOOLCHAIN=local` ("don't auto-upgrade the gotoolchain"), so
Go uses only the toolchain installed in the image, and it refuses to load a module whose `go`
line is newer than itself. The image sets it so a build uses the compiler the image tag names
rather than silently downloading another, which is what makes builds reproducible.

**3. Taking the patch without raising the `go` line.**

Not with that dependency version: your `go` line must be at least as high as every
requirement's. Options: upgrade the toolchain (usually the right answer); find a patched
release of the dependency on a line that still supports your Go version, if the maintainers
backported the fix; or, as a last resort, apply the fix with a `replace` directive to a fork,
which you then have to maintain.

**4. `RUN go mod tidy` in a Dockerfile.**

`tidy` resolves the module graph inside the build and can add, remove or change `go.mod` and
`go.sum` entries, depending on what it finds online and on the build's tags. The image is then
built from dependency metadata nobody reviewed or committed. Even when it changes nothing, it
makes "what was built" depend on the network at build time. `go mod download` with the
committed files, or `go build -mod=readonly`, builds exactly what's in the repository and
fails if it's inconsistent.

**5. `replace lib => lib v1.79.3`, then something requires v1.81.0.**

v1.79.3. A replace without a version on the left replaces "all versions of the module", so
minimal version selection picks v1.81.0 and the replace then swaps it for v1.79.3. The build
succeeds unless the code needs an API added after v1.79.3. `go list -m lib` shows it:
`lib v1.81.0 => lib v1.79.3`. The pin should have been a requirement, `go get lib@v1.79.3`,
which sets a floor and lets newer requirements win. If a replace is truly needed, give it a
left-hand version (`replace lib v1.78.0 => lib v1.79.3`) so it only touches the version it was
meant to fix, and delete it once the graph has moved past.

**6. A server-side gRPC advisory in a client-only service.**

Run `govulncheck` (or read the advisory's affected symbols) and check whether any call path
from your code reaches the vulnerable functions. If the flaw is in server-side authorisation
and the service never starts a gRPC server, govulncheck reports the module as imported with no
call stacks, which its tutorial says means "You may not need to take any action". Upgrade at the
next normal opportunity rather than with an emergency workaround. If a call stack *is* reported,
treat it as urgent. A version scanner can't make this distinction, so its severity isn't the
service's severity.

---

## File 09 — Worker pools

**1. Paths and digests swapped, tests passing.**

The results are appended in completion order and paired by index. Tests usually pass one file
per record, or files of similar size served by a fast local fake, so completion order happens
to match input order. In production, two files of different sizes or a slow first request make
the second finish first, and `results[0]` belongs to `files[1]`. A failure is worse: the failed
item leaves no result, so every later result shifts by one (measured: with the first of three
failing, input 0 was paired with result B and input 1 with result C).

**2. Ten messages, four fail.**

The caller sees `nil`. The pool logs one of the four errors and returns `nil`. A consumer that
commits on `nil` commits all ten offsets, so the four failed messages are neither processed nor
dead-lettered: they're gone, and the only trace is one log line covering one of them.

**3. A contract that distinguishes outcomes.**

Return a result per input, by index, each carrying its own error, and let the caller count:
all `nil` is success, some is partial, all is total failure. If the caller only needs a single
error, return `errors.Join` of all of them, so none is hidden. The first error alone is right
only when the first failure makes the rest pointless and you cancel them, which is `errgroup`'s
contract. It's the wrong contract for a batch where each item stands alone.

**4. 1,000 chunks, 5 at a time, 200 ms each, one 20-second context.**

Each wave of five takes 200 ms, so 20 seconds allows 100 waves, 500 chunks. The remaining 500
fail with `context deadline exceeded`, mostly without ever doing work if acquisition watches the
context, or after waiting for a slot if it doesn't. Per-chunk 20-second timeouts derived from the
batch context change nothing: a child's deadline is the earlier of its own and its parent's
(measured: a child asking for 10 s under a 300 ms parent got 300 ms). To finish, either the
batch deadline must cover `chunks ÷ concurrency × time per chunk`, or each chunk needs a context
that isn't a child of the batch's.

**5. Acquiring the semaphore without watching the context.**

`sem <- struct{}{}` blocks until a slot frees, whatever the context says. After cancellation,
every queued goroutine still waits its turn, then starts its call, which fails at once because
the context is done: no useful work, but each holds a slot and a goroutine until then. Selecting
on `ctx.Done()` alongside the send lets them give up immediately. `errgroup.SetLimit` limits how
many functions run at once, and with `WithContext` every function started after the first error
receives an already-cancelled context, so a function that checks it returns at once instead of
doing the work.

**6. `defer recover()` but `wg.Done()` at the end of the loop body.**

The panic unwinds past the `wg.Done()` at the end of the loop body, so it never runs for that
task. Where the recover sits only changes what happens next: deferred per task, the worker
carries on with other tasks; deferred around the whole worker, the worker exits and the others
share its load. The `WaitGroup` count
never reaches zero, `wg.Wait()` blocks for ever, and the whole batch hangs, along with whatever
called it. Put `defer wg.Done()` at the top of the per-task function so it runs on every exit.

---

## Things to try

- Reproduce file 02's measurement: a local server, 200 requests through a shared Transport,
  then through a new one per request. Print `runtime.NumGoroutine()` and count `StateNew`
  connections with `Server.ConnState`.
- Reproduce file 03's: `WriteTimeout: 500ms`, a handler that sleeps two seconds and prints
  `r.Context().Err()`. Then wrap it in `http.TimeoutHandler` and compare.
- Build a static binary that calls `time.LoadLocation`, and run it `FROM scratch` with and
  without `import _ "time/tzdata"`.
- Write a test that copies a struct holding a map and a slice, mutates the copy, and asserts
  the original is unchanged. Watch it fail, then fix the code under test rather than the
  test.
- Write a pool that appends results under a mutex, give the first input a 50 ms delay, and
  print which result lands at index 0. Then rewrite it to write `results[i]` and compare.
- Pin a transitive dependency with a version-less `replace`, require a newer version from
  another module, and read `go list -m` for it.

## Questions worth asking me

- "Here's our HTTP client setup. What breaks under load?"
- "What should this handler's timeout budget be, given what it calls?"
- "Where in this call chain should retries live?"
- "Is this retry helper correct? Check the first wait, the cap, and what it retries."
- "What does our graceful shutdown actually wait for?"
- "Which other OS files does a `scratch` image need for this binary?"
- "Here's our worker pool helper. What does a caller learn when half the tasks fail?"
- "Here's our `go.mod`. Which `replace` lines are still doing anything?"
