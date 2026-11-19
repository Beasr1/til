# 5. Forms and the shape of "no value"

Assumes file 01 (§1.2 on erased types, §1.5 on what JSON and inputs do to values) and file 02
§2.2 (background refetches).

## 5.1 The problem

A record editor, and six bugs found in it over a year:

- A document-type dropdown never shows the saved value. The API sends `code: 4`; the form schema
  says `string`; the dropdown's options are `"4"`. Fixed once in `defaultValues`, then found
  again months later in the `reset()` call, which had its own copy of the mapping.
- A lookup code of `0` is never enriched with its description. The guards read `if (code && !desc)`.
- Typing `.45` into a numeric field snaps back to the old value, and the change is never sent.
- A user types an email into a "recipients" chip input and clicks Save without pressing Enter.
  The alert saves with no recipients.
- Opening a record and saving it without changes creates every absent field as `""` or `0`.
- A config parser was "fixed" from `parseInt(env.P) || 0.1` to `parseFloat(env.P) ?? 0.1`. Now,
  when the variable is unset, the value is `NaN` and the feature it controls silently turns off.

Each one is a conversion that lost information: between the API's types and the form's, between
text and number, or between different kinds of "nothing".

## 5.2 One adapter at the API edge

The API's shape and the form's shape are different types, and pretending otherwise spreads
conversions across the codebase:

| | API (JSON) | Form state | Input element |
|---|---|---|---|
| A lookup code | `4` (number) | whatever the widget needs | `"4"` — an `<option>`'s value is a string |
| An absent field | key missing | must be represented somehow (§5.4) | — |
| A date | `"2026-03-08T00:00:00Z"` | depends on the picker (file 04) | `"2026-03-08"` for `<input type="date">` |

TypeScript can't catch the mismatch at the boundary, because the types are erased (file 01 §1.2):
if the API type says `code: string` and the server sends `4`, nothing checks. Write the
conversion **once**, as a pair of functions, and use them everywhere the form is filled or
submitted:

```ts
type Api  = { docType?: { code: number } | null; nickname?: string | null };
type Form = { docType: string; nickname: Field<string> };   // Field: see §5.4

export function toForm(a: Api): Form { /* every conversion, in one place */ }
export function fromForm(f: Form): Partial<Api> { /* the inverse */ }

useForm({ defaultValues: toForm(record) });
// ...and later, after a refetch:
reset(toForm(record));        // same function, so the two can't drift apart
```

