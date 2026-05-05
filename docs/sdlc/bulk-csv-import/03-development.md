# Development Plan — Bulk CSV import for accounts

**Feature slug:** `bulk-csv-import`
**Date:** 2026-05-04

## 1. PR breakdown

| # | Title | Scope | Approx LOC | Behind flag? |
|---|---|---|---|---|
| 1 | feat: add import_jobs migration + repository | DB migration + repository module + unit tests | ~180 | No |
| 2 | feat(imports): presign and start endpoints | Two API handlers + zod validators + integration test | ~250 | Yes |
| 3 | feat(imports): worker with papaparse + dedup | BullMQ processor + parser + per-row outcomes | ~320 | Yes |
| 4 | feat(imports): progress SSE + GET endpoints | SSE handler + status lookup + frontend client | ~200 | Yes |
| 5 | chore(imports): observability + flag rollout config | Datadog dashboards JSON + LaunchDarkly seed | ~80 | n/a |

## 2. Per-PR detail

### PR 1 — feat: add import_jobs migration + repository
- **Files added**:
  - `migrations/2026_05_05_add_import_jobs.sql`
  - `src/services/imports/repository.ts`
  - `src/services/imports/types.ts`
  - `tests/services/imports/repository.test.ts`
- **Files modified**: `src/services/index.ts` (export ImportsRepository)
- **Reviewers**: 1 backend, 1 DBA
- **Risk**: low — additive migration, no backfill, no data plane impact

### PR 2 — feat(imports): presign and start endpoints
- **Files added**:
  - `src/services/imports/handlers/presign.ts`
  - `src/services/imports/handlers/start.ts`
  - `src/services/imports/validators/file.ts`
  - `tests/integration/imports.presign.test.ts`
- **Files modified**: `src/routes.ts` (register routes), `src/services/imports/service.ts`
- **Notes**: presign returns `max_size_bytes` so UI can validate client-side too; start endpoint enforces parallel-job cap by tenant.
- **Reviewers**: 1 backend, 1 security (auth-touching)
- **Risk**: medium — touches request boundary

### PR 3 — feat(imports): worker with papaparse + dedup
- **Files added**:
  - `src/services/imports/workers/processImport.ts`
  - `src/services/imports/parsers/csvStream.ts`
  - `src/services/imports/validators/row.ts`
  - `tests/services/imports/processImport.test.ts`
  - `tests/services/imports/csvStream.test.ts`
- **Files modified**: `src/workers/index.ts` (register processor)
- **Notes**: streaming parse with backpressure; per-batch insert (1000 rows) for throughput; dedup via UNIQUE (tenant_id, external_id) on accounts table — relies on existing constraint.
- **Reviewers**: 1 backend (lead), 1 DBA (write patterns)
- **Risk**: medium — high write volume

### PR 4 — feat(imports): progress SSE + GET endpoints
- **Files added**:
  - `src/services/imports/handlers/progress.ts`
  - `src/services/imports/handlers/get.ts`
  - `tests/integration/imports.progress.test.ts`
  - frontend: `app/imports/[id]/page.tsx` (separate repo)
- **Notes**: SSE keeps connection 5 min max, then client reconnects. Falls back to polling every 3s if SSE fails.
- **Reviewers**: 1 backend, 1 frontend
- **Risk**: low

### PR 5 — chore(imports): observability + flag rollout config
- **Files added**:
  - `dashboards/imports.json` (Datadog)
  - `launchdarkly/imports.json` (flag definitions seed)
  - `runbooks/imports.md` (linked to from stage 7)
- **Notes**: Default flag state = OFF in prod. Stage 6 plan controls the actual rollout schedule.
- **Reviewers**: 1 SRE
- **Risk**: none

## 3. Naming & style

Match existing Aurora conventions (validated against `src/services/accounts/`):

- File names: `kebab-case.ts` for modules, `camelCase.ts` for handlers
- Class / type names: `PascalCase`
- Functions: `camelCase`
- Test files alongside module files, suffix `.test.ts`
- Match existing import ordering (eslint enforces); run `npm run lint:fix` before commit
- Logger: `pino` with `tenant_id` always in context; never log row contents (PII)

## 4. Self-review checklist

- [ ] Each PR builds and passes lint independently (`npm run build && npm run lint`)
- [ ] No `console.log` or commented-out code
- [ ] All public functions have JSDoc; types exported via `types.ts`
- [ ] Migration is reversible (add `-- DOWN` section)
- [ ] No PII in logs (only counts, IDs, file_sha256)
- [ ] No secrets in code or fixtures (S3 creds via task role)
- [ ] Feature flag default OFF in `launchdarkly/imports.json`
- [ ] CHANGELOG.md updated with [Unreleased] entry
- [ ] No new dependencies without security review (papaparse is the only addition)

## 5. Coding-agent prompt

Paste this into Cursor / Claude Code / Copilot Workspace to start PR 1:

> "Implement PR 1 from `.sevaai-sdlc/bulk-csv-import/03-development.md` against the Aurora codebase. The design is in `02-design.md` (sections 3.1, 3.3, 3.5). Create the listed files following Aurora's conventions in `src/services/accounts/` (Fastify + zod + pino + node-pg). Stop before writing tests — tests are scoped to PR 1 but follow patterns from the testing stage. The migration must be additive and reversible."

## 6. Hand-off

Next stage: **Testing** (`sdlc-testing`). Artifact: `04-testing.md`.
