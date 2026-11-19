# 2. Consuming payloads you don't control

Assumes [file 01](01-the-installed-client-problem.md). That chapter was written
from the side that owns the schema. This one is written from the other side: you
decode a payload somebody else produces — a third-party API, another team's
service, a message re-encoded by a library in another language — and you find out
about their changes when your output goes wrong.

## The problem: a missing field that decodes as a valid answer

An upstream renames a field, or quietly stops sending it. Your decoder doesn't
complain. The struct field it would have filled is left at its zero value, and
zero happens to be a meaningful answer: the first member of an enum that means
"active", a count that passes a threshold check, a boolean that means "not
flagged". From that moment every record gets the same answer, and nothing in your
logs says why.

The two ways this shows up are mirror images:

- **Mass rejections.** The zero means "no record", so every valid customer fails.
- **False passes.** The zero means "fine", so every record sails through a check
  that is no longer looking at anything.

The wrong assumption underneath both is that the decoder will tell you when the
payload doesn't match your type. Most decoders are built to do the opposite.

## What decoders actually do

Here is the behaviour of four common JSON decoders for the situations that matter.
Each cell was checked by running it (Go 1.24, serde 1.0.228 with serde_json
1.0.143, nlohmann/json 3.11.3, and `JSON.parse` in Node), not recalled.

| Situation | Go `encoding/json` | Rust serde (derive) | JS `JSON.parse` | C++ nlohmann/json |
|---|---|---|---|---|
| Key missing, plain field | zero value, **no error** | error: `missing field` | property is `undefined` | `value(k, d)` returns `d`; `at(k)` throws |
| Key missing, `*T` / `Option<T>` | `nil` | `None` | `undefined` | — |
| `null` into a plain field | **no effect, no error** | error: `invalid type: null` | `null` | `get<int>()` throws |
| Unknown key | ignored | ignored | kept as a property | kept |
| Wrong type (`"one"` into an int) | that field skipped, rest decoded, error returned | whole decode fails | whatever arrived | throws |
| `254.0` into an integer | error | error | becomes `254`, indistinguishable | **silently converted** |
| Key differs only in case | **matched anyway** | not matched | not matched | not matched |

The Go column is the one that surprises people, and the package documentation
says all of it plainly. On unknown keys:

> By default, object keys which don't have a corresponding struct field are
> ignored (see [Decoder.DisallowUnknownFields] for an alternative).

On null:

> The JSON null value unmarshals into an interface, map, pointer, or slice by
> setting that Go value to nil. Because null is often used in JSON to mean "not
> present," unmarshaling a JSON null into any other Go type has no effect on the
> value and produces no error.

And on type mismatches:

> If a JSON value is not appropriate for a given target type, or if a JSON number
> overflows the target type, Unmarshal skips that field and completes the
> unmarshaling as best it can. If no more serious errors are encountered,
> Unmarshal returns an [UnmarshalTypeError] describing the earliest such error. In
> any case, it's not guaranteed that all the remaining fields following the
> problematic one will be unmarshaled into the target object.

