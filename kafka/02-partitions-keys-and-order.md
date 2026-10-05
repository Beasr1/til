# 2. Partitions, keys and order

## The problem

Two events about the same customer, "details changed" then "re-verified", are produced
in that order and consumed in the opposite order. The consumer marks the customer
current, then stale, and the record is wrong until somebody touches it again. Kafka
didn't lose anything and didn't break any promise. It promised much less about order
than the code assumed. This chapter is about exactly what it promises, and the
producer-side setting that decides whether you get even that.

## A topic is a set of logs

A **topic** is split into **partitions**, and each partition is an append-only log:
every message gets the next **offset** in that partition, and consumers read it in
offset order. That's the whole ordering guarantee. The current Kafka documentation
([Introduction](https://kafka.apache.org/42/getting-started/introduction/)):

> Kafka guarantees that any consumer of a given topic-partition will always read that
> partition's events in exactly the same order as they were written.

Older editions put the limit more bluntly
([0.9 Introduction](https://kafka.apache.org/090/getting-started/introduction/)):

> Kafka only provides a total order over records within a partition, not between
> different partitions in a topic.

```
topic "customer-events", 3 partitions

partition 0:  [0] A:changed   [1] C:verified   [2] A:verified   →
partition 1:  [0] B:changed   [1] B:verified                    →
partition 2:  [0] D:changed                                     →
```

A consumer of partition 0 is guaranteed to see `A:changed` before `A:verified`. Nothing
relates the position of anything in partition 0 to anything in partition 1. Partitions
are consumed in parallel, by different consumers or different threads, at different
speeds.

So "events for one customer arrive in order" holds only if **every event for that
customer goes to the same partition**. That's what the message key is for.

## The key chooses the partition, if the producer uses it

A message has an optional **key**. The producer's **partitioner** (or **balancer**)
decides which partition each message goes to, and the common convention is: hash the key,
take it modulo the number of partitions. Same key, same partition, ordered.

The Kafka documentation describes this as if it were a rule: "Events with the same event
key (e.g., a customer or vehicle ID) are written to the same partition". That sentence
describes the official Java client. It isn't enforced by the broker, which stores
messages in whichever partition the producer names. Which partition the producer names
is decided by the *client library*, and clients disagree:

| Client | Default for a keyed message | Source |
|---|---|---|
| Java client (and librdkafka's `murmur2` setting) | Hash of the key (murmur2) | kafka-go's `Murmur2Balancer` is documented as "compatible with the partitioner used by the Java library" |
| kafka-go `Writer` | **Round-robin. The key is ignored** | kafka-go `writer.go`: "The default is to use a round-robin distribution" |
| kafka-go `LeastBytes` | **Picks the partition that has received the fewest bytes. The key is ignored** | kafka-go `balancer.go`: the `Balance` method never reads `msg.Key` |
| kafka-go `Hash` | FNV-1a hash of the key | kafka-go `balancer.go`: "the same algorithm used by the Sarama Producer" |

So a Go service can carefully build a key such as `customerID + "_" + tenantID`, set it
on every message, configure `Balancer: &kafka.LeastBytes{}` because that's what an
example used, and get **no per-key ordering at all**. Everything compiles, every message
is delivered, and events for one customer are spread across all partitions. It's
invisible in testing with one partition, because with one partition everything is
ordered.

> ⚠️ **The trap: two producers, two hash functions.** Even with hashing on, the Java
> client's murmur2 and kafka-go's default `Hash` (FNV-1a) send the same key to
> *different* partitions. If a Java service and a Go service both produce events for
> customer A to one topic, A's events land in two partitions and have no relative order.
> Pick one partitioner for the topic, and in Go use `Murmur2Balancer` if Java producers
> share the topic.

## Changing the partition count moves keys

Hash modulo partition count means adding partitions changes where most keys go. A key
that hashed to partition 2 of 3 may go to partition 5 of 6. Events produced before the
change sit in the old partition and events produced after it in the new one, and a
consumer can process the newer ones first. If per-key order matters, choose the
partition count up front, generously, and treat increasing it as a migration.

## No order across topics, or across producers

Even with perfect keying, order exists only *within one partition of one topic*. Two
common designs quietly assume more:

**Two topics.** Service X publishes "customer details changed" to topic T1. Service Y
publishes "customer re-verified" to topic T2. A consumer reads both. Nothing orders a T1
message relative to a T2 message, not even when they share a key, so the consumer can
see "re-verified" before the "changed" that it supersedes.

**Two producers, one partition.** Two service instances each publish an event for the
same key, at nearly the same time. The partition records them in the order they reached
the broker, which isn't necessarily the order things happened in the instances.

The way out is the same in both cases: **stop relying on arrival order and compare the
events' own timestamps or versions.** Put the time the change *happened* (or a
monotonic version) in the event, and make the consumer's update conditional on it:

```sql
-- resolve only staleness that happened before the verification
UPDATE staleness
   SET resolved = true, resolved_by = $txn
 WHERE customer_id = $1 AND resolved = false
   AND detected_at <= $verified_at;
```

Without the last line, a "re-verified" event that arrives late resolves a staleness that
was detected *after* the verification, and the customer looks current when they aren't.

> **Teacher's aside.** "Kafka guarantees ordering" is the most common sentence people
> half-remember about Kafka. The real guarantee is *per partition*, and getting per-key
> order out of it depends on a client setting most people never look at. Across topics
> there's no ordering at all. A design that needs a cross-topic order needs the order
> written into the data, not inferred from arrival.

## Check yourself

1. A Go service sets a key on every message and uses `&kafka.LeastBytes{}`. The topic has
   six partitions. Where do two consecutive messages with the same key go, and what does
   a consumer see?
2. Why does every ordering bug in this chapter disappear in a local test with a
   one-partition topic?
3. A topic is produced to by a Java service and a Go service using kafka-go's `Hash`
   balancer. Both key by customer id. What breaks, and what's the fix?
4. You grow a topic from 6 to 12 partitions on a busy day. Describe one customer's events
   around the change, and what a consumer might do wrong.
5. A consumer reads "detected stale" from one topic and "verified" from another. Write
   the condition that makes "verified" resolve only the staleness it actually supersedes,
   and say what field each event must carry.
