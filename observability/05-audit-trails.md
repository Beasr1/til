# 5. Audit trails

## The problem

A compliance review asks for the record of who signed which documents last quarter. The audit
table stops months ago. Around then the API was rewritten; the only
code that wrote audit records lived inside the old handlers, and the new handlers never called
it. Publishing was "best effort": a failure, or in this case an absence, logged a warning at most.
Nobody read warnings, and nobody noticed an event that stopped happening. Months of trail are
gone and can't be rebuilt.

An audit trail looks like logging, so it gets built like logging: sprinkled through handlers,
allowed to fail quietly, filtered when it's inconvenient. It isn't logging. This chapter is about
what makes it different, and the four habits that stop it vanishing.

## Audit is not telemetry

Telemetry answers "how did the system behave?" for the engineers running it. An **audit trail**
answers "who did what, to which thing, when, and was it allowed?" for someone who wasn't there:
a compliance officer, an investigator, a customer disputing an action. The two differ in nearly
every property that matters:

| | Telemetry (traces, logs, metrics) | Audit trail |
|---|---|---|
| Question | How did the system behave? | Who did what, and when? |
| Reader | Engineers, during an incident | Auditors, investigators, customers, sometimes courts |
| Completeness | Sampling and loss are acceptable | A missing record is a defect |
| Volume | High; every step, retry and state change | Low; a few business events per transaction |
| Retention | Days to weeks, chosen by cost | Set by policy or regulation, often years |
| Mutability | Rotated and deleted freely | Append-only, tamper-evident |
| Failure to write | Drop it and move on | Must be noticed, and possibly must block the action |

