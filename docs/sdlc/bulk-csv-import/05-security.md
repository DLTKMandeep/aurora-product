# Security Review — Bulk CSV import for accounts

**Feature slug:** `bulk-csv-import`
**Date:** 2026-05-04
**Status:** Draft
**BLOCKING:** no

## 1. STRIDE per component

| Component | Spoofing | Tampering | Repudiation | Info disclosure | DoS | Elevation |
|---|---|---|---|---|---|---|
| **Presign API** | JWT bound to tenant + admin role | request body validated by zod | audit_events row on every presign | URL TTL 5 min; signed only for one bucket prefix | per-tenant rate limit 10/min | tenant-id from JWT, never from body |
| **S3 PUT** | bound by presigned URL | server-side size + MIME re-validated post-upload | S3 access logs to centralized S3 | bucket private (no public ACL); SSE-S3 enabled | size cap 50MB enforced both client + server | n/a |
| **Start API** | JWT auth | s3_key validated against presign-issued prefix | audit_events row | only returns counts to caller's tenant | parallel-job cap 1 per tenant | scope check on tenant_id |
| **Worker** | BullMQ token | tenant_id pulled from job, not from CSV row | audit_events on completed/failed | row content never logged | global concurrency cap 8 | job-scope tenant_id is authoritative |
| **Audit log** | append-only via separate role | INSERT-only role; no UPDATE/DELETE | inherent | restricted read role | n/a | n/a |
| **Outcome CSV** | served via presigned GET | tenant scoping in repo layer | audit_events on download | URL TTL 5 min | n/a | n/a |

## 2. OWASP Top 10 (2021) mapping

| ID | Risk | Applicable? | Mitigation |
|---|---|---|---|
| A01 | Broken Access Control | yes | RBAC: only `admin` role on tenant; tenant_id always from JWT; repo refuses cross-tenant reads (unit-tested) |
| A02 | Cryptographic Failures | yes | TLS 1.3 in transit; SSE-S3 at rest; file_sha256 stored for integrity |
| A03 | Injection | yes | Parameterized SQL (node-pg); CSV parsed never eval'd; zod validates per-row before insert; LF/CRLF stripped before display in UI |
| A04 | Insecure Design | yes | Per-cell byte cap (4KB) prevents zip-bomb-like single-cell attacks; row cap prevents memory exhaustion |
| A05 | Security Misconfiguration | yes | S3 bucket private; lifecycle rules; new IAM role with least privilege (read+write only on `aurora-imports/{tenant}/*`) |
| A06 | Vulnerable & Outdated Components | yes | Snyk + Dependabot scan; papaparse 5.x is current; no transitive deps with known CVEs at design time |
| A07 | Identification & Authentication Failures | yes | Existing Auth0 OIDC; admin role required; session 1h, refresh 30d |
| A08 | Software & Data Integrity Failures | yes | file_sha256 captured; audit_events immutable; signed S3 presigned URLs |
| A09 | Security Logging & Monitoring Failures | yes | audit_events for every job state change; Datadog alert on job-failure rate spike |
| A10 | SSRF | n/a | Worker reads only from project's own S3 bucket; no user-controlled URLs |

ASVS L2 controls cited: V1.5 (input validation), V2.1 (authentication), V4.1 (access control), V5.1 (validation), V7.1 (logging), V12.5 (file upload).

## 3. Auth & authz

| API | Caller | Required role/scope | Tenancy | Audit event |
|---|---|---|---|---|
| `POST /v1/imports/presign` | end-user | `admin` | tenant-scoped | `imports.presign.requested` |
| `POST /v1/imports` | end-user | `admin` | tenant-scoped | `imports.job.started` |
| `GET  /v1/imports/:id` | end-user | `admin` or `viewer` | tenant-scoped | `imports.job.viewed` (sampled) |
| `GET  /v1/imports/:id/progress` | end-user | `admin` or `viewer` | tenant-scoped | none (high-frequency) |

