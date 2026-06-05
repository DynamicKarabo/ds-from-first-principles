# Chapter 11: Distributed Transactions

A transaction is a sequence of operations that must execute atomically — either all succeed or all fail. On a single machine, transactions are enforced by a database engine using locking, logging, and recovery. Across multiple machines, atomicity requires coordination.

This chapter examines the protocols that provide transactional guarantees across nodes — and the fundamental limits that make them different from consensus.

---

## 11.1 The Atomicity Problem

Consider a payment: debit account A and credit account B. These two operations are on different machines (A on Node 1, B on Node 2). The system must ensure one of two outcomes:

- **Both succeed.** Money leaves A and arrives at B.
- **Both fail.** Money stays in A. B is unchanged.

It must not produce: debit succeeds on Node 1, but Node 2 crashes before crediting B. Money disappears from A without appearing in B. This is the **atomicity problem** — an operation spanning multiple nodes must commit everywhere or abort everywhere.

Atomicity is not the same as consensus (Chapter 7). Consensus asks "what value do we agree on?" Atomic commit asks "did this operation commit or abort?" The two are related but different — and the difference is crucial for understanding distributed transactions.

---

## 11.2 Two-Phase Commit (2PC)

The classic distributed commit protocol. A coordinator orchestrates two phases:

**Phase 1 — Prepare:**
1. Coordinator sends "prepare to commit" to all participants.
2. Each participant writes the transaction to its write-ahead log (Chapter 8) and responds "ready" or "abort."
3. "Ready" means "I have written the transaction to durable storage and can commit it." The participant is now in a prepared state and cannot unilaterally abort.

**Phase 2 — Commit:**
1. If all participants responded "ready," the coordinator writes "commit" to its log and sends "commit" to all participants.
2. If any participant responded "abort" or timed out, the coordinator writes "abort" to its log and sends "abort" to all participants.
3. Participants receive the decision, write it to their logs, and respond "acknowledged."

### The problem with 2PC

2PC is **blocking**. If the coordinator crashes after sending "prepare" but before sending the commit/abort decision, participants are stuck in the prepared state. They hold locks on the transaction's data. They cannot commit (they need the coordinator's decision). They cannot abort (they promised they would commit if asked). They can only wait.

A prepared participant that cannot reach the coordinator must hold its locks indefinitely. In a distributed system (Axiom 1 — asynchrony, Axiom 2 — independent failure), this means the participant can be blocked forever.

This is why **2PC is not consensus.** Consensus (Chapter 7) guarantees termination — a new leader is elected and the protocol continues. 2PC blocks on coordinator failure because there is no mechanism to replace the coordinator. The prepared participants cannot elect a new coordinator because they have made an irrevocable promise (the "ready" vote) that only the original coordinator can resolve.

2PC solves the *agreement* problem (all participants commit or all abort) but does not solve the *termination* problem (the protocol must eventually complete). Consensus solves both.

---

## 11.3 Three-Phase Commit (3PC)

3PC adds a third phase to 2PC to avoid blocking:

**Phase 1 — CanCommit:** Coordinator asks "can you commit?" (non-binding). Participants respond yes/no.
**Phase 2 — PreCommit:** If all responded yes, coordinator sends "prepare to commit." Participants acknowledge and enter prepared state.
**Phase 3 — DoCommit:** After all acknowledgements, coordinator sends "commit."

The key addition: a timeout in the prepared state. If a participant does not receive the final commit/abort decision within a timeout, it unilaterally aborts (because it cannot be the only one to commit). If enough participants time out and abort, the transaction is aborted even if others were ready to commit.

3PC avoids the blocking problem of 2PC — participants can recover via timeout. But it introduces a new failure mode: if the coordinator sends "precommit" to some participants and crashes before sending it to others, the participants that received "precommit" will commit (after the coordinator restarts and resolves), while those that did not will abort (on timeout). This violates atomicity if the timeout is not coordinated perfectly — which it cannot be in an asynchronous system.

3PC is **not** used in practice for this reason. The theoretical safety improvement over 2PC does not hold under realistic failure assumptions.

---

