# Chapter 8: The Log

Consensus gives us something deceptively simple: **an agreed total order of events.** A sequence of entries that every correct node has seen, in the same order. This is the **log.**

The log is the most important abstraction in distributed systems. It recurs across every remaining layer of this course — replication, transactions, streaming, and event sourcing all build on the idea of an ordered, durable, agreed-upon sequence of entries.

---

## 8.1 What Consensus Produces

When Raft reaches consensus on a set of values, the result is a sequence of committed entries on each node. This sequence is:

- **Ordered:** Entries appear in the order they were committed. Entry index 5 comes before entry index 6, and every node agrees on this.
- **Durable:** Once an entry is committed, it will not be lost. It persists on a majority of nodes.
- **Deterministic:** Given the same sequence of entries, a deterministic process produces the same state on every replica.

This is a **log** — an append-only, totally ordered sequence of entries. The log is the output of consensus. It is also the input to everything else.

---

## 8.2 The Log Abstraction

A log is defined by three operations:

```
append(entry) → position    // Add an entry. Returns its position.
read(position) → entry      // Read the entry at a position.
trim(position)              // Entries before this point may be discarded.
```

Properties:
- Entries are immutable once committed. You cannot modify an entry in place.
- The log grows in one direction (append-only).
- Any node with the log can reconstruct the same state by replaying entries from position 0.

This last property is the most important. If you define state as `f(log)` — the result of applying every entry in the log in order to an initial state — then any node with the same log has the same state. This is the foundation of **state machine replication** (Chapter 10).

---

## 8.3 The Log is Not New

The log predates distributed systems by decades. It appears in:

- **Write-ahead logging (WAL) in databases:** A database writes intended changes to a log before applying them to the data files. If the database crashes mid-write, the log is replayed on recovery to restore a consistent state. This pattern (write to log first, then to state) is the origin of the term "write-ahead" and appears in PostgreSQL's WAL, MySQL's binlog, SQLite's journal, and every modern database.

- **Filesystem journalling:** ext3, ext4, XFS, and NTFS all use a journal (log) for crash recovery. The filesystem writes intended metadata changes to the journal before applying them. On recovery, the journal is replayed.

- **Event sourcing:** Store the log as the primary source of truth. Current state is derived by replaying events. This inverts the traditional database model — the log is the database, not a recovery mechanism.

- **Kafka:** A distributed log as a product (Chapter 14). Kafka is essentially the WAL abstraction extracted, generalised, and scaled horizontally.

Distributed consensus did not invent the log. It provided the missing piece: letting **multiple nodes agree on the same log.** The log itself is a concept that's been solving consistency problems for decades.

---

## 8.4 From Consensus to Replication

Given a log that multiple nodes agree on, you can build **state machine replication**:

1. Start all replicas at the same initial state.
2. Feed each replica the same sequence of entries from the log.
3. Each replica applies the entries deterministically.
4. All replicas converge on the same state.

If one replica crashes and recovers, it replays the log from the last known committed entry. If a new replica joins, it receives the full log from an existing replica. If a replica is suspected dead (Chapter 6), a different replica takes over using the same log.

This is how distributed databases, consensus-based systems, and replicated state machines work. The log is the medium of agreement. The replicas are deterministic functions of the log.

---

## 8.5 The Log as a Bridge

The log connects the concepts in this course:

| Layer | The Log Appears As |
|-------|-------------------|
| 7 (Consensus) | Raft's replicated log — entries committed by majority |
| 10 (Replication) | SMR replays the log on each replica |
| 11 (Transactions) | Write-ahead log for atomicity — log of intents |
| 14 (Streaming) | Kafka = distributed log as a product |
| 14 (Event Sourcing) | The log as the source of truth — state is derived |

After this chapter, the course alternates between:
- **Things built on the log** (replication, transactions, streaming)
- **Constraints that exist because of the log** (CAP, partitioning, consistency tradeoffs)

The log is the thread. Once you understand that consensus produces an agreed log, and that systems built on that log inherit its properties (ordered, durable, deterministic), the relationships between the remaining topics become clearer.

---

## 8.6 The Limits of a Single Log

A single log is not a complete distributed system. It has limits:

**Scalability.** A single log handled by a single leader (Raft) cannot handle more throughput than one node can process. To scale beyond a single leader, you must either partition (split the log across multiple shards — Chapter 12) or relax the total ordering guarantee.

**Availability.** The log is unavailable during leader elections (Chapter 7). If the leader is partitioned away from a majority, the log cannot accept writes until a new leader is elected. This is a direct consequence of the CAP theorem (Chapter 9).

**Size.** The log grows indefinitely as entries are appended. Practical systems trim old entries (snapshotting) or delete them entirely. The log is a logical abstraction; the physical storage must handle bounded size.

**Global ordering cost.** Total ordering (every entry goes through a single leader) is expensive. Many workloads do not need total ordering — they need ordering *per partition* (Kafka's model), ordering *per key* (causally), or no ordering at all (eventual consistency). The log is a tool, not a requirement.

---

## 8.7 Design-Space Beat

**Log-based vs traditional DB:** A log-based system (Kafka, event sourcing) makes the log the primary storage. A traditional database makes the current state the primary storage, with the log as a recovery mechanism. The choice is about which failure you optimise for:

- **Log as primary:** Optimises for recovery and replay. You can always reconstruct state from the log. But querying state requires replaying the log (or maintaining materialised views).
- **State as primary:** Optimises for query performance. Current state is immediately available. But recovery from failure requires the log to ensure consistency (WAL).

Neither is universally correct. The choice depends on whether your workload is dominated by writes (log as primary handles this well) or reads (state as primary handles this well).

**The Journal as Design Pattern:** The log is so fundamental that many system problems reduce to "how do we arrange the journal?" A transaction system without isolation is just a log with validation. A queue is a log with a consumer offset. A replicated database is a log with subscribers. Understanding the log deeply — not just as a data structure but as a design pattern — is one of the most transferable skills in distributed systems.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| What consensus produces | An agreed total order of events — a log |
| Log abstraction | Append-only, ordered, durable, immutable entries |
| Log predates DS | Write-ahead logging, filesystem journalling, event sourcing |
| State machine replication | All replicas apply the same log → same state |
| Limits of a single log | Scalability, availability, size, global ordering cost |
| Log as design pattern | Recurring solution across databases, queues, replication, streaming |

**Next:** The log gives us agreement, but it also forces a choice. During a network partition, you cannot both maintain total order (the log) and remain available. This choice is the CAP theorem — Chapter 9.
