# 1. The installed-client problem

## The problem that makes all the rules

You have a data format — a wire protocol between two services, or between an
application and a component installed on someone's machine. You want to change
it. You control the source of both halves, your test suite is green, and the
change is obviously correct.

The thing that makes this hard is not the change. It is that **you do not control
which version is running.**

If the format crosses a network between services you deploy together, you have
some control: a coordinated release is unpleasant but possible. If it crosses a
boundary into software that is *installed* — a desktop agent, a mobile app, a
device daemon, a library a customer compiled against — you have none. Some
machine is running a version from eighteen months ago, its owner is not going to
upgrade it this week, and your new writer is going to send data to it tomorrow.

So the requirement is uncomfortable but simple:

> **Every version of the format must be readable by every reader you have ever
> shipped.** Not "the previous one" — every one that is still out there.

## Why your tests never catch this

Sit with why this class of bug survives good engineering practice, because it
explains the whole discipline.

A test suite builds both halves from the same commit. It exercises
new-writer-against-new-reader, which is one configuration. The field has at least
three:

| Configuration | Occurs | Tested by default |
|---|---|---|
| new writer → new reader | after everyone upgrades | ✅ |
| **new writer → old reader** | **immediately, and for years** | ❌ |
| old writer → new reader | during a rollback, or a lagging client | ❌ |

The middle row is the normal state of a deployed system and the one nobody's CI
builds. That is why format compatibility is enforced by *rules you follow* rather
than *errors you get* — and why the failure arrives as corrupted data on a
customer's machine rather than a red build.

> **Teacher's aside.** People reach for version numbers here, and version numbers
> do not solve it. Semantic versioning tells a *human* that something
> incompatible happened; it doesn't help a reader parse bytes it has never seen a
> schema for. A major-version bump is a statement of intent, not a mechanism. The
> mechanism has to be in the format itself: the reader must be able to encounter
> something it does not understand and keep going. Everything below is in service
> of that one capability.

## Identity is the number, not the name

The mechanism is to give every field a **number**, and to make that number the
field's identity on the wire.

Consider a reader that knows fields 1, 2 and 3, receiving data that also contains
field 4. It has never seen a schema mentioning field 4. What can it do? Only one
thing usefully: recognise that a field it doesn't know is present, skip over it,
and carry on with the fields it does know. For that to work, the encoding must
tell it the field's *number* and *how long it is*, and nothing else about it needs
to be comprehensible.

This is why the field name is irrelevant on the wire. The name exists for the
programmer; it is compiled away. Cap'n Proto's language reference says so
directly:

> Any symbolic name can be changed, as long as the type ID / ordinal numbers stay
> the same.

And the Protocol Buffers guide states the counterpart about numbers:

> This number cannot be changed once your message type is in use because it
> identifies the field in the message wire format.

Two consequences follow, and they surprise people in opposite directions:

