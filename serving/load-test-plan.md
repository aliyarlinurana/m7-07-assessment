# Load Test Plan — NorthStar Inference Service

## Objectives

Validate that the inference service meets the SLOs defined in `serving/slos.yaml` under
realistic traffic conditions before every production promotion. The load test is run
automatically by the CI/CD pipeline (`cicd/.github/workflows/deploy-model.yml`) against
the staging environment after the green deployment is provisioned.

---

## Tool

**k6** (Grafana k6 v0.50+)

Chosen because:
- Scripted in JavaScript; scenarios and checks are version-controlled alongside the service.
- Native Prometheus remote-write output — results flow into the same Prometheus stack used for production monitoring.
- Supports staged ramp-up (warmup → steady → spike → teardown) in a single script.
- k6 Cloud not required; self-hosted OSS version runs in CI on a `c6i.2xlarge` load generator.

---

## Target Environment

| Parameter | Value |
|---|---|
| Environment | Staging (`api-staging.northstar.internal/v1`) |
| Inference replicas | 8 (mirrors production; see `serving/capacity-plan.md`) |
| Redis | Staging cluster (same topology as production, seeded with anonymised embeddings) |
| Model version | Green candidate being promoted |
| Load generator | 1 × c6i.2xlarge (same AZ as staging) |

---

## Test Scenarios

### Scenario 1 — Baseline Ramp (Functional)

Verifies correctness at low load before stress testing.

| Phase | Duration | Virtual Users | RPS (approx.) |
|---|---|---|---|
| Warmup | 2 min | 0 → 50 | 0 → 100 |
| Steady | 5 min | 50 | ~100 |
| Teardown | 1 min | 50 → 0 | 100 → 0 |

**Pass criteria:**
- p95 latency < 120 ms
- Error rate == 0%
- All response bodies match schema (k6 `check()`)
- `X-Model-Version` header present on every response

---

### Scenario 2 — Peak Load (Stress)

Validates the system at the 800 RPS target defined in the capacity plan.

| Phase | Duration | Virtual Users | RPS (approx.) |
|---|---|---|---|
| Warmup | 3 min | 0 → 200 | 0 → 400 |
| Steady (peak) | 15 min | 400 | ~800 |
| Teardown | 2 min | 400 → 0 | 800 → 0 |

**Pass criteria (must all pass for CI green):**

| Metric | Threshold | SLO reference |
|---|---|---|
| p95 end-to-end latency | < 120 ms | `slo_latency_p95` |
| p99 end-to-end latency | < 200 ms | `slo_latency_p99` |
| HTTP error rate (5xx) | < 0.5% | `slo_availability` canary gate |
| Cold-start fallback rate | < 10% | `slo_cold_start_rate` |
| Successful responses with `X-Model-Version` | 100% | API contract (openapi.yaml) |

---

### Scenario 3 — Spike Test

Simulates a sudden traffic burst (flash sale, push notification) above the sustained peak.

| Phase | Duration | Virtual Users | RPS (approx.) |
|---|---|---|---|
| Baseline | 3 min | 200 | ~400 |
| Spike | 2 min | 600 | ~1,200 |
| Recovery | 5 min | 200 | ~400 |
| Teardown | 1 min | 0 | 0 |

**Pass criteria:**
- p95 latency returns to < 120 ms within 60 s of spike end (recovery check).
- Error rate during spike < 2% (brief degradation acceptable; service must not fall over).
- No pod OOM kills or crash-loops during spike.

This scenario validates HPA scale-out behaviour. The test records how long it takes for the
autoscaler to add replicas (expected: ~90 s). If the spike is shorter than scale-out lag, the
64 ms headroom in the latency budget must absorb the overload.

---

### Scenario 4 — Cold-Start Traffic Mix

Simulates 20% of requests from cold-start users (new user IDs not present in Redis).

| Phase | Duration | Virtual Users | RPS | Cold-start % |
|---|---|---|---|---|
| Steady | 10 min | 400 | ~800 | 20% |

**Pass criteria:**
- Cold-start responses have `cold_start: true` in body.
- Cold-start p95 latency < 120 ms (popularity list is also pre-materialized in Redis).
- No errors on cold-start path.

---

### Scenario 5 — Redis Failover (Chaos)

Validates the circuit-breaker and cold-start fallback during a Redis primary failure.
**Run manually only** (not in CI); requires coordination with SRE.

**Procedure:**
1. Run steady load at 800 RPS.
2. Kill the Redis primary shard (simulate with `redis-cli DEBUG SLEEP 60` or stop the pod).
3. Observe: circuit-breaker opens within 5 s; cold-start fallback activates.
4. Verify: cold-start rate spikes to ~100%; p95 latency stays < 120 ms (fallback is fast).
5. Allow Sentinel to promote replica (~30 s).
6. Verify: cold-start rate returns to baseline within 2 minutes of Redis recovery.

**Pass criteria:**
- No 500 errors during failover window.
- Cold-start fallback activates within 5 s of Redis becoming unreachable.
- Service recovers automatically within 2 minutes of Redis restoration.

---

## k6 Script Structure

```
load-tests/
  k6-scenarios.js          # Main script: exports all scenarios
  helpers/
    auth.js                # Generates valid user_id values
    checks.js              # Shared response validation checks
    thresholds.js          # Shared pass/fail thresholds (matches slos.yaml)
  data/
    user_ids_warm.csv      # 50,000 user IDs known to have Redis embeddings
    user_ids_cold.csv      # 5,000 user IDs not in Redis (cold-start)
```

Key k6 thresholds (defined in `thresholds.js`, must match `slos.yaml`):

```javascript
export const thresholds = {
  http_req_duration: [
    'p(95)<120',   // slo_latency_p95
    'p(99)<200',   // slo_latency_p99
  ],
  http_req_failed: ['rate<0.005'],  // slo_availability canary gate (0.5%)
  checks: ['rate>0.999'],           // schema + header checks
};
```

---

## CI Integration

The load test runs automatically in the CI/CD pipeline at the **STAGING → CANARY** gate
(see `cicd/.github/workflows/deploy-model.yml`, step `load-test`):

1. Green pods reach `Running` state on staging.
2. k6 runs Scenario 1 (baseline) and Scenario 2 (peak) in sequence.
3. k6 exits with code 0 (all thresholds pass) → pipeline proceeds to canary.
4. k6 exits with code 1 (any threshold fails) → pipeline fails; green deploy is torn down;
   PagerDuty alert fires at `warning` severity.

Scenario 3 (spike) and Scenario 4 (cold-start mix) run as **advisory** jobs (non-blocking in CI)
but results are posted to `#ml-deployments` Slack channel for review.

---

## Reporting

k6 writes results to:
- **Prometheus remote-write** → grafana dashboard `NorthStar Load Test Results`.
- **k6 JSON summary** → archived as CI artefact for 90 days.
- **Slack** (`#ml-deployments`) → pass/fail summary posted by CI pipeline.

Grafana dashboard panels:
- p95 and p99 latency over test duration.
- RPS achieved vs. target.
- Error rate by HTTP status code.
- Cold-start rate over time.
- Redis connection pool saturation.
- HPA replica count (Scenario 3 only).
