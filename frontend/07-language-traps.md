# 7. Language traps in UI code

Assumes file 01. Each section is independent; read the one whose symptom you have. The Go side
of the first trap — struct copies that share maps and slices — is
[go-services/06-copies-that-share-memory.md](../go-services/06-copies-that-share-memory.md).

## 7.1 The problem

Five bugs, each caused by a language rule rather than by the logic around it:

- An editor merges an org's saved settings over the defaults. After one org saves, *every* org's
  defaults have changed.
- The same merge, given JSON from the server, is flagged by a security scanner as a
  prototype-pollution sink.
- Adding an optional `search` parameter to a fetch function breaks the build in five unrelated
  files.
- A "90 days" option in a chart's range selector shows an empty chart. The option is still offered;
  the code that handles it was deleted a month earlier.
- A debugging panel shows `[object Object]` where nested data should be, and a script that edits
  text with `replace` pastes a whole document into itself.

None of these throws an error at the point of the mistake. That's what makes them traps.

## 7.2 Spread copies one level

`{ ...defaults, ...saved }` looks like a merge. It's a shallow copy: MDN says "a single spread
creates a shallow copy of the original object", and for arrays that spread "effectively goes one
level deep while copying"
([MDN, Spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax)).
Two consequences:

1. **Nested sections are replaced, not merged.** If `saved.profile` exists, it replaces
   `defaults.profile` wholesale, so a key added to the defaults later never reaches orgs that have
   saved settings.
2. **A recursive merge into a spread copy writes into the original.** Someone fixes (1) with a
   hand-written `deepMerge({ ...defaults }, saved)`. The outer object is new, but
   `copy.profile` *is* `defaults.profile`, so recursing into it mutates the shared constant. I
   reproduced this in Node 26: after one merge, `DEFAULTS.profile.photo` was `true`.

The same thing happens in Go, where copying a struct copies its map headers, not the maps:

```go
type Template struct {
	Sections map[string]map[string]bool
}

var defaults = Template{Sections: map[string]map[string]bool{
	"profile": {"name": true, "photo": false},
}}

func main() {
	t := defaults                         // copies the struct, not the maps inside it
	t.Sections["profile"]["photo"] = true // writes into the map defaults also points at
	fmt.Println(defaults.Sections["profile"]["photo"]) // true
}
```

<details><summary>Rust</summary>

```rust
// Rust can't alias a mutable shared default by accident: you must clone or borrow.
// Clone on owned maps copies the nested maps too.
let base = defaults();
let mut t = base.clone();
t.get_mut("profile").unwrap().insert("photo".into(), true);
println!("{}", base["profile"]["photo"]); // false
```

</details>

<details><summary>C++</summary>

```cpp
// std::map has value semantics: copying it copies the nested maps.
Sections t = defaults;
t["profile"]["photo"] = true;          // defaults unchanged
// Sharing comes back as soon as you copy a pointer instead:
auto shared = std::make_shared<Sections>(defaults);
auto alias = shared;                   // same map
(*alias)["profile"]["photo"] = true;   // visible through shared
```

</details>

<details><summary>JavaScript</summary>

```js
const copy = { ...DEFAULTS };          // new outer object
copy.profile === DEFAULTS.profile;     // true: same nested object
const deep = structuredClone(DEFAULTS);
deep.profile === DEFAULTS.profile;     // false
```

</details>

I ran all four (Go 1.24, Rust 1.96, Apple clang, Node 26). The fix in JavaScript: start from
a deep copy — `structuredClone(defaults)`, which "creates a deep clone of a value using the
structured clone algorithm" and throws `DataCloneError` for values it can't serialise, such as
functions ([MDN, structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone)) —
or write the merge so it builds new objects at every level instead of assigning into existing ones.

## 7.3 Merging untrusted JSON can rewrite every object

`JSON.parse('{"__proto__": {"isAdmin": true}}')` produces an object with an *own* property named
`__proto__` — harmless so far. A naive recursive merge then does `target["__proto__"]`, which reads
`Object.prototype`, and recurses into it. In Node 26, after one such merge into `{}`,
`({}).isAdmin` was `true` for every object in the program. The same works through
`{"constructor": {"prototype": {...}}}`, which I also reproduced.

OWASP calls this "a critical vulnerability that can allow attackers to manipulate an application's
JavaScript objects"
([OWASP Prototype Pollution Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html)).
What's interesting is which defences it recommends, because they're not the one teams usually
reach for:

| Defence | OWASP's position |
|---|---|
| `Map` / `Set` instead of object literals for keyed data | Recommended first |
| `Object.create(null)` — objects with no prototype to pollute | Recommended |
| `Object.freeze` on built-in prototypes | Recommended, with a warning that libraries which modify prototypes can break |
| Node's `--disable-proto=delete` | Defence in depth |
| A deny-list of `__proto__`, `constructor`, `prototype` keys in your merge | Not among its recommendations; it notes pollution "is still possible using `constructor.prototype` properties" |