## 4. Data classification

| Field | Classification | At rest | In transit | Retention | PII? |
|---|---|---|---|---|---|
| account.email | Confidential | AES-256 (RDS) | TLS 1.3 | as long as account exists | yes |
| account.name | Confidential | AES-256 (RDS) | TLS 1.3 | as long as account exists | yes |
| account.phone | Restricted | AES-256 + column encryption | TLS 1.3 | as long as account exists | yes |
| import_jobs.file_sha256 | Internal | AES-256 (RDS) | TLS 1.3 | 7 years (SOC 2) | no |
| import_jobs counts/timestamps | Internal | AES-256 (RDS) | TLS 1.3 | 7 years (SOC 2) | no |
| Source CSV in S3 | Confidential | SSE-S3 | TLS 1.3 | 30 days, then auto-delete | yes |
| Errors CSV in S3 | Confidential | SSE-S3 | TLS 1.3 | 90 days, then auto-delete | yes |
| audit_events row | Internal | AES-256 (RDS) | TLS 1.3 | 7 years (SOC 2) | no (only IDs + counts) |

## 5. Dependencies & supply chain

- **New deps**: `papaparse@5.x` (MIT)
- License review: MIT, compatible with Aurora's BSL.
- SBOM update: yes (CI pipeline regenerates on merge).
- Snyk scan: clean at design time; will rescan on PR 3 build.
- No transitive deps with known CVEs.

## 6. Secrets

| Secret | Where stored | Rotation | Owner |
|---|---|---|---|
| S3 access (worker) | IAM task role (no static creds) | n/a (role-based) | platform |
| S3 access (presign) | IAM task role | n/a | platform |
| BullMQ Redis password | Vault path `aurora/{env}/redis/bullmq` | quarterly | platform |
| Postgres credentials | Existing (no change) | quarterly | platform |

No new static secrets.

## 7. GenAI-specific threats

n/a — this feature does not use an LLM. (If future iteration adds AI-driven row repair / suggestion, re-run security with GenAI threats.)

## 8. Compliance map

| Framework | Controls touched | Evidence required |
|---|---|---|
| **SOC 2 Type II** | CC6.1 (access controls), CC6.6 (audit logging), CC7.2 (anomaly detection), CC8.1 (change management) | audit_events rows for every job; deploy ticket; design + security artifacts |
| **GDPR** | Art. 5(1)(c) data minimization, Art. 17 right to erasure, Art. 32 security of processing | Outcome CSV retention 90d; deletion of account triggers cascade to import row references; encryption in transit + at rest documented |
| **HIPAA-readiness FY27** | n/a this iteration | n/a — accounts are not PHI; if PHI ever flows here, re-evaluate |

## 9. Pen test plan

After ship, internal red team should attempt:

1. **Tenant isolation bypass** — try to read tenant B's import via path traversal in `s3_key` or by tampering with JWT `tenant_id` claim
2. **CSV bomb** — upload a 50MB file with one cell containing 50MB of "A" — should be rejected by per-cell cap
3. **Timing attack on dedup** — measure response time difference between create vs skipped to enumerate existing accounts
4. **Replay** — replay a presigned URL after TTL expiry — should 403
5. **CSRF on start endpoint** — confirm JSON-only request bodies + Origin header check
6. **IDOR** — sequentially scan import job IDs across tenants

## 10. Sign-off

| Severity | Count | Mitigated | Accepted with sign-off |
|---|---|---|---|
| Critical | 0 | — | — |
| High | 0 | — | — |
| Medium | 1 (timing-attack on dedup) | partial — constant-time comparison logged as follow-up | — |
| Low | 2 (errors CSV TTL, audit row volume) | accepted as designed | — |

Status: **non-BLOCKING**. Proceed to deployment stage.

## 11. Hand-off

Next stage: **Deployment** (`sdlc-deployment`). Artifact: `06-deployment.md`.
