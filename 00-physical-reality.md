# Chapter 0: Physical Reality

This course derives every distributed systems concept from first principles. But first principles means we start before computers, before operating systems, before networks. We start with the physical universe — because every constraint that defines distributed systems is ultimately a physical constraint.

Three physical facts underpin everything that follows. They are not derived from anything. They are simply how the universe works.

---

## 0.1 Fact One: The Speed of Light is Finite

The speed of light in a vacuum is approximately 3 × 10⁸ metres per second. This is an upper bound on how fast information can travel. No signal, no particle, no causal influence can exceed this speed. This is not a technology limitation — it is a law of physics.

From Johannesburg to Frankfurt is approximately 8,500 km. The minimum round-trip time for any signal between these two locations is:

```
(2 × 8.5 × 10⁶ m) / (3 × 10⁸ m/s) ≈ 0.057 s ≈ 57 ms
```

No engineering optimisation, no faster hardware, no better protocol can reduce this below ~57 ms. It is a physically irreducible floor.

Within a single datacenter, distances are shorter (100–500 m) but the same logic applies. A signal crossing a datacenter takes ~3 µs at the speed of light in fibre (which is slower than vacuum — approximately 2 × 10⁸ m/s). Add switching, queuing, and processing delay at each intermediate node, and the latency within a datacenter is typically 100–500 µs.

This matters because local memory access on a single machine takes approximately 100 ns. A datacenter round trip is **1,000–5,000× slower** than local memory. The difference between "same machine" and "different machine" is not quantitative — it is qualitative.

**Consequence 1: Latency is irreducible.** There is always a nonzero lower bound on how quickly one machine can learn about another. You cannot perfectly synchronise state across machines in zero time.

**Consequence 2: You cannot distinguish "dead" from "slow" in finite time.** If a message from Machine A to Machine B is delayed, Machine B cannot know whether the delay is caused by a transient network condition or by Machine A having crashed. These two situations are observationally indistinguishable during the delay.

This second consequence is the most important idea in distributed systems. It appears again in Chapter 7 (FLP impossibility) and runs through the entire course.

---

## 0.2 Fact Two: Entropy Always Increases

The Second Law of Thermodynamics states that the entropy of an isolated system never decreases over time. In practical terms: systems tend toward disorder. Everything degrades. Nothing lasts forever.

This means that every physical component has a finite Mean Time Between Failures (MTBF):

| Component | Typical MTBF / endurance |
|-----------|-------------------------|
| Consumer SSD | 1,500 TBW (terabytes written) |
| Server DDR4 RAM | ~2 × 10⁶ hours (ECC-correctable errors common daily) |
| Data-centre power supply | ~5 × 10⁵ hours |
| Hard disk drive | ~5 × 10⁵ to 1 × 10⁶ hours (annual failure rate 1–5%) |
| Networking switch | ~2 × 10⁵ hours |
| Fan (server chassis) | ~5 × 10⁴ hours |

These numbers matter because they are not "worst case" — they are *average*. In a cluster of 1,000 machines with a 2% annual failure rate, you expect a machine failure approximately every 18 days. In a cluster of 10,000 machines, approximately every 2 days.

At scale, hardware failure is not an anomaly. It is a regular, expected event.

Over time, the following will happen with probability approaching 1:
- A disk will develop bad sectors
- A DIMM will produce a correctable ECC error, then an uncorrectable one
- A power supply will fail
- A network cable will be disconnected accidentally
- A CPU will overheat
- A cosmic ray will flip a bit in memory

**Consequence 1: Failure is expected, not exceptional.** Any system that assumes perfect hardware will fail in production. Distributed systems are designed around the assumption that components *will* fail.

**Consequence 2: Failures are, to first approximation, independent.** One disk failing tells you nothing about whether another disk in a different machine will fail. (This independence is an idealisation — in reality, failures are often correlated: shared power supplies, shared network switches, shared software bugs, shared deployment pipelines. The course returns to this tension in Chapter 4 and later chapters on multi-zone and multi-region architecture.)

