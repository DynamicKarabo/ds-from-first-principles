# Chapter 14: Streaming and the Log

The log (Chapter 8) is what consensus produces — an agreed total order of events. This chapter examines what happens when you treat the log itself as the product, not just a coordination mechanism.

---

## 14.1 The Log as a Product

In most distributed systems, the log is infrastructure. It's the write-ahead log of a database, the consensus log of Raft, the journal of a filesystem. It's a means to an end — consistency, recovery, replication.

In streaming systems, the log is the end. The log is the data. Consumers read from the log, process entries, and produce results. Producers write to the log. The log is the source of truth.

This shift — from log-as-mechanism to log-as-product — is the foundation of event streaming, event sourcing, and the data pipeline architectures that dominate modern data infrastructure.

---

## 14.2 Kafka: A Distributed Log

Apache Kafka is the canonical implementation of the log-as-product. It is best understood not as a message queue (which is how it is often marketed) but as a **distributed, partitioned, replicated commit log.**

Kafka's architecture:

- **Topic:** A named log. Entries are appended and read sequentially.
- **Partition:** A topic is split into one or more partitions. Each partition is an ordered, immutable sequence of entries. Partitions are the unit of parallelism — one partition is read by one consumer in a consumer group.
- **Producer:** Writes entries to a topic. Controls which partition each entry goes to (by key hash or explicit assignment).
- **Consumer:** Reads entries from a topic. Maintains an **offset** — the position in the partition log that has been read so far.
- **Broker:** A Kafka server that stores partitions and serves producer/consumer requests.
- **Replication:** Each partition is replicated across multiple brokers (configurable replication factor). One broker is the leader for each partition; others are followers. Kafka uses a Raft-derived consensus (KRaft) for metadata management.

**Key design decisions:**

1. **Partitioning as scale (L12).** Kafka splits topics into partitions across brokers. Each partition is independent — it has its own order, its own leader, its own throughput. This is data partitioning applied to the log abstraction.

2. **Immutable log.** Kafka does not delete entries when consumed. Entries are retained for a configurable period (days to weeks) or until the log reaches a size threshold. Consumers can re-read old entries by seeking to an earlier offset.

3. **Offset-based consumption.** Consumers track their position (offset) in each partition. If a consumer crashes and restarts, it resumes from its last committed offset. This provides at-least-once delivery semantics.

4. **Consumer groups.** Multiple consumers can coordinate to read a topic. Each partition is assigned to exactly one consumer in the group. This enables horizontal scaling of consumption.

### Kafka vs Traditional Message Queues

| Property | Traditional Queue (RabbitMQ, SQS) | Kafka |
|----------|---------------------------------|-------|
| Model | Queue (consume and delete) | Log (retain and replay) |
| Throughput | Thousands to tens of thousands/sec | Hundreds of thousands to millions/sec |
| Ordering | Per-queue (bounded) | Per-partition (strong) |
| Replay | Not supported | Supported (offset seek) |
| Message retention | Until acknowledged | Configurable time/size |
| Consumer state | Broker-managed (queue) | Consumer-managed (offset) |

---

## 14.3 Event Sourcing

Event sourcing inverts the traditional data model:

- **Traditional database:** Store current state. The log (WAL) is a recovery mechanism.
- **Event sourcing:** Store the log (sequence of events). Current state is derived by replaying events.

If you want to know a customer's current address in a traditional database, you read the `customers` table. In event sourcing, you replay all `AddressChanged` events for that customer and apply them in order.

Event sourcing strengths:
- **Temporal query:** You know the state at any point in time (replay events up to time T).
- **Causality preservation:** The log is append-only; events cannot be modified or deleted.
- **Audit trail:** Every state change is recorded. No "who changed this value and when?" question.
- **Multiple projections:** Different consumers can derive different state representations from the same event stream (e.g., one consumer builds a relational view, another builds a search index).

Event sourcing weakness:
- **Current state requires computation.** Query latency is proportional to the number of events replayed. Mitigated by **snapshots** (periodically store derived state and start replay from the snapshot).
- **Schema evolution is harder.** Events are immutable; changing their structure requires new event versions.
- **Deletion is hard.** Complying with data deletion regulations (GDPR) requires either rewriting the log or maintaining a tombstone filter.

---

## 14.4 Stream Processing

Stream processing operates on the log in real time. Instead of running a batch job on yesterday's data, the processor reads each log entry as it arrives and produces results immediately.

**Flink, Kafka Streams, Spark Streaming** are the dominant stream processors. The core abstraction:

```
input_stream → operator → output_stream
```

Each operator (map, filter, join, aggregate) reads from one or more input streams and writes to one or more output streams. Operators maintain state (counts, sums, windows) that is updated as each entry arrives.

The challenge of stream processing: **exactly-once semantics in a streaming context.** The same entry may appear twice (producer retry, consumer recovery). The stream processor must deduplicate or handle the duplicate idempotently. This returns us to the exactly-once patterns from Chapter 11.

---

## 14.5 The Log as Through-Line

The log recurs through the course:

| Chapter | The Log's Role |
|---------|---------------|
| 7 (Consensus) | Raft's replicated log — total order of committed entries |
| 8 (The Log) | Introduced as the unifying abstraction |
| 10 (Replication) | SMR replays the log; WAL for atomicity |
| 11 (Transactions) | Write-ahead log for atomic commit |
| 14 (Streaming) | Kafka = distributed log as a product; event sourcing |

The insight: the log is a single idea that appears across consensus, replication, transactions, and streaming because it solves a single fundamental problem — **capturing a sequence of events that can be replayed, replicated, and agreed upon.**

---

## 14.6 Design-Space Beat

**Log-based vs traditional database architecture:**

| Dimension | Log-Based (Kafka/Event Sourcing) | Database-Centric |
|-----------|--------------------------------|------------------|
| Primary storage | The log (append-only) | Current state (mutated in place) |
| Query pattern | Replay events to compute state | Direct state lookups |
| Write throughput | Very high (append only) | Limited by index/buffer management |
| Read latency | High for uncached queries | Low (indexed) |
| Temporal queries | Native (replay to any point) | Requires history tables or CDC |
| Schema flexibility | High (add event types freely) | Requires migrations |

The choice is workload-dependent. Applications that need high write throughput, temporal queries, and multiple consumers benefit from log-based architecture. Applications that need low-latency point reads and complex queries benefit from traditional databases. The two are increasingly combined — CDC (Change Data Capture) pipes database changes into a log-based streaming system, giving you both fresh state and temporal capabilities.

**Sharpening the decision:** Event sourcing is frequently over-applied. Use it when you genuinely need temporal queries (audit logs, compliance, state reconstruction at any point) or multiple divergent projections from the same event stream. Avoid it when your primary workload is point reads of current state — a traditional database with a CDC pipeline to a streaming system handles both cases without forcing event sourcing's replay complexity on every query.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Log as product | The log is the data, not just a coordination mechanism |
| Kafka | Distributed partitioned log — high throughput, replay, retention |
| Event sourcing | Store events, derive state. Temporal queries, audit trail, projections. |
| Stream processing | Operators on real-time event streams — Flink, Kafka Streams |
| Log as through-line | Recurs across consensus, replication, transactions, streaming |

**Next:** Logs produce data. But how do you know the system is healthy? Chapter 15 — Observability.
