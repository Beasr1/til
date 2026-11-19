# 5. Fanning out

## The problem

A request needs data from several places that don't depend on each other: three tables,
four services. Doing them one after another adds their latencies together. Doing them at
once costs only the slowest. So you start a goroutine for each, collect the results, and
carry on. The hand-written version of that pattern contains a number that has to be updated
every time someone adds a source, and when it's one too small, requests hang and goroutines
leak. It isn't caught at compile time, and it isn't caught by tests that don't wait for a
timeout.

## The hand-rolled pattern

Here's the shape that turns up in many codebases:

```go
var wg sync.WaitGroup
results := make(chan Result, 3)   // ← must equal the number of senders
errs    := make(chan error, 3)    // ← and so must this
done    := make(chan struct{})

wg.Add(3)                          // ← and this
go fetch(ctx, sourceA, &wg, results, errs)
go fetch(ctx, sourceB, &wg, results, errs)
go fetch(ctx, sourceC, &wg, results, errs)

go func() { wg.Wait(); close(done); close(results); close(errs) }()

select {
case <-ctx.Done():
    return ctx.Err()
case <-done:
    // drain errs, then results
}
```

It works. Each worker sends into a buffered channel and finishes, the `WaitGroup` reaches
zero, `done` closes, and the main goroutine drains the buffers. Its correctness rests on one
fact: **every send must succeed without a reader**, because nobody reads until after `done`,
and `done` only closes after every worker has returned. From chapter 1: a send on a buffered
channel succeeds without blocking only "if the buffer is not full".

## What happens when the number is wrong

Someone adds a fourth source: a fourth `go fetch(...)` and `wg.Add(4)`, and the buffer stays
at 3. Measured with five workers and a buffer of four, under a two-second context deadline:

```
5 workers, buffer 5: n=5 err=<nil>
5 workers, buffer 4: n=0 err=context deadline exceeded after 2.1s; goroutines left behind=2
```

The fifth worker's send finds the buffer full and blocks. It never returns, so `wg.Wait()`
never returns, so `done` never closes. The request waits until its deadline and fails. If the
context has no deadline, it waits for ever. And two goroutines are left blocked permanently,
the stuck worker and the one waiting on the `WaitGroup`. Nothing will ever unblock them. Every
such request adds two to the goroutine count (chapter 1: blocked goroutines are never
collected).

> ⚠️ The bug is easiest to introduce in a **merge**. One branch adds a source and bumps
> `wg.Add`, another branch changes the buffer sizes, and the merged result compiles with
> numbers that no longer agree. Nothing flags it in review, because each side's diff was
> correct.

There are quieter problems in the same shape:

- **The first error wins, the rest are discarded, and nobody stops the others.** When source
  A fails at 10 ms, B and C keep running to completion, holding connections, for a result
  that will be thrown away.
- **The error and result channels are separate**, so the caller can't easily tell which source
  failed, or whether a "result" is a zero value sent after an error.
- **On `ctx.Done()` the function returns** while workers are still running. That's fine only
  because the buffers let them finish. Undersize a buffer, and that path leaks too.

## `errgroup` removes the numbers

