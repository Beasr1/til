# 1. Foundations — what a frontend does with your data

**Who can skip this:** if you can explain why a React component's state survives one re-render
and vanishes on the next, what "server state" means and why it differs from a `useState`
value, and why a TypeScript type says nothing about what the API actually sent, go straight to
file 02. Everyone else: every later chapter assumes the vocabulary here.

## 1.1 The problem: a page is a long-lived program holding copies

A backend engineer usually pictures a web page the way the server sees it: a request comes in,
a response goes out, done. A modern single-page app breaks that picture. The page loads once
and then runs for hours, asking the server for data, keeping **copies** of it in memory,
showing those copies, letting the user edit them, and sending changes back.

Every bug in this course is a bug about one of those copies:

| Copy | Lives in | The question that goes wrong | File |
|---|---|---|---|
| A list or record fetched from the API | The query cache | Is this copy still true? | 02 |
| One page of a much larger result set | The query cache, plus table state | What does this copy *not* contain? | 03 |
| A timestamp turned into "the 8th of March" | A `Date`, a formatted string | Which clock defined that day? | 04 |
| A record turned into editable fields | Form state | Did the conversion lose "no value"? | 05 |

The rest of this chapter is the vocabulary those four chapters share: the layers (§1.2), how
React decides what to draw (§1.3), the two kinds of state (§1.4), and the boundaries where data
changes shape (§1.5).

## 1.2 The layers

| Layer | What it is | What it can't tell you |
|---|---|---|
| **The DOM** | The browser's live tree of elements. What the user actually sees and clicks | Anything about where the data came from |
| **JavaScript** | The language the browser runs. Dynamically typed: a value carries its type at runtime | Whether a value has the shape you expected, until you check |
| **TypeScript** | JavaScript plus static type annotations, checked at build time and then stripped | What arrived over the network. Types are erased before the code runs |
| **React** | A library that turns your state into DOM, and re-does it when state changes | When server data is stale. That's a separate concern (§1.4) |
| **A data library** (TanStack Query in this course) | A cache for server data, with rules for when to refetch | What your API means; it only knows keys and functions |

