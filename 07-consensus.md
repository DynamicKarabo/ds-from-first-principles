# Chapter 7: Consensus and Agreement

Consensus is the crown jewel of distributed systems — the algorithm that lets multiple machines agree on a value despite failures. It is the foundation upon which reliable distributed storage, coordination, and state machine replication are built.

This chapter traces the problem from its impossibility result through the practical algorithms that solve it in production today.

---

## 7.1 The Agreement Problem

A set of N machines (processes), each starting with an initial value. They must all decide on one value. Some machines may crash. Messages may be lost, delayed, or duplicated. The system must guarantee:

- **Agreement:** No two correct processes decide on different values.
- **Validity:** If a process decides on a value, that value was proposed by some process.
- **Termination:** Every correct process eventually decides on some value.

These three properties define consensus. Without termination, the system could wait forever (safe but useless). Without agreement, the system could reach different decisions on different nodes (split-brain). Without validity, the system could decide on an arbitrary value unrelated to any proposal.

---

## 7.2 FLP Impossibility

In 1985, Fischer, Lynch, and Paterson proved a devastating result: **in an asynchronous system (no bound on message delay), consensus cannot be guaranteed in finite time.** Even with a single faulty process, there is no deterministic algorithm that always reaches consensus.

**Why FLP holds:** A coordinator proposes a value. Some processes acknowledge. Some are slow or dead. The coordinator cannot wait forever (it must eventually decide), but it also cannot know if all processes have received its proposal. If it decides before hearing from all, it risks inconsistency (the slow process might wake up with a different proposal). If it waits, it may wait forever (the slow process might be dead).

The proof works by constructing a chain of states where, no matter how far the protocol progresses, there is always a possible execution path that leads to disagreement. Since the system is asynchronous, the protocol cannot distinguish "the message hasn't arrived yet" from "it will never arrive" — and therefore cannot safely decide.

**What FLP means in practice:** It does not mean consensus is impossible. It means consensus requires *some* relaxation of the asynchronous model. Practical systems use one of two workarounds:

1. **Timing assumptions (failure detectors).** If processes can agree on a bound on message delay (at least probabilistically), they can use timeouts to detect failures and make progress. This is what Raft and Paxos do — they use failure detectors (Chapter 6) to move the system forward when a node appears to be dead.

2. **Randomization.** Instead of deterministic timeouts, use random waits to break symmetry and avoid deadlock. This is how Raft's leader election works (randomized election timeouts between 150–300 ms).

FLP is the reason consensus protocols have timeouts. Without timeouts, the system could wait forever. With timeouts, the system sacrifices perfect asynchrony — it assumes that if a node hasn't responded within the timeout, it is probably dead — but gains termination.

> **A note on "deterministic":** The FLP result uses "deterministic" to mean "an algorithm whose behavior is fully determined by its inputs — no randomness." Raft's leader election escapes this by introducing randomization (randomized election timeouts). Meanwhile, when this chapter describes Raft as having "DETERMINISTIC safety" (Section 7.5.3), it means something different: the safety guarantee is non-probabilistic — a committed entry is guaranteed to remain committed, not "probably committed." Same word, two meanings: FLP's "deterministic" refers to the *protocol's operation*; Raft's "deterministic safety" refers to the *guarantee's certainty.* The two are compatible — Raft uses randomization in its operation (escaping FLP) to provide a deterministic guarantee.

---

## 7.3 Quorum

The foundational mechanism of practical consensus is the **quorum** — a majority of nodes (N/2 + 1) whose agreement is sufficient to commit a decision.

Why a majority? Consider two groups that can both be reachable simultaneously. If each group has a majority of nodes, their memberships must overlap — at least one node belongs to both groups. This overlapping node can detect conflicting decisions.

For a system with N nodes, the quorum size is `⌊N/2⌋ + 1`:

| Nodes (N) | Quorum Size | Failures Tolerated |
|-----------|-------------|-------------------|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |

