# 6. Batching consumers, commits and rebalances

## The problem

Processing one message at a time is slow when each message ends in a database write, so you
batch: poll, keep the records in a buffer, and process the buffer when it reaches a size or a
timer fires. It's the right design, and it quietly breaks two assumptions from
[chapter 3](03-offsets-and-delivery-guarantees.md). The client's automatic commit was designed
for consumers that finish each poll before the next, so with a buffer it can commit records
you haven't processed. And a rebalance can move a partition away while its records sit in
your buffer. This chapter is about what the client does in those moments, and what you have to
do instead.

Assumes chapters 1 and 3. Client behaviour is quoted from the Java consumer's documentation
and from the source of the Go client `twmb/franz-go` (v1.19.5, `pkg/kgo/config.go`,
`consumer.go`, `consumer_group.go`). Measurements are from a single-broker Kafka 4.3 in a
throwaway container, with franz-go v1.19.5.

## The shape of a batching consumer

```go
for {
    select {
    case <-ctx.Done():
        return                              // buffered records: neither processed nor committed
    case <-ticker.C:
        if len(buf) > 0 { process(buf); commit(); buf = nil }
    default:
        pollCtx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
        fetches := client.PollFetches(pollCtx)   // short timeout so the ticker gets a turn
        cancel()
        fetches.EachRecord(func(r *kgo.Record) { buf = append(buf, r) })
        if len(buf) >= minBatch { process(buf); commit(); buf = nil }
    }
}
```

The buffer spans **many polls**. That one fact is what the rest of the chapter is about.

## Automatic commit assumes you finish each poll

