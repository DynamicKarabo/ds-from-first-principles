# Chapter 3: The Dependency Problem (Containers)

The OS solved the intra-machine isolation problem: multiple processes can share one machine without corrupting each other's memory or crashing the kernel. But it did not solve a subtler problem: the environment gap.

This chapter examines that gap, the field of solutions, and why one solution (containers + Docker) became the dominant packaging format — a choice that fundamentally shaped how distributed systems are built and deployed.

---

## 3.1 The Environment Gap

Your application works on your laptop. You deploy it to production. It breaks.

Why? Consider the things that differ between environments:

| Thing That Differs | Your Laptop | Production Server |
|-------------------|-------------|-------------------|
| OS | macOS 15 | Ubuntu 24.04 |
| Python version | 3.12 | 3.9 |
| Installed libraries | 47 packages (pip list) | 14 packages (server baseline) |
| File paths | `/Users/you/project/data/` | `/var/app/data/` |
| Users/groups | Your user (sudo) | `www-data` (no sudo) |
| Available memory | 32 GB | 2 GB |
| /tmp directory | Writable | `noexec` mount |
| System packages | homebrew installs | apt packages, different versions |
| Timezone/date | SAST, correct | UTC, might drift |
| Networking | Localhost, no firewall | Security groups, SELinux |

Each difference is a potential failure mode. Individually, each one is minor. In combination, they guarantee that production differs from development in ways that break the application.

This is the **environment gap** — the difference between where software is built and where it runs. Before containers, the standard response was: "production support team, please make production look like the developer's machine." This is impractical (production machines are shared, managed, and secured differently) and fragile (the production team doesn't know what the developer's machine actually looks like).

---

## 3.2 The Field of Viable Answers

The environment gap is not new. Solutions evolved over decades:

### Static Binaries
Compile all dependencies into a single binary. Works for Go, Rust, and C/C++ (with care). Does not work for interpreted languages (Python, Ruby, JavaScript) or for software that depends on system libraries (glibc version, OpenSSL, libcurl).

**Where it works:** Go services, Rust tools, C/C++ network daemons.
**Where it fails:** Python data pipelines, Ruby on Rails apps, Node.js microservices with native modules.

### Chroot
The `chroot` system call changes a process's view of the filesystem root. A process sees `/` as a specific directory tree, not the real root. This can isolate filesystem access — a web server in a chroot jail cannot access files outside its designated directory.

