# 4. Deadlines and retries

## The problem

A dependency slows down. Your service retries, sensibly. Your caller retries too, and so
does its caller. Within a minute the dependency is receiving several times its normal
traffic, from requests whose original users gave up long ago, and it never recovers until
someone turns things off. Retries are the commonest way a small outage becomes a large
one. The fix is not "don't retry". It's to retry only what can succeed, only while someone
is still waiting, and only at one layer.

## A deadline belongs to the request, not to the call

A request arrives with an implicit budget: the caller's timeout, the user's patience,
the load balancer's idle limit. Everything the handler does should fit inside that budget,
and stop when it's used up. In Go, the budget travels as the request's **context**:

```go
func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()
    person, err := h.people.Fetch(ctx, id)       // honours the deadline
    ...
}
```

Two common breaks in the chain:

- **A client helper that makes its own context.** `context.WithTimeout(context.Background(),
  30*time.Second)` inside the helper gives each call thirty seconds regardless of how much
  of the request's budget is left, and keeps going after the request is cancelled.
- **A retry loop given `context.Background()`.** Its `select` on `ctx.Done()` looks like
  cancellation support. With a background context it can never fire.

Either way, work continues after nobody is waiting for it.

## Add up the worst case

Here's a configuration that looks reasonable piece by piece:

| Setting | Value |
|---|---|
| Outbound call timeout | 30 s |
| Retry attempts | 5 |
| Backoff | 100 ms doubling, capped at 10 s |
| Retry "max elapsed time" | 60 s |
| Server `WriteTimeout` | 30 s |

Now add up the worst case: five attempts that each time out after 30 s, with backoff in
between, is more than 150 s. The elapsed-time cap doesn't save you if it's only checked
*between* attempts. An attempt that starts at 59 s runs to 89 s. Meanwhile the server
stopped being able to write a response at 30 s (chapter 3). So for over two minutes the
service does work whose result can't be delivered, and the client sees a bare connection
error at the end.

The rule: **the retry budget must fit inside the caller's deadline.** The simplest way to
make that true is to derive every attempt's timeout from the remaining context deadline
rather than from a fixed number, and to check the deadline *before* starting an attempt,
not just after one fails.

## Retry only what can succeed

A retry is worth trying when the failure is transient: a timeout, a connection reset, a
`503`, a `429` (after honouring `Retry-After`). It's pointless, and adds load, when the
answer is final:

| Response | Retry? |
|---|---|
| Connection refused, reset, timeout | Yes |
| `502`, `503`, `504` | Yes |
| `429 Too Many Requests` | Yes, after the `Retry-After` delay |
| `400`, `401`, `403`, `404`, `409`, `422` | No. The same request will get the same answer |
| `500` | Depends. Often a bug, which won't fix itself |

A retry helper whose `RetryableErrors` field exists but is never consulted retries a
`404 Not Found` five times, with backoff, for every lookup of something that doesn't exist.

And retry only operations that are safe to repeat. `GET` is. A `POST` that creates
something isn't, unless the server deduplicates with an idempotency key, because the
"failed" first attempt may have succeeded and only the response was lost.

## Backoff, and why it needs jitter

**Exponential backoff** (wait 100 ms, then 200, 400 and so on, up to a cap) gives a
struggling dependency room to recover. On its own it has a flaw: every client that failed
at the same moment retries at the same moments too, so the dependency gets synchronised
waves of traffic. **Jitter**, a random component in each wait, spreads them out. Marc
Brooker's [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
works through the variants with simulations. "Full jitter", waiting a random time between
zero and the exponential value, is a good default.

```go
wait := time.Duration(rand.Int64N(int64(min(maxWait, base<<attempt))))
```

Two small bugs are common in hand-written backoff loops. Check yours for both:

- **The first wait is already doubled.** If the loop computes `next = current * 2` before
  the first sleep, an "initial interval" of 100 ms actually waits 200 ms first.
- **The loop sleeps after the last attempt**, or reports the final error as a timeout.

## Retry at one layer

If each layer of a call chain retries three times, a single failing request at the bottom
of a chain *n* layers deep is attempted up to 3<sup>*n*</sup> times: 9 for two layers, 27
for three, 243 for five. The arithmetic is the whole argument. Brooker's Builders' Library
article [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
covers the practice in depth. The usual conclusion is to retry at one layer, usually the
one closest to the failure that still knows whether the operation is safe to repeat, and
let the others fail fast.

> **Teacher's aside.** A retry is a bet that the next attempt will go differently. That's
> a good bet against a dropped packet and a bad one against a dependency that's
> overloaded, because your retry is part of the overload. This is why the better systems
> cap retries as a *fraction of traffic* (a retry budget or token bucket) rather than per
> request. When everything is failing, they stop retrying almost entirely.

## Check yourself

1. A handler's context is `r.Context()`, but it calls a helper that internally uses
   `context.Background()`. The user closes the tab after one second. List everything that
   keeps running, and for how long.
2. Using the configuration table in this chapter, compute the longest a single handler
   can run, and say what the client receives.
3. A lookup returns `404` for a record that doesn't exist yet. The retry helper retries
   all errors. Describe the load effect during a burst of lookups for new records.
4. Why isn't exponential backoff alone enough when a thousand clients fail at the same
   instant? What does full jitter change?
5. A request passes through an API gateway, a service and a client library, each
   retrying up to three times. How many attempts can reach the database for one user
   click? Where would you keep retries, and why there?
6. Your outbound call is a `POST` that creates a payment. It timed out. Under what
   conditions is it safe to retry?
