# Design — Churn-Risk Slack Alerts

**Feature slug:** `churn-slack-alerts`
**Status:** Draft
**Date:** 2026-05-05
**Supersedes:** None (first ADR for this feature; references ADR-0001 *Schema-per-tenant data isolation*)

## 1. Context

Per `01-requirements.md`, Aurora's daily churn-risk recompute today writes to a `score` table that CSMs only see when they next log in. We will push the *upward* crossing of `score >= 0.8` into a tenant-controlled Slack workspace, with a 24h per-account cooldown, audit trail, and per-tenant burst cap. Threshold and direction are fixed for v1; configurability is out of scope.

This design resolves the open questions from §6 of the requirements doc:

| # | Question | Resolution |
|---|---|---|
| 1 | Crossing baseline | Immediately prior non-backfill `score` row for the same `(tenant_id, account_id)`. No "last N days" window. |
| 2 | Token storage | Encrypted in **the tenant schema**, using KMS envelope encryption with a tenant-scoped data key. See ADR-0002. |
| 3 | Slack rate limits / burst | Per-tenant token-bucket in Redis (default 50 enqueues/hour). Excess dropped with audit reason `tenant_burst_cap`. |
| 5 | LaunchDarkly flag scope | Per-tenant boolean `slack-alerts.enabled`. No percentage rollout. |
| 6 | Score-driver allowlist | Static allowlist in `config/slack-driver-allowlist.json`, code-reviewed; new driver fields default to *excluded*. |
| 7 | "Test alert" | In scope for v1. Tenant-admin-only endpoint, bypasses cooldown, payload prefixed `[TEST]`. |

Question #4 (DPA / sub-processor) is a legal blocker tracked separately; design assumes "yes, we are a sub-processor" and bakes in revocability.

## 2. High-Level Design (HLD)

### 2.1 Component diagram

```mermaid
flowchart LR
  subgraph Client
    NextUI[Next.js Settings UI]
  end

  subgraph Aurora API [Fastify 4 — ECS Fargate]
    OAuthH[slack/oauth handlers]
    RoutingH[slack/routing handlers]
    AuditH[slack/audit handlers]
    TestH[slack/test-alert handler]
  end

  subgraph Aurora Worker [BullMQ — ECS Fargate]
    Detector[churn/crossing-detector]
    Dispatcher[slack/dispatcher]
  end

  subgraph Data
    PG[(Postgres 16<br/>tenant schemas)]
    Redis[(Redis 7<br/>cooldown + bucket)]
    KMS[AWS KMS]
    SM[AWS Secrets Manager<br/>SLACK_CLIENT_*]
  end

  subgraph External
    Slack[Slack API]
    LD[LaunchDarkly]
    DD[Datadog APM/Logs]
  end

  NextUI -->|JWT| OAuthH
  NextUI -->|JWT| RoutingH
  NextUI -->|JWT| AuditH
  NextUI -->|JWT| TestH

  OAuthH -->|exchange code| Slack
  OAuthH -->|encrypt token| KMS
  OAuthH --> PG
  RoutingH --> PG
  AuditH --> PG
  TestH -->|enqueue test job| Redis

  Detector -->|read score rows| PG
  Detector -->|enqueue alert job| Redis

  Dispatcher -->|consume| Redis
  Dispatcher -->|cooldown check| Redis
  Dispatcher -->|burst-cap check| Redis
  Dispatcher -->|decrypt token| KMS
  Dispatcher -->|read connection + routing| PG
  Dispatcher -->|chat.postMessage| Slack
  Dispatcher -->|write audit row| PG

  OAuthH -.flag check.-> LD
  Dispatcher -.flag check.-> LD
  Aurora API -.metrics/logs.-> DD
  Aurora Worker -.metrics/logs.-> DD
  OAuthH -.fetch creds.-> SM
```

### 2.2 Service responsibilities

| Component | Responsibility | Owns |
|---|---|---|
| `slack/oauth` (API) | OAuth v2 start/callback, token encryption, disconnect, reconnect detection | `slack_connection` lifecycle |
| `slack/routing` (API) | CRUD on default + rule-based channel routing | `slack_routing_rule` rows |
| `slack/audit` (API) | Read-only paginated activity feed for tenant admins | Query layer over `slack_alert_audit_*` partitions |
| `slack/test-alert` (API) | Tenant-admin one-shot enqueue with `[TEST]` payload, bypassing cooldown | n/a (uses dispatcher) |
| `churn/crossing-detector` (Worker) | After each `score_run` completes, find rows that crossed `< 0.8 -> >= 0.8` and enqueue exactly one BullMQ job per crossing | Idempotency key `(account_id, score_run_id)` |
| `slack/dispatcher` (Worker) | Consume BullMQ jobs; check flag, cooldown, burst cap, connection state; render payload via allowlist; post to Slack; write audit row | Outbound delivery + retry/backoff |

