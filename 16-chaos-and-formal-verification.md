# Chapter 16: Chaos Engineering and Formal Verification

Every chapter in this course has described a design for failure. Redundancy, consensus, replication, reconciliation loops — each is a mechanism for ensuring the system survives when components fail.

But design is not reality. How do you *know* your system actually survives failures? How do you *know* the consensus implementation correctly handles leader crashes? How do you *know* the replication strategy recovers from a network partition?

This chapter examines two complementary approaches to verification: chaos engineering (which tests running systems) and formal verification (which proves specifications correct). Neither alone is sufficient.

---

## 16.1 The Verification Problem

You have designed for failure at every layer. But:

- Your consensus implementation may have a bug that only manifests under specific timing conditions.
- Your load balancer may route traffic to a dead node because its health check interval is too long.
- Your database replication may diverge under a rare combination of crashes and restarts.
- Your timeout configuration may cause cascading failures under load.

Unit tests exercise individual components. Integration tests exercise component interactions. But controlled testing cannot reproduce the failure modes of a distributed system: network partitions, bursty delays, correlated failures, and the unbounded combinations of partial failures.

**The only way to know if the system survives failures is to create real failures and observe what happens.**

---

## 16.2 Chaos Engineering

Chaos engineering is the practice of injecting failures into a running system to test its resilience. It follows the experimental method:

1. **Define the steady state.** What does "healthy" look like? Measurable outputs: request latency, error rate, throughput, resource utilisation.

2. **Form a hypothesis.** "The system continues serving with < 200ms p99 latency and < 0.1% error rate when one node is killed."

3. **Inject a failure.** Kill the node. Introduce latency on the network link. Partition the cluster. Corrupt data.

4. **Observe.** Measure the steady-state variables during and after the injection.

5. **Update the hypothesis.** Did the system survive within the defined bounds? If yes, confidence increases. If no, you found a gap — fix it and repeat.

### Experiments Mapped to Layers

| Experiment | Layer Tested | What It Verifies |
|-----------|-------------|-----------------|
| Kill a worker node | L4 (distribution) | Does the orchestrator reschedule pods? Are other nodes unaffected? |
| Kill a control plane node | L7 (consensus) | Does Raft elect a new leader? Is there data loss during the election? |
| Network partition between two groups | L9 (CAP) | Does the minority partition stop serving (CP) or serve stale data (AP)? |
| Introduce 500ms latency on a service link | L1, L5 (time, ordering) | Do timeouts fire correctly? Does backpressure propagate? Is there cascading timeout? |
| Saturate CPU on a worker node | L3 (cgroups) | Does the container runtime enforce limits? Do other containers experience interference? |
| Kill the leader etcd node | L7 (consensus) | Can the cluster re-establish quorum? How long does the election take? |
| Corrupt a write to storage | L1 (data integrity) | Does ECC or checksumming catch the corruption? |

### What Chaos Engineering Cannot Do

Chaos engineering tests the system as it is, under conditions you choose. It cannot:

- **Prove the system is correct.** No amount of successful experiments proves a protocol implementation is bug-free. You only know the system survived *these specific* failure injections — there may be others you didn't think of.

- **Test every state.** A distributed system has an astronomical number of possible states. Chaos testing covers a tiny fraction.

- **Verify safety properties.** Safety (e.g., "no committed entry is ever lost") cannot be verified by testing, because the violating state might occur on a path you didn't test.

Chaos engineering is necessary but not sufficient. It tests **liveness** (the system makes progress under failure) and **performance** (the system stays within operational bounds). For safety verification, you need formal methods.

---

## 16.3 Formal Verification (TLA+)

TLA+ (Temporal Logic of Actions) is a formal specification language for describing and verifying distributed systems. A TLA+ specification describes:

- The **state** of the system (variables and their possible values)
- The **actions** that transition between states
- The **safety** properties that must hold in every state
- The **liveness** properties that must eventually hold