The TypeScript row is the one that bites people from typed backend languages. The handbook is
blunt: "Type annotations never change the runtime behavior of your program"
([TypeScript Handbook, The Basics — Erased Types](https://www.typescriptlang.org/docs/handbook/2/basics.html#erased-types)).
If you write `type Doc = { code: string }` and the API sends `{ "code": 4 }`, nothing complains.
The value is a number at runtime; your type is a promise nobody checked. That's why file 05
insists on one runtime conversion at the API edge.

## 1.3 How React decides what to draw

React's model is one sentence: **the UI is a function of state.** You write **components** —
functions that take inputs (**props**) and return a description of the UI. React calls them,
compares the description with the last one, and changes only the DOM that differs.

Three terms carry the rest of the course:

| Term | Meaning |
|---|---|
| **Render** | React calling your component function to get a fresh description. Happens on the initial render and whenever the component's state, or an ancestor's, is updated ([react.dev, Render and Commit](https://react.dev/learn/render-and-commit)) |
| **Commit** | React applying the difference to the DOM |
| **State** | A value React remembers for a component between renders, set with `useState` (or a library hook built on it). Setting it schedules a re-render |

The one rule that surprises people is where state lives. It isn't stored "in" your function or
your variable. React holds it and attaches it to a **position in the tree**: "React associates
each piece of state it's holding with the correct component by where that component sits in
the render tree," and "if it gets removed, or a different component gets rendered at the same
position, React discards its state"
([react.dev, Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)).

So this innocent-looking component destroys whatever the user had typed into `<Filters>` each
time a fetch starts, because for that render the position holds a `<Spinner>` instead:

```tsx
function Results({ isFetching }: { isFetching: boolean }) {
  if (isFetching) return <Spinner />;     // <Filters> is unmounted here...
  return <><Filters /><Table /></>;       // ...and mounted fresh, empty, here
}
```

Keep the controls mounted and put the loading indicator next to them, not in place of them.
File 06 §6.4 works through this case, and §6.2 shows the same "React tree, not DOM tree" rule
applied to events.

> **Teacher's aside.** People coming from backend code read a component as "a function that
> runs", and expect local state to behave like a local variable or like a field on an object.
> It's neither. It's a slot React keeps at a position in the tree. Same component type at the
> same position: the slot survives. Anything else at that position, even for one render: the
> slot is thrown away. Most "my input lost its value" bugs are this rule.

**Effects** (`useEffect`) run after a commit and are for synchronising with something outside
React. They come up in file 02 only as the wrong tool: re-fetching in an effect when a filter
changes is a symptom of a cache key that's missing the filter.

## 1.4 Two kinds of state

Not all state is the same kind of thing, and treating it as one is the root of most data bugs.

| | **Client state** | **Server state** |
|---|---|---|
| Example | Is the menu open? Which tab? What's typed in the search box? | The list of orders. The current user's permissions |
| Who owns it | This tab, alone | The server. You hold a copy |
| Can it change behind your back? | No | Yes — another user, another tab, a background job |
| Is reading it instant? | Yes | No. It needs a request |
| Right tool | `useState`, URL parameters | A cache with refetch rules |

The TanStack Query docs describe server state as data that "is persisted remotely in a location
you may not control or own", "requires asynchronous APIs", "implies shared ownership and can be
changed by other people without your knowledge", and "can potentially become 'out of date'"
([TanStack Query, Overview](https://tanstack.com/query/latest/docs/framework/react/overview)).
Each of those clauses is a requirement your code has to meet. File 02 builds the model; the
short version is that server state is a **cache**, and every cache problem — keys, staleness,
invalidation — applies.

Two terms from TanStack Query you'll see throughout:

| Term | Meaning |
|---|---|
| **Query** | A read: a key plus a function that fetches the data for that key. `useQuery({ queryKey, queryFn })` |
| **Mutation** | A write: a function that changes data on the server. Afterwards you tell the cache which queries it made stale |

## 1.5 The boundaries where data changes shape

Data crosses several boundaries on its way from the database to a pixel, and each one has its
own idea of "no value", "number" and "time":

```
database ─▶ API (JSON) ─▶ query cache ─▶ form / table state ─▶ input elements (strings)
   NULL        null,          whatever        your types          "" and text,
               absent key     arrived                             always
```

Three JSON facts matter for every later chapter. I checked each in Node 26:

| You serialise | JSON gets | Consequence |
|---|---|---|
| A property set to `undefined` | Nothing — the key is dropped | "Absent" and "undefined" collapse into one on the wire |
| `NaN` | `null` | A failed number parse can quietly become "no value" |
| A `Date` | An ISO string in UTC, like `"2026-03-08T00:00:00.000Z"` | The zone the user meant is gone (file 04) |

And on the input side, an HTML `<input>`'s `value` is always a string — even
`<input type="number">` — and is the empty string whenever the text isn't a valid number (file
05 §5.3).

## 1.6 Where each idea is taught

| Idea | File |
|---|---|
| Query keys, staleness, invalidation, `enabled`, placeholder data | 02 |
| Server-side pagination, filtering, counts; cursors behind page numbers | 03 |
| Instants vs calendar days, IANA zones, `Intl`, Temporal | 04 |
| Form adapters, raw text for numbers, `undefined` / `null` / `""`, `??` | 05 |
| Portals and event bubbling, loading boundaries, refs | 06 |
| Shallow copies, prototype pollution, callback arguments, exhaustive switches, `replace` | 07 |

A note on code. Where an idea is language-neutral (a cursor cache, a day boundary, a presence
check) the example is in Go, with Rust, C++ and JavaScript versions folded underneath. Where the
idea only exists in React or a browser API, the example is in TypeScript, because there's no
honest Go version of a React hook.

## Check yourself

1. A teammate says "the API returns `code: string`, it's in the types." What would you check
   before believing that, and why can't the TypeScript compiler check it for you?
2. A `<Filters>` component sits inside `{isLoading ? <Skeleton/> : <Filters/>}`. The user types
   into a filter, which starts a fetch. Describe, render by render, what happens to the text
   they typed.
3. Classify each as client or server state, and say what goes wrong if you store it the other
   way: the selected tab; the current user's role; the text in a search box; the number of
   unread notifications.
4. A form sends `{ nickname: undefined }` to mean "the user cleared the nickname". What does the
   server receive, and what has it lost?
5. Your component re-fetches in a `useEffect` whenever `status` changes. It works. Using only
   §1.4, explain what's wrong with the design and what the cache should have known instead.
