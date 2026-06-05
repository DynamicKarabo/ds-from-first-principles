# Distributed Systems from First Principles — Course Architecture (v3)

---

## Core Philosophy

This course traces every distributed systems concept back to physical reality. But it does something else: at every layer, it explores the field of viable answers, explains why the winner won, and names the tradeoff it accepted.

The goal is not to produce students who can recite protocols. It's to produce students who, given a constraint, can **enumerate the solution space, reason about tradeoffs, and make sound engineering decisions** — exactly the skill required to design distributed systems.

---

## Three Axioms (from Physical Reality)

The course rests on two load-bearing axioms that generate the impossibility results, plus a third that is pedagogically convenient to state independently (though it follows from the first).

**Axiom 1: Asynchrony** — There is no bound on message delay. A message can take arbitrarily long to arrive, and there is no way to distinguish "the node is dead" from "the message is still in transit" in finite time. This is the source of FLP impossibility, the fundamental limit on consensus.

*Physical root:* The speed of light is finite (≈ 3×10⁸ m/s), plus unbounded queuing and processing delay at intermediate nodes creates arbitrarily large variance. The two together mean you cannot bound end-to-end delay.

**Axiom 2: Partial failure** — Components fail independently, and failure is partial (some components fail while others remain healthy). One node dying tells you nothing about whether another has died. The network can partition, leaving both sides running and neither aware of the other.

*Physical root:* Entropy — systems tend toward disorder, hardware degrades, bits flip. But crucially, independence is an *idealising assumption* we make deliberately and then interrogate. In reality, failures are often correlated (shared power, shared switches, shared bugs, shared deploys). The course returns to this tension when discussing multi-zone and multi-region architectures.

**Axiom 3: No shared clock** — Different machines experience time differently. Clock drift, NTP synchronisation error, and relativity of simultaneity mean two nodes cannot perfectly agree on when an event occurred.

*Relationship to Axiom 1:* This is a consequence of asynchrony plus physical separation. It's stated separately because the ordering problem (happens-before, logical clocks) is foundational to everything that follows.

---

## Three Mathematical Lenses

Every layer is analysed through the lens (or combination of lenses) appropriate to it. These are complementary tools, not a partition of the material — many results require more than one.

| Lens | What It Handles | Where It Applies |
|------|----------------|-----------------|
| **Logical / ordering** | Causality, happens-before, vector clocks, consistent cuts | Time & Ordering, Consistency Models |
| **Adversarial / worst-case** | Safety proofs, quorum intersection, FLP, fault tolerance | Consensus, Replication, CAP |
| **Probabilistic** | Failure detection under uncertainty, tail latency, clock bounds | Failure Detection, Observability, TrueTime |

**Discipline rule:** No lens is claimed as the "universal language." The honest insight is that different layers need different mathematics.

---

## The Two-Tier Design-Space Structure

At every layer, the course follows a fixed closing beat:

> **The constraint → The field of viable answers → Why the winner won → What tradeoff it accepted**

This trains a specific habit: students learn to expect that every solution sits in a field of alternatives, and that engineering is the practice of choosing among tradeoffs — not applying deterministic derivations.

**Deep dives** (Tier 2) at three layers where the design fight was itself the curriculum:

1. **Containers (Layer 3)** — Docker vs VMs, jails, static binaries. Establishes the "tradeoffs accepted" lens early.
2. **Consensus (Layer 7)** — Raft vs Paxos, Zab, Viewstamped Replication. A lesson about understandability as an engineering goal.
3. **Orchestration (Layer 13)** — Kubernetes vs Mesos, Nomad, Swarm, and the Borg inheritance. Ecosystem strategy, extensibility, declarative vs imperative.

**Closing-beat discipline:** Not every layer has a meaningful design-space fork. If the constraint genuinely forced one solution, the beat may be skipped. Forcing tradeoff analysis where there was no real fork produces hollow content and undermines the tool.

**Depth rule for Tier 1 beats:** Include an alternative only if a competent team genuinely chose it at the time, or if understanding why it lost clarifies the constraint. Everything else is trivia.

**Best Tier-1 material:** Alternatives whose losing option still has legitimate use cases today (e.g., fixed timeouts, host networking). "When is the simple answer still right?" teaches more than a one-sided victory.

---

## The 18-Layer Chain (L0–L17)

Each layer is derived from the problem created by the previous layer. Each layer ends with the design-space beat. Key design-space forks are marked with [DEEP DIVE].

