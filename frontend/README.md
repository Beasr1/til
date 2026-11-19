# Frontend Data Handling — A Course

A course on the bugs a web frontend gets from the data it handles: copies of server data that
go stale, pages mistaken for whole tables, timestamps mistaken for days, form conversions
that lose the difference between "zero", "empty" and "not there", and the React and JavaScript
rules underneath them that fail without an error. It's written for someone who
knows a backend language well and has picked up React along the way, without ever being taught
how the pieces think.

**This is reference learning material.** It's about React, TanStack Query and Table, the
JavaScript `Date` and Temporal APIs, and the HTML standard, not about any particular app. The
examples are neutral on purpose: orders, invoices, a transactions table, a config flag.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Every chapter opens with the problem the idea exists to solve.
- Every behavioural claim was checked against a spec, the library's docs, or measured (Node 26,
  Go 1.24, November 2026), and the text says which. Where I couldn't confirm something, the text
  says that too.
- Language-neutral examples are in Go, with Rust, C++ and JavaScript versions folded underneath.
  React- and browser-specific examples are TypeScript, because they have no Go equivalent.
- Every file ends with **Check yourself** questions. Answers are in `09-exercises.md`.

## The primary sources

| Source | What it's for | Link |
|---|---|---|
| **react.dev** | Rendering, state and its position in the tree, portals, refs | <https://react.dev/learn>, <https://react.dev/reference/react-dom/createPortal> |
| **TanStack Query docs** | Query keys, invalidation, `enabled`, dependent queries, placeholder data, defaults | <https://tanstack.com/query/latest/docs/framework/react/overview> |
| **TanStack Table docs** | Client- vs server-side processing, `manual*` options, pagination | <https://github.com/TanStack/table/tree/main/docs> |
| **Use The Index, Luke — No Offset** | Why offset pagination drifts and slows | <https://use-the-index-luke.com/no-offset> |
| **MDN Web Docs** | `Date`, `Intl.DateTimeFormat`, `??`, spread, `structuredClone`, `String.prototype.replace`, `stopPropagation` | <https://developer.mozilla.org/en-US/docs/Web/JavaScript> |
| **ECMA-402** | The `Intl` spec, including the `timeZone` option | <https://tc39.es/ecma402/> |
| **TC39 Temporal proposal** | The new date-time types; status and polyfills | <https://github.com/tc39/proposal-temporal> |
| **MDN browser-compat-data / web-features** | Where Temporal ships, and its Baseline status | <https://github.com/mdn/browser-compat-data>, <https://github.com/web-platform-dx/web-features> |
| **date-fns-tz** | Zone-aware formatting for date-fns | <https://github.com/marnusw/date-fns-tz> |
| **WHATWG HTML** | `<input type="number">`: sanitisation, bad input, `min`/`max` | <https://html.spec.whatwg.org/multipage/input.html> |
| **Zod** | Schema coercion and preprocessing | <https://zod.dev/api> |
| **React Hook Form** | `defaultValues`, `reset`, `values` | <https://react-hook-form.com/docs/useform> |
| **RFC 7396** | JSON Merge Patch: what `null` and absence mean | <https://www.rfc-editor.org/rfc/rfc7396> |
| **OWASP Authorization Cheat Sheet** | Why client-side permission checks aren't the control | <https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html> |
| **OWASP Prototype Pollution Prevention Cheat Sheet** | What prototype pollution is and which defences OWASP recommends | <https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html> |
| **TypeScript Handbook** | Erased types; `never` and exhaustiveness checking | <https://www.typescriptlang.org/docs/handbook/2/narrowing.html> |

Support statements are dated and come from browser-compat-data 8.1.9 (12 November 2026). They
will go stale; recheck MDN before quoting one.

## Reading order

### Part 1 — Foundations (file 01 is the prerequisite for everything else)

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [Foundations](01-foundations.md) | Say where React keeps state and when it throws it away, tell client state from server state, and name what JSON and inputs do to "no value" |
| 6 | [Rendering traps](06-rendering-traps.md) | Predict which tree an event or a piece of state follows through a portal, place loading states so inputs survive, and choose refs over state for values that never render |

File 06 was added later and keeps its number; read it straight after file 01.

