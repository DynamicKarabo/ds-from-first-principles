# Chapter 13: Orchestration (Kubernetes)

The preceding chapters built up the concepts: containers (L3), distribution (L4), failure detection (L6), consensus (L7), replication (L10), transactions (L11), and partitioning (L12). This chapter shows how these concepts are integrated into a single system — Kubernetes — that manages containers across a cluster.

**Explicit caveat:** Kubernetes is one contingent design point, not a logical necessity. The components described here could have been arranged differently, and several alternatives (Mesos, Nomad, Swarm) took different approaches. Mapping each K8s component to a distributed systems concept is a pedagogical exercise — it shows what problems Kubernetes solves, not that it had to solve them this way.

---

## 13.1 The Orchestration Problem Restated

You have N machines, each running containers. Machines can fail (Axiom 2), network can partition (Axiom 2), latency is unpredictable (Axiom 1), and clocks are unsynchronised (Axiom 3). You have M services, each requiring a specific number of replicas, specific resources, and specific network configuration.

The orchestration problem: **ensure the actual state of the cluster continuously matches the desired state, despite continuous failures.**

This is a control loop. Desired state (declared by the operator) → compare with actual state (observed from the cluster) → take corrective action → repeat.

---

## 13.2 Component Mapping

Each Kubernetes component solves a specific distributed systems problem derived from the preceding chapters:

### etcd — Consensus and the Log (L7, L8)

etcd is a distributed key-value store using Raft consensus. It stores the cluster's desired state — the truth about what should be running, where it should be placed, and what configuration it needs.

In distributed systems terms: etcd is the **agreed log** (L8) that every component reads from and writes to. Without etcd, different parts of the cluster would have divergent views of desired state, leading to inconsistency (split-brain, conflicting scheduling decisions).

etcd configuration follows quorum arithmetic (L7): 3 nodes for most deployments (tolerates 1 failure), 5 nodes for production across availability zones (tolerates 2 failures).

### API Server — Single Authority (L4)

The API server is the single front door to the cluster. All communication goes through it — `kubectl`, `kubelet`, the scheduler, the controller manager, and external integrations.

In distributed systems terms: the API server solves the **single authority problem** (L4 — who decides?). Without a single front door, multiple components could issue conflicting state changes. The API server is the gate — it authenticates, authorises, validates, and writes to etcd.

### Controller Manager — Reconciliation Loop (L10)

The controller manager runs a set of controllers, each responsible for a specific resource type (Deployment, Service, ConfigMap, etc.). Each controller runs the same loop:

```
for {
  desired = etcd.get(resource spec)
  actual = apiServer.get(current state)
  if actual != desired {
    apiServer.create/update/delete(actions to close the gap)
  }
  watch for changes or sleep → repeat
}
```

In distributed systems terms: this is the **reconciliation loop** (L10 — desired vs actual state). The controller manager continuously compares the state of the cluster against what etcd says it should be. It closes any gap it finds. This is how Kubernetes self-heals — if a pod crashes, the controller sees the gap (desired replicas = 3, actual = 2) and creates a new pod.

### Scheduler — Bin-Packing Under Uncertainty (L12)

The scheduler watches for unscheduled Pods (pods that have been created but not assigned to a node). For each unscheduled pod, it evaluates all nodes against the pod's requirements and selects the best fit.

In distributed systems terms: the scheduler solves the **bin-packing problem under uncertainty** (L12) — it must place workloads onto machines while accounting for:
- Resource requests (minimum guaranteed) and limits (maximum burst)
- Node constraints (taints, tolerations, affinities, anti-affinities)
- Current node utilisation (which is a snapshot, not a guarantee of future load)
- Spreading workloads across nodes for fault tolerance (Axiom 2)

The scheduler does not know future resource usage. It uses requests/limits as prior beliefs about workload behavior — updated by monitoring data after placement.

### kubelet — Node Agent and Failure Detection (L6)

The kubelet runs on every node. It watches the API server for Pods assigned to its node. When a new assignment appears, the kubelet:
1. Pulls the container image
2. Creates the container via the container runtime (L3 — namespaces, cgroups)
3. Reports status back to the API server
4. Runs health checks (liveness, readiness, startup probes)

In distributed systems terms: the kubelet is the **node agent and local failure detector** (L6). It extends the cluster's management to the individual node. It detects when containers fail (crash loop back-off), when nodes are unhealthy (node conditions), and when the node is overloaded (resource pressure). It reports these conditions to the control plane.

### Container Runtime — Container Execution (L3)

The container runtime (containerd, CRI-O) is the component that actually creates namespaces, applies cgroups, and runs container processes. It implements the CRI (Container Runtime Interface), which allows different runtimes to be swapped without changing Kubernetes.

In distributed systems terms: the container runtime is the **execution layer** (L3) that makes the container abstraction physically real. It is not part of Kubernetes's distributed systems logic — it is the mechanism through which Kubernetes manifests its decisions.

### kube-proxy — Service Discovery and Routing (L10)

kube-proxy runs on every node and maintains network rules that route traffic to the correct Pods. It watches the API server for Service and Endpoint changes, then updates iptables (or IPVS, or eBPF) rules to direct traffic to the right container.

In distributed systems terms: kube-proxy implements **service discovery and load balancing** (L10) — how clients find the right replica. It ensures that traffic to a Service is distributed across healthy Pods, even as Pods are created, destroyed, or moved between nodes.

---

## 13.3 [DEEP DIVE] Why Kubernetes Won

