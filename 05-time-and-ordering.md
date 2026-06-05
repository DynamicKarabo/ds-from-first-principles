# Chapter 5: Time and Ordering

In single-machine computing, ordering events is trivial. If event A happens before event B on the same machine, a clock provides the order. Even if two events occur on different threads, a shared memory bus and a common clock let you determine sequence.

In distributed systems, there is no common clock (Axiom 3). Two events on two different machines cannot be ordered by their physical timestamps with any reliability. This chapter examines how distributed systems solve the ordering problem without a shared clock.

---

## 5.1 The Problem: No Global Clock

Chapter 0 established that different machines experience time differently. Clock drift, NTP sync error, and the physics of simultaneity mean that `time_A` on Machine A and `time_B` on Machine B cannot be directly compared.

Consider: Machine A records an event at its local time `14:03:12.000`. Machine B records an event at its local time `14:03:12.001`. Which event happened first?

You cannot answer this question without knowing the clock skew between the two machines. If Machine A's clock is 2 seconds ahead of Machine B's, the event on Machine B actually happened first despite having a later timestamp. If the clocks are perfectly synchronised (which they never are), Machine A's event happened first.

With unbounded clock skew — which is the practical reality — physical timestamps are useless for determining event order across machines.

This is not a measurement problem. It is not "our NTP implementation is imprecise." It is a fundamental consequence of the structure of spacetime: there is no global "now" that all observers agree on. Two events that are simultaneous in one reference frame are not simultaneous in another.

For practical engineering, we need a way to order events without relying on physical clock synchronisation.

---

## 5.2 The Happens-Before Relation (Lamport 1978)

Leslie Lamport solved the ordering problem in his 1978 paper "Time, Clocks, and the Ordering of Events in a Distributed System." His insight: we don't need physical time to order events causally. We need only a partial order based on message delivery.

The **happens-before** relation (denoted `→`) is defined by three rules:

1. If events A and B occur on the same process (machine), and A occurs before B in that process's local order, then `A → B`.

2. If event A is a process sending a message, and event B is another process receiving that message, then `A → B`. (The send must precede the receipt.)

3. If `A → B` and `B → C`, then `A → C` (transitivity).

Two events are **concurrent** (denoted `A || B`) if neither `A → B` nor `B → A`.

This is a *partial* order — some events are ordered relative to each other, and others are not. Events that are causally related (one could have influenced the other) are ordered. Events that are independent are concurrent.

This is the correct way to reason about time in a distributed system: not "which event happened first in physical time?" but "could one event have causally influenced the other?" If yes, the causal order must be preserved. If no, the order does not matter.

---

## 5.3 Lamport Clocks

A **Lamport clock** is a simple algorithm that assigns a logical timestamp to every event, respecting the happens-before relation.

Each process maintains a counter (initialised to 0):

1. Before executing an event, the process increments its counter: `L = L + 1`. The event's timestamp is the new value of `L`.

2. When a process sends a message, it includes its current counter value in the message.

3. When a process receives a message, it sets its counter to `max(L_local, L_message) + 1`.

This guarantees the **clock consistency property**: if `A → B`, then `L(A) < L(B)`. The converse is not true — if `L(A) < L(B)`, we cannot conclude `A → B`. Lamport clocks give us *one direction* of the ordering.

