# 9. Exercises and answers

Try each one before reading the answer. Most can be checked in a browser console or a scratch
React app in a few minutes, and that's better than reading the answer.

---

## File 01 — Foundations

**1. "The API returns `code: string`, it's in the types." What do you check, and why can't the compiler?**

Look at a real response: the network tab, a recorded fixture, or the server's schema or source. A
TypeScript type is a claim you wrote; the compiler checks your code against that claim, never the
claim against the server, because types are erased before the code runs (file 01 §1.2). The only
thing that can check a response is code that runs on the response — a schema parse or an adapter
at the API edge (file 05 §5.2).

**2. `{isLoading ? <Skeleton/> : <Filters/>}` and the user types. What happens, render by render?**

Render 1: `isLoading` is false, `<Filters>` mounts at that position with empty state. The user
types; `<Filters>`' state holds the text. The keystroke starts a fetch, so render 2 has
`isLoading` true: the position now holds `<Skeleton>`, a different component type, so React
discards `<Filters>` and its state. Render 3, when the fetch ends: `<Filters>` mounts again as a
new instance with empty state. The text, the focus and the cursor position are gone. Fix: render
`<Filters>` unconditionally and put the skeleton next to it, or over the results only.

**3. Classify, and say what goes wrong stored the other way.**

| Value | Kind | Stored the other way |
|---|---|---|
| Selected tab | Client | As server state, every tab click is a request and a loading state |
| Current user's role | Server | As client state, it's copied once and never refreshed, so a role change elsewhere is invisible until reload |
| Search box text | Client | As server state, each keystroke would round-trip; the *results* are server state, keyed by the text |
| Unread count | Server | As client state, it drifts as soon as anything happens in another tab or device |

**4. `{ nickname: undefined }` to mean "cleared". What does the server get?**

`JSON.stringify` drops properties whose value is `undefined`, so the body is `{}`. The server
can't tell "cleared" from "not mentioned", and under the usual patch conventions it leaves the
nickname as it was. To clear it, send `null` (or whatever the API defines), which survives
serialisation (file 05 §5.4).

**5. Refetching in a `useEffect` on `status` change. What's wrong?**

Server state is a cache (§1.4), and a cache entry should be identified by everything that
determines its contents. If `status` changes what the server returns, it belongs in the query
key; then a new `status` is simply a different entry and the library fetches it. The effect is
compensating for a key that's missing an input. It also leaves the cache wrong: the entry for the
old key now holds data for the new status, so anything else reading that key gets the wrong data
(file 02 §2.3).

---

## File 02 — Server state is a cache

**1. Key `['orders', page]`, but `queryFn` also reads `sortBy`. Change the sort: what happens?**

The key doesn't change, so `useQuery` returns the existing entry and doesn't fetch: the list
stays in the old order. (It may *eventually* change if a background refetch happens — on window
focus, say — and the refetch reads the new `sortBy`, which makes the bug look intermittent.)
The effect calling `refetch()` forces a fetch, so the screen updates, but the result is stored
under `['orders', page]` — the same entry as the old sort. Switching back to the old sort won't
fetch, prefetching is wrong, and any other component reading that key gets whichever sort ran
last. Put `sortBy` in the key and delete the effect.

**2. Renamed to `['invoice-summary', id]`; invalidation still says `['invoice']`. What happens?**

Prefix matching compares array elements: `'invoice'` and `'invoice-summary'` are different
strings, so `['invoice']` is not a prefix of `['invoice-summary', id]`. Nothing is invalidated;
the save succeeds; the screen shows old data until the entry goes stale and something triggers a
refetch. `invalidateQueries` matching zero queries is a valid outcome, not an error, so nothing
reports it. A shared key factory makes the rename a compile error instead.

**3. `enabled: false` query, invalidated after a mutation. Does it refetch?**

