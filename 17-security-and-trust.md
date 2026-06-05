# Chapter 17: Security and Trust

Every previous chapter assumed nodes that follow the protocol honestly. They may crash, they may be slow, they may be partitioned — but they do not lie, they do not collude, and they do not send conflicting messages to different peers.

This chapter examines what happens when this assumption is removed.

---

## 17.1 The Crash-Fault Assumption Revisited

Chapter 7 made explicit: the entire discussion of consensus, replication, and coordination (Chapters 7–14) assumed a crash-failure model. Under this model:

- Nodes may crash (stop responding).
- Nodes may recover (with their persisted state intact).
- Nodes may be slow or partitioned away from the cluster.
- BUT nodes will not send conflicting information, forge messages, or collude.

This assumption is reasonable when all nodes are controlled by a single organisation, running in a single data centre, with physical and network security in place. It is not reasonable when:

- Nodes are operated by different organisations (blockchain, cross-organisational workflows)
- Communication crosses untrusted networks (public internet, external APIs)
- An attacker has compromised some nodes (security breach, supply chain attack)
- The system must operate under adversarial conditions (financial systems, national infrastructure)

---

## 17.2 Crash-Fault vs Byzantine-Fault Tolerance

The crash-fault model and the Byzantine-fault model differ in what they assume about faulty nodes:

| Property | Crash-Fault Model | Byzantine-Fault Model |
|----------|------------------|----------------------|
| Faulty node behavior | Stops responding entirely | May behave arbitrarily |
| Message forging | Not possible | Possible (sends conflicting messages to different peers) |
| Collusion | Not possible | Possible (multiple faulty nodes coordinate) |
| Timing violations | Not considered | Possible (delays messages selectively) |
| Required replicas | 2f + 1 (f = tolerated failures) | 3f + 1 |
| Minimum quorum | f + 1 | 2f + 1 |

The critical difference: **in the Byzantine model, a faulty node can lie.** It can tell one peer "I accept your proposal" and a different peer "I reject your proposal." It can selectively delay messages to influence the outcome. Multiple faulty nodes can coordinate to subvert the protocol.

This makes Byzantine fault tolerance (BFT) fundamentally harder than crash fault tolerance — and more expensive. For the same failure tolerance:

| Tolerated Failures (f) | Crash-Fault Nodes Required | Byzantine Nodes Required |
|----------------------|---------------------------|------------------------|
| 1 | 3 | 4 |
| 2 | 5 | 7 |
| 3 | 7 | 10 |

The `3f + 1` requirement for Byzantine consensus (vs `2f + 1` for crash consensus) means that BFT systems require more nodes to tolerate the same number of failures. This is the cost of defending against liars.

---

## 17.3 Practical BFT: PBFT and HotStuff

**PBFT (Practical Byzantine Fault Tolerance)** was the first practical BFT protocol (1999). It tolerates f Byzantine failures with 3f + 1 nodes, using a three-phase protocol (pre-prepare, prepare, commit) and cryptographic signatures to detect lying.

PBFT's key insight: even though Byzantine nodes can lie, correct nodes can verify messages using digital signatures. If a node receives conflicting messages from the same source, it knows that source is faulty. The protocol's quorum intersection ensures that correct nodes can make progress despite lying nodes.

PBFT's limitation: it requires O(N²) messages per decision (each node sends to every other node). A 10-node cluster sends 100 messages per consensus round. A 100-node cluster sends 10,000. This limits scalability.

**HotStuff** (2019) improved PBFT's communication complexity to O(N) (linear) by using a leader-based approach similar to Raft, combined with threshold signatures. HotStuff is the consensus protocol used by the Diem (Libra) blockchain.

HotStuff's key innovation: instead of every node talking to every other node, the leader collects votes, aggregates them into a single signature (threshold signature), and broadcasts the aggregated result. This reduces O(N²) to O(N) at the cost of increased latency (more phases per round).

