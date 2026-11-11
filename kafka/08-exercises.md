# 8. Exercises

Worked answers to every **Check yourself** question, then things to try and questions
worth asking me.

---

## File 01 — What Kafka is

**1. Two groups, A processes an event. Can B still read it?**

Yes. Reading doesn't remove anything. A moving its committed offset past the event changes
only A's bookmark. B reads the event whenever B's own offset reaches it. For B never to see
it, the event would have to be deleted by retention (or compaction) before B gets there:
B being down, or lagging, for longer than the topic's retention.

**2. Ten days off, default retention.**

Default `retention.ms` is seven days, so everything older than seven days has been deleted,
including events B's committed offset points at or beyond. On restart the committed offset
refers to data that no longer exists, so the client's reset policy applies: start at the
earliest *remaining* event, or at the latest. Either way three days of events are gone with
no error raised. You'd only notice through a gap in the data, or by monitoring consumer lag
against retention.

**3. Why positions, and the cost for parallelism.**

Storing one number per partition per group is tiny and fast, whatever the volume. Storing
which individual messages are done would cost space and coordination per message. The cost
is that the commit means "everything before this offset is done". A consumer that processes
several messages from one partition at once can't commit a later one while an earlier one is
still running, or a crash loses the earlier one. It has to track completion and commit only
up to the first unfinished message, or parallelise across partitions instead.

**4. RF 3, default `min.insync.replicas`, ISR shrinks to the leader.**

Not safe. With `min.insync.replicas=1`, `acks=all` means "all in-sync replicas", and the only
in-sync replica is the leader itself. The write is acknowledged after the leader alone has it.
When the leader's disk fails, no other broker has the event, and the producer was told it
succeeded. `min.insync.replicas=2` would have made the leader *reject* the write while only one
replica was in sync, so the producer would have known.

**5. Eight consumers, six partitions.**

Six of them each own one partition and do the work. The other two own nothing and sit idle.
They're useful only as standbys, taking over if a member fails. Adding consumers beyond the
partition count adds no throughput. To scale further you need more partitions, which brings
chapter 2's key-movement problem.

---

## File 02 — Partitions, keys and order

**1. Keyed messages with `LeastBytes`, six partitions.**

`LeastBytes` sends each message to whichever partition has received the fewest bytes from
this writer so far, and never looks at the key. Two consecutive messages with the same key
usually go to *different* partitions, because the first one just made its partition the
fullest. Consumers of different partitions process them independently, so the second
message can be handled before the first. The key is carried along as metadata and does
nothing for ordering.

**2. Why a one-partition topic hides all of it.**

With one partition every message, whatever its key, goes into one log, read in order by one
consumer. Balancer choice, hash function and partition count can't matter when there's
nothing to choose between. Local Kafka setups and test fixtures usually create
single-partition topics, so the configuration that breaks ordering is never exercised
before production.

**3. Java producer and kafka-go `Hash` on one topic.**

The Java client hashes keys with murmur2. kafka-go's `Hash` uses FNV-1a. For the same key
they usually pick different partitions, so one customer's events are split across two
partitions depending on which service produced them, with no relative order between the
two halves. Fix: use kafka-go's `Murmur2Balancer`, documented as compatible with the Java
partitioner (and check its `Consistent` setting matches how the Java side treats keys).

**4. 6 to 12 partitions on a busy day.**

Before the change, customer A's key hashes to, say, partition 2 of 6. After it, the same
hash modulo 12 may give partition 8. Events produced before the change wait in partition 2,
events produced after it go to partition 8. If partition 2 has lag and partition 8 doesn't,
the consumer processes A's newer events first, applying an old "details changed" after a
new "re-verified" and leaving A marked stale. Avoid it by sizing partitions up front, or by
draining consumers before the change, or by making the consumer compare event timestamps
instead of trusting arrival order.

**5. Resolving only the staleness the verification supersedes.**

```sql
UPDATE staleness
   SET resolved = true, resolved_at = now(), resolved_by = $verification_id
 WHERE customer_id = $1 AND tenant_id = $2 AND resolved = false
   AND detected_at <= $verified_at;
```

The staleness event must carry `detected_at`, the time the underlying change happened (not
when the event was published or consumed). The verification event must carry
`verified_at`, the time of the verification itself. If either side only has "when I
received it", the comparison is again about arrival order.