Key observations:
- **A single node is not a quorum** (unless you have only one node, in which case it's not a distributed system).
- **Even numbers of nodes do not improve fault tolerance** — 4 nodes tolerate the same 1 failure as 3 nodes, at the cost of a larger quorum. This is why consensus systems use odd numbers (3, 5, 7).
- **Two nodes are the worst configuration** — no fault tolerance (if one node dies, the quorum of 2 cannot be reached), but all the complexity of consensus.

---

## 7.4 The Crash-Failure Model

**IMPORTANT: The entire consensus discussion in this chapter assumes a crash-failure model.** Nodes may stop responding, but they will not lie, send conflicting messages, or collude. The Byzantine case (malicious nodes) is a separate axis introduced in Chapter 17 (Security and Trust). Every consensus and replication claim in Chapters 7–14 carries this unstated assumption — it is made explicit here rather than deferred.

Under the crash-failure model:
- A node that fails stops responding permanently (crash-stop) or can recover with its persisted state (crash-recovery).
- A node does not send contradictory messages to different recipients.
- A node does not collude with other nodes to subvert the protocol.
- Network failures can partition the cluster, but nodes on either side of the partition behave correctly within their partition.

These assumptions hold for most practical systems (data centre clusters, managed cloud infrastructure) where nodes are controlled by a single organisation. They do not hold for public blockchains, cross-organisational consensus, or adversarial environments.

---

## 7.5 Raft

Raft is a consensus algorithm designed for understandability. It decomposes consensus into three relatively independent sub-problems, each of which can be understood and verified separately.

### 7.5.1 Leader Election

Raft elects a single leader among the N nodes. The leader is responsible for making all decisions (proposing values, replicating logs). Having a single leader simplifies the protocol significantly — instead of N nodes negotiating every decision, one node proposes and the others confirm.

**Election process:**

1. Each node starts as a **follower**. It expects periodic heartbeats from the leader.
2. If a follower receives no heartbeat within the **election timeout** (randomized between 150–300 ms), it becomes a **candidate** and starts a new election.
3. The candidate votes for itself and sends **RequestVote** messages to all other nodes.
4. A node votes for the candidate with the most up-to-date log (see safety, below).
5. If a candidate receives votes from a **majority** of nodes (including itself), it becomes the leader.
6. If no candidate wins (split vote — two candidates each receiving votes from a subset of nodes), the election times out and a new one begins with fresh random timeout values.

The randomisation of election timeouts is the practical workaround to FLP. It ensures that even if two nodes become candidates simultaneously, their different timeout distributions ensure a single leader is eventually elected with high probability. The probability of perpetual split votes decreases exponentially with each round.

### 7.5.2 Log Replication

Once a leader is elected, all client requests go through the leader:

1. Client sends a request to the leader.
2. Leader appends the request as a new entry in its **log** (Chapter 8).
3. Leader sends **AppendEntries** RPCs to all followers, containing the new entry.
4. A majority of followers acknowledge receiving the entry.
5. The leader commits the entry (considers it final) and responds to the client.
6. The leader notifies followers of the committed entry in subsequent AppendEntries RPCs.

The leader's log is the authoritative source. Followers must accept entries from the leader. If a follower's log differs from the leader's, the leader forces the follower to adopt the leader's version.

**Commitment rule:** An entry is committed when the leader that created it has replicated it to a majority of the cluster. This ensures that any future leader will see the committed entry (because any quorum must overlap with the majority that acknowledged it).

### 7.5.3 Safety

The safety guarantee: **if a log entry is committed on one node, it is committed on all nodes.** No future event can cause a committed entry to be lost or overwritten.

Safety is ensured by the **leader completeness property**: a candidate must have a log at least as up-to-date as a majority of the cluster to become leader. This is implemented by comparing the last log entry's term and index in the RequestVote process:

- If a voter's own log is more up-to-date than the candidate's (higher term, or equal term but longer log), the voter rejects the request.
- This ensures that the winning candidate has all committed entries from previous terms.

The leader completeness property, combined with the majority-replication rule, ensures that a committed entry is present on any node that could become leader — and therefore no future leader can overwrite it.

### 7.5.4 Raft in Summary

Raft's three sub-problems each solve a specific failure mode that physics makes possible:

| Sub-problem | Failure Mode | Solution |
|-------------|-------------|----------|
| Leader election | Leader crashes (entropy / Axiom 2) or becomes unreachable (c / Axiom 1) | Randomized timeouts, majority vote |
| Log replication | Leader can die mid-write (entropy) — some followers got the entry, others didn't | Majority acknowledgement before commit |
| Safety | A new leader might not have all committed entries (c + entropy) | Leader must have most up-to-date log (completeness property) |

Raft is DETERMINISTIC safety. A committed entry is committed, full stop. There is no probabilistic reasoning in Raft's safety guarantees — the leader completeness property, combined with quorum intersection, ensures that committed entries are never lost.

---

## 7.6 etcd: Raft in Production

etcd is a distributed key-value store that uses Raft for consensus. It stores the cluster's state for Kubernetes and other distributed systems.

etcd's configuration recommendations follow directly from quorum arithmetic:
- **1 node:** Not fault-tolerant. Use only for development/testing.
- **3 nodes:** Tolerates 1 failure. Recommended for most deployments.
- **5 nodes:** Tolerates 2 failures. Recommended for production across multiple availability zones.
- **7 nodes:** Maximum practical size. Tolerates 3 failures. The overhead of consensus (more messages, slower elections) outweighs the benefit beyond 7.

A 2-node etcd cluster offers no fault tolerance — if either node fails, the cluster cannot reach a majority and becomes unavailable. Yet 2-node clusters are a common deployment error because "two is more than one" intuitively seems better. The quorum arithmetic tells us it is not.

---

## 7.7 [DEEP DIVE] Raft vs Paxos

The consensus problem was first solved by Leslie Lamport's **Paxos** algorithm (published in 1998, though the work dates to 1989). Paxos was the only consensus algorithm for over a decade. Raft was published in 2014 by Diego Ongaro and John Ousterhout. Today, Raft is significantly more widely deployed.

Why did Raft win?

### Paxos

Paxos is notoriously difficult to understand. Lamport wrote the original paper using an allegory of a parliament on an ancient Greek island — charming for a mathematics audience, impenetrable for engineers. The algorithm itself is minimal and elegant but spread across multiple interacting components (prepare, promise, accept, learned phases; distinguished proposer; multiple overlapping instances for log entries).

Practical implementations of Paxos are famously divergent — Multi-Paxos, Fast Paxos, Cheap Paxos, Stoppable Paxos, each with different tradeoffs. There is no single "Paxos implementation" to follow. Engineers implementing Paxos for the first time almost always get it wrong — the algorithm's simplicity hides subtle edge cases.

### Raft

Raft was designed with a different goal: **understandability.** Ongaro and Ousterhout explicitly set out to create a consensus algorithm that could be taught in a single lecture and implemented correctly by a competent engineer.

Raft achieves this by:

1. **Decomposing consensus into three sub-problems** (leader election, log replication, safety) instead of one unified protocol.

2. **Strong leadership** — one leader makes all decisions, rather than allowing any node to propose. This simplifies almost every aspect of the protocol.

3. **Randomized timeouts** instead of complex failure detector logic for leader election.

4. **A strict log ordering** — the leader forces its log on followers, rather than allowing divergent logs to be reconciled through complex merging.

### Key Differences

| Aspect | Paxos | Raft |
|--------|-------|------|
| Leadership | Multiple proposers (weak leadership) | Single strong leader |
| Log consistency | Flexible (divergent logs allowed, merged later) | Strict (leader forces its log on followers) |
| Election | Leaderless election via prepare/promise rounds | Randomized timeout election |
| Understandability | Notorious for being hard to understand | Designed for teachability |
| Implementation complexity | High — many edge cases | Lower — well-specified state machine |
| Performance | Slightly faster in some edge cases (no leader bottleneck for reads) | Slightly slower in edge case with leader bottleneck |

### The Winning Insight

Raft won because **understandability is an engineering goal, not a nicety.** A protocol that is difficult to understand is difficult to implement, difficult to debug, difficult to extend, and difficult to trust. Raft sacrificed marginal performance and theoretical elegance for comprehensibility — and in doing so, became the consensus algorithm that teams actually deploy.

The tradeoff Raft accepted: **leader bottleneck.** All writes must go through the leader. In a system with a single leader that processes all requests, throughput is limited by that leader's capacity. Paxos (with multiple proposers) can theoretically spread load across nodes. In practice, the simplicity and reliability of Raft's strong-leader model outweighs this limitation for almost all workloads.

**Honest caveat:** Raft is not the "correct" consensus protocol — it is a particular set of tradeoffs that won under particular conditions (crash-fault, non-Byzantine, moderate scale). For Byzantine environments, different protocols (PBFT, HotStuff) are needed. For extremely large clusters, hierarchical consensus (raft groups in a tree) or epidemic protocols may be more appropriate.

---

## 7.8 Design-Space Beat

**Consensus vs 2PC:** A common confusion is equating consensus with two-phase commit (2PC). The critical difference: consensus *lives through coordinator failure* (it elects a new leader). 2PC *blocks* on coordinator failure (the coordinator is a single point of failure). This is the distinction that matters — consensus is resilient to failure, 2PC is not. 2PC is a transaction protocol, not a consensus protocol. (Detailed in Chapter 11.)

**What consensus does not provide:** Consensus guarantees agreement on a value. It does not guarantee:
- **Availability during leader election.** Raft is unavailable during elections (~150–300 ms). For most systems this is acceptable; for latency-critical systems it is not.
- **Scalability beyond ~7 nodes.** The message complexity of consensus (O(N²) messages per decision) limits the practical cluster size.
- **Tolerance of Byzantine faults.** As noted, Raft assumes crash-fault model only.
- **Real-time ordering guarantees.** Raft's ordering is based on log indices and terms, not physical time. For real-time ordering, TrueTime (Chapter 5) is needed.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Consensus problem | N machines must agree on one value despite failures |
| FLP impossibility | In async systems, no deterministic protocol can guarantee consensus in finite time — workaround via timing assumptions |
| Quorum | Majority (N/2 + 1) ensures at most one group can decide |
| Crash-fault model | Assumes nodes stop but don't lie. Byzantine case deferred to Ch 17 |
| Raft | Three sub-problems: leader election, log replication, safety |
| Raft safety | DETERMINISTIC — committed entries are never lost (leader completeness + quorum intersection) |
| Raft vs Paxos | Understandability is the winning insight. Raft sacrificed elegance for teachability |
| etcd | Production Raft — 3 or 5 nodes, never 1 or 2 |

**Next:** Consensus produces an agreed total order of events. That agreed order is a **log.** The log is the most important abstraction in distributed systems — the through-line connecting consensus, replication, transactions, and streaming. Chapter 8.