No. The docs say a disabled query ignores `invalidateQueries` and `refetchQueries`. Two fixes:
(a) after the mutation, call that query's own `refetch()` (it needs to be reachable from where
the mutation finishes); (b) better, make `enabled` a condition (`enabled: panelOpen`) — while
the panel is open the query is enabled, so invalidation refetches it, and while closed there's
nothing on screen to refresh.

**4. `!perms?.includes(need)` and an admin. Trace two renders and fix.**

Render 1: identity not decoded, `user` is null, the permissions query is disabled, `perms` is
undefined. `undefined?.includes(need)` is undefined, `!undefined` is true: `<AccessDenied/>`.
If that component redirects, the admin is gone. Render 2 (if they're still there): `user` is
set, the query starts, `perms` is still undefined: denied again. Only after the fetch would
`perms` be defined. Fix: three states.

```tsx
if (!authResolved || perms === undefined) return <Spinner />;  // not yet
if (!perms.includes(need)) return <AccessDenied />;            // no
return children;                                               // yes
```

For an admin who bypasses client permissions, decide that from the resolved identity before
looking at `perms` at all.

**5. `placeholderData: (prev) => prev` vs `staleTime: 5000`.**

`staleTime: 5000` makes cached entries count as fresh for five seconds. It doesn't stop the
blank: a *new* filter is a new key with no entry, so the first visit to each filter still shows
the loading state. What it does do is delay refetches of entries you've seen, so users can look
at five-second-old data. `placeholderData` leaves freshness alone and, for an entry with no data,
shows the previous key's data while fetching, without writing it into the cache. Its risk is the
mirror image: for a moment the old filter's numbers are on screen under the new filter's label.
Use `isPlaceholderData` to dim them or mark them as updating.

**6. Permission revoked in another tab. When can this tab find out?**

With the defaults, the permissions entry is stale immediately (`staleTime` 0), so it refetches
when a new component using it mounts, when the window regains focus, and when the network
reconnects. With §2.8's fixes it also refetches on every mount of a guarded page
(`refetchOnMount: 'always'`), and it's invalidated the first time any API call returns 403. The
one to rely on is the 403: it's the server saying, at the moment it matters, that the client's
copy is wrong. The others make the UI more accurate sooner; none of them is the control.

---

## File 03 — Tables over paginated APIs

**1. Client-side filter on page 1; the matches are on page 7.**

The table filters the 20 rows it holds. None match, so the user sees an empty table, probably
with a "no results" message. Nothing indicates a problem because client-side filtering did
exactly what it does — it can't know rows exist that it never received. Fix: `manualFiltering`
plus the filter in the request and query key.

**2. Stats from `fetchTransactions({ limit: 1000 })`.**

At 400 rows: correct — every row was fetched. At 1,000: correct by luck, and indistinguishable
from the next case. At 40,000: the counts describe whichever 1,000 rows the API returned first
(often the newest, or arbitrary without an `ORDER BY`), so "total" says 1,000 and every
breakdown is a sample presented as a census. The check: if `rows.length === limit`, you're
truncated; show that or keep paging. The real fix is a stats endpoint that counts on the server
with the same filters.

**3. Manual pagination, page 9, filter narrows to three results.**

The request goes out with the new filter and `pageIndex` 8, so the server returns page 9 of a
three-row result: nothing. The user sees an empty table on "page 9 of 1". The missing line is a
reset of `pageIndex` to 0 in the filter's change handler — `manualPagination` disables the
automatic reset, and a fully manual table has no client-side filter model to trigger one.

**4. Page→cursor map: visit 1, 2, 3, back to 1, change sort.**

| Step | Map |
|---|---|
| Visit page 1 (response gives cursor A) | `{1: start, 2: A}` |
| Visit page 2 (gives B) | `{1: start, 2: A, 3: B}` |
| Visit page 3 (gives C) | `{1: start, 2: A, 3: B, 4: C}` |
| Back to page 1 | unchanged — cursors are still valid for this query |
| Change sort | `{1: start}` — page 1 under the new sort; everything else is gone |

Without the reset, page 2 under the new sort would be fetched with cursor A — "after the row that
ended page 1 *in the old order*", applied to the new order. You'd get rows starting at an
arbitrary point, with some rows never shown and others shown twice as the user pages on.

**5. Why "page 3" after you've seen it, but not "page 9" before?**

Page *N*'s cursor only exists once page *N−1*'s response has handed it over. You've received
page 3's cursor (from page 2), so you can return to it; nobody has told you where page 9 starts.
For "Last", the server needs a total count (to know which page is last) and a way to fetch it —
typically by reversing the sort and taking the first page. A total count means counting every
matching row, which on a large filtered table is a full scan of the matching index range or
worse; that cost is a big part of why keyset APIs often omit totals.

---

## File 04 — Dates are not days

**1. Tokyo (UTC+9) user, `new Date(2026, 2, 8).toISOString()`.**

Local midnight in UTC+9 is 15:00 UTC the day before: `"2026-03-07T15:00:00.000Z"`. A New York
user on that date is at UTC−5 (DST starts later that day), so local midnight is
`"2026-03-08T05:00:00.000Z"`. If the server treats days as UTC, the Tokyo user's range gains the
last nine hours of 7 March and (at the other end) loses the last nine hours of their end day; the
New York user's range misses the first five hours of 8 March UTC and gains five hours after the
end. Each user silently sees a different set of rows for "the same" range.

**2. UTC-defined half-open bounds.**

```js
const from = new Date(Date.UTC(2026, 2, 8));   // 2026-03-08T00:00:00Z, inclusive
const to   = new Date(Date.UTC(2026, 2, 11));  // 2026-03-11T00:00:00Z, exclusive: covers the 10th
```

Query with `ts >= from AND ts < to`. Adding 86,400,000 ms is right for UTC days because UTC has
no daylight saving — every UTC day is 24 hours (leap seconds aside, which `Date` ignores). It's
wrong for New York because 8 March 2026 is 23 hours long there, so "start + 24 h" lands at 01:00
on the 9th.

**3. The offset-shifting formatter, viewed from New York and Tokyo.**

The shifted instant's local wall clock reads like the original instant's UTC wall clock, and
`toLocaleTimeString()` formats in the viewer's zone, so both viewers see (roughly) the UTC time
of the event. Near a DST change the New York viewer can see it an hour off, because the offset
was read at one instant and applied at another. The author probably expected "the time in a
specific zone" — which needs `Intl.DateTimeFormat` with `timeZone`, not arithmetic.

**4. `const end = range.to; end.setHours(…)` in a component.**

`end` and `range.to` are the same object. `setHours` changes it in place, so the value inside
React state is now 23:59:59.999 — but React only re-renders when you call the setter, and you
didn't. The picker keeps showing what it last rendered (the date looks unchanged), while the next
query sends the mutated value. If the function runs twice, nothing visibly changes (it's already
end-of-day), which hides the bug. Worse, if the same `Date` was also used as `from`, the start of
the range has moved to the end of the day.

**5. Santiago, 6 September 2026.**

Midnight doesn't exist that day in `America/Santiago`; clocks jump from 00:00 to 01:00. "Today" is
`[2026-09-06T01:00-03:00, 2026-09-07T00:00-03:00)` — 23 hours. Naive Go code
(`time.Date(2026, 9, 6, 0, 0, 0, 0, loc)`) returned 23:00 on 5 September (−04:00), so "today"
starts an hour early: orders placed between 23:00 and 24:00 on the 5th are counted as the 6th's,
and also as the 5th's if the 5th's interval is computed correctly. File 04 §4.4's
`startOfDay` detects the jump with `t.Day() != d` and moves to `ZoneBounds()`.

**6. Safari in November 2026: `Temporal.Now.plainDateISO()`?**

Not safe. MDN's compat data (November 2026) shows Temporal in Chrome, Edge, Firefox and Node,
but in Safari only in a beta, and web-features marks it not Baseline. Check MDN's compat table (or
your own analytics for Safari share) at the time, and either load a polyfill before use or feature-
detect (`typeof Temporal !== 'undefined'`) and fall back. For "today's date in a zone" without
Temporal, use `new Intl.DateTimeFormat('en', { timeZone, year: 'numeric', month: '2-digit',
day: '2-digit' }).formatToParts(new Date())` and read the `year`, `month` and `day` parts. (The
common shortcut of formatting with the `en-CA` locale gave `2026-03-09` in Node 26, but it relies
on that locale's display pattern, which isn't a contract.)