**Example:**
```
Process P1:  send → L=3
                ↓
Process P2:      receive → L = max(3, L2) + 1
```
If P1's Lamport clock was 3 when it sent, and P2's local clock was 1 when it received, P2 becomes `max(3, 1) + 1 = 4`. Any event on P2 after this point will have a Lamport clock greater than 4, which is greater than 3 (P1's send). This captures the causal relationship.

**Limitation:** Lamport clocks cannot detect concurrent events. If `L(A) = 3` and `L(B) = 4`, we know `A → B` is possible, but we don't know if it's actually true — B might be unrelated to A but happened to receive a later timestamp. Lamport clocks tell us "these events might be causally related" but not "these events are definitely concurrent."

---

## 5.4 Vector Clocks

Vector clocks extend Lamport clocks to capture both the ordering and the concurrency relation.

Each process maintains a *vector* of counters — one entry per process in the system. For a system with N processes, each process holds an N-element vector `V[]`.

1. Before executing an event, process `i` increments its own entry: `V_i[i] = V_i[i] + 1`.

2. When sending a message, process `i` includes its entire vector `V_i`.

3. When receiving a message with vector `V_m`, process `i` updates:
   - `V_i[j] = max(V_i[j], V_m[j])` for all `j`
   - `V_i[i] = V_i[i] + 1`

Event ordering with vector clocks:
- `V(A) ≤ V(B)` if every element of `V(A)` is ≤ the corresponding element of `V(B)`.
- `V(A) < V(B)` if `V(A) ≤ V(B)` and at least one element is strictly less.
- `A → B` if and only if `V(A) < V(B)`.
- `A || B` if neither `V(A) < V(B)` nor `V(B) < V(A)`.

Vector clocks give us *both directions* of the ordering and can detect concurrency:

- If `V(A) = [2, 1, 0]` and `V(B) = [2, 1, 3]`, then `V(A) < V(B)`, so `A → B`.
- If `V(A) = [2, 1, 0]` and `V(B) = [1, 2, 0]`, then neither is less than the other, so `A || B`.

**Cost:** Vector clocks require a known, fixed set of processes (the size of the vector). In a dynamic system where processes join and leave, the vector grows and must be re-sized. The communication overhead also grows — every message carries the full vector, which for a system with 100 processes is 100 integers per message. In practice, optimised versions (version vectors, dotted version vectors) reduce this overhead.

---

## 5.5 Physical Clocks in Practice: NTP

Logical clocks solve the ordering problem but they are not a complete replacement for physical time. We still need physical time for:
- Coordinating time-based actions (cron jobs, certificate expiry, cache TTLs)
- Human understanding (logs need wall-clock timestamps for debugging)
- External ordering (if a transaction happens at 14:03, the user needs to know when that is in their timezone)

Network Time Protocol (NTP) synchronises clocks across machines by querying a hierarchy of time servers, each stratum synchronising to the stratum above, with stratum 0 being atomic clocks or GPS receivers.

NTP accuracy depends on network distance:
- Directly connected to stratum 1: ~1 ms
- Within a datacenter: ~0.1–1 ms
- Across the internet: ~5–50 ms
- Across continents: ~10–100 ms

NTP can also cause clocks to *jump backwards* if it determines the local clock is too far ahead and corrects it. This can break systems that assume monotonic time — a timestamp generated after the jump may be numerically smaller than one generated before it.

**The practical rule:** Never trust a physical timestamp from a remote machine to make ordering decisions. Use it for human-readable logging and coarse-grained coordination, but rely on logical clocks for causal ordering.

---

## 5.6 TrueTime (Google Spanner)

Google's Spanner database introduced a novel approach: instead of trying to synchronise clocks perfectly (which is impossible), **expose the uncertainty as an explicit interval.**

TrueTime uses GPS receivers and atomic clocks on each machine, combined with careful synchronisation, to give each machine a clock with a known error bound. When Spanner asks for the current time, it receives not a single value but an interval `[earliest, latest]` — the true time is guaranteed to lie somewhere within this interval.

The uncertainty is typically 1–7 ms for GPS-based TrueTime (data centres spread across a continent) and can be smaller for atomic-clock-only configurations within a single building.

TrueTime enables a correctness guarantee called **external consistency** (linearizability with real time). Spanner assigns timestamps that are guaranteed to reflect the true order of events. The key mechanism:

1. Spanner reads the current TrueTime interval `[e, l]`.
2. It chooses a commit timestamp `t ≥ l` (later than the latest possible current time).
3. All replicas agree that `t` is the commit timestamp.
4. Before making the commit visible, Spanner waits until `TrueTime.Now().earliest > t` (the **commit wait**).

This ensures that any later transaction will see a TrueTime interval whose earliest value is greater than `t`, guaranteeing the correct ordering.

TrueTime is the place where probabilistic reasoning about clock synchronisation becomes explicit and rigorous. Instead of claiming "our clocks are synchronised to within X ms," TrueTime says "we don't know the precise time, but we know the bound on our uncertainty, and we design around that bound."

---

## 5.7 Design-Space Beat: Logical vs Physical vs TrueTime

There is no single best approach to time in distributed systems. The right choice depends on the correctness guarantees you need and the cost you can bear.

| Approach | Strength | Weakness | When To Use |
|----------|----------|----------|-------------|
| Lamport clocks | Simple, guarantees causal ordering | Cannot detect concurrency. Requires extension for dynamic membership | Ordering broadcast messages, total-order multicast |
| Vector clocks | Detects both ordering and concurrency | O(N) size per message. Requires fixed process set | Detecting conflicts in collaborative editing, Dynamo-style replication |
| NTP-based timestamps | Human-readable, no protocol overhead | Unbounded error, can jump backwards | Logging, monitoring, coarse-grained coordination |
| TrueTime | External consistency (real-time ordering) | Requires GPS/atomic clocks, ~1–7 ms uncertainty | Globally distributed transactions requiring linearizability |

**When logical time is sufficient:** Any system where events are causally related through messages. If all communication goes through explicit messages, Lamport clocks capture the causal order with minimal overhead. This covers most distributed systems.

**When physical time is necessary:** Systems that interact with the external world (certificate validation, legal records, financial settlement timestamps). These require wall-clock time, not logical time.

**When TrueTime is necessary:** Globally distributed databases that must provide linearizability (Spanner's use case). The cost is significant (GPS clocks in every datacenter, commit wait adds latency) but the guarantee cannot be achieved any other way.

---

## 5.8 The Relationship to Axiom 3

The entire problem of time and ordering is a direct consequence of Axiom 3 (no shared clock). If all machines shared a perfect global clock, there would be no need for Lamport clocks, vector clocks, or TrueTime — you would simply timestamp every event with the global time and compare timestamps.

But such a clock does not exist, cannot exist (relativity of simultaneity), and even approximations (NTP) have unbounded error. Logical time is the engineering response to a physical constraint.

The axiomatic thread is direct: no shared clock → cannot order by physical time → happens-before relation → Lamport clocks (partial ordering) → vector clocks (full ordering + concurrency) → TrueTime (bounded uncertainty as an alternative approach).

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Cannot order by physical time | Clock skew, NTP error, relativity make timestamps unreliable across machines |
| Happens-before relation | Causal ordering based on message delivery, not wall clocks |
| Lamport clocks | Per-process counters, guarantee `A → B` implies `L(A) < L(B)` |
| Vector clocks | N-dimensional vectors, detect both ordering AND concurrency |
| NTP | Approximates physical time but cannot guarantee correctness for ordering |
| TrueTime | Exposes clock uncertainty as a bounded interval — the honest approach to physical time |

**Next:** We can now order events. But we still don't know if a remote node is alive. That is the problem of failure detection — the subject of Chapter 6.
