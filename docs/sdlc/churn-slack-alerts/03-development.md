# Development Plan — Churn-Risk Slack Alerts

**Feature slug:** `churn-slack-alerts`
**Date:** 2026-05-05
**Inputs:** [`01-requirements.md`](./01-requirements.md), [`02-design.md`](./02-design.md)

## 0. Pre-implementation corrections

Two facts surfaced while grounding this plan against the live repo. They are blockers for PR 1 and must land in the docs before code starts.

| # | Issue | Fix |
|---|---|---|
| 0.1 | `02-design.md` numbers the new ADR as **ADR-0002**, but `docs/architecture.md` lists ADR-0002 as *BullMQ over SQS* and ADR-0003 as *Auth0*. | Renumber the Slack-token-storage ADR to **ADR-0004** in `02-design.md` §4 and in §8's hand-off. PR 1 carries the doc-only edit. |
| 0.2 | The repo has **no `tsconfig.json`, no `.eslintrc.*`, no `.prettierrc`, and no `.github/workflows/`** — only `package.json` and stub `src/`/`migrations/`/`tests/` directories. The requirements doc's "80% coverage gate", "Conventional Commits", and "PR ≤400 lines" all assume tooling that doesn't yet exist. | PR 1 scaffolds the TS toolchain and the first GitHub Actions workflow. This adds 3 SP of infra overhead beyond the Stage-1 sizing — see §1 budget reconciliation. |

## 1. PR breakdown

Story-point calibration: **1 SP ≈ 1 engineer-day**. Stage-1 sizing was M+M+M+M+S+M (Stories 1-6), so target budget = 5+5+5+5+2+5 = **27 SP** of feature work. Adding the §0 infra prereqs (PR 1) brings total to **30 SP**, which fits the 6-week implementation window in the requirements timeline (M2-M5) for 1 BE + 0.5 FE.

| # | Title | Story / Component | LOC | SP | Behind flag? |
|---|---|---|---|---|---|
| 1 | `chore(infra): scaffold TS + ESLint + CI baseline` | infra (covers §0.1, §0.2) | ~250 | 3 | n/a |
| 2 | `feat(score): add score_run.is_backfill flag` | SDLCTEST-4 prereq | ~120 | 2 | n/a (column add only) |
| 3 | `feat(slack): OAuth connect/disconnect` | SDLCTEST-2 / `slack/oauth` | ~380 | 5 | `slack-alerts.enabled` (off) |
| 4 | `feat(slack): channel routing CRUD` | SDLCTEST-3 / `slack/routing` | ~320 | 4 | flag |
| 5 | `feat(churn): upward 0.8 crossing detector` | SDLCTEST-4 / `churn/crossing-detector` | ~280 | 4 | flag (per-tenant gate before enqueue) |
| 6 | `feat(slack): dispatcher worker + retry` | SDLCTEST-5 / `slack/dispatcher` | ~360 | 5 | flag |
| 7 | `feat(slack): cooldown + burst cap + needs_reconnect` | SDLCTEST-6 / `slack/dispatcher` | ~220 | 2 | flag |
| 8 | `feat(slack): activity audit + test alert` | SDLCTEST-7 / `slack/audit` + `slack/test-alert` | ~340 | 4 | flag |
| 9 | `chore(slack): pilot LD rollout + ops links` | flag config + runbook stub | ~80 | 1 | n/a (toggles flag) |
| | **Total** | | **~2,350** | **30** | |

Each PR stays under the 400-line PR-size budget from `README.md`. PR 6 is the closest to the limit; if it exceeds 400, lift the `payload.ts` block-renderer into a second PR before merging.

## 2. Per-PR detail

References below use the §3.1 module IDs from `02-design.md`.

### PR 1 — `chore(infra): scaffold TS + ESLint + CI baseline`

