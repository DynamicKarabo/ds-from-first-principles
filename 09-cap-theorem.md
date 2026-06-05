# Chapter 9: CAP Theorem

The CAP theorem is the most famous result in distributed systems. It is also the most misunderstood. This chapter derives CAP from first principles, shows what it actually means, and — crucially — shows what it does *not* mean.

---

## 9.1 The Problem

You have a distributed system. The network partitions — some nodes cannot reach others (Axiom 2). Both sides of the partition continue running.

A client makes a request to a node on one side of the partition. This node has data that may be outdated (the other side may have received updates it hasn't propagated). The node has two options:

1. **Respond immediately** with whatever data it has locally. If the data is stale, the client receives an incorrect answer. This is **availability** — the system responds, but potentially with stale data.

2. **Refuse to respond** until it can verify that its data is current by contacting the other side. Since the partition prevents this contact, it must wait indefinitely. The client eventually times out. This is **consistency** — the system does not respond until it can guarantee a correct answer.

**This is the CAP tradeoff.** During a network partition, you cannot simultaneously answer immediately AND answer correctly. You must choose: availability (answer immediately, possibly stale) or consistency (don't answer until you can guarantee correctness).

---

## 9.2 The Derivation

The CAP theorem (Brewer 2000, proven by Gilbert and Lynch 2002) states that a distributed system can guarantee at most two of three properties:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes see the same data at the same time (linearizability — see Chapter 10).

- **Availability (A):** Every non-failing node responds to every request within a reasonable time. The system does not reject or delay requests.

- **Partition tolerance (P):** The system continues operating despite arbitrary message loss or network failures.

The theorem is often presented as "you can only pick two of CP, CA, AP." This is misleading. **You cannot avoid partition tolerance.** The network *will* partition (Axiom 2 — entropy degrades switches, cables, and routers). Partition tolerance is not optional.

The real choice is:

> During a partition, do you sacrifice consistency or availability?

**CP (Consistency + Partition tolerance):** The system refuses to serve requests from the minority partition until the partition heals. etcd, Zookeeper, HBase, MongoDB (default configuration).

**AP (Availability + Partition tolerance):** The system continues serving requests from all partitions, accepting that data may be stale. Cassandra, DynamoDB, Riak, CouchDB.

There is no CA (Consistency + Availability) in practice — it would require a network that never partitions, which does not exist.

---

## 9.3 PACELC Extension

The CAP theorem addresses what happens *during a partition*. But most of the time, there is no partition. What tradeoff exists when the network is healthy?

The **PACELC** extension (Abadi 2010) captures this:

> If partitioned (P), choose between availability (A) and consistency (C).
> Else (E), choose between latency (L) and consistency (C).

Even during normal operation, there is a tradeoff:
- Choosing consistency means you must synchronise with other nodes before responding. This adds latency.
- Choosing low latency means you may serve stale data (weaker consistency).

This is the tradeoff that most systems face most of the time — not the dramatic partition scenario, but the everyday decision of whether to wait for confirmation or respond immediately.

---

## 9.4 What CAP Does NOT Mean

CAP is widely misunderstood. Several common beliefs are wrong:

**Misconception 1: "CP systems are unavailable during partitions."**

In a CP system with 3 nodes, if one node is isolated, the other 2 nodes form a quorum (majority) and continue operating. Only the isolated node becomes unavailable. The system as a whole is available as long as a majority survives. This is not "the whole system is down" — it is "the minority partition cannot serve."

**Misconception 2: "AP systems lose data."**

AP systems do not lose committed data — they serve potentially stale reads. A write to one partition may not have propagated to the other. When the partition heals, the systems reconcile (via read-repair, hinted handoffs, or CRDTs — Chapter 10). Data is reconciled, not lost. The risk is stale *reads*, not lost *writes*.

**Misconception 3: "You choose your system's CAP properties once and they are fixed."**

Many systems support configuration along the C/A axis. Cassandra allows tuning read and write consistency levels per-query (`ONE`, `QUORUM`, `ALL`). The same system can be CP for some operations and AP for others. CAP describes a spectrum, not a binary toggle.

**Misconception 4: "CAP is the only tradeoff that matters."**

CAP tells you what happens during a partition. It does not address:
- Performance under normal operation (latency, throughput)
- Durability (will the data survive a node crash?)
- Operational complexity (how hard is this to run?)
- Cost (how much infrastructure is needed?)

A system that makes the "right" CAP choice can still be wrong for your workload because of performance or operational considerations.

---

## 9.5 Design-Space Beat

**When is AP the wrong choice?**

Any system where serving stale data produces incorrect results:
- Financial systems and banking ledgers (double-spend or incorrect balance)
- Inventory management (selling the same item twice to different customers)
- Distributed locking (claiming a lock is free when it is not)
- Leader election (two nodes both believing they are the leader)

These systems need consistency, even at the cost of availability during partitions.

**When is CP overkill?**

Any system where short-term staleness is acceptable:
- Content delivery networks (a slightly stale page is better than no page)
- Social media feeds (your timeline can be a few seconds behind)
- Product catalogues (a price that updated 30 seconds ago is good enough)
- Monitoring dashboards (a metric that is 10 seconds stale is not a problem)

These systems choose availability because consistency is not worth the availability cost.

**The practical rule:** If you cannot determine which side of the C/A tradeoff your application falls on, it is probably AP-tolerant. Systems that genuinely need CP are rare and usually involve money, safety, or exclusive resource ownership.

---

## 9.6 The Relationship to the Log

The CAP theorem is a direct consequence of the log being the medium of agreement (Chapter 8). During a partition:

- **CP:** The minority partition cannot append to the log (no quorum). It refuses to serve because it cannot guarantee its log position is current.

- **AP:** The minority partition continues appending to its own log. When the partition heals, the logs must be reconciled. Reconciliation is the subject of Chapter 10 (replication and consistency).

The choice between CP and AP is fundamentally a choice about what happens to the log during a partition: stop writing (CP) or keep writing but accept divergence (AP).

---

## Summary

| Concept | Key Idea |
|---------|----------|
| The CAP choice | During partition, choose: respond immediately (AP) or respond correctly (CP) |
| Partition tolerance is not optional | Networks will partition — you cannot opt out |
| PACELC | Even without partitions, there is a latency/consistency tradeoff |
| Common misconceptions | CP ≠ "whole system down." AP ≠ "data loss." CAP is not static. |
| CP when | Stale data produces incorrect results (money, locks, ownership) |
| AP when | Staleness is tolerable and availability matters more |

**Next:** Once we choose where to fall on the C/A spectrum, we need mechanisms for keeping replicas synchronised. This is the subject of Chapter 10 — Replication and Consistency.
