# Requirements — Bulk CSV import for accounts

**Feature slug:** `bulk-csv-import`
**Status:** Approved by product
**Author:** sevaai-sdlc / sdlc-requirements skill
**Date:** 2026-05-04

## 1. Summary

Tenant admins of Aurora today create accounts one-at-a-time via the API or UI form. Onboarding a Fortune-500 customer with hundreds of accounts is painful; CS reps copy-paste for hours. This feature lets a tenant admin upload a CSV of up to 50,000 accounts in one shot, validates each row, dedupes against existing data, processes asynchronously, and reports per-row results. Closes the #2 driver of CS-time burn this quarter and unblocks two enterprise deals worth ~$1.4M ARR.

## 1a. Planning

| Sub-activity | Detail |
|---|---|
| **Project scope** | In: CSV upload (UTF-8, comma/tab separated), validation, dedup, async processing, progress + outcome report, audit log, SOC 2 evidence. Out: XLSX import, scheduled imports, API-driven imports (no admin UI), partial failure rollback (per-row outcomes only). |
| **Objectives & goals** | Reduce account-onboarding time from ~3h per 1K accounts (manual) to <5 min for 1K, <30 min for 50K. |
| **Resource planning** | 2 backend (data-platform squad), 1 frontend (portal squad), 0.5 designer, 0.5 SRE, partner: tenant-admin team. |
| **Timeline** | Design lock 2026-05-15; first PR 2026-05-20; canary 2026-06-10; GA 2026-06-24. |
| **Success metrics** | (1) p95 import-job completion time at 50K rows < 30 min. (2) Self-serve onboarding share = 60% of new tenant onboardings within 60 days post-GA. (3) CS time per onboarding -50%. |

## 2. Personas

| Persona | Role | Primary need |
|---|---|---|
| Tenant admin | enterprise customer's IT/CS leader | upload accounts in bulk without engineering help |
| CS rep | Aurora's customer success | help customer recover from a partially-failed import |
| Auditor | SOC 2 reviewer | see who imported what, when, with what outcome |

## 3. Business goal

Two F500 deals (~$1.4M ARR) explicitly require self-serve bulk onboarding. CS-time savings expected at ~12 hours per major onboarding. Pull-forward of revenue: target two deals close in Q3 vs Q4.

## 4. User stories

### Story 1: Upload + validate
- **As a** tenant admin, **I want** to upload a CSV file of accounts via the portal, **so that** I don't have to create them one-by-one.
- **Acceptance criteria**
  - Given a valid CSV with 1-50,000 rows, when the admin uploads, then a job is created and a progress page is shown.
  - Given a CSV >50,000 rows or >50MB, when uploaded, then the request is rejected with a clear error.
  - Given a CSV with malformed rows (missing required field, bad email format), when uploaded, then validation completes and per-row errors are listed.
- **Edge cases**: empty file, BOM-prefixed UTF-8, CRLF vs LF endings, embedded commas in quoted fields, emoji in account names, Excel-exported CSV with double-encoded quotes.
- **Estimate**: M

### Story 2: Async processing + dedup
- **As a** tenant admin, **I want** the import to keep running even if I close the browser, **so that** I can come back later.
- **Acceptance criteria**
  - Given a job in progress, when the user closes the browser and returns, then they see current progress.
  - Given a row whose `external_id` already exists for the tenant, when processed, then it's marked `skipped: duplicate` (not updated, not errored).
  - Given a worker crash mid-job, when the worker restarts, then processing resumes from the last successful row.
- **Edge cases**: parallel jobs from the same tenant, duplicate rows within the same file, idempotency on retry.
- **Estimate**: M

### Story 3: Outcome report
- **As a** tenant admin, **I want** a per-row outcome report after the job completes, **so that** I can fix and re-upload the failures.
- **Acceptance criteria**
  - Given a completed job, when the user opens it, then they see total / created / skipped / errored counts and a downloadable CSV of errors.
  - Given an errored row, then the report shows the row number, the original values, and the validation error.
- **Estimate**: S

### Story 4: Audit log (SOC 2)
- **As an** auditor, **I want** every import logged with who, when, file hash, and outcome, **so that** the import is reviewable in evidence collection.
- **Acceptance criteria**
  - Given a completed job, then the audit log records: tenant_id, user_id, file_sha256, row_count, created_count, errored_count, started_at, completed_at, outcome_csv_path.
  - Given a deleted user, the historical audit row remains (reference by ID).
- **Estimate**: S

### Story 5: Progress visibility
- **As a** tenant admin, **I want** real-time progress (% complete, estimated time remaining), **so that** I know whether to wait or come back later.
- **Acceptance criteria**
  - Given a running job, then the UI updates progress at least every 10 seconds.
  - Given a job stuck >5 min without progress, then the UI surfaces a "may need support" CTA.
- **Estimate**: S

## 5. Out of scope

- Excel (.xlsx) imports — CSV only this iteration; XLSX in a follow-up.
- Update-existing-accounts — first iteration is create-or-skip, not upsert.
- Scheduled / recurring imports.
- API-driven bulk imports (admin UI only).

## 6. Risks & open questions

| # | Risk / question | Owner | Resolve by |
|---|---|---|---|
| 1 | What happens if the upload finishes but worker crashes before any row is processed? Should we retry automatically? | tech lead | design lock |
| 2 | Per-tenant rate limits — should one tenant be able to run 5 jobs in parallel, or cap at 1? | product | design lock |
| 3 | Error CSV retention — keep forever, 90 days, or 7 days? Trade-off vs storage cost + GDPR. | privacy + compliance | design lock |

## 7. Compliance flags

- **PII**: account email, contact name, optional phone — all PII. Triggers `sdlc-security` deeper review.
- **Tenant isolation**: imports MUST stay within the tenant's schema. Cross-tenant leak = critical.
- **Audit (SOC 2)**: Story 4 is required evidence.
- **GDPR**: EU customers — import file storage retention must respect data subject deletion requests.
- **Accessibility (WCAG 2.1 AA)**: progress UI must be screen-reader compatible; not n/a.

## 8. Hand-off

Next stage: **Design** (`sdlc-design`). Artifact: `02-design.md`.
