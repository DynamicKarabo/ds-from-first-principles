# Chapter 10: Replication and Consistency

Consensus gives us a log that multiple nodes agree on. The CAP theorem forces us to choose what to sacrifice during a partition. This chapter examines the mechanisms for keeping data synchronised across replicas — and the consistency guarantees those mechanisms provide.

Replication is the bridge between "we have an agreed log on one node" and "we have consistent state across N nodes."

---

## 10.1 Why Replicate

You replicate data for three reasons, each following from the Axioms:

| Reason | Axiom | Purpose |
|--------|-------|---------|
| **Fault tolerance** | Axiom 2 (partial failure) | If one node dies, another has the data |
| **Performance** | Axiom 1 (asynchrony) | Serve reads from geographically closer nodes |
| **Throughput** | Finite resources (Ch 1) | Distribute read load across multiple nodes |

Fault tolerance is the primary reason — it is the direct consequence of Axiom 2. If nodes could not fail, or if losing a node's data were acceptable, replication would be optional. Since nodes fail and data must survive, replication is mandatory.

> **Model reminder:** The discussion of replication in this chapter, and of transactions in Chapter 11, continues to assume the **crash-fault model** introduced in Chapter 7 — nodes may stop responding but they will not lie, forge messages, or collude. This assumption is the foundation of all the consistency guarantees and replication strategies described here. The Byzantine case (where nodes can behave maliciously) is examined in Chapter 17, and nearly everything in Chapters 10–11 would need re-examination under that model.

---

## 10.2 Consistency Models

A **consistency model** defines what data a read is allowed to return, given the history of writes. Models are ordered from strongest (least flexible, hardest to implement) to weakest (most flexible, easiest to implement):

### Linearizability (Strongest)

Every read returns the value of the most recent write. The system appears as a single machine — operations take effect atomically, at some point between invocation and response. All replicas agree on the global order of all operations.

**Cost:** Coordination is required for every operation. Throughput is limited by the slowest node. Availability is limited by the CAP theorem (CP systems are linearizable; AP systems are not).

**Examples:** etcd, Zookeeper (sync reads), Spanner, Raft's replicated log.

### Sequential

Each process (client) sees its own operations in program order. Different processes may see different interleavings, as long as each process's view is consistent with some total order of all operations.

Sequential is slightly weaker than linearizable because there is no constraint about *real time* — two operations from different clients can be ordered arbitrarily as long as each client's own order is preserved.

### Causal

Causally related writes are seen in the correct order by all observers. Concurrent (unrelated) writes may be seen in different orders by different observers.

This model respects the happens-before relation (Chapter 5). If operation A causally precedes operation B (A sent a message that B received, or A happened before B on the same client), all observers see A before B. If A and B are concurrent, observers may see them in either order.

**Examples:** CRDTs, DynamoDB (session consistency), COPS (causal consistency system from CMU).

### Eventual (Weakest)

Given enough time without new writes, all replicas will converge to the same value. There is no guarantee about *when* convergence happens, and reads may return stale data arbitrarily long if writes continue.

This is the minimum guarantee for a distributed system. Without eventual consistency, replicas could diverge permanently even in the absence of new writes.

**Examples:** DNS, CDNs, DynamoDB (default, with tuning), Cassandra (with consistency level ONE).

**The consistency-performance tradeoff:**

```
Strength:  Linearizable > Sequential > Causal > Eventual
Latency:  Higher        <          <        < Lower
Available: Lower        <          <        < Higher
```

---

## 10.3 Replication Strategies

### Primary-Backup (Single-Leader Replication)

One node (the primary/leader) accepts writes. It replicates the write log to backup nodes (followers). Followers serve reads but cannot accept writes. If the primary fails, one backup is promoted to primary.

**Strengths:** Simple. Strong consistency (if reads also go to the primary).
**Weaknesses:** Single point of failure during promotion. Writes bottleneck on one node. Read scalability limited by followers.
**Used in:** MySQL (leader-based), PostgreSQL (streaming replication), MongoDB (primary-secondary).

### Chain Replication

Writes go through a chain of replicas: the first replica (head) accepts the write, forwards to the next, which forwards to the next, and so on until the last replica (tail) acknowledges success. Reads are served by the tail.

**Strengths:** High write throughput (linear in chain length). Simple ordering (the chain defines the order). Strong consistency.
**Weaknesses:** High write latency (each write traverses the entire chain). Tail becomes read bottleneck.
**Used in:** Azure Storage (original design), RabbitMQ (quorum queues variant).

### Quorum Replication (Dynamo-Style)

Writes must be acknowledged by W replicas (the write quorum). Reads must be acknowledged by R replicas (the read quorum). The system is consistent if `W + R > N` (the read and write quorums overlap by at least one node).

Tunable parameters:
- `W = N, R = 1`: Optimised for fast reads, expensive writes.
- `W = 1, R = N`: Optimised for fast writes, expensive reads.
- `W = R = QUORUM`: Standard configuration (N/2 + 1). Balanced.

`W + R > N` ensures that every read quorum overlaps every write quorum — at least one node with the latest write is guaranteed to be in the read set.

**Strengths:** Tunable consistency. High availability (no single point of failure). Good for multi-datacenter deployments.
**Weaknesses:** Vector clocks needed for conflict detection. Read-repair for convergence. Complexity of handling sloppy quorums and hinted handoffs.
**Used in:** Amazon DynamoDB, Cassandra, Riak, Voldemort.

### State Machine Replication (SMR)

All replicas start at the same state. They receive the same sequence of operations (the log — Chapter 8) and apply them deterministically. Since each operation is deterministic, all replicas converge on the same state.