### Part I: Foundations

**Layer 0 — Physical Reality**
- The three axioms (asynchrony, partial failure, no shared clock)
- Speed of light (c ≈ 3×10⁸) as the root of asynchrony
- Entropy as the root of independent failure
- Relativity of simultaneity as the root of the clock problem
- The memory hierarchy is a physics constraint (L1 → L2 → L3 → RAM → SSD → HDD), not a design choice

**Layer 1 — Computing Hardware**
- Transistors → logic gates → flip-flops (state!) → registers + clock → ALU → CPU
- Three non-negotiables of any computer: CPU (compute), RAM (fast volatile state), Storage (slow persistent state)
- The Von Neumann bottleneck
- Defines "node" for the rest of the course: an independent failure domain with its own clock that communicates only by messages
- (The hardware definition is kept as a simplifying intuition, explicitly flagged as such)

**Layer 2 — The Operating System**
- Multiple processes on one machine → resource conflicts
- Process isolation (address spaces, the kernel as trust boundary)
- Virtual memory (MMU, mapping, swapping)
- File systems (flat block device → hierarchy + naming)
- System calls (the controlled gateway between user code and hardware)
- Process vs thread (isolation cost vs concurrency)

**Layer 3 — The Dependency Problem (Containers)**
- Problem: "Works on my machine" — conflicting dependencies across environments
- Field of viable answers: chroot, jails, static binaries, VMs, containers
- [DEEP DIVE] Why Docker won: layer sharing, registry ecosystem, Dockerfile as convention, speed vs VMs, density tradeoff
- Linux primitives: namespaces (PID, net, mount, user, UTS, IPC), cgroups (CPU/memory/I/O limits)
- Dockerfile → Image (layered FS) → Container (running instance)
- Tradeoff accepted: shared kernel limits isolation (no Windows containers on Linux hosts, no kernel version agility)

### Part II: Distribution

**Layer 4 — The Distribution Problem**
- Why one machine isn't enough: finite resources, single point of failure, physical scaling limits
- Multiple machines → cannot share memory → must communicate over networks
- The network is slow (5000× slower than local RAM), unreliable, and can partition
- **Partial failure** is the defining problem of distributed systems
- The three axioms crystallise at this layer

**Layer 5 — Time and Ordering**
- No shared clock → cannot order events by physical time
- Happens-before relation (Lamport 1978)
- Lamport clocks (capture causal order, not concurrency)
- Vector clocks (capture both)
- NTP: how real systems approximate time, clock skew as a probability distribution
- TrueTime (Google Spanner): explicitly bounded clock uncertainty — the Bayesian moment for clocks
- **Design-space beat:** Logical clocks vs physical clocks vs TrueTime — when each is appropriate

### Part III: Coordination

**Layer 6 — Failure Detection and Membership**
- Problem: can't know if a remote node is alive (Axiom 1)
- Field of viable answers: fixed timeouts, adaptive timeouts, phi-accrual
- [Tier 1 beat] Fixed timeouts: simple, fragile, wrong under variable latency
- Phi-Accrual failure detector: model heartbeat arrival times as a probability distribution, compute suspicion level φ. The one place Bayesian reasoning is genuinely correct and important
- **Design-space beat:** When are fixed timeouts still the right choice? (Low-variance datacenter networks, simplicity-critical systems)
- Gossip membership (SWIM): O(N²) → O(N log N), information spreads exponentially
- Membership changes trigger consensus (bridge to Layer 7)

**Layer 7 — Consensus and Agreement**
- The agreement problem: N machines must agree on one value despite crashes and message loss
- FLP impossibility: in an asynchronous system, consensus cannot be guaranteed in finite time. Grounded in Axiom 1, not in c+entropy
- Workaround: timing assumptions (failure detectors + randomized timeouts)
- Quorum: N/2 + 1 guarantees at most one group can agree. Arithmetic of odd numbers
- Raft: leader election (randomized timeouts), log replication (majority acknowledge), safety (committed is committed)
- This is DETERMINISTIC safety — a committed entry is committed, full stop
- **IMPORTANT: The entire consensus discussion assumes a crash-failure model.** Nodes may stop responding, but they will not lie, send conflicting messages, or collude. The Byzantine case (malicious nodes) is a separate axis introduced in Layer 17. Every consensus and replication claim in Layers 7–14 carries this unstated assumption — it is made explicit here rather than deferred.
- [DEEP DIVE] Raft vs Paxos vs Zab vs Viewstamped Replication. The winning insight: understandability is an engineering goal, not a nicety. Raft won because decomposing consensus into three sub-problems made it teachable, verifiable, and implementable.
- etcd as production Raft instance
- **Derivation caveat:** Raft is not the "correct" consensus protocol — it's a particular set of tradeoffs that won under particular conditions (crash-fault, non-Byzantine, moderate scale)