The deny-list is still the quickest patch for an existing merge, and if you use one it must cover
all three names. But the structural fix is not to merge arbitrary keys at all: copy only the keys
your schema knows (file 05 §5.2's adapter), or use a maintained merge utility.

## 7.4 Passing a function reference hands it the caller's arguments

```ts
async function getOrganizations() { … }
useQuery({ queryKey: ['orgs'], queryFn: getOrganizations });   // fine, today
```

Later, `getOrganizations(search?: string)` gains an optional parameter. TanStack Query calls every
query function with a `QueryFunctionContext` object — `queryKey`, `client`, `signal`, `meta`
([Query Functions](https://tanstack.com/query/latest/docs/framework/react/guides/query-functions))
— so `search` now receives that object. TypeScript caught it here as a type error; in plain
JavaScript the request would have gone out with `search=[object Object]`.

It's the same trap as the old `['1', '2', '3'].map(parseInt)`, which is `[1, NaN, NaN]` (checked in
Node 26): `map` passes the index as the second argument, and `parseInt` reads it as the radix.

The rule: **only pass a function reference where you control both signatures.** Otherwise wrap it,
so the call site says which arguments are meant:

```ts
useQuery({ queryKey: ['orgs', search], queryFn: () => getOrganizations(search) });
```

And notice that `search` went into the key too (file 02 §2.3).

## 7.5 A removed case should fail the build

The range selector offers `'7d' | '30d' | '90d'`. The transform that consumes it has a `switch`
whose `default` logs an error and returns nothing. Someone removes the `'90d'` case; the selector
still offers it; selecting it renders an empty chart. Nothing failed at build time because the
`default` branch accepted everything.

TypeScript can make this a compile error. The handbook's exhaustiveness pattern: in the `default`,
assign the value to `never`. "The `never` type is assignable to every type; however, no type is
assignable to `never`", so if any case is unhandled, the assignment fails to compile
([TypeScript Handbook, Narrowing — Exhaustiveness checking](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking)).

```ts
const RANGES = ['7d', '30d', '90d'] as const;      // the selector renders from this list
type ChartRange = (typeof RANGES)[number];

function bucketDays(r: ChartRange): number {
  switch (r) {
    case '7d':  return 1;
    case '30d': return 1;
    case '90d': return 7;
    default: {
      const unreachable: never = r;                // compile error if a case is missing
      throw new Error(`unhandled range ${unreachable}`);
    }
  }
}
```

Two parts make it work: the selector and the `switch` share one type, and the `default` proves
exhaustiveness instead of swallowing the rest.

## 7.6 `String(obj)` and `replace` both reinterpret your data

**`[object Object]`.** Converting a plain object to a string calls `Object.prototype.toString`,
which returns `"[object Object]"` — not its contents. `String(value)`, template literals and string
concatenation all do it. For a panel that shows an API response whose shape keeps changing, don't
hand-build a field renderer that has to know every nested type; pretty-print the whole thing with
`JSON.stringify(value, null, 2)` in a `<pre>` with wrapping turned on.

**`$` in replacements.** When the replacement passed to `String.prototype.replace` is a string, `$`
sequences are patterns, not text. From MDN's table
([MDN, String.prototype.replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace)):

| In the replacement | Inserts |
|---|---|
| `$$` | A literal `$` |
| `$&` | The matched text |
| `` $` `` | Everything *before* the match |
| `$'` | Everything *after* the match |
| `$1`, `$<name>` | Capture groups (only with a RegExp pattern) |

So inserting a code snippet that happens to contain `` $` `` into a document copies the entire
preceding document into the middle of it. Measured in Node 26: `"x=1".replace("=", "$`")` is
`"xx1"`. MDN: "the above-mentioned special replacement patterns do *not* apply for strings returned
from the replacer function." When the replacement is arbitrary text, use a function — `s.replace(anchor, () => text)` — or `s.split(anchor).join(text)`. And remember that a string pattern
replaces only the first occurrence.

> **Teacher's aside.** Four of these five traps share a shape: an API that looks like it takes
> *data* actually interprets it. `__proto__` looks like a key and is an accessor; a function
> reference looks like a value and receives arguments you didn't choose; a replacement string looks
> like text and is a template; `String(obj)` looks like serialisation and is a type tag. When data
> you don't control flows into one of these, ask "is this treated as data, or as instructions?"

## Check yourself

1. `const merged = deepMerge({ ...DEFAULTS }, saved)` with a merge that assigns into `target[k]`
   when both sides are objects. After org A saves `{ profile: { photo: true } }`, what does org B
   see when it opens the editor with no saved settings? Trace the object identities.
2. Your merge skips `__proto__` but not `constructor`. Write a JSON payload that still pollutes
   `Object.prototype`, and say which of OWASP's recommended defences would have stopped it without
   any key list.
3. `items.map(formatPrice)` worked until `formatPrice(amount, currency = 'USD')` gained its second
   parameter. What does the output look like now, and why did nothing fail?
4. The selector's options come from `RANGES` and the `switch` has the `never` default. A developer
   deletes `case '90d'`. What happens at build time? What if they delete `'90d'` from `RANGES`
   instead — and what if the options were a hand-written array of strings instead of `RANGES`?
5. A script inserts a user's template into a page with `html.replace('<!-- slot -->', template)`.
   The template contains `Total: $&`. What ends up in the page? Give two fixes, and say why one of
   the common "fixes" — escaping `$` as `$$` — is easy to get wrong.
