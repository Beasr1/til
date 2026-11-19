# 4. Dates are not days

Assumes file 01 (§1.5 covers what JSON does to a `Date`). The server-side cousin — clocks and
zones inside a container — is [go-services/07-time-in-a-container.md](../go-services/07-time-in-a-container.md).

## 4.1 The problem

Three bugs, one root:

- A date-range filter. The user picks 8 March to 10 March. The picker hands back `Date`
  objects for the start and end of those local days, the code sends `.toISOString()`, and for a
  user in Tokyo (UTC+9) the range arrives as `2026-03-07T15:00:00Z` to `2026-03-10T14:59:59.999Z`.
  To a server that counts days in UTC, nine hours of the 7th are in and nine hours of the 10th are
  out.
- A formatter meant to show times "in the office's zone". It took the epoch milliseconds, added
  `getTimezoneOffset()` minutes, and formatted the result. Every time it showed was the UTC wall
  clock, not the office's.
- A "due today" count on a server. It computed midnight with `now.setHours(0, 0, 0, 0)`, which
  used the container's zone (UTC, not the users') and also overwrote `now`, so every later
  comparison against "now" was really against midnight.

The root: **a calendar day is not a moment in time.** "8 March" is a label, and which stretch of
the timeline it labels depends on where you're standing.

## 4.2 Four different things called "a date"

Imagine a phone call between Tokyo and New York. There's one **instant** when it starts — both
people agree on it. But one of them writes "Sunday 21:00" in their diary and the other writes
"Sunday 08:00", and near midnight they'd write different *days*. The instant is shared; the
day and the time on the wall are local.

| Concept | Example | Has a zone? | JavaScript type today | Temporal type |
|---|---|---|---|---|
| **Instant** | the call started at `2026-03-08T12:00:00Z` | No — it doesn't need one | `Date` | `Temporal.Instant` |
| **Calendar date** | 8 March 2026 | No — which is why it's ambiguous | none (people misuse `Date`) | `Temporal.PlainDate` |
| **Wall-clock date-time** | 8 March 2026, 08:00 | No | none | `Temporal.PlainDateTime` |
| **Zoned date-time** | 08:00 on 8 March 2026 in `America/New_York` | Yes, an IANA zone | none | `Temporal.ZonedDateTime` |

The `Date` row is the trap in the name. MDN: "`Date` objects encapsulate an integral number that
represents milliseconds since the midnight at the beginning of January 1, 1970, UTC" — an
instant. "The local timezone is not stored in the date object", but "the basic methods to fetch
the date and time or its components all work in the local (i.e., host system) time zone"
([MDN, Date](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date)).
So `getDate()` answers "what day is this instant *on the machine running the code*" — the
user's laptop in the browser, usually UTC on a server.

An **IANA zone** is a name like `Asia/Tokyo` or `America/New_York` that stands for a region's
whole history of offsets and daylight-saving rules. An **offset** like `-05:00` is one moment's
answer. New York is `-05:00` in winter and `-04:00` in summer; the zone knows when it switches,
the offset doesn't.

## 4.3 First decide which zone defines the day

Before writing code, answer: when a user says "8 March", whose 8 March?

| Data | The day is defined by | Why |
|---|---|---|
| Logs and events written by machines in UTC | UTC | Everyone debugging agrees on boundaries; show "UTC" on screen |
| A user's own activity ("what did I do today?") | The user's zone | Matches their memory |
| A business's operations ("today's orders at the shop") | The business's zone | The shop's day doesn't move when the viewer travels |

There's no universally right answer; there's a wrong one, which is "whatever zone the code
happens to run in". The bugs in §4.1 all let the runtime decide.

## 4.4 Build day boundaries from year, month and day

Once you know the zone, a day is the **half-open** interval from the first instant of that day
to the first instant of the next: `[start, nextStart)`. Half-open, rather than ending at
`23:59:59.999`, because it never misses a sliver at the end and because consecutive days tile
with no gap or overlap.

