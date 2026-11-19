# 8. Glossary

Reference, not reading. The file in brackets is where the term is taught.

## React and the browser

| Term | Meaning |
|---|---|
| **Component** | A function that takes props and returns a description of UI (01) |
| **Props** | A component's inputs, passed by its parent (01) |
| **State** | A value React keeps for a component between renders; setting it schedules a re-render (01) |
| **Render / commit** | React calling your component for a fresh description / applying the difference to the DOM (01) |
| **Position in the tree** | Where a component sits among its parent's children. State is attached to it; a different component type there discards the state (01) |
| **Effect** (`useEffect`) | Code that runs after a commit to sync with something outside React. Re-fetching in one is usually a missing query-key input (01, 02) |
| **Controlled input** | An input whose displayed value comes from React state on every render (05) |
| **Erased types** | TypeScript annotations are removed before the code runs, so they never check runtime data (01) |
| **React tree vs DOM tree** | The component hierarchy you wrote in JSX vs the elements in the page. Events, context and state follow the React tree (06) |
| **Portal** | Renders a component's output into a different DOM node; it stays a child in the React tree (06) |
| **`target` / `currentTarget`** | The DOM node the event started at / the element whose handler is running (06) |
| **`stopPropagation`** | Stops the event reaching ancestors' handlers; doesn't stop default behaviour or same-element handlers (06) |
| **Loading boundary** | The part of the tree a loading placeholder replaces; should wrap data, not the controls that cause fetches (06) |
| **Ref** (`useRef`) | A mutable box React keeps between renders; changing it doesn't re-render. For values that never affect what's drawn (06) |

## Server state and caching

| Term | Meaning |
|---|---|
| **Client state** | State this tab owns alone: open menus, selected tab, typed text (01) |
| **Server state** | A copy of data the server owns; it can change without you and goes out of date (01, 02) |
| **Query** | A read, identified by a key, with a function that fetches it (01, 02) |
| **Mutation** | A write; afterwards you invalidate what it made stale (01, 02) |
| **Query key** | The array that identifies a cache entry; must contain every input the fetch depends on (02) |
| **Stale / fresh** | Whether cached data may be refetched; TanStack Query treats data as stale immediately by default (`staleTime: 0`) (02) |
| **Invalidation** | Marking matching entries stale and refetching those on screen; matches by key prefix unless `exact: true` (02) |
| **Key factory** | One module that builds every query key, so renames and invalidations stay consistent (02) |
| **`enabled`** | Whether a query runs automatically. `false` also makes it ignore invalidation (02) |
| **Dependent query** | A query whose `enabled` waits on another's result; a request waterfall by design (02) |
| **`isPending` / `isLoading`** | No data yet / first fetch actually in flight (02) |
| **Placeholder data** | Data shown while an entry has none (often the previous key's), not written to the cache (02) |
| **`refetchOnMount: 'always'`** | Refetch on every mount even if the data is fresh (02) |
| **Three-state gate** | Loading / allowed / denied; "not resolved yet" is never "denied" (02) |

## Pagination

| Term | Meaning |
|---|---|
| **Server-side (manual) processing** | The server filters, sorts, counts and paginates; the table only holds state (03) |
| **`manualPagination` / `manualFiltering` / `manualSorting`** | TanStack Table options saying the data is already processed (03) |
| **`rowCount` / `pageCount`** | Totals you pass so a manual table knows how many pages exist; `pageCount: -1` means unknown (03) |
| **Facet count** | The count shown next to a filter option; an aggregate, so a server job (03) |
| **Offset pagination** | "Skip N rows"; slows with depth and drifts when rows are inserted (03) |
| **Keyset / cursor pagination** | "Rows after this one"; stable and fast, but can't jump to arbitrary pages (03) |
| **Cursor** | An opaque bookmark into one ordering of one filtered set (03) |
| **Page→cursor map** | The client's record of which cursor fetches each reached page; cleared when the query changes (03) |

## Time

| Term | Meaning |
|---|---|
| **Instant** | A point on the timeline, the same for everyone. A JavaScript `Date` is one (04) |
| **Calendar date** | A day label like 8 March 2026, with no zone (04) |
| **Wall-clock time** | A local date and time with no zone (04) |
| **Zoned date-time** | An instant plus the IANA zone it's viewed in (04) |
| **IANA zone** | A region's name (`America/New_York`) standing for its whole history of offsets and DST rules (04) |
| **Offset** | One moment's difference from UTC (`-05:00`); not a zone (04) |
| **Half-open interval** | `[start, end)`: includes the start, excludes the end, so days tile exactly (04) |
| **DST gap** | Local times skipped when clocks jump forward; in some zones it includes midnight (04) |
| **Temporal** | The TC39 date-time API (Stage 4, March 2026); not yet in a Safari release as of November 2026 (04) |

## Forms and values

| Term | Meaning |
|---|---|
| **Adapter (`toForm` / `fromForm`)** | The single pair of functions converting between API and form shapes (05) |
| **Coercion** | Converting a value's type; `z.coerce.number()` uses `Number()`, which turns `""` into `0` (05) |
| **Bad input** | HTML's state for a number field whose text can't be converted; its `value` is then empty (05) |
| **Tri-state field** | Absent / present (including `null` and `""`) / being removed (05) |
| **JSON Merge Patch** | RFC 7396: `null` removes a member, an absent member is left alone (05) |
| **Truthiness** | `0`, `""`, `NaN`, `null`, `undefined`, `false` all count as false (05) |
| **`??`** | Nullish coalescing: falls back only on `null` or `undefined` — not `NaN`, not `0` (05) |
| **`!= null`** | True for anything except `null` and `undefined` (05) |
| **`Number.isFinite`** | True only for a number that isn't `NaN` or ±Infinity; the check a parse result needs (05) |

## Language traps

| Term | Meaning |
|---|---|
| **Shallow copy** | A new outer object whose nested objects are shared with the original; what spread and `Object.assign` make (07) |
| **`structuredClone`** | Deep copy via the structured clone algorithm; throws `DataCloneError` on functions (07) |
| **Prototype pollution** | Writing to `Object.prototype` (often via `__proto__` or `constructor.prototype` keys in merged JSON), changing every object (07) |
| **Null-prototype object** | `Object.create(null)`: has no prototype to pollute (07) |
| **`QueryFunctionContext`** | The object TanStack Query passes as a query function's first argument (07) |
| **Point-free callback** | Passing `f` instead of `x => f(x)`; `f` then receives whatever arguments the caller supplies (07) |
| **Exhaustiveness check** | Assigning the leftover value to `never` in a `switch` default so a missing case fails to compile (07) |
| **Replacement patterns** | `$&`, `` $` ``, `$'`, `$$`, `$1` in a string passed to `replace`; not applied to a replacer function's result (07) |