`golang.org/x/sync/errgroup` packages the pattern without the counting. From its
[documentation](https://pkg.go.dev/golang.org/x/sync/errgroup): it "provides synchronization,
error propagation, and Context cancellation for groups of goroutines working on subtasks of a
common task", and `WithContext` returns a context that "is canceled the first time a function
passed to Go returns a non-nil error or the first time Wait returns, whichever occurs first."

```go
g, ctx := errgroup.WithContext(ctx)
var a ResultA
var b ResultB
g.Go(func() (err error) { a, err = fetchA(ctx); return })
g.Go(func() (err error) { b, err = fetchB(ctx); return })
if err := g.Wait(); err != nil {
    return err
}
// use a and b
```

What changed:

| Hand-rolled | errgroup |
|---|---|
| `wg.Add(n)` and two buffer sizes, kept in step by hand | No counts. Each `g.Go` is one task |
| Results through a channel, order lost | Each task writes its own variable or slice index. No channel, nothing to block on |
| First error returned, others keep running | First error cancels `ctx` for all of them, *if* they pass it on |
| Separate goroutine to close channels | `Wait` returns when every task has returned |

Measured: five tasks, one failing immediately and the others waiting a second unless
cancelled, `g.Wait()` returned `worker 3 failed` at once, because the shared context told the
others to stop.

The cancellation only helps if each task passes `ctx` to whatever it calls. A task that
ignores it runs to completion and `Wait` waits for it.

For many tasks, `SetLimit` caps concurrency: "SetLimit limits the number of active goroutines
in this group to at most n." Use it when the fan-out is over a list whose length you don't
control. A thousand lookups shouldn't become a thousand simultaneous database queries.

> **Teacher's aside.** Hand-rolled concurrency usually goes wrong in the bookkeeping, not the
> concurrency: a counter, a buffer size, a channel closed twice or never. Each is a fact stated
> in two places that must agree. `errgroup` removes the duplication, so there's nothing to keep
> in step. When you review concurrent code, look for numbers that must match other numbers.

## One failure, or all of them?

`errgroup` fails the whole request on the first error. Sometimes that's wrong. A page that
shows data from three sources might prefer two sections and an error notice to a failed page.
Then each task should capture its own error into its own result, return `nil` to the group, and
let the caller decide. Choose deliberately. "Any source failing fails everything" is a product
decision as much as a technical one.

## Goroutines that outlive their caller

Everything so far has been request-scoped: the goroutines start, finish, and the handler returns.
The harder cases are goroutines whose life isn't the request's, and cleanup that has to happen
whatever the request did. Each failure below was silent. Outputs are from Go 1.24.4.

### A goroutine lives as long as its context

From the [`context` documentation](https://pkg.go.dev/context): "When a Context is canceled, all
Contexts derived from it are also canceled." A goroutine that loops until `ctx.Done()` therefore
lives exactly as long as whichever context it was handed, not as long as the code that started it
intended. A message consumer started during initialisation, with the context the initialisation
code was using, stopped the moment initialisation finished and cancelled that context. It returned
cleanly, logged nothing, and the messages piled up. Measured, the same shape:

```
consumer on setup ctx stopped: context canceled
```

So choose the context by the goroutine's intended lifetime ([chapter 1](01-how-a-go-service-runs.md),
"`context`"):

| The goroutine is… | Give it | So that |
|---|---|---|
| Part of the request: a fan-out, a lookup | `r.Context()`, or a child of it | It stops when the client goes |
| A background loop: a consumer, a ticker, a cache refresher | A process-lifetime context from `main`, such as `signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)`, and registered with shutdown | It runs until the process is told to stop, and shutdown waits for it ([chapter 3](03-the-http-server.md)) |
| A side effect a request starts and mustn't cancel: an audit write, a notification | `context.WithTimeout(context.WithoutCancel(r.Context()), d)` | It keeps the request's values, such as a trace id, drops its cancellation, and still has a bound |
| Cleanup that must run: releasing a lock, compensating a reservation | The same: `WithoutCancel` plus its own short timeout | It runs even when the request was cancelled |

`signal.NotifyContext` returns a context "marked done (its Done channel is closed) when one of the
listed signals arrives". `context.WithoutCancel` returns one that "is not canceled when parent is
canceled. The returned context returns no Deadline or Err, and its Done channel is nil", which is
why it always needs a timeout on top.

### A fire-and-forget goroutine needs its own recovery

A panic unwinds only its own goroutine. From the [spec](https://go.dev/ref/spec#Handling_panics):
deferred functions run "up to any deferred by the top-level function in the executing goroutine.
At that point, the program is terminated". `net/http` recovers a panicking handler, assuming "the
effect of the panic was isolated to the active request" ([`Handler`](https://pkg.go.dev/net/http#Handler)),
but that covers the handler's goroutine only. A nil-map write in a goroutine the handler started
with `go` takes down the whole process, and every request in flight with it.

A best-effort side effect, started with `go` and not awaited, needs three things that the request
path gets for free:

- **Its own `recover`**, deferred at the top of the goroutine, which logs the panic with its stack
  and counts it as an error.
- **Copied inputs.** Capture the values it needs before the `go` statement. A pointer to a struct
  the handler goes on modifying is [chapter 6](06-copies-that-share-memory.md)'s shared-memory bug
  with a race on top.
- **A bounded lifetime**, from the table above.

Not everyone agrees on recovering. `errgroup` deliberately doesn't propagate a task's panic to
`Wait`. Its source explains that doing so "delays panics arbitrarily, making bugs harder to
detect", "turns f's panic stack into a mere value, hiding it from crash-monitoring tools", and
"risks deadlocks that hide the panic entirely"
([`errgroup.go:81-91`](https://github.com/golang/sync/blob/v0.20.0/errgroup/errgroup.go#L81-L91)).
So a panicking `errgroup` task still crashes the process. Both positions are defensible, for
different work. Recover in optional side effects, where crashing every in-flight request is a worse
outcome than a missed audit row, and make the recovered panic loud. Don't wrap essential work in a
blanket `recover`: a panic there means the program's state is wrong, and carrying on hides it.

### A recovered panic must still deliver a result

A hand-rolled fan-out wrapped each goroutine in a helper that recovered panics, and each goroutine
sent its result on a channel as its last line. When a lookup panicked, the helper caught it, but the
send never ran, so the receiver waited for ever. Recovering the panic had turned a crash into a
hang. Measured:

```
  safeGo recovered: assignment to entry in nil map
send at end: receiver still waiting after 1s
```

Send from a deferred function instead, which runs whether the goroutine returns or panics, and
recover in the same place so the receiver learns why:

```go
ch := make(chan result, 1) // room for the one send, so it never blocks
go func() {
    var res result
    defer func() {
        if r := recover(); r != nil {
            res.err = fmt.Errorf("lookup panicked: %v", r)
        }
        ch <- res
    }()
    res.v = lookup(ctx)
}()
```

```
send from defer: err = lookup panicked: assignment to entry in nil map
```

The spec's [defer rule](https://go.dev/ref/spec#Defer_statements) is what makes this work: deferred
functions are invoked "immediately before the surrounding function returns", and during a panic,
"Any functions deferred by F are then executed as usual." `errgroup` gets the same guarantee for
normal returns by calling `g.done()` in a defer. It's the hand-rolled version that has to remember,
and [chapter 9](09-worker-pools.md) makes the same point about a worker's `wg.Done()`.

<details><summary>Rust</summary>

```rust
// A std::thread has a join handle, and a panic arrives there as Err. Nobody waits for ever.
let handle = std::thread::spawn(move || lookup());
match handle.join() {
    Ok(v) => println!("got {v}"),
    Err(panic) => eprintln!("lookup panicked: {:?}", panic.downcast_ref::<&str>()),
}
```

</details>

<details><summary>C++</summary>

```cpp
// std::async stores an exception in the future, and get() rethrows it.
auto fut = std::async(std::launch::async, [] { return lookup(); });
try {
    auto v = fut.get();
} catch (const std::exception& e) {
    std::cerr << "lookup failed: " << e.what() << '\n';
}
// A raw std::thread whose function throws calls std::terminate: the analogue of Go's crash.
```

</details>

<details><summary>JavaScript</summary>

```js
// A throw inside an async function rejects its promise; allSettled waits for all of them.
const results = await Promise.allSettled([lookupA(), lookupB(), lookupC()]);
for (const r of results) {
  if (r.status === "rejected") console.error("lookup failed:", r.reason);
}
```

</details>

Go is the odd one out here: a goroutine has no handle to join, so a result or an error reaches the
caller only if the goroutine's own code sends it.

### Cleanup on a fresh context, and work shorter than its lease

A handler took a distributed lock for the record it was updating, with a `defer lock.Release(ctx)`.
When a slow step made the proxy in front time out, the request's context was cancelled, the
deferred release failed at once with `context canceled`, and the record stayed locked until the
lock's TTL expired. Measured, with a release function that checks its context:

```
release(req ctx): context canceled
release(WithoutCancel+timeout): <nil>
```

```go
defer func() {
    cctx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 2*time.Second)
    defer cancel()
    if err := lock.Release(cctx); err != nil {
        log.Error("lock release failed", "key", key, "err", err)
    }
}()
```

The lease has a second rule. Redis's [distributed-locks page](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
defines the lock validity time as "both the auto release time, and the time the client has in order
to perform the operation required before another client may be able to acquire the lock again". If
the work under the lock can take longer than the TTL, the lock expires mid-work, a second worker
takes it, and two workers are in the critical section at once. So bound the work with a deadline
strictly shorter than the TTL, `context.WithTimeout(ctx, ttl-margin)`. The same page recommends
fencing tokens where correctness matters, because a paused process can still overrun.

### A typed nil is not a nil error

One more trap from the same fan-out. A helper returned a `*AppError` that was nil on success,
through a function whose result type was `error`. The [Go FAQ](https://go.dev/doc/faq#nil_error)
explains why that's never nil: "An interface value is nil only if the V and T are both unset", and
a nil pointer stored in an `error` is "(T=*MyError, V=nil)", so "it will always look as if there was
an error even if nothing bad happened." Wrapping makes it worse. `errors.Is` calls the error's
`Unwrap` method, and an `Unwrap` that reads a field through the nil receiver panics. Measured:

```
typed nil == nil: false
recovered: runtime error: invalid memory address or nil pointer dereference
```

Return `error` from every function that can fail, and write a literal `return nil` on success. If a
custom error type might be used through a nil pointer, guard its methods:
`func (e *AppError) Unwrap() error { if e == nil { return nil }; return e.Err }`.
[Chapter 11](11-classifying-a-dependencys-errors.md) has the rest of error wrapping.

## Check yourself

1. In the hand-rolled pattern, why does nobody read from `results` until `done` is closed, and
   why does that make the buffer size critical?
2. With four workers and a buffer of three, and a context with no deadline, describe the request
   and the process an hour later.
3. Why is this bug most likely to appear in a merge, and what in code review would catch it?
4. With `errgroup.WithContext`, source A fails at 10 ms and B is in the middle of a database
   query. What happens to B's query, and what does B have to do for that to work?
5. A handler fans out one lookup per item in a request body with no size limit. What goes wrong
   under a large request, and what's the one-line fix?
6. When should a fan-out *not* fail on the first error, and how do you structure the tasks then?
7. A handler takes a lock with a 30-second TTL, does work that usually takes 5 seconds and has no
   deadline of its own, and runs `defer lock.Release(r.Context())`. Under load, the proxy in front
   times out requests at 25 seconds. Describe the two different ways this goes wrong, and fix both.
8. A fan-out helper recovers panics in each goroutine and sends the result as the goroutine's last
   statement. Its result type holds `err *LookupError`, and the function that reads the channel
   ends with `return res.v, res.err`, its result types being `(string, error)`. Its caller checks
   `if err != nil`. One day a lookup panics; another day every lookup succeeds. What does that
   caller see each time, and why?
