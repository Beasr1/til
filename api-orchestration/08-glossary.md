# 8. Glossary

Reference, not reading. Each entry points at the file that explains it. Terms several
files use list each of them.

| Term | Meaning | Where |
|---|---|---|
| **`412 Precondition Failed`** | A condition such as `If-Match` evaluated to false, so the method wasn't applied | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **`428 Precondition Required`** | The server refuses requests that aren't conditional (RFC 6585 §3) | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **`abort_all`** | tokio `JoinSet` method that stops every task at its next `.await`; forced, unlike Go's `context` | [02 §2.3](02-graphs-and-scheduling.md#23-the-scheduler-loop) |
| **`AbortController` / `AbortSignal`** | JavaScript's cooperative cancellation: the controller aborts, and APIs holding the signal reject with an `AbortError` | [02 §2.3](02-graphs-and-scheduling.md#23-the-scheduler-loop) |
| **`about:blank`** | The default problem type: no meaning beyond the HTTP status code | [06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457) |
| **Activity** | Temporal's unit for anything non-deterministic or side-effecting, such as a call to another system, retried under a retry policy; workflow code stays deterministic, and one step maps to one activity | [01 §1.6](01-foundations.md#16-the-landscape), [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **Agents Rule of Two** | Meta's rule: until prompt injection can be reliably detected, an agent session may hold at most two of [A] untrustworthy input, [B] sensitive systems or data, [C] state change or external communication; all three needs supervision | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **`all_success`** | Rule: run only if every parent succeeded and every edge was taken. Airflow's default | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Allow-list vs block-list** | OWASP Case 1 (known destinations → allow-list) vs Case 2 (any URL → block-list, "not an impenetrable wall") | [05 §5.8](05-calling-urls-you-dont-control.md#58-allow-lists-vs-block-lists) |
| **Alternative IP spelling** | Non-dotted forms (decimal `2130706433`, hex `0x7f.1`, shorthand `127.1`, IPv4-mapped `::ffff:127.0.0.1`) that resolve to an address a naive string check misses | [05 §5.2](05-calling-urls-you-dont-control.md#52-why-validating-the-url-string-doesnt-work) |
| **Ancestor, descendant** | A step's parents, their parents and so on; its children, theirs and so on | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Approval fatigue** | Reviewers' judgement degrading as they approve at volume; how "a person publishes" turns into a rubber stamp | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Array constructor** | `[ ... ]` in an expression; builds a new array, so the result is an array even for zero or one item | [03](03-mapping-data-between-steps.md) |
| **`$assert`** | JSONata function that fails evaluation with `D3141` and your message when its condition is false; the way to make a mapping fail loudly | [03](03-mapping-data-between-steps.md) |
| **At-least-once execution** | A step may run more than once, for example after a crash between its call and its checkpoint; what every durability level gives for side effects | [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **Backtracking regex engine** | One that tries a way to match and goes back to try others on failure; some patterns then take exponential time. V8's Irregexp and `std::regex` are examples | [03](03-mapping-data-between-steps.md) |
| **Behind a cycle** | A step that can never start because an ancestor is on a cycle, though it isn't on one itself | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **Broken edge** | An edge whose parent failed or was upstream-failed; both rules here turn it into `upstream_failed` | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **CaMeL** | Debenedetti et al.'s framework for code-then-execute, with capabilities that police data flows | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Cancelled** | Step status: fail-fast or the caller's deadline stopped it, before or during its call | [02 §2.7](02-graphs-and-scheduling.md#27-failure-policy-fail-fast-or-carry-on) |
| **Carry on** | Failure policy that lets independent branches finish after a step fails; descendants are decided by their rules | [02 §2.7](02-graphs-and-scheduling.md#27-failure-policy-fail-fast-or-carry-on) |
| **Catalogue** | A platform's own list of what exists (nodes, apps, plugins, scripts, API operations) with their parameters; what a builder generates against | [07 §7.2](07-agents-that-build-workflows.md#72-what-the-products-do) |
| **CEL** | Common Expression Language: an embeddable expression language designed to terminate, with RE2 regexes | [03](03-mapping-data-between-steps.md) |
| **Checkpoint** | A step's result written durably as the step finishes, so a resumed run can skip it | [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **Choreography** | Coordination where each service reacts to events from the others, with no central plan. The opposite of orchestration; not this course's subject | [01 §1.2](01-foundations.md#12-the-chain-as-data) |
| **Claim check** | Storing a large body elsewhere and passing a small reference to it instead (Hohpe and Woolf, 2003) | [04 §4.7](04-retries-idempotency-and-durability.md#47-large-payloads-and-the-claim-check) |
| **Coalescing operator (`??`)** | Falls back only when the left side is missing; `null ?? 1` is `null` | [03](03-mapping-data-between-steps.md) |
| **Code-first** | Writing the server's handlers and types, and letting the framework emit the API description, as Huma does | [06 §6.3](06-describing-apis.md#63-spec-first-or-code-first) |
| **Code-then-execute** | The agent writes an explicit program, which then runs on untrusted data; generalises plan-then-execute. A workflow definition is one | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Compensation** | A new step that semantically undoes an earlier one, such as a refund for a charge; not a rollback | [04 §4.2](04-retries-idempotency-and-durability.md#42-retry-the-step-not-the-run) |
| **Concurrency limit** | The most steps a run may have in flight at once. Per run, not per upstream | [02 §2.4](02-graphs-and-scheduling.md#24-the-ready-set-and-a-concurrency-limit) |
| **Conditional edge** | An edge followed only if a condition holds on the parent's response | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Constrained decoding** | Generation restricted so the output matches a schema; the mechanism behind structured outputs | [07 §7.6](07-agents-that-build-workflows.md#76-why-not-generate-the-whole-definition-at-once) |
| **Curated tool** | A tool shaped like the agent's task, rather than a wrapper around one API endpoint | [07 §7.4](07-agents-that-build-workflows.md#74-the-harness-what-the-agent-can-touch) |
| **`CURLOPT_OPENSOCKETFUNCTION`** | libcurl callback replacing `socket()`, given the resolved peer address; return `CURL_SOCKET_BAD` to refuse. Fires before connect, per hop | [05 §5.5](05-calling-urls-you-dont-control.md#55-refusing-non-public-addresses-at-dial-time), [05 §5.6](05-calling-urls-you-dont-control.md#56-where-each-stack-lets-you-see-the-address) |
| **`CURLOPT_PREREQFUNCTION`** | libcurl callback after connect, before the request, given the primary IP; refuses after connecting rather than before | [05 §5.6](05-calling-urls-you-dont-control.md#56-where-each-stack-lets-you-see-the-address) |
| **DAG** | Directed acyclic graph: edges have a direction and no path returns to its start. Airflow now spells its workflows "Dag" | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Data document** | The JSON a run builds up and mappings are evaluated against: the run's input plus each finished ancestor's status, headers and body, keyed by step id | [01 §1.5](01-foundations.md#15-the-running-example-end-to-end), [03](03-mapping-data-between-steps.md) |
| **Declaration order** | The order steps appear in the definition's `steps` array; used to break ties in the ready set | [02 §2.4](02-graphs-and-scheduling.md#24-the-ready-set-and-a-concurrency-limit) |
| **Default operator (`?:`)** | Falls back whenever the left side is falsy, including `0`, `false` and `""` | [03](03-mapping-data-between-steps.md) |
| **Delay-seconds** | The number-of-seconds form of `Retry-After`, `1*DIGIT`, measured from when the response arrived | [04 §4.4](04-retries-idempotency-and-durability.md#44-honouring-retry-after) |
| **Deterministic grader** | An eval check with no model in it: same input, same verdict; fast and free | [07 §7.10](07-agents-that-build-workflows.md#710-evals-knowing-the-builder-works) |
| **Deterministic run** | Same definition, input and upstream responses give the same result. Holds for what steps see and send; not for timing, or for what cancellation catches | [02 §2.9](02-graphs-and-scheduling.md#29-deterministic-ordering) |
| **Dev/test override** | An explicit, narrow allow of exact `ip:port` pairs for local work, set only from trusted config, off by default | [05 §5.9](05-calling-urls-you-dont-control.md#59-the-devtest-override-must-be-explicit) |
| **Dial-time check** | Validating the actual IP the socket is about to connect to, on every connection and redirect hop; the one SSRF check that holds | [05 §5.3](05-calling-urls-you-dont-control.md#53-the-fix-check-the-resolved-ip-at-connection-time) |
| **DNS pinning** | Caching a name's first resolution to resist rebinding; a browser-era defence that doesn't rescue a server-side client | [05 §5.2](05-calling-urls-you-dont-control.md#52-why-validating-the-url-string-doesnt-work) |
| **DNS rebinding** | Attacker-controlled name that resolves public at check time and private at dial time, so a validated name still reaches a private IP | [05 §5.2](05-calling-urls-you-dont-control.md#52-why-validating-the-url-string-doesnt-work) |
| **Draft** | A workflow version being edited; it never runs for real | [07](07-agents-that-build-workflows.md) |
| **Drift check** | A CI step that regenerates code from the description and fails if anything changed | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **Dry-run** | Running a draft with safe steps sent and unsafe ones rendered rather than sent | [07 §7.7](07-agents-that-build-workflows.md#77-dry-run-the-agents-ground-truth) |
| **Durable execution** | A library or service that records a workflow's progress and resumes it after a crash; makes recording exactly-once, not side effects | [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **EARS** | Easy Approach to Requirements Syntax: "WHEN [condition/event] THE SYSTEM SHALL [expected behavior]" | [07 §7.8](07-agents-that-build-workflows.md#78-specs-writing-down-what-to-build) |
| **Edge** | "B depends on A", optionally with a condition | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Egress proxy** | A hardened out-of-process control (e.g. Smokescreen) that resolves and filters all outbound traffic independently of the engine | [05 §5.10](05-calling-urls-you-dont-control.md#510-defence-in-depth-an-egress-control-outside-the-engine) |
| **Entity tag (`ETag`)** | An opaque label for one version of a resource's representation; strong by default, weak with `W/` | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **`errors` extension** | The list of `detail` and `pointer` pairs in RFC 9457's validation example; an example extension, not a standard member | [06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457) |
| **`$eval`** | JSONata function that parses and evaluates a string as an expression; turns data into code, so treat it as dangerous in a mapping | [03](03-mapping-data-between-steps.md) |
| **Evaluator** | The library that runs an expression, such as jsonata-js or gnata; different evaluators can give different results | [03](03-mapping-data-between-steps.md) |
| **Evaluator parity** | Two evaluators giving the same result for the same expression and data; what a preview silently depends on | [03](03-mapping-data-between-steps.md) |
| **Event History** | Temporal's append-only log of a workflow execution's events, persisted by the Temporal Service and replayed to rebuild state | [01 §1.6](01-foundations.md#16-the-landscape), [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **Excessive agency** | OWASP LLM03:2026 (LLM06:2025): damage done through an LLM's actions, rooted in excessive functionality, permissions or autonomy | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Express workflow** | Step Functions type that runs up to five minutes without persisting state between transitions; the synchronous variant returns the result to the caller | [01 §1.4](01-foundations.md#14-job-style-and-request-style) |
| **Expression injection** | Building expression source from user input or upstream data, so data becomes code; avoided by passing data as input or bindings. CVE-2025-68613 is the worked example | [03](03-mapping-data-between-steps.md), [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Expression language** | A small language for computing values from data, stored as text in a definition rather than deployed as code; JSONata is one | [03](03-mapping-data-between-steps.md) |
| **Extension member** | A problem-details member defined by a problem type; clients must ignore ones they don't recognise | [06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457) |
| **External Storage** | Temporal's built-in claim check for large payloads; public preview as of October 2026 | [04 §4.7](04-retries-idempotency-and-durability.md#47-large-payloads-and-the-claim-check) |
| **Fail-fast** | Failure policy that cancels the run on the first failure: running steps are cancelled, unstarted ones never start | [02 §2.7](02-graphs-and-scheduling.md#27-failure-policy-fail-fast-or-carry-on) |
| **Fan-out, fan-in** | One step feeding several; several feeding one | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Finished ancestors rule** | A step's mapping sees only ancestors that have finished, so what it sees doesn't depend on timing | [01 §1.5](01-foundations.md#15-the-running-example-end-to-end) |
| **Fragment form** | A JSON Pointer written as a URI fragment, `#/age`, with percent-encoding (RFC 6901 §6); this course uses the plain form in `pointer` fields | [06 §6.6](06-describing-apis.md#66-json-pointer-rfc-6901) |
| **Global unicast (`2000::/3`)** | The only IPv6 range IANA assigns as globally routable; refuse everything outside it, then special-purpose blocks within it | [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **Globally Reachable** | The IANA special-purpose registries' column ([RFC 8190](https://www.rfc-editor.org/rfc/rfc8190.txt)): whether a block is forwardable beyond one administrative domain; the registry's own public/not-public answer | [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **gnata** | Go implementation of JSONata 2.x, with RE2 regexes, opt-in guardrails and a WebAssembly build | [03](03-mapping-data-between-steps.md) |
| **Guardrail** | A limit an evaluator enforces on an expression someone else wrote: depth (`D1011`), time (`D1012`) or sequence length (`D2015`) | [03](03-mapping-data-between-steps.md) |
| **Handle** | An id a stateless MCP server mints and takes back as a tool argument, such as a draft id; "a name, not a capability" | [07 §7.4](07-agents-that-build-workflows.md#74-the-harness-what-the-agent-can-touch) |
| **Host, client, server (MCP)** | The LLM application; the connector inside it; the service offering tools, resources and prompts | [07 §7.4](07-agents-that-build-workflows.md#74-the-harness-what-the-agent-can-touch) |
| **HTTP-date** | A timestamp in one of RFC 9110's three formats; recipients must accept all three, senders must use IMF-fixdate | [04 §4.4](04-retries-idempotency-and-durability.md#44-honouring-retry-after) |
| **I-JSON** | RFC 7493's interoperable profile of JSON; recommends sending numbers beyond double precision, such as 64-bit ids, as strings | [03](03-mapping-data-between-steps.md) |
| **Idempotency fingerprint** | The draft's name for a digest of the request a server stores beside the key, to spot a key reused with a different body | [04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key) |
| **`Idempotency-Key`** | Request header carrying a client-chosen key the server deduplicates on; an expired IETF Internet-Draft, not a standard | [04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key) |
| **`If-Match`** | Request header that makes a method conditional on the resource's current entity tag, compared strongly | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **IMDSv2** | AWS metadata hardening (session `PUT` + token header); protects only when the SSRF "merely allows the attacker to control the URL", not when it can set headers | [05 §5.10](05-calling-urls-you-dont-control.md#510-defence-in-depth-an-egress-control-outside-the-engine) |
| **IMF-fixdate** | The preferred HTTP-date format, `Sun, 06 Nov 1994 08:49:37 GMT` | [04 §4.4](04-retries-idempotency-and-durability.md#44-honouring-retry-after) |
| **In-degree** | A step's number of parents not yet placed (Kahn) or resolved (scheduler) | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **Indirect prompt injection** | Injection through content the model reads from elsewhere: a web page, a document, an API response | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Instance metadata service** | A cloud endpoint at a fixed link-local address serving instance data and temporary role credentials; the prime SSRF target | [05 §5.1](05-calling-urls-you-dont-control.md#51-the-problem) |
| **IPv4-mapped IPv6** | `::ffff:0:0/96`; `::ffff:127.0.0.1` *is* `127.0.0.1`. Unmap before any address check | [05 §5.2](05-calling-urls-you-dont-control.md#52-why-validating-the-url-string-doesnt-work), [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **JMESPath** | A JSON query language that can build objects but has a fixed set of functions | [03](03-mapping-data-between-steps.md) |
| **Job-style engine** | Triggered, runs for seconds to days with nobody waiting on a connection, and stores run state durably between steps | [01 §1.4](01-foundations.md#14-job-style-and-request-style) |
| **jq** | A JSON processing language with recursive functions and Oniguruma regexes | [03](03-mapping-data-between-steps.md) |
| **JSON Merge Patch** | A partial copy of a document in which `null` means remove (RFC 7396); can't set `null` or edit part of an array | [06 §6.7](06-describing-apis.md#67-json-patch-rfc-6902-for-partial-edits) |
| **JSON Patch** | A list of `add`, `remove`, `replace`, `move`, `copy` and `test` operations applied atomically (RFC 6902) | [06 §6.7](06-describing-apis.md#67-json-patch-rfc-6902-for-partial-edits) |
| **JSON Pointer** | A string naming one value in a JSON document, such as `/steps/2/id`: `/`-separated tokens with `~0` for `~` and `~1` for `/` (RFC 6901). JSON Patch uses it for `path` | [06 §6.6](06-describing-apis.md#66-json-pointer-rfc-6901), [07 §7.5](07-agents-that-build-workflows.md#75-errors-the-agent-can-act-on) |
| **JSON Schema 2020-12** | The schema language whose superset is OpenAPI 3.1's Schema Object | [06 §6.2](06-describing-apis.md#62-openapi-31-and-why-json-schema-matters) |
| **JSONata** | A JSON query and transformation language: a reference implementation plus ports that agree on a shared test suite and differ outside it | [03](03-mapping-data-between-steps.md) |
| **jsonata-core** | Rust implementation of JSONata, with guardrails and a C ABI | [03](03-mapping-data-between-steps.md) |
| **jsonata-cpp** | C++ implementation of JSONata on nlohmann/json, with timeout and stack guardrails and a pluggable regex engine | [03](03-mapping-data-between-steps.md) |
| **jsonata-js** | The JavaScript reference implementation of JSONata, published on npm as `jsonata` | [03](03-mapping-data-between-steps.md) |
| **JSONPath** | RFC 9535's syntax for selecting and extracting values from JSON; it can't build new documents | [03](03-mapping-data-between-steps.md) |
| **Kahn's algorithm** | Topological sort that repeatedly places a step with no unplaced parents; leftovers are on or behind a cycle. Kahn, 1962 | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **Lease** | An owner and a heartbeat on a run, so another instance can tell a dead owner from a live one and take the run over | [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **Lethal trifecta** | Simon Willison's name for private data, untrusted content and external communication in one agent | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Link-local address** | `169.254.0.0/16` (incl. `169.254.169.254`) / `fe80::/10`: reachable only on the local link, never routed; not public | [05 §5.1](05-calling-urls-you-dont-control.md#51-the-problem), [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **Lost update** | Two clients read, modify and write the same resource, and the second write silently erases the first | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **Mapping** | An expression that builds a step's request, or the run's output, from the data document | [01 §1.3](01-foundations.md#13-vocabulary) |
| **MCP** | Model Context Protocol: an open JSON-RPC protocol through which agents discover and call tools. Revision 2026-07-28 made it stateless | [07 §7.4](07-agents-that-build-workflows.md#74-the-harness-what-the-agent-can-touch) |
| **Model-free runtime** | An engine that runs a published plan with no model in the loop, so run-time data can't steer it | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **[NEEDS CLARIFICATION]** | Spec Kit's marker for an open question in a spec; at most three per spec | [07 §7.8](07-agents-that-build-workflows.md#78-specs-writing-down-what-to-build) |
| **`net.BlockList`** | Node deny-list of addresses/ranges, checked in the socket's connect after lookup; silently ignored by `tls.connect` | [05 §5.5](05-calling-urls-you-dont-control.md#55-refusing-non-public-addresses-at-dial-time), [05 §5.6](05-calling-urls-you-dont-control.md#56-where-each-stack-lets-you-see-the-address) |
| **`net.Dialer.Control`** | Go hook called after socket creation, before connect, with the resolved address; return an error to refuse. The dial-time check | [05 §5.5](05-calling-urls-you-dont-control.md#55-refusing-non-public-addresses-at-dial-time) |
| **`none_failed_min_one_success`** | Rule: run if no parent failed and at least one edge was taken; the usual join after a branch | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Not-taken edge** | An edge whose parent was skipped, or whose condition was false | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Nothing (undefined)** | JSONata's result for a missing path; not `null`, and it drops the key from any object being built | [03](03-mapping-data-between-steps.md) |
| **Object constructor** | `{ "key": expr, ... }` in an expression; builds a new object, leaving out keys whose value is nothing | [03](03-mapping-data-between-steps.md) |
| **On a cycle** | A step from which following edges leads back to itself | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **OpenAPI description** | A JSON or YAML document of an API's paths, operations, parameters, bodies and responses, with schemas | [06 §6.2](06-describing-apis.md#62-openapi-31-and-why-json-schema-matters) |
| **openapi-fetch** | A typed `fetch` client driven by openapi-typescript's types; returns `data`, `error` and `response` rather than throwing | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **openapi-typescript** | Generator that turns an OpenAPI description into TypeScript types, with no runtime code | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **OpenFlow** | Windmill's JSON/YAML format for flows, published as an open standard with Windmill as its reference implementation | [01 §1.6](01-foundations.md#16-the-landscape) |
| **Optimistic concurrency** | Letting clients edit without locks and refusing a write whose base version is no longer current | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **Orchestration engine** | A program that runs workflow definitions: validates them, schedules steps, maps data between them, applies failure policy and records each run | [01 §1.2](01-foundations.md#12-the-chain-as-data) |
| **Outcome grading** | Judging what an agent produced rather than the sequence of tool calls it made | [07 §7.10](07-agents-that-build-workflows.md#710-evals-knowing-the-builder-works) |
| **Outcome unknown** | The state of a step whose non-idempotent call may or may not have taken effect; distinct from failed | [04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key) |
| **Parent, child** | In the edge A → B, A is B's parent and B is A's child. Airflow says "upstream" and "downstream" task | [01 §1.3](01-foundations.md#13-vocabulary) |
| **pass@k** | Probability that at least one of k trials succeeds | [07 §7.10](07-agents-that-build-workflows.md#710-evals-knowing-the-builder-works) |
| **pass^k** | Probability that all k trials succeed | [07 §7.10](07-agents-that-build-workflows.md#710-evals-knowing-the-builder-works) |
| **Path** | Field names joined by `.`; it visits every item of any array it passes through | [03](03-mapping-data-between-steps.md) |
| **Pending** | Step status: waiting for its parents to resolve | [02 §2.3](02-graphs-and-scheduling.md#23-the-scheduler-loop) |
| **Plan-then-execute** | The agent fixes its list of actions before it reads untrusted data, so the data can't add actions, though it can still change their inputs | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Portable test suite** | jsonata-js's JSON test cases (1,686 at v2.2.2) that ports run; its JavaScript-only tests don't travel | [03](03-mapping-data-between-steps.md) |
| **Predicate** | `[condition]` after a path step; keeps only the items for which the condition is true | [03](03-mapping-data-between-steps.md) |
| **Preview** | Evaluating a mapping against a sample run in the editor, to show the request before anything is sent | [03](03-mapping-data-between-steps.md) |
| **Problem details** | RFC 9457's JSON error shape, `application/problem+json`, with `type`, `status`, `title`, `detail`, `instance` and extensions; its validation example adds an `errors` list of `detail` and `pointer` | [06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457), [07 §7.5](07-agents-that-build-workflows.md#75-errors-the-agent-can-act-on) |
| **Problem type** | The URI in `type` that identifies a kind of problem and defines its extensions | [06 §6.5](06-describing-apis.md#65-errors-as-problem-details-rfc-9457) |
| **Prompt injection** | Text a model treats as instructions when it was meant as data; OWASP LLM01 in both the 2025 and 2026 lists | [07 §7.9](07-agents-that-build-workflows.md#79-safety) |
| **Protocol error (MCP)** | A JSON-RPC error for an unknown tool, a malformed request or a server fault; less likely to reach the model | [07 §7.5](07-agents-that-build-workflows.md#75-errors-the-agent-can-act-on) |
| **Published** | The workflow version that runs for real | [07](07-agents-that-build-workflows.md) |
| **RE2** | A regex engine that guarantees time linear in the input by leaving out backreferences and look-around; Go's `regexp` uses its syntax | [03](03-mapping-data-between-steps.md) |
| **Ready, ready set** | A step whose parents have all resolved and whose rule says run, waiting for a slot; the set of them, kept in declaration order | [02 §2.4](02-graphs-and-scheduling.md#24-the-ready-set-and-a-concurrency-limit) |
| **ReDoS** | Regular expression denial of service: input or a pattern chosen to make a backtracking engine take exponential time | [03](03-mapping-data-between-steps.md) |
| **Reference token** | One `/`-separated segment of a JSON Pointer: a member name or an array index | [06 §6.6](06-describing-apis.md#66-json-pointer-rfc-6901) |
| **`ReferencedPaths`** | gnata method listing the input paths an expression reads; lets an engine reject references to non-ancestors at save time. Static analysis, not a security boundary | [03](03-mapping-data-between-steps.md) |
| **`RegexEngine`** | jsonata-js option that replaces the regex constructor, for example with RE2 | [03](03-mapping-data-between-steps.md) |
| **Rendered request** | The exact request an unsafe step would have sent, returned by a dry-run instead of being sent | [07 §7.7](07-agents-that-build-workflows.md#77-dry-run-the-agents-ground-truth) |
| **Request validation middleware** | A layer that checks each request against the API description before the handler runs; generated servers don't do it themselves | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **Request-style composition** | Steps run inside one inbound request and must finish within the caller's deadline, with state in memory | [01 §1.4](01-foundations.md#14-job-style-and-request-style) |
| **`Resolve` / `dns_resolver`** | reqwest's resolved-address hook; runs for hostnames only — IP literals skip it, so it needs a literal pre-check | [05 §5.5](05-calling-urls-you-dont-control.md#55-refusing-non-public-addresses-at-dial-time), [05 §5.6](05-calling-urls-you-dont-control.md#56-where-each-stack-lets-you-see-the-address) |
| **Resolved** | A step that has reached a final status: succeeded, failed, skipped, upstream_failed or cancelled | [02 §2.3](02-graphs-and-scheduling.md#23-the-scheduler-loop) |
| **Resume / re-run** | Resuming continues a run under its run id, so derived keys repeat; re-running starts a new run with new keys | [04 §4.3](04-retries-idempotency-and-durability.md#43-retrying-a-post-safely-idempotency-key) |
| **Retry policy** | Temporal's per-activity settings: initial interval, backoff coefficient, maximum interval, maximum attempts, non-retryable errors | [04 §4.5](04-retries-idempotency-and-durability.md#45-one-attempt-or-the-whole-sequence) |
| **`Retry-After`** | Response header saying how long to wait before trying again, as delay-seconds or an HTTP-date (RFC 9110 §10.2.3) | [04 §4.4](04-retries-idempotency-and-durability.md#44-honouring-retry-after) |
| **Rule** | What a step does once its parents resolve: run, skip, or become upstream-failed. Airflow calls these trigger rules | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Run** | One execution of a workflow with one input | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Run record** | What's known about a run: each step's status, response, error and timing | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Safe method** | RFC 9110: a method whose semantics are read-only (GET, HEAD, OPTIONS, TRACE); a promise about the request, not about the server | [07 §7.7](07-agents-that-build-workflows.md#77-dry-run-the-agents-ground-truth) |
| **Saga** | A long-lived transaction written as a sequence of steps, each with a compensation (Garcia-Molina and Salem, 1987) | [04 §4.2](04-retries-idempotency-and-durability.md#42-retry-the-step-not-the-run) |
| **Schedule-To-Close** | Temporal's bound on a whole activity execution, every attempt and wait included; default unlimited | [04 §4.5](04-retries-idempotency-and-durability.md#45-one-attempt-or-the-whole-sequence) |
| **Schedule-To-Start** | Temporal's bound on how long one attempt may wait in a task queue; never retried | [04 §4.5](04-retries-idempotency-and-durability.md#45-one-attempt-or-the-whole-sequence) |
| **Scheduler** | The part of an engine that decides which step starts next. Kestra calls this the Executor, and uses "Scheduler" for triggers | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Sequence** | JSONata's collection of results; a sequence of one item is the item itself | [03](03-mapping-data-between-steps.md) |
| **Sequence guardrail** | `sequence` / `WithSequence`: error `D2015` past a sequence length. Bounds length, not memory | [03](03-mapping-data-between-steps.md) |
| **Sequential proxy** | KrakenD's mode for chaining backends, passing earlier response fields into later URLs; its docs call it a last resort | [01 §1.6](01-foundations.md#16-the-landscape) |
| **Shared Address Space (CGNAT)** | `100.64.0.0/10` ([RFC 6598](https://www.rfc-editor.org/rfc/rfc6598.txt)): carrier-grade NAT, private but distinct from RFC 1918 | [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **Single owner** | Design where one goroutine holds all of a run's state and step goroutines only report results to it on a channel | [02 §2.2](02-graphs-and-scheduling.md#22-why-errgroup-isnt-enough) |
| **Singleton equivalence** | JSONata treats a value and a one-item array as the same, so one match gives a value, not an array | [03](03-mapping-data-between-steps.md) |
| **Skip cascade** | A skipped step's edges become not taken, so `all_success` children skip too, and so on down the graph | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Skipped** | Step status: a condition or rule chose not to run it. A normal outcome, unlike upstream_failed | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Spec-driven development** | Writing a spec before generating code with AI; Böckeler separates spec-first, spec-anchored and spec-as-source | [07 §7.8](07-agents-that-build-workflows.md#78-specs-writing-down-what-to-build) |
| **Spec-first** | Writing the API description first and generating server interfaces and client types from it | [06 §6.3](06-describing-apis.md#63-spec-first-or-code-first) |
| **SSRF** | Server-side request forgery: making a server issue a request, from its network, that the requester couldn't issue themselves. [CWE-918](https://cwe.mitre.org/data/definitions/918) | [05 §5.1](05-calling-urls-you-dont-control.md#51-the-problem) |
| **Stack guardrail** | `stack` / `WithStack`: error `D1011` past a depth. jsonata-js counts eval-apply cycles, gnata counts lambda calls | [03](03-mapping-data-between-steps.md) |
| **Standard workflow** | Step Functions type that runs up to a year and persists state between transitions | [01 §1.4](01-foundations.md#14-job-style-and-request-style) |
| **Start-To-Close** | Temporal's bound on a single attempt; how a crashed worker is detected | [04 §4.5](04-retries-idempotency-and-durability.md#45-one-attempt-or-the-whole-sequence) |
| **Step** | One unit of work, here almost always one HTTP call. Graph theory's node | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Strict server** | oapi-codegen's generated interface with one method per operation, a typed request and typed per-status responses | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **Strong / weak comparison** | Strong: both tags strong and identical. Weak: identical ignoring `W/`. `If-Match` requires strong | [06 §6.8](06-describing-apis.md#68-optimistic-concurrency-etag-if-match-and-412) |
| **Structured outputs** | Claude API feature that constrains a response to a JSON Schema; recursive schemas aren't supported | [07 §7.6](07-agents-that-build-workflows.md#76-why-not-generate-the-whole-definition-at-once) |
| **Taken edge** | An edge whose parent succeeded and whose condition, if any, held | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Tarjan's algorithm** | Linear-time strongly connected components; groups the steps on cycles into their separate cycles. Tarjan, 1972 | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **`test` operation** | A JSON Patch operation that fails the whole patch unless a value is as expected | [06 §6.7](06-describing-apis.md#67-json-patch-rfc-6902-for-partial-edits) |
| **Time-of-check / time-of-use (TOCTOU)** | The gap between inspecting a value and acting on it; every string-based SSRF defeat is one | [05 §5.2](05-calling-urls-you-dont-control.md#52-why-validating-the-url-string-doesnt-work) |
| **Timeout guardrail** | `timeout` / `WithTimeout`: error `D1012` past a time. Checked between steps, so it can't interrupt one regex match or built-in | [03](03-mapping-data-between-steps.md) |
| **Tool directive** | A `tool` line in `go.mod` (Go 1.24) that pins a command-line tool such as a generator as a dependency | [06 §6.4](06-describing-apis.md#64-what-codegen-produces-one-endpoint-both-sides) |
| **Tool execution error (MCP)** | A tool result with `isError: true`, carrying feedback the model can use to correct itself | [07 §7.5](07-agents-that-build-workflows.md#75-errors-the-agent-can-act-on) |
| **Topological order** | An order of steps in which every parent comes before its children | [02 §2.1](02-graphs-and-scheduling.md#21-validating-the-graph) |
| **Trigger** | What starts a run: an inbound request, a schedule, a webhook, a queue message | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Turing-complete** | Able to express any computation, including loops that never end; JSONata is, CEL is designed not to be | [03](03-mapping-data-between-steps.md) |
| **Type array** | OpenAPI 3.1's way to allow null, `type: [string, "null"]`, replacing 3.0's `nullable: true` | [06 §6.2](06-describing-apis.md#62-openapi-31-and-why-json-schema-matters) |
| **Unique-local address (ULA)** | `fc00::/7` ([RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193)): IPv6 private space; not globally reachable | [05 §5.4](05-calling-urls-you-dont-control.md#54-which-ranges-to-refuse) |
| **Upstream** | In this course, the API a step calls. In Airflow, a task's dependencies | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Upstream failed** | Step status: a parent it needed failed, so it never ran. Airflow's `upstream_failed`; GitHub Actions calls this "skipped" | [02 §2.6](02-graphs-and-scheduling.md#26-conditional-edges-and-skip-cascades) |
| **Workflow** | A definition: steps and the dependencies between them | [01 §1.3](01-foundations.md#13-vocabulary) |
| **Workflow definition** | A workflow written down as data (steps, dependencies, mappings, policies) that an engine executes | [01 §1.2](01-foundations.md#12-the-chain-as-data) |
| **Workflow determinism** | The rule that orchestration code, given the same inputs and recorded results, makes the same calls in the same order, so it can be replayed or resumed | [04 §4.6](04-retries-idempotency-and-durability.md#46-how-much-of-a-run-survives-a-crash) |
| **XCom** | Airflow's mechanism for passing small values between tasks, stored in its database by default | [01 §1.6](01-foundations.md#16-the-landscape) |