The React Hook Form docs give the reason the mapping tends to get duplicated: "`defaultValues`
are cached. To reset them, use the `reset` API"
([useForm](https://react-hook-form.com/docs/useform), source on
[GitHub](https://github.com/react-hook-form/documentation/blob/master/src/content/docs/useform.mdx)).
Two entry points, so two places to forget a conversion — unless both call `toForm`.

The same docs warn about the `values` prop, which re-fills the form when its input changes: it
"replaces anything the user has typed", so if a query refetches on window focus (file 02 §2.2)
the user's edits vanish unless you set `resetOptions: { keepDirtyValues: true }`.

> ⚠️ **Schema coercion is not a conversion policy.** Zod's `z.coerce.number()` calls
> `Number(input)` ([Zod API, Coercion](https://zod.dev/api)), and `Number("")` is `0`,
> `Number(null)` is `0`, `Number(" ")` is `0` (checked in Node 26). An empty field coerced that way
> becomes a valid zero. Use coercion only where an empty input can't occur, or preprocess `""` to
> `undefined` first so the "required" check can see it.

## 5.3 Numeric inputs hold text until you commit

The `.45` bug. The input was controlled like this:

```tsx
<input type="number" value={rate}
       onChange={(e) => setRate(parseFloat(e.target.value) || 0)} />
```

The user types `.` first. `parseFloat(".")` is `NaN`, `NaN || 0` is `0`, so state becomes `0`,
React re-renders the input with `0`, and the user's keystroke is gone.

There's a second layer underneath. For `<input type="number">`, the HTML standard says user agents
"must not allow the user to set the value to a non-empty string that is not a valid floating-point
number", and while the user's text can't be converted, the control is "suffering from bad input"
([HTML, Number state](https://html.spec.whatwg.org/multipage/input.html#number-state-(type=number))).
Read literally, that means `e.target.value` is `""` while the box visibly contains `-` or `1e` (I
haven't measured this browser by browser). So the element itself may hide the in-between text from
you.

The fix is to stop pretending the field holds a number while the user is typing:

```tsx
const [rateText, setRateText] = useState(String(initialRate));   // raw text is the state

<input inputMode="decimal" value={rateText}
       onChange={(e) => setRateText(e.target.value)} />           // never parse on keystroke

function commit() {                                                // on blur or on submit
  const n = Number(rateText);
  if (rateText.trim() === '' || !Number.isFinite(n)) return setError('Enter a number');
  save(n);
}
```

Using a text input with `inputMode="decimal"` (which asks mobile keyboards for digits) avoids the
bad-input layer entirely. Two more facts from the same section of the standard:

- `min` and `max` don't stop anyone typing an out-of-range value. They feed constraint validation
  (the control is "suffering from an underflow" or "overflow"); your commit step still has to check.
- `type="number"` "is not appropriate for input that happens to only consist of numbers but isn't
  strictly speaking a number" — the standard's examples are credit card numbers and US postal
  codes. A lookup code is an identifier; it belongs in a select or a text field, as a string.

## 5.4 Three kinds of nothing

JavaScript gives you three ways to say "no value", and JSON keeps only two:

| Value | Means, by convention | Survives `JSON.stringify`? |
|---|---|---|
| `undefined` / key absent | "not mentioned" — leave whatever is there | No; the key is dropped (file 01 §1.5) |
| `null` | "present, and empty" — or "clear it", depending on the API | Yes |
| `""` | "present, and the empty string" | Yes |

JSON Merge Patch (RFC 7396) is one API convention that makes the difference load-bearing: "Null
values in the merge patch are given special meaning to indicate the removal of existing values in
the target", while members the patch doesn't mention are left untouched
([RFC 7396 §1](https://www.rfc-editor.org/rfc/rfc7396#section-1)). Under that rule, sending `""`
sets a field, sending `null` deletes it, and leaving it out changes nothing. A mapper that writes
`data.x || ''` for every field turns all three into "set it to empty" — which is the "save without
changes creates every field" bug.

Forms make this awkward, and here the sources pull in different directions. The domain needs
"absent" to survive; React Hook Form says you "should avoid providing `undefined` as a default
value, as it conflicts with the default state of a controlled component". The resolution is not
to pick one: represent absence explicitly in the *form* type, and convert at the edge (§5.2).

```ts
type Field<T> =
  | { state: 'absent' }                  // key not in the record; UI shows an "Add" button
  | { state: 'present'; value: T | null } // includes null and ""; UI shows the input and "Remove"
  | { state: 'removing' };               // user clicked Remove; UI shows "Restore"

// fromForm: absent → omit the key; present → send the value; removing → send null
```

Adding and removing are now things the user *does*, recorded in state, rather than guesses made
from whether the box happens to be empty.

The chip input is the same idea in a different costume. The pending text in the input box is part
of the form's value, even though it hasn't become a chip. On submit, trim it, validate it, and
either add it to the list or block the save with an error. Never drop it silently.

## 5.5 Testing for presence: `0` and `""` are values

| `v` | `!!v` (truthy) | `v != null` | `v ?? d` | `v \|\| d` | `Number.isFinite(v)` |
|---|---|---|---|---|---|
| `undefined` | false | false | `d` | `d` | false |
| `null` | false | false | `d` | `d` | false |
| `0` | **false** | true | `0` | **`d`** | true |
| `""` | **false** | true | `""` | **`d`** | false |
| `NaN` | false | true | **`NaN`** | `d` | false |
| `false` | false | true | `false` | `d` | false |

I generated that table in Node 26. The bold cells are the bugs from §5.1:

- **Truthiness treats `0` and `""` as missing.** `if (code && !desc)` skips code `0`. Test presence
  with `code != null` (loose `!=` against `null` matches exactly `null` and `undefined`), or
  `typeof code === 'number'`.
- **`||` replaces valid falsy values.** `parseInt('0.1') || 0.1`: `parseInt` stops at the dot and
  returns `0`, and `0 || 0.1` is `0.1`, so any configured value below 1 was ignored, and `0` could
  never switch the feature off.
- **`??` doesn't replace `NaN`.** MDN: `??` "returns its right-hand side operand when its left-hand
  side operand is `null` or `undefined`"
  ([MDN, Nullish coalescing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)).
  `parseFloat(undefined)` is `NaN`, not `undefined`, so `parseFloat(env.P) ?? 0.1` is `NaN` whenever
  the variable is unset. Every comparison with `NaN` is false, so `Math.random() < NaN` never
  fires.

The general shape of the fix is the same in any language: **find out whether the value is there,
then parse it and check the parse succeeded, and only fall back when it's absent.** A failed parse
is an error to report, not a reason to use the default.

```go
// probability reads a value in [0, 1] from the environment.
// Unset or empty: the default. Set but invalid: an error. 0 is a valid value.
func probability(name string, def float64) (float64, error) {
	raw, ok := os.LookupEnv(name)
	if !ok || raw == "" {
		return def, nil
	}
	v, err := strconv.ParseFloat(raw, 64)
	if err != nil || math.IsNaN(v) || v < 0 || v > 1 {
		return 0, fmt.Errorf("%s=%q is not a probability", name, raw)
	}
	return v, nil
}
```

The `math.IsNaN` check isn't paranoia: Go's `strconv.ParseFloat` accepts the string `"NaN"` without
error, and `NaN < 0 || NaN > 1` is false, so without it `P=NaN` would pass. I ran it with Go 1.24:
unset → 0.1, `"0"` → 0, `"NaN"`, `"abc"` and `"1.5"` → errors.

<details><summary>Rust</summary>

```rust
use std::env;

/// Unset or empty: the default. Set but invalid: an error. 0 is a valid value.
fn probability(name: &str, default: f64) -> Result<f64, String> {
    let raw = match env::var(name) {
        Ok(s) if !s.is_empty() => s,
        _ => return Ok(default),
    };
    // f64::from_str accepts "NaN" and "inf"; the range check below rejects them
    // because it's written as "is inside", which NaN never is.
    match raw.parse::<f64>() {
        Ok(v) if (0.0..=1.0).contains(&v) => Ok(v),
        _ => Err(format!("{name}={raw:?} is not a probability")),
    }
}
```

</details>

<details><summary>C++</summary>

```cpp
#include <cstdlib>
#include <optional>
#include <string>

// Unset or empty: the default. Set but invalid: nullopt. 0 is a valid value.
std::optional<double> probability(const char* name, double def) {
    const char* raw = std::getenv(name);
    if (raw == nullptr || *raw == '\0') return def;
    char* end = nullptr;
    double v = std::strtod(raw, &end);
    if (*end != '\0') return std::nullopt;            // trailing junk, or nothing parsed
    if (!(v >= 0.0 && v <= 1.0)) return std::nullopt; // written so NaN fails
    return v;
}
```

</details>

<details><summary>JavaScript</summary>

```js
// Unset or empty: the default. Set but invalid: throw. 0 is a valid value.
function probability(name, def) {
  const raw = process.env[name];
  if (raw === undefined || raw.trim() === '') return def;
  const v = Number(raw);              // Number, not parseFloat: "0.5abc" is NaN, not 0.5
  if (!Number.isFinite(v) || v < 0 || v > 1) throw new Error(`${name}=${raw} is not a probability`);
  return v;
}
```

</details>

> **Teacher's aside.** `||` and `??` both look like "default if missing", and people choose
> between them by habit. They answer different questions. `||` asks "is this falsy?"; `??` asks
> "is this `null` or `undefined`?". Neither asks "is this a valid number?" — and for parsed input,
> that's the only question that matters. A parse result needs a validity check, not a fallback
> operator.

## Check yourself

1. The API returns `{ docType: { code: 4 } }` and the form's select has options with values `"1"`
   to `"9"`. The form schema is `z.string()`. What does the select show, and what does the schema
   do on submit if the value is never touched? Where should the fix go so it can't be missed twice?
2. `z.coerce.number().min(1)` validates a "quantity" field. The user clears the box and submits.
   What value does the schema see, and does validation pass? What would you change?
3. Walk through what happens, keystroke by keystroke, when a user types `-2` into a text input
   controlled as `value={String(n)}` with `onChange={e => setN(parseFloat(e.target.value) || 0)}`,
   starting from an empty box with `n = 0` shown as `"0"` and the text selected. Then explain why
   holding the raw string fixes it.
4. A record has `middleName` absent, `nickname: null` and `suffix: ""`. A mapper builds the form
   with `x || ''` and the submit sends every field. Under JSON Merge Patch rules, what does the
   server do to each of the three, and which of those is a change the user didn't make?
5. Fill in the value for each: `0 ?? 5`, `0 || 5`, `NaN ?? 5`, `parseInt("0.9") || 1`,
   `parseFloat(undefined) ?? 1`. For each one that's surprising, name the input that would have
   produced the bug in production.
6. A teammate proposes `const p = Number(env.P) || 0.1` as "the simple fix". Give two inputs where
   it's wrong, and say why the Go `probability` function's structure (presence, parse, validate)
   doesn't have either problem.