**Consequence 3: Reliability cannot be proven; it can only be engineered for and measured.** Because failures are probabilistic, no amount of testing can *prove* a system will survive a given fault. You can only design for it, observe it under fault injection (Chapter 16), and update your confidence.

---

## 0.3 Fact Three: There is No Global Clock

Different machines experience time differently. Every physical clock drifts — quartz crystals oscillate at slightly different rates depending on temperature, manufacturing variance, and age. Two clocks that are perfectly synchronised at time T will disagree at time T + δ.

Network Time Protocol (NTP) attempts to synchronise clocks across machines, but:
- NTP sync precision is typically 1–50 ms over the internet
- Even within a datacenter, NTP sync is ~0.1–1 ms
- Clock drift between NTP sync intervals can be significant (a 20 ppm crystal drifts 1.7 seconds per day)
- Leap seconds, daylight saving changes, and NTP misconfiguration can cause time to jump backwards

More fundamentally, the speed of light imposes a limit on clock synchronisation. Two machines cannot simultaneously agree on "now" because the signal carrying the time information takes time to arrive. This is not a measurement error — it is a consequence of the structure of spacetime (the relativity of simultaneity). Two events that are simultaneous in one reference frame are not simultaneous in another moving relative to it.

In practice, this means:
- `timestamp_A` on Machine A and `timestamp_B` on Machine B cannot be directly compared with nanosecond precision
- You cannot determine the order of two events by comparing their wall-clock timestamps if they occurred on different machines
- If you need to order events across machines, you need a logical ordering scheme, not physical timestamps

**Consequence 1: Causal ordering depends on protocol design, not on accurate clocks.** Lamport's happened-before relation (Chapter 5) replaces physical time with logical time based on message delivery.

**Consequence 2: "Simultaneous" is not well-defined across nodes.** Two events may appear simultaneous on one node and ordered on another. This is the origin of the ordering problem in distributed systems.

**Consequence 3: Clock uncertainty can be modelled probabilistically.** Google's TrueTime (used in Spanner) exposes clock uncertainty as an explicit interval [earliest, latest] — acknowledging that the precise time is not a single value but a probability distribution. This is one of the places where probabilistic reasoning becomes necessary (see Chapter 5).

---

## 0.4 The Energy Constraint (bonus)

A corollary fact worth naming: maintaining state requires energy.

- **RAM** stores bits as charge in capacitors. This charge leaks — DRAM must be refreshed approximately every 64 ms. Losing power means losing all data in RAM within milliseconds.
- **Storage (SSD)** stores charge in floating-gate transistors. This is more stable than DRAM (retention measured in years) but still degrades with each write cycle (program/erase wear).
- **Storage (HDD)** stores bits as magnetic domains on a spinning platter. More stable than flash (decade-scale retention) but orders of magnitude slower and sensitive to physical shock.

This creates the memory hierarchy that every computer depends on:

| Layer | Technology | Access Time | Volatile? | Relative Cost/GB |
|-------|-----------|-------------|-----------|------------------|
| L1 cache | SRAM | ~1 ns | Yes | 1000× |
| L2 cache | SRAM | ~4 ns | Yes | 500× |
| L3 cache | SRAM | ~12 ns | Yes | 200× |
| RAM | DRAM | ~100 ns | Yes | 1× |
| SSD | NAND flash | ~10 µs | No | ~0.1× |
| HDD | Magnetic | ~10 ms | No | ~0.01× |

The existence of this hierarchy is a physical constraint — there is no single storage technology that is simultaneously fast, dense, non-volatile, and cheap. Every system, from a microcontroller to a global distributed database, must navigate this tradeoff.

---

## 0.5 From Physical Facts to Distributed Systems Axioms

The three physical facts translate into two load-bearing axioms plus a third that follows from the first two but is worth stating separately.