### 2.3 Tech stack decisions

Aligned with `package.json` and `.sevaai-sdlc.yaml`. No new runtime dependencies.

| Concern | Choice | Why |
|---|---|---|
| HTTP framework | Fastify 4 (existing) | Schema-first via JSON Schema + zod for runtime validation matches existing pattern |
| Validation | zod 3 (existing) | Already in `dependencies`; schemas double as TS types |
| Queue | BullMQ on Redis 7 (existing) | Score-recompute pipeline already uses BullMQ; reuse worker harness |
| OAuth client | Slack OAuth v2 directly via `fetch` | Adding `@slack/oauth` (~ 800 KB transitive) for one endpoint pair is not justified; we already have `pino` redaction and `zod` for the response shape |
| Slack post client | Slack Web API directly via `fetch` | Same rationale; `@slack/web-api` adds 1.2 MB and we use 2 endpoints |
| Crypto | AWS KMS via `@aws-sdk/client-kms` (new) | Envelope encryption per ADR-0002 |
| Flag eval | LaunchDarkly Node SDK (existing in worker) | Per-tenant boolean evaluation |

### 2.4 Integration points

| System | Direction | Protocol | Trust boundary |
|---|---|---|---|
| Slack API (`slack.com/api`) | Outbound | HTTPS REST | External — tenant-controlled workspace |
| AWS KMS | Outbound | AWS SDK (SigV4) | Internal AWS account |
| AWS Secrets Manager | Outbound | AWS SDK (SigV4) | Internal AWS account; holds `SLACK_CLIENT_ID/SECRET` |
| LaunchDarkly | Outbound | HTTPS streaming | External SaaS |
| Datadog | Outbound | DogStatsD + log shipper | External SaaS |
| Postgres / Redis | In-VPC | mTLS (SPIFFE) | Internal |

## 3. Low-Level Design (LLD)

### 3.1 Module / package structure

```
src/services/slack/
├── oauth/
│   ├── handlers.ts          # GET /v1/slack/oauth/start, /callback, POST /disconnect
│   ├── exchange.ts          # POST oauth.v2.access against Slack
│   └── tokens.ts            # encrypt/decrypt via KMS envelope
├── routing/
│   ├── handlers.ts          # CRUD on routing rules + default channel
│   └── repo.ts              # tenant-schema-aware queries
├── audit/
│   ├── handlers.ts          # GET /v1/slack/activity (paginated)
│   └── repo.ts
├── dispatcher/
│   ├── worker.ts            # BullMQ consumer
│   ├── cooldown.ts          # Redis: SET NX with EX=86400 keyed by (tenant,account)
│   ├── burst-cap.ts         # Redis token bucket per tenant
│   ├── payload.ts           # render mrkdwn block from allowlisted drivers
│   └── post.ts              # Slack chat.postMessage, retry on 429/Retry-After
├── test-alert/
│   └── handlers.ts          # POST /v1/slack/test-alert (admin-only)
├── types.ts                 # zod schemas + inferred types
└── config/
    └── slack-driver-allowlist.json

src/services/churn/
└── crossing-detector/
    ├── worker.ts            # subscribed to score_run.completed
    └── detector.ts          # SQL: find crossings, enqueue jobs idempotently

migrations/
├── 1714939200000_score-run-is-backfill.{up,down}.sql
├── 1714939300000_slack-connection.{up,down}.sql
├── 1714939400000_slack-routing-rule.{up,down}.sql
└── 1714939500000_slack-alert-audit-partitioned.{up,down}.sql
```

### 3.2 API contracts

All endpoints are mounted under the existing tenant-scoped router; tenant context is resolved from the `aud` claim of the @fastify/jwt token. Admin-only endpoints additionally check `role IN ('tenant_admin','owner')`.

