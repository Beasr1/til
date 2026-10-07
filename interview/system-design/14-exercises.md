# 14. Exercises & answers

Every "Check yourself" question, with worked answers. Commit to an answer before reading
mine. In this subject the reasoning is the answer: if you got the right conclusion by a
different route, check the route holds up.

---

## File 1 — Foundations

**1. Average 40 ms, p99 2 s, "we're fine".**
Look at the distribution: p95, p99, p99.9, and *which* requests are slow. One in a hundred
requests taking two seconds is a lot of unhappy users at scale, and the slow ones are often
the important ones (big accounts with the most data, checkout pages with the most calls).
If a page makes several such requests, the chance that at least one is slow is much higher
than 1% (§1.2, *The Tail at Scale*). The average hides all of that.

**2. Three services in series at 99.9%; then a redundant product service.**
`0.999³ ≈ 0.997`, about 99.7%. With two independent product-service replicas, that tier's
availability becomes `1 − 0.001² ≈ 99.9999%`, so the path is about `0.999² × 0.999999 ≈ 99.8%`:
the gateway and auth service now dominate. "Independent" is doing a lot of work because
replicas share deploys, configuration, a region, a database and bugs. A bad release takes
both down together, and then the redundancy buys nothing.

**3. Little's Law: 500 QPS, 200 ms → 2 s.**
Before: `500 × 0.2 = 100` in flight. After: `500 × 2 = 1,000` in flight, ten times as many,
if arrivals keep coming at the same rate. What runs out first depends on the server: the
connection pool to the dependency (requests queue waiting for a connection), worker threads
in thread-per-request servers, then memory and file descriptors. In Go, goroutines are cheap,
so it's usually the outbound pool or memory, which is why a slow dependency needs a timeout
([file 08 §8.2](08-reliability-patterns.md#82-timeouts-and-deadlines)).

**4. 99.9% over 30 days; 50 minutes down by day 20.**
The budget is 0.1% of 43,200 minutes, 43.2 minutes. You've spent 50: the budget's gone, with
ten days left. The error-budget policy says freeze risky change: postpone the migration, or
make it safer (smaller steps, a rollback plan, off-peak), until the window resets.

**5. 50 shards, p99 100 ms, p50 5 ms.**
`1 − 0.99⁵⁰ ≈ 0.39`. About 39% of page loads wait for at least one 100 ms+ response, so the
page's typical latency is set by the shards' *tail*, not their median. Optimise variance:
hedged requests, fewer shards per page, timeouts with partial results. Making the median
faster does little.

**6. "Design Instagram", first three things.**
Something like: "Let me pin down scope: are we designing posting photos and the home feed,
or also stories, DMs, search? I'll assume posting and the feed. Then I'd like rough scale:
daily users and posts per day. Then I'll estimate, sketch the API, and pick one hard part to
go deep on, probably feed generation." The two non-functional requirements to pin first:
scale with its read-to-write ratio (it decides fan-out strategy and storage), and how fresh
the feed must be (seconds of delay allowed decides whether fan-out can be asynchronous).
Photo storage durability is a close third.

---

## File 2 — Estimation

**1. 250 million requests a day.**
`250,000,000 / 86,400 ≈ 2,900` QPS average. Peak: I'd plan for 3× (~9,000) and say so. The
factor depends on traffic shape: a user base in one time zone (most of India) has a sharper
daily peak than a global one; event-driven products (sales, sports) can spike far above any
daily pattern, and need load shedding as well as capacity.

**2. 30 sequential cache calls in-region.**
At 0.25–0.5 ms per round trip, 7.5–15 ms of pure network wait before any work. Batch them
(one `MGET`, or pipelining), run independent ones in parallel, or keep a small in-process
cache for the hottest keys.

**3. 2 KB URLs.**
Per month: `100M × ~2 KB = 200 GB`; ten years: 24 TB; replicated: ~72 TB. QPS is unchanged.
It moves the database from "one big machine" towards "a few shards or tiered storage", and
the cache holds a quarter as many links per GB. Not a different architecture, but the
storage plan changes, which is exactly what the estimate is for.

**4. "Colin Scott's page says 16 µs for an SSD read in 2026."**
That page's numbers are extrapolated from trend assumptions anchored around 2012–2014, not
measured on 2026 hardware; its source code fixes the SSD figure after 2014 rather than
measuring it. A measured source (napkin-math, March 2026) gives 100 µs for a random 8 KiB SSD
read. Answer with the order of magnitude, tens to a hundred microseconds depending on device
and load, and say where each number comes from.

**5. The chat assumption that matters most.**
Messages per user per day, or the fraction of users online at peak. Messages per day drives
write QPS and storage linearly: 10× fewer and one sharded relational cluster might do; 10×
more and storage tiering becomes urgent. Concurrent online users drives the connection tier,
which is the one stateful, hard-to-scale part. Either is defensible if argued.

**6. Photo app: 10 million uploads a day at 2 MB.**
`10M × 2 MB = 20 TB/day`, about 7.3 PB a year, ~22 PB with three replicas (object stores
often use erasure coding, which costs less than 3×). Photos go in object storage served
through a CDN; the database holds metadata (owner, key, size, timestamps), a few hundred
bytes per photo. Blobs in the database make backups, replication and caching all worse, and
gain nothing: nobody queries inside a JPEG.

---

## File 3 — Scaling a service

**1. Cart in a Go map, three instances, round robin.**
Each request lands on a different instance, so the cart seems to empty and refill at random:
add an item (instance 1), view the cart (instance 2, empty), add another (instance 3). A
deploy or crash loses carts outright. Fixes: store carts in a shared store (Redis or the
database), or, for small carts, in a signed client-side token. Sticky sessions hide the
symptom until an instance goes away.

**2. HTTP/2 clients behind an L4 balancer.**
The balancer assigned each of the 20 connections to a backend once. The heavy client's
connection, and all its requests, sit on one backend: one CPU line near the top, four much
lower, regardless of algorithm. Fix: balance per request with an HTTP/2-aware L7 proxy
(Envoy and similar), or client-side balancing across several connections.

**3. Why two random choices work.**
The bad outcome in random assignment is landing on an overloaded server. Sampling two and
taking the less loaded makes that very unlikely, because both must be overloaded, and
Mitzenmacher showed the improvement over one choice is exponential while a third choice adds
only a constant factor. It needs no global view: each balancer looks at two servers' load
rather than all of them, so many independent balancers don't need to coordinate.

**4. Liveness checks the database; the database fails over for 30 s.**
Every pod's liveness probe fails; after the failure threshold Kubernetes restarts every pod.
Restarting pods serve nothing, so the outage is now the service's too, not only the
database's. When the database returns, all pods start at once: cold caches, connection-pool
storms against a database that has just failed over. The fix is a liveness check that tests
only the process itself, with dependency checks in readiness (if anywhere).

**5. 4 → 40 instances and slower.**
Likely: the database connection count jumped tenfold, so the database spends memory and CPU
on connections, or clients wait for connection slots; or a shared resource (a cache node, a
lock, a hot row) is now contended ten times harder. Tell them apart by looking at the
database: connection count and wait events versus lock waits and one hot key's traffic.
Per-instance in-memory caches also get colder, since each sees a tenth of the traffic.

---

## File 4 — Caching

**1. Delete versus set on write.**
Two writers: A writes v1 to the database, B writes v2 after it. Their cache updates race:
B's `set(v2)` arrives, then A's delayed `set(v1)`. The database says v2, the cache says v1,
until the TTL. With deletes, both writers delete; the order of two deletes doesn't matter;
the next reader loads v2. Deletes are idempotent and commute; sets of different values
don't.

**2. The stale-set window.**
Until the TTL expires, or another write deletes the key. Shorten it with a short TTL; remove
it with leases (the cache refuses a `set` whose lease was invalidated by an intervening
delete), or by a second, delayed delete after each write, which catches sets that raced the
first.

**3. Locking Put but not Get in an LRU.**
`Get` moves the node to the front of the list: it writes. Two concurrent `Get`s, or a `Get`
racing a `Put`'s eviction, corrupt the list's pointers (a node unlinked twice, a cycle), or
read a node that's being evicted. In Go it's also a data race on the map, which the runtime
can detect and crash on ("concurrent map read and map write"). Every operation needs the
lock.

**4. Redis default settings, memory full.**
The default `maxmemory-policy` is `noeviction`, so writes fail with an out-of-memory error
once the limit is hit; with no `maxmemory` set at all on a 64-bit system, Redis grows until
the operating system intervenes. Set `maxmemory` and a policy such as `allkeys-lru` or
`allkeys-lfu` for a cache.

**5. 98% hit rate, database provisioned for 3× normal; cache restarts.**
Normally the database sees 2% of reads; its capacity is 3 × 2% = 6% of total read traffic.
Cold, it sees 100%: about 17× its capacity, so it falls over and the cache can't refill.
Prevention: request coalescing or leases so each key is loaded once; warming the cache, or
restarting cache nodes one at a time; shedding load at the database (or the app) so it
serves some requests rather than none.

**6. Name plus prices in one response.**
Split it: a public prices endpoint (`Cache-Control: public, max-age=…, s-maxage=…`, maybe
`stale-while-revalidate`) the CDN can cache for everyone, and a personalised endpoint for the
user's details marked `private` (or `no-store`). Or cache the page shell and fetch the
personalised fragment separately. Never let a CDN cache a response containing one user's
data.

---

## File 5 — Data storage

**1. Read replicas for a write-heavy workload.**
Every replica must apply every write, so replicas add read capacity and no write capacity;
they even add a little leader work streaming the log. What helps writes: a bigger leader,
batching writes, removing unnecessary writes or indexes, moving some write traffic to
another store, or sharding.

**2. "No orders yet" after placing one.**
Replication lag: the read hit a follower that hadn't applied the write. Fix 1: read the
user's own recent data from the leader for a short time after they write. Cost: more leader
load, and you must track "recently wrote". Fix 2: return the write's log position to the
client and only read from followers that have reached it. Cost: complexity, and waits if
every follower lags.

**3. Orders sharded by `order_id`, queried by user.**
"This user's orders" must ask every shard and merge: scatter-gather, as slow as the slowest
shard, and more expensive as shards grow. Shard by `user_id` instead, and a user's orders
live together and come back in one query. That makes fetching one order by id harder (embed
the user or shard id in the order id, or keep a lookup), and very large customers become hot
shards.

**4. N = 3 quorums.**
Overlapping: any R + W > 3, so (1,3), (2,2), (3,1), (2,3), (3,2), (3,3). For write-heavy, W
= 1 with R = 3 makes writes cheapest, but any one node down makes strict reads fail (they
need all three); (2,2) tolerates one node down for both reads and writes, which is why it's
Dynamo's common setting. W = 1 also means an acknowledged write lives on one node until it
spreads.

**5. `% 10` → `% 9` after a node dies.**
A key keeps its node only when `h % 10 == h % 9`, which happens for about 1 in 10 keys. So
~90% of keys move and miss at once, and the database gets almost the full read load. With
consistent hashing, only the dead node's keys (about 10%) move, to their clockwise
neighbours.

**6. 3,755 versus 2,694 keys.**
Each node owns the arcs before its virtual-node positions, and with 100 random positions per
node, arc lengths vary; the sum over 100 arcs still differs between nodes by tens of
percent. FNV-1a's mixing on similar strings ("cache-a#17") adds to it. Reduce it with more
virtual nodes or a better-mixing hash (or bounded-load variants). Teams accept the residual
imbalance because the alternative costs memory and lookup time, and capacity headroom covers
it.

---

## File 6 — Consistency and CAP

**1. "An AP database for the cart."**
Precisely: "During a network partition, a replica that can't reach the others still accepts
cart reads and writes; reads may be stale and concurrent writes may conflict, and conflicts
are merged when the partition heals (for example, by taking the union of items added)."
That states the per-operation behaviour and the conflict policy, which "AP" leaves out.

**2. Postgres with async replicas: neither C nor A.**
Not CAP-consistent: a read from a lagging replica can return a value older than a completed
write, which linearizability forbids. Not CAP-available: during a partition, a replica cut
off from the leader can't accept writes (and the leader may be unreachable for clients on
the replica's side), so some non-failing nodes can't answer some requests.

**3. Useless but CAP-available, and useful but not.**
CAP-available and useless: a system that answers every request, eventually, after an hour;
CAP counts that. Meets a 99.9% SLO, not CAP-available: a single-leader database that refuses
writes on the minority side of a partition. Partitions are rare enough that it hits its SLO,
yet during one, non-failing nodes return errors, which CAP-availability forbids.

**4. Display name shows the old value.**
Read-your-writes. Ways to provide it: route that user's reads to the leader for a few
seconds after a write; have the client remember the write's position or version and read
only from replicas that have caught up; or have the client display its own write locally
(update the UI from the response rather than re-reading).

**5. A month with no partitions.**
Latency against consistency, every request: waiting for replicas (synchronous replication,
quorum reads, reading from the leader) to be consistent, or answering from local or lagging
copies to be fast. On a dashboard: write latency including replication, and replication lag
on the followers you read from.

**6. N=3, R=2, W=2, two concurrent writers.**
Both writes can succeed on different overlapping pairs: A on nodes 1 and 2, B on 2 and 3,
landing in different orders on node 2. A read may see A on one node and B on another; which
is "latest"? There's no single order, so the system resolves (last-write-wins by timestamp,
or keeps siblings). Linearizability needs every operation to take effect at one point in a
single global order; quorum overlap only guarantees a read *contacts* a node that saw a
completed write, not that the writes were ordered. Sloppy quorums weaken even the overlap.

---

## File 7 — Asynchronous work

**1. Ack before processing; daily crash.**
At-most-once: the message acknowledged just before a crash is never processed, so some
orders silently get no email or no warehouse notification. Fix: acknowledge after
processing (at-least-once). New problem: after a crash, the message being processed is
delivered again, so handlers must be idempotent.

**2. Idempotent "add ₹100 for event E".**
Store the event id in a `processed_events` table with a unique constraint, in the same
database transaction as the balance update: insert the id; if it conflicts, skip; otherwise
update the balance; commit. Same transaction, because otherwise a crash between "mark
processed" and "update balance" either loses the credit or applies it twice on redelivery.

**3. PAID (v3) after REFUNDED (v4).**
The final status is PAID, which is wrong: each write was idempotent, but the later-applied
one was older. Fix: store the version with the order and update only when the incoming
version is newer (`UPDATE … WHERE version < $new`), so v3 arriving after v4 is ignored.

**4. Dual-write failure orders.**
DB succeeds, publish fails: the order exists; the warehouse never hears of it; nothing ships
and nothing alerts. Publish succeeds, DB rolls back: the warehouse ships an order that
doesn't exist. With an outbox, the event row is written in the same transaction as the order,
so both exist or neither does; the relay publishes it afterwards, at least once, and the
warehouse deduplicates by event id.

**5. Lag grows; 4 instances, 4 partitions.**
A partition is read by one consumer in a group, so the fifth instance gets no partition and
sits idle. Options: make processing faster (batching, removing per-message round trips,
parallelism within a partition where order allows), or add partitions (which moves keys to
new partitions, so per-key order across the change needs care) and then instances.

**6. Email outage sends everything to the DLQ.**
The DLQ holds 20 minutes of perfectly good emails, mixed in with any real poison messages,
and someone has to replay them all. The worker should have classified the failure as
transient (timeouts, 5xx) and retried with backoff, or paused consumption until the provider
recovered, and kept the DLQ for permanent failures.

---

## File 8 — Reliability patterns

**1. Token bucket, capacity 10, 1/s, 15 at t = 0.**
Ten succeed (the full bucket), five are rejected. At t = 1, one token has refilled, so the
request succeeds; the same each second after. Every request succeeds from t = 1, because
the arrival rate equals the refill rate.

**2. Fixed window boundary burst.**
Send 100 at 09:00:59.5 (counted in the 09:00 window) and 100 at 09:01:00.0 (a fresh 09:01
window): 200 in half a second, both windows within limit. The sliding window counter at
09:01:00.0 weights the previous window's 100 by almost its full overlap, `100 × (1 − 0/60) =
100`, so the estimate is already at the limit and the second burst is rejected.

**3. JS bucket in Redis with awaits.**
Request 1 `await`s the read (1 token); while it waits, request 2 runs and also reads 1 token;
both compute "allowed" and write 0. The event loop guarantee covers only code between
awaits. Fix: one atomic operation in Redis, a Lua script (which Redis runs atomically) doing
refill, check and decrement together, or an atomic `INCR` for a fixed-window counter.

**4. Three layers, three attempts each.**
`3³ = 27` attempts at the database for one failure. With three *retries* (four attempts) per
layer, `4³ = 64`, which is the SRE book's figure. Keep retries at one layer, usually the one
closest to the failure that knows whether the operation is safe to repeat, and have the
others fail fast. Add a retry budget so retries stop when the dependency is broadly failing.

**5. 60 s outage: timeout only versus breaker.**
(a) Every call waits the full 10 s: at R requests per second, about 10R requests hold
threads and connections at any moment, and all 60R calls are made to a dependency that's
down. Users wait 10 s for an error. (b) After 5 failures (about 10 s in, depending on
concurrency) the breaker opens; for 30 s calls fail instantly, holding nothing; a half-open
trial fails and it opens again; when the dependency recovers, the next trial closes it. A
handful of calls are made instead of thousands, and users get an immediate fallback.

**6. Hedging reads versus "create order".**
A hedged read that runs twice just reads twice. A hedged "create order" that runs twice
creates two orders. It becomes safe when the operation is idempotent: both copies carry the
same idempotency key, and the server executes it once and returns the stored result to the
other.

---

## File 9 — API design

**1. Browser and bidirectional gRPC.**
Browsers don't expose HTTP/2 framing to JavaScript, so a native gRPC client can't run there;
gRPC-Web goes through a proxy and doesn't support client or bidirectional streaming. Use a
WebSocket (or SSE plus normal requests) between browser and an edge service, which talks
gRPC to the backend.

**2. Offset paging repeats a post.**
A new post was inserted at the top between page loads; every item shifted down one, so the
last item of page 1 became the first of page 2. A cursor endpoint returns "items older than
the last one you saw", by sort key with a unique tiebreaker, so insertions above the cursor
don't move anything below it.

**3. Same key, different amount.**
Reject the second request with an error saying the key was used with different parameters.
Charging ₹600 would let a key mean two operations; returning the ₹500 result would tell the
client its ₹600 payment succeeded when it didn't. An error exposes the client bug.

**4. Two requests, one key, 5 ms apart.**
Both try to insert the key row; the unique constraint lets exactly one succeed. The winner
processes the payment. The loser sees the conflict and finds the key "in progress", with no
stored response yet. It should return a `409 Conflict` (or similar) telling the client to
retry shortly, not process the payment, and not wait indefinitely.

**5. Rupees to paise without breaking v1.**
Add a new field (`total_paise`, or `total_amount` as a decimal string with a currency) and
keep `total` populated as before, so v1 apps keep working. Or introduce a new API version
that changes the field, while v1 clients keep receiving the old shape. Never change `total`'s
type or meaning in place.

**6. HTTP caching and GraphQL.**
REST reads are `GET`s on distinct URLs, which browsers and CDNs cache by URL with standard
headers. GraphQL typically sends every query as a `POST` to one URL, which shared caches
don't store, and two queries for overlapping data share nothing. Deployments use persisted
queries sent as `GET`s (hashable, cacheable), client-side normalised caches, and server-side
caching of resolvers or data loaders.

---

## File 10 — Worked designs

**1. Hash-prefix collisions.**
Two URLs whose hashes share the first 7 characters would map to one code. The write path
must insert with a unique constraint on the code; on conflict, check whether the existing row
is the same long URL (then return it) or a different one (then rehash, for example with a
salt or counter appended, and retry). Without the check, a later URL silently overwrites or
is lost.

**2. 301, then analytics and takedowns.**
Browsers and proxies may cache a 301 and stop asking you, so click counts undercount and a
disabled link keeps redirecting for anyone with it cached. Switch to 302 (or 307) for links
you need to count or control, and accept the extra load: it's the cache and redirect service
you already sized for.

**3. Limiter's Redis fails for two minutes.**
Fail-open: every request is allowed; abusive clients get through and the API is unprotected,
but legitimate traffic is unaffected. Fail-closed: every request is rejected with 429, which
is a two-minute API outage caused by the limiter. Fail-open suits a limiter that protects
capacity; fail-closed suits one that enforces a hard paid quota or protects a fragile
downstream.

**4. Phone reconnects to a different gateway.**
It reconnects, authenticates, and the presence map now points the user at the new gateway.
The client sends, per conversation, the last `seq` it has; the server returns everything
after it, in `seq` order, from the message store. Anything also redelivered from an in-flight
push is dropped by the client because it already has that `message_id`.

**5. 30 million followers, fan-out on write.**
30 million feed inserts for one post. At 1 million inserts a second, that's 30 seconds of the
whole fan-out fleet's capacity, and posts queued behind it wait up to that long. The hybrid
avoids it: store the celebrity's post once and merge it in at read time. Separately, fan-out
queues for large accounts keep them from delaying everyone else.

**6. The assumption that matters most.**
One example: the chat design assumes small groups. If groups had 100,000 members, each
message becomes 100,000 deliveries, per-conversation sequencing becomes a hot spot, and the
design shifts towards the feed's fan-out problem (channels read on demand, not pushed). Any
design is fine; the point is naming which requirement the architecture rests on.

---

## File 11 — LLD and machine coding

**1. Why `Lot` takes a `SlotFinder`.**
Requirement change: a new allocation rule (bikes in car slots, nearest to the entrance,
reserved slots) is a new type, with `Lot` unchanged. Testing: pass a fake finder that
returns a chosen slot, to test `Lot`'s bookkeeping without depending on allocation logic.

**2. "Bikes may use a car slot".**
A finder that searches for a free slot that fits the vehicle's type, and if none is found
and the vehicle is a bike, searches again for a free car slot. It relies on open/closed (new
behaviour as a new type) and dependency inversion (`Lot` depends on the interface). The
invariant (Liskov): it must return only a free slot the vehicle can physically use, or none.
`Lot` marks whatever it returns as occupied without checking.

**3. `Car`, `Bike`, `Truck` each with `Park`.**
Pricing lives in three places, so a tariff change is three edits that must stay consistent.
An electric car needs a fourth struct duplicating most of `Car`, and the "needs a charging
slot" rule ends up inside a vehicle rather than in allocation. The variation between vehicle
types was data (which slot, which rate), so it belongs in fields and in the strategies, not in
types with behaviour.

**4. Double-booking in the JavaScript version.**
Make `park` async and put an `await` between finding a slot and marking it occupied (for
example, to save a record to a database). Two parks can both find slot `L1-C1` free before
either marks it. In Go, two goroutines can interleave without any `await`, so the code needs
the mutex already, and has it.

**5. Database plus three instances.**
The mutex lives in one process's memory; three instances have three mutexes, so two
instances can each allocate the same slot. The database must arbitrate: a conditional update
(`UPDATE slots SET occupied = true WHERE id = $1 AND occupied = false`, checking the row
count), a unique constraint on active tickets per slot, or a row lock in a transaction.

**6. Splitwise, first 15 minutes.**
Operations: add user, add expense (payer, amount, participants, split type), show balances
for one user and for all, maybe settle up. Entities: `User`, `Expense`, `Split`, and a
balance ledger (who owes whom). First interface: `SplitStrategy`, implemented by equal,
exact and percentage splits, because split rules are what the interviewer will extend
("now add shares"), and it keeps validation (percentages sum to 100, exact amounts sum to the
total) inside each strategy. Balances are derived from expenses, updated on each expense, in
memory behind a small store.

---

## File 12 — The AI era

**1. Evidence on system design rounds.**
No verified source says system design rounds have changed. The verified changes are to
coding rounds: Canva's blog (company source) describes an AI-assisted coding round; Meta's
change is reported by 404 Media (press, quoting an internal message); Google's in-person
discussion comes from CNBC's report of town-hall audio, with stronger "confirmed" claims
second-hand; Anthropic's candidate guidance is its own page. "GenAI system design" questions
come from prep firms (commercial) and candidate posts.

**2. 6-second summaries.**
Stream the output so text appears early (perceived latency; TTFT drops to well under the
total). Precompute summaries asynchronously when tickets are created or updated, so the
page reads a stored result (real latency at read time, near zero). Shorten the output or the
prompt, and put fixed instructions first so provider prompt caching applies (real latency
and cost, somewhat). Showing a spinner improves neither.

**3. 100,000 tokens per minute at 2,500 per request.**
40 requests a minute, sustained. Your limiter should count estimated tokens (input plus the
maximum output you request), per user and globally, and keep headroom below the provider
limit. A request that would exceed the budget should queue for the next window, degrade (a
smaller model or shorter answer), or get a clear "try again shortly", rather than being sent
to collect a 429.

**4. Semantic cache for banking.**
"How do I block my debit card?" and "How do I unblock my debit card?" can be very similar in
embedding space and need opposite answers; so can "What's my balance?" for two different
users if the cache key ignores the user. Allow semantic caching for public, generic,
non-personal FAQ answers; never for anything account-specific, transactional, or where a
wrong answer causes harm.

**5. Question first, policy document after.**
Provider caching reuses a common *prefix*. With the per-request question first, every
prompt's prefix differs from the first token, so the 20 pages after it are never reused. Put
instructions and the policy document first, the user's question last.

**6. RAG answer quoting a forbidden page.**
At retrieval: the top-k search ran over all chunks, not just ones the user may read, and the
retrieved text went into the prompt. Permissions must be applied as a filter on the vector
search (each chunk carries the page's access metadata), before anything reaches the model.
Filtering the answer afterwards is too late; the model has already read the text.

---

## Things to try

1. **Run the four consistent-hash implementations.** Copy the Go, Rust, C++ and JavaScript
   rings from file 05, run each over the same 10,000 keys, and confirm they assign identically.
   Then raise virtual nodes from 100 to 1,000 and watch the per-node counts converge.
2. **Break the JavaScript LRU on purpose.** Remove the `delete` before `set` in `get`, and
   write the three-line test that shows it became FIFO.
3. **Watch a stampede.** Put a 50 ms sleep in a "database" function behind an in-process
   cache, fire 1,000 concurrent requests for one cold key, and count database calls. Add
   Go's `singleflight` and count again.
4. **Compare backoff without and with jitter.** Simulate 1,000 clients failing at t = 0 and
   retrying with plain exponential backoff, then full jitter; histogram arrivals per 10 ms.
   You should be able to reproduce the shape of Brooker's charts.
5. **Time yourself on a machine coding problem.** Splitwise, 90 minutes, one language, a
   `main` that runs a scenario. Then ask someone to give you a follow-up requirement and
   count how many files you had to touch.
6. **Estimate a system you use.** Pick an app you use daily, estimate its peak QPS and
   yearly storage from public user numbers, and write down which assumption you trust least.

## Questions worth asking me

- "Run me through designing <a system> with the file 01 method, and interrupt me the way an
  interviewer would."
- "Here's my design for X. Where would a senior interviewer push, and what's the weakest
  part?"
- "Give me a follow-up requirement for my parking lot / Splitwise code, and review how I
  absorbed it."
- "What changes in the chat design if groups have 100,000 members?"
- "Explain Raft or Paxos at the level a system design round needs." (Not covered here yet.)
- "How do distributed transactions work: two-phase commit and sagas?" (Not covered here yet.)
- "Design a payment system, and show me where idempotency keys and the outbox go."
- "What would an SDE-2 versus a senior answer to this prompt look like?"
- "Which of these chapters' claims are the least well supported, and how would I check?"