Kubernetes was not the first container orchestrator. It was not the simplest. It was not the most feature-rich at launch. It won through a combination of factors, each worth examining because they represent genuine engineering strategy decisions.

### The Borg Inheritance

Kubernetes was built by Google engineers who had spent a decade operating **Borg** — Google's internal cluster management system, described in a 2015 paper. Borg managed billions of containers across Google's global infrastructure. It had been battle-tested at a scale no other orchestrator had approached.

Kubernetes inherited Borg's key architectural decisions:
- **Pods** (groups of containers that share a network namespace and storage) — Borg's "alloc" and "task" abstraction.
- **Controllers** (reconciliation loops that maintain desired state) — Borg's "Borgmaster" control loop.
- **Declarative state** (the user declares what they want, not how to achieve it) — Borg's "Borgcfg" configuration language.
- **Labels and selectors** (loose coupling via metadata) — Borg's "attributes."

The Borg paper validated Kubernetes's architecture. It was not a new design being tested — it was a proven design being productised.

### Declarative vs Imperative

Kubernetes's most consequential design decision: **the user declares desired state, and the system makes it true.** This is the declarative model.

```
Imperative: "Run container X on node Y with arguments A, B, C."
Declarative: "I want 3 replicas of container X, each with 0.5 CPU and 256MB RAM."
```

The declarative model is more powerful because it enables the reconciliation loop. The system continuously compares desired state against actual state and corrects drift. The operator describes *what* the system should look like; the system handles *how*.

This is a direct inheritance from Borg's control loop — and a direct application of the state machine replication principle (L10) at the orchestration level.

### Extensibility via CRDs and Controllers

Kubernetes's second-consequential design decision: **everything is an API resource, and anyone can define new ones.** Custom Resource Definitions (CRDs) allow operators to define new resource types (e.g., "Certificate," "Database," "ServiceMonitor"). Controllers then watch these resources and take action.

This created an ecosystem effect. Instead of Kubernetes defining all possible resource types, the community built controllers for everything: cert-manager (TLS certificates), Prometheus Operator (monitoring), Crossplane (cloud infrastructure), Istio (service mesh), ArgoCD (GitOps), Knative (serverless).

Each controller is a reconciliation loop for its own resource type. The Kubernetes API server becomes a platform that runs distributed control loops — not just a container orchestrator.

### The Network Effect

Kubernetes won the same way Docker won (L3 — registry network effect): the more tools integrated with Kubernetes, the more valuable Kubernetes became. By 2018, every cloud provider had a managed Kubernetes service. Every monitoring tool had a Kubernetes integration. Every CI/CD pipeline supported Helm charts.

Mesos had better resource isolation (a Linux kernel feature that Kubernetes lacked until 2023). Nomad was simpler to set up (single binary, no etcd requirement). Docker Swarm was natively integrated with Docker. But none had the ecosystem. Kubernetes had the API that every tool integrated with.

### The Tradeoff Kubernetes Accepted

Kubernetes's tradeoff: **complexity for generality.** Kubernetes can orchestrate almost any workload, but it requires significant operational expertise to run well. The learning curve is steep. The control plane components, networking setup, security configuration, and ongoing maintenance are non-trivial.

For the tradeoff to be worth it, the workload must benefit from the features that make Kubernetes complex:
- **Multiple services communicating across nodes** (service discovery, load balancing)
- **Automatic scaling and self-healing** (controller manager, HCAs)
- **Zero-downtime deployments** (rolling updates, blue-green, canary)
- **Multi-environment consistency** (same manifests in dev, staging, prod)

A single-container workload on a single node gains nothing from Kubernetes and pays the full complexity cost.

---

## 13.4 Derivation Caveat

Kubernetes is presented here as *one* answer to the orchestration problem. The chain maps each component to a distributed systems concept, and the mapping is pedagogically useful — it shows that Kubernetes is not arbitrary magic but a systematic assembly of known solutions.

But the mapping is also illustrative, not deterministic. Other orchestrators made different tradeoffs:
- **Nomad** omits etcd entirely (uses Raft embedded in the server binary — simpler, but no etcd's watch API for controllers).
- **Mesos** separates resource offer from task execution (a two-level scheduler that Kubernetes explicitly rejected for its simplicity-informed model).
- **Swarm** tightly integrated with Docker's API (simpler for Docker-native workflows, less extensible).

Each was a reasonable design point. Kubernetes's design won for the reasons above — but it is not "the correct orchestrator." It is the one that won under particular conditions (Google's experience, ecosystem strategy, the right moment in the container adoption curve).

---

## Summary

| Component | DS Concept | Problem Solved |
|-----------|-----------|---------------|
| etcd | Consensus (L7), The Log (L8) | Agreement on desired state |
| API server | Single authority (L4) | One front door for state changes |
| Controller manager | Reconciliation (L10) | Continuous desired-vs-actual correction |
| Scheduler | Bin-packing (L12) | Workload placement under uncertainty |
| kubelet | Failure detection (L6) | Node-level agent and health monitor |
| Container runtime | Containers (L3) | Namespaces, cgroups, execution |
| kube-proxy | Service discovery (L10) | Traffic routing to correct replicas |

**Winning factors:** Borg heritage, declarative model, CRD extensibility, ecosystem effects.

**Next:** Distributed systems produce distributed data. The log (L8) gives us a way to manage this data. But what if the log *is* the product? Chapter 14 — Streaming and The Log.
