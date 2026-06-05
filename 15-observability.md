# Chapter 15: Observability

Distributed systems produce partial, conflicting evidence about their own health. A request succeeds on one node, fails on another. A latency spike has no single cause. A node that appears dead to one peer appears healthy to another.

Observability is the practice of reasoning about system state from these incomplete signals. It is not "monitoring" — monitoring detects that something is wrong. Observability identifies *what* is wrong and *where*, across a system where no single component has the full picture.

---

## 15.1 The Observability Problem

Three sources of uncertainty, each traceable to the axioms:

| Axiom | Observability Consequence |
|-------|-------------------------|
| A1 (Asynchrony) | You cannot know if a slow request was caused by a transient delay or a systemic issue — the same indistinguishability problem as failure detection |
| A2 (Partial failure) | Part of the system can be healthy while another part is failing. A 200 response from one service does not mean the user's request succeeded |
| A3 (No shared clock) | Correlating events across services requires accurate timestamps, which you cannot trust across machines |

These three together produce the observability problem: **you have partial, conflicting, and potentially misordered evidence about a system whose state you must understand in order to operate.**

---

## 15.2 The Three Pillars

Observability is built on three data types, each with different strengths:

### Metrics — Aggregated Numeric Data

Metrics are numeric values sampled over time: request count, error rate, latency percentiles, CPU utilisation, memory consumption.

**Strengths:** Cheap to collect and store. Support aggregation (sum, average, percentile) across services. Provide a high-level health signal (is error rate elevated? is latency increasing?).

**Weaknesses:** Metrics cannot identify individual failing requests. They cannot correlate causes across services. A high p99 latency tells you something is wrong but not *what* or *where*.

**Key frameworks:**
- **RED (Rate, Errors, Duration)** — The three metrics every service should expose: request rate, error count/rate, and request duration (latency). Focuses on the user-facing health of the service.
- **USE (Utilization, Saturation, Errors)** — The three metrics every *resource* should expose. For every resource (CPU, memory, disk, network): how full is it (utilisation), how much queueing (saturation), and how many errors.

### Logs — Event Records

Logs are structured or unstructured records of discrete events. A request arrives. A database query executes. A connection closes. Each is a log entry with a timestamp, severity, and structured context.

**Strengths:** Individual event detail. Ability to search for specific patterns ("find every request that returned 503 with no auth header"). Context-rich.

**Weaknesses:** High volume (potentially terabytes/day). Expensive to store and query at scale. Logs from different services must be correlated across timestamps — which requires Axiom 3's precision.

**Structural best practice:** Each log entry should be a structured event (JSON) with a schema, not a free-text string. `{"event": "order_created", "order_id": "12345", "user_id": "67890", "amount": 29.99}` is queryable. `"Order 12345 created for user 67890 for $29.99"` requires text search.

### Traces — Request Lifecycle

A trace follows a single request as it propagates across services. Each **span** represents one unit of work (a service call, a database query, a computation). Spans are nested and linked by a trace ID, which is propagated through the request's headers.

**Strengths:** Cross-service correlation. Identifies which service in a chain is causing latency or errors. The only signal that can answer "why was *this specific* request slow?"

**Weaknesses:** High implementation cost (every service must propagate trace context). Sampling is necessary for high-throughput systems (tracing 100% of requests is often impractical).

**Distributed tracing and Axiom 3:** Traces rely on accurate timestamps to determine the order of spans. Without Axiom 3's awareness, trace visualisation can show spans in the wrong order. Google's Dapper paper introduced the technique that most tracers (Jaeger, Zipkin, OpenTelemetry) implement.

---

## 15.3 Monitoring vs Debugging

Monitoring and debugging are different activities with different requirements:

| | Monitoring | Debugging |
|--|-----------|-----------|
| Goal | Detect that something is wrong | Find *what* and *where* |
| Signal | Metrics (aggregated) | Traces + logs (individual) |
| Alerting | Predefined thresholds | Ad-hoc investigation |
| Time scale | Seconds to minutes (realtime) | Minutes to hours (post-hoc) |
| Audience | Operations team (on-call) | Developer (incident investigation) |

**The observability triage pattern:**

1. **Metrics alert** → "Error rate on the checkout service is 5% (threshold: 1%)" → *Something is wrong*
2. **Traces** → "90% of failing requests show a timeout calling the payment service" → *Where the failure is*
3. **Logs** → "Payment service is showing 'connection pool exhausted' on the authorisation database" → *What the root cause is*

Metrics detect. Traces locate. Logs diagnose. This is the observability workflow in practice.

---

## 15.4 Backpressure and Flow Control

Observability reveals when a system is overloaded. Backpressure is the mechanism for propagating that information upstream so the system can protect itself.

When a service cannot keep up with incoming requests:
- It can buffer (queue requests) — increases latency, memory usage
- It can drop requests (load shedding) — increases error rate
- It can propagate backpressure upstream — the caller slows down or stops sending

Backpressure is a direct consequence of Axiom 1 (asynchrony) applied to capacity. An upstream service cannot know how loaded a downstream service is without being told. Backpressure is the feedback signal.

**Observation → backpressure → recovery loop:**
1. Metrics detect downstream service is overloaded (p99 latency > 1s)
2. Tracing identifies which upstream services are calling it
3. Upstream services slow their request rate (circuit breaker, adaptive throttling)
4. Downstream service recovers
5. Metrics confirm recovery
6. Upstream services resume normal request rate

---

## 15.5 Design-Space Beat

**When is each pillar the right primary signal?**

| Situation | Primary Signal | Why |
|-----------|---------------|-----|
| Is the system up? | Metrics (RED) | Count == 0 means the service is not serving |
| Is latency acceptable? | Metrics (latency percentiles) | p99 > 500ms means users are waiting |
| Why is *this* request failing? | Trace + Logs | Trace shows path, log shows error reason |
| Capacity planning | Metrics (USE) | Resource utilisation trends predict exhaustion |
| Debugging a known issue | Logs | Patterns in event records reveal the bug |
| Optimising a slow endpoint | Traces | Which span in the chain is the bottleneck? |

**The practical rule:** Start with RED metrics for every service (know if it's healthy). Add tracing for services in the critical request path. Add structured logging for services that handle complex business logic. Add more as needed.

**Under a fixed observability budget:** Metrics first (they're cheapest and cover detection). Tracing second (it requires explicit instrumentation but pays for itself during incidents). Structured logging third (most expensive to store and query at scale, so invest here only after metrics and traces are in place). This ordering maximises coverage per unit of infrastructure cost.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Observability problem | Distributed systems produce partial, conflicting, misordered evidence about their health |
| Metrics (RED/USE) | Aggregated numeric signals — cheap, alertable, good for detection |
| Logs | Structured event records — good for diagnosis, expensive at scale |
| Traces | Cross-service request lifecycle — the only signal that correlates across services |
| Monitoring vs debugging | Metrics detect, traces locate, logs diagnose |
| Backpressure | Flow control signal, propagated upstream — capacity protection |

**Next:** You can observe the system. But how do you *know* it will survive failures? Chapter 16 — Chaos Engineering and Formal Verification.
