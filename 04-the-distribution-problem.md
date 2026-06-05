# Chapter 4: The Distribution Problem

The first three chapters built a picture of a single machine: how it works, how it isolates workloads, and how it packages software. But a single machine has hard limits — limits imposed by physics and economics. This chapter explains why one machine is never enough, and why going beyond one machine changes everything.

---

## 4.1 Why One Machine Isn't Enough

Every machine has ceilings:

**Physical limits.** A single server has maximum RAM (typically a few terabytes), maximum CPU cores (typically 64–256), maximum storage (tens of terabytes attached directly). You cannot arbitrarily increase these — motherboard slots fill, power budgets are exhausted, chassis space runs out. A machine that processes 100,000 requests per second cannot be upgraded to process 1,000,000 requests per second by adding a single component; it requires a fundamentally different architecture.

**The single point of failure.** A machine *will* fail. Chapter 0 established this: every component has a finite MTBF. A 1,000-machine cluster expects a failure every few weeks. A single machine has no redundancy — when it fails, everything running on it stops. The only way to survive a machine death is to have another machine ready to take over.

**Physical scaling limits.** Moore's Law (transistor density doubles every ~2 years) is slowing. Clock speeds have plateaued (~4 GHz for consumer CPUs since 2005). Single-thread performance improves at ~3% per year. You cannot make a single CPU arbitrarily powerful. Scaling up (vertical scaling) has hard ceilings.

**Cost.** A machine that is twice as reliable costs approximately ten times as much (the reliability-cost curve is exponential, not linear). A machine with twice the capacity costs more than twice as much (component costs scale superlinearly at the high end). It is cheaper to buy many commodity machines than one ultra-reliable super-machine.

These constraints force a fundamental decision: either accept the limits of one machine, or use many machines together. Distributed systems choose the latter. But this choice introduces a set of problems that are not present in a single-machine system.

---

## 4.2 The Critical Leap: Machines Cannot Share Memory

Two programs running on the same machine share a common resource: RAM. Process A writes to address `0x1000`; Process B, on the same CPU, can read from address `0x1000` (with the appropriate permissions). They operate on the same physical hardware, connected by the same memory bus, synchronised by the same clock.

Two programs running on different machines do not have this. Machine A's CPU has direct access to Machine A's RAM. It does not have direct access to Machine B's RAM. The only way to get data from Machine B to Machine A is via the network.

This is the critical distinction:

| Aspect | Same Machine | Different Machines |
|--------|-------------|-------------------|
| Memory access | Direct (load/store instructions) | Via network messages |
| Latency | ~100 ns (RAM) | ~500 µs (datacenter), ~100 ms (geographic) |
| Bandwidth | ~50 GB/s (DDR5) | ~10–100 Gbps (network) |
| Reliability | Virtually zero packet loss | Packet loss, retransmission, timeouts |
| State sharing | Trivial (shared memory) | Requires consensus protocols |

The difference between "same machine" and "different machine" is not a matter of degree — it is a difference in kind. The fundamental assumptions of single-machine computing (instantaneous state sharing, reliable communication, a shared clock) do not hold across machines.

**The fundamental distributed systems problem:** You need coordination between machines, but coordination requires communication, and communication is unreliable and slow.

---

## 4.3 The Network

Communication between machines happens over a network. The network imposes three realities that single-machine computing does not confront:

**Latency.** It takes time for signals to travel between machines. Within a single datacenter, round-trip latency is 100–500 µs. Between continents, it's 50–300 ms. This is not fast enough for synchronous coordination at the rate of CPU clock cycles. A CPU running at 3 GHz executes 1,500,000 instructions in the time it takes for a datacenter round-trip. During a cross-continent round-trip, it executes 900,000,000 instructions.

**Unreliability.** Networks drop packets. They do so for many reasons: buffer overflow at a switch, electromagnetic interference on a cable, a misconfigured firewall, a failing network interface card, a cable accidentally disconnected. The network has no way to notify the sender that a packet was dropped. The sender can only detect the loss via a timeout (waiting long enough that the packet should have arrived) — which returns us to Axiom 1.

**Partition.** The network can split. A switch failure, a cable cut, or a software bug can separate a cluster into two groups that cannot communicate. Both groups are still running. Each believes the other has failed. Neither can know for certain that the other exists.

This is different from a node failure. When a node fails, one side of the conversation stops. When a partition occurs, both sides continue operating, unaware of each other — and possibly making conflicting decisions.

---

## 4.4 Partial Failure

In a single machine, failure is binary. The machine crashes (power loss, kernel panic, hardware fault) and stops working. Or it is working. There are edge cases (a hung process, a slow disk), but generally the unit of failure is the machine.

In a distributed system, failure is *partial*. The network drops a packet but both endpoints are healthy. One node is slow but not dead. A request is processed but the response is lost. A node receives a duplicate request because the first response timed out. The database is up but the authentication service is down. The system is neither fully working nor fully broken — it is in a state where some parts work and others don't, and the working parts cannot reliably determine which is which.

Partial failure is the defining problem of distributed systems. Every protocol, every consensus algorithm, every failure detector is designed to handle partial failure. There is no counterpart to this problem in single-machine computing.

**The deepest consequence of partial failure:** You can never be certain about the state of a remote system. Not perfectly, not in finite time, not without physical co-location. Every observation is evidence, not truth. This irreducible uncertainty is the thread that runs through every layer of the rest of this course.

