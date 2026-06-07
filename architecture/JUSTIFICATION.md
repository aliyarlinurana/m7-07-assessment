# JUSTIFICATION.md — Scenario X: Personalized In-App Recommendations

## 1. Pattern Choice: Two-Tier Serving (Pre-Compute + Online Scoring)

### What we chose
A **pre-compute-heavy, two-tier architecture**: all user embeddings and candidate sets are materialized
into Redis before request time; the online inference service performs only lightweight re-ranking
at request time.

### Why not alternatives?

| Alternative | Why rejected |
|---|---|
| **Fully real-time feature computation** | Computing user embeddings on the fly from raw event history adds 200–400 ms — blows the 120 ms p95 budget before any model inference happens. |
| **Pure ANN retrieval (no re-ranker)** | ANN alone cannot apply session context (last 3 clicks) or business rules (margin, inventory). A re-ranking step is required for production relevance. |
| **Streaming feature pipeline (Kafka + Flink)** | Reduces staleness to seconds but adds significant operational complexity. Given 1-hour batch cadence is acceptable for the product requirement (last-30-day signals), the cost is not justified. |
| **Single-tier (batch only, no online model)** | Cannot react to within-session behaviour (cold-start routing, session context). Leaves relevance quality on the table. |

### Trade-offs accepted
- User vectors are up to ~1 hour stale. Acceptable: purchase intent signals over 30-day windows
  shift slowly enough that 1-hour lag does not materially hurt relevance.
- Redis cluster becomes a single operational dependency on the hot path. Mitigated by Redis Sentinel
  / Cluster mode with automatic failover and a circuit-breaker in the inference service.

---

## 2. Feature Store: Redis (Pre-Computed Embeddings)

### What we chose
Redis as the **online feature store**, populated by a batch pipeline every 1–6 hours.
Keys: `user:{user_id}:embedding` (float32 vector), `candidate:{segment}:set` (item ID list).

### Rationale
- Sub-millisecond `GET` latency keeps the feature-lookup step well within the 120 ms budget
  (~2–5 ms observed at p99 for payloads < 10 KB).
- Redis's in-memory model means no cold-read penalty; every active user's vector is hot.
- TTL-based eviction naturally handles churned users without manual cleanup.

### Trade-offs accepted
- Memory cost: ~50 bytes per float32 × 128 dims × 10 M active users ≈ **64 GB RAM** for user
  embeddings. Addressable with Redis Cluster sharding across 4 × 32 GB nodes.
- No sub-minute freshness. Accepted — see §1 above.

---

## 3. Inference Service: FastAPI + In-Process Scoring

### What we chose
A **FastAPI service** (3 replicas behind the load balancer) that loads the re-ranking model
in-process (ONNX runtime or TorchScript) rather than calling a separate model server.

### Rationale
- Eliminates one network hop compared to a sidecar or remote model server (TorchServe, Triton).
  Each saved hop is worth ~5–15 ms on the hot path.
- FastAPI's async request handling saturates CPU efficiently at 800 RPS with lightweight models
  (re-ranker over ~200 candidates is not GPU-bound).
- Simple horizontal scaling: add replicas behind the load balancer; no shared GPU pool to manage.

### Trade-offs accepted
- Model update requires a rolling pod restart (blue/green deploy), introducing ~30 s of mixed-version
  traffic. Managed by the A/B router's `model_version` flag — no user sees inconsistent state.
- If the re-ranker grows to a GPU-bound deep model, the service must be refactored to call a GPU
  inference backend. Flagged as a known future inflection point in ADR-0002.

---

## 4. Cold-Start Strategy: Popularity Fallback (Pre-Materialized)

### What we chose
Users with fewer than 3 browsing events in the last 30 days are routed to a **Cold-Start Service**
that returns a pre-materialized popularity list scoped to the user's onboarding-declared category.

### Rationale
- Zero additional latency: the popularity list is stored in Redis exactly like user embeddings.
  The fallback path respects the same 120 ms SLO.
- Category-scoped fallback is meaningfully better than a global bestseller list for conversion,
  and requires no personalisation signal beyond onboarding data.
- Implementation is trivial to test and operate; failure mode degrades gracefully to global list.

### Trade-offs accepted
- Users who declared an incorrect category during onboarding receive worse cold-start recommendations.
  Accepted until behavioural signals accumulate (typically 3–5 sessions).
- Popularity signal is updated in the same 1-hour batch — highly viral items may lag. Low-risk
  for cold-start users who are least sensitive to this.

---

## 5. A/B Testing: Separate Router Layer