Build those instants from the calendar parts, in the chosen zone:

| The day is defined by | Start of 8 March 2026 | In JavaScript |
|---|---|---|
| UTC | `2026-03-08T00:00:00Z` | `Date.UTC(2026, 2, 8)` (months count from 0) |
| The browser's zone | local midnight | `new Date(2026, 2, 8)` |
| A named zone | midnight there, as an instant | Temporal, or a tz library (below) |

Don't compute the end as `start + 24 hours`. On 8 March 2026 New York moves its clocks forward,
so that day is 23 hours long — I measured it in Node 26, both with `Date` and with Temporal's
`hoursInDay`. And in a few zones the clocks change *at* midnight, so midnight doesn't exist that
day. `America/Santiago` springs forward at 00:00 on 6 September 2026; Temporal says that day
starts at 01:00. Go's `time.Date` asked for midnight there returned 23:00 *on 5 September* —
its docs say that for a skipped time "the choice of time zone, and therefore the time, is not
guaranteed" ([Go, time.Date](https://pkg.go.dev/time#Date)). The Go version below handles it:

```go
// startOfDay returns the first instant of calendar day y-m-d as observed in loc.
func startOfDay(y int, m time.Month, d int, loc *time.Location) time.Time {
	// Normalise first, so that day 32 of March becomes 1 April.
	y, m, d = time.Date(y, m, d, 12, 0, 0, 0, time.UTC).Date()
	t := time.Date(y, m, d, 0, 0, 0, 0, loc)
	if t.Day() != d {
		// Midnight doesn't exist on this day (a DST gap at 00:00) and Go chose a time on
		// the day before. The day really starts when the new offset begins.
		_, t = t.ZoneBounds()
	}
	return t
}

// dayBounds returns the half-open interval [start, end) covering y-m-d in zone.
func dayBounds(y int, m time.Month, d int, zone string) (start, end time.Time, err error) {
	loc, err := time.LoadLocation(zone)
	if err != nil {
		return time.Time{}, time.Time{}, err
	}
	return startOfDay(y, m, d, loc), startOfDay(y, m, d+1, loc), nil
}
```

I ran it with Go 1.24: New York, 8 March gives `[05:00Z, 04:00Z next day)`, 23 hours; Santiago,
6 September gives `[01:00 −03, 00:00 −03 next day)`, 23 hours, matching Temporal.

<details><summary>Rust</summary>

```rust
use chrono::{DateTime, NaiveDate, TimeZone};
use chrono_tz::Tz;

/// Half-open [start, end) for the calendar day in `zone`.
/// None if midnight is skipped by a DST change (see the Go version for the fix).
fn day_bounds(y: i32, m: u32, d: u32, zone: Tz) -> Option<(DateTime<Tz>, DateTime<Tz>)> {
    let day = NaiveDate::from_ymd_opt(y, m, d)?;
    let next = day.succ_opt()?;
    let start = zone.from_local_datetime(&day.and_hms_opt(0, 0, 0)?).earliest()?;
    let end = zone.from_local_datetime(&next.and_hms_opt(0, 0, 0)?).earliest()?;
    Some((start, end))
}
```

Run with chrono 0.4 and chrono-tz 0.10: New York 8 March works; Santiago 6 September returns
`None`, because a skipped local time has no mapping.

</details>

<details><summary>C++</summary>

```cpp
#include <chrono>
#include <utility>
using namespace std::chrono;

// Half-open [start, end) for the calendar day ymd in the named zone (C++20).
// choose::earliest: if midnight falls in a DST gap, use the moment the gap ends.
std::pair<sys_seconds, sys_seconds> dayBounds(year_month_day ymd, const char* zone) {
    const time_zone* tz = locate_zone(zone);
    local_days day{ymd};
    auto start = tz->to_sys(local_seconds{day}, choose::earliest);
    auto end   = tz->to_sys(local_seconds{day + days{1}}, choose::earliest);
    return {start, end};
}
```

