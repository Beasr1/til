# Schema Evolution — A Course

A short course on a problem that only appears once software is out of your hands:
how to change a data format when you do not control which version of the reader
is running — and, from the other side, how to read a format whose writer you
don't control.

**This is reference learning material.** Everything here is general, checked
against the formats' own documentation (Protocol Buffers, Cap'n Proto, Google's
API Improvement Proposals) and, for decoder behaviour, by running the decoders.
The motivating failures are the ordinary ones: a field renumbered to tidy up the
schema, which compiled cleanly, passed every test, and silently corrupted data on
machines nobody could redeploy; and an upstream field that stopped arriving,
decoded as zero, and turned into a valid-looking answer for every record.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how — the problem that forced each rule comes before the rule.
- Every file ends with **Check yourself** questions. Answers are in
  [05-exercises.md](05-exercises.md).
- Every compatibility claim is quoted from the format's own documentation, not
  recalled.
- Code examples are in Go, with Rust, C++ and JavaScript versions in collapsible
  blocks underneath.

## The one thing to understand first

Every rule in this course is downstream of a single asymmetry:

> **In a distributed or installed system, the reader is usually older than the
> writer — and you cannot make it not be.** Tests run both halves at the same
> version, so they verify the one configuration that never occurs in the field.

And the mechanism that follows from it:

> **A field's identity on the wire is whatever the reader uses to find it.** In a
> tagged binary format that's its number: names are for humans and free to
> change. In JSON it's the name, so a rename is a renumber. Either way, the only
> thing a receiver knows about an unfamiliar field is its identity, and changing
> that identity is never an edit — it's removing one field and adding another.

## Reference implementations