## 11.4 The Saga Pattern

A saga is a sequence of local transactions, each with a compensating action. If a transaction fails, the compensating actions of all previous transactions are executed to undo their effects.

```
Example: Book a flight, hotel, and car rental. All must succeed or none.

If flight succeeds, hotel succeeds, but car rental fails:
  → Run hotel compensation (cancel hotel booking)
  → Run flight compensation (cancel flight booking)
  → Saga completes with all bookings cancelled.
```

**Compensating transaction:** An operation that semantically undoes a prior operation. "Cancel hotel booking" compensates "book hotel." The compensation is not guaranteed to restore the exact prior state — it may fail (e.g., the hotel booking was non-refundable, and the compensation is "issue a credit note," not "un-book the room").

**Saga vs 2PC:**

| Property | 2PC | Saga |
|----------|-----|------|
| Isolation | Serializable (locks held until commit) | None (intermediate state visible) |
| Atomicity | Guaranteed (write-ahead log + coordinator) | Best-effort (compensations may fail) |
| Blocking | Yes (on coordinator failure) | No (each step is independent) |
| Latency | O(N) round-trips | O(1) per step (parallelisable) |
| Failure handling | Atomic abort | Compensating actions |

The saga pattern accepts that intermediate state may be visible (lack of isolation) and that compensations may not perfectly restore the original state. In exchange, it avoids the blocking problem of 2PC and allows long-running transactions (hours or days) that are impractical with 2PC's lock-holding approach. Sagas are the dominant pattern for microservice transactions.

---

## 11.5 Idempotency

Idempotency is the practical bedrock of correctness in distributed systems. An operation is idempotent if applying it multiple times produces the same result as applying it once.

- `SET balance = 100` is idempotent (run it 10 times → balance is 100).
- `ADD 10 TO balance` is NOT idempotent (run it 10 times → balance increases by 100).
- `INVOICE_ID = UUID()` is NOT idempotent (each call produces a different UUID).

Idempotency is achieved by assigning an **idempotency key** to each operation. Before processing an operation, the system checks if this key has been seen before. If yes, return the previous result (idempotent replay). If no, execute and store the result keyed by the idempotency key.

This is how payment systems work: every payment request carries a unique `Idempotency-Key` header. If the payment service receives the same key twice (due to a network retry), it returns the result of the first execution instead of processing the payment again.

Idempotency is essential because networks are unreliable (Axiom 1 — asynchrony). Clients retry on timeout. Without idempotency, a retry causes duplicate execution.

---

## 11.6 Design-Space Beat

**When is 2PC the right choice?**
- Short-lived transactions (milliseconds, not hours)
- Small number of participants (2–5)
- Reliable coordinator (single machine with high MTBF)
- Strong isolation required (no intermediate state visible)

2PC is appropriate for a small, fast transaction within a single datacenter where the coordinator is stable. It is inappropriate for long-running, multi-service workflows across unreliable infrastructure.

**When must you use Sagas?**
- Long-running transactions (hours, days)
- Many participants (microservice orchestration)
- Unreliable infrastructure (any component can fail)
- Isolation is acceptable to sacrifice

Sagas are appropriate when the transaction spans organisational boundaries, runs for a long time, or involves heterogeneous systems.

**When is a simple retry enough?**
- Network flakiness (timeouts, dropped connections)
- Idempotent operations already in place
- No atomicity requirement across systems

A simple retry is often the right answer for transient failures. Complex commit protocols should only be added when the failure of a single operation creates inconsistency that a retry cannot fix.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Atomicity problem | Multiple nodes must commit or abort together — partial failure makes this hard |
| 2PC | Prepare → commit/abort. Blocking on coordinator failure (not consensus) |
| 2PC vs consensus | 2PC = agreement but no termination. Consensus = both. |
| 3PC | Avoids blocking but introduces new failure modes — not used in practice |
| Sagas | Compensating transactions — no isolation, best-effort atomicity |
| Idempotency | Operation IDs prevent duplicate execution — practical bedrock |

**Next:** Data must be distributed across nodes for scale (not just for replication). Chapter 12 — Data Partitioning.