---

## 4.5 The Three Axioms Crystallise

At this point in the course, the three axioms established in Chapter 0 move from theoretical to operational:

**Axiom 1 (Asynchrony)** is now a practical constraint. You need to know if a remote machine is alive. You send a heartbeat. It may arrive or it may be delayed. You cannot distinguish between "the machine is dead" and "the heartbeat was delayed" in finite time. This is not an edge case — it is the normal state of a distributed system.

**Axiom 2 (Partial failure)** is now the design assumption. Any component can fail at any time, independently of others. The network can partition. The system must continue operating correctly despite these failures. Since failures are partial, the system must handle every possible combination of failed and working components.

**Axiom 3 (No shared clock)** is now a practical limitation. To order two events that occurred on different machines, you cannot rely on their timestamps — the clocks may disagree. You need a logical ordering scheme that does not depend on physical time synchronisation.

These three axioms, operating together, define the difficulty of distributed systems:

> Machines cannot share memory. Communication is unreliable and slow. Components fail independently. There is no common clock. And you must build systems that work anyway.

---

## 4.6 Two Fundamental Approaches

Given these constraints, there are two philosophical approaches to building distributed systems:

**Approach 1: Pretend the network is reliable.** Use timeouts generously, retry on failure, and assume the system will converge. This works for some applications (web serving, content delivery) and breaks catastrophically for others (financial transactions, distributed consensus).

**Approach 2: Design for the network's worst behaviour.** Assume every message may be lost, every response may be a duplicate, every node may be dead, and every clock may be wrong. This is the approach taken by consensus protocols, distributed databases, and the rest of this course.

Approach 1 is simpler but unreliable. Approach 2 is harder but correct. The rest of the course is a systematic exploration of Approach 2.

---

## 4.7 What Changes When You Have N Machines

The move from one machine to many changes the questions you ask:

| Single Machine | Distributed System |
|---------------|-------------------|
| Is the process running? | Is the remote node alive? |
| What's in memory? | Which node has the latest data? |
| Execute this instruction. | Send this message and wait for acknowledgement. |
| The clock says 14:03:12. | The clocks disagree by 37 ms. |
| If I write to disk, it persists. | If I write to one node, is the data safe? |
| Everything runs or nothing does. | Some parts are running and others aren't. |

Each question maps directly to one of the three axioms:
- "Is the remote node alive?" → Axiom 1 (asynchrony — you cannot know for certain)
- "Is the data safe on one node?" → Axiom 2 (partial failure — that node can die)
- "The clocks disagree" → Axiom 3 (no shared clock)

---

## 4.8 A Note on RPC Semantics

Since all distributed systems communication is by messages, the semantics of sending and receiving messages matter. A **remote procedure call (RPC)** is the distributed analogue of a function call — one machine requests that another machine perform an operation and return a result. But an RPC is *not* a function call, because the network sits between caller and callee:

- **At-most-once semantics:** The caller sends the request once. If no response arrives within a timeout, the caller may try again or give up. Simple, but the caller cannot distinguish "the callee never received the request" from "the callee processed it but the response was lost."

- **At-least-once semantics:** The caller retries on timeout until it receives a response. Guarantees the request is processed at least once, but the callee may process it multiple times (duplicate execution). The caller must handle duplicates idempotently (see Chapter 11).

- **Exactly-once semantics:** The caller sends exactly once and the callee processes exactly once. This is the ideal but requires deduplication, idempotency checks, and often consensus — it is expensive and only justified for operations where duplication has material consequences (payments, inventory decrements).

These semantics follow directly from Axiom 1 (asynchrony — you cannot distinguish a lost message from a delayed one), which forces the caller to make a choice on timeout: retry (risking duplicates) or give up (risking the operation never happening). This choice is the root of the exactly-once problems discussed in Chapters 10 and 11.

---

## 4.9 Design-Space Beat

There is no alternative to addressing the distribution problem if you need more capacity or reliability than one machine provides. The only alternative is to stay with a single machine — which is the right choice for many workloads.

**When a single machine is sufficient:**
- The workload fits in one machine's RAM and storage
- The reliability requirements are below the MTBF of one machine
- The traffic fits within one machine's network and I/O capacity
- Any downtime can be tolerated until the machine is repaired or replaced

**When multiple machines are necessary:**
- The workload exceeds one machine's capacity (unlikely to shrink)
- The application must survive a machine failure (MTBF is fixed)
- The application serves users across geographic regions (latency is a requirement)

A significant fraction of software projects do not need distributed systems. A team that adds Kubernetes to a single-machine workload has not solved a distributed systems problem — they have added complexity without justification. The threshold is not "it would be cool to learn Kubernetes" but "one machine cannot handle this workload, and the cost of distribution is worth paying."

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Why one machine isn't enough | Finite capacity, single point of failure, physical scaling limits, cost |
| The critical leap | Machines cannot share memory — they communicate only via network |
| Network properties | Slow, unreliable, can partition |
| Partial failure | The defining problem of distributed systems — some parts fail while others work, and you cannot reliably tell which is which |
| Three axioms crystallise | Asynchrony, partial failure, no shared clock now operational constraints |
| Single machine vs distributed | Every question changes when the answer involves a remote machine |

**Next:** The first concrete consequence of no shared clock: we cannot order events by physical time. We need a logical ordering scheme. This is the subject of Chapter 5 — Time and Ordering.
