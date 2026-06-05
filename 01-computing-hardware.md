# Chapter 1: Computing Hardware

The previous chapter established the physical constraints within which all computing operates. This chapter traces how those constraints produce the building blocks of a computer — and what a "node" actually is from a distributed systems perspective.

The goal is not a complete course in computer architecture. It is to establish: what a computer fundamentally *is*, what it can and cannot do, and which properties matter when connecting many of them together.

---

## 1.1 The Transistor

A transistor is a voltage-controlled switch. Apply a voltage to the gate terminal, and current flows between source and drain. Remove the voltage, and current stops.

This is a physical device. It operates at the quantum-mechanical level (electrons moving through doped silicon). It has physical limits: switching speed is bounded by capacitance and electron mobility, and every switch dissipates heat. These limits — speed, heat, density — are the physical constraints that define the capabilities of any computer.

Two states. On or off. That is all a transistor provides. Everything else in computing is built from organising billions of these switches in patterns.

---

## 1.2 From Transistors to Logic

Combine transistors in patterns to create **logic gates**. 

- **AND gate**: output is 1 only if both inputs are 1.
- **OR gate**: output is 1 if either input is 1.
- **NOT gate**: output is the inverse of input.

From these three (AND, OR, NOT), you can build any Boolean function. Any mathematical operation, any decision, any computation can be expressed as a combination of these three logic gates.

This is the first abstraction: the transistor switch becomes the logic gate. The gate hides the physics and presents a logical operation.

**XOR gate**: output is 1 if inputs differ. Built from AND, OR, and NOT. XOR is important because it is the core of addition — `A XOR B` gives the sum bit, `A AND B` gives the carry bit. One-bit addition requires exactly six transistors.

From XOR and AND, you build an **adder**. From adders, you build an **Arithmetic Logic Unit (ALU)**. From the ALU plus control logic, you build the data path of a CPU.

This is a key derivation chain: transistor → gate → adder → ALU → CPU. Every step follows logically from the previous one.

---

## 1.3 The Flip-Flop: Creating Memory

Logic gates alone are not enough for computation. A circuit with no memory can only transform current inputs to current outputs (combinatorial logic). It cannot remember what happened before.

A **flip-flop** is two gates whose outputs feed back into each other's inputs. This creates a circuit with two stable states — it "remembers" one bit.

This is the second great abstraction after the logic gate. The logic gate hides physics and exposes Boolean logic. The flip-flop hides Boolean logic and exposes *state* — the ability to remember across time.

From flip-flops, we build **registers** (groups of flip-flops sharing a clock signal) and **memory arrays** (grids of storage cells with addressing logic).

Every computer is built from exactly two things: combinatorial logic (gates) for transforming data, and sequential logic (flip-flops) for storing data.

---

## 1.4 The Clock

Multiple flip-flops must update their state simultaneously for the circuit to behave predictably. The signal that coordinates this is the **clock** — a regular oscillating signal that tells every flip-flop "update now."

The clock exists because signals take time to propagate through gates. Different paths through the circuit take different amounts of time. If flip-flops updated whenever their inputs settled, the circuit would produce wrong results — later signals would override earlier ones. The clock forces all updates to happen at discrete, synchronised moments.

Clock speed is limited by the **critical path** — the longest path any signal must travel through gates between two clocked updates. This is determined by: gate delay (transistor switching speed), wire delay (RC constant of the interconnect), and fan-out (how many gates the signal drives).

The clock is where physics directly constrains performance: the speed of light in silicon, the capacitance of wires, and the heat from switching all limit how fast a clock can run.

---

## 1.5 The CPU

The Central Processing Unit orchestrates everything. At its core is the **fetch-decode-execute** cycle:

1. **Fetch**: read the next instruction from memory (at the address stored in the Program Counter register).
2. **Decode**: determine what operation this instruction encodes (add, load, store, branch, etc.).
3. **Execute**: perform the operation (activate the ALU, access memory, update registers).
4. **Increment the Program Counter** and repeat.

This is the entire computer, reduced to a loop. The CPU does not "know" anything. It does not "understand" your program. It reads numbers from memory, decodes them as instructions, and performs the corresponding operation on other numbers. Three billion times per second.

**The Von Neumann bottleneck:** Instructions and data share the same memory bus. The CPU can either fetch an instruction OR read/write data — not both simultaneously. This creates a bandwidth bottleneck that limits performance regardless of CPU speed. The gap between CPU speed and memory speed has grown from roughly equal in the 1980s to a factor of ~100–1000 today.

The **cache hierarchy** exists because of this gap. CPU registers (~1 cycle) → L1 cache (~4 cycles, ~32 KB) → L2 cache (~12 cycles, ~256 KB) → L3 cache (~40 cycles, ~8 MB) → RAM (~100–300 cycles, up to hundreds of GB). Each level is a tradeoff between speed and size — a direct consequence of the physics of signal propagation and the economics of memory technology.

---

## 1.6 RAM

Random Access Memory stores data in arrays of capacitors (DRAM) or cross-coupled inverters (SRAM). Each bit is a tiny charge stored in a capacitor that leaks over time. DRAM must be refreshed every ~64 ms — reading and rewriting every row in sequence — or the data decays.

This is why RAM is **volatile**. The charge leaks. Remove power, and within milliseconds, all data is gone. This is not a design flaw — it is a direct consequence of storing information as electric charge in a capacitor.

