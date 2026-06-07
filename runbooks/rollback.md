# Rollback Runbook — NorthStar Inference Service

**Owner:** On-call SRE
**Last updated:** 2026-06-07
**Estimated execution time:** < 5 minutes (flag flip) | < 15 minutes (full verification)

---

## When to Roll Back

Roll back immediately if **any** of the following are true:

| Condition | Source | Threshold |
|---|---|---|
| HTTP error rate (5xx) | `NorthStarAvailabilityFastBurn` alert | > 14× burn rate over 1 h |
| p95 latency | `NorthStarLatencyP95FastBurn` alert | > 14× burn rate over 1 h |
| Cold-start fallback rate spike | `NorthStarColdStartRateHigh` alert | > 10% of requests |
| Unexpected model version in production | `NorthStarModelVersionMismatch` alert | 2+ versions outside experiment window |
| NDCG online proxy drop | Canary gate (deploy-model.yml) | > 1% below champion |
| Manual call by ML Lead or SRE Lead | — | Any production quality concern |

Thresholds match `monitoring/alerts.yaml` and `serving/slos.yaml`.

---

## Step 1 — Confirm the incident (1 min)

- [ ] Open Grafana dashboard: `https://grafana.northstar.internal/d/northstar-infer`
- [ ] Confirm the alert is genuine (not a Prometheus scrape gap or dashboard bug).
- [ ] Check `X-Model-Version` label in the dashboard — confirm which model version is degraded.
- [ ] If degradation is on the **champion** model (not the challenger), this is NOT a model rollback. Page the SRE Lead and skip to Step 5.

---

## Step 2 — Flip router flag to champion (< 5 s)

This is the primary rollback action. It routes 100% of traffic back to the champion model without restarting any pods.

**Option A — GitHub Actions (recommended):**
```bash
gh workflow run deploy-model.yml \
  -f action=rollback \
  -f target_version=<challenger_version>
```
Replace `<challenger_version>` with the version being rolled back (e.g. `v2.5.0`).

**Option B — LaunchDarkly UI (fastest):**
1. Open [LaunchDarkly → NorthStar → `model-version` flag](https://app.launchdarkly.com)
2. Set rollout: `champion: 100%`, `challenger: 0%`
3. Save. Traffic shifts in < 5 seconds.

**Option C — LaunchDarkly API (scripted):**
```bash
curl -s -X PATCH \
  "https://app.launchdarkly.com/api/v2/flags/northstar/model-version" \
  -H "Authorization: ${LAUNCHDARKLY_API_KEY}" \
  -H "Content-Type: application/json; domain-model=launchdarkly.semanticpatch" \
  -d '{
    "instructions": [{
      "kind": "updateRuleVariationOrRollout",
      "rolloutWeights": {"champion": 100000, "challenger": 0}
    }]
  }'
```

---

## Step 3 — Verify traffic is back on champion (2 min)

- [ ] In Grafana, confirm the `model_version` label in `http_requests_total` shows only the champion version within 30 seconds.
- [ ] Confirm error rate is dropping: `NorthStarAvailabilityFastBurn` alert should resolve within 2 minutes.
- [ ] Confirm p95 latency is recovering.
- [ ] Run a manual smoke test:
```bash
curl -s -X POST https://api.northstar.internal/v1/recommendations \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: rollback-smoke-test-$(date +%s)" \
  -d '{"user_id": "usr_smoketest_001", "num_results": 5}' \
  | jq '{cold_start, model_version, count: (.recommendations | length)}'
```
Expected: `model_version` matches champion tag; `cold_start: false`; 5 results returned.

---

## Step 4 — Preserve green pods for post-mortem (do not terminate immediately)

- [ ] Green pods remain running for **60 minutes** post-rollback (per `model-registry.yaml` `preserve_green_pods_minutes: 60`).
- [ ] **Do not** scale down or delete the green deployment until the post-mortem is complete.
- [ ] Export logs from green pods before they are terminated:
```bash
kubectl logs -l app=northstar-infer,slot=green \
  -n northstar-prod \
  --since=2h > /tmp/green-pod-logs-$(date +%Y%m%d-%H%M).txt
```

---

## Step 5 — Update model registry state

- [ ] Mark the challenger version as `ROLLED_BACK` in MLflow:
```bash
python - <<'EOF'
import mlflow, os
client = mlflow.tracking.MlflowClient(os.environ["MLFLOW_TRACKING_URI"])
client.transition_model_version_stage(
    "northstar-reranker",
    "<challenger_version>",
    "Archived"
)
client.set_model_version_tag(
    "northstar-reranker",
    "<challenger_version>",
    "rollback_reason",
    "canary gate failure — see incident ticket"
)
EOF
```

---

## Step 6 — Create incident ticket and notify

- [ ] Create incident ticket in Jira (project: MLOPS, component: serving):
  - Title: `Rollback: northstar-reranker <challenger_version> — <brief reason>`
  - Link to Grafana snapshot of the degradation window.
  - Link to PagerDuty incident.
- [ ] Post in `#incidents` Slack:
```
🔴 *Rollback executed* — northstar-reranker `<challenger_version>` rolled back to champion `<champion_version>`.
Traffic returned to champion in < 5 s. Green pods preserved for post-mortem.
Incident ticket: <JIRA link>
```
- [ ] Notify `#ml-deployments` that deployments are frozen until root cause is identified.

---

## Step 7 — Decide on retraining (do not rush)

Per `model-registry.yaml` `queue_retraining: false` (default) — retraining is not automatic after a rollback. The ML Engineer and ML Lead decide after root cause analysis:

| Root cause | Action |
|---|---|
| Model regression (NDCG drop) | Retrain with corrected training data or architecture |
| Infrastructure issue (Redis, OOM) | Fix infrastructure; rollback was sufficient |
| Data pipeline bug | Fix pipeline; retrain on clean data |
| Flaky test / false positive rollback | Re-promote champion-compatible challenger after review |

---

## Escalation

| Scenario | Escalate to |
|---|---|
| Rollback does not stop degradation within 5 min | SRE Lead (pagerduty escalation) |
| Champion model is also degraded | SRE Lead + ML Lead immediately |
| Redis cluster failure | SRE Lead + AWS support |
| PagerDuty alert not resolving after rollback | SRE Lead |

---

## Reference

- Alert definitions: `monitoring/alerts.yaml`
- SLO thresholds: `serving/slos.yaml`
- Blue/green strategy: `architecture/adr/0002-blue-green-deploy.md`
- Registry state machine: `lifecycle/model-registry.yaml` → `rollback_policy`
- CI/CD rollback job: `cicd/.github/workflows/deploy-model.yml` → job `rollback`