- **Files added**:
  - `tsconfig.json` (Node 20, ESM, `strict: true`, `target: ES2022`, `moduleResolution: "Bundler"`)
  - `.eslintrc.cjs` (`@typescript-eslint/recommended`, `import/order`, `no-floating-promises`)
  - `.prettierrc.json` (2-space, single quotes, trailing comma `all`)
  - `.github/workflows/ci.yaml` — see §5 for jobs
  - `.github/workflows/migration-parity.yaml` — see §5
  - `.github/PULL_REQUEST_TEMPLATE.md` — see §4
  - `.github/CODEOWNERS` — backend leads + SRE for `migrations/**`
  - `commitlint.config.cjs` (`@commitlint/config-conventional`, scope-enum: `slack|churn|score|infra|ui|deps`)
  - `.husky/commit-msg`, `.husky/pre-commit`
- **Files modified**:
  - `package.json` — add devDeps: `commitlint`, `husky`, `prettier`, `eslint-plugin-import`, `@typescript-eslint/*`, `@aws-sdk/client-kms`, `@aws-sdk/client-secrets-manager`
- **Files deleted**: none
- **Reviewers**: Eng lead, SRE on-call
- **Risk**: low — additive; nothing depends on anything yet.
- **Doc edit**: in the same PR, fix `02-design.md` ADR number from **0002 → 0004** (§0.1).

### PR 2 — `feat(score): add score_run.is_backfill flag`

- **Files added**:
  - `migrations/1714939200000_score-run-is-backfill.up.sql`
  - `migrations/1714939200000_score-run-is-backfill.down.sql`
- **Files modified**:
  - `src/services/score/types.ts` — add `is_backfill: boolean` to the `ScoreRun` zod schema (file is currently empty; create if absent and document in CHANGELOG)
- **Reviewers**: 1 backend, 1 DBA (CODEOWNERS auto-assigns)
- **Risk**: low — Postgres 16 fast-path column add with default. No backfill, no rewrite.
- **Note**: this PR is the only one touching the shared (non-tenant) schema. Score recompute callers must start passing `is_backfill` in PR 5.

### PR 3 — `feat(slack): OAuth connect/disconnect` — SDLCTEST-2

- **Component**: `slack/oauth` (design §3.1)
- **Files added**:
  - `migrations/1714939300000_slack-connection.up.sql`
  - `migrations/1714939300000_slack-connection.down.sql`
  - `src/services/slack/oauth/handlers.ts` — `GET /v1/slack/oauth/start`, `/callback`, `POST /v1/slack/disconnect`
  - `src/services/slack/oauth/exchange.ts` — `oauth.v2.access` POST against Slack
  - `src/services/slack/oauth/tokens.ts` — KMS `GenerateDataKey` + AES-256-GCM wrapper; explicit zeroing of plaintext DEK
  - `src/services/slack/oauth/state.ts` — HMAC-signed `state` nonce with TTL
  - `src/services/slack/types.ts` — zod schemas
  - `src/lib/kms.ts` — thin client wrapper used by oauth + dispatcher
- **Files modified**:
  - `src/index.ts` — register the oauth route plugin
- **Behind flag**: `slack-alerts.enabled` evaluated server-side; if off, `/start` returns 404 (not 403, to avoid leaking feature existence).
- **Reviewers**: 1 backend, **1 security** (auth-touching change → IriusRisk entry required, per `.sevaai-sdlc.yaml`).
- **Secrets**: `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_STATE_HMAC_KEY`, `SLACK_KMS_CMK_ID` — provisioned by SRE before merge; PR includes `.env.example` updates.

### PR 4 — `feat(slack): channel routing CRUD` — SDLCTEST-3