---

## 17.4 When Byzantine Fault Tolerance Matters

In practice, most production systems use crash-fault tolerance (Raft, etcd, Kafka, Kubernetes) because:
- **Cost.** Byzantine tolerance requires 3× more nodes and significantly more complex implementations.
- **Trust domain.** Most systems operate within a single trust domain (one organization, one data centre). The crash-fault model is sufficient.
- **Simplicity.** Raft is implementable by a competent engineer. PBFT and HotStuff require deep protocol expertise.

Byzantine fault tolerance becomes necessary when:

| Scenario | Why Crash-Fault Is Not Enough |
|----------|------------------------------|
| **Public blockchain** (Bitcoin, Ethereum) | Nodes are operated by unknown parties. Anyone can be malicious. |
| **Cross-organizational consensus** | Different companies operate nodes. Each has its own incentives. A node might lie for competitive advantage. |
| **Supply chain / provenance** | Different parties have different interests. A supplier might falsify records. |
| **Critical infrastructure** | State-level attackers may compromise some nodes. The system must survive active subversion. |
| **Financial settlement** | A financial institution might selectively delay or reorder transactions for its own benefit. |

If your system operates within a single organisation's data centre (Kubernetes cluster, internal microservices), crash-fault tolerance is sufficient. If your system spans organisational boundaries or faces adversarial conditions, Byzantine tolerance becomes necessary.

---

## 17.5 Trust Models

The choice of consensus protocol is a **trust model** decision: which nodes do you trust, and for what?

| Trust Model | Trusted Components | Unstrusted Components | Example |
|------------|-------------------|----------------------|---------|
| Crash-fault (same org) | All nodes follow protocol | None (nodes can crash) | etcd, Raft |
| Byzantine (open) | > 2/3 of nodes are honest | Any node can be malicious | Blockchain |
| Hybrid (permissioned) | vetted nodes are likely honest | External participants, client nodes | Consortium blockchain |
| Trusted execution (TEE) | Hardware (SGX, Nitro) | Operator of the machine | Confidential computing |

Each trust model corresponds to a different protocol choice and a different operational cost.

---

## 17.6 Design-Space Beat

**When is crash-fault tolerance (Raft) sufficient?**
- Single-organization data centre operations
- Microservices within a VPC
- Internal databases and coordination services
- The cost of a Byzantine attack is lower than the cost of BFT

**When is Byzantine tolerance necessary?**
- Multiple organizations jointly operate the cluster
- The consequence of a participant lying is catastrophic
- Public-facing network with unknown node operators
- Regulatory requirements for tamper-proof audit trails

**The cost spectrum:**

```
Complexity:   Raft (crash) < PBFT (Byzantine) < HotStuff (Byzantine)
Nodes:        3-5           < 4-7               < 4-7 (same as PBFT)
Throughput:   High          < Medium             < Medium (optimised)
Scalability:  ~7 nodes      < ~20 nodes          < ~100 nodes (HotStuff linear)
```

For the workloads covered in this course (Chapters 7–14), the crash-fault assumption is adequate. The Byzantine dimension is a separate axis — important for blockchain and cross-organisational systems, but distinct from the crash-fault distributed systems problems that constitute the core of the field.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Crash-fault model | Nodes crash but do not lie. Sufficient for single-org systems. |
| Byzantine-fault model | Nodes can behave arbitrarily. Requires 3f + 1 replicas. |
| PBFT | First practical BFT protocol. O(N²) messages. |
| HotStuff | Linear BFT via leader + threshold signatures. |
| When BFT matters | Cross-organizational, adversarial, public network contexts |
| Trust models | Crash → Permissioned BFT → Open BFT → TEE — different costs, different guarantees |

**Next:** Congratulations. You have traced every concept in distributed systems from the speed of light through consensus, replication, transactions, orchestration, observability, and verification. The ARCHITECTURE.md file in the repository describes the full course map.
