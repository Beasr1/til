# 13. Glossary

Reference, not reading. Each entry points at the chapter that explains it.

| Term | Meaning | Where |
|---|---|---|
| **Anti-entropy** | Background comparison of replicas (often with Merkle trees) to find and repair differences | [05](05-data-storage.md) |
| **At-least-once** | Delivery that never loses a message but may repeat it. The normal choice; needs idempotent handlers | [07](07-asynchronous-work.md) |
| **At-most-once** | Delivery that never repeats a message but may lose it | [07](07-asynchronous-work.md) |
| **Availability** | Informally: the fraction of time or requests the service works. In CAP: every request to a non-failing node gets a response, eventually | [01](01-foundations.md), [06](06-consistency-and-cap.md) |
| **Back-pressure** | A slow consumer making a fast producer slow down, rather than letting a buffer grow without bound | [07](07-asynchronous-work.md) |
| **Bulkhead** | Separate resource pools per dependency, so one slow dependency can't exhaust them all | [08](08-reliability-patterns.md) |
| **Cache-aside** | The app reads the cache, falls back to the database on a miss, fills the cache; writes go to the database and delete the cache entry | [04](04-caching.md) |
| **Cache stampede** (thundering herd) | Many requests missing on the same key at once and all recomputing it | [04](04-caching.md) |
| **CAP theorem** | Gilbert & Lynch 2002: no read/write register can be both linearizable and always available when messages can be lost | [06](06-consistency-and-cap.md) |
| **Cascading failure** | "A failure that grows over time as a result of positive feedback" (SRE book) | [08](08-reliability-patterns.md) |
| **CDN** | Content delivery network: caches at many locations near users | [04](04-caching.md) |
| **Circuit breaker** | Wrapper that stops calling a failing dependency for a while, failing fast instead; closed, open, half-open | [08](08-reliability-patterns.md) |
| **Consistent hashing** | Mapping keys to servers so that adding or removing a server moves only about 1/N of keys | [05](05-data-storage.md) |
| **Consumer lag** | How far behind the newest message a consumer is | [07](07-asynchronous-work.md) |
| **Cursor pagination** (keyset) | Paging by "after this sort key" rather than by offset; stable under inserts | [09](09-api-design.md) |
| **Dead-letter queue** | Where messages that can never succeed are sent, with the reason, for a human | [07](07-asynchronous-work.md) |
| **Dual write** | Writing two systems (database and broker) with no shared transaction, so one can succeed without the other | [07](07-asynchronous-work.md) |
| **Error budget** | The unreliability an SLO allows (100% minus the target), spent on change | [01](01-foundations.md) |
| **Eventual consistency** | If writes stop, replicas converge eventually; no bound on when | [06](06-consistency-and-cap.md) |
| **Fan-out on write / on read** | Precomputing each reader's feed when a post is made, versus assembling it when it's read | [10](10-worked-designs.md) |
| **Failover** | Promoting a follower to leader when the leader fails | [05](05-data-storage.md) |
| **Full jitter** | Backoff that sleeps a random time between zero and the exponential ceiling | [08](08-reliability-patterns.md) |
| **Hedged request** | Sending a duplicate request to another replica after a short delay and taking the first answer | [08](08-reliability-patterns.md) |
| **Hinted handoff** | A stand-in node holding a write for a down replica and handing it back on recovery | [05](05-data-storage.md) |
| **Hot key** | One key receiving a disproportionate share of traffic, overloading its shard or cache node | [05](05-data-storage.md) |
| **Idempotency key** | A client-generated id sent with a request so the server can recognise and deduplicate retries | [09](09-api-design.md) |
| **Idempotent** | Doing it twice has the same effect as doing it once. RFC 9110: PUT, DELETE and safe methods are | [07](07-asynchronous-work.md), [09](09-api-design.md) |
| **L4 / L7 load balancer** | Balancing by TCP connection (transport layer) versus by HTTP request (application layer) | [03](03-scaling-a-service.md) |
| **Latency** | Time for one operation | [01](01-foundations.md) |
| **Leader–follower replication** | One node accepts writes and streams its log to followers | [05](05-data-storage.md) |
| **Leaderless replication** | Clients write to and read from several replicas directly, using quorums (Dynamo-style) | [05](05-data-storage.md) |
| **Linearizability** | The system behaves as if there's one copy and each operation takes effect at an instant | [06](06-consistency-and-cap.md) |
| **Little's Law** | L = λW: requests in flight = arrival rate × time in system | [01](01-foundations.md) |
| **Load shedding** | Rejecting some work early and cheaply when overloaded, rather than doing all of it slowly | [08](08-reliability-patterns.md) |
| **LLD** | Low-level design: types, interfaces and responsibilities inside one component | [11](11-lld-and-machine-coding.md) |
| **LRU / LFU** | Evict the least recently / least frequently used entry | [04](04-caching.md) |
| **Machine coding round** | Building a working, extensible program to a spec in a fixed time, commonly reported at Indian product companies | [11](11-lld-and-machine-coding.md) |
| **Multi-leader replication** | Several nodes (often one per region) accept writes; conflicts must be resolved | [05](05-data-storage.md) |
| **Outbox** (transactional) | Writing an event to a table in the same transaction as the data, and publishing it from there | [07](07-asynchronous-work.md) |
| **p99** | The latency 99% of requests are faster than | [01](01-foundations.md) |
| **PACELC** | Abadi: during a Partition, trade Availability against Consistency; Else, Latency against Consistency | [06](06-consistency-and-cap.md) |
| **Power of two choices** | Pick two servers at random and use the less loaded | [03](03-scaling-a-service.md) |
| **Prompt caching** | A provider reusing its processing of a repeated prompt prefix | [12](12-the-ai-era.md) |
| **QPS** | Queries (requests) per second | [01](01-foundations.md), [02](02-estimation.md) |
| **Quorum** (N, R, W) | N replicas; a write needs W acknowledgements, a read R answers; R + W > N makes them overlap | [05](05-data-storage.md) |
| **RAG** | Retrieval-augmented generation: retrieve relevant passages and include them in an LLM prompt | [12](12-the-ai-era.md) |
| **Read-your-writes** | A user always sees their own earlier writes | [06](06-consistency-and-cap.md) |
| **Replication lag** | How far a follower is behind the leader | [05](05-data-storage.md) |
| **Retry budget** | A cap on retries as a fraction of traffic, so retries stop when everything is failing | [08](08-reliability-patterns.md) |
| **Semantic cache** | Reusing an answer to a *similar* earlier question, found by embedding similarity | [12](12-the-ai-era.md) |
| **Sharding** (partitioning) | Splitting data so each machine holds a subset of keys | [05](05-data-storage.md) |
| **SLI / SLO / SLA** | The measurement; the target for it; the contract with consequences | [01](01-foundations.md) |
| **Sliding window counter** | Rate limiting that weights the previous window's count by its remaining overlap | [08](08-reliability-patterns.md) |
| **Sloppy quorum** | Writing to the first N *healthy* nodes when home replicas are down | [05](05-data-storage.md) |
| **SOLID** | Five object-oriented design principles: single responsibility, open/closed, Liskov substitution, interface segregation, dependency inversion | [11](11-lld-and-machine-coding.md) |
| **Split brain** | Two nodes both believing they're the leader | [05](05-data-storage.md) |
| **Stateless service** | A server that keeps nothing between requests that another server would need | [03](03-scaling-a-service.md) |
| **Sticky sessions** | Routing a user to the same server every time | [03](03-scaling-a-service.md) |
| **Throughput** | Operations per unit time | [01](01-foundations.md) |
| **Token bucket** | Rate limiting with a bucket of up to *b* tokens refilled at *r* per second, one spent per request | [08](08-reliability-patterns.md) |
| **TTFT** | Time to first token of a streamed LLM response | [12](12-the-ai-era.md) |
| **TTL** | Time to live: how long a cache entry is kept | [04](04-caching.md) |
| **Vector clock** | Per-replica counters attached to a value, to tell whether two versions are ordered or concurrent | [05](05-data-storage.md) |
| **Virtual nodes** | Placing each server at many points on a consistent-hash ring to even out load | [05](05-data-storage.md) |
| **Write-ahead log (WAL)** | Changes logged before data files are changed; what replicas replay | [05](05-data-storage.md) |
| **Write-through / write-behind** | Writing the cache and database together / writing the cache now and the database later | [04](04-caching.md) |
