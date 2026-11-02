# 8. Recording a run, and following it live

Assumes [file 01](01-foundations.md) for the vocabulary, the run record above all, and
[file 02](02-graphs-and-scheduling.md) for the scheduler: one goroutine owns a run's state,
and step goroutines only report to it. §8.4 leans on
[file 04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash)
for how much of a run survives a crash, and §8.6 on
[go-services file 03](../go-services/03-the-http-server.md) for what a Go server's timeouts
bound.

## 8.1 The problem

A run has started. Four parties want something from it while it goes:

- The person who clicked **Run** in the editor wants to watch `user`, then `orders` and
  `prefs`, then `crm` turn green, not stare at a spinner.
- Whoever is on call tomorrow wants to read what happened: which step failed, with what
  status, after how long, having sent what.
- The engine itself, if it dies half-way, needs to know what had already happened
  ([file 04](04-retries-idempotency-and-durability.md)).
- In a request-style run ([file 01 §1.4](01-foundations.md#14-job-style-and-request-style)),
  the caller wants an answer that says whose fault a failure was.

Each has an obvious implementation, and each obvious implementation breaks something:

| Wanted | The obvious way | What breaks | Section |
|---|---|---|---|
| A record | Write the run record when the run ends | A crash loses the whole run, and a run in progress shows nothing | §8.2 |
| A record that costs the run nothing | Save each step's result before carrying on | The run now waits for every save: the scheduler inherits the database's latency | §8.2 |
| A record you can debug from | Store every request and response whole | It stores bearer tokens, cookies and 30 MB bodies | §8.3 |
| A record you can trust after a crash | Mark interrupted runs failed at start-up | Correct with one process, wrong the moment two overlap | §8.4 |
| A live view | Stream events to the browser | A proxy holds them until the end, a timeout cuts them, a slow browser holds memory or the run | §8.5–8.7 |
| An answer for the caller | Return the upstream's error | The caller can't tell its own mistake from an upstream's | §8.8 |

The chapter takes them in that order, with file 01's running example: fetch a user, then their
orders and preferences in parallel, then post a summary to a CRM.

## 8.2 Record each step as it ends

Write each step's result down when the step resolves, not once when the run does. Two things
come of it. A crash loses only what was in flight, not the steps that had finished. At level 2
of [file 04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash)
that write is the checkpoint a resumed run relies on; at level 1, where nothing resumes, it's
still what tells someone which steps had finished, and which call was in flight, when the
process died. And a run that's still going
has a record that shows its progress, which is where a live view (§8.5) starts from.

The cost is a write per step, and where that write happens decides whether it costs the run
anything. n8n's workflow settings put the trade in one line: "Save execution progress … If set
to Save, the workflow resumes from where it stopped in case of an error. This may increase
latency" ([n8n: workflow settings](https://docs.n8n.io/workflows/settings/)).

It increases latency when the scheduler waits for the write. File 02's scheduler is one
goroutine that receives a step's report, resolves it and starts the children it unblocks. Put
a save inside that loop and the children start only once the database has answered. **A
scheduler that waits on a database inherits the database's latency**, once for every step on
the critical path.

I ran the running example with file 01 §1.5's invented durations (120, 300, 80 and 150 ms)
under a scheduler goroutine shaped like file 02's, against a stand-in database whose every
write takes 100 ms. First the scheduler saved each record itself, then it handed them to a
writer goroutine:

```
inside the scheduler
  user   finished at  121 ms
    saved user   at  223 ms
  prefs  finished at  305 ms
    saved prefs  at  407 ms
  orders finished at  525 ms
    saved orders at  628 ms
  crm    finished at  780 ms
    saved crm    at  882 ms
  the run finished at 882 ms
    saved run    at  983 ms
  everything saved by 983 ms

on a writer goroutine
  user   finished at  122 ms
  prefs  finished at  204 ms
    saved user   at  224 ms
    saved prefs  at  326 ms
  orders finished at  424 ms
    saved orders at  525 ms
  crm    finished at  576 ms
  the run finished at 576 ms
    saved crm    at  677 ms
    saved run    at  778 ms
  everything saved by 778 ms
```

Inside the loop the run took 882 ms instead of 576. Three saves sat on the critical path:
`user`'s, before `orders` and `prefs` could start; `orders`'s, before `crm` could; `crm`'s,
before the run could end. `prefs`'s save cost nothing, because `orders` was still running
while it happened. On the writer goroutine the run finished at 576 ms, as if the database
weren't there, and the record caught up 200 ms later.

```mermaid
flowchart LR
    S["scheduler goroutine<br/>(file 02)"] -->|"Record: never blocks"| C[["channel<br/>room for n + 1 records"]]
    C --> W["writer goroutine<br/>one per run"]
    W -->|"however slow"| DB[("database")]
    S -->|"append, under a short lock"| L["the run's event log<br/>in memory (§8.7)"]
    L --> B["browsers following<br/>the run (§8.5)"]
    style C fill:#264653,stroke:#8ecae6,color:#fff
```

The writer is a goroutine per run, fed through a channel. This is the code that produced the
second timeline; the harness around it is left out:

```go
// record is one row of the run record: a step's final status, or the run's.
type record struct {
	Step   string // "" for the run itself
	Status string
}

// recorder saves one run's records, in the order they were made, on its
// own goroutine, so the scheduler never waits for the database.
type recorder struct {
	records chan record
	done    chan struct{}
}

// startRecorder's channel has room for every record the run can make: one
// per step, since each step resolves exactly once, and one for the run. So
// Record never blocks, however slow save is.
func startRecorder(steps int, save func(record) error) *recorder {
	r := &recorder{records: make(chan record, steps+1), done: make(chan struct{})}
	go func() {
		defer close(r.done)
		for rec := range r.records {
			if err := save(rec); err != nil {
				log.Printf("recording %q: %v", rec.Step, err) // the run carries on
			}
		}
	}()
	return r
}

func (r *recorder) Record(rec record) { r.records <- rec }

// Close waits until every record sent has been saved.
func (r *recorder) Close() { close(r.records); <-r.done }
```

Three decisions in it carry the weight:

- **The channel has room for every record the run will make.** Each step resolves exactly
  once, into one of five final states ([file 02 §2.3](02-graphs-and-scheduling.md#23-the-scheduler-loop)),
  and the run ends once, so n steps make n + 1 records. A send blocks only when the buffer is
  full ([go-services file 01](../go-services/01-how-a-go-service-runs.md#channels-block-and-the-size-decides-when)),
  and this one can't fill. It's file 02 §2.7's rule again: take the capacity from the same
  slice as the steps, so the two can't drift. Record anything more, such as a row per attempt
  (file 04 §4.5) or a `running` row as each step starts, and you have to count that too, or
  the guarantee is gone.
- **One writer per run keeps the records in order.** A goroutine per save would race: a slow
  write of `running` could land after the write of `succeeded`, and the record would say the
  step is still going. One goroutine reading one channel saves them in the order the scheduler
  made them.
- **A failed save is logged, not returned.** The run's outcome doesn't depend on its record.
  What the record should then say about itself is a design choice; flagging the run's record
  as incomplete beats a gap nobody notices.

> ⚠️ **The channel is a window of records not yet saved, and a crash loses it.** At level 1
> of file 04 §4.6 that costs little: the run is lost anyway, and the record lags by a few
> writes at most. At level 2, a resumed run reads the record to decide what to skip, so a step
> whose result was still in the channel runs again. A level-2 engine either waits for a step's
> checkpoint before starting its children, which puts the latency back, or accepts the extra
> repeats and leans harder on idempotency keys
> ([file 04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key)),
> which it needs for the step in flight anyway.

## 8.3 What to keep, and what to leave out

People read a run record to debug: what did `crm` send, and what came back? The tempting
answer is to store each request and response whole. Two things in them shouldn't be kept as
they arrive: secrets, and anything big.

### Secrets, by name

Some headers exist to carry credentials. RFC 9110 defines `Authorization`'s value as
"credentials containing the authentication information of the user agent"
([§11.6.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-11.6.2)), and
`Proxy-Authorization`'s the same way for a proxy
([§11.7.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-11.7.2)). `Cookie` and
`Set-Cookie` carry session state ([RFC 6265 §4.2 and §4.1](https://www.rfc-editor.org/rfc/rfc6265.html#section-4)),
which is a credential in all but name. OWASP's Logging Cheat Sheet lists what "should usually
not be recorded directly in the logs", including "Session identification values", "Access
tokens", "Authentication passwords" and "Encryption keys and other primary secrets"
([OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)). A run
record is a log by another name.

So the record **redacts** by name: those four headers, and any header or query parameter whose
name contains `key`, `token`, `secret` or `password`. Query parameters matter because an API
that can't take a header takes `?api_key=` or `?access_token=`, and the URL is the first thing
anyone records.

```go
const redacted = "REDACTED"

// sensitive reports whether a header or query parameter with this name may
// carry a secret: the credential headers, and any name that looks like one.
func sensitive(name string) bool {
	n := strings.ToLower(name)
	switch n {
	case "authorization", "proxy-authorization", "cookie", "set-cookie":
		return true
	}
	return strings.Contains(n, "key") || strings.Contains(n, "token") ||
		strings.Contains(n, "secret") || strings.Contains(n, "password")
}

// redactHeaders copies h with sensitive values replaced. For the two
// Authorization headers it keeps the scheme, such as "Bearer": that isn't
// secret, and it's often the answer to a 401.
func redactHeaders(h http.Header) http.Header {
	out := make(http.Header, len(h))
	for name, values := range h {
		if !sensitive(name) {
			out[name] = slices.Clone(values)
			continue
		}
		auth := strings.HasSuffix(strings.ToLower(name), "authorization")
		for _, v := range values {
			if scheme, _, ok := strings.Cut(v, " "); auth && ok {
				v = scheme + " " + redacted
			} else {
				v = redacted
			}
			out[name] = append(out[name], v)
		}
	}
	return out
}

// redactURL replaces the values of sensitive query parameters, keeping
// their names and order, and the password in a user:password@ part.
func redactURL(raw string) string {
	u, err := url.Parse(raw)
	if err != nil {
		return redacted // can't tell where a secret would be
	}
	params := strings.Split(u.RawQuery, "&")
	for i, p := range params {
		name, _, hasValue := strings.Cut(p, "=")
		n, err := url.QueryUnescape(name)
		if err != nil {
			n = name
		}
		if hasValue && sensitive(n) {
			params[i] = name + "=" + redacted
		}
	}
	u.RawQuery = strings.Join(params, "&")
	return u.Redacted() // the standard library's own redaction: the password only
}
```

### Size, without breaking the body

Bodies can be large, and products that store run data cap them. AWS Step Functions allows
"256 KiB of data as a UTF-8 encoded string" as any task's, state's or execution's input or
output, one of the quotas it lists as "hard quotas that cannot be changed"
([quotas](https://docs.aws.amazon.com/step-functions/latest/dg/service-quotas.html)), and its
advice for more is to "use Amazon Simple Storage Service (Amazon S3) to store the data" and pass
a reference to it instead ([best practices](https://docs.aws.amazon.com/step-functions/latest/dg/sfn-best-practices.html)).
Temporal's limit on a single payload "is 2 MB on Temporal Cloud, but is configurable on
self-hosted deployments with a default of 2 MB", and its fix for anything bigger "is called the
claim check pattern" ([Temporal: payload size limits](https://docs.temporal.io/troubleshooting/blob-size-limit-error)).
[File 04 §4.7](04-retries-idempotency-and-durability.md#47-large-payloads-and-the-claim-check)
covers the claim check and the lifecycle problem it brings.

A run record has a cheaper option, because it needn't keep a big body to be useful: keep its
start, say how big it was, and say it was cut. That's a different limit from the one on what a
step may receive ([file 05 §5.7](05-calling-urls-you-dont-control.md#57-the-rest-of-the-client)).
A step may need all 5 MB of a response for its mapping while the record keeps 64 KiB of it.

"Keep the start" has two traps in it:

- **JSON cut short isn't JSON.** Store the first 64 KiB of a JSON body in a `jsonb` column and
  the insert fails; return it as JSON from your API and the client's parser fails. So a capped
  body becomes a string preview, with a flag that says it was cut.
- **UTF-8 cut inside a character isn't UTF-8.** A character outside ASCII takes two to four
  bytes, and every byte after the first has the high bits `10`
  ([RFC 3629 §3](https://www.rfc-editor.org/rfc/rfc3629.html#section-3)). Cut between them and
  a stray byte is left at the end. I cut `café` after its fourth byte, between the two bytes of
  `é`. Go's `encoding/json`, whose strings are "coerced to valid UTF-8, replacing invalid bytes
  with the Unicode replacement rune", turned it into `"caf�"` without a word. Postgres 17.11
  refused to make text of it: `invalid byte sequence for encoding "UTF8": 0xc3`. So the cut
  moves back to the start of a character, which `utf8.RuneStart` recognises: "Second and
  subsequent bytes always have the top two bits set to 10."

```go
// Body is what the run record keeps of a response body.
type Body struct {
	JSON      json.RawMessage `json:"json,omitempty"`      // the whole body, if it's JSON and fits
	Preview   string          `json:"preview,omitempty"`   // otherwise its start, if it's text
	Truncated bool            `json:"truncated,omitempty"` // Preview isn't all of it
	Size      int             `json:"size"`                // bytes received
}

// keepBody keeps b whole if it's JSON within limit. Otherwise it keeps a
// text preview of at most limit bytes. JSON cut short isn't JSON, so the
// preview is a string, and it's cut where a character starts: UTF-8 cut
// inside a character isn't UTF-8.
func keepBody(b []byte, limit int) Body {
	if len(b) <= limit && json.Valid(b) {
		return Body{JSON: b, Size: len(b)}
	}
	cut := min(len(b), limit)
	for cut > 0 && cut < len(b) && !utf8.RuneStart(b[cut]) {
		cut-- // b[cut] continues a character: cut before that character
	}
	if !utf8.Valid(b[:cut]) {
		cut = 0 // not text: keep only the size
	}
	return Body{Preview: string(b[:cut]), Truncated: cut < len(b), Size: len(b)}
}
```

I ran the four functions with Go 1.27.2 over six headers, three URLs and four bodies, with the
limit set to 64 bytes so that cutting shows. The bodies are a small JSON object; a JSON object
whose 64th byte falls inside an `é`; RFC 6749 §5.1's example token response; and the first 16
bytes of a PNG. The driver is left out:

```
headers
  Authorization: Bearer REDACTED
  Content-Type: application/json
  Cookie: REDACTED
  Idempotency-Key: REDACTED
  X-Api-Key: REDACTED
  X-Request-Id: r-81f2
urls
  https://api.example.com/v1/orders?user=u-42&api_key=REDACTED&access_token=REDACTED&page=2
  https://svc:xxxxx@internal.example.com/v1/sync
  https://hooks.slack.com/services/<team>/<bot>/<secret>
bodies, limit 64 bytes
  naive cut: valid UTF-8 false, valid JSON false
  {"json":{"id":"u-42","name":"Ada"},"size":26}
  {"preview":"{\"id\":\"u-42\",\"note\":\"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa","truncated":true,"size":80}
  {"preview":"{\"access_token\":\"2YotnFZFEjr1zCsicMWpAA\",\"token_type\":\"example\",","truncated":true,"size":159}
  {"truncated":true,"size":16}
```

The run used Slack's documented placeholder webhook, all zeros and `X`s. It's shown here as
`<team>/<bot>/<secret>` because secret scanners, GitHub's push protection among them, match the
shape of a webhook URL and can't tell a placeholder from a live one. The function passed it
through unchanged either way, which is the point.

The headers kept their names and lost their values; `Authorization` kept its scheme.
`Idempotency-Key` went too, because its name contains `key`. The rule over-redacts, and here it
hides the value you'd want when debugging a retry
([file 04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key)).
Over-redacting is the safe way to be wrong; just know it happens. The query lost `api_key` and
`access_token` and kept `user` and `page`. `URL.Redacted`, the standard library's own
redaction, replaced the password in `svc:hunter2@` and nothing else, as its documentation
says: "Only the password in u.User is redacted." The second body was cut at 63 bytes, before
the `é`, not at 64. The PNG kept only its size.

### What redaction by name misses

The output already shows two misses. The third URL came through whole, and the token response
kept its access token in the preview. A name rule can't see a secret where no name says so:

| Where the secret is | Example | Caught by the name rule? |
|---|---|---|
| A response body | A token endpoint's answer, `{"access_token":"2YotnFZFEjr1zCsicMWpAA", … "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}` ([RFC 6749 §5.1](https://www.rfc-editor.org/rfc/rfc6749.html#section-5.1)) | No |
| A request body | A token refresh sent as a form, `…&client_id=s6BhdRkqt3&client_secret=7Fjfp0ZBr1KtDRbnfVdmIw` ([§2.3.1](https://www.rfc-editor.org/rfc/rfc6749.html#section-2.3.1)) | No |
| A URL path | A Slack incoming webhook: "Your webhook URL contains a secret" ([Slack](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/)) | No |
| A value made from a secret | A JWT signed with a stored key, sent in a header named `X-Signature` | No |

Applying the same rule to the keys of a JSON or form body would catch the first two, whose
names contain `token` and `secret`. It wouldn't catch a secret under any other name, or one
inside a string. The other two approaches have holes of their own:

| Approach | Example | What it misses |
|---|---|---|
| By name | The code above | Secrets under innocent names, in bodies and in paths |
| By value: mask the secrets you know | GitHub Actions masks registered secrets in logs, and is frank about the limits: "automatic redaction is not guaranteed", because "redaction largely relies on finding an exact match for the specific secret value", and a JWT made from a secret must be registered itself "or else it won't be redacted" ([GitHub](https://docs.github.com/en/actions/reference/security/secure-use)) | Every secret you didn't hand out: the token an upstream returns is new to you |
| Everything | n8n's "Redact production execution data" setting "hides the input and output data of each node and replaces it with a redacted indicator" ([n8n](https://docs.n8n.io/workflows/settings/)) | Nothing, and leaves nothing to debug from |

An engine can do better than any one of them by combining them. Mask by value the secrets it
stores for workflows, wherever they turn up; let authors mark a step's bodies as sensitive, a
token exchange being the obvious case; and apply the name rule to everything else.

> ⚠️ **A redacted run record is still sensitive data.** Every approach above filters what
> someone thought of in advance, and none is a guarantee. Give the record the access control
> and the retention period you'd give the secrets it might still hold.

## 8.4 Runs a crash interrupts

When the process dies, every run it was executing stops mid-step, and the record says `running`
for ever unless something corrects it. File 04 §4.6 lays out the spectrum of answers; this
section is about what the two ends mean for the record.

**Durable execution** resumes the run. Temporal describes its version: "if the Worker crashes,
the Worker uses the Event History to replay the code and recreate the state of the Workflow
Execution to what it was immediately before the crash. It then resumes progress from the point
of failure as if the failure never occurred" ([Temporal: Event History](https://docs.temporal.io/encyclopedia/event-history)).
There the record and the recovery mechanism are the same thing: the history is what gets
replayed.

**Marking them failed** is the simple end. At start-up, find the runs still marked `running`
and fail them with a reason, `interrupted`, so nobody waits for a run that no process is
executing (file 04 §4.6, level 1). A step that was in flight gets the same treatment, and for a
POST, `outcome unknown` ([file 04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key))
is more honest than `failed`: the request may have arrived.

That sweep is only safe when the process doing it is the only one. If two run at once, the new
one's "runs still marked running" includes runs the old one is executing right now. It marks
them failed. The old process then finishes them and either writes `succeeded` over `failed`,
after anyone watching has acted on the failure, or, if its writes are conditional on the status
still being `running`, can't write at all, and the record says a successful run failed.

Two processes overlap more often than "we run one server" suggests. Kubernetes's default way of
replacing a Deployment's pods, RollingUpdate, starts new pods before old ones stop: `maxSurge`
defaults to 25%, and the absolute number "is calculated from the percentage by rounding up"
([Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge)),
so a one-replica Deployment surges to two. Its other strategy, Recreate, is the one where "All
existing Pods are killed before new ones are created", and even that has a caveat: delete a pod
by hand and "the replacement will be created immediately (even if the old Pod is still in a
Terminating state)" ([Recreate](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#recreate-deployment)).

So the start-up sweep needs either a deployment that stops one process before starting the
next, or an owner and a heartbeat on every run, the **lease** from file 04 §4.6, so that it
fails only runs whose owner has stopped renewing.

A restart takes the in-memory event log of §8.7 with it. A browser that reconnects after one
gets the run's final state from the database, `failed: interrupted`, not a replay. That's also
why §8.5's event ids can start at 1 for every run: no run outlives the process that numbered
its events. An engine that resumes runs needs ids that survive the restart, such as a sequence
number from the record.

## 8.5 Following a run with server-sent events

The live view needs the server to push. A browser offers three ways:

| | Polling the run record | Server-sent events | WebSocket |
|---|---|---|---|
| Direction | The client asks, again and again | Server to client | Both ways: "two-way communication between a client … to a remote host" ([RFC 6455](https://www.rfc-editor.org/rfc/rfc6455.html)) |
| On the wire | Ordinary requests | One ordinary HTTP response that keeps going | HTTP for the handshake only: "Its only relationship to HTTP is that its handshake is interpreted by HTTP servers as an Upgrade request" (§1.7) |
| After a dropped connection | The next poll catches up | The browser reconnects and says which event it saw last | Nothing in the browser's API resumes; the page reconnects and re-syncs itself |
| Costs | A request per interval, and up to an interval of lag | An open response per viewer | An open connection per viewer, and a second protocol for every proxy on the path |

A run's progress flows one way, and the live view needs resuming far more than a back channel,
so **server-sent events** (SSE) fit. The authority is the WHATWG HTML Living Standard's
[§9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html), which I read as last
updated 26 October 2026.

### The format

An event stream is the body of one HTTP response with `Content-Type: text/event-stream`, which
"must always be encoded as UTF-8"
([§9.2.5](https://html.spec.whatwg.org/multipage/server-sent-events.html#parsing-an-event-stream)).
The browser reads it line by line
([§9.2.6](https://html.spec.whatwg.org/multipage/server-sent-events.html#event-stream-interpretation)):

| A line that… | Means |
|---|---|
| is empty | "Dispatch the event": everything since the previous blank line is one event |
| starts with `:` | "Ignore the line". A comment, and how a server keeps a quiet stream alive (§8.6) |
| is `event: name` | The event's type. Without one it's `message` |
| is `data: text` | Appended to the event's data, with a line feed between `data` lines |
| is `id: value` | Sets the "last event ID buffer", unless the value contains a NUL |
| is `retry: digits` | Sets the reconnection time in milliseconds, if the value "consists of only ASCII digits" |
| is anything else | Ignored |

A line may end in CRLF, LF or a lone CR, and one space after the colon is dropped. An event the
stream stops in the middle of is thrown away: "If the file ends in the middle of an event,
before the final empty line, the incomplete event is not dispatched." Here are the run's first
two events, with a comment between them, as this section's server sends them:

```
id: 1
event: step
data: {"step":"user","status":"succeeded"}

: keep-alive

id: 2
event: step
data: {"step":"prefs","status":"succeeded"}

```

### Ids, and what a reconnect sends

The `id` field is what makes resuming possible. When an event is dispatched, the browser sets
the stream's "last event ID string" to the id buffer, and "The buffer does not get reset", so
the id carries over to later events that lack one. When the connection drops, the browser waits
the reconnection time, "an implementation-defined value, probably in the region of a few
seconds" unless `retry` set it, and asks again. If the last event ID string isn't empty, it adds
it as a header: "Set (`Last-Event-ID`, lastEventIDValue) in request's header list"
([§9.2.3](https://html.spec.whatwg.org/multipage/server-sent-events.html#sse-processing-model)).
**`Last-Event-ID`** is the browser telling the server where it got to, and the server sends
what came after.

Three rules about that reconnect decide how the server must behave, and none is obvious:

1. **It reconnects after a clean end too.** When the response ends without a network error,
   the spec says to "reestablish the connection". The way out is a status code: "a client can
   be told to stop reconnecting using the HTTP 204 No Content response code"
   ([§9.2.1](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events-intro)).
   So a server whose run has finished answers a reconnect that already has the last event with
   a 204. I checked with undici's `EventSource` (Node 24.14.1, behind
   `--experimental-eventsource`) against this section's Go server: five events, then, three
   seconds after the stream ended, a reconnect with `Last-Event-ID: 5`, a 204, and
   `readyState` `CLOSED`.
2. **An event counts only once it's complete.** The last event ID string changes when the event
   is dispatched, at its blank line. An event cut off half-way doesn't count, so the reconnect
   asks for it again.
3. **Any answer but a 200 event stream ends it for good.** "if res's status is not 200, or if
   res's `Content-Type` is not `text/event-stream`, then fail the connection", and "Once the
   user agent has failed the connection, it does not attempt to reconnect."

> ⚠️ **A deploy can end a live view silently.** A reconnect that lands while the server is
> restarting meets the proxy's error page, not a stream. nginx 1.27.5, measured, answered
> `502 text/html` at once when nothing was listening behind it, and `504 text/html` when the
> server had vanished and its connect timeout ran out (§8.8). Either fails the `EventSource`
> for good. With undici's `EventSource`, I dropped a connection after two events and answered
> the reconnect with a 502: two connections, `readyState` `CLOSED`, no third attempt. A page
> using `EventSource` has to watch `onerror` for `readyState === EventSource.CLOSED` and start
> a new one itself, with backoff.

```mermaid
sequenceDiagram
    participant B as Browser (EventSource)
    participant S as Server
    B->>S: GET /runs/r1/events
    S-->>B: 200 text/event-stream, then id: 1, id: 2
    rect rgb(255, 225, 225)
    Note over B,S: the connection drops
    end
    Note over B: waits the reconnection time
    B->>S: GET, Last-Event-ID: 2
    S-->>B: 200, then id: 3 to id: 5 from the log
    Note over S: the run is over, so the response ends
    B->>S: GET, Last-Event-ID: 5
    S-->>B: 204 No Content
    Note over B: CLOSED: no more reconnects
```

### The server

Here is the server in Go. It keeps the run's events in memory, numbered from 1; replays what a
client missed; flushes every event; sends a comment after a second with nothing sent; clears
the deadline `Server.WriteTimeout` would put on the response (§8.6 shows that cut); and answers
a client that already has the whole run with a 204. It's `main.go` as I ran it with Go 1.27.2.
Left out are `main`, which serves `GET /runs/r1/events` from a server with
`ReadHeaderTimeout: 5s` and `WriteTimeout: 3s`, and `fakeRun`, a goroutine standing in for the
scheduler that appends the running example's four steps and then the run's outcome, the first
at 0.5 s and the rest 1.5 s apart.

```go
const keepAlive = time.Second // 15 s or so in production

// runLog holds one run's events in memory, numbered from 1, so that a
// client that connects late, or reconnects, can be sent what it missed.
type runLog struct {
	mu      sync.Mutex
	events  []string      // events[i] is event i+1, already formatted
	done    bool          // the run has finished: no more events
	changed chan struct{} // closed, then replaced, on every append
}

func newRunLog() *runLog { return &runLog{changed: make(chan struct{})} }

func (l *runLog) append(event, data string, last bool) {
	l.mu.Lock()
	defer l.mu.Unlock()
	id := len(l.events) + 1
	l.events = append(l.events, fmt.Sprintf("id: %d\nevent: %s\ndata: %s\n\n", id, event, data))
	l.done = last
	close(l.changed) // wakes every client waiting in serve
	l.changed = make(chan struct{})
}

// since returns the events after id, the id to resume from next time,
// whether the run is over, and a channel that closes on the next append.
// One lock covers all four, so no event can fall between "here's what you
// missed" and "wait for more".
func (l *runLog) since(id int) ([]string, int, bool, <-chan struct{}) {
	l.mu.Lock()
	defer l.mu.Unlock()
	if id < 0 || id > len(l.events) {
		id = 0 // an id this log never issued: start from the beginning
	}
	return l.events[id:], len(l.events), l.done, l.changed
}

func (l *runLog) serve(w http.ResponseWriter, r *http.Request) {
	// A reconnecting EventSource sends the id of the last event it saw.
	last, _ := strconv.Atoi(r.Header.Get("Last-Event-ID"))
	if _, next, done, _ := l.since(last); done && last == next {
		// It has the whole run. EventSource reconnects even after a clean
		// end; 204 is what tells it to stop.
		w.WriteHeader(http.StatusNoContent)
		return
	}

	rc := http.NewResponseController(w)
	// Server.WriteTimeout is a deadline for the whole response, counted
	// from when the request was read. A stream has to outlive it, so clear
	// it for this response. The zero time means no deadline.
	if err := rc.SetWriteDeadline(time.Time{}); err != nil {
		http.Error(w, "can't stream", http.StatusInternalServerError)
		return
	}
	h := w.Header()
	h.Set("Content-Type", "text/event-stream")
	h.Set("Cache-Control", "no-store")
	h.Set("X-Accel-Buffering", "no") // tells nginx not to buffer this response

	idle := time.NewTimer(keepAlive)
	defer idle.Stop()
	send := func(s string) bool {
		if _, err := io.WriteString(w, s); err != nil {
			return false
		}
		idle.Reset(keepAlive)
		return rc.Flush() == nil // out of Go's buffer and onto the wire
	}

	w.WriteHeader(http.StatusOK)
	if rc.Flush() != nil {
		return
	}
	for {
		events, next, done, changed := l.since(last)
		for _, e := range events {
			if !send(e) {
				return
			}
		}
		last = next
		if done {
			return // the run is over and this client has all of it
		}
		select {
		case <-changed:
		case <-idle.C: // a second with nothing sent
			if !send(": keep-alive\n\n") {
				return
			}
		case <-r.Context().Done():
			return
		}
	}
}
```

To run it, I started four `curl -sSN` clients from a small Node driver that stamps each line of
their output with the seconds since the server started. A connects at 0.2 s. B connects at 4.2 s
with `Last-Event-ID: 1`, as a browser would after dropping its connection just after event 1. C
connects at 8.0 s, after the run, with `Last-Event-ID: 2`. D connects at 8.5 s with
`Last-Event-ID: 5`, which is all of it, and prints only the status:

```
 0.2s  [A connects]
 0.5s  id: 1
 0.5s  event: step
 0.5s  data: {"step":"user","status":"succeeded"}
 0.5s
 1.5s  : keep-alive
 1.5s
 2.0s  id: 2
 2.0s  event: step
 2.0s  data: {"step":"prefs","status":"succeeded"}
 2.0s
 3.0s  : keep-alive
 3.0s
 3.5s  id: 3
 3.5s  event: step
 3.5s  data: {"step":"orders","status":"succeeded"}
 3.5s
 4.5s  : keep-alive
 4.5s
 5.0s  id: 4
 5.0s  event: step
 5.0s  data: {"step":"crm","status":"succeeded"}
 5.0s
 6.0s  : keep-alive
 6.0s
 6.5s  id: 5
 6.5s  event: run
 6.5s  data: {"status":"succeeded"}
 6.5s
 6.5s  [curl exit 0]

 4.2s  [B connects, Last-Event-ID: 1]
 4.2s  id: 2
 4.2s  event: step
 4.2s  data: {"step":"prefs","status":"succeeded"}
 4.2s
 4.2s  id: 3
 4.2s  event: step
 4.2s  data: {"step":"orders","status":"succeeded"}
 4.2s
 5.0s  id: 4
 5.0s  event: step
 5.0s  data: {"step":"crm","status":"succeeded"}
 5.0s
 6.0s  : keep-alive
 6.0s
 6.5s  id: 5
 6.5s  event: run
 6.5s  data: {"status":"succeeded"}
 6.5s
 6.5s  [curl exit 0]

 8.0s  [C connects, Last-Event-ID: 2]
 8.0s  id: 3
 8.0s  event: step
 8.0s  data: {"step":"orders","status":"succeeded"}
 8.0s
 8.0s  id: 4
 8.0s  event: step
 8.0s  data: {"step":"crm","status":"succeeded"}
 8.0s
 8.0s  id: 5
 8.0s  event: run
 8.0s  data: {"status":"succeeded"}
 8.0s
 8.0s  [curl exit 0]

 8.5s  [D connects, Last-Event-ID: 5]
 8.5s  HTTP 204
 8.5s  [curl exit 0]
```

B got events 2 and 3 the moment it connected, then event 4 live at 5.0 s, the same moment A
did. C connected after the run and got 3, 4 and 5 at once, then the end of the response. D
already had everything and got a 204. The keep-alives come a second after the last thing sent,
so A gets one between every pair of events, and C, which never waits, gets none.

Three things in the code are easy to get wrong:

- **One lock for "what you missed" and "wait for more".** `since` returns the missed events
  and the channel to wait on under the same lock. Take them under two locks and an event
  appended in between is never sent: it isn't in the copy, and the channel it closed is one the
  handler never waited on.
- **Closing a channel is Go's broadcast.** Every client blocked in `select` wakes at once, and
  `append` makes a fresh channel for the next event. A channel send would wake one client.
- **An id the log never issued means start again.** A `Last-Event-ID` larger than the log, as
  after the restart in §8.4, gets the whole log, not silence.

<details><summary>Rust</summary>

Standard library only, compiled with rustc 1.96.0 (`--edition 2024 -O`), clean under clippy. A
thread per connection; a `Mutex` and a `Condvar` do the job of Go's closed channel, and
`wait_timeout_while` is the keep-alive timer too. `fake_run`, the same stand-in on a thread, is
left out.

```rust
// One run's progress as server-sent events. Standard library only.
use std::io::{BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

const KEEP_ALIVE: Duration = Duration::from_secs(1); // 15 s or so in production

/// One run's events in memory, numbered from 1, so a client that connects
/// late, or reconnects, can be sent what it missed.
#[derive(Default)]
struct RunLog {
    events: Vec<String>, // events[i] is event i+1, already formatted
    done: bool,          // the run has finished: no more events
}

type Shared = Arc<(Mutex<RunLog>, Condvar)>;

fn append(run: &Shared, event: &str, data: &str, last: bool) {
    let (lock, changed) = &**run;
    let mut log = lock.lock().unwrap();
    let id = log.events.len() + 1;
    log.events
        .push(format!("id: {id}\nevent: {event}\ndata: {data}\n\n"));
    log.done = last;
    changed.notify_all(); // wakes every client waiting in serve
}

/// Reads the request head and returns its Last-Event-ID, or 0. None means
/// the client went away before finishing the head.
fn last_event_id(stream: &TcpStream) -> std::io::Result<Option<usize>> {
    let mut reader = BufReader::new(stream);
    let mut line = String::new();
    let mut last = 0;
    loop {
        line.clear();
        if reader.read_line(&mut line)? == 0 {
            return Ok(None);
        }
        if line == "\r\n" || line == "\n" {
            return Ok(Some(last));
        }
        if let Some((name, value)) = line.split_once(':')
            && name.eq_ignore_ascii_case("last-event-id")
        {
            last = value.trim().parse().unwrap_or(0);
        }
    }
}

fn serve(mut stream: TcpStream, run: Shared) -> std::io::Result<()> {
    let Some(mut last) = last_event_id(&stream)? else {
        return Ok(());
    };
    let (lock, changed) = &*run;
    {
        let log = lock.lock().unwrap();
        if log.done && last == log.events.len() {
            // It has the whole run; 204 tells EventSource to stop reconnecting.
            return stream.write_all(b"HTTP/1.1 204 No Content\r\nConnection: close\r\n\r\n");
        }
    }
    stream.set_nodelay(true)?; // send each write at once; Go's net does this by default
    // No Content-Length and no chunked coding: the body ends when we close.
    stream.write_all(
        b"HTTP/1.1 200 OK\r\nContent-Type: text/event-stream\r\nCache-Control: no-store\r\n\
          X-Accel-Buffering: no\r\nConnection: close\r\n\r\n",
    )?;
    loop {
        // Take what this client hasn't seen, under the lock; write it after.
        let (missed, done) = {
            let mut log = lock.lock().unwrap();
            if last > log.events.len() {
                last = 0; // an id this log never issued: start from the beginning
            }
            log = changed
                .wait_timeout_while(log, KEEP_ALIVE, |l| l.events.len() == last && !l.done)
                .unwrap()
                .0;
            (log.events[last..].to_vec(), log.done)
        };
        if missed.is_empty() && !done {
            stream.write_all(b": keep-alive\n\n")?; // a second with nothing sent
        }
        for e in &missed {
            stream.write_all(e.as_bytes())?; // straight to the socket: std has no buffer here
        }
        last += missed.len();
        if done {
            return Ok(()); // the run is over and this client has all of it
        }
    }
}

fn main() -> std::io::Result<()> {
    let run: Shared = Arc::default();
    let fake = Arc::clone(&run);
    thread::spawn(move || fake_run(&fake));

    let listener = TcpListener::bind("127.0.0.1:8090")?;
    for stream in listener.incoming() {
        let (stream, run) = (stream?, Arc::clone(&run));
        thread::spawn(move || serve(stream, run)); // an error just ends this client
    }
    Ok(())
}
```

Two things Go did for free. `set_nodelay(true)` turns off Nagle's algorithm, which
[RFC 1122 §4.2.3.4](https://www.rfc-editor.org/rfc/rfc1122.html#section-4.2.3.4) describes as
buffering "all user data … until the outstanding data has been acknowledged". Go's `TCPConn`
documents "The default is true (no delay)"; a socket accepted by Rust's `TcpListener` reported
`nodelay()` as `false`. Over loopback I couldn't measure a difference in the replay either way.
And this server sends no chunked coding, so its body ends when it closes the connection. A cut
connection then looks exactly like the end
([http file 02](../http/02-versions-and-framing.md#http11-is-a-byte-stream-with-no-envelope)),
and only the run's final event tells a client it has everything. §8.6 shows nginx treating such
a body differently as well.

</details>

<details><summary>C++</summary>

C++20 and POSIX sockets, compiled with Apple clang 17 (`-std=c++20 -O2 -Wall -Wextra
-Wpedantic`, no warnings). A detached thread per connection; `std::condition_variable::wait_for`
is both the wake-up and the keep-alive timer. `fake_run` is left out.

```cpp
// One run's progress as server-sent events. C++20 and POSIX sockets.
#include <arpa/inet.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/socket.h>
#include <unistd.h>

#include <algorithm>
#include <cctype>
#include <charconv>
#include <chrono>
#include <condition_variable>
#include <csignal>
#include <mutex>
#include <optional>
#include <string>
#include <string_view>
#include <thread>
#include <vector>

using namespace std::chrono_literals;
constexpr auto keep_alive = 1s; // 15 s or so in production

// One run's events in memory, numbered from 1, so a client that connects
// late, or reconnects, can be sent what it missed.
struct RunLog {
    std::mutex mu;
    std::condition_variable changed;
    std::vector<std::string> events; // events[i] is event i+1, already formatted
    bool done = false;               // the run has finished: no more events

    void append(const std::string& event, const std::string& data, bool last) {
        std::lock_guard lock(mu);
        events.push_back("id: " + std::to_string(events.size() + 1) + "\nevent: " + event +
                         "\ndata: " + data + "\n\n");
        done = last;
        changed.notify_all(); // wakes every client waiting in serve
    }
};

static bool send_all(int fd, std::string_view s) {
    while (!s.empty()) {
        ssize_t n = ::send(fd, s.data(), s.size(), 0);
        if (n <= 0) return false;
        s.remove_prefix(size_t(n));
    }
    return true;
}

// Reads the request head and returns its Last-Event-ID, or 0. nullopt
// means the client went away before finishing the head.
static std::optional<size_t> last_event_id(int fd) {
    std::string head;
    char buf[1024];
    while (head.find("\r\n\r\n") == std::string::npos) {
        ssize_t n = ::recv(fd, buf, sizeof buf, 0);
        if (n <= 0 || head.size() > 16384) return std::nullopt;
        head.append(buf, size_t(n));
    }
    std::transform(head.begin(), head.end(), head.begin(),
                   [](unsigned char c) { return char(std::tolower(c)); });
    size_t last = 0;
    if (auto at = head.find("\r\nlast-event-id:"); at != std::string::npos) {
        const char* p = head.data() + at + 16;
        while (*p == ' ') ++p;
        std::from_chars(p, head.data() + head.size(), last); // leaves 0 if no digits
    }
    return last;
}

static void serve(int fd, RunLog& run) {
    int one = 1;
    setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof one); // send each write at once
    auto last = last_event_id(fd);
    bool has_all = false;
    if (last) {
        std::lock_guard lock(run.mu);
        has_all = run.done && *last == run.events.size();
    }
    if (has_all) // it has the whole run; 204 tells EventSource to stop reconnecting
        send_all(fd, "HTTP/1.1 204 No Content\r\nConnection: close\r\n\r\n");
    // No Content-Length and no chunked coding: the body ends when we close.
    bool ok = last && !has_all &&
              send_all(fd, "HTTP/1.1 200 OK\r\nContent-Type: text/event-stream\r\n"
                           "Cache-Control: no-store\r\nX-Accel-Buffering: no\r\n"
                           "Connection: close\r\n\r\n");
    while (ok) {
        // Take what this client hasn't seen, under the lock; write it after.
        std::vector<std::string> missed;
        bool done;
        {
            std::unique_lock lock(run.mu);
            if (*last > run.events.size()) *last = 0; // an id this log never issued
            run.changed.wait_for(lock, keep_alive,
                                 [&] { return run.events.size() > *last || run.done; });
            missed.assign(run.events.begin() + std::ptrdiff_t(*last), run.events.end());
            done = run.done;
        }
        if (missed.empty() && !done) ok = send_all(fd, ": keep-alive\n\n"); // a second with nothing sent
        for (const auto& e : missed) ok = ok && send_all(fd, e); // send() is unbuffered
        *last += missed.size();
        if (done) break; // the run is over and this client has all of it
    }
    close(fd);
}

int main() {
    std::signal(SIGPIPE, SIG_IGN); // a write to a closed socket fails instead of killing us
    static RunLog run;
    std::thread(fake_run, std::ref(run)).detach();

    int ls = socket(AF_INET, SOCK_STREAM, 0);
    int one = 1;
    setsockopt(ls, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8090);
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    if (bind(ls, reinterpret_cast<sockaddr*>(&addr), sizeof addr) != 0 || listen(ls, 64) != 0)
        return 1;
    for (;;) {
        int fd = accept(ls, nullptr, nullptr);
        if (fd >= 0) std::thread(serve, fd, std::ref(run)).detach();
    }
}
```

`SIGPIPE` is ignored because a write to a connection the client has closed would otherwise end
the process; with it ignored, `send` returns an error and only that client's thread ends. The
Rust version's notes on `TCP_NODELAY` and on ending the body by closing apply here too.

</details>

<details><summary>JavaScript</summary>

An ES module on `node:http`, run with Node 24.14.1, no packages. The stand-in run at the bottom,
a loop that calls `append` and sleeps, is left out. Node runs this on one thread, so nothing can
be appended between the replay and joining the client set, and there's no lock. What it needs
instead is care with `write()`, which never blocks: §8.7 measures what happens without it.

```js
// One run's progress as server-sent events, with node:http.
import http from "node:http";

const KEEP_ALIVE = 1000; // ms; 15 s or so in production

// One run's events in memory, numbered from 1, so a client that connects
// late, or reconnects, can be sent what it missed.
const events = []; // events[i] is event i+1, already formatted
let done = false; // the run has finished: no more events
const clients = new Set();

function append(event, data, last) {
  events.push(`id: ${events.length + 1}\nevent: ${event}\ndata: ${data}\n\n`);
  done = last;
  for (const c of clients) pump(c);
}

// write() never blocks. Past the high-water mark it queues the data in this
// process and returns false, and nothing stops you writing more. So stop at
// false and carry on at 'drain'. The log is the only buffer: a client that
// falls behind is just a smaller c.next.
function write(c, s) {
  c.waiting = !c.res.write(s);
  c.idle.refresh(); // restart the keep-alive countdown
}

function pump(c) {
  while (!c.waiting && c.next < events.length) write(c, events[c.next++]);
  if (done && c.next === events.length && !c.res.writableEnded) c.res.end();
}

http
  .createServer((req, res) => {
    // A reconnecting EventSource sends the id of the last event it saw.
    let last = Number.parseInt(req.headers["last-event-id"] ?? "0", 10) || 0;
    if (done && last === events.length) {
      return res.writeHead(204).end(); // it has the whole run; 204 stops EventSource
    }
    if (last < 0 || last > events.length) last = 0; // an id this log never issued
    res.writeHead(200, {
      "content-type": "text/event-stream",
      "cache-control": "no-store",
      "x-accel-buffering": "no", // tells nginx not to buffer this response
    });
    const c = { res, next: last, waiting: false };
    c.idle = setInterval(() => c.waiting || write(c, ": keep-alive\n\n"), KEEP_ALIVE);
    res.on("drain", () => {
      c.waiting = false;
      pump(c);
    });
    res.on("close", () => {
      clearInterval(c.idle);
      clients.delete(c);
    });
    clients.add(c);
    pump(c); // the replay; one thread, so no event can arrive in between
  })
  .listen(8090, "127.0.0.1");
```

</details>

All four print the same transcript, timestamps included, to a tenth of a second.

### Reading the stream with a bearer token

The browser's own client, `EventSource`, takes a URL and one option
([§9.2.2](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface)):

```
dictionary EventSourceInit {
  boolean withCredentials = false;
};
```

Setting `withCredentials` to true sets the request's credentials mode to `include`, which
sends cookies. There's no way to add a header. An app whose API calls carry
`Authorization: Bearer …` has three options:

| Option | Cost |
|---|---|
| Authenticate the stream with a cookie | Fine if the app already has a session cookie; a second auth mechanism if it doesn't |
| Put the token in the URL, `?access_token=` | The URL lands in access logs and proxy logs, the very reason §8.3 redacts that parameter. If you must, make it a short-lived, single-use ticket |
| Read the stream with `fetch()` | You write what `EventSource` did for you: the parsing, the last id, the reconnecting |

Microsoft's fetch-event-source library exists for the third. Its README gives "You cannot pass
in custom request headers" among `EventSource`'s restrictions, and makes the proxy point from
the warning above: "if the gateway returns an error, you might want to handle it correctly"
([Azure/fetch-event-source](https://github.com/Azure/fetch-event-source)). Its last release,
2.0.1, was in April 2021. Here's the third option without a library. It keeps `EventSource`'s
rules but one: a 5xx means try again.

```js
// Follows an event stream with fetch(), which can send a bearer token where
// EventSource can't. It keeps EventSource's rules (parse, remember the last
// id, reconnect with Last-Event-ID, honour retry:, stop at 204) and drops
// one: a 5xx from a proxy means "try again", not "give up".
export async function follow(url, token, onEvent) {
  const s = { lastId: "", retry: 3000 }; // ms; the spec leaves the first wait to the browser
  for (;;) {
    const headers = { authorization: `Bearer ${token}` };
    if (s.lastId) headers["last-event-id"] = s.lastId;
    const res = await fetch(url, { headers, cache: "no-store" }).catch(() => null);
    if (res?.status === 204) return; // the server says there's nothing more
    if (res?.ok && res.headers.get("content-type")?.startsWith("text/event-stream")) {
      try {
        for await (const ev of parse(res.body, s)) {
          if (onEvent(ev) === false) return; // the caller has what it needs
        }
      } catch {
        // the connection dropped mid-stream: reconnect below
      }
    } else if (res && res.status < 500 && res.status !== 408 && res.status !== 429) {
      throw new Error(`can't follow ${url}: ${res.status}`); // a 401 won't fix itself
    }
    await new Promise((r) => setTimeout(r, s.retry)); // network error, 5xx, or a clean end
  }
}

// parse yields the events in a text/event-stream body, following the rules
// in WHATWG HTML §9.2.6, and updates s.lastId and s.retry as it goes.
async function* parse(body, s) {
  let buf = "";
  let data = [];
  let type = "";
  let id = ""; // the spec's "last event ID buffer", one per connection
  let afterCR = false;
  for await (let chunk of body.pipeThrough(new TextDecoderStream())) {
    // Lines end in CRLF, LF or CR. A chunk ending in CR has ended its line
    // already; if the next chunk starts with LF, that LF was the CR's pair.
    if (afterCR && chunk.startsWith("\n")) chunk = chunk.slice(1);
    afterCR = chunk.endsWith("\r");
    buf += chunk;
    const lines = buf.split(/\r\n|\n|\r/);
    buf = lines.pop();
    for (const line of lines) {
      if (line === "") { // a blank line dispatches the event
        s.lastId = id; // only now: an event cut off before here doesn't count
        if (data.length) yield { type: type || "message", data: data.join("\n"), id };
        data = [];
        type = "";
        continue;
      }
      if (line.startsWith(":")) continue; // a comment: keep-alives end up here
      const colon = line.indexOf(":");
      const field = colon < 0 ? line : line.slice(0, colon);
      let value = colon < 0 ? "" : line.slice(colon + 1);
      if (value.startsWith(" ")) value = value.slice(1);
      if (field === "data") data.push(value);
      else if (field === "event") type = value;
      else if (field === "id" && !value.includes("\0")) id = value;
      else if (field === "retry" && /^\d+$/.test(value)) s.retry = Number(value);
    }
  }
}
```

I ran it with Node 24.14.1 against a server that scripted each connection. The first sent
`retry: 500` and CRLF line endings, split mid-name (`event: st` | `ep`) and between a CR and
its LF, then cut event 3 off before its blank line. The second answered 502. The third used lone
CRs:

```
0.0s  connection 1: authorization sent, Last-Event-ID (none)
0.1s  step id 1: {"n":1}
0.1s  step id 2: {"n":2}
0.7s  connection 2: authorization sent, Last-Event-ID 2
1.2s  connection 3: authorization sent, Last-Event-ID 2
1.2s  step id 3: {"n":3}
1.3s  run id 4: {"status":"succeeded"}
1.3s  follow returned
```

The reconnects asked for everything after event 2, not 3, because event 3 never got its blank
line, and the third connection sent it again. Against a 401, and against a 200 that wasn't an
event stream, `follow` stopped with an error instead of retrying. The lone CR is the subtle
part. My first version waited for the next chunk to see whether a CR at the end of a chunk was
half of a CRLF. When the server then went quiet, the run's final event sat in the buffer and was
never dispatched. A CR ends its line at once; the next chunk's leading LF, if any, is the one to
drop.

## 8.6 What sits between the server and the browser

The stream works on a laptop. Then it goes behind a proxy, and the live view updates once, at
the end, or drops after a minute. Every layer between the handler and the browser can cut a
long response or hold it back:

| Layer | What it does to a stream | Measured here | The fix |
|---|---|---|---|
| The server's write timeout | Cuts the response a fixed time after the request was read | Go, `WriteTimeout: 3s`: cut at 3.5 s, curl error 18 | Clear it for the stream's response with `ResponseController` |
| Proxy buffering | Holds events until the response ends | nginx: all five events at 8.8 s, for some responses | Send `X-Accel-Buffering: no` |
| The proxy's read timeout | Closes a stream that's been silent too long | nginx, `proxy_read_timeout 3s`: cut 3 s after the last byte | A comment line well inside the timeout |
| Compression | May hold events to fill its window | Express's `compression`: all five at 7.5 s | Make each flush reach the client, or don't compress the stream |

```mermaid
flowchart LR
    H["handler<br/>Write + Flush"] --> G["Go's write deadline<br/>WriteTimeout"]
    G --> N["proxy<br/>buffering · read timeout · gzip"]
    N --> B["browser<br/>EventSource or fetch()"]
    style N fill:#264653,stroke:#8ecae6,color:#fff
```

The proxy is highlighted because it's the layer you can't see from either end: the server
flushed and the browser waited, and neither knows the events sat in between.

### The server's own write timeout

[Go-services file 03](../go-services/03-the-http-server.md#writetimeout-doesnt-stop-your-handler)
shows that `WriteTimeout` doesn't bound a handler's work. On a stream it does something worse:
it bounds the response. "WriteTimeout is the maximum duration before timing out writes of the
response. It is reset whenever a new request's header is read"
([net/http](https://pkg.go.dev/net/http#Server)). In the source, `readRequest` sets the
connection's write deadline to the moment the request was read plus `WriteTimeout`, so the
whole response has to be written by then, however long it was meant to run.

Here's §8.5's server with `WriteTimeout: 3s` and the `SetWriteDeadline` block taken out:

```
 0.2s  [A connects]
 0.5s  id: 1
 0.5s  event: step
 0.5s  data: {"step":"user","status":"succeeded"}
 0.5s
 1.5s  : keep-alive
 1.5s
 2.0s  id: 2
 2.0s  event: step
 2.0s  data: {"step":"prefs","status":"succeeded"}
 2.0s
 3.0s  : keep-alive
 3.0s
 3.5s  curl: (18) transfer closed with outstanding read data remaining
 3.5s  [curl exit 18]
```

A's deadline was 3.2 s. The keep-alive at 3.0 s got through; event 3, at 3.5 s, didn't, and the
server closed the connection mid-stream. B, connected at 4.2 s, finished at 6.5 s, inside its
own three seconds, and was fine. That's what makes this a production bug and not a test
failure: streams shorter than the timeout work. A browser reconnects and the replay hides the
cut, so what users see is a view that hiccups every 30 seconds, or whatever the timeout is.

The fix is per response. Go 1.20 added `http.ResponseController`, whose `SetWriteDeadline` "sets
the deadline for writing the response. … A zero value means no deadline"
([net/http](https://pkg.go.dev/net/http#ResponseController.SetWriteDeadline)). The Go 1.20
release notes' own example is this case:
`rc.SetWriteDeadline(time.Time{}) // disable Server.WriteTimeout when sending a large response`
([Go 1.20](https://go.dev/doc/go1.20)). Every other route keeps the timeout. A **write
deadline** cleared to zero has a cost, though, and §8.7 comes back to it: a client that stops
reading can now hold the handler for ever.

> ⚠️ **A middleware wrapper can hide the controller.** `NewResponseController` expects "the
> original value passed to the Handler.ServeHTTP method, or … an Unwrap method returning the
> original ResponseWriter", and otherwise its methods return an error matching
> `ErrNotSupported`. I wrapped the writer in the usual logging struct, which embeds
> `http.ResponseWriter` to record the status code: `SetWriteDeadline` and `Flush` both failed
> with `feature not supported`. Adding `func (r *statusRecorder) Unwrap() http.ResponseWriter`
> fixed both. That's why §8.5's server checks the error and refuses to stream, rather than
> stream under a deadline it couldn't clear.

Node's http server has nothing like it by default: `server.timeout` is "Default: 0 (no
timeout)", and `requestTimeout`'s 300 s covers "receiving the entire request from the client",
not the response ([node:http](https://nodejs.org/docs/latest-v24.x/api/http.html#servertimeout)).
The Rust and C++ servers set no timeouts at all.

### Proxy buffering

A reverse proxy may read the server's response before passing it on. nginx calls this
**buffering**, and it's on by default: "When buffering is enabled, nginx receives a response from
the proxied server as soon as possible, saving it into the buffers … When buffering is disabled,
the response is passed to a client synchronously, immediately as it is received"
([proxy_buffering](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering)).

I put nginx in front of the Go server in Docker, with only `proxy_pass` in its `location`, and
stamped the client's lines as before. The server's run started 3 s after the server did, so the
client was connected before the first event:

| nginx | Talks to the server in | Server sends `X-Accel-Buffering: no` | The five events reached the client at |
|---|---|---|---|
| 1.27.5 | HTTP/1.0, its default | No | 8.8 s, all together, the response headers with them |
| 1.27.5 | HTTP/1.0 | Yes | 2.8, 4.3, 5.8, 7.3, 8.8 s |
| 1.27.5 | HTTP/1.0, with `proxy_buffering off` | No | 2.7, 4.2, 5.7, 7.2, 8.7 s |
| 1.27.5 | HTTP/1.1, set with `proxy_http_version` | No | 2.8, 4.3, 5.8, 7.3, 8.8 s |
| 1.30.6 | HTTP/1.1, its default | No | 2.6, 4.1, 5.7, 7.2, 8.7 s |
| 1.30.6 | HTTP/1.0, set with `proxy_http_version` | No | 8.8 s, all together |
| 1.30.6 | HTTP/1.1, the server ending its body by closing, as the Rust and C++ servers do | No | 10.5 s, all together |
| 1.30.6 | HTTP/1.1, ending by closing | Yes | 3.0, 4.5, 6.0, 7.5, 9.0 s |

(The last two rows used a raw-socket stand-in for the Rust and C++ servers, whose events began
at 3.0 s.)

The deciding variable isn't nginx's version. It's whether nginx can tell how long the next
piece of the body is. A chunked body tells it: every chunk carries its length. A body that ends
when the connection closes doesn't. An HTTP/1.0 response of unknown length has to be that kind,
and so is the Rust and C++ servers' response, whichever version nginx speaks. In nginx's source
(1.30.0, `src/event/ngx_event_pipe.c`), a buffered response's partly filled buffer is handed on
early only `if (p->free_raw_bufs && p->length != -1)`, and the chunked filter sets that length
to the "minimal amount of data we want to see". A close-delimited body leaves it at −1, so the
events wait for a full buffer or the end. nginx also changed its default between the two versions I
ran: "Since 1.29.7, version 1.1 is used by default. Before 1.29.7, version 1.0 was used by
default" ([proxy_http_version](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version)).
So the same server behind the same configuration buffered on one nginx and streamed on the
other.

Don't depend on any of that. The fix that worked in every row is the header: "Buffering can also
be enabled or disabled by passing “yes” or “no” in the “X-Accel-Buffering” response header
field." nginx keeps it to itself: "By default, nginx does not pass the header fields “Date”,
“Server”, “X-Pad”, and “X-Accel-...” from the response of a proxied server to a client"
([proxy_hide_header](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_hide_header)),
and it wasn't among the headers the client received. `proxy_buffering off` in the configuration
works too, for every response in that `location`.

One more detail from the first row: the response headers waited as well. With
`postpone_output 0;` they arrived at once while the events still waited, because by default
nginx postpones "the transmission of client data … until nginx has at least size bytes of data
to send", with a size of 1460
([postpone_output](https://nginx.org/en/docs/http/ngx_http_core_module.html#postpone_output)).
An `EventSource` behind it wouldn't even fire `open` until the run was over.

### The proxy's idle timeout

nginx's `proxy_read_timeout` "Defines a timeout for reading a response from the proxied server.
The timeout is set only between two successive read operations, not for the transmission of
the whole response. If the proxied server does not transmit anything within this time, the
connection is closed." Its default is 60 s
([proxy_read_timeout](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout)).
A run whose slow step takes two minutes is two minutes of silence on the stream, and the
stream dies at one.

I set `proxy_read_timeout 3s` and spaced the events 5 s apart. With no keep-alives, the client
got event 1 at 2.8 s, then `curl: (18)` at 5.8 s, and nginx logged `upstream timed out (110:
Connection timed out) while reading upstream`, on both versions. With a comment after each
second of silence (on 1.27.5), all five events arrived, at 2.8, 7.8, 12.8, 17.8 and 22.8 s.

That's what a **keep-alive comment** is for. The WHATWG authoring notes: "Legacy proxy servers
are known to, in certain cases, drop HTTP connections after a short timeout. To protect against
such proxy servers, authors can include a comment line (one starting with a ':' character)
every 15 seconds or so" ([§9.2.7](https://html.spec.whatwg.org/multipage/server-sent-events.html#authoring-notes)).
The rule is to send one after a silence well short of the smallest idle timeout anywhere on the
path: nginx's 60 s, a load balancer's, a corporate proxy's you'll never see. Fifteen seconds
leaves room. The examples use one second only so the transcripts show them.

### Compression

Compression can hold events too, and whether it does depends on the layer:

- **Express's `compression` middleware** says so in its README: "Because of the nature of
  compression this module does not work out of the box with server-sent events. To compress
  content, a window of the output needs to be buffered up in order to get good compression", and
  its fix is calling `res.flush()` after each event
  ([expressjs/compression](https://github.com/expressjs/compression#server-sent-events)).
  Measured with compression 1.8.2 on `node:http`: without `res.flush()`, all five events arrived
  together at 7.5 s; with it, each arrived on time, still gzipped.
- **nginx's gzip** is off by default, and when on compresses only `text/html` unless
  `gzip_types` adds more ([gzip_types](https://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_types)).
  With `gzip_types text/event-stream` and an unbuffered response, nginx 1.27.5 passed each event
  through as it came, compressed.
- **Go's `gzhttp`**, from klauspost/compress v1.20.3, documents that it "supports the
  http.Flusher interface", and has an `Unwrap` method, so `ResponseController` reaches through it.

So check each layer that compresses: whether it compresses `text/event-stream` at all, and if it
does, whether your flush reaches the client through it. A stamped client like the ones above,
run through the real stack, answers both. Leaving the stream uncompressed is the simplest way to
be sure.

> ⚠️ **Over HTTP/1.1, each open stream holds one of very few connections.** The WHATWG notes
> warn that "Clients that support HTTP's per-server connection limitation might run into trouble
> when opening multiple pages from a site if each page has an EventSource to the same domain".
> MDN gives the limit as 6 per browser and domain, and says that over HTTP/2 the number of
> streams is "negotiated between the server and the client"
> ([MDN: EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). Seven tabs
> watching runs over HTTP/1.1 means the seventh waits, along with every other request to that
> site. Serve the stream over HTTP/2, or share one stream between tabs.

## 8.7 Replay, and clients that fall behind

### How long to keep the log

§8.5's servers keep each run's events in memory, numbered, and that log does three jobs. A
browser that opens a run late gets everything so far. One that reconnects gets what it missed.
And live clients read new events from it. It's bounded by the run: a few events per step.

When the run ends, keep the log a while longer: long enough for a browser that lost its
connection near the end to reconnect, which takes "a few seconds" by the spec's default, plus
room for slower stragglers. That while is the **replay window**, and a few minutes is a
defensible choice, not a standard one. After it, drop the log and answer requests for the run's
events with its final state from the database, as one **snapshot** event that replaces whatever
the client has, then end the response. Give the snapshot an id the server recognises as final,
so that a reconnect carrying it gets the 204.

> **Teacher's aside.** It's natural to think of the live stream as the run's progress and the
> database as its archive. Reverse it. The record (§8.2) is the truth, and the stream is a
> cache of recent changes to it, with an expiry. `Last-Event-ID` doesn't make delivery
> reliable. It's the browser asking "what came after 7?", and the answer depends entirely on
> what the server still has. `EventSource` has no way to notice a gap; it dispatches whatever
> arrives. So on every connection the server owes the client one of two things: exactly the
> events after the id it asked for, or an explicit "start again from this" that replaces its
> state. A server that does neither, such as one whose capped log quietly starts at event 9 when
> asked for 8, produces a view that's wrong with nothing on the screen to say so.

### Clients that fall behind

Browsers read at their own speed. One on a train reads slowly; one in a background tab may stop
reading altogether. The run mustn't care, and the server mustn't spend unbounded memory on it.
There are five ways to treat a client that can't keep up:

| Policy | What happens | What it costs | Seen in |
|---|---|---|---|
| Block the run | Publishing waits until every client has taken the event | One stuck browser stops the run | A publisher sending on an unbuffered channel per client |
| Buffer without limit | Queue whatever the client hasn't taken | Memory grows with every stuck client, without bound | Node, if you ignore `write()`'s return value (measured below) |
| Drop events silently | Skip the client for events it can't take | A gap the client can't detect: the view is simply wrong | — |
| Drop the client | Close its connection past a limit; it reconnects with `Last-Event-ID` and catches up from the log | Reconnect churn, and the limit has to allow for the replay (below) | Redis pub/sub |
| A cursor into the log | Each client is a position in the shared log, and its writer stops while its socket is full | A stuck client holds a connection until a stall timeout drops it | §8.5's servers |

Redis is the instructive example of dropping the client. Its server closes a pub/sub client
whose output buffer outgrows its limits, "a default hard limit of 32 megabytes and a soft limit
of 8 megabytes per 60 seconds"
([Redis: client handling](https://redis.io/docs/latest/develop/reference/clients/#output-buffer-limits)).
But pub/sub has no replay, so the dropped client's messages are gone: "If the subscriber is
unable to handle the message (for example, due to an error or a network disconnect) the message
is forever lost" ([Redis: pub/sub](https://redis.io/docs/latest/develop/pubsub/#delivery-semantics)).
With a numbered log and `Last-Event-ID`, dropping a client loses nothing, which is what makes it
safe here. And once the log is kept anyway, the per-client buffer can shrink to a number: the
client's position in the log. A client that falls behind is a smaller number.

Node shows the difference most clearly, because its `write()` never blocks. Its documentation:
"While calling write() on a stream that is not draining is allowed, Node.js will buffer all
written chunks until maximum memory usage occurs … Since TCP sockets may never drain if the
remote peer does not read the data, writing a socket that is not draining may lead to a remotely
exploitable vulnerability" ([node:stream](https://nodejs.org/docs/latest-v24.x/api/stream.html#writablewritechunk-encoding-callback)).
I served two clients three ways with Node 24.14.1: one that sent its request and never read a
byte while the run appended 100,000 events of 1 KiB, and a healthy one that joined a run that
already had 1,000:

| Strategy | The client that never reads | The healthy client joining late |
|---|---|---|
| Write everything | Still connected, with 99,088 KiB queued for it in the server | All 1,000 events |
| Drop the client past 64 KiB queued | Dropped at 64 KiB | **None**: cut before its first byte arrived |
| Cursor into the log | Still connected, with 65 KiB queued | All 1,000 events |

> ⚠️ **A drop threshold must be larger than the largest burst you send at once, and for an
> event stream that's the replay.** The late client's 1,000 events were written in one turn of
> the event loop, so they queued in the process before the kernel took any, and the 64th tripped
> the limit. A browser would reconnect, start its replay, be dropped, and do it again for ever,
> on the long runs people most want to watch.

The cursor kept the stuck client's queue near Node's high-water mark, 65,536 bytes for a
`ServerResponse` in Node 24 (measured), but it didn't drop it. That's the last piece: a deadline
on writes that make no progress. Each stack has one, and in each the default is to wait for ever:

| Stack | Where a stuck client's backlog sits | What drops it | Default |
|---|---|---|---|
| Go | The kernel's socket buffers; the handler blocks in `Write` | `ResponseController.SetWriteDeadline` before each write | §8.5's cleared deadline: never |
| Rust, std | The kernel's buffers; the thread blocks | `TcpStream::set_write_timeout`; with `None`, "write calls will block indefinitely" | `None` |
| C++, POSIX | The kernel's buffers; the thread blocks in `send` | `SO_SNDTIMEO`: "The default for this option is the value zero, which indicates that a send operation will not time out" ([POSIX](https://pubs.opengroup.org/onlinepubs/9799919799/functions/V2_chap02.html)) | 0 |
| Node | The process, if you ignore `write()`'s result; one high-water mark with a cursor | `response.setTimeout(ms, …)`, the socket's inactivity timeout | `server.timeout`: "0 (no timeout)" |

Measured in Go, with a client that sent its request and never read: with the deadline cleared,
the handler wrote between 538 and 1,472 KiB, depending on how far macOS had grown the socket
buffers (it autotunes them up to 4 MiB), and was still blocked in `Write` ten seconds later.
With a 2 s deadline set before each write, the flush that blocked failed after 2.0 s with
`i/o timeout` and the handler returned. In Node, `res.setTimeout(2000, …)` dropped the same
kind of client 2 s after its socket stopped taking data, while an idle client kept busy by
keep-alive comments stayed connected. Both used test harnesses, not the §8.5 servers. For the
Go server, the change is in `send`: a deadline per write in place of clearing it once, which
makes the clear at the top unnecessary. I ran §8.5's server with this `send` and its transcript
was unchanged:

```go
send := func(s string) bool {
	// A deadline per write, not none: a client that stops reading is
	// dropped after ten seconds, and catches up from the log when it's back.
	if rc.SetWriteDeadline(time.Now().Add(10*time.Second)) != nil {
		return false
	}
	if _, err := io.WriteString(w, s); err != nil {
		return false
	}
	idle.Reset(keepAlive)
	return rc.Flush() == nil
}
```

## 8.8 Answering a waiting caller: whose fault was it?

In a request-style run, the caller is still holding the request that started it. When the run
fails, the status code is the first thing the caller's software reads, and it should say whose
fault the failure was, because that decides what the caller does next: fix its request, retry
later, or page someone.

| What failed | Status | Whose fault | RFC 9110's definition |
|---|---|---|---|
| The caller's input doesn't match the workflow's input schema | `400` or `422`, as problem details with pointers ([file 06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457)) | The caller's | — |
| A step's upstream refused the connection or answered with an error | `502 Bad Gateway` | An upstream's | "the server, while acting as a gateway or proxy, received an invalid response from an inbound server it accessed while attempting to fulfill the request" ([§15.6.3](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6.3)) |
| A step's upstream didn't answer in time | `504 Gateway Timeout` | An upstream's, or a timeout set too tight | "the server, while acting as a gateway or proxy, did not receive a timely response from an upstream server it needed to access in order to complete the request" ([§15.6.5](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6.5)) |
| The workflow's own logic: a mapping that failed, an `$assert` that fired, or a bug in the engine | `500 Internal Server Error` | The workflow's author, or the engine's | "the server encountered an unexpected condition that prevented it from fulfilling the request" ([§15.6.1](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6.1)) |

Both 502 and 504 assume a gateway, which RFC 9110 defines as "an intermediary that acts as an
origin server for the outbound connection but translates received requests and forwards them
inbound to another server or servers" ([§3.7](https://www.rfc-editor.org/rfc/rfc9110.html#section-3.7)).
An engine running a request-style run is one, with a graph in the middle. nginx draws the same
line for its own upstreams: measured, 1.27.5 answered 502 at once when nothing was listening
behind it (`connect() failed (111: Connection refused)`), and 504 when the upstream had vanished
and `proxy_connect_timeout` ran out (its default is 60 s; I set 2 s).

One stretch is worth admitting. A step whose upstream answered a well-formed `503` didn't
receive "an invalid response" in the protocol's sense. But the engine couldn't fulfil the
request with what it got, and passing the upstream's status through would be worse: a 503 from
the engine says the engine is unavailable, and a 404 says the workflow's own URL doesn't exist.
A 502 with a body that names the step is the honest reading.

That body is a problem detail ([file 06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457))
with extension members, which RFC 9457 allows: "Problem type definitions MAY extend the problem
details object with additional members that are specific to that problem type"
([§3.2](https://www.rfc-editor.org/rfc/rfc9457.html#section-3.2)).

```json
{
  "type": "https://example.com/problems/upstream-failed",
  "title": "A step's upstream failed",
  "status": 502,
  "detail": "Step prefs got 503 Service Unavailable from its upstream.",
  "instance": "/runs/7f3a9c2e",
  "step": "prefs",
  "upstreamStatus": 503
}
```

`instance` is the run record's address, so whoever is allowed to can read the whole run. The
response itself carries no upstream body, no headers and no stack: RFC 9457's security
considerations warn against "making implementation details such as a stack dump available
through the HTTP interface" ([§5](https://www.rfc-editor.org/rfc/rfc9457.html#section-5)).
[File 02 §2.8](02-graphs-and-scheduling.md#28-panics-in-step-goroutines) puts a panicking step's
stack into the run record, and that's where it stays.

## Check yourself

1. In §8.2's first timeline the scheduler saves each record before carrying on. Which saves
   delayed the run, which didn't, and why? Now each save takes 400 ms instead of 100. Using the
   invented step durations (120, 300, 80 and 150 ms), how long does each version of the run
   take, and when is everything saved? What does someone reading the record at 1,000 ms see in
   the writer-goroutine version?
2. A workflow's first step POSTs `grant_type=client_credentials&client_id=c-1&client_secret=…`
   to a token endpoint, which answers with an access token in a JSON body, as RFC 6749 §5.1
   describes. The second calls
   `https://api.example.com/v1/orders?access_token=<that token>`. The third posts a summary to a
   Slack incoming webhook. With §8.3's code, which secrets end up in the run record, and for
   each, what would have kept it out?
3. A request-style run fails because `prefs`'s upstream answered `503` with `Retry-After: 30`.
   The engine copies both into its own response. What does a well-behaved client conclude, and
   what goes wrong for other workflows? What should the engine send instead, and what should it
   have done with the `Retry-After` before answering at all?
4. A live view that works in development misbehaves in production, behind nginx and a Go server
   with `WriteTimeout: 30 * time.Second` that never touches the write deadline. (a) The progress
   bar sits at zero and jumps to done when the run ends. (b) During a two-minute step the
   browser's network panel shows the stream reconnecting about once a minute. (c) Every run
   longer than 30 s reconnects about every 30 s. For each, name the layer, the mechanism and the
   fix. (b) and (c) lose no events: why not, how would you tell them apart from the server side,
   and what would turn either into lost events?
5. A teammate fixes the Node server's memory growth by dropping any client with more than 64 KiB
   queued. Tests pass. Then a customer with a 3,000-step run reports that its live view never
   loads, and the server's logs show the same browser connecting every few seconds. Explain what
   happens on each connection, why the tests passed, and how the cursor design avoids it without
   giving up the protection.
6. The engine runs as one replica under Kubernetes's default Deployment strategy, and marks runs
   still `running` as failed at start-up. A deploy starts while run R is half-way. Walk through
   what the new process does to R's record, what the old process does when it finishes R a few
   seconds later, and what two browsers see: one whose stream stays on the old process, and one
   that reconnects and lands on the new one. Why can't `Last-Event-ID` help the second? Give two
   ways out, and what each costs.

(Answers in [12-exercises.md](12-exercises.md).)
