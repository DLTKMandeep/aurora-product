# Deployment Plan — Bulk CSV import for accounts

**Feature slug:** `bulk-csv-import`
**Date:** 2026-05-04
**Release window:** 2026-06-10 09:00 ET — 17:00 ET (canary); 2026-06-24 GA
**Approver:** Eng lead (data-platform) + SRE on-call

## 1. Environment promotion

```
dev (auto on PR merge to main)
  -> staging (auto on main; nightly perf run)
  -> canary 1% (manual gate; eng lead clicks)
  -> prod (auto on canary green for 4h)
```

Pre-prod gate is "all 5 PRs merged + dashboards deployed + runbook signed off."

## 2. Feature flag plan

- **Flag**: `imports.bulk_csv_enabled`
- **Provider**: LaunchDarkly
- **Default**: `false` in prod
- **Rollout schedule**:
  - T+0h:    1% of tenants (~3 customers)  — canary
  - T+4h:    10% (~30 customers)
  - T+1d:    50%
  - T+3d:    100%
- **Per-tenant override**: explicit allow-list for the two F500 customers driving the requirement (they get the feature on day 1 of canary)
- **Kill switch**: flip flag to `false` globally if SLO violated for >5 min — see runbook

## 3. Canary analysis

| SLI | Baseline | Threshold | Action |
|---|---|---|---|
| API error rate (`POST /v1/imports*`) | 0.1% | +0.5pp | auto rollback |
| API p95 latency (presign) | 80ms | +50% (>120ms) | auto rollback |
| Worker job success rate | n/a (new) | <95% | hold rollout, page DRI |
| BullMQ queue depth | <10 | >100 sustained 5min | page DRI |
| S3 5xx rate (PUT/GET aurora-imports) | 0.05% | +0.5pp | hold + investigate |
| Tenant satisfaction (in-app survey on completed jobs) | n/a (new) | <4/5 | hold next phase |

Canary analysis tool: Datadog Continuous Verification with the dashboard from PR 5.

## 4. Database migrations

- Migration file: `migrations/2026_05_05_add_import_jobs.sql`
- Forward: `CREATE TABLE import_jobs ... CREATE UNLOGGED TABLE import_job_rows ...` — additive, instant on each tenant schema
- Backward: `DROP TABLE import_jobs; DROP TABLE import_job_rows;` — safe to revert (no data dependencies)
- Backfill: not required (new tables, no existing data)
- Applied in: PR 1 (separate from code changes — expand-contract compliant)

## 5. Rollback runbook

When the canary SLI breaches the threshold or a manual abort is needed:

1. **Disable the flag** (most cases this is sufficient)
   ```bash
   ld feature-flag-toggle imports.bulk_csv_enabled --env prod --state false
   ```
   Confirm in LaunchDarkly UI within 30s.

2. **Verify error rate returns to baseline** (Datadog: Imports dashboard)

3. **If error rate persists** (suggests deployed code is bad even with flag off):
   ```bash
   kubectl rollout undo deployment/api -n prod
   kubectl rollout undo deployment/worker -n prod
   ```

4. **If Postgres-related**:
   - Confirm migration didn't lock anything: `SELECT * FROM pg_locks WHERE NOT granted;`
   - Migration is additive — no rollback needed unless table is interfering. To drop: `DROP TABLE import_jobs CASCADE; DROP TABLE import_job_rows CASCADE;` (only after confirming no in-flight jobs).

5. **Page DRI** if not resolved in 15 min.

6. **Open postmortem ticket** within 24h.

## 6. Release notes

### User-facing
- **New**: tenant admins can upload up to 50,000 accounts via CSV from Settings → Accounts → Bulk Import. Per-row validation, duplicate detection, and error report included.

### Internal
- New API endpoints: `POST /v1/imports/presign`, `POST /v1/imports`, `GET /v1/imports/:id`, `GET /v1/imports/:id/progress` (SSE)
- New DB tables: `import_jobs`, `import_job_rows`
- New S3 bucket: `aurora-imports-prod`
- New env vars: `IMPORTS_S3_BUCKET`, `IMPORTS_MAX_BYTES`, `IMPORTS_MAX_ROWS`, `IMPORTS_MAX_CELL_BYTES`, `IMPORTS_TENANT_PARALLEL`, `IMPORTS_WORKER_CONCURRENCY` (defaults safe; only override per env if needed)
- New IAM role: `aurora-imports-worker-role`
- New BullMQ queue: `imports`

### Deprecations
- None.

## 7. Comms

| Audience | Channel | Timing |
|---|---|---|
| Customer support | `#cx-releases` Slack | T-24h before canary |
| Sales / CS | release-bulletin email | T-24h before GA |
| Two F500 design partners | direct CSM email | T-7d before canary (they're allowlisted day 1) |
| Customers (general) | status page entry | T+0 GA |
| Internal eng | `#releases` Slack | T+0 each phase |

## 8. Cost & capacity delta

- **API QPS**: +<1% (new endpoints used at low volume)
- **Worker compute**: peak +20% during business hours (typical import bursts at 10am-noon ET); fits within current Fargate auto-scale ceiling
- **Postgres**: ~50K row INSERTs / second / tenant during a 50K import; well within RDS r6g.2xlarge headroom
- **S3 storage**: ~5GB / month at full rollout (avg import ~10MB; 500 imports / month estimated)
- **Egress**: negligible
- **Inference cost**: n/a (no LLM in feature)
- **Action**: no infrastructure scaling change required.

## 9. Hand-off

Stage 7 (Maintenance) was not requested for this run. The runbook stub is in `runbooks/imports.md` (PR 5); a fuller `07-maintenance.md` artifact would normally close out the dossier with SLOs, on-call cheat sheet, postmortem template, and feedback-loop triggers.