---

## File 05 — Forms and the shape of "no value"

**1. `code: 4`, options `"1"`–`"9"`, schema `z.string()`.**

The select compares the form value `4` (a number) with option values, which are strings, finds no
match, and shows the placeholder. If the user never touches it, the form value is still the number
`4`, and `z.string()` rejects it on submit with a type error the user can't see the cause of. The
fix goes in the single `toForm` adapter (`String(a.docType?.code ?? '')`) used by both
`defaultValues` and `reset`, with the inverse in `fromForm` — so there's one place to get it
right.

**2. `z.coerce.number().min(1)`, field cleared.**

The input's value is `""`. `Number("")` is `0`. The schema sees `0`, and `min(1)` fails — so here
validation happens to catch it, with a misleading message ("must be at least 1" rather than
"required"). With `.min(0)` or no minimum, an empty field would pass as a real zero. Change it so
`""` becomes `undefined` before the number check — `z.preprocess(v => v === '' ? undefined : v,
z.coerce.number().min(1))` — or keep the field as text and convert in your own commit step.

**3. Typing `-2` with `parseFloat(...) || 0`.**

Starting from `"0"` selected. Type `-`: the input's text is `"-"`; `parseFloat("-")` is `NaN`;
`NaN || 0` is `0`; state is `0`; the input re-renders as `"0"`. The minus sign is gone. Type `2`:
the text is now `"02"`; `parseFloat("02")` is `2`; the input shows `"2"`. The user wanted −2 and
got 2, with no error. Holding the raw string means every intermediate text (`"-"`, `"."`, `"1e"`)
is kept as typed, and parsing happens once, on commit, where an invalid value produces an error
instead of a silent substitution.

