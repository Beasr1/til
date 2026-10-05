# 2. Exercises

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
- JSON APIs have no field numbers. What plays the role of identity there, and what
  is the equivalent of a renumber?
- Where does this discipline stop being worth it? Is there a real category of
  format where breaking changes are genuinely cheaper than tombstones?
