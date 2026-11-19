# 5. Exercises

Worked answers to every **Check yourself** question, plus things to try.

---

## File 01 — The installed-client problem

**1. Why does a green test suite prove almost nothing about wire compatibility?**

Because it builds both halves from one commit, so it only ever exercises
new-writer-against-new-reader. The configuration it never builds is **new writer →
old reader**, and that is the normal state of any system with installed software in
it: the moment you release, your new writer starts talking to every older reader
still deployed, and it keeps doing so for as long as those machines exist.

The deeper point is that this is not a gap you can close by adding a test to the
same suite, because the suite has only one version of the code in it. Catching it
needs an artefact from the past — a recorded byte stream, or an older build of the
reader kept as a fixture — which is the shape of the answer to question 3 in
"Things to try".

**2. A rename plus a renumber in one commit.**

The rename is free: `userId` → `accountId` changes no bytes, because the name is
compiled away and the number is the identity. An old reader is entirely unaffected
and never learns the name changed.

The renumber is the dangerous half. Say the field was 5 and the one below was 6,
now swapped. The new writer emits the account id tagged 6 and the other field's
value tagged 5. An old reader looks up 5, finds whatever its schema says 5 is, and
interprets the bytes as that type. If the types differ it may fail to parse — the
good outcome. If they happen to be compatible, it parses successfully and stores
each value in the wrong place, with no error anywhere.

What makes this a classic is that the two changes look like one tidy-up commit, and
review attention goes to the rename because that's the part that shows up in the
diff as a word.

**3. Why is a successful parse worse than an error?**

Because an error stops something and gets reported, and a wrong parse propagates.
The bad case is that the old meaning and the new data are structurally compatible —
both integers, both strings — so the reader accepts the value and hands it onward as
if it were the thing it expects. The wrong value then flows into whatever consumed
the old field: a counter, a comparison, a log line, a database column, another
service.

The Protocol Buffers docs list "leaked PII/SPII" among the outcomes, and that is
this case made concrete: a number that used to be innocuous is now, say, an
identifier or an address, and every old reader routes it wherever the old field
went — including into logs that are retained and shipped. The corruption is silent,
it is retroactive in the sense that you cannot tell which records were affected
without knowing each machine's version, and the fix requires knowing the population
of deployed readers.

**4. What does a closed list of safe changes buy you?**

It reverses the burden of proof. A list of prohibitions invites the argument "this
isn't on the forbidden list, so it's fine", and that argument is made in good faith
about changes nobody thought to forbid. A closed list plus "any change not listed
above should be assumed NOT to be safe" means a reviewer's question is always the
same: *which of the four permitted changes is this?* If the answer requires
explanation, the change is unsafe until someone demonstrates otherwise.

This is a general technique for rules that protect against an unbounded space of
mistakes. Allowlists survive imagination failures; denylists don't.

**5. `timeoutMs` becoming seconds.**

Every structural rule is satisfied — same number, same type, same name — so no
tooling complains and the change ships. Then: every **old writer** keeps sending
milliseconds, which the new reader now interprets as seconds, so a 30000 ms timeout
becomes 30000 seconds, roughly eight hours. And every **new writer** sends seconds
to old readers, so a 30 second timeout becomes 30 milliseconds. Both directions
break, in opposite and equally confusing ways, and the symptom in each case is a
timeout behaving absurdly rather than anything resembling a parse failure.

The safe sequence is to treat a meaning change as a field change:

1. Add a **new** field with the next free number — `timeoutSeconds`.
2. Writers populate the new field; keep populating the old one while old readers
   exist.
3. Readers prefer the new field and fall back to the old one when it is absent.
4. Once no reader depends on the old field, stop writing it — and never reuse its
   number.

The old field's tombstone is the record of what happened, which is worth more than
the tidiness of removing it.

**6. Both ends deployed together — what can you relax?**

Honestly, quite a lot: if every writer and reader is replaced in one rollout, you
can renumber and retype freely, because no old reader survives the deploy.

What you are betting on is the completeness and atomicity of that claim, and the
bet has more parts than it first appears:

- **Rollback.** A rollout that is reverted puts old readers back in front of new
  writers, which is the untested configuration with the added detail that you are
  already in an incident.
- **In-flight data.** Anything queued, retried, cached or persisted between the two
  deploys was written by the old schema and is read by the new one. A message
  broker or an outbox turns a synchronous protocol into a stored one, and stored
  data outlives the deploy.
- **Partial rollout.** Canaries, blue/green, a pod that didn't restart, a region
  deployed an hour later.

And the single future event that converts the bet into the installed-client
problem: **someone outside your deployment starts reading the format.** A partner
integration, a mobile app, a CLI a customer pins, an SDK you publish, a debugging
tool somebody kept. At that moment the format becomes permanent and you will not be
told it happened. This is why teams that own both ends still adopt the discipline
early: the cost of following it is a tombstone comment, and the cost of adopting it
late is that the compatibility you need already doesn't exist.

