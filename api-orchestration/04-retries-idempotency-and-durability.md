# 4. Retries, idempotency and durability

Assumes [file 01](01-foundations.md) for the vocabulary and [file 02](02-graphs-and-scheduling.md)
for the scheduler that owns a run's state. Assumes
[go-services file 04](../go-services/04-deadlines-and-retries.md) for retrying a single call:
which failures are worth retrying, backoff with jitter, fitting retries inside a deadline, and
retrying at one layer. None of that is repeated here.

## 4.1 The problem

Take the running example: fetch a user; then fetch their orders and their preferences in
parallel; then post a summary to a CRM. The steps are `user`, `orders`, `prefs` and `crm`, and
only `crm` changes anything, because it's a POST.

The `crm` POST times out after ten seconds. Did the CRM create the summary? The engine can't
tell. The request may never have arrived, or the CRM may have created the record and the
response got lost on the way back. Retry, and the customer may get two summaries. Give up, and
they may get none. Now make it worse: halfway through the retry, a deploy kills the engine
process. When a new process starts, which steps of the run happened, which half-happened, and
which should run again?

[go-services file 04](../go-services/04-deadlines-and-retries.md) answers the retry question
for one call inside one request. This chapter is about what changes when the thing being
retried is one **step** inside a **run**. Three things change:

- **A run outlives the request, and often the process.** A step may be retried for minutes or
  days, by a process that didn't start the run.
- **A run has a past.** Earlier steps have produced results, and some have changed things in
  other systems. A retry has to work with that past rather than redo it.
- **Somebody else wrote the workflow.** The engine doesn't know which upstreams are safe to
  call twice. The workflow's author has to tell it.

| | One call inside a request ([go-services 04](../go-services/04-deadlines-and-retries.md)) | One step inside a run |
|---|---|---|
| Who's waiting | A caller, with a deadline | A caller in a request-style run; often nobody in a job-style one ([file 01](01-foundations.md)) |
| How long retries can go on | Seconds: whatever's left of the request | As long as the step's own bound allows, which can be hours |
| What came before | Nothing you'd lose by giving up | Earlier steps' results, and their side effects |
| What a crash loses | The request; the client can send it again | The run's progress, unless something wrote it down |
| What a retry repeats | The call | The step's call, and never the earlier steps |

## 4.2 Retry the step, not the run

The tempting implementation of "retry" is to start the run again. It repeats every step. For
`user`, `orders` and `prefs`, which are GETs, that's wasted work at best. At worst they return
different data the second time, because the user changed their preferences in between, and
`crm` posts a summary that matches neither attempt's view of the world. For any earlier step
with a side effect, starting again repeats the side effect.