```yaml
openapi: 3.0.3
info:
  title: Aurora Slack-Alerts API (additive subset)
  version: 0.1.0
servers:
  - url: https://app.aurora.io/api
security:
  - bearerAuth: []

paths:
  /v1/slack/oauth/start:
    get:
      summary: Begin Slack OAuth v2 flow
      description: Admin-only. Returns a 302 to slack.com with a signed state nonce.
      responses:
        '302': { description: redirect to Slack }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '403': { $ref: '#/components/responses/Forbidden' }

  /v1/slack/oauth/callback:
    get:
      summary: Slack OAuth v2 redirect target
      description: Exchanges code, encrypts token via KMS, persists connection.
      parameters:
        - in: query
          name: code
          schema: { type: string }
          required: true
        - in: query
          name: state
          schema: { type: string }
          required: true
      responses:
        '302': { description: redirect to /settings/integrations?slack=connected }
        '400': { $ref: '#/components/responses/BadRequest' }
        '409':
          description: workspace already connected to another tenant in same Slack team
        '502': { description: Slack OAuth exchange failed }

  /v1/slack/disconnect:
    post:
      summary: Revoke and purge the tenant's Slack connection
      description: Admin-only. Calls auth.revoke on Slack, deletes the row in same tx, writes audit event.
      responses:
        '204': { description: disconnected }
        '401': { $ref: '#/components/responses/Unauthorized' }
        '403': { $ref: '#/components/responses/Forbidden' }
        '404': { description: no connection to disconnect }

  /v1/slack/routing/default:
    put:
      summary: Set the default Slack channel for this tenant
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [channel_id]
              properties:
                channel_id: { type: string, pattern: '^[CG][A-Z0-9]{8,}$' }
      responses:
        '200': { description: updated }
        '400': { $ref: '#/components/responses/BadRequest' }
        '422': { description: bot is not a member of channel }

  /v1/slack/routing/rules:
    get:
      summary: List routing rules in evaluation order
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/RoutingRule' }
    post:
      summary: Create a routing rule
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RoutingRuleInput' }
      responses:
        '201': { description: created }

  /v1/slack/routing/rules/{id}:
    delete:
      parameters:
        - in: path
          name: id
          schema: { type: string, format: uuid }
          required: true
      responses:
        '204': { description: deleted }

  /v1/slack/activity:
    get:
      summary: Paginated audit feed (last 30 days)
      parameters:
        - in: query
          name: cursor
          schema: { type: string }
        - in: query
          name: status
          schema: { type: string, enum: [delivered, suppressed, failed] }
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items: { $ref: '#/components/schemas/AuditEntry' }
                  next_cursor: { type: string, nullable: true }
        '403': { $ref: '#/components/responses/Forbidden' }

  /v1/slack/test-alert:
    post:
      summary: Send a [TEST]-prefixed alert through the live pipeline
      description: Bypasses cooldown but obeys flag, connection state, burst cap.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [account_id]
              properties:
                account_id: { type: string, format: uuid }
      responses:
        '202': { description: enqueued }
        '409': { description: tenant not connected }
        '429': { description: burst cap exceeded }

components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  schemas:
    RoutingRule:
      type: object
      properties:
        id:           { type: string, format: uuid }
        owner_team:   { type: string }
        channel_id:   { type: string }
        priority:     { type: integer }
    RoutingRuleInput:
      type: object
      required: [owner_team, channel_id]
      properties:
        owner_team:   { type: string, maxLength: 128 }
        channel_id:   { type: string, pattern: '^[CG][A-Z0-9]{8,}$' }
        priority:     { type: integer, minimum: 0, maximum: 1000 }
    AuditEntry:
      type: object
      properties:
        ts:           { type: string, format: date-time }
        account_id:   { type: string, format: uuid }
        status:       { type: string, enum: [delivered, suppressed, failed] }
        reason_code:  { type: string, nullable: true }
        channel_id:   { type: string, nullable: true }
        slack_message_ts: { type: string, nullable: true }
  responses:
    Unauthorized: { description: missing or invalid JWT }
    Forbidden:    { description: not a tenant admin }
    BadRequest:   { description: validation error }
```

Internal BullMQ job contract:

```ts
// queue: slack.alert
type SlackAlertJob = {
  tenant_id: string;       // uuid
  account_id: string;      // uuid
  new_score: number;       // [0,1]
  prior_score: number | null;
  score_run_id: string;    // uuid — idempotency partition
  is_test: boolean;        // bypass cooldown when true
};
// jobId = `${tenant_id}:${account_id}:${score_run_id}` for idempotency
```