**7. Tags added to a struct that had none.**

Every key in the response changes. Without tags, `encoding/json` uses "the field
name as the object key", so clients have been reading `KeyID`, `OrgID`,
`CreatedAt`. After the change they get `key_id`, `org_id`, `created_at`. To every
existing client this is a rename of every field at once, which in JSON is the
equivalent of renumbering every field in protobuf.

What clients see depends on how they read. A strict decoder errors. Most don't:
a JS client reads `response.KeyID`, gets `undefined`, and passes it on. Pass
`undefined` to a date formatter and you may get a thrown exception that takes the
page down, far from the cause; pass it to a template and you get blank fields.

The service's tests stayed green because they encode and decode with the same
struct. The struct changed on both sides of the test at once, so the test checked
that the struct agrees with itself. The test that would have caught it asserts on
the bytes: marshal a value, decode it into a `map[string]any`, and check that the
documented key names are present and the right type. Better still, keep a
recorded response from the previous release as a fixture and check the new code
still produces the same keys.

**8. Optional in the API, still failing.**

Three likely places, from the table in file 01: a second request type with a
copy-pasted validator that still requires the field; an extraction or mapping step
that treats a record without the field as incomplete and drops it; and a client
SDK, sample app or config loader in another language that refuses to send a
request without it. A `NOT NULL` column or a form schema are two more.

The disagreement is about who enforces. Google's AIPs describe request APIs, where
the server is the only party that checks whether a field is present. Removing
REQUIRED there is a server-side relaxation: old clients keep sending the field and
nothing changes for them; new clients may omit it. The proto2 guide describes a
format where *every reader* checks required fields at parse time. There, the
writer that stops sending the field is the problem, because any stale reader
"will consider messages without this field to be incomplete" and may drop them.

So what decides it is whether any reader you don't control enforces the rule. If
only your server does, relaxing is safe and immediate. If installed readers do,
relax the readers first, wait until the ones you care about have upgraded, and
only then let writers omit the field. Tightening is unsafe in both worlds, because
old writers don't know they're now supposed to send it.

---

## File 02 — Consuming payloads you don't control

**1. An int field the upstream stops sending.**

Every record decodes without error and every record has `accountStatus == 0`,
because `encoding/json` only writes struct fields whose keys it finds; a missing
key leaves the field at its zero value. Zero means "active" in the upstream's
enum, so your service reports every account as active: the open ones, the
closed ones, and the ones that were never opened. If your logic
runs the other way (absent means "no account"), you reject all of them instead.

