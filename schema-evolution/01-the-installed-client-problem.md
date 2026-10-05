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
  wire. An old reader carries on reading the same number.
- **Renumbering a field is catastrophic.** And it is the change most likely to be
  proposed as a tidy-up.

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
| Rename a field | ✅ | the name isn't on the wire |
| Reorder fields in the source | ✅ | position isn't the identity |
| Reorder/renumber the field *numbers* | ❌ | old readers misread every moved field |
| Reuse a retired number for something new | ❌ | old readers apply the old meaning to new data |
| Change a field's type | ❌ | old readers parse the old type from new bytes |
| Change a default value | ❌ | absent-field behaviour differs by version |
| Remove a field | ⚠️ | see below |

> ⚠️ **Append-only means append at the *end of the number space*, not the end of
> the file.** Reusing a gap left by a deleted field is the same mistake as
> renumbering, and it looks tidier, which is what makes it dangerous.

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
