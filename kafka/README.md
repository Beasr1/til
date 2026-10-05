# Kafka from the Application Side — A Course

A short course on what Kafka actually promises an application, and on the client-library
settings that decide whether you get it: message order, delivery after failures, and
getting an event out of a request handler without losing it.

**This is reference learning material.** Everything here is general Kafka, checked
against the Apache Kafka documentation and, for client behaviour, against the source of
the Go clients `segmentio/kafka-go` and `twmb/franz-go`, because that's where defaults are
actually set. The motivating failures are mundane: events for one entity processed out of
order despite being keyed, a failed message that vanished between a dead-letter publish and a
commit, events silently lost when a deploy stopped a process, and a batching consumer whose
automatic commits ran ahead of its processing.

I wrote this as a teacher, not as a peer. That means:

- I explain things you might already know. Skim if so.
- Why before how: the failure comes before the rule that prevents it.
- Every chapter ends with **Check yourself** questions. Answers are in
  [07-exercises.md](07-exercises.md).
- Where a client library and the Kafka documentation disagree, both are quoted. The
  disagreement is usually the lesson.

## The one thing to understand first

> **Kafka promises order within one partition and durability up to the commit, and
> nothing else. Everything stronger, such as per-key order, exactly-once effects, or
> events that match your database, is built by your code and your client configuration,
> and the defaults differ between clients.**

The broker stores messages in whichever partition the producer names and remembers one
offset per partition per consumer group. Which partition, which offset, and when it's
committed are all decided client-side.

## Reference implementations

