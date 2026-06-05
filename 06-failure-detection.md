# Chapter 6: Failure Detection and Membership

Before you can agree on anything (Chapter 7) or coordinate actions across machines, you need to know who is part of the group and whether they are alive. This chapter examines how distributed systems detect failures — and why perfect detection is impossible.

---

## 6.1 The Detection Problem

Machine A needs to know if Machine B is alive. How?

Machine A can send a message to Machine B: "Are you alive?" Machine B can reply: "Yes." But this itself is a message exchange — subject to the same network delays, losses, and timeouts as any other communication.

Machine A sends a heartbeat request. It waits. No response arrives. What does this mean?

**Possibility 1:** Machine B has crashed. It cannot respond.
**Possibility 2:** Machine B is running, but the request was lost in transit.
**Possibility 3:** Machine B received the request, responded, but the response was lost.
**Possibility 4:** Machine B responded, but the response is delayed (network congestion, queuing, switch buffer overflow).
**Possibility 5:** Machine B is running but overloaded (its CPU is saturated, it cannot process the heartbeat in time).

Possibilities 1–5 are observationally indistinguishable during the waiting period. Axiom 1 (asynchrony) states that there is no bound on message delay — you cannot distinguish a dead node from a slow network in finite time.

**The fundamental truth:** You can never be 100% certain that a remote node is dead. You can only reach a threshold of suspicion where you *act as if* it is dead.

---

## 6.2 Fixed Timeouts

The simplest failure detector: expect a heartbeat every T seconds. If no heartbeat arrives within `T × N` seconds (where N is a tolerance for missed heartbeats), declare the node dead.

```
if (now - last_heartbeat > timeout) {
    declare("dead")
}
```

**Problems with fixed timeouts:**

1. **Too short → false positives.** A transient network spike causes a healthy node to be declared dead. The cluster rebalances, moves data, runs a new election — all unnecessary. The cost of a false positive can be high (data movement, leader election, cache warming).

2. **Too long → slow detection.** The cluster continues routing traffic to a dead node. Requests fail. The user sees errors. The system should have detected the failure and recovered by now.

3. **Cannot adapt.** The same timeout that works well at 2 AM (low traffic, stable network) may produce false positives during peak hours (high traffic, queuing delays). A fixed timeout is optimal for exactly one network condition.

4. **No information about confidence.** A fixed timeout returns a binary alive/dead. It does not say "I'm fairly confident" or "I'm barely confident." The consumer of the detection (consensus protocol, load balancer) must decide what to do without knowing how reliable the detection is.

Fixed timeouts are simple to implement and understand. They work well in stable, low-variance environments. They perform poorly in any system with variable network latency or transient congestion.

---

## 6.3 Phi-Accrual Failure Detector

The phi-accrual failure detector, introduced by Naohiro Hayashibara et al. in 2004, replaces the fixed threshold with a probabilistic model. Instead of asking "has the timeout expired?" it asks "how certain are we that the node is dead?"

**How it works:**

1. Maintain a sliding window of recent heartbeat inter-arrival times.

2. From this window, estimate the probability distribution of heartbeat arrival times. (Typically modelled as a normal distribution, though the implementation may use a simpler approximation.)

3. When a heartbeat is overdue by time `t`, compute the **suspicion level** φ:

```
φ = -log₂(P(later than t | alive))
```

This is the negative log (base 2) of the probability that a heartbeat would arrive more than `t` time units after the expected time, *assuming the node is alive*.

4. A high φ means the observed delay is very unlikely under the "node is alive" hypothesis — so we should suspect the node is dead. A low φ means the delay is within normal variance — the node is probably alive.

**Interpretation of φ values:**

| φ | Probability heartbeat arrives this late if node is alive | Interpretation |
|---|--------------------------------------------------------|----------------|
| 0 | 50% | Normal variance |
| 1 | ~10% | Mildly suspicious |
| 2 | ~1% | Suspicious |
| 3 | ~0.1% | Very suspicious |
| 5 | ~0.3% | Strong suspicion |
| 8 | ~0.004% | Effectively certain |

A threshold is set (typically φ = 5–8). When φ exceeds the threshold, the node is declared dead. But the detector never reaches 100% certainty — it only crosses a threshold.

**Why this is better than fixed timeouts:**

| Dimension | Fixed Timeout | Phi-Accrual |
|-----------|--------------|-------------|
| Adapts to network conditions | No | Yes — learns the distribution from recent data |
| Provides confidence information | Binary (alive/dead) | Continuous (φ value) |
| False positives in variable networks | Common | Rare (adapts to variance) |
| False negatives with strict timeout | Slow detection | Adjusts automatically |

**The Bayesian connection:** Phi-accrual is the one place in distributed systems where Bayesian reasoning is genuinely correct and important. The detector maintains a belief about whether a node is alive, updates this belief with each new observation (heartbeat arrival time), and triggers action when the belief crosses a threshold. It is Bayesian inference on system state, applied to a production problem.

---

## 6.4 Gossip Membership (SWIM)

Failure detection answers "is a specific node alive?" But a distributed system also needs to know *which nodes are members of the cluster*. New nodes join. Nodes leave gracefully. Nodes crash and must be removed from the membership list.

Membership protocols solve: given N nodes, how does every node learn about the current set of members?