**4. `middleName` absent, `nickname: null`, `suffix: ""`, mapped with `x || ''`.**

The form holds `""` for all three, and the submit sends all three as `""`. Under Merge Patch:
`middleName: ""` *creates* the field with an empty value — a change the user didn't make;
`nickname: ""` *sets* a previously null field to empty — also a change, if the server distinguishes
null from empty; `suffix: ""` is unchanged. With the `Field<T>` type: `middleName` is
`absent` (omitted from the patch), `nickname` is `present` with `null` (send nothing, or `null`
only if the user changed it), `suffix` is `present` with `""`.

**5. Fill in the values.**

| Expression | Value | Why |
|---|---|---|
| `0 ?? 5` | `0` | `0` isn't null or undefined |
| `0 \|\| 5` | `5` | `0` is falsy |
| `NaN ?? 5` | `NaN` | `NaN` isn't null or undefined |
| `parseInt("0.9") \|\| 1` | `1` | `parseInt` stops at `.`, gives `0`, which is falsy |
| `parseFloat(undefined) ?? 1` | `NaN` | `parseFloat` returns `NaN` for a non-numeric input; `??` ignores it |

The surprising ones and their production inputs: `parseInt("0.9") || 1` — any fractional config
value like `P=0.9` silently became 1 (or the default); `parseFloat(undefined) ?? 1` — the variable
simply not being set.

**6. `Number(env.P) || 0.1`.**

`P=0` gives `0 || 0.1` = 0.1: you can't turn the feature off. `P=abc` gives `NaN || 0.1` = 0.1: a
typo is silently replaced by the default instead of failing. (Also `P=` gives `Number("")` = 0,
then 0.1 — arguably fine, but by accident.) The Go function separates three questions: is it set
(`LookupEnv`, empty check) → default only here; does it parse (`ParseFloat` error) → error if not;
is it valid (`IsNaN`, range) → error if not. `0` passes all three and survives; `abc` fails the
second and is reported.