The Java consumer's documentation
([KafkaConsumer](https://kafka.apache.org/43/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html))
states the condition under which auto-commit is safe:

> Using automatic offset commits can also give you "at-least-once" delivery, but the
> requirement is that you must consume all data returned from each call to poll(Duration)
> before any subsequent calls, or before closing the consumer. If you fail to do either of
> these, it is possible for the committed offset to get ahead of the consumed position, which
> results in missing records.

A batching consumer fails that requirement by design: it calls poll again with the previous
poll's records still in the buffer.

franz-go has autocommit on by default, every five seconds (`AutoCommitInterval`: "overriding
the default 5s"). By default it isn't greedy. It commits what has *previously* been polled,
not the most recent poll: `GreedyAutoCommit` is described as committing "everything that has
been polled when autocommitting (the dirty offsets), rather than committing what has
previously been polled". So it protects you from losing the latest poll, and nothing older.
With a buffer spanning many polls, the older ones are exactly what's at risk.

Measured: 100 records produced; a consumer polls for 12 seconds, buffers everything,
processes nothing (its batch threshold is never reached), then stops without closing:

```
mode=autocommit buffered=100 processed=0 committed_offset_after_crash=100
mode=manual     buffered=100 processed=0 committed_offset_after_crash=-1
```

With autocommit left on, the group's committed offset reached 100. A replacement consumer
would start after every one of those records, and none of them was ever processed. With
`DisableAutoCommit()`, nothing was committed, and a replacement would read all 100 again.

> ⚠️ **Calling a manual commit doesn't switch automatic commit off.** A consumer that calls
> `CommitUncommittedOffsets` after each batch, but never passed `DisableAutoCommit()`, is
> running both. The manual commit gives at-least-once for each batch, and the background
> commit quietly turns the buffer into at-most-once. It's an easy configuration to have for
> months, because nothing fails until a process dies with a full buffer.

## Rebalances happen while you process

In franz-go, group membership runs independently of your loop. From the `BlockRebalanceOnPoll`
documentation: "By default, a consumer group is managed completely independently of
consuming. A rebalance may occur at any moment. If you poll records, and then a rebalance
happens, and then you commit, you may be committing to partitions you no longer own. This will
result in duplicates."

The `DisableAutoCommit` documentation walks through the case: you've committed offset 4 and
processed to 30; the partition moves to member B; without a commit at revocation B starts at 4
and reprocesses 4–29. "Worse, you, member A, can rewind member B's commit". Its recommendation
for manual committers:

> If you are committing offsets manually (have disabled autocommitting), it is highly
> recommended to do a proper blocking commit in OnPartitionsRevoked.

Your options, and what each costs:

| Approach | What happens at a rebalance | Cost |
|---|---|---|
| Do nothing special | Records polled for a revoked partition may be processed twice, by you and the new owner | Duplicates; handlers must be idempotent (chapter 3) |
| `OnPartitionsRevoked` that processes or drops that partition's buffered records, then commits | New owner starts where you stopped | Callback must finish within the rebalance timeout (default 60s) |
| `BlockRebalanceOnPoll` + `AllowRebalance` after each batch | Rebalances that lose partitions wait until you allow them | A slow batch delays the whole group; "the big tradeoff is that by blocking rebalances, you put your group member at risk of waiting so long that the group member is kicked from the group because it exceeded the rebalance timeout" |

## Eager and cooperative rebalancing

In the classic **eager** protocol, every member gives up all its partitions at the start of a
rebalance and gets an assignment back at the end: a stop-the-world pause for the whole group.
[KIP-429](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol)
added **incremental cooperative** rebalancing: only the partitions that move are revoked, in a
follow-up rebalance, and sticky assignment keeps the rest where they were. In franz-go that's
`kgo.Balancers(kgo.CooperativeStickyBalancer())`.

Cooperative rebalancing shortens pauses during scaling and rolling deploys. It doesn't change
the commit rules above, and it doesn't mean "no duplicates": a partition that does move still
moves with whatever you'd polled but not committed.

> **Teacher's aside.** A common justification for switching balancers reads: "with eager
> rebalancing, our long batch blocks the rebalance until it finishes, the coordinator times us
> out, and we're ejected". That's a description of `BlockRebalanceOnPoll`, not of eager
> rebalancing. Without that option, franz-go rebalances underneath a running batch whichever
> protocol you use, and the cost is duplicate processing, not ejection. Be sceptical of
> comments that put a percentage on the duplicates. The number depends on batch size, commit
> frequency and how often the group rebalances, so it isn't a property of the protocol.

## Commit with a context that can still succeed

Batching consumers often run inside a deadline: a scheduled job that consumes for an
hour, or a process shutting down. The commit at the end of a batch then uses the same
context, which may already have expired. Measured: a consumer whose run window was a
three-second context processed past the window, then committed with that context:

```
mode=expired polled=100 commit_err=context deadline exceeded
committed_offset=-1
```

The batch was processed and the commit failed, so the next run processes it again. It's safe
under at-least-once, but every run's last batch is duplicated, and if the batch takes longer
than the window, *every* batch is. Give the commit its own short, detached deadline
(`context.WithTimeout(context.WithoutCancel(ctx), 5*time.Second)`), and stop *taking new
records* when the window closes, not stop committing.

## Errors inside the batch

A batch handler that fans work out to goroutines usually collects errors. If it logs them and
returns success, the loop commits the batch, and the failed records are gone: chapter 3's
"committing regardless of outcome" without the dead-letter step. The pattern, and the worker
pool shape that hides it, is in [go-services/09](../go-services/09-worker-pools.md). The rule
for this chapter: the batch is committed only after every record in it has either succeeded or
been written, confirmed, to a dead-letter topic.

## Check yourself

1. A consumer buffers records across polls and commits manually after each batch, but never
   disables autocommit. Its batch threshold is high and traffic is low. Describe what's
   committed after a minute, and what a crash loses.
2. Why is franz-go's non-greedy autocommit safe for a consumer that processes each poll before
   the next, and not for one that buffers?
3. With autocommit disabled and no `OnPartitionsRevoked`, partition 2 moves from A to B in the
   middle of A's batch. A then commits. Walk through what B processes, and how A's commit can
   make it worse.
4. A team switches from eager to cooperative rebalancing to "stop duplicates during deploys".
   What does the switch improve, and what doesn't it change?
5. A scheduled consumer runs for an hour under one context. Why might its last batch be
   processed twice every run, and what's the fix?
6. You add `BlockRebalanceOnPoll` so commits are always for partitions you own. Batches take
   ninety seconds. What happens when a new member joins?