The security guidance draws the same line. The [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
lists "Audit trails e.g. data addition, modification and deletion, data exports" among the
security uses of logging, separate from "General debugging" and "Performance monitoring", and says
that "Process monitoring, audit, and transaction logs/trails etc. are usually collected for
different purposes than security event logging" and "they should be kept separate."
[OWASP ASVS 5.0 V16](https://github.com/OWASP/ASVS/blob/master/5.0/en/0x25-V16-Security-Logging-and-Error-Handling.md)
opens with "Security logs are distinct from error or performance logs". For organising log
management as a whole, NIST's [SP 800-92, *Guide to Computer Security Log Management*](https://csrc.nist.gov/pubs/sp/800/92/final)
(September 2006) is the long-standing reference, though it predates cloud-native tooling.

The practical consequence: **send audit events to their own store, and keep them few.**

## Keep it to a few events

The temptation, once there's an audit topic, is to send it everything: every state transition,
every retry, every SDK callback. That turns the trail into a second, worse telemetry system, and
buries the five records an auditor needs under five thousand they don't.

Pick the events that answer "who did what" for your domain, usually the lifecycle of the things
people act on: created, approved, signed, downloaded, exported, deleted, and permission changes.
State machines, retries and timings stay in telemetry, which is built for them. Link the two by
putting the trace id on each audit record, so an investigator who needs the "how" can follow it
into the traces (for as long as telemetry retention allows).

Each record answers OWASP's "when, where, who and what" (ASVS 16.2.1 asks for the same):

| Field | Example | Note |
|---|---|---|
| When | `2026-03-04T10:15:02Z` | UTC or an explicit offset (ASVS 16.2.2) |
| Who | Actor id, actor type (user, service account) | A service call without a user is a different actor type, not a blank |
| What | `document.signed` | A fixed vocabulary, not free text |
| Which | Object id | Your id, not a personal identifier (chapter 4) |
| Outcome | `success`, `denied`, `failed` | Denials matter as much as successes |
| From where | Client address, user agent, as forwarded by your proxy | Read the forwarded header your proxy sets, not the proxy's own address |
| Trace | Trace id | The bridge to telemetry |

## Lift it out of the handlers

A duty that lives inside handlers lives only as long as those handlers. Rewrite the routes, move
to a new framework, delete an old API version, and the audit writes go with them, along with any
other cross-cutting duty that was inlined there (idempotency checks are a common companion). The
code review for the rewrite sees new handlers that work; it can't see a call that isn't there.

Put audit in one module, invoked from shared middleware for request-driven events, with a table
mapping each route to its audit action. Then make the table enforceable: a test that walks every
registered route that changes state and fails if it has no audit mapping, or an explicit "not
audited" entry with a reason. A new route can't silently skip the trail, because the build breaks.

Some events aren't known at request time: a signing outcome that arrives from an asynchronous
worker, for example. Emit those where the outcome becomes known, but through the same module, so
there is still one place to read to see every audit event the system produces.

## Redact by allowlist; don't drop the action

A common first version stores the whole request (`params`, `body`, `query`) with each record. Then
someone notices that the body of "create user" contains a password, and the fix is to stop auditing
"create user". Now the most security-relevant actions, creating accounts and changing credentials,
are exactly the ones missing from the trail.

The fix is the same allowlist idea as chapter 4: each audit action declares the fields it records.
"User created" records the new user's id, role and the creating actor. The password was never on
the list, so there's nothing to strip. A field added to the request body later isn't recorded until
someone adds it to the list on purpose.

```go
type Action struct {
    Name   string
    Fields []string // allowlist of request fields to record
}

var auditActions = map[string]Action{ // keyed by route pattern
    "POST /users":            {Name: "user.created", Fields: []string{"role", "org_id"}},
    "PUT /users/{id}/role":   {Name: "user.role_changed", Fields: []string{"role"}},
    "GET /documents/{id}/pdf": {Name: "document.downloaded"},
}

func Audit(pub *Publisher, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
        next.ServeHTTP(rec, r)

        action, ok := auditActions[r.Pattern]
        if !ok {
            return
        }
        ev := Event{
            Time:    time.Now().UTC(),
            Actor:   actorFrom(r.Context()),
            Action:  action.Name,
            Object:  r.PathValue("id"),
            Outcome: outcome(rec.status),
            Fields:  pick(r.Context(), action.Fields), // fields captured by the decoding layer
            TraceID: trace.SpanContextFromContext(r.Context()).TraceID().String(),
        }
        pub.Enqueue(r.Context(), ev) // never blocks the response; see below
    })
}
```

Two cautions on this sketch. `r.Pattern` here relies on the same request object passing through the
mux (chapter 4, "Metric labels"). And reading the body in middleware after the handler consumed it
doesn't work, so the handler's decoding layer has to stash the allowlisted fields in the context.

<details><summary>Rust</summary>

```rust
use axum::{extract::{MatchedPath, Request}, middleware::Next, response::Response};

async fn audit(state: AppState, req: Request, next: Next) -> Response {
    let route = req.extensions().get::<MatchedPath>().map(|p| p.as_str().to_owned());
    let actor = actor_from(&req);
    let res = next.run(req).await;
    if let Some(action) = route.as_deref().and_then(|r| state.audit_actions.get(r)) {
        let ev = Event::new(actor, action.name, outcome(res.status()));
        if state.audit_tx.try_send(ev).is_err() {
            tracing::error!(action = action.name, "audit event dropped: queue full");
            state.audit_dropped.increment(1);
        }
    }
    res
}
```

</details>

<details><summary>C++</summary>

```cpp
void AuditMiddleware(const Request& req, Response& res, const Next& next) {
  next(req, res);
  auto it = kAuditActions.find(req.route_pattern);
  if (it == kAuditActions.end()) return;
  Event ev{NowUtc(), ActorFrom(req), it->second.name, Outcome(res.status)};
  if (!audit_queue.try_push(std::move(ev))) {  // bounded queue, never blocks
    LOG_ERROR("audit event dropped: queue full action=" + it->second.name);
    audit_dropped.Increment();
  }
}
```

</details>

<details><summary>JavaScript</summary>

In Fastify, an `onResponse` hook runs after the response has been sent; `onSend` runs before, so
awaiting a database write there delays every response.

```js
fastify.addHook('onResponse', async (request, reply) => {
  const action = auditActions[request.routeOptions.url];
  if (!action) return;
  publisher.enqueue({
    time: new Date().toISOString(),
    actor: actorFrom(request),
    action: action.name,
    outcome: outcome(reply.statusCode),
    fields: pick(request.body, action.fields),
  });
});
```

</details>

## Write it off the response path, and say what that costs

Writing the audit record synchronously before the response goes out adds a database round trip to
every audited request, and makes the user wait on a write that isn't theirs. The usual move is to
enqueue and return, with a background publisher draining the queue. The OWASP cheat sheet supports
the instinct: "Ensure failures in the logging processes/systems do not prevent the application from
otherwise running."

But be honest about the trade. An in-memory queue loses whatever is in it when the process dies,
and a full queue drops events. For telemetry that's fine. For an audit trail, it means "best effort",
and whether best effort is acceptable is a policy decision, not an engineering default. Where it
isn't, the standard technique is an **outbox**: write the audit record in the same database
transaction as the action it describes, and publish from that table asynchronously. The action and
its record then commit or fail together, and the publish can be retried. There's no free version;
you're choosing between latency, loss and complexity, and the choice should be written down.

## Make silence loud

The gap at the start of this chapter wasn't caused by a failure. It was caused by an
absence, and absences don't produce errors. Two habits catch both:

1. **Log publish failures at ERROR, with a counter.** A dropped audit event is a defect, not a
   warning. The counter lets you alert on it; the log line tells you which event.
2. **Alert on silence.** For each audit action, you know roughly how often it should happen. An
   alert that fires when `document.signed` events are zero for several hours while signing traffic
   is non-zero catches the deleted call, the misrouted topic and the consumer that stopped. OWASP
   asks for the same thing in general form: "Enable processes to detect whether logging has
   stopped".

And protect what you keep. OWASP recommends tamper detection, read-only storage "as soon as
possible", and that "All access to the logs must be recorded and monitored"; ASVS 16.4.2 and 16.4.3
ask that logs can't be modified and are sent "to a logically separate system". An audit trail the
audited system can edit isn't evidence of much.

> **Teacher's aside.** The dangerous property of a best-effort side effect isn't that it can fail.
> It's that its failure looks exactly like success: the request returns 200, the user is happy, the
> tests pass. Anything you'd be upset to discover missing months later needs a signal that fires on
> its absence, not just on its errors. That's true of audit trails, billing events, outgoing
> notifications and backups alike.

## Check yourself

1. A colleague proposes sending every state transition of a verification session to the audit
   topic, "so we have everything". Give two concrete costs, and say where those transitions belong.
2. The audit middleware stores the full request body. To stop storing passwords, the team removes
   the audit mapping for "create user" and "change password". What's now true of the trail, and what
   should they have done?
3. During a route rewrite, all handlers are replaced and the test suite passes. What test would
   have failed if the new routes had no audit mapping, and what does it check?
4. Audit events are enqueued in memory and published by a background goroutine. List the ways an
   event can be lost, and describe a design in which the action and its audit record cannot diverge.
5. Publishing failures are logged at WARN, and the trail is empty for months without anyone
   noticing. Which of the two "make silence loud" habits would have caught it, and why wouldn't the
   other one?
