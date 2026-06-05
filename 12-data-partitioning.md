# Chapter 12: Data Partitioning

Replication (Chapter 10) spreads data across nodes for reliability. Partitioning spreads data across nodes for *scale* — no single node can hold all the data, so the data must be split across machines.

This chapter examines how to split data efficiently and what happens when the cluster changes size.

---

## 12.1 The Partitioning Problem

You have 10 TB of data. Each node can hold 2 TB. You need 5 nodes.

But the data cannot be split arbitrarily — related pieces of data must be on the same node (or at least queryable together). A user's profile should be on the same node as their orders. A product catalogue should be findable without querying every node.

The partitioning problem: given a set of data items and a set of nodes, assign each item to a node such that:
- Load is balanced (no node has significantly more data or requests than others)
- Related data stays together (minimise cross-node queries)
- Adding or removing nodes changes the assignment minimally (rebalancing is cheap)

---

## 12.2 Naive Hashing

The simplest approach: hash the key and mod by the number of nodes.

```
node = hash(key) % N
```

**Problem:** When N changes (adding or removing a node), `hash(key) % N` changes for most keys. Adding one node to a 10-node cluster causes approximately 90% of keys to map to a different node. This means 90% of data must be moved to new locations.

For a 10 TB cluster, adding one node requires moving ~9 TB of data. This is expensive, slow, and causes significant load on the cluster during rebalancing.

---

## 12.3 Consistent Hashing

Consistent hashing solves the rebalancing problem. The key insight: instead of mapping keys to nodes directly, map both keys and nodes to positions on a ring (typically a 64-bit hash space).

**How it works:**

1. Each node is assigned one or more positions on the hash ring (by hashing the node's identifier).
2. Each key is hashed to a position on the ring.
3. A key is assigned to the *nearest node in clockwise direction* from its ring position.

**When a node is added:** It claims responsibility for keys in the arc between its position and the previous node's position. Only those keys must be moved — roughly `1/N` of the keys, where N is the number of nodes. For a 10-node cluster, adding a node moves ~10% of data (vs ~90% with naive hashing).

**When a node is removed:** Its keys are reassigned to the next node clockwise. Again, only `1/N` of keys are affected.

**Virtual nodes:** Each physical node may own multiple positions on the ring (virtual nodes). This spreads load more evenly — if one physical node leaves, its keys are spread across many other nodes (not just the single clockwise neighbour), and new nodes receive roughly equal shares of keys. Virtual nodes also handle heterogeneous capacity: a node with twice the capacity gets twice as many virtual nodes.

---

## 12.4 Range Partitioning

Instead of hashing, partition keys into contiguous ranges. Key "A" through "M" go to Node 1. Key "N" through "Z" go to Node 2.

**Strengths:** Efficient range queries (scanning all keys from "2024-01-01" to "2024-01-31" hits at most two partitions instead of all of them). Easy to reason about. Manual control over placement.

**Weaknesses:** Range skew can cause hot spots (if most writes are in "2024-01" range, that node is overloaded. The "2024-03" node sits idle). Rebalancing requires splitting ranges, which is more complex than consistent hashing.

**Used in:** Bigtable, HBase, MongoDB (default).

---

## 12.5 Directory-Based Partitioning

Maintain a separate lookup service that maps key ranges to nodes. A client queries the directory to find which node has the data, then queries that node directly.

**Strengths:** Complete flexibility. Can move data between nodes arbitrarily (just update the directory). Can rebalance without affecting any node's addressing scheme.

**Weaknesses:** The directory is a single point of failure (or requires its own distributed coordination). Every request incurs additional latency for the lookup. The directory must be updated atomically when data is moved.

**Used in:** Netflix Eureka (service discovery), some DynamoDB configurations (metadata service). Generally avoided for data partitioning due to the directory being both a bottleneck and a coordination challenge.

---

## 12.6 Rebalancing Strategies

When a node is added or removed, data must be moved. Three strategies:

**Fixed number of partitions:** Create many more partitions than nodes (e.g., 1,000 partitions for 10 nodes → 100 partitions per node). Each partition is a unit of data owned by one node. When a node joins, it takes ownership of some partitions from existing nodes. When a node leaves, its partitions are reassigned. Only partitions (not keys) are moved — and partitions are much fewer than keys.

**Dynamic partitioning:** Split partitions when they grow too large (range-based splitting). Merge partitions when they grow too small. Used by Bigtable/HBase. Good for range-partitioned data where load varies over time.

**Consistent hashing with virtual nodes:** When a node joins, it picks up the arcs between existing nodes (consistent hashing). Virtual nodes ensure the load is spread evenly. No explicit partition management needed.

---

## 12.7 Hotspot Mitigation

Even with perfect partitioning, some nodes may receive disproportionate load:

**Popular key (celebrity effect):** A single key (e.g., a trending user's profile) receives orders of magnitude more requests than the average. Solution: split the hot key into secondary random keys, distribute reads across them, and merge results.

**Write-heavy partition:** Time-series data written to the same range (e.g., "current hour") overloads one partition while others are idle. Solution: write to a different partition from the start (e.g., hash-based time buckets), or use a write buffer that distributes across nodes before committing.

**Load shedding:** When a node is overloaded, reject a percentage of requests with "retry later" responses (HTTP 429 / 503). The clients back off and retry, giving the node time to recover. Better than accepting all requests and degrading for everyone.

**Hedged requests:** Send the same read request to multiple replicas (with different partition assignments) and use the first response. Reduces tail latency at the cost of increased load.

---

## 12.8 Design-Space Beat

| Strategy | Strengths | Weaknesses | Examples |
|----------|-----------|------------|----------|
| Consistent hashing | Minimal rebalancing, good spread, handles heterogeneity (virtual nodes) | Range queries hit many nodes | Cassandra, DynamoDB, Riak |
| Range partitioning | Efficient range queries, predictable placement | Hot spots, complex rebalancing | Bigtable, HBase, MongoDB |
| Fixed partitions | Simple, predictable | Partitions may grow unevenly | Kafka (topic partitions) |
| Directory-based | Maximum flexibility | Directory is bottleneck/SPOF | Netflix Eureka |

The choice depends on your query pattern:

- **Choose consistent hashing** when your primary access pattern is point lookups by key (read/write by user ID, order ID, session token). It minimises rebalancing and handles heterogeneous capacity well via virtual nodes.
- **Choose range partitioning** when you frequently scan contiguous key ranges (time-series queries, paginated search results, reporting over date ranges). The efficient range scans justify the hot-spot risk and rebalancing cost.
- **Choose directory-based partitioning** when you need maximum placement flexibility — moving data between nodes arbitrarily — and can tolerate an additional lookup hop and the directory becoming a coordination challenge.
- **Avoid naive hashing** in any system that will grow or shrink. The 90% reshuffle cost on resize makes it suitable only for fixed-size clusters.

---

## Summary

| Concept | Key Idea |
|---------|----------|
| Why partition | Data exceeds one node's capacity — must split across machines |
| Naive hashing | hash(key) % N — simple, but 90% of data moves on resizing |
| Consistent hashing | Keys and nodes on a ring. Only 1/N moves on resizing. Virtual nodes for balance. |
| Range partitioning | Keys by sorted range. Efficient range scans, prone to hot spots. |
| Rebalancing | Fixed partitions, dynamic splits, or consistent hashing |
| Hotspot mitigation | Popular key splitting, write buffering, load shedding, hedged requests |

**Next:** We have replication, transactions, and partitioning — the building blocks of distributed data systems. Now we examine how these concepts are packaged into a practical orchestration system: Kubernetes. Chapter 13 — Orchestration.
