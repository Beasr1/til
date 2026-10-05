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
