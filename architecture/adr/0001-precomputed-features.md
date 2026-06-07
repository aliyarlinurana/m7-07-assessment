# ADR-0001: Pre-Compute User Embeddings into Redis Rather Than Computing at Request Time

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-07 |
| **Deciders** | ML Platform Lead, Backend Lead, Product Engineering Lead |
| **Ticket** | MLOPS-114 |

---

## Context

The recommendation system must return ranked results within a **120 ms p95 end-to-end budget** at **800 RPS peak**. Personalisation requires a user embedding — a 128-dimensional float32 vector derived from the user's last 30 days of browsing and purchase history.

There are two fundamental approaches to making this vector available at request time:

1. **Pre-compute and cache**: run a batch pipeline every 1–6 hours that materialises every active user's embedding into an online store (Redis), so the inference service reads a pre-built vector at request time.
2. **Real-time computation**: compute the embedding on-the-fly from raw event history at every request, using either a feature server or a direct database call.

The choice is the single most consequential architectural decision for this system because it determines whether the 120 ms budget is achievable at all.

---

## Decision

**We will pre-compute user embeddings and candidate sets in batch and store them in Redis.**

The inference service performs only a lightweight Redis `GET` (user vector + candidate set) followed by re-ranking inference over ~200 candidates. It never touches the data warehouse or a streaming feature store at request time.

---

## Rationale

### Latency arithmetic

The 120 ms budget must cover: network ingress (~5 ms) + A/B flag lookup (~1 ms) + feature retrieval + model inference + network egress (~5 ms). That leaves roughly **109 ms** for feature retrieval and inference combined.

| Approach | Feature retrieval latency | Inference latency | Total (p95 estimate) |
|---|---|---|---|
| **Pre-compute → Redis GET** | 2–5 ms (p99, payload < 10 KB) | 20–40 ms (re-rank 200 candidates, CPU) | **27–50 ms** ✅ |
| Real-time from DW (Snowflake) | 200–600 ms | 20–40 ms | **220–640 ms** ❌ |
| Real-time from streaming store (Flink) | 15–50 ms | 20–40 ms | **35–90 ms** ✅ (marginal) |
| Real-time embedding model call | 80–150 ms | 20–40 ms | **100–190 ms** ❌ |

Redis is the only approach that gives comfortable headroom under the 120 ms ceiling with no additional infrastructure on the hot path.

### Why not the streaming feature store?

A Kafka + Flink pipeline could reduce feature staleness from ~1 hour to ~10–30 seconds and keep feature retrieval under 50 ms. However:

- Operational complexity increases significantly: Kafka cluster, Flink job management, exactly-once semantics, schema evolution, backpressure tuning.
- The product requirement specifies **last-30-day signals**. A 1-hour lag on 30-day aggregates is negligible — purchase intent shifts at the scale of days, not minutes.
- The engineering team does not currently have streaming infrastructure in production. Introducing it for this system adds a new operational domain with no other workloads sharing the cost.
- The streaming path is a **known migration path** if requirements change (e.g., real-time session signals, sub-minute freshness SLA). The batch-first architecture does not foreclose it.

### Why Redis specifically?

| Store | p99 GET latency | Memory model | Operational complexity |
|---|---|---|---|
| **Redis (in-memory)** | 2–5 ms | All-hot, no cold-read penalty | Medium (Cluster mode, Sentinel) |
| DynamoDB | 5–15 ms | Hot/warm tiering | Low (managed) |
| Cassandra | 5–20 ms | Disk-backed, compaction overhead | High (self-managed) |
| PostgreSQL | 10–50 ms | Disk-backed | Low–medium |

Redis's fully in-memory model guarantees that every active user's vector is hot — there is no cache-miss penalty for recently active users, which is exactly the population we serve. TTL-based eviction handles churned users automatically.

---

## Consequences

### Positive
- Hot path is deterministic: gateway → Redis GET → re-rank inference → response. No branching on feature availability.
- Redis GET at p99 is 2–5 ms, leaving 115+ ms for all other work — comfortable margin.
- Batch pipeline failures are isolated from the serving path; stale-but-present vectors continue serving until the next successful run.
- Simple horizontal scaling: add Redis Cluster shards for capacity, add inference replicas for throughput.

### Negative / mitigations

| Risk | Mitigation |
|---|---|
| **Feature staleness up to ~1 hour** | Acceptable given 30-day signal window. Documented in SLO as a known characteristic, not a defect. |
| **Redis is a single operational dependency on the hot path** | Redis Cluster mode with 3 shards + replicas; Redis Sentinel for automatic failover. Circuit-breaker in inference service falls back to cold-start popularity list if Redis is unreachable. |
| **Memory cost: ~64 GB for 10 M active user embeddings** | Sharded across 4 × r6g.2xlarge nodes (32 GB each). Scales linearly with user base — reviewed at 15 M users. |
| **New users get no embedding until next batch run** | Cold-start service handles users with < 3 events in 30 days. See ADR-0002 for cold-start path. |
| **Batch pipeline failure means stale vectors** | Pipeline has retry logic (3 attempts, exponential backoff). PagerDuty alert fires if pipeline has not completed within 2× its expected cadence. Inference service continues on last-known-good vectors. |

---

## Alternatives Considered and Rejected

### A — Real-time embedding computation on hot path
Compute the user embedding from raw event history at every request using an embedding model call or feature server.

**Rejected because:** Even a fast embedding model call adds 80–150 ms, which alone blows the 120 ms budget before any re-ranking inference occurs. A feature server reading from the DW adds 200–600 ms.

### B — Kafka + Flink streaming feature pipeline
Materialise embeddings continuously as events arrive, reducing staleness to seconds.

**Rejected because:** Streaming infrastructure adds significant operational complexity for a benefit (sub-minute freshness on 30-day aggregates) that is not required by the product. Documented as a future migration path in `JUSTIFICATION.md §1`.

### C — DynamoDB as online feature store instead of Redis
Use AWS DynamoDB (managed, serverless) instead of Redis for the online feature store.

**Rejected because:** DynamoDB p99 GET latency is 5–15 ms — workable but ~3× slower than Redis for this payload size. More importantly, DynamoDB's throughput costs at 800 RPS × 2 reads/request (user vector + candidate set) would be ~$1,800/month in read capacity units, compared to ~$800/month for Redis Cluster. Redis also supports native vector similarity operations if we later co-locate ANN retrieval with the feature store.

---

## Review Checkpoint

This decision should be revisited if any of the following conditions are met:

- Active user base exceeds **20 M** (Redis memory cost crosses $3,000/month; evaluate DynamoDB or Aerospike).
- Product requires **sub-5-minute feature freshness** (triggers streaming pipeline evaluation).
- Re-ranker model grows to require GPU inference (changes the latency arithmetic; streaming store margin shrinks).
