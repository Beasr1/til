# 12. Retiring what others still use

Assumes [file 01](01-foundations.md) for the vocabulary, and
[file 06 §6.9](06-describing-apis.md#69-one-call-shared-versions-that-never-change-pinned) for
versions that never change and the workflows that pin them. §12.4 and §12.5 assume you know what a
database transaction and a connection pool are.

## The problem

An engine that people build on fills up. Workflows made to try something out, API calls nobody
calls any more, named values for a partner who left. Sooner or later someone wants them gone, and
the obvious button is Delete. It fails three ways.

- **Something still points at it.** A workflow pins version 3 of a saved call
  ([06 §6.9](06-describing-apis.md#69-one-call-shared-versions-that-never-change-pinned)), so deleting the
  call breaks the workflow the next time it runs. The pin lives inside a JSON document, so no
  foreign key is there to refuse the delete.
- **The record loses its meaning.** A run from last month says it called version 3. If version 3
  is gone, "what did that run do?" has no answer, and an audit asks exactly that.
- **A name comes back.** A workflow is served at a URL made from its name. Delete it, let someone
  make a new one with the same name, and an old caller still calling that URL reaches a workflow
  it never meant to call.

The fix is to retire things instead: take them out of use, keep them, show that they're retired,
and let them come back. Most of this chapter is the detail that makes that hold: what "in use"
means (§12.3), the race between checking and retiring (§12.4), a lock that can jam a connection
pool (§12.5), names (§12.6), secrets (§12.7), a read-only screen (§12.8), and keeping what
happened (§12.9).

## 12.1 Delete, soft delete, archive

Three words get used for this, and they aren't the same.

- **Delete** removes the row. Nothing points at it any more, because nothing can.
- **Soft delete** keeps the row but hides it: a `deleted_at` column that every query has to
  remember to filter on. The row is meant to be gone, and mostly is.
- **Archive** keeps the row and *shows* it, as retired. It's a state, like "draft" or "published",
  with a way back.

The case against soft deletion is worth reading before building any of these. In "Soft Deletion
Probably Isn't Worth It" ([brandur.org/soft-deletion](https://brandur.org/soft-deletion)): "The
first is that soft deletion logic bleeds out into all parts of your code", and "Another consequence
of soft deletion is that foreign keys are effectively lost." Both are true of archiving too. The
difference is intent. A soft-deleted row is something the system pretends isn't there, so every
place that forgets the filter is a bug. An archived row is something the system *says* is retired,
so a run or a revision that points at it still means something: "it called version 3, which has
since been archived".

What others do:

| System | What it does | Quote |
|---|---|---|
| GitHub repositories | Archive, read-only, and back | "You can archive a repository to make it read-only for all users and indicate that it's no longer actively maintained." "You can also unarchive repositories that have been archived." ([docs](https://docs.github.com/en/repositories/archiving-a-github-repository/archiving-repositories)) |
| Stripe prices | Archive what's in use; what uses it goes on | `active`: "Whether the price can be used for new purchases. Defaults to `true`." ([API](https://docs.stripe.com/api/prices/update)) |
| HashiCorp Vault KV v2 | Soft delete, then destroy | "The `kv` v2 plugin uses soft deletes to make data inaccessible while allowing data recovery." ([docs](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)) |
| Google Secret Manager | Disable before destroy | "Before destroying a secret version, try disabling it first and observe your application's behavior." ([docs](https://docs.cloud.google.com/secret-manager/docs/destroy-secret-version)) |
| AWS Secrets Manager | Delete after a window | "Until the recovery window ends, you can recover a secret you previously deleted." ([docs](https://docs.aws.amazon.com/secretsmanager/latest/userguide/manage_delete-secret.html)) |

Every one of them has a way back. GitHub states the rule an archive implies, the one to build:
"To make changes in an archived repository, you must unarchive the repository first."

## 12.2 Refuse, or let it run on

Stripe and GitHub let you archive something that's still in use. A subscription on an archived
price goes on billing; it just can't be bought again. The other design refuses: nothing that's in
use can be archived, and the refusal names what uses it.

| | Let it run on | Refuse while in use |
|---|---|---|
| What "archived" means | No *new* use | Nothing runs on it |
| What breaks | Nothing, today | The archive, until the users move |
| What you learn | Little: it's still running somewhere | Exactly who still depends on it |
| The cost | Archived things that still matter | An extra step before retiring |

Neither is wrong. Letting it run on suits a catalogue with outside customers, who can't be made
to move. Refusing suits a system whose users are all yours, where "archived" should mean "safe to
forget". Whichever you pick, say it in the word: an archived price that still bills is a different
promise from an archived API call that nothing calls.

## 12.3 What "in use" means

"In use" can't mean "every saved revision that ever pinned it". Revisions never change and are
kept for good ([06 §6.9](06-describing-apis.md#69-one-call-shared-versions-that-never-change-pinned)),
so by that count everything stays in use for ever. It has to mean what can run: each workflow's
**latest revision**, which an editor runs and a save builds on, and its **published revision**,
which its callers reach. That's the **live set**.

Live isn't only direct. A workflow can call another at a pinned revision, and that revision pins
its own API calls. If `orders` at revision 5 calls `billing` at revision 3, and only revision 3
uses `old-crm`, then archiving `old-crm` breaks `orders`, though nothing in `orders` names it, and
though `billing`'s own latest revision has moved on. So "in use" is **reachability** from the live
set, through every pin, however far in. And drafts may use each other, so the walk has to end on a
cycle.

Here's the whole rule, as I ran it with Go 1.24.4:

```go
package main

import "fmt"

// Ref names one saved thing at one number, which never changes: a workflow
// at a revision, or a saved API call at a version.
type Ref struct {
	Name string
	At   int
}

// pins is what each saved revision uses, exactly.
var pins = map[Ref][]Ref{
	{"orders", 6}:  {{"users-api", 2}},
	{"orders", 5}:  {{"billing", 3}}, // served, and pins an old billing
	{"billing", 4}: {{"new-crm", 1}}, // billing's latest and served
	{"billing", 3}: {{"old-crm", 1}}, // nothing else reaches old-crm
	{"loopy", 1}:   {{"loopy", 1}},   // a draft may use itself
	{"retired", 2}: {{"legacy", 1}},  // an old revision, not live
}

// live is what can run: each workflow's latest and served revisions.
var live = []Ref{{"orders", 6}, {"orders", 5}, {"billing", 4}, {"loopy", 1}}

// reaches says whether from uses target, however far in. seen is every
// revision already walked, so a cycle ends.
func reaches(from Ref, target string, seen map[Ref]bool) bool {
	if seen[from] {
		return false
	}
	seen[from] = true
	for _, p := range pins[from] {
		if p.Name == target || reaches(p, target, seen) {
			return true
		}
	}
	return false
}

// users is every live revision that reaches target.
func users(target string) []Ref {
	var out []Ref
	for _, r := range live {
		if r.Name != target && reaches(r, target, map[Ref]bool{}) {
			out = append(out, r)
		}
	}
	return out
}

func main() {
	for _, t := range []string{"users-api", "old-crm", "new-crm", "billing", "legacy"} {
		if us := users(t); len(us) > 0 {
			fmt.Printf("archive %-9s refused: %v\n", t, us)
		} else {
			fmt.Printf("archive %-9s free\n", t)
		}
	}
}
```

```
archive users-api refused: [{orders 6}]
archive old-crm   refused: [{orders 5}]
archive new-crm   refused: [{billing 4}]
archive billing   refused: [{orders 5}]
archive legacy    free
```

Read the second line slowly: `old-crm` is refused for `orders`, through a revision of `billing`
that `billing` itself no longer uses. The last line is the other half of the rule: `legacy` is
pinned by a saved revision, but not a live one, so it's free to go. `loopy` uses itself and the walk
still ends, because `seen` refuses to visit a revision twice.

The refusal names the root, `orders at revision 5`, not the path to `old-crm`. That's the cheap
answer, and it can puzzle: nothing in `orders` mentions `old-crm`. A path, `through billing at
revision 3`, costs one more column in the walk.

<details><summary>Rust</summary>

Standard library only, built with rustc 1.96 and `--edition 2024`. It prints the same five lines.

```rust
use std::collections::{HashMap, HashSet};

// Ref names one saved thing at one number, which never changes.
type Ref = (&'static str, u32);

// reaches says whether from uses target, however far in. seen is every
// revision already walked, so a cycle ends.
fn reaches(
    pins: &HashMap<Ref, Vec<Ref>>,
    from: Ref,
    target: &str,
    seen: &mut HashSet<Ref>,
) -> bool {
    if !seen.insert(from) {
        return false;
    }
    pins.get(&from).is_some_and(|ps| {
        ps.iter()
            .any(|&p| p.0 == target || reaches(pins, p, target, seen))
    })
}

fn main() {
    // What each saved revision uses, exactly.
    let pins: HashMap<Ref, Vec<Ref>> = HashMap::from([
        (("orders", 6), vec![("users-api", 2)]),
        (("orders", 5), vec![("billing", 3)]),
        (("billing", 4), vec![("new-crm", 1)]),
        (("billing", 3), vec![("old-crm", 1)]),
        (("loopy", 1), vec![("loopy", 1)]),
        (("retired", 2), vec![("legacy", 1)]),
    ]);
    // What can run: each workflow's latest and served revisions.
    let live: [Ref; 4] = [("orders", 6), ("orders", 5), ("billing", 4), ("loopy", 1)];

    for t in ["users-api", "old-crm", "new-crm", "billing", "legacy"] {
        let us: Vec<Ref> = live
            .iter()
            .copied()
            .filter(|&r| r.0 != t && reaches(&pins, r, t, &mut HashSet::new()))
            .collect();
        if us.is_empty() {
            println!("archive {t:<9} free");
        } else {
            let said: Vec<String> = us.iter().map(|(n, a)| format!("{{{n} {a}}}")).collect();
            println!("archive {t:<9} refused: [{}]", said.join(" "));
        }
    }
}
```

`HashSet::insert` returns `false` when the value was there already, which is the cycle check in
one call.

</details>

<details><summary>C++</summary>

C++20 and the standard library only, built with Apple clang 17 and
`-std=c++20 -Wall -Wextra -Wpedantic`, no warnings. It prints the same five lines.

```cpp
#include <iomanip>
#include <iostream>
#include <map>
#include <set>
#include <string>
#include <utility>
#include <vector>

// Ref names one saved thing at one number, which never changes.
using Ref = std::pair<std::string, int>;
using Pins = std::map<Ref, std::vector<Ref>>;

// reaches says whether from uses target, however far in. seen is every
// revision already walked, so a cycle ends.
bool reaches(const Pins& pins, const Ref& from, const std::string& target,
             std::set<Ref>& seen) {
  if (!seen.insert(from).second) return false;
  auto it = pins.find(from);
  if (it == pins.end()) return false;
  for (const auto& p : it->second)
    if (p.first == target || reaches(pins, p, target, seen)) return true;
  return false;
}

int main() {
  // What each saved revision uses, exactly.
  const Pins pins{
      {{"orders", 6}, {{"users-api", 2}}},
      {{"orders", 5}, {{"billing", 3}}},
      {{"billing", 4}, {{"new-crm", 1}}},
      {{"billing", 3}, {{"old-crm", 1}}},
      {{"loopy", 1}, {{"loopy", 1}}},
      {{"retired", 2}, {{"legacy", 1}}},
  };
  // What can run: each workflow's latest and served revisions.
  const std::vector<Ref> live{
      {"orders", 6}, {"orders", 5}, {"billing", 4}, {"loopy", 1}};

  for (const std::string t :
       {"users-api", "old-crm", "new-crm", "billing", "legacy"}) {
    std::string said;
    for (const auto& r : live) {
      std::set<Ref> seen;
      if (r.first != t && reaches(pins, r, t, seen)) {
        if (!said.empty()) said += " ";
        said += "{" + r.first + " " + std::to_string(r.second) + "}";
      }
    }
    std::cout << "archive " << std::left << std::setw(9) << t;
    if (said.empty())
      std::cout << " free\n";
    else
      std::cout << " refused: [" << said << "]\n";
  }
}
```

`std::set::insert` returns a pair whose `.second` is `false` for a value already there.

</details>

<details><summary>JavaScript</summary>

An ES module, run with Node 26.13.0, no packages. It prints the same five lines.

```js
// A ref is "name@number": one saved thing at one number, which never changes.

// What each saved revision uses, exactly.
const pins = {
  "orders@6": ["users-api@2"],
  "orders@5": ["billing@3"],
  "billing@4": ["new-crm@1"],
  "billing@3": ["old-crm@1"],
  "loopy@1": ["loopy@1"],
  "retired@2": ["legacy@1"],
};

// What can run: each workflow's latest and served revisions.
const live = ["orders@6", "orders@5", "billing@4", "loopy@1"];

const nameOf = (ref) => ref.slice(0, ref.indexOf("@"));

// reaches says whether from uses target, however far in. seen is every
// revision already walked, so a cycle ends.
function reaches(from, target, seen) {
  if (seen.has(from)) return false;
  seen.add(from);
  return (pins[from] ?? []).some(
    (p) => nameOf(p) === target || reaches(p, target, seen),
  );
}

for (const t of ["users-api", "old-crm", "new-crm", "billing", "legacy"]) {
  const users = live.filter(
    (r) => nameOf(r) !== t && reaches(r, t, new Set()),
  );
  console.log(
    `archive ${t.padEnd(9)} ` +
      (users.length > 0
        ? `refused: [${users.map((r) => `{${r.replace("@", " ")}}`).join(" ")}]`
        : "free"),
  );
}
```

Refs are strings here, as a `Map` keyed by an array compares arrays by identity, not by value.

</details>

### In SQL, one query

A database can do the same walk with a recursive query, so the check needs no round trip per
revision. The seed is the live set; each step joins a revision to the revisions its nodes pin.
PostgreSQL's `UNION`, unlike `UNION ALL`, is what ends it on a cycle: in each round it will
"discard duplicate rows and rows that duplicate any previous result row"
([WITH Queries](https://www.postgresql.org/docs/current/queries-with.html)). The same page warns
when that isn't enough: "often a cycle does not involve output rows that are completely
duplicate". A row that also carried a depth, or a path, would differ every round, and the query
would never end. Keep the walked row to what identifies a revision: the root it came from, and the
revision it reached.

## 12.4 Check, then act

Retiring is two steps: check that nothing live uses the thing, then mark it retired. Anything that
can make something live in between breaks the check. That's a time-of-check to time-of-use race,
in MITRE's words: "The product checks the state of a resource before using that resource, but the
resource's state can change between the check and the use in a way that invalidates the results of
the check" ([CWE-367](https://cwe.mitre.org/data/definitions/367.html)).

So list what makes something live, and ask of each whether it can slip in:

| What happens | Can it make an archived thing live? | Needs the lock? |
|---|---|---|
| A draft saved that uses it | Yes, but a draft may use archived things: it shows a problem, and can't run or publish | No |
| A publish of a revision that uses it | Yes, and it's then served | **Yes** |
| A restore of something that uses it | Yes, but restore is never refused: what it uses shows as a problem | No |
| A run that started before the archive | It runs on pinned versions, which stay | No |

Only the publish needs to be kept out. One lock that both an archive and a publish take does it:
whichever comes second waits, then sees what the first did. A row lock can't, as the two don't
touch the same row. PostgreSQL's **advisory locks** are for exactly this: "locks that have
application-defined meanings. These are called advisory locks, because the system does not enforce
their use — it is up to the application to use them correctly"
([Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)). Take the
transaction-level kind, `pg_advisory_xact_lock(n)`, as those "are automatically released at the
end of the transaction, and there is no explicit unlock operation": a return path that forgets to
unlock can't leave it held.

A publish holds it across its whole check: read the revision, read everything it pins, decide
whether it has problems, then write. That's what makes it safe, and it's what causes §12.5.

## 12.5 A lock, a pool, and a jam

A connection pool hands out a fixed number of connections. pgx's, for one: "MaxConns is the
maximum size of the pool. The default is the greater of 4 or runtime.NumCPU()"
([pgxpool](https://pkg.go.dev/github.com/jackc/pgx/v5/pgxpool)). A transaction holds its connection
from `BEGIN` to the end, *including while it waits for a lock*.

Now let the publish's check read what the revision pins through the pool, on a *second*
connection, while its transaction holds the lock on the first. One publish is fine. Four at once,
on a pool of four: each takes a connection and waits for the lock; the one that gets it asks the
pool for a second connection; there's none, because the other three hold one each, waiting for the
lock it holds. Nobody moves until something times out.

That needs every connection taken before the holder asks for its second, which a `BEGIN` round
trip makes all but certain under load. Here's the measurement, a pool of four simulated with a
channel, the lock with another, a 200 ms deadline, as I ran it with Go 1.24.4:

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// pool is a connection pool of cap(p) connections.
type pool chan struct{}

func (p pool) acquire(ctx context.Context) error {
	select {
	case p <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (p pool) release() { <-p }

// publish takes a connection, begins, and takes the one lock, as a
// transaction would, holding the connection while it waits. With second, it
// then asks the pool for another connection to read what it needs.
func publish(ctx context.Context, p pool, lock chan struct{}, second bool) error {
	if err := p.acquire(ctx); err != nil {
		return err
	}
	defer p.release()
	time.Sleep(time.Millisecond) // BEGIN, a round trip
	select {
	case lock <- struct{}{}:
		defer func() { <-lock }()
	case <-ctx.Done():
		return ctx.Err()
	}
	if second {
		if err := p.acquire(ctx); err != nil {
			return err
		}
		defer p.release()
	}
	time.Sleep(time.Millisecond) // the reads and the write
	return nil
}

func run(publishes int, second bool) (done int64, took time.Duration) {
	ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
	defer cancel()
	p, lock := make(pool, 4), make(chan struct{}, 1)
	start := time.Now()
	var wg sync.WaitGroup
	for range publishes {
		wg.Add(1)
		go func() {
			defer wg.Done()
			if publish(ctx, p, lock, second) == nil {
				atomic.AddInt64(&done, 1)
			}
		}()
	}
	wg.Wait()
	return done, time.Since(start).Round(time.Millisecond)
}

func main() {
	for _, n := range []int{3, 4, 8} {
		for _, second := range []bool{true, false} {
			how := "one connection"
			if second {
				how = "a second connection"
			}
			done, took := run(n, second)
			fmt.Printf("%d at once, %-20s %d published, in %v\n", n, how, done, took)
		}
	}
}
```

```
3 at once, a second connection  3 published, in 5ms
3 at once, one connection       3 published, in 5ms
4 at once, a second connection  0 published, in 200ms
4 at once, one connection       4 published, in 6ms
8 at once, a second connection  0 published, in 201ms
8 at once, one connection       8 published, in 10ms
```

Three runs gave the same counts, with times within a few milliseconds. At three, there's always a
spare connection, so nothing jams. At four, or eight, every publish gives up at the deadline: not
slow, *none*. Without the `BEGIN` sleep, my first version never jammed at all, as each goroutine
finished before the next started. A test that doesn't model the round trip passes, and the server
jams in production.

Two fixes:

- **Read on the same connection.** Pass the transaction to whatever reads what the revision pins,
  so the check never asks the pool for anything while it holds the lock. That's the one I'd pick:
  the read is then also inside the transaction's snapshot.
- **Don't hold a connection while waiting.** Take a lock outside the database first, or use
  `pg_try_advisory_xact_lock`, which doesn't wait, and retry. That works, but a retry loop
  is a second mechanism to get right.

A statement timeout ends the jam in a real database: each waiter's `pg_advisory_xact_lock` fails
once its timeout passes, freeing its connection. The publishes still fail, so the timeout makes it
recoverable, not correct.

## 12.6 Names stay taken

If an archived workflow's name is free, someone makes a new workflow with it, and every caller
still calling the old URL reaches the new one: different input, different answers, possibly
different data. GitHub hits the same thing with renames, where the old name redirects to the new:
"If you create a new repository under your account in the future, do not reuse the original name
of the renamed repository. If you do, redirects to the renamed repository will no longer work"
([Renaming a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/renaming-a-repository)).

So keep the unique constraint as it is, archived rows included, and make the refusal say why: not
"that name is taken", which sends the user hunting through a list that doesn't show it, but "an
archived workflow has it: restore it, or pick another".

## 12.7 Disable before you destroy

Secrets show the split most clearly. Retiring a secret could mean two things:

- **Disable**: nothing can read it, the value stays, and it can come back.
- **Destroy**: the value is gone for good. Vault's "permanently deleted" entry: "Vault purges the
  underlying version data and marks the key metadata as destroyed"
  ([KV v2](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)). Google's: "Destroying a
  secret version is a permanent action. After a version is destroyed, you can't access the secret
  data or restore the version to another state"
  ([Destroy a secret version](https://docs.cloud.google.com/secret-manager/docs/destroy-secret-version)).

Every provider here puts disable first, and AWS makes it mandatory: a deleted secret is
"inaccessible and scheduled for deletion after a recovery window of a minimum of seven days"
([Delete a secret](https://docs.aws.amazon.com/secretsmanager/latest/userguide/manage_delete-secret.html)).

An archived secret that's kept sealed has a use beyond Restore. An engine that masks secrets in what
runs keep ([08 §8.9](08-recording-a-run-and-following-it-live.md#89-what-a-run-keeps-of-a-secret)) masks by value: it replaces every
copy of the secret it finds. If archiving dropped the value, a run that still met it, echoed back
by an API, say, would keep it in the clear. So the masker reads every secret, archived or not.

The tension is the law, not the code. GDPR's storage limitation says personal data is to be
"kept in a form which permits identification of data subjects for no longer than is necessary"
([Art. 5(1)(e)](https://gdpr-info.eu/art-5-gdpr/)). A secret that's personal data and kept sealed
for ever doesn't fit. Destroying has to exist too, as its own action: drop the value, keep the
record that there was one, its name, and when it went.

## 12.8 Read-only is the server's job

An archived workflow should still open, to read: its steps, its revisions, its runs. The screen
has to stop every change, and it's tempting to do that only in the UI. Don't. The server refuses
every change to something archived, a save, a publish, a run, a new key, with one answer, and the
UI is the courtesy on top. A future agent, a script, or a stale tab never sees the UI.

The UI side has one tidy trick. A disabled `<fieldset>` "causes all the form control descendants
of the fieldset element" to be disabled
([HTML](https://html.spec.whatwg.org/multipage/form-elements.html#the-fieldset-element)): one
element instead of a `disabled` on every input. Two snags turned up in practice:

- **Tabs are buttons.** A panel's tabs, inside the fieldset, were disabled too, so nothing on a
  second tab could be read. The fix was to render each tab as a `div` with the tab role, which a
  fieldset doesn't reach. Base UI's tab takes `nativeButton={false}` for that: "Set to `false` if
  the rendered element is not a button (for example, `<div>`)"
  ([Tabs](https://base-ui.com/react/components/tabs)).
- **Not every control is a form control.** Base UI's checkbox is a `span` with a hidden input. The
  fieldset disables the input, so a click does nothing, but the span still takes focus and tells a
  screen reader nothing. It has to be told it's disabled itself.

And one about testing it. In jsdom, a test's `click()` on a button disabled by its fieldset still
reached the click handler, as a script-dispatched event; in Chromium, Playwright's forced click on
the same button did nothing. A unit test can check that a control *is* disabled (`:disabled`), but
only a real browser shows that it stays shut.

## 12.9 Keeping what happened

Restore raises the last question: when something is archived and then restored, is the archive
gone from the record? If the archive was a column, `archived_at`, then Restore clears it, and yes.
An audit trail needs the events themselves: each archive, each restore, each change of a value,
with when and by what, appended and never rewritten.

Two choices shape such a list.

- **Who it's for, worked out once.** When a value changes, the engine can work out then which
  workflows read it, and put the change on each one's list. Or it can work that out every time a
  list is read. The first keeps history true to what happened: a workflow that stops reading the
  value later still shows it was told. The second rewrites the past each time.
- **Active, worked out, not stored.** "A newer version waits on you" is true while a step pins the
  older one. Storing an `active` flag means something has to clear it, at the right moment, on a
  clock for expiries. Working it out as the list is read, from how things stand, needs nothing to
  run, and can't forget. The cost: undo the fix, move the step back, and the item is active
  again, which is honest but can surprise.

How long others keep such events is a reminder that "for good" is a choice: GitHub keeps
notifications marked Done "for 3 months"
([docs](https://docs.github.com/en/subscriptions-and-notifications/how-tos/viewing-and-triaging-notifications/managing-notifications-from-your-inbox)),
Stripe's events can be retrieved "for 30 days" ([API](https://docs.stripe.com/api/events)), and
Kubernetes keeps its own for an hour, `--event-ttl` defaulting to `1h0m0s`
([kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)).
Those are notifications and operational events, cheap to lose. An audit trail is the opposite
case.

## Check yourself

1. Add a live revision `{"reports", 1}` that pins `{"orders", 5}` to the example in §12.3. What does
   archiving `old-crm` report now, and why both lines? Then archive `orders` instead: which live
   revisions refuse it?
2. Why is the live set each workflow's latest and published revisions, and not every saved
   revision? What would change, for what can ever be archived, if it were every revision?
3. Write out, step by step, the interleaving of an archive of `old-crm` and a publish of `orders`
   at revision 5 that ends with a served revision using an archived call. Which single change
   prevents it, and why don't draft saves need it?
4. In §12.5, with a pool of 10, how many publishes at once jam, if each takes a second connection
   while holding the lock? Why did three never jam? Why did my first version, without the `BEGIN`
   sleep, never jam at all?
5. A workflow named `quotes` is archived and its URL `/serve/quotes` stops answering. A teammate
   wants to make a new `quotes`. What could go wrong for an old caller, and what should the
   refusal say?
6. Why does an engine that masks secrets in what runs keep need an archived secret's value? What
   would destroying a value have to keep, and what would a run that met it afterwards keep?
7. An item "a newer version waits on you" is worked out from whether the latest revision pins an
   older version. Someone takes up the new version, saves, then moves back. What does the list
   show, and what would a stored `active` flag have shown instead?

(Answers in [14-exercises.md](14-exercises.md).)