| Source | What it settles |
|---|---|
| [Protocol Buffers — language guide](https://protobuf.dev/programming-guides/proto3/) | Field numbers are permanent, `reserved` exists, what reusing or renumbering costs, and that names matter in the JSON encoding |
| [Protocol Buffers — field presence](https://protobuf.dev/programming-guides/field_presence/) | Why zero and absent are the same bytes for a plain proto3 scalar |
| [Protocol Buffers — proto2 guide](https://protobuf.dev/programming-guides/proto2/) | Why `required` was removed, and why relaxing it is hard when every reader enforces it |
| [Cap'n Proto — language reference](https://capnproto.org/language.html) | The explicit list of what may and may not change in a schema, and the "assume not safe" default |
| [Go `encoding/json`](https://pkg.go.dev/encoding/json) | Missing keys, null on non-pointers, unknown keys, first-error-only type mismatches, `DisallowUnknownFields`, `UseNumber` |
| [serde attributes](https://serde.rs/container-attrs.html) | `rename_all`, `deny_unknown_fields`, `default` — the Rust side of strictness and naming |
| [MessagePack spec](https://github.com/msgpack/msgpack/blob/master/spec.md) | Separate integer and float families, so a re-encoder can change a number's type |
| [AIP-180](https://google.aip.dev/180), [AIP-203](https://google.aip.dev/203), [AIP-126](https://google.aip.dev/126) | Backwards compatibility rules for APIs: renames, semantics, required fields, enum values |
| [Fowler — Tolerant Reader](https://martinfowler.com/bliki/TolerantReader.html) and [RFC 9413](https://www.rfc-editor.org/rfc/rfc9413.html) | The case for lenient readers, and the case against silently accepting faulty input |
| [Sato — Parallel Change](https://martinfowler.com/bliki/ParallelChange.html) | Expand, migrate, contract |
| [Confluent — schema evolution and compatibility](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html) | BACKWARD, FORWARD and FULL compatibility, and which side must deploy first |
| [Semantic Versioning](https://semver.org/) | The version-number half of the same problem, and where it stops helping |

## Reading order

[File 01](01-the-installed-client-problem.md) is the prerequisite for everything
else.

**Part 1 — Owning the schema**

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [The installed-client problem](01-the-installed-client-problem.md) | Say which schema changes are safe without looking them up, explain why a rename is free in protobuf and a renumber in JSON, and say which way relaxing a required field is safe |

**Part 2 — Living with other people's schemas**

| # | File | After this you can… |
|---|------|---------------------|
| 2 | [Consuming payloads you don't control](02-consuming-payloads-you-dont-control.md) | Predict what your decoder does with a missing, null, unknown or mistyped field, keep absent, null and zero apart, and handle enum values you've never seen |
| 3 | [Expand and contract](03-expand-and-contract.md) | Change a field's shape or meaning without breaking a reader, retire a field safely, and add an event field so producer and consumer can deploy in either order |

**Reference**

| # | File | After this you can… |
|---|------|---------------------|
| 4 | [Glossary](04-glossary.md) | Look up a term and find the file that teaches it |
| 5 | [Exercises & answers](05-exercises.md) | Check the model formed, and go deeper |

What isn't written, and belongs here: how tag-based and positional formats differ
(Protocol Buffers and Cap'n Proto versus a packed C struct), what changes when a
schema is part of a *persisted* format rather than a wire format, how Avro's
writer/reader schema resolution compares, and how capability negotiation relates
to schema versioning. The questions at the end of file 05 are the honest list.

## If you're short on time

- **10 minutes:** file 01, the sections "Identity is the number" and "The rules,
  and why each one exists".
- **About to change a schema right now:** file 01's decision table, then file 03's
  phase table.
- **A third-party field just went missing and things are quietly wrong:** file 02,
  "What decoders actually do" and "Three states, not two", then file 03's "Deprecation
  includes a reader audit".
- **Adding a field to an event two teams consume:** file 03, "Events: carry facts,
  and default to the old behaviour".

## The one-paragraph summary of everything

Once software is installed on machines you do not control, you permanently lose
the ability to upgrade both ends of a format at once — so every format you ship
must be readable by versions of your own code that were written before the change
existed. The mechanism that makes this possible is to give each field an identity
on the wire — a **number** in Protocol Buffers or Cap'n Proto, the **name** in
JSON — so a reader encountering one it does not know can skip that field and carry
on. That gives a small set of permanently safe changes (add a new field, rename in
a numbered format, reorder the source, relax a rule once every reader enforcing it
has relaxed) and a set that silently corrupts data or blanks it out: renumbering,
reusing a retired number or name, changing a field's type or meaning, renaming a
JSON key, tightening a required field. The Protocol Buffers documentation states
that a field number "cannot be changed once your message type is in use", and
Cap'n Proto says "any change not listed above should be assumed NOT to be safe".
On the reading side, most decoders are built to be tolerant: Go's `encoding/json`
leaves a missing field at zero, ignores null on non-pointers and ignores unknown
keys, so an upstream rename arrives as a valid-looking zero rather than an error.
The defence is to keep absent, null and zero as distinct states, choose deciding
fields by measuring their presence in real data, be strict in tests and tolerant
but counted in production, give every enum switch a fail-closed unknown branch,
and route on machine codes rather than labels. Every change that feels like
editing a field is really adding one and later removing another — **expand and
contract** — with a reader audit and a fallback counter in between, and for events,
a new field whose absence means "behave as before", so producer and consumer can
deploy in either order. None of these failures are compile errors, and none show
up in a test suite that runs both halves at the same version.

## How to use me

Example questions worth asking:

- "I need to rename a field and change its type. What's the safe sequence?"
- "We control both ends and deploy them together. Which of these rules can I
  actually drop, and what am I betting on?"
- "How would I write a test that would have caught a renumbering — or a JSON
  rename?"
- "Our upstream says a field is always present. How do I check, and what do I do
  if it isn't?"
- "Should this reader reject unknown fields or ignore them? Argue both sides for
  my case."
- "I want to split a field that carries two meanings. Walk me through the phases
  for our producer and its three consumers."