(All three from the [`encoding/json` docs](https://pkg.go.dev/encoding/json#Unmarshal).)
That last paragraph matters later. Go reports only the *first* mismatch, and it
explicitly doesn't promise the rest of the struct is complete.

Nowhere does it say what happens to a struct field whose key is absent, because
nothing happens to it: the decoder only writes fields it finds keys for. A fresh
struct keeps its zero values; a reused one keeps whatever it held before.

Rust's serde is stricter by default — a missing non-`Option` field is an error —
but it loosens exactly where people loosen it in practice. Mark every field
`Option<T>` "to be safe" and a renamed key becomes `None` with no error at all,
which is the Go behaviour again by another route.

## Three states, not two

The fix starts with seeing that "the field has no useful value" is really three
different facts:

```
  {}                    absent   — the producer didn't send the key
  {"status": null}      null     — the producer sent the key with no value
  {"status": 0}         zero     — the producer sent a real value that is zero
```

They usually mean different things. Absent often means "this producer version
doesn't know the field" or "the upstream stopped filling it"; null often means
"known, but not applicable to this record"; zero is an answer. A plain integer
collapses all three into one value, and every type system needs some help to keep
them apart:

| Type | absent | null | zero |
|---|---|---|---|
| Go `int` | `0` | `0` (or the previous value) | `0` |
| Go `*int` | `nil` | `nil` | `&0` |
| Go custom type with `UnmarshalJSON` | distinguishable | distinguishable | distinguishable |
| Rust `u32` | error | error | `0` |
| Rust `Option<u32>` | `None` | `None` | `Some(0)` |
| Rust `Option<Option<u32>>` + "present" helper | `None` | `Some(None)` | `Some(Some(0))` |
| JS | `!Object.hasOwn(o, k)` | `o[k] === null` | `0` |

A pointer or `Option` separates zero from "not provided", and that is enough for
most fields. When absent and null mean different things — a PATCH body where null
means "clear it", or an upstream where absent means "field retired" — you need the
third state too. In Go, the hook is that `encoding/json` calls a type's
`UnmarshalJSON` only when the key is present, and calls it "including when the
input is a JSON null":

```go
// Field records whether a key was present, and whether it was null.
type Field[T any] struct {
	Present bool // the key appeared in the object
	Null    bool // it appeared with the value null
	Value   T
}

func (f *Field[T]) UnmarshalJSON(b []byte) error {
	f.Present = true
	if string(b) == "null" {
		f.Null = true
		return nil
	}
	return json.Unmarshal(b, &f.Value)
}

type Account struct {
	Status Field[int] `json:"status"`
}

// {}               -> Present=false
// {"status":null}  -> Present=true, Null=true
// {"status":0}     -> Present=true, Null=false, Value=0
```

<details>
<summary>Rust version</summary>

`deserialize_with` runs only when the key is present, and `default` fills in
`None` when it isn't. The `serde_with` crate ships the same idea as
`serde_with::rust::double_option`.

```rust
use serde::{Deserialize, Deserializer};

#[derive(Deserialize, Debug)]
struct Account {
    // None = absent, Some(None) = null, Some(Some(n)) = a value
    #[serde(default, deserialize_with = "present")]
    status: Option<Option<i64>>,
}

fn present<'de, D, T>(d: D) -> Result<Option<Option<T>>, D::Error>
where
    D: Deserializer<'de>,
    T: Deserialize<'de>,
{
    Option::<T>::deserialize(d).map(Some)
}
```

</details>

<details>
<summary>C++ version (nlohmann/json)</summary>

```cpp
#include <nlohmann/json.hpp>
using nlohmann::json;

struct IntField {
    bool present = false;
    bool null = false;
    long long value = 0;
};

IntField read_int(const json& obj, const char* key) {
    IntField f;
    auto it = obj.find(key);
    if (it == obj.end()) return f;          // absent
    f.present = true;
    if (it->is_null()) { f.null = true; return f; }
    f.value = it->get<long long>();         // throws on a string; truncates 254.5
    return f;
}
```

</details>

<details>
<summary>JavaScript version</summary>

```js
function readField(obj, key) {
  if (!Object.hasOwn(obj, key)) return { present: false, null: false };
  if (obj[key] === null) return { present: true, null: true };
  return { present: true, null: false, value: obj[key] };
}
```

</details>

> **Teacher's aside.** Protocol Buffers has this problem built in, and its answer
> is worth stealing. In proto3, a plain scalar field has *implicit presence*:
> "fields with implicit presence are not serialized if they contain their default
> value" ([field presence guide](https://protobuf.dev/programming-guides/field_presence/)),
> so zero and absent are the same bytes. (Marking the field `optional` gives it
> *explicit presence*, where the API "also stores whether or not a field has been
> set".) That is why the language guide insists
> the zero member of every enum mean nothing: it "should have the name
> `ENUM_TYPE_NAME_UNSPECIFIED` or `ENUM_TYPE_NAME_UNKNOWN`", and "it is also
> recommended that this first, default value have no semantic meaning other than
> 'this value was unspecified'" ([proto3 guide](https://protobuf.dev/programming-guides/proto3/)).
> When an upstream you don't control puts a real state at 0 — "active", "valid",
> "success" — you have inherited exactly the trap that convention exists to
> prevent, and the pointer is how you opt back out of it.

## Choose the deciding field by measuring it

A decision has to read *some* field. The spec tells you which field ought to be
there; the data tells you which one is. They differ more often than you'd think,
and when they do the spec loses.

So before a field is allowed to decide anything, sample real payloads and count,
for that field and its alternatives:

| Count | Why |
|---|---|
| How often the key is absent | a field missing in a large share of records cannot decide alone |
| How often it is null | null may be the producer's way of saying "not applicable" |
| How often it holds each value | zero being common is a warning if zero is a meaningful state |
| Whether it arrives in more than one shape | a bare string in some records and an object in others |

Two rules follow. Prefer the **raw fact** to a derived flag: an expiry date you
can compare with today is more robust than a status field someone else computed
from it, and keeps working when they stop computing it. And make **"absent" its
own outcome**, separate from every real answer, so that an upstream change shows
up as a spike in "absent" on a dashboard rather than as a flood of rejections.

## How strict should the reader be?

There are two respectable positions, and they disagree.

Martin Fowler's [Tolerant Reader](https://martinfowler.com/bliki/TolerantReader.html)
(2011) argues for leniency: "be as tolerant as possible when reading data from a
service", "only take the elements you need, ignore anything you don't", so that
the provider can "make any change that ought not to break your code". Ignoring
unknown keys is this principle, and it's why decoders default to it.

[RFC 9413, *Maintaining Robust Protocols*](https://www.rfc-editor.org/rfc/rfc9413.html)
(2023, Informational) argues the costs of the **robustness principle** — "be
conservative in what you send, and liberal in what you accept" — and specifically
of its liberal half: "negative consequences to interoperability
accumulate over time if implementations silently accept faulty input", and an
interpretation "that advocates for tolerating unexpected inputs is no longer
considered best practice in all scenarios". Its section 5.1 names the alternative
"virtuous intolerance": generating fatal errors for unspecified conditions instead
of attempting recovery.

The two are talking about different things, and that resolves most of it. Fowler
is about fields you **don't read**: ignore them, so the provider can add things.
RFC 9413 is about input you **do rely on** being wrong: accepting it silently is
how a bug becomes a de facto standard. Every failure in this chapter is a field
the reader relied on going missing or changing shape, which is the RFC's
territory, not Fowler's.

What that gives you in practice:

| Where | Unknown keys | Fields the decision depends on | Type mismatches |
|---|---|---|---|
| Contract tests against captured real payloads | **reject** (`DisallowUnknownFields`, `deny_unknown_fields`) so the test notices new and renamed keys | must be present | fail the test |
| Production | ignore | absent → explicit "cannot evaluate" outcome, counted | tolerate only deliberately, and count it |

> ⚠️ **Swallowing Go's `UnmarshalTypeError` is a real option and a real trap.**
> Because the decoder carries on past a mismatch, ignoring the error keeps a
> drifting upstream working: you get every field except the bad one. But the bad
> field is now zero, which is this chapter's opening bug, and the docs don't
> promise the fields after it were decoded either. If you tolerate it, log the
> error, count it per field, and treat that field as absent, not as zero.

## Numbers that change type on the way

JSON has one number type. Many of the formats and languages around it don't, and
a value crossing between them can change shape without changing value.

MessagePack, for instance, has separate type families: "Integer represents an
integer" and "Float represents a IEEE 754 double precision floating point number"
([spec](https://github.com/msgpack/msgpack/blob/master/spec.md)). A re-encoder
that holds a number as a float at any point on the way can turn the integer `254`
into `254.0`: Rust's serde_json, for instance, writes an `f64` of 254 as
`254.0`. A strict integer decoder then rejects a value that is, numerically,
exactly right.
Go and serde both refuse `254.0` for an integer field; nlohmann/json accepts it
and would just as silently truncate `254.7`.

Two related shape changes come from Go's encoder:

- **Nil collections become null.** The docs say "a nil slice encodes as the null
  JSON value", and a nil map does the same (checked by running it; the
  [`vmihailenco/msgpack`](https://github.com/vmihailenco/msgpack) encoder also
  writes nil for both — `encodeMapValue` in `encode_map.go` and
  `encodeSliceValue` in `encode_slice.go`, v5.4.1). A consumer in another language that typed the field
  as a list or map then rejects the whole record. Initialise collections before
  encoding, or make the consumer's field optional with a default.
- **Large integers lose precision through `interface{}`.** Decoding into `any`
  stores numbers as `float64`, so `9007199254740993` comes back as
  `9007199254740992`. `Decoder.UseNumber` "causes the Decoder to unmarshal a
  number into an interface{} as a Number instead of as a float64", keeping the
  original digits.

If you know a field's value is always whole but its wire type may wobble, accept
integral floats and reject everything else, in one named type:

```go
// WholeNumber accepts 254 and 254.0 for an integer field, and rejects
// 254.5, "254", true and anything else.
type WholeNumber int64

func (n *WholeNumber) UnmarshalJSON(b []byte) error {
	if string(b) == "null" {
		return nil // same as encoding/json: null leaves the value alone
	}
	if len(b) > 0 && b[0] == '"' {
		// json.Number would accept "254" too; we don't want that
		return fmt.Errorf("want a number, got a string: %s", b)
	}
	var num json.Number
	if err := json.Unmarshal(b, &num); err != nil {
		return err
	}
	if i, err := num.Int64(); err == nil {
		*n = WholeNumber(i)
		return nil
	}
	f, err := num.Float64()
	if err != nil || f != math.Trunc(f) || math.Abs(f) > 1<<53 {
		return fmt.Errorf("want a whole number, got %s", b)
	}
	*n = WholeNumber(f)
	return nil
}
```

<details>
<summary>Rust version</summary>

```rust
use serde::{de::Error, Deserialize, Deserializer};

fn whole_number<'de, D: Deserializer<'de>>(d: D) -> Result<i64, D::Error> {
    let n = serde_json::Number::deserialize(d)?;
    if let Some(i) = n.as_i64() {
        return Ok(i);
    }
    match n.as_f64() {
        Some(f) if f.fract() == 0.0 && f.abs() <= 9_007_199_254_740_992.0 => Ok(f as i64),
        _ => Err(D::Error::custom(format!("want a whole number, got {n}"))),
    }
}

#[derive(serde::Deserialize)]
struct Limits {
    #[serde(deserialize_with = "whole_number")]
    max_length: i64,
}
```

</details>

<details>
<summary>C++ version (nlohmann/json)</summary>

```cpp
#include <cmath>
#include <stdexcept>
#include <nlohmann/json.hpp>

long long whole_number(const nlohmann::json& v) {
    if (v.is_number_integer()) return v.get<long long>();
    if (v.is_number_float()) {
        double d = v.get<double>();
        if (std::trunc(d) == d && std::fabs(d) <= 9007199254740992.0)
            return static_cast<long long>(d);
    }
    // nlohmann's own get<long long>() would truncate 254.7 without a word
    throw std::invalid_argument("want a whole number, got " + v.dump());
}
```

</details>

<details>
<summary>JavaScript version</summary>

`JSON.parse` already turns `254.0` into `254`, so JS can't see the difference.
The useful check runs the other way: JS accepts anything, so assert the shape.

```js
function wholeNumber(v) {
  if (typeof v !== "number" || !Number.isSafeInteger(v)) {
    throw new TypeError(`want a whole number, got ${JSON.stringify(v)}`);
  }
  return v;
}
```

</details>

## Enums: the value you've never seen

File 01 noted that adding an enum member is structurally safe and still leaves an
old reader with no case for it. From the consumer's side, that is the normal state
of affairs: the producer adds a member when it suits them, and your build hasn't
heard of it.

Google's API guidance is careful here. [AIP-180](https://google.aip.dev/180) says
"for enum values specifically, be aware that it is possible that user code does
not handle new values gracefully", allows new values freely only for enums "only
used in request messages", and asks that response enums "which are expected to
receive new values should document this". [AIP-126](https://google.aip.dev/126)
goes further: "for enums that change frequently, the API should use a string and
document the format." Protocol Buffers' proto3 enums are *open*: "unrecognized
enum values will be preserved in the message", and in languages such as Go "the
unknown enum value is simply stored as its underlying integer representation"
([proto3 guide](https://protobuf.dev/programming-guides/proto3/)). So a value
outside your constants is not hypothetical; the decoder is designed to hand you
one.

Three habits make that safe.

**Give every switch an explicit unknown branch, and choose which way it fails.**
For anything that gates an action — a payment, an approval, a verdict — unknown
must mean "not success". A default branch that falls through to the first case,
or to the success path, is the silent misbehaviour file 01 warned about.

**Mirror an upstream integer enum as named constants, and cite where they came
from.** A raw `4` in a condition is a guess; a constant with a comment pointing at
the upstream definition is a claim someone can check. Guessing is easy to get
wrong in a specific way: years of exit codes teach you that 0 means success, and
nothing obliges a third-party widget to agree.

**Route on a stable machine code, never on a display label, and normalise it in
one place.** A label like `"Gold member"` is written for people: it gets translated,
reworded and recased. If a machine code is on offer, use it. If a label is all
there is, trim and case-fold it in a single helper that every code path calls, so
two paths can't disagree about whether `"Gold"` and `"GOLD"` match.

```go
// WidgetStatus mirrors the status integers a third-party checkout widget posts.
// Copied from the widget's published enum (cite the file and version here).
// These numbers are theirs: never renumber them to suit our code.
type WidgetStatus int

const (
	WidgetCancelled WidgetStatus = 0 // not success: 0 is not "OK" here
	WidgetTimeout   WidgetStatus = 1
	WidgetError     WidgetStatus = 2
	WidgetSuccess   WidgetStatus = 3
)

type Outcome int

const (
	OutcomeFailed Outcome = iota // the zero value is the safe one
	OutcomeCancelled
	OutcomeSucceeded
)

var unknownStatus = expvar.NewInt("widget_status_unknown")

func outcomeOf(s WidgetStatus) Outcome {
	switch s {
	case WidgetSuccess:
		return OutcomeSucceeded
	case WidgetCancelled:
		return OutcomeCancelled
	case WidgetTimeout, WidgetError:
		return OutcomeFailed
	default: // a value this build has never heard of
		unknownStatus.Add(1)
		return OutcomeFailed
	}
}

// normaliseCode is the one place an upstream label becomes a lookup key.
func normaliseCode(s string) string {
	return strings.ToUpper(strings.TrimSpace(s))
}
```

Upper-casing is fine for ASCII machine codes. For human-language labels,
case-insensitive matching is a Unicode question rather than an ASCII one; Go's
`strings.EqualFold` compares "under simple Unicode case-folding, which is a more
general form of case-insensitivity" ([docs](https://pkg.go.dev/strings#EqualFold)).

<details>
<summary>Rust version</summary>

A Rust `match` on a Rust enum is exhaustive, which is exactly why the *wire*
integer needs its own catch-all variant: the compiler can only force you to handle
the values you told it about.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum WidgetStatus {
    Cancelled,
    Timeout,
    Error,
    Success,
    Unknown(i64), // anything this build has never heard of
}

impl From<i64> for WidgetStatus {
    fn from(n: i64) -> Self {
        match n {
            0 => WidgetStatus::Cancelled,
            1 => WidgetStatus::Timeout,
            2 => WidgetStatus::Error,
            3 => WidgetStatus::Success,
            other => WidgetStatus::Unknown(other),
        }
    }
}

#[derive(Debug, PartialEq)]
enum Outcome { Failed, Cancelled, Succeeded }

fn outcome_of(s: WidgetStatus) -> Outcome {
    match s {
        WidgetStatus::Success => Outcome::Succeeded,
        WidgetStatus::Cancelled => Outcome::Cancelled,
        WidgetStatus::Timeout | WidgetStatus::Error | WidgetStatus::Unknown(_) => Outcome::Failed,
    }
}

fn normalise_code(s: &str) -> String {
    s.trim().to_uppercase()
}
```

</details>

<details>
<summary>C++ version</summary>

```cpp
#include <algorithm>
#include <cctype>
#include <string>

enum class WidgetStatus : int { Cancelled = 0, Timeout = 1, Error = 2, Success = 3 };
enum class Outcome { Failed, Cancelled, Succeeded };

// Cast from the wire with static_cast<WidgetStatus>(n): C++ happily holds
// values outside the listed enumerators, so the default branch is reachable.
Outcome outcome_of(WidgetStatus s) {
    switch (s) {
        case WidgetStatus::Success:   return Outcome::Succeeded;
        case WidgetStatus::Cancelled: return Outcome::Cancelled;
        case WidgetStatus::Timeout:
        case WidgetStatus::Error:     return Outcome::Failed;
        default:                      return Outcome::Failed;  // unknown: fail closed
    }
}

std::string normalise_code(std::string s) {
    auto not_space = [](unsigned char c) { return !std::isspace(c); };
    s.erase(s.begin(), std::find_if(s.begin(), s.end(), not_space));
    s.erase(std::find_if(s.rbegin(), s.rend(), not_space).base(), s.end());
    std::transform(s.begin(), s.end(), s.begin(),
                   [](unsigned char c) { return std::toupper(c); });
    return s;
}
```

</details>

<details>
<summary>JavaScript version</summary>

```js
// Mirrors the widget's published enum; cite the source and version here.
const WidgetStatus = Object.freeze({ Cancelled: 0, Timeout: 1, Error: 2, Success: 3 });

function outcomeOf(status) {
  switch (status) {
    case WidgetStatus.Success:   return "succeeded";
    case WidgetStatus.Cancelled: return "cancelled";
    case WidgetStatus.Timeout:
    case WidgetStatus.Error:     return "failed";
    default:                     return "failed"; // unknown: fail closed
  }
}

const normaliseCode = (s) => s.trim().toUpperCase();
```

</details>

## Check yourself

1. An upstream stops sending `accountStatus`. Your Go struct declares it as `int`,
   and in the upstream's enum 0 means "active". Walk through what your service
   does with the next thousand records, and say why no error appears anywhere.
2. A Go struct is reused across loop iterations to save allocations. One record
   has `"status": 1`, the next has `"status": null`. What does the second record
   decode to, and which sentence of the `encoding/json` docs explains it?
3. A teammate wraps every field of a Rust response type in `Option<T>` so that
   "the service never crashes on a bad payload". What did they gain, what did they
   lose, and how would you get the safety back without giving up the gain?
4. Fowler says ignore what you don't need; RFC 9413 says silently accepting faulty
   input causes long-term harm. Give one concrete reader where following Fowler is
   right and one where following him would have hidden a production bug, and say
   what distinguishes them.
5. A config value travels from a Go service as MessagePack, through a library that
   re-encodes it as JSON, into a strictly typed decoder that expects an integer. It
   starts failing with a validation error that mentions neither MessagePack nor
   floats. What is the likely cause, and what are the two places you could fix it?
6. A third-party widget adds a new status value, 4, meaning "user needs to retry".
   Your switch has a `default` that returns the cancelled outcome, because that
   seemed harmless. Describe what users experience, and say which way the default
   should fail and why.