A TLA+ model checker (TLC) explores all reachable states of the specification and checks whether the safety and liveness properties hold. If the model checker finds a violation, it produces an **error trace** — the exact sequence of states that leads to the violation.

**What TLA+ provides that testing cannot:**
- **Exhaustive state exploration.** The model checker tests *every* reachable state, within the bounds of the model.
- **Safety verification.** If the spec says "committed entries are never lost," the model checker proves there is no reachable state where a committed entry is absent.
- **Design-time debugging.** TLA+ finds design flaws before a single line of implementation code is written.

**Practical use:** Amazon, Microsoft, and other companies use TLA+ to verify critical distributed systems. Amazon's DynamoDB, S3, and EBS have been verified with TLA+. The typical pattern: write a TLA+ spec for the consensus protocol, verify correctness, then implement the protocol in code following the spec.

---

## 16.4 Implementation Testing (Jepsen)

Jepsen is a testing framework for checking whether a real distributed system implementation matches its claimed consistency guarantees. Jepsen:

1. Deploys the system under test (Cassandra, MongoDB, etcd, whatever).
2. Generates a workload of reads and writes.
3. Injects failures (crashes, partitions, latency spikes) during the workload.
4. Records the history of all operations and responses.
5. Checks whether the history violates the system's claimed consistency model (linearizability, serializability, etc.).

**What Jepsen provides that TLA+ cannot:**
- Tests the *actual implementation*, not a specification.
- Catches implementation bugs, not design bugs.
- Tests performance under realistic network conditions.

**What Jepsen cannot do:**
- Prove the system is correct (it only finds violations if they occur during the test run).
- Verify safety exhaustively (it's a testing tool, not a model checker).

---

## 16.5 The Relationship

Chaos engineering, TLA+, and Jepsen are complementary, not interchangeable:

| Tool | What It Tests | What It Finds | What It Cannot Do |
|------|--------------|--------------|------------------|
| Chaos engineering | Running system under failure injection | Liveness bugs, performance degradation, operational gaps | Prove safety, find edge-case bugs |
| TLA+ | Specification (mathematical model) | Design flaws, safety violations | Find implementation bugs, test performance |
| Jepsen | Implementation under failure injection | Implementation bugs that violate consistency | Prove correctness, exhaustive testing |

The honest picture:
- **Chaos tests what actually happens** in the real system under specific conditions.
- **TLA+ proves what *must* happen** given a correct specification.
- **Jepsen checks what *did* happen** in your specific build during a specific test run.

All three together = confidence. Neither alone is sufficient. Presenting chaos as the complete verification strategy is dangerous — it creates a false sense of security. Presenting TLA+ as a replacement for testing is equally wrong — a correct specification does not guarantee a correct implementation.

---

## 16.6 Design-Space Beat

The verification budget is limited. Spending time on TLA+ means less time on chaos testing, and vice versa. The right allocation depends on the cost of being wrong:

- **Safety-critical systems** (distributed consensus, financial ledgers, access control): invest heavily in TLA+. A safety violation in these systems is catastrophic. Chaos testing alone is insufficient.

- **Performance-critical systems** (CDN, caching, data pipeline): invest in chaos engineering. Liveness and performance under failure matter more than edge-case correctness. TLA+ is overkill for a cache that can tolerate staleness.

- **New protocol implementations** (custom Raft variant, new replication protocol): both. TLA+ for the spec, chaos testing for the implementation, Jepsen for verification.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Verification problem | Design is not reality — systems fail in ways that controlled testing cannot predict |
| Chaos engineering | Inject real failures into running systems, observe behavior, update confidence |
| Liveness vs safety | Chaos tests liveness (system makes progress). It cannot verify safety. |
| TLA+ | Formal specification language. Exhaustive model checking of safety/liveness properties. |
| Jepsen | Implementation testing for consistency violations |
| Complementary | All three needed. None alone is sufficient. |

**Next:** The entire course so far assumes nodes that fail but do not lie. What happens when nodes can cheat? Chapter 17 — Security and Trust.