Not compiled here: Apple's libc++ doesn't ship the C++20 time-zone database, so check this on
GCC 14 or MSVC before relying on it.

</details>

<details><summary>JavaScript</summary>

```js
// With Temporal (see §4.7 for where it ships).
function dayBounds(isoDate, zone) {
  const day = Temporal.PlainDate.from(isoDate);          // "2026-03-08"
  const start = day.toZonedDateTime(zone);               // first instant, even if 00:00 is skipped
  const end = day.add({ days: 1 }).toZonedDateTime(zone);
  return [start.toInstant(), end.toInstant()];
}

// Without Temporal, for a UTC-defined day only:
const start = new Date(Date.UTC(2026, 2, 8));
const end = new Date(Date.UTC(2026, 2, 9));
```

</details>

> ⚠️ `new Date("2026-03-08")` is **UTC** midnight, while `new Date("2026-03-08T00:00")` is
> **local** midnight. MDN: "date-only forms are interpreted as a UTC time and date-time forms are
> interpreted as a local time", a "historical spec error" kept for compatibility. In New York,
> `new Date("2026-03-08").getDate()` is 7.

## 4.5 Never shift the milliseconds to fake a zone

The formatter from §4.1:

```ts
// WRONG: tries to "remove the browser's offset" so the time shows as some other zone
const shifted = new Date(d.getTime() + d.getTimezoneOffset() * 60_000);
format(shifted, 'HH:mm');
```

This builds a *different instant* whose local wall clock happens to read like UTC's. Two things
go wrong:

1. It shows the UTC wall clock, not the zone you wanted. Unless the target zone is UTC, every
   time is off by that zone's offset.
2. It isn't even reliably UTC. `getTimezoneOffset()` is read at the original instant, but the
   shifted instant can sit on the other side of a daylight-saving change. Running in New York, an
   instant at `06:30Z` on 8 March 2026 came out as 07:30. I measured that in Node 26.

