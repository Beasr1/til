# 9. Worker pools

## The problem

Chapter 5 fanned out a handful of independent calls inside a request. A worker pool is the
same idea at batch scale: a list of hundreds of items, a fixed number of goroutines, and a
result per item. Pools are usually written once, put in a helper package, and reused
everywhere, so their quirks spread. Three quirks show up again and again: results that come
back in a different order from the inputs, errors that are logged and then dropped, and one
deadline shared by work that's still waiting its turn. Each one turns into wrong data rather
than a crash, which is why they last.

Assumes chapter 1 (goroutines, channels, `context`) and chapter 5 (`errgroup`). The outputs
below come from a small program that copies the shape of a real pool; names are invented.

## The pool in question

```go
func pool[T any](ctx context.Context, workers int, tasks []T, fn func(T) error) error {
    ch := make(chan T, len(tasks))
    errCh := make(chan error, len(tasks))
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        go func() {
            for t := range ch {
                if err := fn(t); err != nil { errCh <- err }
                wg.Done()
            }
        }()
    }
    for _, t := range tasks { wg.Add(1); ch <- t }
    close(ch)
    wg.Wait()
    close(errCh)
    select {
    case err, ok := <-errCh:
        if !ok { return nil }
        log.Println("error processing:", err)
        return nil                     // ← the caller never sees it
    case <-ctx.Done():
        return ctx.Err()
    default:
        return nil
    }
}
```

Callers collect results through a closure: `mu.Lock(); results = append(results, r);
mu.Unlock()`.

## Results arrive in completion order

`append` under a mutex records results in the order tasks *finish*. Any code that then pairs
`results[i]` with `inputs[i]` is pairing by coincidence. Measured, two inputs where the first
is slower:

```
paired by index: input[0]=item-A  <->  result-of-item-B
paired by index: input[1]=item-B  <->  result-of-item-A
```

And with one failure, every later pair shifts by one:

```
pool saw error, logs it and returns nil: extract failed for item-A
paired by index: input[0]=item-A  <->  result-of-item-B
paired by index: input[1]=item-B  <->  result-of-item-C
```

Nothing fails. Each result is a valid result, attached to the wrong input. With one input per
batch, or inputs that happen to take equal time, tests pass every time.

The fix is to never let order carry meaning:

| Fix | How |
|---|---|
| Write to a slot | Pre-size `results := make([]R, len(inputs))` and have task `i` write `results[i]`. No mutex needed, since each index has one writer |
| Carry the key | Put the input's identity in the result (`Result{InputID: …}`) and join on it |
| Use `errgroup` | `g.Go(func() error { results[i], err = fn(ctx, inputs[i]); return err })`, which is the slot pattern with error handling (chapter 5) |

> ⚠️ When a result carries *some* of its identity (one field set from the input after the
> call) but the caller reads the rest from `inputs[i]`, the record is half right. A field like
> "source" can be correct while "id" and "path" belong to a different input. Review every place
> where a result and an input meet, and check they meet by key.

## A pool that logs and returns nil

The `select` at the end reads at most one error, logs it, and returns `nil`. The caller can't
tell a batch where every task failed from one where none did. In the measurement above, the
caller got `nil` with one of three inputs failed.

When that caller is a queue consumer, `nil` means "commit". So the failed item's message is
committed without being processed or dead-lettered: at-most-once delivery for exactly the
messages that needed attention. (See [kafka/03](../kafka/03-offsets-and-delivery-guarantees.md),
"Committing regardless of outcome".)

Two smaller problems in the same lines:

- The `default` branch is never taken: `errCh` is closed by then, so the first case is always
  ready. It isn't a bug, but it suggests the author expected a non-blocking check, and readers
  will too.
- A panic in `fn` kills the process, because nothing recovers in the worker goroutines, and if
  it were recovered, `wg.Done()` would be skipped and `wg.Wait()` would hang. Put `wg.Done()` in
  a `defer` if you ever add recovery.

What a pool should return is a decision, not an accident: the first error (and cancel the
rest, which is `errgroup`), every error joined (`errors.Join`), or a per-item result that the
caller inspects. "Log one, return nil" is none of these.

## One deadline, shared by work still in the queue

A pool, or a semaphore around goroutines, limits concurrency. A deadline created *once* for
the whole batch is then shared by tasks that are running and tasks that haven't started.
Measured: six calls of 100 ms each, at most two at a time, under one 300 ms deadline:

```
chunk 5 ok at 100ms
chunk 0 ok at 100ms
chunk 1 ok at 200ms
chunk 2 ok at 200ms
chunk 4 failed at 300ms: context deadline exceeded
chunk 3 failed at 300ms: context deadline exceeded
```

The last two calls were healthy. They failed because they spent their budget waiting for a
slot. Raising the deadline doesn't scale: the total time is roughly `tasks ÷ concurrency ×
per-task time`, and a deadline sized for today's batch is too small for next month's.

The usual repair is a deadline per task, started when the task starts, plus an overall cap.
But a child context can't outlive its parent. From the same run:

```
child asked for 10s, its deadline is 300ms from start
```

`context.WithTimeout(parent, d)` gives the earlier of the parent's deadline and `d`. A helper
that computes "overall cap = per-task timeout × 10" inside a function whose caller already
passed a 20-second context has a cap of 20 seconds, whatever the multiplier says.

Also notice that `sem <- struct{}{}` doesn't watch the context. A task that's still waiting for
a slot when the deadline passes keeps waiting, then starts, and fails immediately. Acquire with
a `select` on `ctx.Done()` so a cancelled task gives up its place in the queue, or use
`errgroup.SetLimit` with tasks that check their context first, so work started after
cancellation returns at once.

> **Teacher's aside.** People reason about concurrency limits as if they only slow things
> down. They also change *when* each task starts, and anything measured from the start of the
> batch, a deadline, a timestamp, an ordering, now means something different for task 1 and
> task 600. When you add a limit, look for every value computed once before the loop and ask
> whether it was meant per task.

## Check yourself

1. A pool processes two files per record and appends `(digest, err)` results under a
   mutex. The caller stores `results[i]` against `files[i].path`. Why did every test pass, and
   under what production conditions are paths and digests swapped?
2. Given the pool above, a batch of ten messages where four fail. What does the caller see, and
   what does a queue consumer that commits on `nil` do with the four?
3. Rewrite the pool's contract so the caller can tell success, partial failure and total
   failure apart. What would you return, and why not just the first error?
4. 1,000 chunks, 5 at a time, each takes 200 ms, one 20-second context for the batch. How many
   chunks succeed? What changes if each chunk gets its own 20-second timeout derived from that
   context?
5. Why does `sem <- struct{}{}` before `select { case <-ctx.Done(): … }` waste work after
   cancellation, and how does `errgroup.SetLimit` avoid it?
6. A worker recovers panics with `defer recover()` but calls `wg.Done()` at the end of the loop
   body. What happens to the batch when one task panics?