---

## File 06 — Rendering traps

**1. Row `onClick`, menu items in a portal: why do both fire?**

In the DOM the menu item is a child of a `<div>` under `<body>`, nowhere near the `<tr>`. In the
React tree it's a descendant of `<Row>`, because `<Row>` rendered the `<Menu>` that rendered the
portal. React dispatches its events along the React tree, so the click bubbles from the item's
handler up to `<Row>`'s and runs `open` after `download`. The inspector shows the DOM tree, so the
author looked at the item, saw it wasn't inside the row, and concluded the event couldn't get
there — correct for native DOM listeners, wrong for React handlers.

**2. `stopPropagation` on the item, but the record opens when you click "…".**

The "…" trigger button isn't in a portal; it's inside the row in both trees. Clicking it bubbles to
the row through ordinary DOM-and-React bubbling and opens the record before the menu has even
appeared. The trigger needs its own `stopPropagation` (or fix 1 or 2 from §6.3, which cover it
automatically).

**3. `currentTarget.contains(target)` and the menu moves inline.**

No. Now the item *is* a DOM descendant of the row, so `contains` is true and the row opens the
record again. Fix 2 only distinguished portal clicks. Either keep the check and also skip clicks
that land on interactive elements (`(e.target as Element).closest('button, a, [role="menuitem"]')`),
or — better — use fix 1: no row-wide handler, a link on the primary cell. That design doesn't care
where the menu renders.

**4. Values survive, focus doesn't — and what a draft would lose.**

The values live in `OrdersPage`'s state, which stays mounted above the early return, so they
survive. Focus lives in the DOM: the `<input>` element is removed when `<PageSkeleton>` replaces the
tree and a new `<input>` is created afterwards, which has never been focused. If `<Filters>` kept
its own draft in `useState`, that state is attached to `<Filters>`' position, which disappears for
the skeleton render — so the draft would reset to empty (or to the prop) on every fetch, losing
whatever the user had typed but not yet submitted.

**5. `lastFetchedAt` in a ref, rendered as "Updated 2 minutes ago".**

Changing `ref.current` doesn't trigger a re-render, and reading it during render breaks the rule
react.dev states for refs. The first render reads the value once; later fetches update the ref but
nothing re-renders, so the label is frozen. The value *is* used for rendering, so it's state (or,
better, derived from the query: TanStack Query exposes `dataUpdatedAt` on the result). To make
"2 minutes ago" tick, you also need something that re-renders over time, such as an interval that
sets a `now` state.

---

## File 07 — Language traps in UI code

**1. `deepMerge({ ...DEFAULTS }, saved)` after org A saves.**

`{ ...DEFAULTS }` is a new outer object, but its `profile` property is the same object as
`DEFAULTS.profile`. The merge sees both sides of `profile` are objects and recurses with
`target = copy.profile`, i.e. `DEFAULTS.profile`, and assigns `photo = true` into it. `DEFAULTS` is
module-level, so it lives as long as the page (or the server process). Org B, with no saved
settings, gets `{ ...DEFAULTS }` — whose `profile` now says `photo: true`. Every org sees A's
change until reload. Fix: `deepMerge(structuredClone(DEFAULTS), saved)`, or a merge that returns new
objects at each level.

**2. Skips `__proto__` but not `constructor`.**

`{"constructor": {"prototype": {"isAdmin": true}}}`. The merge reads `target.constructor`, which for
a plain object is the `Object` function (inherited), recurses, reads `Object.prototype` through
`.prototype`, and assigns `isAdmin` onto it. (I reproduced exactly this in Node 26.) Without any key
list: merging into an `Object.create(null)` target (it has no inherited `constructor`), keeping
keyed data in a `Map`, or freezing `Object.prototype` would each have stopped it. Best of all, copy
only the keys your schema defines.

**3. `items.map(formatPrice)` and a new `currency` parameter.**