RAM is fast relative to storage (~100 ns access time) because it is electrically close to the CPU and because its addressing logic is optimised for random access. But it is:
- Volatile (loses state on power loss)
- Expensive per GB (relative to storage)
- Dense enough for working data sets (tens to hundreds of GB per machine)

---

## 1.7 Storage

Storage retains data without power. Two dominant technologies:

**SSD (Solid State Drive):** Stores charge in floating-gate transistors. A floating gate is an electrode surrounded by insulating material, buried within the transistor. Electrons tunnel through the insulator when a high voltage is applied, becoming trapped in the floating gate. The trapped charge shifts the transistor's threshold voltage, representing a stored bit.

The insulator degrades with each write. Typical consumer SSDs endure 1,500 TBW (terabytes written). After that, the insulator can no longer reliably trap charge. This is a physical wear-out mechanism — SSDs have finite write endurance.

SSD access time is ~10 µs — approximately 100× slower than RAM — because reading a floating-gate transistor is slower than reading a DRAM capacitor (the charge is harder to sense), and because the controller must manage wear leveling, error correction, and bad block mapping.

**HDD (Hard Disk Drive):** Stores bits as magnetic domains on a spinning platter. A read/write head on a mechanical arm moves to the correct track (seek time ~5–10 ms) and waits for the platter to spin the correct sector under the head (rotational latency ~2–5 ms). Total access time ~10–20 ms — approximately 100,000× slower than RAM.

HDDs are cheaper per GB than SSDs and have higher write endurance, but are sensitive to physical shock and have much higher latency due to mechanical movement.

---

## 1.8 The Three Non-Negotiables

Every general-purpose computer has three components that together define what it means to "compute":

1. **CPU** — processes data and executes instructions
2. **RAM** — stores data temporarily, with fast access, but loses everything on power loss
3. **Storage** — stores data permanently, with slower access, but survives power loss

Remove any one, and you do not have a general-purpose computer. A CPU without RAM cannot store intermediate results. A CPU with RAM but no storage cannot load a persistent operating system or user data. Storage without a CPU is just a brick of data with no processing capability.

These three components map to the three things any computing system must do:
- **Compute** (transform data)
- **Remember transiently** (hold working state during computation)
- **Remember permanently** (retain data across power cycles)

---

## 1.9 What Is a Node?

In distributed systems, a "node" is often defined in hardware terms: a computer with a CPU, RAM, and storage, connected to a network. This is a useful intuition but not the right definition for understanding distributed systems.

The correct definition for this course:

> **A node is an independent failure domain with its own clock that communicates only by messages.**

Each property is directly load-bearing:

- **Independent failure domain** — One node's failure does not imply another's. This follows from Axiom 2 (partial failure). If nodes could fail in lockstep, most distributed systems problems would disappear. The fact that they fail independently is what makes coordination necessary and difficult.

- **Own clock** — Each node experiences time differently. This follows from Axiom 3 (no shared clock). If all nodes shared a perfect global clock, the ordering problem would be trivial.

- **Communicates only by messages** — Nodes do not share memory. They cannot directly read each other's RAM. All communication happens over a network. This is the root of the asynchrony problem (Axiom 1) — messages take time, can be lost, and can be delayed.

The hardware definition (CPU + RAM + Storage) is true but incomplete. It describes what a node is made of. The failure-domain definition describes what a node *means* in a distributed system — and that meaning is what drives every design decision in the course.

**A note on edge cases:** This definition includes machines without disks (diskless workers that boot over PXE), machines with disaggregated memory (CXL-attached memory pools), and ephemeral serverless containers (firecracker microVMs). Each is still a failure domain with its own clock that communicates by messages. The hardware definition is a simplification that covers the common case; the failure-domain definition is the general one.

---

## 1.10 The Memory Hierarchy as a Physics Constraint

The memory hierarchy is not a design choice. It is a consequence of physics: the faster a memory technology is, the more expensive and less dense it is. This follows from:

- **Speed of light** — signals take time to travel. Larger memory arrays are physically farther from the CPU (longer wires, more gates between you and the data).
- **Energy requirements** — faster memory uses more power (SRAM requires 4–6 transistors per bit vs DRAM's 1 transistor + 1 capacitor per bit). Heat dissipation limits density.
- **Entropy** — less storage technologies are more reliable, every storage technology degrades over time. Error correction codes (ECC) are necessary at scale.

The specific numbers change with technology generations, but the hierarchy itself does not. There will never be a single storage technology that is simultaneously fast as L1 cache, large as an HDD, and cheap as both. The tradeoff is physically enforced.

This matters for distributed systems because:
- Memory-bound workloads are constrained by how much RAM a single node can hold
- Storage-bound workloads are constrained by disk throughput and capacity per node
- The gap between RAM and storage speed (~100,000×) means that data in RAM is fundamentally different from data on disk — surviving a node failure means getting data from one to the other

---

## 1.11 What This Chapter Established

1. Everything in computing is built from one physical primitive (the transistor switch) organised in patterns

2. A computer has exactly three essential components: CPU (compute), RAM (fast transient state), Storage (slow persistent state)

3. A node in a distributed system is defined not by its hardware but by its properties: independent failure domain, own clock, communicates only by messages

4. The memory hierarchy is a physical constraint, not a design choice — it exists because faster memory is more expensive and less dense than slower memory

**Next:** Now that we know what a single computer is, we examine the problems that arise when multiple programs try to share one — the operating system.
