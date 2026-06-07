# ADR-0002: Blue/Green Rolling Deploy for Model Updates

| Field | Value |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-07 |
| **Deciders** | ML Platform Lead, SRE Lead, Product Engineering Lead |
| **Ticket** | MLOPS-118 |

---

## Context

The inference service runs 3 FastAPI replicas behind a load balancer. The re-ranking model is loaded **in-process** (ONNX runtime or TorchScript) rather than via a remote model server. This means a model update requires a pod restart — there is no hot-swap mechanism without a process boundary.

We need a deploy strategy that satisfies three constraints simultaneously:

1. **No hard downtime** — the service must continue serving at 800 RPS during a model update.
2. **Mixed-version window is bounded and observable** — during rollout, some replicas will run the old model and some the new one. This must be detectable and time-limited.
3. **Fast rollback** — if the new model degrades metrics, we must be able to revert within minutes, not hours.

There are three candidate strategies: recreate (hard cutover), rolling update, and blue/green (parallel environment switch).

---

## Decision

**We will use a blue/green deploy strategy with traffic controlled by the A/B router's `model_version` flag.**

- The "blue" environment is the current production deployment (champion model).
- The "green" environment is a new deployment spun up in parallel with the challenger model.
- The A/B router shifts traffic from blue to green incrementally: 5% → 25% → 100%, with automated metric gates at each step.
- On rollback, the router returns all traffic to blue instantly — no pod restart required.

---

## Rationale

### Strategy comparison

| Strategy | Downtime | Mixed-version window | Rollback time | Complexity |
|---|---|---|---|---|
| **Recreate** | ~30–60 s | None | Redeploy (~5 min) | Low |
| **Rolling update** | None | ~60–90 s (3 replicas) | Redeploy (~3 min) | Low |
| **Blue/green (chosen)** | None | Controlled by router (minutes to hours) | **< 5 s** (flag flip) | Medium |
| Canary (separate) | None | Indefinite until promoted | < 5 s | Medium–High |

Rolling update was the first candidate considered. It is simpler than blue/green but has one critical flaw for this system: during the ~90 s window where replicas run different model versions, there is no way to control which users see which version. This breaks A/B experiment validity — a user mid-session could receive recommendations from two different models, corrupting the impression log.

Blue/green solves this because traffic routing is controlled by the A/B router at the `user_id` level. The same user always hits the same model version within an experiment window, preserving statistical validity.

### Rollback is a flag flip, not a redeploy

The most operationally significant advantage of this approach is rollback speed. If the green deployment shows regression (NDCG drop > 2%, error rate spike, latency breach), the on-call engineer sets the router flag back to blue. Traffic shifts in under 5 seconds. The green pods remain running for post-mortem analysis.

Compare to rolling update rollback: a new rollout must be initiated, which takes 3–5 minutes and creates another mixed-version window.

### Mixed-version window is intentional and observable

During a blue/green promotion, both versions serve production traffic simultaneously. This is by design — it is the A/B experiment. The `X-Model-Version` response header identifies which version served each request. Prometheus tracks per-version latency, error rate, and prediction score distribution. The experiment platform joins impression logs on `model_version` to compute per-variant lift.

The mixed-version window is bounded by the promotion schedule defined in the registry:

| Step | Traffic to green | Gate condition | Max wait |
|---|---|---|---|
| Shadow | 0% (logs only) | No errors in 100 requests | 15 min |
| Canary | 5% | Error rate < 0.5%, p95 < 120 ms | 30 min |
| Ramp | 25% | NDCG ≥ champion − 1%, no SLO breach | 2 h |
| Full | 100% | Same as ramp | — |

Automated gates check these conditions. A failure at any step triggers an automatic rollback and a PagerDuty alert.

### In-process model loading justification

The choice to load the model in-process (rather than calling a remote model server like Triton or TorchServe) eliminates one network hop per request (~5–15 ms). At 800 RPS with a 120 ms budget, this margin matters. The trade-off — that model updates require a pod restart rather than a model swap — is directly addressed by the blue/green strategy. The two decisions are coupled: blue/green makes in-process loading operationally safe.

---

## Consequences

### Positive
- Zero downtime model updates.
- Rollback in < 5 seconds via router flag — no deployment pipeline involved.
- Mixed-version traffic is controlled, observable, and experiment-valid.
- Blue environment stays live during promotion, providing immediate fallback capacity.
- `X-Model-Version` header flows through to analytics for clean per-variant attribution.

### Negative / mitigations

| Risk | Mitigation |
|---|---|
| **~30 s of mixed-version traffic during rolling pod restart** | Managed by A/B router's `model_version` flag — users are pinned to their assigned version. The 30 s window affects only users whose requests land on a restarting pod; they receive a brief 503 handled by the load balancer's retry. |
| **Double compute cost during blue/green window** | Green spins up 3 additional replicas during promotion (~2–4 hours max). Cost impact: ~$0.80/hour extra on c6i.xlarge. Acceptable for the rollback safety it provides. |
| **Flag service (LaunchDarkly) outage** | Inference service startup config defines champion model as default. A flag-service outage causes all traffic to fall back to champion — safe. |
| **Green environment accumulates if promotion stalls** | Max promotion window is 4 hours. After 4 hours without full promotion, the green deploy is automatically torn down and the incident is escalated. |
| **Prediction score distribution differs between blue and green** | Monitored by Evidently per `model_version`. A divergence alert fires if score distributions differ by KL > 0.15 unexpectedly (i.e., outside a known experiment window). |

---

## Alternatives Considered and Rejected

### A — Recreate (hard cutover)
Tear down all old replicas, spin up new ones. Simple, no mixed-version window.

**Rejected because:** Introduces 30–60 seconds of hard downtime. At 800 RPS, this means ~24,000–48,000 failed requests per cutover. Unacceptable for a home-screen feature in a retail app.

### B — Rolling update (Kubernetes default)
Replace replicas one at a time. No downtime, simple.

**Rejected because:** During the ~90 s rollout window, replicas run different model versions with no user-level routing control. This corrupts A/B experiment impression logs. The product team's continuous experimentation requirement makes this a hard constraint, not a preference.

### C — Remote model server (Triton / TorchServe) with hot-swap
Deploy a separate model server process; swap the model artifact without restarting the inference service.

**Rejected because:** Adds one network hop per request (~5–15 ms), reducing the latency headroom. At the current re-ranker size (~50 MB ONNX), in-process loading is fast enough (~2 s at startup) that there is no operational benefit to a remote server. Flagged in `JUSTIFICATION.md §3` as a future inflection point if the re-ranker grows to a GPU-bound deep model.

---

## Deployment Runbook Reference

Full step-by-step promotion and rollback procedures are in [`runbooks/rollback.md`](../../runbooks/rollback.md).

The CI/CD pipeline that automates the promotion gates is defined in [`.github/workflows/deploy-model.yml`](../../cicd/.github/workflows/deploy-model.yml).

---

## Review Checkpoint

This decision should be revisited if any of the following conditions are met:

- Re-ranker grows to require GPU inference — remote model server (Triton) becomes attractive to avoid per-replica GPU allocation.
- Number of concurrent A/B experiments exceeds 5 — traffic splitting logic may need to move into the inference service itself.
- Kubernetes cluster adopts Argo Rollouts — native progressive delivery tooling may replace the custom A/B router integration.