### What we chose
A dedicated **A/B Router** (feature-flag service, e.g. LaunchDarkly) sits between the API Gateway
and the Inference Service, assigning `model_version` per `user_id` deterministically.

### Rationale
- Keeps experiment logic out of model code. Model engineers and growth engineers can change
  traffic splits without a model deploy.
- Deterministic assignment (hash-based on `user_id`) ensures the same user always sees the same
  variant within an experiment window — critical for statistical validity.
- `X-Model-Version` header flows through to Downstream Consumers, allowing the experiment platform
  to join impression logs against purchase events for per-variant lift computation without
  additional instrumentation.

### Trade-offs accepted
- Adds one additional service to the hot path. Latency impact is negligible (<1 ms) for a
  flag-service lookup (local cache with async refresh).
- LaunchDarkly (or equivalent) becomes a dependency. A flag-service outage defaults to the
  champion model — a safe fallback defined in the inference service's startup config.

---

## 6. Retraining Trigger: Dual (Schedule + Drift Alert)

### What we chose
Retraining is triggered by **either** a daily/weekly schedule **or** a drift alert from Evidently,
whichever fires first.

### Rationale
- Schedule alone risks missing rapid distribution shifts (e.g., a viral product category, a
  supply-chain event emptying inventory). Drift detection closes this gap.
- Drift detection alone risks missing slow, gradual decay that never crosses a single-event
  threshold. Scheduling provides a guaranteed freshness floor.
- Evidently monitors both **input feature distribution** (user embedding drift) and **prediction
  score distribution** (score mean/variance shift), giving two independent signal sources.

### Trade-offs accepted
- Dual triggers can queue concurrent training jobs. Managed by a job queue with deduplication
  (only one training run per model version at a time).
- Drift thresholds require calibration. Initial thresholds are set conservatively (PSI > 0.2
  for feature drift, KL divergence > 0.1 for prediction scores) and will be tuned after the
  first 30 days of production data.

---

## 7. Model Registry Gate: Evaluation + Human Sign-Off

### What we chose
Every trained model must pass **automated evaluation gates** (offline metrics vs. champion) and
a **human sign-off** for major version bumps before the registry promotes it to production.

### Rationale
- Automated gates catch regressions (NDCG drop, precision drop) without human latency.
- Human sign-off for major bumps (new model architecture, new feature schema) ensures that
  business-critical changes are reviewed — a recommendation model directly affects revenue.
- Minor version bumps (same architecture, retraining on fresh data) are fully automated to
  maintain a fast feedback loop.

### Trade-offs accepted
- Human sign-off adds latency to the promotion path for major versions (target: same business
  day). Acceptable given these are infrequent.
- "Major vs. minor" distinction must be codified in the registry schema to avoid ambiguity.
  Defined in `lifecycle/model-registry.yaml`.

---

## 8. Monitoring: Multi-Window Burn-Rate Alerts

### What we chose
SLO alerting uses **multi-window burn-rate** (1 h fast burn + 6 h slow burn) rather than
simple threshold alerts on raw error rate.

### Rationale
- Raw error-rate alerts produce too many false positives during normal traffic variance at 800 RPS.
- Burn-rate alerts fire only when the error budget is being consumed faster than sustainable,
  balancing sensitivity (catch real incidents quickly) with specificity (suppress noise).
- Two windows prevent both missed slow-burn degradations (caught by 6 h window) and delayed
  response to fast spikes (caught by 1 h window).

### Trade-offs accepted
- Burn-rate alerting requires accurate SLO baseline definition upfront. Initial values are
  set from load-test results and tightened after 2 weeks of production data.
- Alert complexity is higher than simple threshold rules; requires team training.

---

## Summary of Key Trade-offs

| Decision | What we gain | What we give up |
|---|---|---|
| Pre-compute features into Redis | Hot path under 120 ms | Up to 1 h feature staleness |
| In-process re-ranker (FastAPI) | One fewer network hop | Rolling restart on model update |
| Popularity fallback (pre-materialized) | SLO-safe cold-start path | Stale viral items for new users |
| Separate A/B router | Experiment/model decoupling | Additional hot-path service |
| Dual retraining trigger | Freshness floor + drift reactivity | Possible concurrent job queue contention |
| Human sign-off for major versions | Governance on revenue-critical changes | Slower major-version promotion |
| Multi-window burn-rate alerts | Low false-positive rate | Higher alerting complexity |

Full ADR detail in [`adr/0001-precomputed-features.md`](./adr/0001-precomputed-features.md)
and [`adr/0002-blue-green-deploy.md`](./adr/0002-blue-green-deploy.md).