- **Component**: `slack/routing` (design §3.1)
- **Files added**:
  - `migrations/1714939400000_slack-routing-rule.up.sql`
  - `migrations/1714939400000_slack-routing-rule.down.sql`
  - `src/services/slack/routing/handlers.ts` — `PUT /v1/slack/routing/default`, `GET|POST /v1/slack/routing/rules`, `DELETE /v1/slack/routing/rules/:id`
  - `src/services/slack/routing/repo.ts`
  - `src/services/slack/routing/channel-membership.ts` — proxies `conversations.list` (paginated, ≤200 per page per requirements §4 Story 2 edge case)
- **Files modified**:
  - `src/services/slack/types.ts` — add `RoutingRule`, `RoutingRuleInput`
  - `src/index.ts` — register routing plugin
- **Behind flag**: yes.
- **Depends on**: PR 3 (uses `slack_connection` row).

### PR 5 — `feat(churn): upward 0.8 crossing detector` — SDLCTEST-4

- **Component**: `churn/crossing-detector` (design §3.1)
- **Files added**:
  - `src/services/churn/crossing-detector/worker.ts` — BullMQ consumer for `churn.score_run.completed`
  - `src/services/churn/crossing-detector/detector.ts` — single SQL pulling crossings (`new>=0.8 AND prior<0.8`), enqueues with `jobId = ${tenant}:${account}:${score_run_id}` for idempotency
- **Files modified**:
  - `src/services/score/recompute.ts` (or wherever recompute lands; create stub if absent) — pass `is_backfill` on `score_run.completed` event
- **Behind flag**: detector calls `LD.variation('slack-alerts.enabled', { tenantId })` *before* enqueuing; flag-off tenants skip enqueue entirely (no audit row written from detector — dispatcher is the sole audit writer).
- **Reviewers**: 1 backend, 1 SRE (queue capacity review).
- **Depends on**: PR 2 (`is_backfill`), PR 3 (so a connection *can* exist; not a hard runtime dep).

### PR 6 — `feat(slack): dispatcher worker + retry` — SDLCTEST-5

- **Component**: `slack/dispatcher` (design §3.1, primary path)
- **Files added**:
  - `src/services/slack/dispatcher/worker.ts` — BullMQ consumer on `slack.alert`
  - `src/services/slack/dispatcher/post.ts` — `chat.postMessage` with `Retry-After`-aware retry (≤5 attempts, then dead-letter)
  - `src/services/slack/dispatcher/payload.ts` — mrkdwn block builder using **the static driver allowlist** at `src/services/slack/config/slack-driver-allowlist.json`
  - `src/services/slack/config/slack-driver-allowlist.json`
  - `src/services/slack/dispatcher/dek-cache.ts` — in-process LRU of decrypted DEKs, 60 s TTL (per ADR consequence)
- **Files modified**:
  - `src/index.ts` (worker entry) — register dispatcher
  - `src/services/slack/types.ts` — `SlackAlertJob` type
- **Behind flag**: yes (re-checked at consume time as well as enqueue).
- **Reviewers**: 1 backend, 1 SRE.

### PR 7 — `feat(slack): cooldown + burst cap + needs_reconnect` — SDLCTEST-6

- **Component**: `slack/dispatcher` (design §3.1, suppression branches)
- **Files added**:
  - `src/services/slack/dispatcher/cooldown.ts` — Redis `SET NX EX 86400` keyed by `(tenant, account)`
  - `src/services/slack/dispatcher/burst-cap.ts` — Redis sorted-set sliding window, default 50 / 60 min
  - `migrations/1714939500000_slack-alert-audit-partitioned.up.sql` (parent + first 2 monthly partitions)
  - `migrations/1714939500000_slack-alert-audit-partitioned.down.sql`
  - `src/services/slack/dispatcher/audit.ts` — single `INSERT slack_alert_audit` writer
  - `ops/cron/slack-audit-partition-rotation.ts` — monthly job (run 25th) creating next month's partition
- **Files modified**:
  - `src/services/slack/dispatcher/worker.ts` — wire cooldown + burst cap + token-revoked → `needs_reconnect` transition