### 3.3 Data model

All new tables live in **the tenant schema** (`tenant_<id>.*`), per ADR-0001. The `score_run` column add is in the shared score schema.

```sql
-- shared schema: flag backfill score_runs so detector can ignore them
ALTER TABLE score_run
  ADD COLUMN is_backfill BOOLEAN NOT NULL DEFAULT FALSE;
CREATE INDEX score_run_not_backfill_idx
  ON score_run (tenant_id, completed_at DESC)
  WHERE is_backfill = FALSE;

-- tenant schema: one row per tenant; tenant_id implicit by schema
CREATE TABLE slack_connection (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id                   TEXT NOT NULL,           -- Slack workspace id
  team_name                 TEXT NOT NULL,
  bot_user_id               TEXT NOT NULL,
  -- envelope encryption: encrypted_dek wraps the per-tenant DEK; ciphertext_token
  -- is AES-256-GCM(dek, plaintext_bot_token); IV stored alongside
  encrypted_dek             BYTEA NOT NULL,           -- KMS-wrapped data key
  ciphertext_token          BYTEA NOT NULL,
  ciphertext_iv             BYTEA NOT NULL,
  ciphertext_tag            BYTEA NOT NULL,
  scopes                    TEXT[] NOT NULL,
  status                    TEXT NOT NULL DEFAULT 'active'
                              CHECK (status IN ('active','needs_reconnect','disconnected')),
  connected_by_user_id      UUID NOT NULL,            -- aurora user id
  connected_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_health_check_at      TIMESTAMPTZ,
  UNIQUE (team_id)                                   -- one Slack workspace per tenant
);

CREATE TABLE slack_routing_rule (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_team    TEXT NOT NULL,             -- account.owner_team value to match
  channel_id    TEXT NOT NULL,             -- Slack channel id
  priority      INTEGER NOT NULL DEFAULT 100,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX slack_routing_rule_owner_team_idx ON slack_routing_rule (owner_team, priority);

-- exactly one row holds the default channel per tenant
CREATE TABLE slack_routing_default (
  tenant_singleton  BOOLEAN PRIMARY KEY DEFAULT TRUE CHECK (tenant_singleton),
  channel_id        TEXT NOT NULL,
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- partitioned by month; partition creation handled by an ops job on the 25th
CREATE TABLE slack_alert_audit (
  id                UUID NOT NULL DEFAULT gen_random_uuid(),
  ts                TIMESTAMPTZ NOT NULL DEFAULT now(),
  account_id        UUID NOT NULL,
  score_run_id      UUID NOT NULL,
  status            TEXT NOT NULL CHECK (status IN ('delivered','suppressed','failed')),
  reason_code       TEXT,                    -- e.g. 'cooldown', 'tenant_burst_cap', 'token_revoked'
  channel_id        TEXT,
  slack_message_ts  TEXT,
  PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

CREATE INDEX slack_alert_audit_account_ts_idx ON slack_alert_audit (account_id, ts DESC);
CREATE INDEX slack_alert_audit_score_run_idx ON slack_alert_audit (score_run_id);
```

Cooldown + burst-cap are **Redis-only** (not Postgres) — they are ephemeral and the cost of "lost on Redis flush" is one extra alert per account, which is acceptable.

```
# 24h cooldown after a delivered alert
slack:cooldown:{tenant_id}:{account_id}   value=1   EX=86400   NX

# token bucket: 50 enqueues / 60-min sliding window
slack:burst:{tenant_id}                   sorted-set of timestamps
```

#### Migration plan

Forward order:
1. `score-run-is-backfill` — add column with default `FALSE`. Safe online (Postgres 16 fast-path; no rewrite).
2. Per-tenant: `slack-connection`, `slack-routing-rule` + `slack-routing-default`, `slack-alert-audit-partitioned` (creates parent + first month + next month partitions).

Rollback order (reverse):
1. Drop the four new tenant-schema tables. Bot tokens are wiped; OAuth must be re-run if rolling forward again. No customer data loss because audit is purely a derivation.
2. `ALTER TABLE score_run DROP COLUMN is_backfill;` — safe; no consumer outside this feature.

LaunchDarkly flag controls *enqueue*, so disabling it is the operational kill switch — migrations don't need to be reverted to stop the feature.

### 3.4 Sequence diagrams