**Axiom 1: Asynchrony** (from Fact 1 + Fact 2)

There is no bound on message delay. A message can take arbitrarily long to arrive, and there is no way to distinguish "the node is dead" from "the message is still in transit" in finite time.

This follows from the finite speed of light (Fact 1) combined with unbounded queuing and processing delay at intermediate nodes (Fact 2 — hardware can fail or degrade, causing arbitrary delays). The two together mean that end-to-end delay cannot be bounded.

This is the source of the FLP impossibility result (Chapter 7) — in an asynchronous system, consensus cannot be guaranteed in finite time.

**Axiom 2: Partial failure** (from Fact 2)

Components fail independently, and failure is partial — some components fail while others remain healthy. One node dying tells you nothing about whether another has died. The network can partition, leaving both sides running and neither aware of the other.

This follows from entropy (Fact 2). Independence is an idealising assumption — the course interrogates it explicitly when discussing correlated failures (shared power, shared switches, shared bugs, shared deploys).

This is the source of the CAP theorem (Chapter 9) — during a network partition, you cannot simultaneously maintain consistency and availability.

**Axiom 3: No shared clock** (derived from Facts 1 and 3)

Different machines cannot perfectly agree on the current time. Ordering events across machines requires logical reasoning, not physical timestamps.

This follows from the speed of light (Fact 1 — time synchronisation signals take time to propagate) and the nature of physical clocks (Fact 3 — drift, NTP error, relativity of simultaneity). It is stated as a separate axiom only because the ordering problem (happens-before, Lamport clocks, vector clocks) is foundational to everything that follows in Chapters 5 and beyond.

---

## 0.6 The Relationship Between These Axioms and the Rest of the Course

Every concept in this course is a response to one or more of these axioms.

| Axiom | Problems It Creates | Solutions Derived |
|-------|-------------------|------------------|
| 1 (Asynchrony) | Cannot know remote state in finite time | Failure detection (Ch 6), FLP impossibility (Ch 7), Raft timeouts (Ch 7) |
| 2 (Partial failure) | Machines die independently, network can partition | Redundancy (Ch 4), Quorum (Ch 7), CAP theorem (Ch 9), replication (Ch 10), chaos engineering (Ch 16) |
| 3 (No shared clock) | Cannot order events by physical time | Happens-before (Ch 5), Lamport/vector clocks (Ch 5), TrueTime (Ch 5) |

The chain of derivation is:

```
Physics (c, entropy, no global clock)
  → Hardware is finite and mortal
    → Single machine isn't enough
      → Need multiple machines
        → Asynchrony + partial failure + no shared clock
          → Every distributed systems concept is a response to these axioms
```

---

## 0.7 Design-Space Beat

This layer has no design-space alternatives. The physical facts are not chosen — they are discovered. There is no alternative to the speed of light, no alternative to entropy, no alternative to clock drift. These are the constraints within which all engineering operates.

The only design choice is *how honestly you acknowledge them.* Systems that pretend failures are rare or that clocks are perfectly synchronised break in production. Systems that design from these axioms survive.

This is the meta-lesson of the course: **good engineering starts with acknowledging the constraints that cannot be changed.** Not fighting them. Not pretending they don't exist. Designing *within* them.

---

## Summary

Three physical facts drive everything in distributed systems:

1. **The speed of light is finite** — information takes time to travel. You cannot have instantaneous knowledge of remote state.
2. **Entropy always increases** — hardware fails. Components degrade. Nothing is perfectly reliable.
3. **There is no global clock** — different machines disagree on time. You cannot order events by timestamp alone.

These produce two load-bearing axioms (asynchrony, partial failure) and one derived principle (no shared clock). Everything else in this course — failure detection, consensus, replication, orchestration, chaos engineering — is a logical consequence of these facts.

The rest of the course begins with the machinery these facts forced into existence: the computer.
