# Chapter 2: The Operating System

A single machine, by itself, is just hardware. It runs one program at a time — boot firmware, then an OS loader, then whatever program the OS loads. But a useful machine must run many programs simultaneously and keep them from interfering with each other. The operating system is the layer that makes this possible.

The constraints are physical. Multiple processes share one CPU (via time-division multiplexing), one RAM pool (via space-division multiplexing), and one set of storage (via abstraction). They must be isolated from each other — a crash in one process must not bring down another. They must share resources fairly. They must not corrupt each other's data.

---

## 2.1 The Problem: Multiple Programs, One Machine

If you run two programs on a single CPU without an OS, they:

1. **Conflict over CPU time.** Program A loops forever. Program B never runs.
2. **Conflict over memory.** Program A writes to address `0x1000`. Program B also writes to address `0x1000`. They corrupt each other's data silently.
3. **Conflict over I/O.** Both programs try to write to the same disk sector. Data is corrupted.
4. **Propagate failures.** Program A dereferences a null pointer. It crashes. But since nothing isolated it from the rest of the machine, the crash may corrupt the state of Program B or leave the machine in an unrecoverable state.

These are not software bugs. They are structural problems that emerge from sharing physical resources. The OS solves each one.

---

## 2.2 Process Isolation

The OS gives each process its own **virtual address space**. Process A sees addresses `0x00000000` to `0xFFFFFFFF` (on a 32-bit system) contiguously. So does Process B. But these are virtual addresses — the **Memory Management Unit (MMU)** translates them to physical RAM frames.

If Process A writes to `0x1000` and Process B writes to `0x1000`, the MMU maps those virtual addresses to *different* physical frames. Neither process can see or corrupt the other's memory.

**Why the MMU exists:** Without virtual memory, every program would need to know at compile-time which physical addresses it would use, and two programs would have to coordinate their address usage. This is impractical. The MMU provides indirection — the physical location of data can change without the program knowing.

The MMU is a hardware unit (part of the CPU), but the OS manages it. The OS sets up page tables that define the mapping for each process. Switching processes (context switch) requires flushing the TLB (translation lookaside buffer — a cache for virtual-to-physical mappings) and loading a different set of page tables.

**The cost of isolation:** Context switches take time (1–10 µs). Each process switch means saving all registers, flushing caches, and loading the new process's state. This is a real performance cost — which is why threads exist (see below).

---

## 2.3 System Calls: The Controlled Gateway

A process cannot:
- Write to disk directly
- Send data over the network
- Allocate physical memory
- Create another process
- Access another process's memory

All of these operations require the OS kernel. The **system call** interface is the controlled gateway through which processes request privileged operations.

Hardware enforces this with **protection rings** (typically ring 0 for the kernel, ring 3 for user processes). The CPU will not execute certain instructions or access certain memory regions unless the current privilege level is ring 0. To switch from ring 3 to ring 0, the process executes a special instruction (e.g., `syscall` on x86-64), which triggers a trap into the kernel.

The kernel verifies the request:
- Does the process have permission to write to this file?
- Is the memory address the process passed valid?
- Does the process have the capability to create a network socket?

Only after verification does the kernel perform the operation on the process's behalf. This is the **trust boundary** of the entire system — every crossing from user code to kernel operation is mediated.

**Why system calls are necessary:** Without them, any process could read any disk sector, corrupt any file, access any network packet, or halt the entire machine. System calls are the enforcement mechanism for all OS security and isolation policies.

---

## 2.4 Virtual Memory

Beyond isolation, virtual memory provides:

**Oversubscription (swapping):** The OS can use storage (disk/SSD) to extend the apparent size of RAM. When physical RAM is full, the OS writes some pages to a swap area on storage and reclaims the physical frames. When a swapped-out page is accessed, a page fault occurs, and the OS loads it back into RAM (possibly swapping out a different page).

The cost is enormous — accessing storage is 100,000× slower than RAM (Chapter 1). Swapping is not a plan for performance; it is a safety net against OOM.

**Shared memory:** Two processes can map the same physical frame into their respective virtual address spaces. This allows inter-process communication at RAM speed (no system call for each data transfer). This is how most IPC mechanisms (pipes, shared buffers) work at the lowest level.

**Memory-mapped files:** A file on storage can be mapped into a process's virtual address space. Reading from that address range triggers a page fault, which loads the relevant file content into RAM. Writing to that range modifies the file. This unifies the file and memory abstractions — a file becomes a persistent range of virtual addresses.

---

## 2.5 File Systems

Storage presents a flat array of blocks. A file system imposes structure:

- **Directories** — hierarchical naming (instead of finding data by block number)
- **Files** — named sequences of bytes (instead of raw blocks)
- **Metadata** — timestamps, permissions, ownership, size
- **Allocation tracking** — which blocks belong to which files, which blocks are free

