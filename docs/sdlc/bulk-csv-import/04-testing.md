# Test Plan — Bulk CSV import for accounts

**Feature slug:** `bulk-csv-import`
**Date:** 2026-05-04
**Coverage target:** 85% line coverage on new code (touches PII + write-heavy path -> stricter than default 80%)

## 1. Pyramid

| Level | Count | Lives in | Runs on |
|---|---|---|---|
| Unit | ~28 | `src/services/imports/__tests__/` | every commit |
| Integration | ~9 | `tests/integration/imports.*.test.ts` | every PR |
| End-to-end | 4 | `tests/e2e/imports.spec.ts` (Playwright) | every PR for changed flows; nightly full sweep |
| Exploratory | 2 charters | manual | once before GA |

## 2. Per-story coverage

### Story 1: Upload + validate
- **Happy path**: 1K-row valid CSV uploads, presign succeeds, start enqueues a job
- **Negative**:
  - File > 50MB -> presign returns 400 with `size_exceeded`
  - Wrong MIME -> rejected
  - Empty body -> rejected
- **Edge cases**:
  - BOM-prefixed UTF-8 (`\ufeff`)
  - CRLF and LF line endings on the same file
  - Embedded commas in quoted fields (`"Doe, John"`)
  - Emoji in account names
  - Excel's double-encoded quotes (`""hello""`)
  - 49,999 rows (just under cap) processes successfully
  - 50,001 rows (just over cap) is rejected without parsing the whole file

### Story 2: Async processing + dedup
- **Happy path**: 5K rows processed, 0 duplicates, all created
- **Negative**:
  - Parallel job from same tenant -> 409 (parallel-job cap = 1)
  - Worker crash mid-job -> on restart, resumes from last checkpointed row (BullMQ + per-row outcomes table)
  - Row with existing external_id for tenant -> outcome = `skipped: duplicate` (not error)
  - Duplicate rows within the same file -> first wins, rest skipped

### Story 3: Outcome report
- Errors CSV downloadable; columns: `row_no, original_row, error_message`
- 0-error case -> no errors CSV produced (UI handles `errors_s3_key = null`)

### Story 4: Audit log
- Audit row written on `started` and `completed` (or `failed`)
- Audit row contains: tenant_id, user_id, file_sha256, counts, timestamps
- Deleting the user does NOT cascade-delete the audit row

### Story 5: Progress visibility
- SSE emits an event every 2 seconds with `rows_processed` and `eta_seconds`
- Client can reconnect to SSE if disconnected; resumes from current state
- Polling fallback every 3 seconds if SSE fails

## 3. Unit test stubs

```ts
// src/services/imports/__tests__/repository.test.ts
describe('ImportsRepository', () => {
  it('creates a job scoped to the tenant');
  it('lists jobs for the tenant in DESC order by created_at');
  it('refuses to read a job belonging to a different tenant');
  it('atomically increments created_count via UPDATE ... RETURNING');
});

// src/services/imports/__tests__/csvStream.test.ts
describe('csvStream', () => {
  it('handles BOM prefix');
  it('handles CRLF and LF line endings');
  it('streams 50,000 rows without buffering all in memory');
  it('rejects rows where any cell exceeds IMPORTS_MAX_CELL_BYTES');
  it('preserves quoted commas');
});

// src/services/imports/__tests__/validators/row.test.ts
describe('row validator', () => {
  it('rejects empty email');
  it('rejects malformed email');
  it('accepts unicode names including emoji');
  it('flags missing required external_id');
});

// src/services/imports/__tests__/processImport.test.ts
describe('processImport worker', () => {
  it('creates accounts for valid rows');
  it('skips duplicates by tenant + external_id');
  it('records errored rows with row_no and message');
  it('writes audit_events on start and completion');
  it('writes errors CSV to S3 when errored_count > 0');
  it('updates job status to completed atomically');
  it('handles worker crash mid-job (resumes via BullMQ retry)');
});
```

## 4. Integration tests

- `POST /v1/imports/presign` with bearer token returns valid presigned URL with TTL 5min
- `POST /v1/imports` with non-existent S3 key returns 400
- Full happy path: upload 100-row CSV, poll until `completed`, verify accounts table has 100 new rows scoped to tenant
- Tenant isolation: tenant A cannot read tenant B's import job (verify 404, not 403, to avoid info leak)
- Migration applies cleanly to a copy of staging Postgres

## 5. E2E / UX tests (Playwright)

- Admin completes upload happy path on the new screen
- Screen is keyboard-navigable; SR reads progress percentage
- Error state shows correct message; errors CSV download triggers
- Progress UI updates within 5s of worker progress

## 6. Exploratory charters

- **Charter A**: 30 min on slow 3G + screen reader. Document any surprises.
- **Charter B**: try to break tenant isolation by tampering with the JWT. Document outcome.

## 7. Test data strategy

- Synthetic fixtures via `factories/account.factory.ts` (existing)
- For perf: scripted 50K-row CSV generator in `scripts/gen-import.ts`; checked into repo
- For e2e: seeded Postgres snapshot per test run

## 8. Performance test

- k6 script importing 10K rows with 5 concurrent tenants
- Target: p95 import-job completion time < 6 minutes for 10K rows; < 30 minutes for 50K rows
- Watch: worker memory, Postgres connections, BullMQ wait time

## 9. Flaky test policy

- SSE-based tests are most likely to flake. Use polling fallback in tests if SSE fails.
- Triage SLA: within 2 business days; quarantine after 2 flakes / 7 days.

## 10. Hand-off

Next stage: **Security** (`sdlc-security`). Artifact: `05-security.md`.