So the unit of retry is the step. The scheduler from [file 02](02-graphs-and-scheduling.md)
keeps each finished step's result in the run's data document
([file 03 §3.2](03-mapping-data-between-steps.md#32-the-runs-data-document)), and a retry of
`crm` reuses `user`, `orders` and `prefs` exactly as they were.

The same reasoning gives a second rule: **build a step's request once, and send that same
request on every attempt.** A mapping that reads the clock or generates an identifier produces
a different body each time it's evaluated. Retrying a different request isn't a retry. It's a
new request, and §4.3 shows that an upstream deduplicating by key is entitled to reject it.
There are two ways to keep the body stable, and they work together: store the request the
step built and resend it, or, as [file 03](03-mapping-data-between-steps.md) recommends, put
any timestamp or id the request needs into the data document once, when the step starts, so
that evaluating the mapping again gives the same body.

### When a later step fails for good

Sometimes `crm` won't succeed however long you wait: the CRM rejects the summary as invalid.
Any earlier step that changed something stays changed. There's no transaction spanning
independent HTTP APIs, so the engine can't roll back. It has three options:

| Approach | What it means | What it costs |
|---|---|---|
| **Stop and report** | The run fails, and its record says which side effects happened | Someone has to clean up, by hand or in another system |
| **Compensate** | Run an undo step for each completed step, newest first | Every undo has to be written, and an undo can fail too |
| **Order the work** | Put irreversible steps last, so most failures happen before anything has changed | Not always possible |

The second is the **saga** pattern, from Hector Garcia-Molina and Kenneth Salem's
"Sagas" (SIGMOD 1987, [doi:10.1145/38713.38742](https://doi.org/10.1145/38713.38742); also a
[Princeton technical report](https://www.cs.princeton.edu/research/techreps/598)). Their saga
is a long-lived transaction "written as a sequence of transactions that can be interleaved with
other transactions", each with a compensating transaction, and the system guarantees that
either all of them complete or the completed ones are compensated. They were writing about a
single database system. Across HTTP APIs you keep the structure and lose the guarantee: a
compensation is a new action, not an undo. A refund isn't an un-charge, and an email can't be
unsent. The engine can record what happened and schedule the compensations. Only the
workflow's author can say what they are, and this is unsolved in any general sense.

The running example is the third case. `crm` is the only step with a side effect and it runs
last, so if it fails for good there's nothing to undo.

## 4.3 Retrying a POST safely: `Idempotency-Key`

RFC 9110 is direct about POST. PUT, DELETE and the safe methods are idempotent
([§9.2.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2.2)); POST and PATCH aren't.
And: "A client SHOULD NOT automatically retry a request with a non-idempotent method unless it
has some means to know that the request semantics are actually idempotent, regardless of the
method, or some means to detect that the original request was never applied."

Before reaching for a mechanism, sort failures by what they tell you:

| Failure | Did the request take effect? | Retry a POST? |
|---|---|---|
| Connection refused, DNS failure | No. Nothing reached the server | Yes |
| `429`, `503` | Almost certainly not: the server is refusing for now ([RFC 6585 §4](https://www.rfc-editor.org/rfc/rfc6585.html#section-4)) or "unable to handle the request" ([RFC 9110 §15.6.4](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6.4)). An intermediary can send them too | Usually, after `Retry-After` (§4.4) |
| Timeout or reset after the request was sent | Unknown | Only with a key the upstream honours |
| Cancelled by the engine after sending, by fail-fast or a run deadline ([file 02 §2.7](02-graphs-and-scheduling.md#27-failure-policy-fail-fast-or-carry-on)) | Unknown | Only with a key the upstream honours |
| `502`, `504` from a gateway | Unknown: the upstream behind it may have done the work | Only with a key the upstream honours |
| Other `4xx` | No, and the answer is final | No |

The ambiguous rows are the ones idempotency keys exist for.

### The mechanism

The client generates a unique key for each *logical* operation and sends it, unchanged, with
every attempt. The server stores the key with a fingerprint of the request and the outcome of
the first attempt. A repeat with the same key gets the stored outcome instead of running again.

The header has been written up as an IETF Internet-Draft,
[draft-ietf-httpapi-idempotency-key-header](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header).
**It is a draft, not a standard.** Its latest revision, -07, is dated 15 October 2025, and the
datatracker shows it as expired since 18 April 2026 (checked 19 October 2026). Its own
implementation-status section lists nine implementations that use a header of this name and ten
more that use the same idea under other header names or in the request body, so it describes a
widespread convention more than it defines one. What it says the server SHOULD do
(§2.6–2.7):

| Situation | Response |
|---|---|
| First request with this key | Process it normally |
| Repeat after the first finished | "the result of the previously completed operation, success or an error" |
| Repeat while the first is still running | `409 Conflict` |
| Same key, different payload | `422 Unprocessable Content` |
| Key missing on an operation that requires one | `400 Bad Request` |

The error bodies in the draft's examples are problem details
([file 06](06-describing-apis.md)). Note the `422` row: it's why §4.2 says to build the request
once. A body that changes between attempts, because a mapping stamped the time into it, turns a
safe retry into an error.

> ⚠️ **The draft's value is quoted.** It defines `Idempotency-Key` as a Structured Field
> String, so its examples read `Idempotency-Key: "8e03978e-40d5-43e8-bc93-6894a57f9324"`, quotes
> included. Many APIs that predate the draft document a bare token instead. Send what each
> upstream's own documentation says; the draft has no authority over an API that doesn't claim
> to follow it.

### Where the engine gets the key

The key must be the same for every attempt of one step in one run, *including attempts made by
a different process after a crash*, and different for every other step and run. The simplest
key with those properties is derived rather than stored:

```
idempotency key = <run id> "/" <step id>
```

Temporal's documentation recommends the same shape for its activities: "You can use a
combination of the Workflow Run ID and the Activity ID as an idempotency key since this is
guaranteed to be consistent across retry attempts but unique among Workflow Executions"
([Activity Definition](https://docs.temporal.io/encyclopedia/activities/activity-definition#idempotency)).

That key is unique only while each step runs once per run. This course doesn't cover steps that
run once per item of a list: [file 02 §2.10](02-graphs-and-scheduling.md#210-what-this-scheduler-doesnt-do)
lists fan-out over data among the things its scheduler doesn't do. If your engine has them, the
key needs a third part, and it should be the item's own identifier, not its position. A list
fetched again after a crash can come back in a different order, and a positional key would then
send one item's request under another item's key, which the upstream answers with the other
item's stored result.

Deriving beats generating a random key per step, because a derived key needs nothing written
down before the first attempt. If the engine crashes after sending and before recording
anything, the next process computes the same key (§4.6 walks through this). Two cautions. The
draft's security section warns that low-entropy keys let one client guess another's and read
its stored responses, so the run id should be random, such as a UUID, not a counter. And decide
what "retry this run" means in the product: *resuming* keeps the run id, so keys repeat and the
upstream deduplicates; *re-running* creates a new run with new keys, so a POST that secretly
succeeded the first time happens again.

### The part the engine can't fix

An idempotency key only works if the upstream implements it, and the engine can't find out
whether it does. RFC 9110 [§5.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.1):
"Other recipients SHOULD ignore unrecognized header and trailer fields." An upstream that has
never heard of `Idempotency-Key` drops it without a word, and the engine's retry creates a
duplicate while looking safe. The draft says the same from the other side: "Without a priori
knowledge, a general client cannot assume the server will respect this request."

So someone has to declare it. A sound default for an engine whose users type in the URLs
([file 05](05-calling-urls-you-dont-control.md)):

- Retry automatically only the ambiguous failures of methods that are idempotent by definition:
  GET, HEAD, OPTIONS, PUT and DELETE.
- Retry an ambiguous POST or PATCH failure only when the step is marked safe to repeat, because
  its upstream honours keys or the operation is naturally idempotent, like an upsert on a
  natural key ([postgres file 06](../postgres/06-upserts-and-duplicate-events.md)).
- Otherwise fail the step as *outcome unknown*, which is a different state from *failed*, and
  leave the decision to a person or a reconciliation step that can ask the upstream what
  happened.

## 4.4 Honouring `Retry-After`

A `503` or `429` often says when to come back. RFC 9110
[§10.2.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-10.2.3):

> Servers send the "Retry-After" header field to indicate how long the user agent ought to wait
> before making a follow-up request. When sent with a 503 (Service Unavailable) response,
> Retry-After indicates how long the service is expected to be unavailable to the client.

The value is either a number of seconds or a date:

```
Retry-After = HTTP-date / delay-seconds
delay-seconds  = 1*DIGIT

Retry-After: Fri, 31 Dec 1999 23:59:59 GMT
Retry-After: 120
```

RFC 6585 [§4](https://www.rfc-editor.org/rfc/rfc6585.html#section-4) adds that a `429` "MAY
include a Retry-After header indicating how long to wait before making a new request." And an
HTTP-date comes in three formats, of which a recipient "MUST accept all three"
([§5.6.7](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.6.7)): the preferred
`Sun, 06 Nov 1994 08:49:37 GMT`, the obsolete RFC 850 form `Sunday, 06-Nov-94 08:49:37 GMT`,
and C's `asctime()` form `Sun Nov  6 08:49:37 1994`, which carries no zone and is "assumed to be
in UTC".

For a step, honouring the header means four decisions:

1. **The server's value is a floor.** Never retry sooner than it asked.
2. **If it asks for more than the step has left, give up now** (§4.5 is about what "left"
   means). Sleeping for two minutes into a step that will time out in one wastes the minute and
   still fails.
3. **Keep the jitter, on top of the floor.** Every client that got
   `Retry-After: Mon, 19 Oct 2026 12:00:30 GMT` will come back at the same second, which is the
   synchronised wave that jitter exists to break up
   ([go-services 04, "Backoff, and why it needs jitter"](../go-services/04-deadlines-and-retries.md#backoff-and-why-it-needs-jitter)).
   RFC 9110 says nothing about this. Adding the usual backoff draw after the floor is a design
   choice.
4. **Ignore a value you can't parse**, and fall back to your own backoff.

Here's that as a function. It takes the attempt number, the raw header, the current time, the
budget left in the step, and a random source, so tests can make it deterministic.

```go
package retry

import (
	"errors"
	"math"
	"net/http"
	"strconv"
	"time"
)

// Backoff is capped exponential backoff with full jitter.
type Backoff struct {
	Base time.Duration // window for the first retry; must be > 0
	Cap  time.Duration // largest window
}

// window is min(Cap, Base·2^attempt), found by doubling so it can't overflow.
func (b Backoff) window(attempt int) time.Duration {
	w := b.Base
	for i := 0; i < attempt && w < b.Cap; i++ {
		w *= 2
	}
	return min(w, b.Cap)
}

// Delay is how long to wait before retry number attempt (0 for the first).
// retryAfter is the failed response's Retry-After value, or "" if it had none.
// budget is what's left of the step's total allowance. ok is false when the
// server asks for longer than that: give up now instead of sleeping first.
// rnd(n) must return a uniformly random value in [0, n).
func (b Backoff) Delay(attempt int, retryAfter string, now time.Time,
	budget time.Duration, rnd func(int64) int64) (d time.Duration, ok bool) {

	d = time.Duration(rnd(int64(b.window(attempt))))
	if floor, has := parseRetryAfter(retryAfter, now); has {
		if floor > budget {
			return 0, false
		}
		d += floor // never sooner than asked; the jitter spreads the herd
	}
	return min(d, budget), true
}

// parseRetryAfter reads RFC 9110 §10.2.3: delay-seconds or an HTTP-date.
func parseRetryAfter(v string, now time.Time) (time.Duration, bool) {
	if v == "" {
		return 0, false
	}
	// ParseUint, not Atoi: Atoi accepts "+5" and "-5".
	secs, err := strconv.ParseUint(v, 10, 32)
	if err == nil {
		return time.Duration(secs) * time.Second, true
	}
	if errors.Is(err, strconv.ErrRange) { // all digits, but over 136 years
		return math.MaxInt64, true
	}
	t, err := http.ParseTime(v) // accepts all three HTTP-date formats
	if err != nil {
		return 0, false // malformed: ignore it and use our own backoff
	}
	return max(t.Sub(now), 0), true
}
```

That's `retry.go` as I ran it, with Go 1.24.4. The table below comes from a separate `main`
that I've left out. It uses a 100 ms base and a 10 s cap, fixes "now" at
`2026-10-19T12:00:00Z`, and replaces the random draw with `n / 2`, the middle of the window, so
every line can be checked by hand:

```
attempt 0    retry-after ""                                 budget 3600000ms -> 50ms
attempt 1    retry-after ""                                 budget 3600000ms -> 100ms
attempt 2    retry-after ""                                 budget 3600000ms -> 200ms
attempt 6    retry-after ""                                 budget 3600000ms -> 3200ms
attempt 7    retry-after ""                                 budget 3600000ms -> 5000ms
attempt 40   retry-after ""                                 budget 3600000ms -> 5000ms
attempt 100  retry-after ""                                 budget 3600000ms -> 5000ms
attempt 0    retry-after "120"                              budget 3600000ms -> 120050ms
attempt 0    retry-after "120"                              budget 60000ms   -> give up
attempt 1    retry-after "Mon, 19 Oct 2026 12:00:30 GMT"    budget 3600000ms -> 30100ms
attempt 1    retry-after "Monday, 19-Oct-26 12:00:30 GMT"   budget 3600000ms -> 30100ms
attempt 1    retry-after "Mon Oct 19 12:00:30 2026"         budget 3600000ms -> 30100ms
attempt 1    retry-after "Mon, 19 Oct 2026 11:00:00 GMT"    budget 3600000ms -> 100ms
attempt 2    retry-after "+5"                               budget 3600000ms -> 200ms
attempt 2    retry-after "1.5"                              budget 3600000ms -> 200ms
attempt 2    retry-after "0x78"                             budget 3600000ms -> 200ms
attempt 2    retry-after " 7"                               budget 3600000ms -> 200ms
attempt 2    retry-after "99999999999"                      budget 3600000ms -> give up
attempt 7    retry-after ""                                 budget 3000ms    -> 3000ms
```

Read it in groups. The first seven lines are plain backoff: the window doubles from 100 ms and
stops at the 10 s cap, and attempts 40 and 100 are no different from attempt 7. The `"120"`
lines show the floor plus jitter, and the give-up when the step has only a minute left. The
three dates are the same instant in the three formats, 30 s after "now". A date in the past
counts as zero. The malformed values are ignored, and an absurdly long delay is a give-up. The
last line is a delay clipped to the budget. (`" 7"` can't arrive from a conforming HTTP library,
because "A field value does not include leading or trailing whitespace" and a parser "MUST
exclude such whitespace" (RFC 9110 [§5.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-5.5)),
but the function doesn't rely on that.)

The other three languages print an identical table. In each, I've left out the driver that
prints it, which is the same cases in the same order.

<details><summary>Rust</summary>

`Cargo.toml` depends on `httpdate = "1.0.3"`, for the three HTTP-date formats; the driver also
uses `rand = "0.10.3"`. Run with rustc 1.96.

```rust
use std::time::{Duration, SystemTime};

/// Capped exponential backoff with full jitter.
pub struct Backoff {
    pub base: Duration, // window for the first retry; must be > 0
    pub cap: Duration,  // largest window
}

impl Backoff {
    /// min(cap, base·2^attempt), found by doubling so it can't overflow.
    fn window(&self, attempt: u32) -> Duration {
        let mut w = self.base;
        for _ in 0..attempt {
            if w >= self.cap {
                break;
            }
            w *= 2;
        }
        w.min(self.cap)
    }

    /// How long to wait before retry number `attempt` (0 for the first).
    /// `None` means the server asked for longer than `budget`: give up now.
    /// `rnd(n)` must return a uniformly random value in `0..n`.
    pub fn delay(
        &self,
        attempt: u32,
        retry_after: Option<&str>,
        now: SystemTime,
        budget: Duration,
        rnd: &mut impl FnMut(u64) -> u64,
    ) -> Option<Duration> {
        let mut d = Duration::from_nanos(rnd(self.window(attempt).as_nanos() as u64));
        if let Some(floor) = retry_after.and_then(|v| parse_retry_after(v, now)) {
            if floor > budget {
                return None;
            }
            d += floor; // never sooner than asked; the jitter spreads the herd
        }
        Some(d.min(budget))
    }
}

/// Retry-After (RFC 9110 §10.2.3): delay-seconds or an HTTP-date.
fn parse_retry_after(v: &str, now: SystemTime) -> Option<Duration> {
    // Check the digits first: str::parse::<u32>() accepts "+5".
    if !v.is_empty() && v.bytes().all(|b| b.is_ascii_digit()) {
        let secs = v.parse::<u32>().map_or(u64::MAX, u64::from); // over 136 years
        return Some(Duration::from_secs(secs));
    }
    let t = httpdate::parse_http_date(v).ok()?; // all three HTTP-date formats
    Some(t.duration_since(now).unwrap_or(Duration::ZERO))
}
```

In production the driver passes `&mut |n| rand::random_range(0..n)`. Stable Rust's standard
library has no random number generator: `std::random` exists in rustc 1.96 but only behind the
unstable `random` feature, so the `rand` crate it is.

</details>

<details><summary>C++</summary>

C++20, compiled with Apple clang 17 (`-std=c++20 -Wall -Wextra -Wpedantic`). No library: the
standard one has what's needed, just about.

```cpp
#pragma once
#include <algorithm>
#include <charconv>
#include <chrono>
#include <cstdint>
#include <ctime>
#include <functional>
#include <iomanip>
#include <locale>
#include <optional>
#include <sstream>
#include <string>

namespace retry {
using std::chrono::milliseconds, std::chrono::seconds, std::chrono::sys_seconds;

// rnd(n) must return a uniformly random value in [0, n).
using Rnd = std::function<std::int64_t(std::int64_t)>;

// An HTTP-date in any of RFC 9110's three formats, as UTC. libc++ has no
// std::chrono::parse yet, so this uses std::get_time and the C++20 calendar.
inline std::optional<sys_seconds> parse_http_date(const std::string& v, sys_seconds now) {
    namespace ch = std::chrono;
    struct Format { const char* spec; bool two_digit_year; };
    static constexpr Format formats[] = {
        {"%a, %d %b %Y %H:%M:%S GMT", false}, // IMF-fixdate
        {"%A, %d-%b-%y %H:%M:%S GMT", true},  // obsolete RFC 850
        {"%a %b %d %H:%M:%S %Y", false},      // obsolete asctime
    };
    for (const auto& f : formats) {
        std::tm tm{};
        std::istringstream in(v);
        in.imbue(std::locale::classic());
        in >> std::get_time(&tm, f.spec);
        if (in.fail() || in.peek() != EOF) continue; // get_time ignores trailing text
        int y = tm.tm_year + 1900;
        if (f.two_digit_year) { // RFC 9110's 50-year rule, not %y's fixed cut-off
            int this_year = int(ch::year_month_day{ch::floor<ch::days>(now)}.year());
            y = this_year - this_year % 100 + y % 100;
            if (y > this_year + 50) y -= 100;
        }
        ch::year_month_day date{ch::year{y}, ch::month(tm.tm_mon + 1), ch::day(tm.tm_mday)};
        if (!date.ok()) continue; // get_time accepts 31 Feb
        return ch::sys_days{date} + ch::hours{tm.tm_hour} + ch::minutes{tm.tm_min} +
               seconds{tm.tm_sec};
    }
    return std::nullopt;
}

// Retry-After (RFC 9110 §10.2.3): delay-seconds or an HTTP-date.
inline std::optional<milliseconds> parse_retry_after(const std::string& v, sys_seconds now) {
    if (v.empty()) return std::nullopt;
    std::uint32_t secs = 0;
    const char* end = v.data() + v.size();
    auto [p, ec] = std::from_chars(v.data(), end, secs); // stops at the first non-digit
    if (p == end && ec == std::errc{}) return seconds{secs};
    if (p == end && ec == std::errc::result_out_of_range) return milliseconds::max();
    if (auto t = parse_http_date(v, now)) return std::max<milliseconds>(*t - now, milliseconds::zero());
    return std::nullopt; // malformed: ignore it and use our own backoff
}

// Capped exponential backoff with full jitter.
struct Backoff {
    milliseconds base; // window for the first retry; must be > 0
    milliseconds cap;  // largest window

    // min(cap, base·2^attempt), found by doubling so it can't overflow.
    milliseconds window(int attempt) const {
        milliseconds w = base;
        for (int i = 0; i < attempt && w < cap; ++i) w *= 2;
        return std::min(w, cap);
    }

    // How long to wait before retry number `attempt` (0 for the first).
    // nullopt means the server asked for longer than `budget`: give up now.
    std::optional<milliseconds> delay(int attempt, const std::string& retry_after,
                                      sys_seconds now, milliseconds budget, const Rnd& rnd) const {
        milliseconds d{rnd(window(attempt).count())};
        if (auto floor = parse_retry_after(retry_after, now)) {
            if (*floor > budget) return std::nullopt;
            d += *floor; // never sooner than asked; the jitter spreads the herd
        }
        return std::min(d, budget);
    }
};
} // namespace retry
```

In production the driver wraps a `std::mt19937_64` in a `std::uniform_int_distribution`.

</details>

<details><summary>JavaScript</summary>

An ES module, run with Node 24. Durations are plain numbers of milliseconds.

```js
// Capped exponential backoff with full jitter. Durations are milliseconds.
// base: window for the first retry (> 0); cap: largest window.
export function backoffWindow(attempt, { base, cap }) {
  let w = base;
  for (let i = 0; i < attempt && w < cap; i++) w *= 2; // not base << attempt: << is 32-bit
  return Math.min(w, cap);
}

// How long to wait before retry number `attempt` (0 for the first).
// null means the server asked for longer than `budget`: give up now.
// rnd(n) must return a uniformly random integer in [0, n).
export function retryDelay(attempt, retryAfter, now, budget, policy, rnd) {
  let d = rnd(backoffWindow(attempt, policy));
  const floor = parseRetryAfter(retryAfter, now);
  if (floor !== null) {
    if (floor > budget) return null;
    d += floor; // never sooner than asked; the jitter spreads the herd
  }
  return Math.min(d, budget);
}

// Retry-After (RFC 9110 §10.2.3): delay-seconds or an HTTP-date.
// `now` is milliseconds since the epoch, like Date.now().
export function parseRetryAfter(v, now) {
  if (!v) return null;
  // Not Number(v) alone: it accepts "0x78", "1e3", " 7" and "".
  if (/^\d+$/.test(v)) return Number(v) * 1000;
  const t = parseHttpDate(v, now);
  return t === null ? null : Math.max(t - now, 0);
}

const MON = { Jan: 0, Feb: 1, Mar: 2, Apr: 3, May: 4, Jun: 5, Jul: 6, Aug: 7, Sep: 8, Oct: 9, Nov: 10, Dec: 11 };
const M = Object.keys(MON).join("|");
const WD = "(?:Mon|Tue|Wed|Thu|Fri|Sat|Sun)";
const T = "(?<h>\\d\\d):(?<mi>\\d\\d):(?<s>\\d\\d)";
const HTTP_DATE = [
  new RegExp(`^${WD}, (?<d>\\d\\d) (?<mo>${M}) (?<y>\\d{4}) ${T} GMT$`), // IMF-fixdate
  new RegExp(`^(?:Mon|Tues|Wednes|Thurs|Fri|Satur|Sun)day, (?<d>\\d\\d)-(?<mo>${M})-(?<y>\\d\\d) ${T} GMT$`), // RFC 850
  new RegExp(`^${WD} (?<mo>${M}) (?<d>[ \\d]\\d) ${T} (?<y>\\d{4})$`), // asctime
];

// Date.parse reads asctime dates as local time and rolls 31 Feb into March,
// so match the three formats exactly and build the instant with Date.UTC.
function parseHttpDate(v, now) {
  for (const re of HTTP_DATE) {
    const g = re.exec(v)?.groups;
    if (!g) continue;
    let y = Number(g.y);
    if (g.y.length === 2) { // RFC 9110's 50-year rule for two-digit years
      const thisYear = new Date(now).getUTCFullYear();
      y += thisYear - (thisYear % 100);
      if (y > thisYear + 50) y -= 100;
    }
    const day = Number(g.d);
    const t = Date.UTC(y, MON[g.mo], day, Number(g.h), Number(g.mi), Number(g.s));
    return new Date(t).getUTCDate() === day ? t : null;
  }
  return null;
}
```

In production the driver passes `(n) => Math.floor(Math.random() * n)`.

</details>

Five differences between the languages turned up while writing these, each of which breaks a
naive version.

> ⚠️ **Integer parsers accept more than `1*DIGIT`.** Go's `strconv.Atoi("+5")` returns 5. Rust's
> `"+5".parse::<u64>()` returns `Ok(5)`. In JavaScript, `Number("0x78")` is 120,
> `Number("1e3")` is 1000, `Number(" 7")` is 7, `Number("")` is 0, and `parseInt("120abc")` is
> 120. C++'s `std::from_chars` rejects the sign but stops at the first non-digit, so `"1.5"`
> parses as 1 unless you check where it stopped. Each version above checks for digits only.

> ⚠️ **JavaScript's `Date.parse` reads the asctime format as local time.** With
> `TZ=Asia/Kolkata`, `Date.parse("Mon Oct 19 12:00:30 2026") - now` was −19,770,000 ms: five and
> a half hours in the past instead of 30 s in the future, so the step would retry at once. It
> also turned `"Sun, 31 Feb 1994 …"` into 3 March, and `Date.parse("120")` into the year 120,
> which is why the digits are checked before any date parsing.

> ⚠️ **`base << attempt` breaks on a step's long horizon.** The one-liner in
> [go-services 04](../go-services/04-deadlines-and-retries.md#backoff-and-why-it-needs-jitter),
> `rand.Int64N(int64(min(maxWait, base<<attempt)))`, is fine for the handful of attempts a
> request allows. A step with unlimited attempts, Temporal's default (§4.5), gets further.
> With a 100 ms base, Go's `base<<37` overflows to a negative duration, `min` picks it, and
> `rand.Int64N` panics with "invalid argument to Int64N". From attempt 63 the shift gives 0,
> which panics too. With a 10 s cap, 37 waits add up to at most about five minutes. In
> JavaScript `<<` works on 32-bit integers: `100 << 25` is −939,524,096 and `1 << 32` is 1. In
> Rust, `base << 64` panicked with "attempt to shift left with overflow" in a debug build and
> returned `base` unchanged in a release build, which masks the shift amount. In C++ a shift by
> the type's width or more is undefined behaviour. Doubling in a loop that stops at the cap
> avoids all four.

> ⚠️ **HTTP-date parsers disagree at the edges.** For an RFC 850 date, RFC 9110 says a two-digit
> year that "appears to be more than 50 years in the future" belongs to the previous century.
> Given `Thursday, 06-Nov-70 08:49:37 GMT`, Go's `http.ParseTime` returned 1970. Rust's
> `httpdate` 1.0.3 also reads `70` as 1970 and then rejected the date, because it checks the
> weekday and 6 November 1970 was a Friday; Go doesn't check the weekday at all. The C++ and
> JavaScript versions above apply the RFC's rule and return 2070, which was a Thursday. None of
> this matters for `Retry-After` in practice, because a sender "MUST generate those timestamps
> in the IMF-fixdate format".

> ⚠️ **Only Go makes a reproducible test easy without injection.** Go's `math/rand/v2`
> top-level functions are seeded for you, and `rand.New(rand.NewPCG(1, 2))` gives the same
> draws on every run (615.498615 ms for attempt 3, twice). JavaScript's `Math.random` can't be
> seeded at all: "The implementation selects the initial seed … it cannot be chosen or reset by
> the user" ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)).
> C++ engines are fully specified, but "The algorithms for producing each of the specified
> distributions are implementation-defined" ([rand.dist.general]/3,
> [draft standard](https://eel.is/c++draft/rand.dist.general)), so a seeded
> `uniform_int_distribution` gives different numbers under libc++ and libstdc++. That's why
> `rnd` is a parameter.

One more caution about the date form. An HTTP-date is compared with the engine's clock, and
delay-seconds isn't. If the engine's clock runs behind the server's, it waits too long; if
ahead, it comes back early and collects another `503`. Delay-seconds is the more robust form
for a server to send.

## 4.5 One attempt, or the whole sequence?

A retried step needs two time limits, and the commonest mistake is having only one of them.

- A limit on **each attempt**, so a hung connection is abandoned and retried rather than
  waited on for ever.
- A limit on **the whole sequence** of attempts and waits, so a step against a dead upstream
  eventually fails instead of retrying until someone notices.

Temporal names both, and its documentation is precise about them
([Detecting Activity failures](https://docs.temporal.io/encyclopedia/detecting-activity-failures)):

| Timeout | Temporal's definition | Default |
|---|---|---|
| **Start-To-Close** | "the maximum time allowed for a single Activity Task Execution", "applied to each Activity Task Execution within an Activity Execution" | None of its own: "If you set only the Schedule-To-Close Timeout, the Temporal Service uses that value as the Start-To-Close Timeout" |
| **Schedule-To-Close** | "the maximum amount of time allowed for the overall Activity Execution, from when the first Activity Task is scheduled to when the last Activity Task, in the chain of Activity Tasks that make up the Activity Execution, reaches a Closed status" | ∞ |
| Schedule-To-Start | How long one attempt may wait in a task queue before a worker picks it up; "enforced for each Activity Task" and "non-retryable by design" | ∞ |

An activity must have one of the first two: "If you set neither timeout, the Temporal Service
rejects the Activity." And Temporal says which matters more: "We strongly recommend setting a
Start-To-Close Timeout," because "The Temporal Server doesn't detect failures when a Worker loses
communication with the Server or crashes. Therefore, the Temporal Server relies on the
Start-To-Close Timeout to force Activity retries." The per-attempt limit is how a crashed worker
is noticed at all.

```
time ──►
├── attempt 1 ──┤ wait ├── attempt 2 ──┤  wait  ├── attempt 3 ──┤   wait   ├─ ... ┤
│◄ Start-To-   ►│      │◄ Start-To-   ►│        │◄ Start-To-   ►│          │      │
│   Close       │      │   Close       │        │   Close       │          │      │
│◄────────────────────────── Schedule-To-Close ─────────────────────────────────►│
│                                                                                 │
first attempt scheduled                                          the step fails here
```

The retry policy sits between them. Its defaults, from
[Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies): initial interval 1 s,
backoff coefficient 2.0, maximum interval 100 × the initial interval, maximum attempts
unlimited, and no non-retryable errors. Activities get this policy by default; "Workflow
Executions do not retry by default." The page recommends bounding retries by time rather than
by count: "we recommend using the Workflow Execution Timeout for Workflows or the
Schedule-To-Close Timeout for Activities to limit the total duration of retries, rather than
using this attribute" (maximum attempts).

Two details on that page connect to §4.4. The retry interval is "the smaller of" the initial
interval times the coefficient "raised to the power of the number of retries" and the maximum
interval. That formula has no random component, and the page doesn't mention jitter; I couldn't
find it stated either way. And an activity can fail with a "next Retry delay" that "will replace
and override whatever the retry interval would be", but "your retries will still cap out under
the Retry Policy's Maximum Attempts, as well as overall timeouts". That is `Retry-After`'s shape
exactly: the upstream names the delay, and the bound on the whole sequence still wins.

For an engine of your own, give every step both numbers, and make each attempt's timeout the
smallest of three things: the step's per-attempt timeout, the time left in the step, and, when a
caller is waiting on a request-style run, the time left in the run. That's go-services' rule,
[derive the timeout from what's left](../go-services/04-deadlines-and-retries.md#add-up-the-worst-case),
applied at two levels. The time left in the step is the `budget` argument in §4.4's `Delay`,
which is what lets it give up instead of sleeping past the end.

## 4.6 How much of a run survives a crash

The engine process will die mid-run: a deploy, an out-of-memory kill, a lost node. What
happens next is a design choice, and the options form a spectrum. Each level keeps more of the
run and costs more to build or operate.

| Level | Where a run's state lives | After a crash | What it needs | What it gives up |
|---|---|---|---|---|
| **1. In-process** | Memory, in goroutines | On restart, mark the runs that were running as failed, "interrupted" | A `runs` row with a status and an owner | Every in-flight run, and any record of whether its side effects happened |
| **2. Checkpoint each step** | A table of step results, written as each step finishes | Resume each unfinished run from its last recorded step; the step in flight runs again | A results table, a way to claim unfinished runs, idempotency keys | Exactly-once execution of the step that was in flight |
| **3. Durable-execution library on Postgres** | The library's tables in your Postgres, for example DBOS Transact | The library resumes each workflow from its last completed step | The library, and deterministic workflow code | Freedom to structure the orchestration code however you like |
| **4. Workflow service** | A separate cluster's event history, for example Temporal | The service reschedules work; workers replay the history | The service, its workers, and deterministic workflow code | Simplicity: a second system to run, and its size limits (§4.7) |

### Level 1: in-process, and honest about it

The run lives in the process. The scheduler in
[file 02](02-graphs-and-scheduling.md#210-what-this-scheduler-doesnt-do) is at this level: "All
run state lives in one goroutine's memory." When the process dies, so does every run in flight.
What you can still do is not lie about it. On startup, find the runs this instance owned that
are still marked running, and mark them failed with a reason, so nobody waits for ever on a run
that no process is executing.

With more than one engine instance, "the runs that were running" isn't enough: some belong to
instances that are alive. Each run needs an owner and a heartbeat, a timestamp the owner
refreshes, and a run whose owner hasn't refreshed it for a while is presumed abandoned. That's
a **lease**, and every level from here up has one, whether you write it or a library does.

Level 1 is a reasonable choice for request-style runs ([file 01](01-foundations.md)), where a
caller is waiting and an error reaches someone who can act on it within seconds. It's a poor one
for a run that takes an hour.

### Level 2: checkpoint each step's result

After each step finishes, write its result to the database: the run's data document, one step
at a time. After a crash, a surviving instance claims the unfinished run through its lease,
reloads the results, skips the steps that are recorded, and runs the rest.

The gap is between "the step's call returned" and "its result was recorded". The upstream's
database and yours are two different databases, and no transaction spans both. A crash in that
window means the step runs again on resume:

```mermaid
sequenceDiagram
    participant E as Engine
    participant C as CRM (upstream)
    participant D as Engine's database
    E->>C: POST /summaries, Idempotency-Key: run-7f3a/crm
    C->>C: create the summary
    C-->>E: 201 Created
    rect rgb(255, 225, 225)
    Note over E: process dies before the result is written
    end
    Note over E,D: a new process resumes run-7f3a; crm has no recorded result
    E->>C: POST /summaries, Idempotency-Key: run-7f3a/crm
    C-->>E: 201 Created, the stored response; no second summary
    E->>D: record crm's result
```

This is where §4.3's derived key earns its keep. The resumed process computes the same key
without having stored anything, and the CRM recognises the repeat. With a random key generated
per attempt, the second POST would look new and create a second summary.

### Level 3: a durable-execution library on Postgres

Writing level 2 properly means writing the leases, the resume loop and the checkpointing, and
testing them against crashes at every point. A durable-execution library does it for you inside
your process. [DBOS Transact](https://github.com/dbos-inc/dbos-transact-golang) for Go (v1.7.0,
15 October 2026, per the Go module proxy) describes itself as "lightweight durable workflow
orchestration on top of Postgres". You register ordinary Go functions as workflows, wrap each
side-effecting call in `dbos.RunAsStep`, and "If your program ever fails, when it restarts all
your workflows will automatically resume from the last completed step." There's "no separate
orchestration server and no infrastructure required besides Postgres"
([architecture](https://docs.dbos.dev/architecture)).

The rules come with it, in the documentation's own words:

- "The workflow function must be deterministic: if executed multiple times, with the same
  arguments and step return values, the workflow should invoke the same steps with the same
  inputs in the same order."
- "Steps are tried at least once but are never re-executed after they complete"
  ([workflow tutorial](https://docs.dbos.dev/golang/tutorials/workflow-tutorial)), and so "Steps
  should be idempotent, meaning it should be safe to retry them multiple times."
- "An assigned workflow ID acts as an idempotency key: if a workflow is called multiple times
  with the same ID, it executes only once."

For an engine that runs user-defined workflows, the mapping is one library workflow per run,
with the run id as its workflow ID, and one step per workflow step. The workflow function is
your interpreter, and determinism is a constraint on it. Given the same definition and the same
recorded results it must make the same calls in the same order, so a run has to keep the
version of the definition it started with, even if someone edits the workflow while it runs.

### Level 4: a workflow service

[Temporal](https://docs.temporal.io/) moves the state out of your database and into a separate
service. Your code runs in workers. The service records an event history for each workflow
execution, and after a crash a worker rebuilds the workflow's state by replaying that history.
A run maps to a workflow execution, and each step to an activity, with §4.5's timeouts and
retry policy.

The determinism rule is stricter and spelled out: "Workflow code must be deterministic to
support replay", and calls that produce commands, such as scheduling an activity, "must not be
reordered, added, or removed without proper Versioning techniques"
([Workflow Definition](https://docs.temporal.io/workflow-definition#deterministic-constraints)).
Pinning the definition version per run matters here for the same reason as at level 3.

What you buy is that the service survives your process, and that retries, timers and timeouts
are kept by something other than the code that failed. What you pay is a second distributed
system to operate, or to buy, and the limits in §4.7.

> **Teacher's aside.** People pick a durable-execution system expecting exactly-once side
> effects. What it gives is exactly-once *recording*. Temporal's guarantee, in its own words: "the
> Activity will be observed as completed exactly once. However, the Activity may be executed
> multiple times and may even partially complete more than once during this process"
> ([Activity Definition](https://docs.temporal.io/encyclopedia/activities/activity-definition#activity-retry-policy)).
> DBOS says its steps "are tried at least once". Neither is a weakness of the product. The
> window in the sequence diagram above exists at every level, because the fact "the CRM created
> the summary" lives in the CRM's database and the fact "we know it did" lives in yours, and
> nothing commits both at once. The only thing that turns at-least-once execution into a single
> effect is the upstream recognising the repeat, by a key it honours or a natural key it upserts
> on. Temporal again: idempotency keys "are enforced by the service you are calling from your
> Activity, not by the Activity itself." So choosing a durability level never answers "is this
> POST safe to repeat?" It only decides how often the question comes up.

## 4.7 Large payloads and the claim check

Suppose `orders` returns 30 MB of order history. At level 2 that's a 30 MB value in a results
row, read back on every resume. At level 4 it's an activity result in the event history, and
the service has limits on those. From Temporal's documentation
([self-hosted defaults](https://docs.temporal.io/self-hosted-guide/defaults),
[Cloud limits](https://docs.temporal.io/cloud/limits),
[blob size limit](https://docs.temporal.io/troubleshooting/blob-size-limit-error)):

| Limit | Value |
|---|---|
| One payload: an activity's input or result, a workflow's input or result | Errors at 2 MB; self-hosted warns at 256 KB. Configurable when self-hosted, fixed at 2 MB on Temporal Cloud |
| One gRPC message between workers and the service | 4 MB |
| One event-history transaction | 4 MB |
| One workflow execution's whole history | 51,200 events or 50 MB; warnings at 10,240 events or 10 MB |

The gRPC limit catches fan-outs: "A Workflow can hit this limit even when every individual
payload is under 2 MB. Scheduling several Activities with moderate-sized inputs, or hundreds of
Activities with tiny inputs in the same Workflow Task can push the combined request past 4 MB."
That's `orders` and `prefs` scheduled together, at scale.

The fix is old. Gregor Hohpe and Bobby Woolf's *Enterprise Integration Patterns* (2003) calls it
the [Claim Check](https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html):
"Store message data in a persistent store and pass a Claim Check to subsequent components." For
an engine, that means writing the large response body to an object store and putting a small
reference into the run's data document instead:

```json
{
  "orders": {
    "$claim": "blob://runs/run-7f3a/orders/body",
    "sha256": "9f2c…",
    "bytes": 31457280,
    "contentType": "application/json"
  }
}
```

The hash lets whoever fetches it check they got the right bytes. That shape is illustrative, not
a standard. Temporal's storage drivers, for instance, return a claim that is "a set of key-value
pairs the driver uses to locate the payload later".

Temporal now builds this in as
[External Storage](https://docs.temporal.io/external-storage), currently in public preview
("APIs and configuration may change before General Availability"). It "offloads payloads to an
external store … and passes a small reference token through the Event History instead", for
payloads at or above a threshold that defaults to 256 KiB.

What a claim check doesn't solve is lifecycle, and this is where the honest state of things is
"your problem". Temporal: "Temporal does not automatically delete payloads from your external
store. Payloads can also be orphaned if a request fails after the upload completes." Its rule
for the expiry is `TTL > Maximum Workflow Run Timeout + Namespace Retention Period`, and for a
workflow with no run timeout, "there is no finite TTL that guarantees safety." Microsoft's
[Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check)
page lists the rest: decide which component owns deletion; write the payload before publishing
the reference, never after; expect duplicate references when retries are involved; and keep the
store available for as long as any reference to it might be read.

Two consequences are specific to an engine. A mapping ([file 03](03-mapping-data-between-steps.md))
can't evaluate an expression over a body it can't see, so the engine either fetches claimed
bodies before evaluating, which brings the size problem back into memory, or exposes only part
of them. And the claim check also keeps data out of the run's history and logs, which can be
the reason to use it even for small payloads that hold sensitive data.

## Check yourself

1. The `crm` POST times out after ten seconds. List what the engine knows and doesn't know
   about the CRM's state. What has to be true for an automatic retry to be safe?
2. An engine rebuilds the `crm` request body from the run's data on every attempt, and the
   mapping writes the current time into the body. The CRM implements the Idempotency-Key draft.
   What does the second attempt get back, and what should the engine change?
3. A step has a 10 s per-attempt timeout and a 60 s bound on the whole step. Backoff starts at
   1 s and doubles, with no jitter, and the upstream hangs on every call. How many attempts
   start, and when does the step fail? What changes if the engine checks the 60 s bound only
   after an attempt finishes, and gives every attempt its full 10 s?
4. A thousand runs call the same upstream and all get `503` with
   `Retry-After: Mon, 19 Oct 2026 12:00:30 GMT`. Describe the upstream's load at 12:00:30 if
   every engine sleeps exactly until then, and what §4.4's `Delay` does differently. Would
   `Retry-After: 30` have avoided the problem?
5. The engine checkpoints each step's result after the step returns. It crashes after the CRM
   answered `201` but before the checkpoint committed. Walk through the restart with the key
   derived from run id and step id, then with a random key generated on each attempt. Then say
   what changes if an operator presses "re-run" instead of "resume".
6. `orders` returns a 30 MB body in a workflow running on Temporal. What fails, and where?
   With a claim check in place, who should delete the stored body, and what's the earliest
   time it's safe to?

(Answers in [09-exercises.md](09-exercises.md).)