---

## File 03 — Offsets and delivery guarantees

**1. Processed twice with no crash.**

A consumer processes offsets 100–120 and has committed up to 110 (an async commit is
pending, or it commits every N messages). A new pod joins during a deploy and triggers a
rebalance. The partition moves to the new pod, which starts from the committed offset, 110,
and processes 110–120 again. Nothing crashed. The ownership change alone caused the replay.

**2. `CommitInterval: time.Second`, killed 200 ms after "committing" 500.**

With a non-zero interval, kafka-go's commits are asynchronous: `CommitMessages` records the
offset locally and a background loop sends it on the next tick. The pod died before the
tick, so the broker still has the previous committed offset, perhaps 450. The replacement
starts there and reprocesses 450–500.

**3. Three goroutines committing their own messages.**

The goroutines take offsets 10, 11 and 12. 11 and 12 finish quickly. 12 commits, and the
committed offset becomes 13 (committing a message commits everything before it). 10 is
still running. The process crashes. On restart, the group resumes at 13. Offset 10 never
finished and is never redelivered.

**4. Renaming a kafka-go group.**

`orders-indexer-v2` has no committed offsets, so kafka-go's default `StartOffset:
FirstOffset` applies: the consumer starts from the oldest retained message and reprocesses
the whole retention window. If handlers are idempotent that's wasted work, and if they
aren't, it's a data incident. With the Java client's default `auto.offset.reset=latest`, the
new group would start at the end of the log and skip everything produced while the rename
was being deployed. Set the start position explicitly, or migrate the old group's offsets to
the new id first.

**5. Handler fails, DLQ publish fails, commits anyway.**

The message is past the committed offset, so Kafka will never redeliver it to this group.
It isn't in the DLQ. Its effect never happened. The only evidence is two log lines, and only
if the log entry includes the topic, partition and offset. Then someone can re-read the
message from the source topic while it's still retained.

**6. Idempotent handler versus exactly-once delivery.**

Exactly-once *delivery* across Kafka and a database is impossible without a shared
transaction, because the database write and the offset commit are two separate writes.
Kafka's transactions give exactly-once processing only when the output is also Kafka. An
idempotent handler makes at-least-once delivery *behave* as exactly-once at the point that
matters, the database, using a mechanism you control: a unique event id or a naturally
idempotent update. It also covers redeliveries from causes Kafka doesn't see, such as a
producer retrying.

---

## File 04 — When a message fails

**1. Malformed timestamp, retried forever.**

The partition stops. Every later message in it waits behind the bad one, and lag grows
without bound. Restarting the pod causes a rebalance. The partition is reassigned (perhaps
to the same pod, perhaps to another), the new owner starts from the committed offset, which
is the poison message, and blocks on it again. It moves around the group like a hot potato,
and never gets past. Only fixing the code, or skipping the offset by hand, frees it.

**2. Ten-minute database outage.**

"DLQ on any error": every message consumed during the outage fails, goes to the DLQ, and is
committed. When the database recovers, the consumer is caught up, and the DLQ holds ten
minutes of perfectly good messages that someone has to find and replay, in an order that may
no longer be correct. "Pause on transient errors": the consumer stops at the first failure,
lag grows for ten minutes, an alert fires on lag, and when the database recovers it resumes
and drains the backlog in order. Nothing to clean up.

**3. Retry topics and ordering.**

A message for key A fails and goes to the retry topic. The consumer commits it and carries
on with the partition, and the next message for A succeeds immediately. When the retry
fires, the older message is processed after the newer one. To tolerate that, the consumer
has to stop trusting arrival order. Each event carries its own time or version, and updates
apply only if they're newer than what's stored.

**4. A DLQ entry with only `"processing failed"`.**

You can't find the original message (no topic, partition or offset). You can't replay it
(no key or value). You can't tell whether replaying would help (no error type or
classification). You can't tell which code version failed it, or correlate it with a deploy.
It tells you only that something failed, which the logs already did.

**5. Two reasons not to health-check by publishing to the DLQ.**

It pollutes the DLQ with fake failures on every start, so DLQ volume can't be used for
alerting, and anything that consumes or replays the DLQ has to know to skip them. And it
couples startup to Kafka being writable: a degraded cluster stops the service from
starting, even if Kafka is incidental to what it serves.

---

## File 05 — Publishing from a request handler

**1. Row without event, despite three retries.**

The process is killed (deploy, OOM, node failure) after the commit and before the publish
completes. Every retry times out because the broker is unreachable for longer than the
retries last. The publish "succeeds" with `acks=0` or `acks=1` and the message is lost on the
broker side. The handler's context is cancelled mid-publish because the client disconnected.
In each case the database transaction has already committed and nothing records that an
event is owed.

**2. Publishing with `r.Context()` in a goroutine.**

The server cancels a request's context when the handler returns. The handler returns
immediately after starting the goroutine, so the publish sees a cancelled context and
fails, usually before it reaches the broker. Fix one: `context.WithTimeout(context.Background(),
…)`, a fresh context with its own deadline that carries none of the request's values. Fix
two: `context.WithTimeout(context.WithoutCancel(r.Context()), …)` (Go 1.21+), which ignores
the request's cancellation but keeps its values, such as trace and request ids, so the
publish still shows up in the request's trace. Either way, add a timeout: `WithoutCancel`
alone has no deadline.

**3. `RequiredAcks` unset, broker restarts.**

The zero value is `RequireNone`. The writer sends and doesn't wait for any
acknowledgement, so every `WriteMessages` returns success. The application believes all
the messages were published. In fact, messages sent to the restarting broker while it was
down or leaderless may never have been stored, and nothing anywhere records that.

**4. A one-second jump in p50.**

Synchronous `WriteMessages` with a batch size above 1 blocks until the batch fills or
`BatchTimeout` expires. One message per request never fills a batch of 100, so every call
waits for the default one-second timeout. Fixes: set `BatchTimeout` to a few milliseconds,
or share one `Writer` across the process so concurrent requests fill batches together. (Or
`BatchSize: 1` if volume is tiny, at the cost of one produce request per message.)

**5. Relay publishes, then crashes before marking sent.**

On restart the relay sees the row still unsent and publishes it again, so downstream gets a
duplicate. Consumers need to deduplicate, typically on the event id that the outbox row
carried, with a unique constraint or an idempotent update. That's ordinary at-least-once
handling, which they needed anyway because of rebalances.

**6. Outbox or fire-and-forget?**

(a) Outbox. If it's lost, a compliance flag stays wrong indefinitely and nothing corrects it.
(b) Fire-and-forget. If it's lost, the first real request fetches the thumbnail a little more
slowly, and that's the whole cost.

## File 06 — Batching consumers, commits and rebalances

**1. Manual commits per batch, autocommit never disabled, low traffic.**

The batch threshold isn't reached for a long time, so records sit in the buffer across many
polls. Every five seconds franz-go's autocommit commits the offsets of every poll except the
most recent one, and those polls' records haven't been processed. After a minute, the
committed offset is close to the end of what's been fetched. A crash loses everything in the
buffer except the last poll's records: the group restarts after them. Measured with 100
records buffered for 12 seconds: committed offset 100, nothing processed.

**2. Non-greedy autocommit, with and without a buffer.**

Non-greedy autocommit commits "what has previously been polled", never the most recent poll.
A consumer that processes each poll before calling poll again has, by the time it polls,
finished everything from the previous poll, so committing it is exactly right: at-least-once.
A buffering consumer polls again before processing, so "previously polled" includes records
still waiting in the buffer, and committing them is at-most-once for those records. It's the
Java documentation's requirement, to consume everything from each poll before the next,
applied to franz-go.

**3. Partition 2 moves mid-batch, no revoke handler.**

A's last commit for partition 2 was, say, offset 100, and A has polled to 160. The partition
moves to B, which starts from 100 and processes 100 onward: A and B both process 100–159.
Then A finishes its batch and calls `CommitUncommittedOffsets`, which may include partition 2
at 160. If B has meanwhile committed 200, A's commit can rewind the group to 160, so the next
owner reprocesses 160–199 as well. The franz-go documentation describes exactly this ("you,
member A, can rewind member B's commit"). A revoke handler that commits or discards partition
2's buffered work before giving it up avoids both.

**4. Eager to cooperative "to stop duplicates".**

It shortens rebalance pauses: only partitions that actually move are revoked, so the rest of
the group keeps consuming during scaling and rolling deploys. It doesn't change what happens
to a partition that does move. Anything polled but uncommitted on that partition is processed
again by the new owner, so duplicates still happen, just on fewer partitions. Fewer
duplicates come from committing at revocation, or from smaller batches.

**5. Hour-long scheduled consumer, last batch twice.**

The batch that's in progress when the deadline passes finishes processing, then commits with
the expired context, and the commit fails with `context deadline exceeded` (measured: commit
error, committed offset unchanged). The next run starts from the old offset and processes that
batch again. Fix: stop polling when the window closes, but give the final commit its own short
deadline from a context detached from the window.

**6. `BlockRebalanceOnPoll` with ninety-second batches.**

A new member joins and a rebalance begins. Your member has polled, so a rebalance that takes
partitions from it waits until you call `AllowRebalance`, up to ninety seconds. The group's
rebalance timeout defaults to 60 seconds, and "if a member does not rejoin within this timeout,
Kafka will kick that member from the group". Your member is removed, its partitions are
reassigned, and its eventual commit fails. Either keep batches well under the rebalance
timeout (`PollRecords` with a cap), raise `RebalanceTimeout`, or don't block rebalances and
handle revocation in a callback.

---

## File 07 — Inside the broker

**1. `kill -9` versus power loss.**

`kill -9` ends the process, not the kernel. The leader put the batch into the page cache with
`write()` before acknowledging it, and the page cache belongs to the kernel, which writes dirty
pages to the device in its own time whether or not the broker is alive. On restart there's no
clean-shutdown marker, so the broker re-reads every segment after its recovery point, finds the
batches valid, and keeps them. Nothing is lost, on that broker or anywhere else.

Power loss takes the page cache with it. Anything written since the last flush, often most of the
active segment, may never have reached the device. On restart, recovery finds the log ends early or
hits an invalid batch, and truncates there. The write is gone from that broker. Consumers can still
read it: `acks=all` meant every in-sync replica had it in its page cache, the other replicas kept
their power, one of them is now the leader, and the restarted broker fetches the write back before
it can rejoin the ISR.

**2. Fetching from 52 345.**

The segment is the one with the largest base offset at or below 52 345, which is 42 000, so
`00000000000000042000.log`. The relative offset is 52 345 − 42 000 = 10 345. The broker binary-searches
the memory-mapped `.index` for the last entry at or below 10 345. The index is sparse, roughly one entry
per 4 KiB of log, so that entry points at a batch somewhat before the target. From that byte position
it reads batch headers forward, using `baseOffset` and `lastOffsetDelta` to step over whole batches,
until it reaches the batch that contains 52 345, and sends from the start of that batch.

Batches go out whole, and a compressed batch can't be cut, so the broker can't start partway through
one. If the batch covers 52 340–52 349, the client receives 52 340–52 344 as well and discards them. The
response may also end with a partial batch, which the client drops and fetches again next time.

**3. One record an hour.**

The segment never reaches 1 GiB, so it rolls by time: when a batch arrives whose timestamp is more than
`segment.ms` (seven days) after the segment's first batch. Each closed segment therefore spans about
seven days. Retention deletes a segment once its *newest* record is older than `retention.ms`, which for
this segment is seven days after it closed. By then its first record has been on disk for about fourteen
days, plus up to five minutes until the next retention check and a minute of `file.delete.delay.ms`.

Raising or lowering `retention.ms` alone doesn't fix the overshoot, because the overshoot comes from how
much time one segment spans. Lower `segment.ms` to bound it (one day gives at most about eight days), then
lower `retention.ms` by the same amount if seven days is a hard maximum (six days plus one). The cost is
more, smaller segment files per partition, and with them more open files and more index files. If the
deadline is strict, check the producers' clocks too, since `CreateTime` timestamps decide both the roll
and the deletion.

**4. The replaying group.**

Caught-up groups and followers read data written seconds ago. It's still in the page cache, so those
reads cost no disk I/O and `sendfile` serves them straight from RAM. The replaying group reads week-old
segments that aren't cached. Those reads go to the device, and the pages they bring in take space in the
same cache, pushing out the recent pages everyone else was about to read. Now the other groups miss the
cache and read from disk too, and producers' appends compete with all of it for I/O. Nothing in Kafka is
misbehaving: there is one kernel-managed cache, and every reader shares it. The usual remedies are to
throttle the replay with a fetch quota, or to give brokers more RAM relative to the data in active use.
How much it hurts depends on that ratio, and there's no general figure.

**5. TLS and CPU.**

Zero copy. Without TLS, the broker hands the kernel a file range and a socket, and `sendfile` moves the
pages from the page cache to the network without the bytes ever entering the JVM. With TLS, encryption
happens in the JVM's user-space TLS library, under keys that differ per connection, so every byte has to
be copied out of the page cache, encrypted, and copied back into the socket. The design page is direct
about it: "`sendfile` is not used when SSL is enabled". The extra CPU goes on those copies and on the
encryption itself. Keeping zero copy would need TLS inside the kernel, which Kafka doesn't support.

**6. One power feed.**

Every acknowledged write that no replica had flushed yet. Kafka flushes when a segment rolls and at
shutdown, not per write, so that can be most of each partition's active segment. `acks=all` with
`min.insync.replicas=2` guaranteed each write was in at least two page caches, but all of those caches
were in the same failure domain, so they all went at once.

Recovery doesn't help because it works by re-fetching from a replica that still has the data, and none
does. Each broker's local recovery truncates at the same kind of point, and the cluster carries on with
those writes missing.

The two remedies:

- **Separate failure domains.** Set `broker.rack` so replicas are spread across racks or zones. This costs
  cross-rack or cross-zone traffic and some latency on every replicated write.
- **Flush on the write path.** Set `flush.messages` or `flush.ms`. At `flush.messages=1`, each append is
  fsynced before it's acknowledged. This costs throughput and latency, because every flush waits for the
  device.

The first is the remedy Kafka's design assumes. The second is the setting the configuration reference
recommends you leave alone.

---

## Things to try

- Run a local Kafka with a six-partition topic. Produce 100 messages with the same key from
  kafka-go using `LeastBytes`, then `Hash`, then `Murmur2Balancer`, and run
  `kafka-console-consumer --property print.partition=true` to see where they went.
- Start a consumer group, commit some offsets, change the group id, and watch where a
  kafka-go reader and a Java consumer start.
- Kill a consumer with `kill -9` mid-batch, using a one-second `CommitInterval`, and count
  the duplicates on restart.
- Build a minimal outbox: one table, one relay loop with `FOR UPDATE SKIP LOCKED`, and a
  consumer that deduplicates by event id. Kill the relay at random and check that nothing is
  lost.
- Write a franz-go consumer that buffers records for fifteen seconds without processing them.
  Run it once with defaults and once with `DisableAutoCommit()`, stop it, and read the group's
  committed offset with `kafka-consumer-groups.sh --describe`.
- Start two members of one group with a long-running batch, add a third, and log
  `OnPartitionsRevoked` calls and duplicate offsets under the eager and cooperative balancers.
- Create a topic with `segment.bytes` set to 1 MiB, produce a few megabytes, and list the partition
  directory to watch segments roll. Then lower `retention.ms` to a minute and watch `.deleted` files
  appear and vanish.
- Run `kafka-dump-log.sh --files <segment>.log --print-data-log` on a segment and find each batch's
  base offset, last offset, CRC and compression codec. Compare a batch from a compressed producer with
  one from an uncompressed one.
- `kill -9` a broker under load, restart it, and find "Recovering unflushed segment" in its log. Stop it
  cleanly, restart it, and confirm the line doesn't appear.

## Questions worth asking me

- "Here's our producer config. What does a successful `Publish` actually guarantee?"
- "We need per-customer ordering across two event types. One topic or two?"
- "How do Kafka transactions and idempotent producers fit with everything here, and when
  are they worth it?"
- "How should the outbox relay handle ordering when several relay instances run?"
- "What's a sensible lag alert for this consumer, and what should it page on?"
- "Here's our batching consumer loop. Which records can be lost or duplicated, and when?"
- "How does the newer consumer group protocol (KIP-848) change rebalancing compared with
  chapter 6?"
- "What changes about retention, recovery and page-cache pressure when tiered storage is on?"
- "How does the log cleaner choose what to compact, and what happens to tombstones?"
- "Walk me through what Eligible Leader Replicas change in the last-replica-standing case."
- "Here's our broker and topic config. What does a power cut in one zone lose?"