SMR is the replication strategy that consensus (Chapter 7) enables directly. Raft's log is an SMR log — all Raft replicas apply the same entries in the same order.

**Strengths:** Strong consistency (linearizability). Well-understood correctness properties.
**Weaknesses:** Leader bottleneck. Throughput limited by one node.
**Used in:** etcd, Zookeeper, Raft-based systems.

---

## 10.4 Partition Handling: CRDTs vs Consensus

After a partition heals, two replicas may have accepted conflicting writes. There are two approaches to reconciliation:

### Consensus (CP approach)

Do not allow writes on both sides of a partition. The minority partition stops accepting writes (Chapter 9 — CP). When the partition heals, no reconciliation is needed — the minority replicas simply catch up on the log from the majority.

**Strength:** Deterministic consistency. No conflict resolution needed.
**Weakness:** Unavailable during partition (minority side cannot accept writes).

### CRDTs (AP approach)

Allow writes on both sides. Use a data structure that converges automatically when writes are merged (Conflict-free Replicated Data Types). Examples:
- **G-Counter (grow-only counter):** Each node tracks its own increments. Merge = max of each node's value.
- **PN-Counter (positive/negative counter):** Two G-Counters — one for increments, one for decrements.
- **LWW-Register (last-writer-wins register):** Each write has a timestamp. Merge = keep the latest timestamp.
- **OR-Set (observed-remove set):** Elements can be added and removed without conflicts.

CRDT properties:
- Merge is commutative (order of merging does not matter).
- Merge is associative (grouping merges does not matter).
- Merge is idempotent (merging the same data multiple times produces the same result).

**Strength:** Available during partition. No coordination needed.
**Weakness:** Business-level conflicts may still require application logic (e.g., two users buying the last item — CRDTs can merge the "items remaining" counter, but cannot decide who gets the item).

---

## 10.5 Read-Repair and Anti-Entropy

In quorum-based replication (Dynamo), stale replicas must be brought up to date:

**Read-repair:** On a read request with `R < N` replicas contacted, the coordinator checks whether any of the contacted replicas have stale data. If so, it updates them with the latest value. This fixes stale data on the read path.

**Anti-entropy (Merkle tree reconciliation):** Periodically, each node exchanges a Merkle tree (hash tree) of its data with a random peer. Any differences are identified and repaired. This fixes stale data that was not reached by read-repair.

Read-repair fixes stale data *when it is read*. Anti-entropy fixes stale data *even if it is never read*. Both are needed for convergence.

---

## 10.6 Exactly-Once Semantics

Exactly-once execution is the holy grail of distributed systems — each operation is processed exactly once, no matter how many times it is sent, no matter how many times the system crashes and retries.

Exactly-once requires three mechanisms working together:

**Idempotency:** An operation that can be applied multiple times with the same result. If a payment of $10 is idempotent, retrying it 100 times still deducts $10 once. Achieved by assigning each operation a unique ID and checking for duplicates.

**Deduplication:** Before processing an operation, check if its ID has already been processed. If yes, return the result of the first execution. If no, execute and store the result.

**Atomic commit (Chapter 11):** An operation that spans multiple nodes must commit or abort on all of them. Without atomic commit, an operation could succeed on one node and fail on another, producing partial execution.

Exactly-once is expensive. Most practical systems guarantee **at-least-once** (operations may be retried, but never lost) and rely on idempotency to absorb duplicates. True exactly-once is reserved for systems where duplicate execution has material consequences (financial transactions, inventory allocation).

---

## 10.7 Design-Space Beats

**Beat 1: Strong consistency vs availability as a product decision.**

The choice between linearizable and eventual consistency is not a technical decision. It is a product decision. The question is not "which consistency model is possible?" but "what happens when the user sees stale data?"

- For a banking app: stale balance → user disputes transactions → operational cost.
- For a social feed: stale timeline → user refreshes → minor annoyance.
- For a CDN: stale page → user might not notice.

The right consistency model is the one that minimises harm when staleness occurs, balanced against the availability and latency cost of preventing it.

**Beat 2: Write-path vs read-path latency — quorum size tradeoffs.**

Choosing `W` and `R` in a quorum system trades latency on one path for latency on the other:

| Configuration | Write Latency | Read Latency | Read Freshness |
|-------------|--------------|-------------|----------------|
| W=1, R=N | Fastest writes | Slowest reads | Guaranteed fresh |
| W=N, R=1 | Slowest writes | Fastest reads | Guaranteed fresh |
| W=QUORUM, R=QUORUM | Balanced | Balanced | Guaranteed fresh |
| W=1, R=1 | Fastest both | Fastest both | Possibly stale |

The `W + R > N` rule ensures consistency regardless of the specific values, but the latency characteristics differ dramatically. A system optimised for writes (W=1, R=N) is very different from one optimised for reads (W=N, R=1).

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Why replicate | Fault tolerance (Axiom 2), performance (Axiom 1), throughput |
| Consistency models | Linearizable → Sequential → Causal → Eventual (ordered by strength) |
| Replication strategies | Primary-backup, chain, quorum (Dynamo), SMR |
| Partition reconciliation | Consensus (CP) vs CRDTs (AP) — two different philosophies |
| Read-repair / anti-entropy | Passive vs active reconciliation in quorum systems |
| Exactly-once | Idempotency + dedup + atomic commit — expensive but correct |

**Next:** Replicated state is the foundation. But operations that span multiple items (transactions) require coordination beyond simple replication. Chapter 11 — Distributed Transactions.