- **Behind flag**: yes.
- **Reviewers**: 1 backend, 1 SRE (Redis capacity), 1 security (audit redaction review).

### PR 8 — `feat(slack): activity audit + test alert` — SDLCTEST-7

- **Components**: `slack/audit`, `slack/test-alert` (design §3.1)
- **Files added**:
  - `src/services/slack/audit/handlers.ts` — `GET /v1/slack/activity` (cursor pagination, 30-day window)
  - `src/services/slack/audit/repo.ts`
  - `src/services/slack/test-alert/handlers.ts` — `POST /v1/slack/test-alert`, admin-only, bypasses cooldown, throttle 5/hour/admin
- **Files modified**:
  - `src/index.ts` — register audit + test-alert plugins
- **Behind flag**: yes; `/v1/slack/activity` returns 404 when flag off.
- **Reviewers**: 1 backend, 1 frontend (UI-coupled — Next.js repo PR is paired but not in this repo).

### PR 9 — `chore(slack): pilot LD rollout + ops links`

- **Files added**:
  - `ops/launchdarkly/slack-alerts.enabled.json` — flag definition (per-tenant boolean, default false)
  - `docs/runbooks/slack-alerts.md` — stub for stage-7
- **Files modified**:
  - `docs/architecture.md` — append ADR-0004 line under "Existing ADRs"
  - `docs/sdlc/churn-slack-alerts/02-design.md` — already corrected in PR 1; this PR adds a back-reference
- **Reviewers**: PM, SRE.
- **Note**: this PR doesn't toggle the flag *on*; it provisions the flag so SRE can flip the 2 pilot tenants per the rollout plan.

## 3. Branch & commit conventions

The single commit on `main` is:

```
feat: initial Aurora project — sample for sevaai-sdlc validation

Includes:
- README, docs/architecture.md, package.json (Node/TS/Fastify/Postgres/BullMQ/AWS)
- .sevaai-sdlc.yaml with project context + tracker config (Jira project SDLCTEST)
- ...
```

So the convention is observable: lowercase `<type>` + colon, en/em-dash separator, body bullet list. We extend it with **scopes** (which the seed commit lacked; add via commitlint in PR 1).

### Branches

```
feat/SDLCTEST-<N>-<short-slug>     # one branch per PR, one Jira story per PR (mostly)
chore/SDLCTEST-1-<scope>           # umbrella-tracked work (PR 1, PR 9)
```

Examples:
- `feat/SDLCTEST-2-slack-oauth`
- `feat/SDLCTEST-4-crossing-detector`
- `chore/SDLCTEST-1-ts-eslint-ci`

Trunk-based: branches < 5 days old (per `README.md` engineering norms). Rebase, don't merge-commit.

### Commits

```
<type>(<scope>): <subject ≤ 70 chars>

<body — wrap at 80, bullet list, present tense>

Refs: SDLCTEST-<N>
```

- `type` ∈ {`feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`, `build`, `ci`}
- `scope` ∈ {`slack`, `churn`, `score`, `infra`, `ui`, `deps`} (commitlint enforces)
- `Refs: SDLCTEST-<N>` line is **required** on the PR's merge commit so Jira's GitHub integration can backlink. PR 1's commitlint hook checks this on the squash subject.

### PR size

Hard cap: 400 net lines (matches `README.md`). PR 6 is the closest to the cap and should be split rather than waived.

## 4. PR template

Lives at `.github/PULL_REQUEST_TEMPLATE.md` (added in PR 1).