- **Renaming a field is free.** `userId` → `accountId` changes nothing on the
  wire. An old reader carries on reading the same number. (That holds for a
  format that puts numbers on the wire. JSON doesn't — see the next section.)
- **Renumbering a field is catastrophic.** And it is the change most likely to be
  proposed as a tidy-up.

## In JSON, the name *is* the number

JSON, and any self-describing text format like it, puts the field **name** on the
wire. There's no number, so the name takes over the job: it's the only thing a
reader knows about a field. Everything this chapter says about numbers applies to
names, and a rename in JSON is a renumber.

The Protocol Buffers guide says so about its own JSON encoding. Reusing a deleted
field's name "is generally safe, except when using TextProto or JSON encodings
where the field name is serialized", and so "you should also reserve the field
name to allow JSON and TextFormat encodings of your message to continue to parse"
([proto3 guide, "Deleting Fields"](https://protobuf.dev/programming-guides/proto3/)).

The failure is quieter than a renumber. A reader that looks up a name and doesn't
find it gets "absent", and as [file 02](02-consuming-payloads-you-dont-control.md)
shows, most decoders turn absent into a zero or a blank without a word. A producer
renames `customerID` to `customerId`; every consumer that reads with a default
renders an empty box.

Names also change without anyone deciding to change them:

| How the name changes | Mechanism |
|---|---|
| Tags added to a Go struct that had none | `encoding/json` uses "the field name as the object key" unless a `json` tag says otherwise ([docs](https://pkg.go.dev/encoding/json#Marshal)). An untagged struct sends `KeyID`; tagging it `key_id` renames every key for every client. |
| A blanket case rule in Rust | serde's `rename_all` will "rename all the fields … according to the given case convention" ([container attributes](https://serde.rs/container-attrs.html)), so the Rust identifier becomes the contract: a field called `dob` goes out as `"dob"`, never `"dateOfBirth"`. |
| A second encoder on the same type | Each Go library reads its own tag. [`vmihailenco/msgpack`](https://github.com/vmihailenco/msgpack) reads the `msgpack` tag and, if there isn't one, falls back to the Go field name (`getFields` in `types.go`, v5.4.1), unless told to use another tag with `SetCustomStructTag`. A type with only `json` tags is sent with different names over MessagePack. |
| A flattening step | State saved to a flat key-value store keeps only the fields someone remembered to flatten. Everything else is silently absent on the next read. |

Three habits close most of this:

- **Define each wire name once.** A constant, or one generated type shared by both
  sides, so a typo is a compile error rather than an empty value.
- **Test the bytes, not the struct.** Serialise, then assert that each expected
  key is present, non-empty and of the right JSON type. Do it once per variant
  of the payload, and write-then-read through any store the data passes through.
  A test that encodes and decodes with the same struct passes whatever the names
  are, because both halves share the mistake.
- **Keep numbers numbers.** `Number.prototype.toFixed` returns "a string
  representing the given number using fixed-point notation"
  ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed)),
  so `score.toFixed(2)` in a response handler changes the field's type from
  number to string. Round numerically, or format only at the presentation edge.

```go
// Wire names, defined once. Producer, consumer and tests all use these.
const (
	KeyOrderID    = "orderId"
	KeyTotalCents = "totalCents"
)

// Order crosses two encoders, so it carries both tags. Without the msgpack
// tag, the msgpack encoder would fall back to the Go names "ID" and "TotalCents".
type Order struct {
	ID         string `json:"orderId"    msgpack:"orderId"`
	TotalCents int64  `json:"totalCents" msgpack:"totalCents"`
}

// TestOrderWireNames checks the bytes, not the struct: it fails if anyone
// renames a tag, drops one, or turns a number into a string.
func TestOrderWireNames(t *testing.T) {
	b, err := json.Marshal(Order{ID: "o-1", TotalCents: 1250})
	if err != nil {
		t.Fatal(err)
	}
	var got map[string]any
	if err := json.Unmarshal(b, &got); err != nil {
		t.Fatal(err)
	}
	if v, ok := got[KeyOrderID].(string); !ok || v == "" {
		t.Errorf("%s missing or empty in %s", KeyOrderID, b)
	}
	if _, ok := got[KeyTotalCents].(float64); !ok { // JSON numbers decode as float64
		t.Errorf("%s missing or not a number in %s", KeyTotalCents, b)
	}
}
```

<details>
<summary>Rust version</summary>

```rust
pub const KEY_ORDER_ID: &str = "orderId";
pub const KEY_TOTAL_CENTS: &str = "totalCents";

#[derive(serde::Serialize)]
pub struct Order {
    #[serde(rename = "orderId")] // explicit, so renaming the Rust field is free again
    pub id: String,
    #[serde(rename = "totalCents")]
    pub total_cents: i64,
}

#[test]
fn order_wire_names() {
    let v = serde_json::to_value(Order { id: "o-1".into(), total_cents: 1250 }).unwrap();
    assert!(v[KEY_ORDER_ID].as_str().is_some_and(|s| !s.is_empty()), "{v}");
    assert!(v[KEY_TOTAL_CENTS].is_i64(), "{v}");
}
```

</details>

<details>
<summary>C++ version (nlohmann/json)</summary>

```cpp
#include <cassert>
#include <string>
#include <nlohmann/json.hpp>

constexpr const char* kOrderId = "orderId";
constexpr const char* kTotalCents = "totalCents";

struct Order { std::string id; long long total_cents; };

nlohmann::json to_json(const Order& o) {
    return {{kOrderId, o.id}, {kTotalCents, o.total_cents}};
}

void test_order_wire_names() {
    auto j = nlohmann::json::parse(to_json({"o-1", 1250}).dump());  // through the bytes
    assert(j.contains(kOrderId) && j[kOrderId].is_string() && !j[kOrderId].get<std::string>().empty());
    assert(j.contains(kTotalCents) && j[kTotalCents].is_number_integer());
}
```

</details>

<details>
<summary>JavaScript version</summary>

```js
const test = require("node:test");
const assert = require("node:assert");

const KEY_ORDER_ID = "orderId";
const KEY_TOTAL_CENTS = "totalCents";

const toWire = (order) => ({ [KEY_ORDER_ID]: order.id, [KEY_TOTAL_CENTS]: order.totalCents });

test("order wire names", () => {
  const got = JSON.parse(JSON.stringify(toWire({ id: "o-1", totalCents: 1250 })));
  assert.ok(typeof got[KEY_ORDER_ID] === "string" && got[KEY_ORDER_ID] !== "");
  assert.ok(Number.isInteger(got[KEY_TOTAL_CENTS])); // not "12.50" from toFixed
});
```

</details>

## Why a renumber is worse than a deletion

Say field 5 is `retryCount` (an integer) and field 6 is `emailAddress` (a string),
and someone swaps them so the schema reads in a nicer order.

The new writer now emits the email address tagged as field 5. An old reader looks
up field 5, finds `retryCount`, and interprets the bytes accordingly. Depending on
the encoding it either fails to parse, or — worse — succeeds, and now a number
derived from somebody's email address is sitting in a counter. Nothing errors. The
Protocol Buffers documentation names this exactly:

> renumbering fields (sometimes done to achieve a more aesthetically pleasing
> number order for fields). Renumbering effectively deletes and re-adds all the
> fields involved in the renumbering, resulting in incompatible wire-format
> changes.

and is explicit about what reuse can cost:

> Reusing a field number makes decoding wire-format messages ambiguous.

with consequences listed as "developer time lost to debugging, a parse/merge error
(best case scenario), leaked PII/SPII, data corruption".

Note the phrase **best case scenario** attached to the *error*. An error is the
good outcome, because it is visible. The bad outcome is a successful parse of the
wrong thing — and the leaked-PII item in that list is the concrete version of it:
a field that used to hold something innocuous now carries something sensitive, and
every old reader routes it wherever the old field went, including into logs.

## The rules, and why each one exists

Cap'n Proto states its safe changes as a closed list and then closes the door:

> New fields, enumerants, and methods may be added to structs, enums, and
> interfaces, respectively, as long as each new member's number is larger than all
> previous members.

> Members can be re-arranged in the source code, so long as their numbers stay the
> same.

> You cannot change a field, method, or enumerant's number.

> You cannot change a field or method parameter's type or default value.

> Any change not listed above should be assumed NOT to be safe.

That last line is the most useful sentence in this chapter. The default is unsafe,
and the burden of proof is on the change.

| Change | Safe? | Why |
|---|---|---|
| Add a field with a number above every existing one | ✅ | old readers skip an unknown number |
| Rename a field | ✅ | the name isn't on the wire (in JSON it is; see above) |
| Reorder fields in the source | ✅ | position isn't the identity |
| Reorder/renumber the field *numbers* | ❌ | old readers misread every moved field |
| Reuse a retired number for something new | ❌ | old readers apply the old meaning to new data |
| Change a field's type | ❌ | old readers parse the old type from new bytes |
| Change a default value | ❌ | absent-field behaviour differs by version |
| Make an optional field required | ❌ | old writers don't send it |
| Make a required field optional | ⚠️ | safe only once every reader that enforces it has relaxed; see below |
| Remove a field | ⚠️ | see below |

> ⚠️ **Append-only means append at the *end of the number space*, not the end of
> the file.** Reusing a gap left by a deleted field is the same mistake as
> renumbering, and it looks tidier, which is what makes it dangerous.

### Required is a promise kept in several places

Making a field required and making it optional look like the same kind of edit in
opposite directions. They aren't, and the sources seem to disagree about which is
safe until you ask who is doing the enforcing.

Google's API guidance treats the two as asymmetric. [AIP-203](https://google.aip.dev/203)
lists "adding REQUIRED to an existing field previously considered OPTIONAL" as
breaking, and "removing REQUIRED from an existing field" as compatible;
[AIP-180](https://google.aip.dev/180) adds that "new required fields must not be
added to existing request messages or resources". Tightening breaks every old
writer that doesn't send the field. Relaxing breaks nobody.

The [proto2 guide](https://protobuf.dev/programming-guides/proto2/) says nearly the
opposite: "It is nearly impossible to safely change a field from `required` to
`optional`", because "if there is any chance that a stale reader exists, it will
consider messages without this field to be incomplete" and "may reject or drop
them". Required fields were troublesome enough that they "were removed from proto3
and editions".

Both are right, about different enforcers. In a request API the server is the
only reader that checks the rule, so relaxing it is a server change, and old
clients that keep sending the field are unaffected. In proto2 every reader checks
the rule at parse time, so a writer that starts omitting the field breaks every
reader that hasn't relaxed yet. The general form: **relaxing is safe once every
reader that enforces the rule has relaxed it, and only then may writers stop
sending the field.** Tightening breaks old writers straight away, whatever the
order.

Which leads to the practical problem. A required rule is rarely enforced in one
place:

| Where required-ness hides | How it bites after you relax the "real" one |
|---|---|
| The request validator | the one everyone remembers |
| A second request type with a copy-pasted validator | one endpoint accepts the request, its twin still returns 400 |
| An extraction or mapping step that checks the record is complete | the request is accepted and then dropped as invalid |
| Logic that uses the field to recognise or classify the input | the input is accepted and silently not recognised |
| An SDK, sample app or config loader in another language | the server accepts it, the client never sends it |
| A `NOT NULL` column or a form schema | the write fails further down, far from the change |

So a contract change is a search, not an edit: grep for the field name in every
repository and language that touches the contract, and where you can, replace
copies of a rule with one shared function, or with a table test that runs every
request type through the same cases.

When you *add* an optional field, give it its own nullable type, validate it only
if it's present, and give its validation failures their own error codes. An
invalid optional field should never be reported as if a required one were
missing, or callers can't tell which mistake they made.

The installed-client version of this has one more twist. Suppose deployed server
versions are known to set a field wrongly — say older builds report `success:
true` on a rejection. A client can't trust that field until every server is
upgraded, and with installed software that may be never. The client's way out is
to **derive its verdict from a field that has always been right** in every
deployed version, such as the terminal status itself, and to treat any value it
doesn't recognise as failure.

## What to do instead of removing a field

Removing a field is where the two formats' advice differs in presentation and
agrees in substance.

Protocol Buffers supports removal but requires you to *retire the number*:

> You must reserve the deleted field number. If you do not reserve the field
> number, it is possible for a developer to reuse that number in the future.

and warns:

> Field numbers should never be reused. Never take a field number out of the
> reserved list for reuse with a new field definition.

Cap'n Proto doesn't enumerate removal among its safe changes at all, which by its
own "assume NOT safe" default means don't.

The practical discipline that satisfies both, and the one to reach for:

1. **Stop writing the field.** The writer omits it; readers that expect it must
   already tolerate absence, which is the argument for making new fields optional
   from the start.
2. **Stop reading the field**, in a later release, once every reader you care
   about has stopped needing it.
3. **Never reuse the number.** Mark it reserved if the format supports saying so,
   and by comment if it doesn't.

The field stays in the schema as a tombstone. That feels like clutter and is the
cheapest possible insurance: the clutter is visible and a reused number is not.

## The corollary people miss: the same applies to meaning

Everything above is about *structure*. There is a quieter version about
*semantics*, and it has no mechanical protection at all.

If field 7 is `timeoutMs` and you decide it now means seconds, nothing in any
schema notices. Every rule above is satisfied — same number, same type, same
name — and every old writer is now sending values a thousand times too small.

The same trap applies to widening an enum. Adding a new enum value is structurally
safe, but an old reader has no case for it. What it *does* with an unrecognised
value is the real question, and if the answer is "hits the default branch and
treats it as the first case", you have shipped a silent misbehaviour through a
technically compatible change.

> **The rule that generalises:** a field's number is its identity, and its
> *documented meaning* is part of its identity too. Changing the meaning is
> changing the field, and the safe way to change a field is to add a new one.

## Check yourself

1. Why does a test suite that passes on every commit provide almost no evidence
   about wire compatibility? Name the configuration it never builds, and say why
   that configuration is the normal one in the field.
2. A colleague renames `userId` to `accountId` and, in the same commit, swaps its
   number with the field below it to keep the schema alphabetical. Which half is
   free and which is dangerous — and what does an old reader do with the next
   message?
3. The Protocol Buffers docs call a parse error the "best case scenario" when a
   field number is reused. What is the worse case, and why is it worse despite
   nothing appearing to go wrong?
4. Cap'n Proto lists four safe changes and then says any change not listed should
   be assumed not safe. What does phrasing it as a closed list rather than a list
   of prohibitions buy you, in terms of how reviews go?
5. You need to change `timeoutMs` to hold seconds instead of milliseconds. Every
   structural rule permits it. Describe what will actually happen after release,
   and give the safe sequence instead.
6. Your team owns both ends of a protocol and deploys them together in one
   Kubernetes rollout. Which of these rules can you genuinely relax, and what
   exactly are you betting on? What single future event turns that bet into the
   installed-client problem?
7. A Go service has returned an untagged struct for a year. A colleague adds
   `json:"key_id"`-style tags to every field to match the house style, and the
   service's own tests stay green. What do existing clients see, why didn't the
   tests notice, and what test would have?
8. An API makes a field optional, and requests without it still fail. Name three
   places the old rule might still live. Then explain why relaxing that field is
   called compatible by Google's AIPs and "nearly impossible" by the proto2 guide,
   and what decides which applies to your system.
