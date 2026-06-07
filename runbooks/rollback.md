# Rollback Runbook — Vision Moderation Model

## When to Roll Back

Roll back **immediately** if any of the following is true for > 2 minutes:

| Signal | Threshold |
|---|---|
| HTTP 5xx error rate | > 0.5% (half the SLO budget) |
| P99 latency | > 1.5 s |
| `AvailabilityBurnFast` | Firing |
| `LatencyP99High` | Firing |
| `ModelVersionMismatch` | Firing AND canary health check failed |
| Quality proxy (human-review override rate) | > 2× baseline |

Do **not** wait for slow-burn alerts to escalate. If two or more signals are yellow simultaneously, roll back.

---

## How to Roll Back

**Option A — Trigger via workflow_dispatch (preferred):**
```
gh workflow run deploy-model.yml \
  -f action=rollback \
  -f target_sha=<last-known-good-sha>
```

**Option B — Run rollback script directly:**
```bash
./scripts/rollback.sh production <last-known-good-sha>
```

Find the last known good SHA in the GitHub Actions run history or in the GHCR image tags for the last green `smoke-staging` run.

---

## What to Verify (all must be green)

- [ ] `AvailabilityBurnFast` — resolved in Alertmanager
- [ ] `LatencyP99High` — resolved in Alertmanager
- [ ] `ModelVersionMismatch` — resolved; all replicas show the rolled-back version
- [ ] Grafana **SLO dashboard** — error rate < 0.5%, p99 < 800 ms for 5 min
- [ ] Grafana **Canary panel** — traffic split back to 100% stable

---

## Who to Notify

| When | Channel | Who |
|---|---|---|
| Rollback initiated | `#incidents` (Slack) | On-call engineer |
| Rollback complete | `#incidents` + Linear ticket | On-call + ML Platform lead |
| SLO breach confirmed | PagerDuty escalation | On-call manager |

---

## What NOT to Do

- **Do not roll forward** before root cause is identified and a fix is confirmed in staging.
- **Do not re-deploy the same SHA** — it will fail the same way.
- **Do not skip smoke tests** in staging under time pressure.
- **Do not manually patch replicas** in production — use the rollback script to keep state consistent.
- **Do not silence alerts** to buy time — address the issue.

---

## When to Roll Forward

Re-deploy only when all of the following are true:

1. Root cause is documented in the incident Linear ticket.
2. Fix is merged, CI is green, and smoke tests pass on staging.
3. On-call lead has approved the re-deploy.
4. Rollback took place more than 30 minutes ago (cooldown period).

Run the normal pipeline — do not bypass scan or staging stages.
