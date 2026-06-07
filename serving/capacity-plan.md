# Capacity Plan — NorthStar Inference Service

## Target Numbers

| Parameter | Value | Source |
|---|---|---|
| Peak RPS | 800 | Product requirement |
| p95 latency budget (end-to-end) | 120 ms | SLO (slos.yaml) |
| p95 inference latency (in-process) | ≤ 40 ms | Benchmark; ADR-0001 latency table |
| p99 Redis GET latency | 2–5 ms | ADR-0001 |
| Model size (ONNX) | ~50 MB | container/README.md |
| Candidate set per request | ~200 items | ADR-0001 |
| Active users (current) | 10 M | Architecture assumption |
| Redis memory (user embeddings) | ~64 GB | 128-dim float32 × 10 M users |

---

## Latency Budget Breakdown

Every request must complete within **120 ms p95** end-to-end. The budget is allocated as:

| Step | Component | Budget (ms) |
|---|---|---|
| Network ingress | API Gateway / LB | 5 |
| A/B flag lookup | LaunchDarkly (local cache) | 1 |
| Redis GET (user vector + candidate set) | Feature Store | 5 |
| Re-ranking inference (200 candidates, ONNX) | Inference Service in-process | 40 |
| Response serialisation + network egress | API Gateway / LB | 5 |
| **Total p95 budget consumed** | | **56 ms** |
| **Headroom** | | **64 ms** |

The 64 ms of headroom absorbs tail-latency variance in Redis and model inference at p95 without breaching the SLO. If Redis p99 reaches 8 ms or inference p99 reaches 60 ms, the combined p95 is still within 120 ms.

---

## Replica Sizing

### Inference Service

**Assumptions:**
- Single request processing time (CPU): ~40 ms for re-ranking 200 candidates.
- ONNX Runtime with `OMP_NUM_THREADS=2`; each replica uses 2 CPU cores effectively.
- FastAPI + Uvicorn async workers: 4 workers × 2 threads = 8 concurrent request slots per replica.
- Target CPU utilisation at peak: ≤ 70% (leaves burst headroom).

**Throughput per replica:**
```
1 replica = 8 concurrent slots
Average request time = 40 ms inference + ~15 ms overhead = ~55 ms total
Max theoretical RPS per replica = 8 / 0.055 ≈ 145 RPS
At 70% utilisation target: 145 × 0.70 ≈ 100 RPS per replica
```

**Replicas required for 800 RPS:**
```
800 RPS / 100 RPS per replica = 8 replicas
```

**Recommendation: 8 replicas** in steady state (up from the initial 3-replica PoC). During blue/green deploy, 16 replicas run simultaneously for up to 4 hours.

**Instance type:** `c6i.2xlarge` (8 vCPU, 16 GB RAM)
- 2 vCPU allocated to OMP threads per worker process; 4 workers per pod.
- 16 GB RAM: ONNX model (~200 MB uncompressed) + Python runtime + request buffers.

### Horizontal Pod Autoscaler (HPA)

| Metric | Target | Min replicas | Max replicas |
|---|---|---|---|
| CPU utilisation | 60% | 4 | 16 |
| RPS per pod (custom metric) | 80 | 4 | 16 |

HPA scale-out lag: ~90 s (Kubernetes default). Pre-scale to 8 replicas during known traffic peaks (daily 18:00–22:00 UTC) via a scheduled CronJob.

---

## Redis Cluster Sizing

| Parameter | Value |
|---|---|
| User embedding size | 128 dims × 4 bytes = 512 bytes per user |
| Per-user Redis key overhead | ~50 bytes |
| Total per user | ~562 bytes |
| Active users | 10 M |
| User embedding total | ~5.6 GB |
| Candidate sets (per segment × 500 items × 8 bytes) | ~8 GB (est. 2,000 segments) |
| Popularity lists + cold-start keys | ~1 GB |
| Redis overhead (metadata, fragmentation ~30%) | ~4.4 GB |
| **Total Redis memory required** | **~19 GB** |

**Cluster configuration:** 3 shards × 2 nodes (primary + replica) = 6 nodes total.
**Instance type:** `r6g.2xlarge` (8 vCPU, 64 GB RAM) — 3 primaries, 3 replicas.
Each primary holds ~6.5 GB of data with 50% headroom for growth.

Review at 20 M active users (memory crosses ~38 GB; add 3 shards or migrate to Aerospike — see ADR-0001 review checkpoint).

---

## Cost Estimate (Monthly, AWS us-east-1)

| Component | Config | Unit cost | Monthly cost |
|---|---|---|---|
| Inference replicas (steady) | 8 × c6i.2xlarge, On-Demand | $0.34/hr | ~$1,958 |
| Inference replicas (blue/green peak, 4 h/deploy, ~4 deploys/month) | 8 extra × 4 h × 4 | $0.34/hr | ~$44 |
| Redis Cluster | 6 × r6g.2xlarge, On-Demand | $0.40/hr | ~$1,747 |
| API Gateway / Load Balancer | ALB, 800 RPS × 730 h | $0.008/LCU-hr (est.) | ~$150 |
| S3 (model artifacts, MLflow) | ~100 GB + requests | $0.023/GB | ~$10 |
| SageMaker training (daily) | ml.p3.2xlarge, ~1 h/day | $3.06/hr | ~$92 |
| Monitoring (Prometheus, Evidently) | 2 × t3.medium | $0.042/hr | ~$61 |
| **Total estimated monthly** | | | **~$4,062** |

**Savings opportunity:** Reserved Instances (1-year) on inference and Redis nodes reduces compute cost by ~40%, bringing the estimate to ~**$2,800/month**.

---

## Failure Mode Capacity

| Scenario | Impact | Mitigation |
|---|---|---|
| 1 inference replica lost | 800 / 7 = 114 RPS per replica (~114% load) | HPA adds replacement within 90 s; brief overload absorbed by 64 ms headroom |
| Redis primary shard failover | ~30 s Redis Sentinel promotion; circuit-breaker serves cold-start | Popularity fallback pre-materialized; SLO degraded but service continues |
| Blue/green deploy (8 + 8 replicas) | Double compute for ≤ 4 h | Budgeted in cost estimate above |
| Traffic spike to 1,200 RPS (+50%) | HPA scales to 12 replicas | 90 s lag; pre-scaling during peak hours prevents gap |

---

## Review Checkpoints

| Trigger | Action |
|---|---|
| Active users > 20 M | Re-evaluate Redis cluster (ADR-0001); review inference replica count |
| Peak RPS > 1,200 sustained | Increase HPA max replicas to 20; consider GPU inference |
| Model size > 200 MB (ONNX gate in model-registry.yaml) | Re-evaluate in-process loading vs. Triton sidecar |
| Monthly cost > $6,000 | Evaluate Spot Instances for inference + Reserved pricing for Redis |