The fix is to keep the instant as it is and ask for it to be *displayed* in a zone. That's what
the `timeZone` option of `Intl.DateTimeFormat` is for; it takes IANA names. It's defined in
ECMA-402, the internationalisation companion to the language spec
([ECMA-402](https://tc39.es/ecma402/); readable version at
[MDN, Intl.DateTimeFormat](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/DateTimeFormat#timezone)):

```ts
const fmt = new Intl.DateTimeFormat('en-GB', {
  timeZone: 'Asia/Tokyo',
  dateStyle: 'medium',
  timeStyle: 'long',              // includes the zone name
});
fmt.format(new Date('2026-03-08T12:00:00Z'));   // "8 Mar 2026, 21:00:00 GMT+9" (Node 26)
```

If you're using date-fns, its time-zone companion `date-fns-tz` provides `formatInTimeZone`, which
does the same thing with date-fns format tokens. Its README says that "for most use cases this
is the only function from this library you will need"
([date-fns-tz](https://github.com/marnusw/date-fns-tz)). The same library's `toZonedTime` returns
a `Date` "which will format as the local time of any time zone": a shifted instant on purpose,
meant for feeding a date picker that only understands local time. It's the §4.1 hack, done
correctly, for one narrow job. Don't send its result to a server or compare it with real
instants.

> **Teacher's aside.** The mental model to unlearn is "a `Date` has a zone and I can change it".
> It doesn't. A `Date` is a number of milliseconds. Any arithmetic on that number moves you to a
> different moment in history. Zones belong to *formatting* (instant → text) and to *building*
> (calendar parts + zone → instant), never to the stored number.

## 4.6 A `Date` is mutable — and React won't notice

`setHours` "changes the `Date` object in place, and returns its new timestamp"
([MDN, setHours](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/setHours)).
Both halves of that bite:

```ts
const [range, setRange] = useState<{ from: Date; to: Date }>(initial);

function toQuery() {
  const end = range.to;            // same object as the one in state
  end.setHours(23, 59, 59, 999);   // mutates the state value; no re-render, no warning
  return { from: range.from.toISOString(), to: end.toISOString() };
}
```

React only re-renders when you call the setter with a new value, so the picker keeps showing the
old date while the state object now holds a different one. On a server, `now.setHours(0,0,0,0)`
does the same to a variable the rest of the function still believes is "now". And because it
returns a number, `const midnight = now.setHours(0,0,0,0)` gives you a timestamp, not a `Date`.

The rule: treat a `Date` as a value. Build a new one (`new Date(y, m, d)`, `Date.UTC(...)`) rather
than calling a setter on one you didn't just create. Temporal's types are immutable, which removes
the problem.

Finally, **label every displayed time with its zone**. "16:00" is ambiguous on a page that some
people read in Tokyo and some in London; "21:00 GMT+9" or "12:00 UTC" isn't. `timeStyle: 'long'`
or `timeZoneName: 'short'` in `Intl.DateTimeFormat` adds it.

## 4.7 Temporal — the fix, and where it ships

Temporal is the TC39 proposal that adds the types in the §4.2 table: immutable, with real
calendar dates and IANA zones. It reached **Stage 4** at TC39's March 2026 meeting, and the
finished-proposals list gives 2027 as its expected publication year
([tc39/proposals](https://github.com/tc39/proposals/blob/main/finished-proposals.md)).

Shipping, from MDN's browser-compat-data 8.1.9 (12 November 2026):

| Runtime | Temporal since |
|---|---|
| Firefox | 139 (May 2025) |
| Chrome, Edge | 144 (January 2026) |
| Node.js | 26.0.0 (May 2026) |
| Safari | Only in Safari 27.3, which is still a beta |

The web-features data marks it as **not Baseline**, because Safari hasn't shipped it in a release.
So in November 2026, a site with Safari users can't rely on a global `Temporal` without a polyfill.
The proposal's README lists several; when I checked, it marked `temporal-polyfill` as a stable
release and `@js-temporal/polyfill` as alpha
([proposal-temporal, Polyfills](https://github.com/tc39/proposal-temporal#polyfills)). Recheck MDN
before quoting this table; it'll change as Safari releases.

What Temporal fixes directly: `PlainDate` makes "8 March" a type that can't be confused with an
instant; `ZonedDateTime` carries its IANA zone, so day arithmetic respects DST (`hoursInDay` is 23
on New York's spring-forward day); and nothing has a setter.

## Check yourself

1. A user in Tokyo (UTC+9) picks "8 March" and the code sends `new Date(2026, 2, 8).toISOString()` as the
   start of the range. What string does the server receive? What string would it receive from a
   user in New York, and which rows does each user silently miss or gain?
2. Rewrite the range bounds for a UTC-defined day using only `Date.UTC`, as a half-open
   interval. Why is "end = start + 86,400,000 ms" right for UTC days but wrong for New York days?
3. A formatter does `new Date(ms + new Date(ms).getTimezoneOffset() * 60000)` and formats with
   `toLocaleTimeString()`. A viewer in New York and one in Tokyo look at the same event. What does
   each see, and what did the author probably expect?
4. Explain why `const end = range.to; end.setHours(23,59,59,999)` inside a React component
   corrupts state without causing a re-render. What does the picker show afterwards, and what
   does the next query send?
5. A shop's "today's orders" count runs on a server in UTC for a shop in `America/Santiago`. On 6
   September 2026, what interval should "today" cover? What does naive "local midnight in
   Santiago" code in Go produce, and which orders does it misfile?
6. Your app must support Safari in November 2026. Is it safe to write `Temporal.Now.plainDateISO()`?
   What would you check, and what would you ship instead?