#### 3.4.1 Connect Slack workspace

```mermaid
sequenceDiagram
  actor Tara as Tenant Admin
  participant UI as Next.js UI
  participant API as Aurora API
  participant SM as Secrets Manager
  participant Slack
  participant KMS
  participant PG as Postgres (tenant schema)

  Tara->>UI: Click "Connect Slack"
  UI->>API: GET /v1/slack/oauth/start (JWT)
  API->>API: verify role=tenant_admin; mint signed state
  API-->>UI: 302 slack.com/oauth/v2/authorize?...&state=...
  UI->>Slack: redirect
  Tara->>Slack: approve scopes
  Slack-->>UI: 302 .../callback?code=...&state=...
  UI->>API: GET /v1/slack/oauth/callback?code&state
  API->>API: verify state HMAC + nonce TTL
  API->>SM: get SLACK_CLIENT_ID, SLACK_CLIENT_SECRET
  API->>Slack: POST oauth.v2.access (code, client creds)
  Slack-->>API: { access_token, team, bot_user_id, scope }
  API->>KMS: GenerateDataKey (CMK=aurora/slack-tokens)
  KMS-->>API: { plaintext_dek, encrypted_dek }
  API->>API: AES-256-GCM(plaintext_dek, access_token) + zero plaintext_dek
  API->>PG: INSERT slack_connection (...)
  PG-->>API: ok
  API-->>UI: 302 /settings/integrations?slack=connected
```

#### 3.4.2 Crossing detection → Slack delivery

```mermaid
sequenceDiagram
  participant Score as Score Recompute Worker
  participant PG as Postgres
  participant Detector as crossing-detector
  participant Q as BullMQ
  participant Disp as slack/dispatcher
  participant LD as LaunchDarkly
  participant R as Redis
  participant KMS
  participant Slack

  Score->>PG: INSERT score rows for run_id=R; UPDATE score_run.completed_at
  Score->>Q: enqueue churn.score_run.completed { tenant_id, run_id, is_backfill }
  Detector->>Q: consume
  alt is_backfill = true
    Detector-->>Q: ack (no-op)
  else is_backfill = false
    Detector->>PG: SELECT crossings (new>=0.8 AND prior<0.8) WHERE run_id=R
    loop per crossing
      Detector->>Q: add slack.alert { jobId = tenant:account:run_id }
      Note over Q: jobId collision = idempotent skip
    end
  end

  Disp->>Q: consume slack.alert
  Disp->>LD: variation('slack-alerts.enabled', tenant)
  alt flag off
    Disp->>PG: audit (suppressed, reason=flag_off)
  else flag on
    Disp->>R: SET NX slack:cooldown:{tenant}:{account} EX 86400
    alt key already existed AND not is_test
      Disp->>PG: audit (suppressed, reason=cooldown)
    else key set
      Disp->>R: ZADD slack:burst:{tenant}; ZREMRANGEBYSCORE > now-3600
      alt count > 50
        Disp->>PG: audit (suppressed, reason=tenant_burst_cap)
      else within cap
        Disp->>PG: SELECT slack_connection, default channel, matching rule
        alt status != active
          Disp->>PG: audit (failed, reason=needs_reconnect)
        else active
          Disp->>KMS: Decrypt(encrypted_dek)
          KMS-->>Disp: plaintext_dek
          Disp->>Disp: AES-GCM decrypt token; render payload via allowlist
          Disp->>Slack: chat.postMessage
          alt 200 ok
            Slack-->>Disp: { ok:true, ts }
            Disp->>PG: audit (delivered, slack_message_ts=ts)
          else 429
            Slack-->>Disp: Retry-After: N
            Disp->>Q: backoff retry (≤ 5)
          else token_revoked / account_inactive
            Disp->>PG: UPDATE slack_connection SET status='needs_reconnect'
            Disp->>PG: audit (failed, reason=<code>)
          end
        end
      end
    end
  end
```

### 3.5 Configuration / environment variables