The file system exists because humans and programs cannot reason about data at the block level. The abstraction hides the physical layout and presents a tree of named objects.

The key insight: a file system is a **database of blocks**. It maps a hierarchical name space to a set of physical storage locations. It must maintain consistency across crashes — which is why journaling file systems write a log of intended changes before applying them. The log (write-ahead log) is one of the most important ideas in distributed systems, appearing again in Chapter 8 (the log as unifying abstraction) and Chapter 11 (transactions).

---

## 2.6 Process vs Thread

A **process** has its own virtual address space, file descriptors, signal handlers, and security context. Creating a process requires duplicating (or copy-on-write mapping) an entire address space — expensive.

A **thread** shares the address space of its parent process. All threads in a process see the same memory, the same file descriptors, the same permissions. Creating a thread is cheap (just allocate a new stack and a thread control block).

| Aspect | Process | Thread |
|--------|---------|--------|
| Address space | Independent | Shared |
| Creation cost | High (address space copy) | Low (stack only) |
| Isolation | Full | None (one thread can corrupt another) |
| IPC | Requires kernel mediation | Shared memory (direct) |
| Context switch | ~1–10 µs | ~0.1–1 µs |

The tradeoff: isolation vs efficiency. Processes are safer but slower to create and switch between. Threads are faster but offer no protection — a null-pointer dereference in one thread can corrupt data used by another.

---

## 2.7 Kernel vs Userspace

The **kernel** runs in privileged mode (ring 0). It has direct access to all hardware — CPU instructions, memory, I/O ports, interrupt controllers. It manages processes, memory, devices, and system calls.

**User space** runs in unprivileged mode (ring 3). Processes cannot execute privileged instructions. They cannot access kernel memory. They cannot directly interact with hardware. Every privileged operation requires a system call.

The separation exists because:
1. A bug in user space should not crash the entire machine
2. A misbehaving process should not corrupt another process's data
3. Hardware access must be coordinated (two processes cannot simultaneously claim the same disk sector)

The kernel is the smallest amount of code that must run in privileged mode. Everything that can run in user space should — this is the principle of least privilege applied to the OS architecture.

Linux implements this separation strictly. The kernel is a monolithic binary, but drivers can be loaded as modules (still kernel code, running in ring 0). User-space drivers exist for some peripherals (USB, some storage controllers) — trading performance for safety.

---

## 2.8 Derivation Chain for This Layer

```
Multiple programs on one machine
  → Programs conflict over resource usage
    → OS must arbitrate resource access
      → CPU: scheduler divides time
      → RAM: virtual memory isolates address spaces
      → Storage: file system provides namespace + allocation
      → Devices: kernel mediates all I/O via system calls
    → OS must survive program crashes
      → Process isolation (address spaces, ring separation)
      → Supervisor/kernel stays up when user programs crash
```

The OS is the first abstraction layer that makes a single machine usable for multiple concurrent workloads. It does not remove the physical constraints — it manages them.

---

## 2.9 Design-Space Beat

The OS design described here (process isolation, virtual memory, kernel/userspace separation) is not the only possible arrangement. Real alternatives existed and still exist:

**Monolithic kernel (Linux, BSD):** All OS services run in kernel space (ring 0). Efficient (no IPC overhead between subsystems) but large attack surface.

**Microkernel (Minix, QNX, seL4):** Only the bare minimum (IPC, scheduling, address space management) runs in kernel space. File systems, device drivers, and protocol stacks run in user space as separate processes. More secure and fault-isolated but slower (IPC between subsystems requires context switches).

**Exokernel (MIT research):** Eliminates abstractions entirely. The kernel only multiplexes hardware resources. Applications manage their own abstractions (their own virtual memory, their own file system). Maximum flexibility, minimum safety guarantees.

The winner in practice (Linux, monolithic) won because performance was more important than fault isolation for most workloads, and because the monolithic architecture was simpler to develop and maintain. The tradeoff accepted: a kernel panic in a device driver takes down the entire machine. This tradeoff is increasingly being revisited with user-space networking (DPDK) and storage (SPDK) for latency-critical workloads.

---

## Summary

The operating system is the layer that makes one machine usable by multiple programs simultaneously. It solves four problems:

| Problem | Solution |
|---------|----------|
| Resource conflicts | Scheduler + virtual memory + file system |
| Process interference | Virtual address spaces (MMU) + protection rings |
| Safety/security boundary | System calls as controlled gateway |
| Failure propagation | Process isolation (kernel survives, user processes don't) |

**Next:** The OS solves isolation *within* one machine. But the real problem for distributed systems is isolation *across* machines — packaging software so it can run anywhere without drifting from its intended configuration. This is the problem containers solve.