No error appears because nothing is wrong as far as the decoder is concerned. The
payload is valid JSON, unknown keys (perhaps the field's new name) are ignored by
default, and a missing key is not a mismatch. The first signal is usually a human
noticing a strange distribution of outcomes, which is why file 02 recommends
making "absent" its own counted outcome: the dashboard shows "absent" jump from
near zero to everything on the day of the change.

The fix has two parts. Declare the field `*int` (or a three-state type) so absent
is `nil` rather than `0`, and handle `nil` explicitly. And find out whether the
data moved elsewhere, which is a conversation with the upstream, not a decoding
problem.

**2. A reused struct and a null.**

The second record decodes to `status: 1`, the previous record's value. The
sentence is: "unmarshaling a JSON null into any other Go type has no effect on the
value and produces no error." `int` isn't an interface, map, pointer or slice, so
null leaves it alone, and "alone" means whatever was there before. With a fresh
struct per record you'd get 0, which is wrong in a different way; with a reused
one you get the last record's answer, which is worse, because it looks plausible.

Two fixes, use both: decode each record into a fresh value, and declare fields
where null is possible as pointers, so null becomes `nil`.

**3. Every field wrapped in `Option<T>`.**

They gained the property that the decoder never fails on a missing or null field.
They lost the property that it fails when a field the code *depends on* goes
missing. serde's default — a missing non-`Option` field is an error — was doing
useful work: it was the alarm that tells you the upstream renamed or dropped a
key. With every field optional, a rename becomes `None`, the code carries on with
a hole in it, and you're back to Go's behaviour by another route.

Getting both: keep the fields the flow actually needs non-optional, or make them
`Option` but check them at the boundary and turn `None` into an explicit,
counted outcome. Add `#[serde(deny_unknown_fields)]` to a test-only variant of the
type (or decode captured real payloads in a test with it) so that a new or renamed
key fails the test suite rather than production. Production stays tolerant of
fields you don't read; the test is strict about the payload's shape.

**4. Fowler versus RFC 9413.**

Following Fowler is right for a reader that consumes three fields of a large
third-party object and ignores the rest. The provider adds a field, renames one
you don't read, restructures a part you never look at: none of it should break
you, and with a tolerant reader none of it does.

Following him would have hidden a bug when the tolerance extends to the fields you
*do* read. A reader that treats a missing status as zero, or swallows a type error
and uses the partially decoded struct, is being tolerant of exactly the input it
depends on. The upstream's change still breaks you; you just find out from
customers rather than from an error.

What distinguishes them is whether the input in question is something you ignore
or something you rely on. Fowler's advice is about the first: "only take the
elements you need, ignore anything you don't". RFC 9413's warning — that harm
accumulates "if implementations silently accept faulty input" — is about the
second. Read that way they don't conflict: ignore what you don't use, and be
loud about what you do.

**5. A config value that fails after a MessagePack-to-JSON hop.**

Likely cause: an integer became a float on the way. MessagePack has separate
Integer and Float families; something in the chain — the re-encoding library, or a
generic map it passed through — represented the number as a float, so the JSON
says `254.0` where it used to say `254`. A strict integer decoder rejects it. The
error mentions validation, or the field name, because that's where the decoder
noticed; it doesn't know the history of the bytes. If the error is also caught and
replaced with a generic one somewhere above, the trail is cold.

Two places to fix it. At the **consumer**, decode that field leniently: accept an
integral float as an integer and reject anything with a fractional part, as the
`WholeNumber` type in file 02 does. At the **re-encoder**, preserve the integer
type, if you own it or can configure it. The consumer fix is the one you can ship
alone, and it's the right one when you don't control the hop. Whichever you do,
stop swallowing the decode exception: the misleading error is half of why this
takes a day.

**6. A new status value falling into "cancelled".**

Users who were told by the widget to retry are treated as if they cancelled. They
see whatever your product shows a user who walked away — perhaps nothing, perhaps
"you cancelled, start again" — and no failure is recorded, so your failure rate
looks fine while users drop out. If "cancelled" triggers a refund or releases a
reservation, the misrouting has side effects too.

The default should fail **closed**, to the outcome that does nothing irreversible
and is visible: "failed", counted on a metric named for unknown values. The
reasoning: you cannot know what an unknown value means, so the only safe
assumption is that it isn't success, and the counter turns "the upstream added
something" into an alert instead of a mystery. "Cancelled" felt harmless only
because it seemed neutral; it was still a specific claim about what the user did.

---

## File 03 — Expand and contract

**1. A `currencyUnit` field to say what `amount` means.**

It changes the meaning of `amount` in place while looking like an additive change.
Old readers don't know `currencyUnit` exists, ignore it as tolerant readers
should, and read `amount` as pounds. Every record from the new product line is
then off by a factor of a hundred, and no schema check notices, because every
structural rule is satisfied. It also leaves `amount` permanently ambiguous: every
future reader has to remember to check another field first.

Expand and contract adds a field per meaning instead: keep `amount` as pounds for
everything that has always used it, and add `amountPence` (or, better, an integer
minor-units field plus an explicit currency code) for the new line. Writers fill
both where both make sense; readers move to the new field one by one, counting
fallbacks; the old field retires when nobody reads it.

**2. Knowing when to move from step 3 to step 4.**

You know from the fallback counter. During step 3, every reader that still takes
the old field increments a metric when it does. When that metric has been flat at
zero for a period longer than your slowest reader's release cycle — and for
installed clients, long enough that the versions you care about have upgraded —
you can stop writing the old field.

If you guess and guess early, some reader you missed sees the old field go absent.
Depending on how it was written, that's a blank on a page, a zero that means
something, or a rule that stops firing — file 02's failure modes, now caused by
you. The counter costs one line per reader; the guess costs an incident.

**3. An upstream will stop sending a field you only read.**

Before the date: find every reader of the field — rules, second code paths,
templates, frontends, SDKs, stored columns, allowlists, dashboards, alert rules —
using file 03's table as the checklist. For each, decide what replaces the field
(often a different field the upstream already sends, which you should measure for
presence as file 02 describes). Change the readers to prefer the replacement and
fall back to the old field while it still arrives, counting fallbacks. Make sure
every reader turns "field absent" into an explicit outcome rather than a silent
return. On the date, watch the fallback and "cannot evaluate" counters.

"We don't write it, so we're not affected" is wrong because removal affects
readers, not writers. Every rule you have that reads the field is a party to its
removal. The worst version is a parse-error-then-return path: the rule doesn't
fail, it just stops applying, and that looks identical to "applied and found
nothing" until someone asks why a warning hasn't gone out in a month.

**4. A rename combined with a tightened validator.**

They skipped step 1: readers accepting both values. By tightening the validator to
the listed values in the same release as the rename, the server became a strict
reader that had never heard of the old value, while every caller was still sending
it. That's a writer-before-reader change in the unsafe direction.

The order that works: release one makes readers accept both the old and the new
spelling (and the tightening, if wanted, lists both). Then callers switch to the
new spelling, at their own pace. Release three removes the old spelling once
nothing sends it, which you can see from a counter on the old value. If the value
also flows the other way, in a token or a response, each side needs to accept both
before the other side starts sending the new one.

**5. A `retryable: bool` with the consumer shipping first.**

Until the producer ships, no event carries `retryable`. In Go the absent `bool`
decodes as `false`, so the new consumer retries nothing: every transient failure
that used to be retried is now final. The bug is invisible in the consumer's
tests, which feed it events that include the field.

To make the deploy order not matter, the absent value has to mean "behave as
before". Two ways. Make the field a `*bool` (or `Option<bool>`), and treat `nil`
as "apply the old retry rule". Or, better, send facts instead of the verdict: add
the failure's error code or category, which an old producer simply doesn't send,
and let the consumer decide retryability from the code — an absent code then falls
through to the old rule naturally. The second keeps the retry policy in the
service that acts on it.

**6. Dropping a column in the same release that stops reading it.**

During the rollout, some instances run the new code and some still run the old
code. The migration runs once, at the start or the end. If it runs first, the old
instances — still serving traffic — query a column that no longer exists and fail
every request that touches it, until the rollout replaces them. If it runs last,
the window is shorter but the problem returns on any rollback.

A rollback restores the old code but not the data. The down migration can
re-create the column; it cannot re-create what was in it, so the rolled-back code
reads empty values where it expects real ones, which is file 02's absent-field
bug, now in your own database. The fix is to split it: one release stops reading
the column (and stops writing it), and a later release, after the first has fully
rolled out and you're confident you won't roll back past it, drops it.

## Things to try

**Break it deliberately, in the smallest possible repo.** Define a two-field
message, serialise a value with it, then swap the two field numbers and
deserialise the *old bytes* with the new schema. Do it once with two fields of the
same type and once with different types. The first gives you silent corruption and
the second gives you an error, which is the whole lesson in two runs.

**Read a real schema's history.** Find a long-lived `.proto` or `.capnp` in any
mature open-source project and read its git history for the field-number column
alone. You will see append-only discipline, reserved tombstones, and — often — a
commit where someone tried to tidy and was reverted.

**Build the test your suite is missing.** Serialise a corpus of messages with
today's schema and commit the *bytes* as a fixture. Then have CI deserialise those
bytes with the current schema on every run. That is a test of old-writer →
new-reader that does not require keeping an old build around, and it fails
precisely when someone changes a number or a type.

**Look for the meaning-change version.** Grep a schema you own for units in field
names — `Ms`, `Seconds`, `Bytes`, `Percent`. Every one of those is a field whose
meaning is load-bearing and unenforced. Then look for the fields that *don't* carry
units and ask how a reader would know.

**Run the decoder table yourself.** Take file 02's table of decoder behaviours and
reproduce each cell in the language you use most: a missing key, a null, an unknown
key, a wrong type, `254.0` into an integer, a key in the wrong case. It takes half
an hour and replaces every assumption you had about your decoder with something
you've seen.

**Measure a real upstream.** Pick one third-party payload you consume and count,
across a day of real traffic, how often each field you read is absent, null, or
zero. Compare that with what the upstream's documentation promises. The gap is the
list of fields that shouldn't be deciding anything on their own.

**Find your silent returns.** Grep a service for the pattern "parse, log on error,
return" in code that applies a business rule. For each, ask what the rule's
outcome is when the field is missing, and whether anyone would notice.

---

## Questions worth asking me

- How do tag-based formats (Protocol Buffers, Cap'n Proto) differ from positional
  ones (a packed C struct, a fixed-width record) in what evolution is even
  possible? Where does that leave a format with no field numbers at all?
- What changes when the schema describes **persisted** data rather than a wire
  format? The old reader problem becomes an old *data* problem, and data doesn't
  get upgraded by deploying.
- How does capability negotiation relate to schema versioning — when should two
  parties agree what they support up front rather than relying on skip-unknown?
- JSON APIs have no field numbers, so the name is the identity (file 01). What does
  JSON Schema, OpenAPI or a consumer-driven contract test add on top of that, and
  which of file 02's failures would each one catch?
- How do Avro and Protocol Buffers differ in what a reader does with a field it
  expects and doesn't find? Avro resolves the writer's schema against the reader's;
  what does that buy over file 02's three-state decoding?
- When an upstream's enum changes often, AIP-126 says use a string. What does a
  consumer lose by switching from a closed enum to a string, and how do you keep
  the fail-closed default?
- Where does this discipline stop being worth it? Is there a real category of
  format where breaking changes are genuinely cheaper than tombstones?