`map` calls its callback with `(element, index, array)`, so `currency` now receives the index: `0`,
`1`, `2`… Prices come out formatted with currency "0", "1", or throw, depending on how
`formatPrice` uses it (`Intl.NumberFormat` with an invalid currency code throws a `RangeError`). In
plain JavaScript nothing complains at the call site, because passing extra arguments is legal. In
TypeScript, `index: number` isn't assignable to `currency: string`, so you'd get a type error —
which is the good outcome. Fix: `items.map((i) => formatPrice(i))`.

**4. Deleting the case vs deleting from `RANGES`.**

Delete `case '90d'`: `r` in the `default` branch can still be `'90d'`, which isn't assignable to
`never` — compile error, pointing at the `switch`. Delete `'90d'` from `RANGES`: the `ChartRange` type
loses `'90d'`, the selector stops offering it, and `case '90d'` becomes a comparison TypeScript
rejects (`TS2678: Type '"90d"' is not comparable to type '"7d" | "30d"'`), so you remove the
dead case. I checked both directions with `tsc --strict`. Both directions are caught. With a hand-written options array of plain strings, the selector
isn't tied to `ChartRange`, so it can offer `'90d'` while the type doesn't contain it; the `switch` still
compiles, and you're back to the original bug — unless the options array is typed as `ChartRange[]`.

**5. `html.replace('<!-- slot -->', template)` with `Total: $&`.**

`$&` inserts the matched text, so the page gets `Total: <!-- slot -->` — the placeholder comment
instead of the amount. With `` $` `` it would have inserted all of the HTML before the slot.
Fixes: `html.replace('<!-- slot -->', () => template)` (replacer results aren't interpreted), or
`html.split('<!-- slot -->').join(template)` (which also replaces every occurrence, not just the
first — decide which you want). Escaping with `template.replaceAll('$', '$$$$')` works, but notice
that the replacement for one `$` has to be written as four, because `$$` in that call is itself a
pattern meaning one `$`. That's the kind of fix that's wrong on the first try.

---

## Things to try

1. **Break a key on purpose.** In a scratch app with TanStack Query and its Devtools, remove a
   filter from a query key and watch the Devtools while you change the filter. Then put it back
   and watch a new entry appear per filter value.
2. **Rename and watch nothing happen.** Rename a query key and leave the invalidation alone. Save
   something and watch the Devtools: no query turns stale. Then introduce a key factory and see
   the compiler find every site.
3. **Run file 04's examples in three zones.** `TZ=Asia/Tokyo node`, `TZ=America/New_York node`,
   `TZ=UTC node`: print `new Date(2026, 2, 8).toISOString()` in each, and
   `Temporal.PlainDate.from('2026-03-08').toZonedDateTime('America/New_York').hoursInDay`.
4. **Find a zone where midnight is skipped** in the coming year: loop over
   `Intl.supportedValuesOf('timeZone')` and the days of the year, and print every day whose
   `toZonedDateTime(zone)` doesn't start at `00:00`.
5. **Audit a codebase for `|| ` defaults.** `grep -nE '\|\| (0|\x27\x27|""|false)'` and classify each
   hit: is `0`/`""`/`false` a valid value there? If yes, it's a bug waiting for that value.

---

## Questions worth asking me

- "Walk me through the TanStack Query cache for this exact component — here's the Devtools state."
- "How do SWR or RTK Query express the same key and invalidation rules?"
- "When should a mutation update the cache directly (`setQueryData`) instead of invalidating?"
- "What does optimistic updating add to the failure modes in file 02?"
- "How would I do keyset pagination with a sort on a non-unique column, end to end?"
- "Show me `Intl.DateTimeFormat.prototype.formatToParts` for building day labels in a zone."
- "Has Safari shipped Temporal yet?" (File 04 §4.7 reflects November 2026.)
- "How do I model a PATCH that can set a field to `null` *and* remove it, when Merge Patch can't?"
- "Show me a merge utility that's safe against prototype pollution, and why."
- "Is my mental model right? Here's what I think happens: …"