**Layer 8 — The Log**
- The log is what consensus produces: an agreed total order of events
- This is the unifying abstraction of the entire course
- Once you have consensus, you have a log. The log is the spine that everything else builds on
- **Bridge to everything downstream:** Replication (replay the log), transactions (log of intents), streaming (publish the log), event sourcing (store the log as the source of truth)
- The log is not a new idea — it predates distributed systems (write-ahead logging in databases). Distributed consensus just lets multiple nodes agree on the same log

### Part IV: Systems

**Layer 9 — CAP Theorem**
- Problem: during a network partition, you face a logical contradiction. You cannot simultaneously answer immediately and answer correctly
- Derivation: partition (inevitable per Axiom 2) + need to respond = must choose
- CP: refuse to answer rather than answer incorrectly (etcd, ZK)
- AP: answer with potentially stale data (Cassandra, DynamoDB)
- PACELC: even without partitions, there's a latency-vs-consistency tradeoff
- **Tier 1 beat:** When is AP the wrong choice? (Financial systems, inventory, locking) When is CP overkill? (CDN, social feed, leaderboard)
- CAP is a theorem, not a slogan. Proven, not assumed

**Layer 10 — Replication and Consistency**
- Problem: data must survive machine death (Axiom 2). Solution: keep copies on multiple machines
- Consistency models (from strongest to weakest):
  - Linearizability (operations appear atomic, all replicas see the same order)
  - Sequential (each replica's operations appear in program order)
  - Causal (causally related operations seen in order, concurrent operations unordered)
  - Eventual (replicas converge given enough time and no new writes)
- Replication strategies:
  - Primary-backup (one writer, multiple readers — simple, single point of failure)
  - Chain replication (write through a chain of replicas — good throughput, high write latency)
  - Quorum replication (Dynamo-style, W + R > N — tunable consistency)
  - State machine replication (deterministic machine + replicated log = consensus-style)
- CRDTs as the AP answer: converge without coordination. Contrast with consensus (CP)
- **Tier 1 beat:** When does eventual consistency actually break your application? When is it fine?
- Read-repair and anti-entropy as the reconciliation machinery
- Exactly-once semantics as the synthesis: idempotency + deduplication + atomic commit
- **Design-space beat:** Strong consistency vs availability — not a technical debate, a product decision. The right answer depends on what breaks if you're wrong
- **Design-space beat:** Write-path latency vs read-path latency — determined by quorum configuration (W + R > N). Choosing W=3, R=1 in a 3-node cluster optimises for reads at the cost of slower writes. W=1, R=3 optimises for writes. Tunable quorums let you pick your position on the latency surface

**Layer 11 — Distributed Transactions**
- Problem: a transaction spanning multiple nodes needs atomic commit
- 2PC: prepare → commit/abort. Why it blocks on coordinator failure (no liveness under Axiom 1)
- Why 2PC ≠ consensus: consensus guarantees liveness under the same conditions where 2PC blocks. This is the key distinction — not "2PC is bad" but "2PC solves a different problem"
- 3PC: avoids blocking but introduces new failure modes
- Saga pattern: sequence of local transactions with compensating actions. No isolation guarantee — you accept it
- Idempotency: the practical bedrock of correctness in real systems
- **Tier 1 beat:** When is 2PC still the right choice? (Short-lived transactions, reliable coordinator, small scale) When must you use Sagas? (Long-lived workflows, heterogeneous systems)

**Layer 12 — Data Partitioning**
- Problem: a single node can't store all the data. Solution: split data across nodes
- Naive hashing: hash(key) → node. Problem: adding/removing a node reshuffles almost everything
- Consistent hashing: each key maps to a position on a ring, each node covers an arc. Adding/removing a node only affects its immediate neighbors
- Virtual nodes: each physical node owns multiple arcs. Handles hot spots, heterogeneous capacity
- Rebalancing: moving data without downtime; the tradeoff between speed and disruption
- **Tier 1 beat:** Consistent hashing vs range-based partitioning vs directory-based partitioning. When each is appropriate
- Hotspot mitigation: load shedding, replica selection, hedging

**Layer 13 — Orchestration (Kubernetes)**
- Problem: N machines, M containers, continuous failures. Who manages the system autonomously?
- Kubernetes as the accumulated solution to the problems derived in layers 4-12
- Component mapping:
  - etcd → consensus (Layer 7) as cluster memory
  - API server → single authority (Layer 4 problem: who decides?)
  - Controller manager → reconciliation loop (Layer 10: desired vs actual)
  - Scheduler → bin-packing under uncertainty (Layer 12: placement)
  - kubelet → local execution (Layer 6: node agent)
  - Container runtime → execution mechanics (Layer 3)
  - kube-proxy → service discovery and routing (Layer 10: reaching the right copy)
- [DEEP DIVE] Why Kubernetes won over Mesos, Nomad, and Docker Swarm. The Borg inheritance (declarative state, pods, services). The API-extensibility bet (CRDs, controllers). Ecosystem effects. Declarative vs imperative: why "desired state" changed everything
- **Explicit caveat:** Kubernetes is one contingent design point, not a logical necessity. Mapping components to problems is pedagogical illustration, not proof that it had to be this way

**Layer 14 — Streaming and the Log**
- Problem: once you have an agreed log (Layer 8), what else can you build with it?
- Kafka: a distributed log. Not a queue, not a message broker — a structured commit log
- Event sourcing: the log as the source of truth. Current state is derived, not stored
- Stream processing: consuming the log in real time (Flink, Kafka Streams)
- **Tier 1 beat:** Log-based systems vs traditional databases vs hybrid (dual-write). When does the log abstraction simplify the system? When does it overcomplicate it?

### Part V: Operations

**Layer 15 — Observability**
- Problem: distributed systems produce partial, conflicting evidence about their own health. You must reason about system state from incomplete signals
- Distributed tracing: reconstructing a single request's path across N services. The fundamental problem: without shared clocks (Axiom 3), how do you correlate events across services?
- Metrics: RED (Rate, Errors, Duration) for services, USE (Utilization, Saturation, Errors) for resources
- Structured logging: each log entry is an event with context, not a string
- Monitoring vs debugging: monitoring detects that something is wrong, debugging identifies what and where
- **Tier 1 beat:** Tracing vs metrics vs logging — when is each the right primary signal? The observability triage pattern

**Layer 16 — Chaos Engineering and Formal Verification**
- Problem: you've designed for failure at every layer. How do you know your design actually works?
- Chaos engineering experiments: hypothesis → inject failure → observe → update belief
- Mapped to layers: kill a node (Layer 4), NetworkPolicy test (Layer 13), partition the network (Layer 9)
- **Chaos tests liveness and performance properties.** It operates on running systems and observes real behavior under fault injection. It CANNOT verify safety — no amount of successful experiments proves a protocol is correct.
- **Formal verification (TLA+):** model-checking a specification against all possible execution paths. Proves safety properties hold in every state. Does not test implementation code or measure real-world performance.
- **Implementation testing (Jepsen):** runs a real system against a known fault model and checks for safety violations (linearizability, consistency). Finds bugs in the *implementation* of a protocol, but cannot prove a protocol is correct — only that a specific set of tests didn't find a violation.
- TLA+ and Jepsen are complementary but NOT co-equal. TLA+ proves a *spec* correct. Jepsen tests an *implementation* for known failure modes. Both are needed. Neither alone is sufficient.
- **The honest picture:** chaos tests what actually happens, TLA+ proves what *must* happen, Jepsen checks what *did* happen in your specific build. All three together = confidence.

**Layer 17 — Byzantine Fault Tolerance and Trust Models**
- Problem: everything so far assumes nodes follow the protocol honestly. What if they don't?
- Crash failures vs Byzantine failures: crash = stops responding, Byzantine = may behave arbitrarily (lie, collude, send conflicting messages)
- Byzantine Fault Tolerance: why it's harder than crash tolerance (N = 3f + 1 vs N = 2f + 1). The additional cost
- In practice: most production systems assume crash-fault model (Raft, etcd, Kafka, K8s) because Byzantine tolerance is 3-10× more expensive
- Trust models: zero-trust networking (Layer 13's NetworkPolicy), identity as a first-class primitive, encryption at rest and in transit
- **Tier 1 beat:** When does crash-fault tolerance become insufficient? (Financial systems at scale, cross-organizational consensus, blockchains/public networks)
- Brief touch on BFT protocols (PBFT, HotStuff) — enough to establish the axis without detouring into the full Byzantine literature

---

## The Log as Through-Line

The log is the unifying abstraction of the entire course. It enters at Layer 8 (right after consensus) and recurs through every subsequent layer:

| Layer | The Log Appears As |
|-------|-------------------|
| 8 (The Log) | Introduced: the log = what consensus produces (agreed total order) |
| 10 (Replication) | SMR replays the log on each replica |
| 11 (Transactions) | Write-ahead log for atomicity |
| 14 (Streaming) | Kafka = distributed log as a product |
| 14 (Event Sourcing) | The log as the source of truth |

This replaces the v1 attempt to use Bayesian reasoning as the universal thread. The log is a genuinely unifying abstraction — it's concrete, historically grounded (write-ahead logging predates distributed systems by decades), and visibly recurs across layers.

---

## Design-Space Beats: Summary

| Layer | Beat | Tier |
|-------|------|------|
| 3 (Containers) | Why Docker won [DEEP DIVE] | 2 |
| 5 (Time) | Logical vs physical vs TrueTime | 1 |
| 6 (Failure Detection) | Fixed timeouts vs phi-accrual — when simple wins | 1 |
| 7 (Consensus) | Raft vs Paxos — understandability as engineering goal [DEEP DIVE] | 2 |
| 9 (CAP) | When is CP overkill? When is AP dangerous? | 1 |
| 10 (Replication) | Strong consistency vs availability as a product decision | 1 |
| 10 (Replication) | Write-path latency vs read-path latency — quorum size tradeoffs | 1 |
| 10 (Exactly-once) | Idempotency + dedup + atomic commit | 1 |
| 11 (Transactions) | 2PC vs Sagas — isolation vs availability | 1 |
| 12 (Partitioning) | Consistent hashing vs range vs directory — crisp decision criteria | 1 |
| 13 (Orchestration) | Why K8s won [DEEP DIVE] | 2 |
| 13 (Orchestration) | When not to use K8s — single service, small team, serverless, edge, latency-critical | 1 |
| 14 (Streaming) | Log-based vs traditional DB vs hybrid; event sourcing over-application warning | 1 |
| 15 (Observability) | Tracing vs metrics vs logging triage; what to instrument first under budget | 1 |
| 16 (Verification) | Chaos first/Jepsen on consistency claim/TLA+ for catastrophic failure | 1 |
| 17 (BFT) | Crash-fault vs Byzantine — when must you pay the BFT cost? | 1 |

---

## The Complete Chain

```
PHYSICAL REALITY (three axioms)
  → HARDWARE (transistors → CPU/RAM/storage)
    → SINGLE MACHINE (OS, isolation, syscalls)
      → CONTAINERS [DEEP DIVE: Docker vs VMs/jails/binaries]
        → DISTRIBUTION PROBLEM (partial failure, three axioms crystallise)
          → TIME AND ORDERING (happens-before, Lamport/vector, TrueTime)
            → FAILURE DETECTION (phi-accrual, gossip, when fixed timeouts win)
              → CONSENSUS [DEEP DIVE: Raft vs Paxos — understandability wins]
                → THE LOG (unifying abstraction — what consensus produces)
                  → CAP THEOREM (partition + response = choice)
                    → REPLICATION + CONSISTENCY (models, strategies, CRDTs, exactly-once)
                      → DISTRIBUTED TRANSACTIONS (2PC, Sagas, idempotency)
                        → DATA PARTITIONING (consistent hashing, rebalancing)
                          → ORCHESTRATION [DEEP DIVE: why K8s won]
                            → STREAMING + THE LOG (Kafka, event sourcing)
                              → OBSERVABILITY (tracing, metrics, logging)
                                → CHAOS ENGINEERING + FORMAL VERIFICATION
                                  → SECURITY + TRUST (crash vs Byzantine)
```

---

## What This Produces

A student who completes this course can:

1. Derive any distributed systems concept from the three axioms (asynchrony, partial failure, no shared clock)
2. For any solution, reason about its design space — what alternatives existed, why the winner won, and what tradeoff it accepted
3. Frame every design decision in terms of **what uncertainty it manages** and **which mathematical lens** applies
4. Look at Kubernetes and see the accumulated history of distributed systems problems, component by component — and recognize that it's one contingent answer among several
5. Design distributed systems by tracing the constraint chain forward, not by copying patterns from existing systems

This is the difference between someone who can *recite* distributed systems and someone who can *do* distributed systems.