| Variable | Purpose | Default | Source |
|---|---|---|---|
| `SLACK_CLIENT_ID` | Slack app client id | n/a | Secrets Manager |
| `SLACK_CLIENT_SECRET` | Slack app client secret | n/a | Secrets Manager |
| `SLACK_OAUTH_REDIRECT_URL` | OAuth callback URL | per-env | Env |
| `SLACK_STATE_HMAC_KEY` | HMAC key for OAuth `state` nonce | n/a | Secrets Manager |
| `SLACK_KMS_CMK_ID` | KMS CMK alias for token envelope encryption | `alias/aurora/slack-tokens` | Env |
| `SLACK_BURST_CAP_PER_HOUR` | Per-tenant enqueue cap | `50` | Env |
| `SLACK_COOLDOWN_SECONDS` | Per-account cooldown window | `86400` | Env |
| `SLACK_DRIVER_ALLOWLIST_PATH` | Path to `slack-driver-allowlist.json` | `./config/slack-driver-allowlist.json` | Env |
| `LD_SDK_KEY` | LaunchDarkly SDK key (existing) | n/a | Secrets Manager |
| `SLACK_DISPATCHER_CONCURRENCY` | BullMQ worker concurrency | `8` | Env |

## 4. ADR-0002 — Slack OAuth token storage

- **Status**: Proposed
- **Context**: Story 1 stores a long-lived Slack bot token per tenant. Two approaches were live: (a) plaintext-or-pgcrypto in the tenant schema (per ADR-0001 isolation), or (b) central encrypted vault (e.g., HashiCorp Vault transit, or AWS Secrets Manager per-tenant secret). Requirements risk #2 explicitly leaves this open.
- **Decision**: Store the ciphertext **in the tenant schema**, encrypted with **AES-256-GCM under a per-tenant data encryption key (DEK)**. The DEK is generated via `KMS:GenerateDataKey` against a single Aurora-owned CMK (`alias/aurora/slack-tokens`); only the wrapped DEK and ciphertext are persisted. Plaintext DEK is held in process memory only for the duration of one decrypt call and zeroed after.
- **Alternatives considered**:
  1. **Plaintext + pgcrypto symmetric column encryption** — rejected: shared symmetric key across tenants violates blast-radius goal; key rotation requires rewriting every row; SOC 2 auditors prefer KMS-backed keys.
  2. **AWS Secrets Manager, one secret per tenant** — rejected: ~10× cost at our tenant projection (Secrets Manager bills per secret), no transactional consistency with the `slack_connection` row (token without row, or vice versa, on partial failure), and cross-region replication complexity.
  3. **HashiCorp Vault transit engine** — rejected: introduces a new operational dependency we don't already run; the only feature it gives us over KMS envelope is automatic rotation, which we get via DEK regeneration on reconnect anyway.
  4. **Central `secrets.slack_token` table outside tenant schemas** — rejected: contradicts ADR-0001 (tenant data must be schema-isolated). A breach of one row should not cross-leak.
- **Consequences**:
  - **Positive**: Per-tenant blast radius. KMS audit trail (CloudTrail) per decrypt. Cheap (KMS data-key generation + AES is microseconds). Rotation = "disconnect + reconnect" — surface that in the UI as a yearly nudge.
  - **Negative**: Each Slack post requires one KMS `Decrypt` call (~5 ms p50). Mitigation: in-process LRU of plaintext DEKs with 60s TTL keyed by `slack_connection.id`, scoped to the dispatcher process; at our throughput (< 100 posts/min/region) this caps KMS spend trivially.
  - **Neutral**: New env var `SLACK_KMS_CMK_ID`; new IAM policy on the worker task role for `kms:Decrypt` on that CMK only; new policy on the API task role for `kms:GenerateDataKey`.
- **Supersedes**: None. Compatible with ADR-0001.

## 5. First-pass threat model (STRIDE)