### Part 2 — Server data

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [**Server state is a cache**](02-server-state-is-a-cache.md) | ⭐ Write query keys that can't go stale, invalidate after renames, refetch disabled queries, gate on permissions without flashing "denied" |
| 3 | [Tables over paginated APIs](03-tables-over-paginated-apis.md) | Move filters, sorts and counts to the server, and put page numbers on top of cursors without serving the wrong rows |

### Part 3 — Values that change shape

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Dates are not days](04-dates-are-not-days.md) | Choose the zone that defines a day, build its bounds from calendar parts, display instants in a zone without arithmetic, and know where Temporal ships |
| 5 | [Forms and the shape of "no value"](05-forms-and-no-value.md) | Convert at one adapter, keep numeric text raw until commit, keep absent / null / empty distinct, and test presence correctly |
| 7 | [Language traps in UI code](07-language-traps.md) | Spot shallow copies, unsafe merges of untrusted JSON, point-free callbacks, non-exhaustive switches, and `replace` patterns before they ship |

### Reference

| # | File | |
|---|------|--|
| 8 | [Glossary](08-glossary.md) | Look things up |
| 9 | [Exercises & answers](09-exercises.md) | Worked answers to every Check yourself, plus things to try |

## If you're short on time

- **30 minutes:** file 01 §1.3–§1.4, then file 02 §2.3–§2.6.
- **A list doesn't update after a save:** file 02 §2.3, §2.4, §2.5, in that order.
- **Counts or filters look wrong on a big table:** file 03 §3.2 and §3.4.
- **A date filter is off by a day for some users:** file 04 §4.2–§4.5.
- **A form loses or invents values:** file 05 §5.2, §5.4, §5.5.
- **A click fires two handlers, or an input loses focus while typing:** file 06 §6.2–§6.4.
- **A security scanner flagged your merge, or defaults changed for everyone:** file 07 §7.2–§7.3.

## The one-paragraph summary of everything

React keeps state at a position in the tree, so anything that swaps what's at that position —
a loading skeleton in place of the inputs — throws state away; events, context and state follow
the tree you wrote in JSX, so a click inside a portal still bubbles to the row that rendered it;
and a value that never affects the screen belongs in a ref, not in state. Data from the server is a cached
copy, not a value you own: every input the fetch reads belongs in its query key; invalidation
matches by key prefix, so renames must be followed everywhere (build keys in one module); a query
with `enabled: false` ignores invalidation and needs an explicit `refetch()`; a dependent query
that hasn't run is "not yet", never "no", so permission gates need three states; placeholder data
keeps the old screen during a refetch; and on a 403 the client's copy of permissions is resynced,
because the server is the control. Once the server paginates, the browser holds one page, so
filters, sorts and counts must run on the server too; a page-number UI over cursors caches one
cursor per reached page and clears them all when the filters, sort or page size change. A `Date`
is an instant, not a day: decide which zone defines the day, build half-open bounds from calendar
parts in that zone (DST makes days 23 or 25 hours long, and in some zones skips midnight),
display with `Intl.DateTimeFormat`'s `timeZone` rather than shifting milliseconds, never call
setters on a `Date` held in state, and label times with their zone; Temporal fixes this properly
but isn't in a Safari release yet. Forms need one adapter between API and form types, raw text
for numeric fields until commit, absent / `null` / `""` kept distinct (Merge Patch gives them
three different meanings), and presence tests that know `0` and `""` are values and that `??`
never catches `NaN`. Underneath it all sit a few language rules: spread copies one level, a merge
of untrusted JSON must not walk into `__proto__` (better, use `Map` or null-prototype objects and
copy only known keys), a function reference passed as a callback receives the caller's arguments,
a `switch` over a union should end in a `never` check, and a string replacement treats `$` as a
pattern.

## How to use me

Ask anything, including:

- "Here's my `useQuery` call — what's missing from the key?"
- "Why doesn't my mutation refresh this screen?"
- "Draw the query lifecycle for a dependent query, with status and fetchStatus at each step"
- "Convert this offset-paginated table to cursors, keeping page numbers"
- "Which zone should define 'today' for this report, and how do I compute it?"
- "Write the `toForm` / `fromForm` pair for this API type"
- "Has Safari shipped Temporal yet?"
- "Why did this click reach that handler? Here's the component tree"
- "Is this merge safe against prototype pollution?"
- "Is my mental model right? Here's what I think happens…"

I'll add chapters if a topic earns one — optimistic updates and accessible data tables are the
obvious next candidates.