```markdown
## Summary

<!-- 1-2 sentences; the "why" goes here, not the "what". -->

## Component(s) touched

<!-- From docs/sdlc/churn-slack-alerts/02-design.md §3.1. Pick one or more: -->
- [ ] `slack/oauth`
- [ ] `slack/routing`
- [ ] `slack/audit`
- [ ] `slack/test-alert`
- [ ] `slack/dispatcher`
- [ ] `churn/crossing-detector`
- [ ] `score` (shared schema)
- [ ] infra / CI / docs

## Tracker

- Jira: SDLCTEST-<N>
- Design: docs/sdlc/churn-slack-alerts/02-design.md §<section>
- ADR (if any): ADR-NNNN

## Behind a feature flag?

- [ ] Yes — flag: `slack-alerts.enabled` (default off)
- [ ] No — explain why this is safe to merge unflagged:

## Migrations

- [ ] No migration in this PR
- [ ] Adds `migrations/<ts>_<name>.up.sql` AND matching `.down.sql`
- [ ] Migration is idempotent / reversible (forward + rollback verified locally)

## Self-review

- [ ] All items in §6 of `03-development.md` checked
- [ ] No secrets, tokens, or PII in code, fixtures, or commit messages
- [ ] `npm run lint && npm run build && npm test` clean locally

## Reviewers

<!-- CODEOWNERS auto-assigns. Tag security if any of: auth, secrets, KMS, payload. -->
```

## 5. New CI checks

PR 1 adds `.github/workflows/ci.yaml` (Node 20, runs on `pull_request` and `push: main`):

| Job | What it does | Failing example |
|---|---|---|
| `lint` | `npm run lint` (ESLint + Prettier check) | unused import |
| `build` | `npm run build` (`tsc --noEmit -p .`) | type error |
| `test` | `npm test -- --coverage` with `--coverageThreshold` 80% on changed files | coverage drop |
| `commitlint` | lints PR title against `commitlint.config.cjs` | `Add slack stuff` (no type) |
| `pr-size` | Fails if net diff > 400 lines (warning only on PRs labeled `infra`) | a 600-line dispatcher PR |
| `secret-scan` | GitHub native + GitGuardian app | a leaked `xoxb-` token |

Plus a dedicated workflow `.github/workflows/migration-parity.yaml`:

```yaml
name: migration parity
on: [pull_request]
jobs:
  parity:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: every up.sql has a matching down.sql
        run: |
          set -euo pipefail
          fail=0
          for up in migrations/*.up.sql; do
            down="${up%.up.sql}.down.sql"
            if [ ! -f "$down" ]; then
              echo "::error file=$up::missing matching $down"
              fail=1
            fi
          done
          for down in migrations/*.down.sql; do
            up="${down%.down.sql}.up.sql"
            if [ ! -f "$up" ]; then
              echo "::error file=$down::orphan down without matching up"
              fail=1
            fi
          done
          exit $fail
      - name: forbid edits to merged migrations
        run: |
          # any migration whose timestamp is older than 7 days must not be modified
          git fetch origin main
          changed=$(git diff --name-only origin/main...HEAD -- 'migrations/*.sql' || true)
          for f in $changed; do
            ts=$(basename "$f" | cut -d_ -f1)
            now_ms=$(date +%s)000
            age_days=$(( (now_ms - ts) / 86400000 ))
            if [ "$age_days" -gt 7 ]; then
              echo "::error file=$f::editing a migration older than 7 days is not allowed; add a new migration"
              exit 1
            fi
          done
```

Two **feature-specific** CI checks (added in PR 6 and PR 8):

| Job | When it runs | What it does |
|---|---|---|
| `slack-driver-allowlist` | any PR touching `src/services/slack/config/slack-driver-allowlist.json` | requires sign-off label `security-reviewed` (CODEOWNERS routes to `@security`); fails otherwise |
| `audit-payload-redaction` | any PR touching `src/services/slack/audit/**` | greps the diff for forbidden tokens (`channel_name`, `user_id`, `user_name` from Slack); fails on match |

## 6. Self-review checklist

Before requesting human review:

- [ ] PR title matches `<type>(<scope>): <subject>` and `Refs: SDLCTEST-<N>` is in the body
- [ ] Touched component(s) ticked in PR template; design section cited
- [ ] `npm run lint && npm run build && npm test` clean locally
- [ ] Coverage on changed lines ≥ 80% (Jest `--coverage` summary)
- [ ] No `console.log`, no commented-out code, no TODOs without a Jira link
- [ ] Every public function exported from `src/services/slack/**` has a one-line JSDoc explaining *why*, not *what*
- [ ] Migrations: matched `.up.sql` + `.down.sql`; ran both against a local Postgres 16 container
- [ ] Feature flag default is **off** in `ops/launchdarkly/slack-alerts.enabled.json`
- [ ] No secrets in fixtures (`tests/fixtures/**` greps clean for `xoxb-`, `xoxp-`, `secret`, `password`)
- [ ] If this PR touches `slack/oauth` or `slack/dispatcher`: pino redaction includes `access_token`, `bot_token`, `ciphertext_*`, `encrypted_dek`
- [ ] If this PR touches the payload builder: payload built only from allowlisted fields; added a unit test that snapshots the output for an account with PII-bearing fields and asserts they're absent
- [ ] CHANGELOG entry added under `## [Unreleased]`

## 7. Coding-agent prompt

Paste this into Cursor / Claude Code / Copilot Workspace at the start of each PR (replace the `{N}` placeholder):

> Implement PR {N} from `docs/sdlc/churn-slack-alerts/03-development.md`. The design is in `docs/sdlc/churn-slack-alerts/02-design.md` — treat §3 (LLD) as the source of truth for module structure, API contracts, DDL, and the sequence diagrams. Match the existing repo conventions: ESM TypeScript with `"type": "module"`, Fastify 4 plugins, zod for runtime validation, pino for logging (with redaction of token fields), pg + node-pg-migrate for SQL, ioredis + bullmq for queues. Follow `.eslintrc.cjs`, `.prettierrc.json`, and `tsconfig.json`. Do **not** write tests in this stage — testing comes from `04-testing.md`. Wrap any user-facing code path in `LD.variation('slack-alerts.enabled', { tenantId })` and bias the failure mode toward "feature off" on flag-eval errors. When you encounter a decision the design didn't pin down, stop and ask — do not invent.

## 8. Tracker

This artifact is the developer brief for the existing Stage-1 Jira stories. **No new tickets are created in this stage.** The implementation tasks below map 1:1 to the existing stories; sub-tasks (if needed) will be created by the implementing engineer at PR-open time and linked back from the PR body.

- **Workstream**: [SDLCTEST-1](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-1) — Churn-Risk Slack Alerts
- **Story → PR mapping**:

| Jira | Story | PR(s) |
|---|---|---|
| [SDLCTEST-2](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-2) | Connect a Slack workspace to a tenant | PR 3 |
| [SDLCTEST-3](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-3) | Configure alert routing per tenant | PR 4 |
| [SDLCTEST-4](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-4) | Detect upward 0.8 crossing and enqueue an alert | PR 2 (prereq), PR 5 |
| [SDLCTEST-5](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-5) | Deliver the alert to Slack with the right payload | PR 6 |
| [SDLCTEST-6](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-6) | Dedup / cooldown so a single account can't spam | PR 7 |
| [SDLCTEST-7](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-7) | Tenant admin can audit and replay Slack alert activity | PR 8 |

PRs 1 and 9 are infra/rollout chores tracked under the SDLCTEST-1 workstream itself, not against any story.

## 9. Hand-off

Next stage: **Testing** (`sdlc-testing`). Artifact: `04-testing.md`.

Inputs the test plan should consume:
- The 9-PR breakdown — each PR is the unit of test gating.
- The §5 CI checks — testing stage will add the `test` job's coverage matrix.
- The two feature-specific CI checks (`slack-driver-allowlist`, `audit-payload-redaction`) — testing stage owns the actual assertions behind these.
- Suppression-reason taxonomy from `02-design.md` §3.4.2 — every reason needs a covered test path.