| Component | Spoofing | Tampering | Repudiation | Info disclosure | DoS | Elevation |
|---|---|---|---|---|---|---|
| `slack/oauth` API | OAuth `state` HMAC + TTL prevents CSRF/replay. JWT auth on entry. | TLS in transit; ciphertext never leaves API to logs (pino redaction on `access_token`). | All connect/disconnect events written to `slack_alert_audit` + Aurora app audit log. | KMS envelope; never log the token; UI never displays it. | Burst cap on OAuth start (rate-limit on signed state issuance, 5/min/admin). | Role check `tenant_admin` on `/start`, `/disconnect`, `/test-alert`. |
| `slack/routing` API | JWT + admin role. | zod-validated `channel_id` regex blocks injection. | Mutation events audited (Aurora app audit, not Slack audit). | Channel IDs are non-secret; safe to log. | Pagination caps `channels.list` proxy at 200/page. | Same role gate as oauth. |
| `slack/audit` API | JWT + admin role. | Read-only. | n/a | Reason codes are enums, not free text — no Slack user IDs / message bodies stored. | Cursor pagination; no offset scans. | Role gate; cross-tenant access impossible because tenant schema is selected from JWT. |
| `slack/test-alert` API | JWT + admin role. | n/a (just enqueues). | Audited as `is_test=true`. | Same allowlist as production payload. | Burst cap applies; per-admin throttle (5/hour). | Role gate. |
| `crossing-detector` worker | Internal service; SPIFFE mTLS. | SQL is parameterised; idempotency key prevents replay-amplification. | Detector writes `slack_alert_audit` with `score_run_id`. | No PII in score rows; backfill flag prevents historical replay. | One job per crossing per run; jobId-keyed dedup in BullMQ. | Worker runs as a least-privileged IAM role (read score, enqueue only). |
| `slack/dispatcher` worker | SPIFFE mTLS internally; Slack TLS externally. | Outbound payload constructed only from allowlisted fields; allowlist diff requires code review. | Every outcome (delivered/suppressed/failed) writes audit row with reason code. | Token plaintext lives ≤ one post; LRU has 60s TTL; pino redaction; payload allowlist enforces no end-user PII. | BullMQ concurrency cap; per-tenant burst cap; Slack 429 honoured with Retry-After. | `kms:Decrypt` scoped to one CMK; cannot read other secrets. |
| Postgres (tenant schema) | mTLS to RDS. | Schema-per-tenant + ADR-0001 boundaries. | Row-level audit on `slack_alert_audit`. | Connection rows hold ciphertext only. | RDS connection pool sized; partitioned audit table prevents bloat. | DB role for worker has `SELECT/INSERT` on these tables only. |
| Redis | TLS + AUTH. | Cooldown/bucket are derivable; loss = at-most one duplicate alert. | n/a (ephemeral). | No tokens or PII in Redis. | Memory bounded by sliding-window TTLs. | Worker role can `SETEX/ZADD` on `slack:*` keys only. |
| External: Slack API | n/a (vendor). | n/a | Slack-side audit log is the tenant's. | Outbound payload reviewed at allowlist boundary. | We respect their rate limits; they enforce theirs. | n/a |
| External: KMS | IAM SigV4. | n/a | CloudTrail records every `Decrypt`/`GenerateDataKey`. | n/a | KMS quotas — well below our < 100/min projection. | CMK key policy restricts to API + dispatcher task roles. |

Full STRIDE + IriusRisk entry happens in stage 5 (`sdlc-security`); this table is the design-stage flag list.

## 6. Open questions

- **Q-A**: Should the LRU plaintext-DEK cache TTL be configurable per env? Lean no for v1 (simpler audit story); revisit if KMS spend grows.
- **Q-B**: Audit retention. Requirements say "last 30 days" in UI; do we keep older partitions for compliance? Proposal: detach partitions older than 90 days into cold storage (S3 + Glacier transition). Confirm with DPO during stage 5.
- **Q-C**: Should `slack/dispatcher` emit a Datadog event for every `needs_reconnect` transition (paging-grade) or only metrics (dashboard-grade)? Proposal: metric only; PagerDuty too noisy for tenant-side actions.
- **Q-D**: Slack Enterprise Grid — out of scope for v1, but the `slack_connection.team_id UNIQUE` constraint will need rework when we add Grid org installs. Note in `99-followups.md`.

## 7. Tracker

This design artifact lives only in markdown for the design stage — no new Jira tickets are created. The parent workstream remains:

- **Workstream**: [SDLCTEST-1](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-1) — Churn-Risk Slack Alerts

Stories SDLCTEST-2 … SDLCTEST-7 (created in stage 1) are the implementation units; ADR-0002 will be linked back to SDLCTEST-1 in stage 3 when development tasks are written.

## 8. Hand-off

Next stage: **Development** (`sdlc-development`). Artifact: `03-development.md`.

Inputs the dev brief should consume from this design:
- The four migrations in §3.3 (forward + rollback).
- The OpenAPI subset in §3.2 (drop into Stoplight; the existing aurora-api spec is the host doc).
- Module structure in §3.1 — six new packages, no edits to existing services beyond the `score_run.is_backfill` column.
- ADR-0002 — copy to `docs/adr/0002-slack-oauth-token-storage.md` as part of the first dev PR.
