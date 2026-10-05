# Schema Evolution — A Course

A short course on a problem that only appears once software is out of your hands:
how to change a data format when you do not control which version of the reader
is running.

**This is reference learning material.** Everything here is general, checked
against the Protocol Buffers language guide and the Cap'n Proto language
reference. The motivating failure is the ordinary one: a field renumbered to tidy
up the schema, which compiled cleanly, passed every test, and silently corrupted
data on machines nobody could redeploy.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how — the problem that forced each rule comes before the rule.
- Every file ends with **Check yourself** questions. Answers are in
  [02-exercises.md](02-exercises.md).
- Every compatibility claim is quoted from the format's own documentation, not
  recalled.

## The one thing to understand first

Every rule in this course is downstream of a single asymmetry:

> **In a distributed or installed system, the reader is usually older than the
> writer — and you cannot make it not be.** Tests run both halves at the same
> version, so they verify the one configuration that never occurs in the field.

And the mechanism that follows from it:

> **A field's identity on the wire is its number, not its name.** Names are for
> humans and are free to change; numbers are the protocol and can never change,
> because the only thing a receiver knows about an unfamiliar field is its number.

## Reference implementations

| Source | What it settles |
|---|---|
| [Protocol Buffers — language guide](https://protobuf.dev/programming-guides/proto3/) | Field numbers are permanent, `reserved` exists, and what reusing or renumbering costs |
| [Cap'n Proto — language reference](https://capnproto.org/language.html) | The explicit list of what may and may not change in a schema, and the "assume not safe" default |
| [Semantic Versioning](https://semver.org/) | The version-number half of the same problem, and where it stops helping |

## Reading order

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [The installed-client problem](01-the-installed-client-problem.md) | Say which schema changes are safe without looking them up, and explain why a rename is free and a renumber is corruption |
| 2 | [Exercises & answers](02-exercises.md) | Check the model formed, and go deeper |

One chapter for now, and it is deliberately the general one. What isn't written,
and belongs here: how tag-based and positional formats differ (Protocol Buffers
and Cap'n Proto versus a packed C struct), what changes when a schema is part of a
*persisted* format rather than a wire format, and how capability negotiation
relates to schema versioning. The questions at the end of file 02 are the honest
list.

## If you're short on time

- **10 minutes:** file 01, the sections "Identity is the number" and "The rules,
  and why each one exists".
- **About to change a schema right now:** file 01's decision table, then its
  "What to do instead of removing a field".
- **Explaining to a colleague why their tidy-up is dangerous:** file 01's opening
  and the teacher's aside after it.

## The one-paragraph summary of everything

Once software is installed on machines you do not control, you permanently lose
the ability to upgrade both ends of a format at once — so every format you ship
must be readable by versions of your own code that were written before the change
existed. The mechanism that makes this possible is to give each field a **number**
that is its identity on the wire, independent of its name or its position in the
source, so a reader encountering a number it does not know can skip that field and
carry on. That gives a small set of permanently safe changes — add a new field with
a number higher than every existing one, rename anything, reorder the source —
and a small set of changes that silently corrupt data: renumbering a field, reusing
a retired number, or changing a field's type. The Protocol Buffers documentation
states that a field number "cannot be changed once your message type is in use
because it identifies the field in the message wire format", and that renumbering
"effectively deletes and re-adds all the fields involved". Cap'n Proto is blunter
still: it lists what you may change and says "any change not listed above should
be assumed NOT to be safe". Crucially, none of these failures are compile errors
and none show up in a test suite that runs both halves at the same version — the
broken configuration is old-reader-meets-new-writer, which is exactly the
configuration your tests never build. So the discipline has to be a rule you
follow rather than a check you rely on, and the cheap enforcement is to make the
schema append-only by convention and review every diff that touches a number.

## How to use me

Example questions worth asking:

- "I need to rename a field and change its type. What's the safe sequence?"
- "We control both ends and deploy them together. Which of these rules can I
  actually drop, and what am I betting on?"
- "How would I write a test that would have caught a renumbering?"
- "What's the equivalent discipline for a JSON API with no schema numbers at all?"