**Limitations:** chroot only isolates the filesystem. The process still shares the same PID space, network stack, inter-process communication, and kernel. A chrooted process can escape the jail with root privileges (it's a simple filesystem redirect, not a security boundary).

### FreeBSD Jails
A more complete isolation mechanism. Each jail has its own filesystem, network stack, users, and processes. A process in a jail cannot see processes outside it, cannot bind to IPs outside its assigned range, and cannot mount filesystems belonging to the host.

**Limitations:** BSD-specific (not on Linux). Still shares the kernel — a kernel vulnerability compromises all jails.

### Virtual Machines (Full Virtualization)
A hypervisor (VMware, KVM, Xen) presents virtual hardware to a guest OS. Each VM runs its own full kernel, its own init system, its own network stack. Isolation is near-total — a kernel crash in one VM does not affect others.

**Cost:** Each VM runs a complete OS (2–4 GB RAM overhead just for the kernel and system processes). Boot time: 30–60 seconds. Density: ~10–20 VMs per physical host.

### Containers
Containers share the host kernel but isolate the process view via namespaces and cgroups. They are not virtual machines — there is no hypervisor, no guest kernel, no separate init system.

**Cost:** Negligible overhead (the kernel is already running). Boot time: milliseconds. Density: 100–200 containers per physical host.

This is the field of viable answers. Each trades off isolation against resource cost.

---

## 3.3 [DEEP DIVE] Why Docker Won

Docker did not invent containers. The underlying technologies (namespaces, cgroups) were in the Linux kernel since 2002 (namespaces) and 2007 (cgroups). LXC (Linux Containers) was released in 2008. Docker entered in 2013.

Why did Docker win over LXC and all earlier container systems?

### 1. Layered Image Format

Before Docker, containers were described by a list of packages or a base filesystem tree. Building a container meant installing packages into a directory. Updating meant rebuilding the entire tree. Sharing meant distributing a multi-gigabyte filesystem.

Docker introduced layered images (based on UnionFS/OverlayFS). Each Dockerfile instruction creates a new read-only layer. Layers are shared between images stored on the same machine. If 100 containers use the same Ubuntu base layer, that layer is stored once.

This changed the economics of distribution:
- Pulling a new version of a service that changes one line of code means downloading only the tiny changed layer, not the entire OS
- Images are composed of reusable parts (base OS, language runtime, app code) rather than monolithic blobs
- Layer caching makes builds incremental — if only the app code changed, only the app layer is rebuilt

### 2. Dockerfile as Convention

LXC required writing configuration files for each container — a mix of chroot-style rootfs paths, cgroup assignments, and network setup scripts. The configuration was different for every system and had no standard format.

Docker introduced the Dockerfile — a declarative, version-controllable recipe for building an image:

```dockerfile
FROM python:3.12-slim
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
```

Every Dockerfile has the same structure: start from a base, add files, run commands, declare the entry point. This single convention made containers *teachable* — you could explain Docker containers by showing a 6-line file.

### 3. Registry Ecosystem (Docker Hub)

Publishing and discovering container images had no standard mechanism before Docker. Docker Hub provided a single public registry — `docker pull nginx` worked immediately, without configuring sources, certificates, or authentication.

This created network effects: the more images on Docker Hub, the more valuable Docker became as a tool (you could just run existing software without installing it). The network effects killed competing packaging formats — if your favourite tool published only a Docker image, you had to use Docker.

### 4. Portability Across Environments

Docker's promise — "build once, run anywhere" — was not fully true (containers share the host kernel, so a Linux container cannot run on Windows without a VM). But it was *sufficiently* true: the same image ran identically on a developer's laptop, a CI server, and a production node, as long as all three had the same kernel architecture (Linux x86_64).

This was a dramatic improvement over the pre-Docker world where deployment instructions were N-page documents titled "How to set up the application on Ubuntu 20.04 with manual steps."

### 5. Speed

A Docker container starts in milliseconds. A VM starts in tens of seconds. This difference is qualitative — it changes what you can do. You can restart a container as part of a deployment rolling update. You can run one container per unit of work in a CI pipeline. Containers are ephemeral and cheap; VMs are persistent and expensive.

### The Tradeoff Docker Accepted

Docker containers are not fully isolated. They share the kernel with the host and with each other. This means:
- A kernel exploit compromises all containers on a host
- You cannot run a container with a different kernel version than the host
- You cannot run Windows containers on a Linux host (or vice versa without a VM)

This is the fundamental tradeoff: **isolation for efficiency.** Containers are fast and dense precisely because they share the kernel. VMs are isolated precisely because they don't.

The bet Docker made: for most use cases, kernel-level isolation is sufficient, and the efficiency gains are worth giving up full hardware isolation. This bet was correct for the dominant use case (Linux server workloads).

---

## 3.4 How Containers Work: Namespaces

Namespaces are a Linux kernel feature that gives each process (or group of processes) the illusion that they have their own isolated instance of a system resource. There are seven namespaces that matter for containers:

| Namespace | What It Isolates | Why Containers Need It |
|-----------|-----------------|----------------------|
| PID | Process IDs | Container processes see only their own process tree, starting at PID 1 |
| Network | Network stack | Container gets its own network interfaces, IP addresses, routing table |
| Mount | Filesystem mount points | Container sees its own filesystem hierarchy (not the host's) |
| UTS | Hostname and domain name | Container can have its own hostname |
| IPC | Inter-process communication | Container cannot access host IPC resources (shared memory, semaphores) |
| User | User and group IDs | Container can run as root internally without having root on the host |
| Cgroup | cgroup root | Container sees its own cgroup hierarchy for resource accounting |

When a container starts, the container runtime creates new namespaces for each of these (or inherits them from the parent, depending on configuration). The process inside the container sees only the isolated view — it cannot see processes, network interfaces, or filesystems outside its namespace.

---

## 3.5 How Containers Work: cgroups

Namespaces provide isolation — the process sees only its own view. But namespaces do not *limit* resource usage. A container could use 100% of CPU, fill all available RAM, or saturate disk I/O, starving other containers on the same host.

**cgroups (control groups)** enforce resource limits. Each container is assigned to a cgroup with limits on:

- **CPU:** maximum CPU time per period (e.g., 0.5 cores, or 50% of one core)
- **Memory:** maximum RAM (soft limit triggers reclaim, hard limit triggers OOM kill)
- **Block I/O:** read/write bandwidth and IOPS limits
- **Network:** traffic prioritisation and bandwidth limits (via `tc`)

The kernel enforces these limits. If a container exceeds its memory limit, the kernel kills processes within that container (not other containers). If a container exceeds its CPU limit, the kernel throttles its scheduling. The neighbouring container is not affected.

This is the physical enforcement mechanism behind containers. Namespaces provide the illusion of a dedicated machine. cgroups ensure no container can starve its neighbours.

---

## 3.6 Docker's Abstraction Chain

Docker packages the namespace + cgroup machinery into three concepts:

| Concept | What It Is | Analogy |
|---------|-----------|---------|
| **Dockerfile** | Recipe for building an image | Blueprint |
| **Image** | Built, immutable snapshot | Compiled binary |
| **Container** | Running instance of an image | Running process |

A Dockerfile specifies the recipe. You build it with `docker build -t my-image .`. The build process executes each instruction, creates a new layer for each step, and produces an image. The image is immutable — any change requires a new build.

You run the image with `docker run my-image`. Docker creates namespaces for the container, applies cgroup limits, sets up the network, mounts the layered filesystem, and starts the container process.

---

## 3.7 What Containers Made Possible

Before containers, deploying software meant:
1. Writing a README with manual installation steps
2. Having a production engineer follow those steps by hand
3. Debugging the inevitable differences between dev and prod
4. Repeating for every environment (staging, prod, DR)

After containers, deploying software means:
1. Writing a Dockerfile
2. `docker build && docker push`
3. `docker pull && docker run` on any target machine

This eliminated an entire class of deployment errors (the environment gap). It reduced deployment from a manual process (error-prone, slow, inconsistent) to an automated one (deterministic, fast, auditable).

Containers also made possible the orchestration layer that follows in Chapter 13 (Kubernetes). Without a standard, lightweight, portable unit of deployment (the container), orchestrating thousands of services across hundreds of machines would be impractical — each service would require different setup, different dependencies, different runtime configurations. Containers standardised the unit, enabling orchestration to focus on placement, scaling, and networking rather than dependency management.

---

## 3.8 Derivation Chain

```
Multiple programs on one machine → OS isolation (Chapter 2)
  → But different machines have different environments
    → "Works on my machine" problem
      → Solution: package everything the app needs into a self-contained unit
        → Static binaries (limited to compiled languages)
        → Virtual machines (too heavy)
        → Chroot/jails (too limited)
        → CONTAINERS: lightweight, fast, standardised
          → Namespaces for isolation
          → cgroups for resource limits
          → Layered images for distribution
          → Dockerfile for reproducibility
```

---

## 3.9 Design-Space Beat: When Containers Are Wrong

Containers are not the universal answer. They are the right answer for the specific constraints of Linux server workloads. The tradeoffs accepted:

- **Not for multi-tenant security isolation.** A kernel exploit in a shared container environment compromises all tenants. AWS Lambda uses microVMs (Firecracker) for multi-tenant isolation, not containers. Google Cloud Run uses gVisor (a user-space kernel) for similar reasons.

- **Not for heterogeneous OS environments.** A Linux container cannot run on Windows without a VM. A container built for ARM cannot run on x86 without emulation.

- **Not for high-security workloads requiring hardware isolation.** Workloads handling classified data or crossing organisational trust boundaries often require full hardware virtualization, not container-level isolation.

- **Not for real-time or determinism-critical workloads.** The shared kernel introduces latency variance (scheduling jitter, interrupt handling) that hard real-time systems cannot tolerate.

The strength of containers is that they work for *most* cloud-native workloads — enough that teams can standardise on one deployment format. The remaining 10% of workloads use VMs, microVMs, or bare metal.

---

## Summary

| Problem | Solution |
|---------|----------|
| Environment gap between dev and prod | Containers package everything the app needs |
| Dependencies conflict on shared machine | Namespaces isolate each container's view |
| One container starves others | cgroups enforce resource limits |
| Distributing software across machines | Layered images + registry (Docker Hub) |
| Reproducibility | Dockerfile as declarative recipe |

**Next:** We have packaged software into containers. Now we face the problem that drives the rest of the course: one machine can't run everything. We need multiple machines. And multiple machines cannot share memory.