| Source | What it settles |
|---|---|
| [Kafka introduction](https://kafka.apache.org/42/getting-started/introduction/) | Per-partition ordering, and the documented key-to-partition behaviour |
| [Kafka design: delivery semantics](https://docs.confluent.io/kafka/design/delivery-semantics.html) | At-most-once and at-least-once as a function of commit position |
| [Topic configs](https://kafka.apache.org/43/configuration/topic-configs/) | Retention, cleanup policy, `min.insync.replicas` |
| [Kafka 4.0 upgrade notes](https://kafka.apache.org/40/getting-started/upgrade/) | ZooKeeper removed; KRaft only |
| [Producer configs](https://kafka.apache.org/43/configuration/producer-configs/) | `acks` values and default |
| [Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/) | `auto.offset.reset` and its default |
| [Kafka 3.0 upgrade notes](https://kafka.apache.org/30/getting-started/upgrade/) | The `acks=all` and idempotence default change |
| [segmentio/kafka-go](https://github.com/segmentio/kafka-go) (`writer.go`, `reader.go`, `balancer.go`) | The Go client's balancers, batching, acks and offset defaults |
| [Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) | The dual-write problem and its standard fix |
| [KafkaConsumer javadoc](https://kafka.apache.org/43/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html) | The condition under which auto-commit is at-least-once |
| [KIP-429](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol) | Eager versus incremental cooperative rebalancing |
| [twmb/franz-go](https://github.com/twmb/franz-go) (`pkg/kgo/config.go`, `consumer_group.go`) | Autocommit defaults, revocation, `BlockRebalanceOnPoll` |

## Reading order

| # | File | After this you can… |
|---|------|---------------------|
| 1 | [What Kafka is](01-what-kafka-is.md) | ⭐ Explain topics, partitions, offsets, brokers, replicas, consumer groups and retention, and why Kafka is a log rather than a queue. **Prerequisite for everything else** |
| 2 | [Partitions, keys and order](02-partitions-keys-and-order.md) | ⭐ Say exactly what order a consumer sees, and check whether your producer honours keys at all |
| 3 | [Offsets and delivery guarantees](03-offsets-and-delivery-guarantees.md) | Predict when a message is processed twice or never, and where a new group starts |
| 4 | [When a message fails](04-when-a-message-fails.md) | Classify failures, design a dead-letter topic someone can use, and avoid poison pills |
| 5 | [Publishing from a request handler](05-publishing-from-a-request.md) | ⭐ Explain the dual-write problem and when an outbox is worth it |
| 6 | [Batching consumers, commits and rebalances](06-batching-consumers-and-rebalances.md) | ⭐ Explain why autocommit loses buffered records, what a rebalance does to a batch in flight, and where a batch's commit belongs |
| 7 | [Exercises & answers](07-exercises.md) | Check the model formed, and go deeper |

Read chapter 1 first, then 2 and 3 in order. Chapter 4 builds on 3's commit rules. Chapter 5
stands alone apart from a reference to at-least-once delivery. Chapter 6 builds on chapter 3,
and belongs after it in reading order; it was added later, so it takes the next number. Not yet
written, and belonging here: Kafka transactions and idempotent producers in depth, the newer
consumer group protocol (KIP-848), consumer lag and capacity, schema registries, and
compaction. The questions at the end of file 07 are the honest list.

## If you're short on time

- **New to Kafka:** file 01, all of it.
- **10 minutes:** file 02, the client-defaults table and the trap after it.
- **A consumer is reprocessing or skipping messages:** file 03, "Where a new group starts"
  and "Committing an offset commits everything before it".
- **Designing a dead-letter topic:** file 04, the classification diagram and the field
  table.
- **Adding an event to an HTTP handler:** file 05 top to bottom.
- **Writing a consumer that batches records:** file 06, the autocommit section and its trap.

## The one-paragraph summary of everything

Kafka is a replicated, append-only log: producers write events to topics, reading doesn't
remove them, and each consumer group keeps its own committed offset per partition, so many
services can read one topic independently and re-read it within retention (seven days by
default). Each partition has a leader and in-sync followers. `acks=all` waits for the in-sync
replicas, and `min.insync.replicas`, default 1, decides how many that must be. A Kafka topic is a set of partition logs, and the only ordering guarantee is within one
partition. Per-key order exists only if the producer sends every message with a given key
to the same partition. The Kafka documentation describes that as how keys work, but it's a
client choice: the Java client hashes keys with murmur2, while kafka-go's default writer is
round-robin and its `LeastBytes` balancer ignores keys entirely. Two producers with
different hash functions split one key across partitions, and adding partitions moves keys.
There's no order at all across topics, so consumers that combine topics must compare
timestamps or versions carried in the events, never arrival order. A consumer group keeps
one committed offset per partition. Committing after processing gives at-least-once
delivery, and redelivery happens on every rebalance and after async commits, not just
crashes, so handlers must be idempotent. Committing an offset commits everything before it,
which makes naive concurrency within a partition lose messages. A new group starts at
`latest` in the Java client and at the first retained offset in kafka-go. Failures must be
classified. Permanent ones go to a dead-letter topic carrying the original coordinates,
payload and reason. Transient ones should pause the consumer rather than flood the DLQ. The
commit must wait for the DLQ write to be confirmed. Publishing an event from a request
handler after a database write is a dual write with no safe ordering. A background
goroutine needs a detached, bounded context, and still loses events on shutdown. kafka-go's
zero value for `RequiredAcks` doesn't wait for the broker at all. The reliable answer is a
transactional outbox: write the event to a table in the same transaction and let a relay
publish it, at least once. Consumers that buffer records across several polls break the
condition auto-commit relies on, finishing each poll before the next, so automatic commits
(on by default in franz-go, every five seconds, committing what was previously polled) can
commit records still sitting in the buffer: measured, 100 buffered, none processed, offset 100
committed. Batching consumers must disable autocommit, commit after each batch, and commit or
discard at revocation, because rebalances happen underneath a running batch. Cooperative
rebalancing shortens pauses but doesn't remove those duplicates, and a commit made with an
already-expired context fails, so every window's last batch runs twice.

## How to use me

Ask me things like:

- "Here's our producer and consumer config. What can go wrong?"
- "Does this consumer survive a rebalance mid-batch without duplicates or loss?"
- "Should this event go through an outbox?"
- "How should this consumer handle a downstream outage?"
- "We're adding a second producer to this topic in another language. What do we check?"