**Naive approach:** every node sends its membership list to every other node every T seconds. This is O(N²) messages per round — 90,000 messages for a 300-node cluster, 10,000,000 for a 1,000-node cluster. Not sustainable.

**Gossip (epidemic) protocol:**

Each node periodically (e.g., every 100 ms) selects a random subset of other nodes (typically 1–3) and exchanges membership information.

- If the selected node responds, both nodes merge their membership views.
- If the selected node does not respond, the gossiper increments a suspicion counter for that node. After a threshold of suspicion reports from different nodes, the suspected node is declared dead and removed.

Information spreads exponentially: after `O(log N)` rounds, all nodes have received the update. A 1,000-node cluster reaches full dissemination in `log₂(1000) ≈ 10` gossip rounds, or about 1 second at 100 ms per round.

**SWIM (Scalable Weakly-consistent Infection-style Membership):**

SWIM is the most widely used gossip membership protocol (Cassandra, Consul, memberlist). Key features:

- **Suspicion rather than immediate death:** When a node fails to respond to a gossip ping, it is marked "suspected" (not "dead"). Other nodes are encouraged to confirm the suspicion by pinging the suspected node themselves. This reduces false positives — a single missed ping does not eject a healthy node.

- **Dissemination via gossip:** When a node confirms a suspicion, the failure is gossiped to the cluster. The gossip mechanism ensures all nodes learn about the failure within `O(log N)` rounds.

- **Lifelessness detection as a byproduct of gossip:** SWIM does not require separate heartbeat messages. The gossip pings serve double duty — they disseminate membership information and detect failures simultaneously.

---

## 6.5 Membership Changes Trigger Consensus

Failure detection and membership are not independent — they feed into consensus (Chapter 7).

When a node is suspected dead:
1. The suspicion is gossiped to the cluster.
2. If enough nodes confirm the suspicion, the node is declared dead.
3. The membership list is updated.
4. If the dead node was a leader in a consensus group (like Raft's leader or an etcd node), a new election must be triggered.
5. Resources (partitions, replicas, workloads) that were assigned to the dead node must be reallocated.

The failure detector's confidence (φ value) directly affects cluster availability:
- A detector with many false positives triggers unnecessary leader elections and data movement.
- A detector with slow detection leaves traffic routing to dead nodes, causing user-visible failures.

The phi-accrual detector's continuous output (φ threshold) is a tunable parameter. Set φ too low (eager to declare death) and you get unnecessary churn. Set φ too high (reluctant to declare death) and you get increased latency for truly dead nodes. There is no universal correct value — it depends on the application's tolerance for false positives vs false negatives.

---

## 6.6 Design-Space Beat: When Fixed Timeouts Are Still Correct

Phi-accrual is more sophisticated than fixed timeouts. But sophisticated is not always better. Fixed timeouts are the right choice when:

**Network conditions are stable and predictable.** A dedicated cluster within a single datacenter, connected by low-latency switches, with no external traffic crossing the same links — this produces highly predictable heartbeat times. A fixed timeout with 2–3 missed heartbeat tolerance works as well as phi-accrual with less complexity.

**Simplicity is more important than optimal performance.** If a system occasionally triggers a false-positive leader election and the cost is low (the election completes in 100 ms, no user impact), the complexity of implementing phi-accrual may not be justified.

**The implementation language or platform does not easily support the statistical computation.** Embedded systems, kernel modules, and resource-constrained environments may not have floating-point support or a normal distribution CDF.

**The team maintaining the system is small.** Phi-accrual requires parameter tuning (window size, φ threshold). A fixed timeout is self-explanatory to any engineer. A phi-accrual threshold is not.

The rule: **choose the simplest failure detector that meets your correctness and performance requirements.** If fixed timeouts have caused observable problems (false positives in production, slow detection causing errors), move to phi-accrual. If not, keep the simpler solution.

---

## 6.7 Derivation Chain

```
Axiom 1 (asynchrony): cannot distinguish dead from slow in finite time
  → Need a failure detection mechanism
    → Fixed timeouts: simple but fragile
      → Too short → false positives
      → Too long → slow detection
      → Cannot adapt to changing conditions
    → Phi-accrual: probabilistic, adaptive
      → Learns heartbeat distribution from data
      → Provides continuous confidence (φ)
      → Honest about uncertainty (never 100% certain)
  → Need to maintain cluster membership
    → O(N²) naive approach doesn't scale
    → Gossip protocols: O(N log N)
      → SWIM: suspicion + dissemination
  → Membership changes trigger consensus
    → Bridge to Chapter 7
```

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Detection problem | Cannot know if a node is dead with certainty (Axiom 1) |
| Fixed timeouts | Simple but fragile — cannot adapt, binary output, false positives or slow detection |
| Phi-accrual | Probabilistic failure detector — models heartbeat distribution, computes suspicion level φ, adapts to conditions |
| φ threshold | Tunable parameter balancing false positives against detection speed |
| Gossip membership | O(N log N) dissemination — SWIM protocol, suspicion before death |
| Failure → consensus | Membership changes (especially leader death) trigger consensus protocol |

**Next:** We can detect failures, but we cannot yet *agree* on anything. Consensus — the subject of Chapter 7 — solves the problem of multiple machines agreeing on a value despite failures.
