# Requirements — Churn-Risk Slack Alerts

**Feature slug:** `churn-slack-alerts`
**Status:** Draft
**Author:** sevaai-sdlc / sdlc-requirements skill
**Date:** 2026-05-05

## 1. Summary

Aurora's worker service recomputes a daily churn-risk `score` per `account`. Today, Customer Success teams have to log in to Aurora to discover that an account has crossed into the "high-risk" band. We will deliver real-time Slack notifications to the tenant's chosen Slack workspace whenever an account's churn-risk score crosses the 0.8 threshold (upward), so that CSMs can intervene within hours instead of the next time they happen to open Aurora. Scope is limited to the *upward* crossing event; downward recoveries and configurable thresholds are out of scope for this iteration.

## 1a. Planning

| Sub-activity | Detail |
|---|---|
| **Project scope** | In: per-tenant Slack workspace connection (OAuth), channel-level routing, upward-crossing alert at fixed threshold 0.8, alert payload with account + score + top drivers + deep link, dedup/cooldown, LaunchDarkly gate. Out: configurable thresholds, downward-recovery alerts, MS Teams/email parity, in-app notification center. |
| **Objectives & goals** | (1) Median time-from-score-crossing-to-CSM-acknowledgement < 4 hours (today: ~36h, measured against next-login). (2) >= 60% of paid tenants connect a Slack workspace within 60 days of GA. (3) Zero P1 incidents involving customer data leaked into the wrong Slack workspace. |
| **Resource planning** | 1 backend eng (worker + Slack integration), 0.5 frontend eng (settings UI in Next.js repo), 0.25 design, security review with IriusRisk (auth-touching: Slack OAuth tokens), CSM-Ops as design partner (2 pilot tenants). Blocking dependency: Auth0 migration NOT a blocker — feature must work on email/password fallback too. |
| **Timeline** | M1 — design + threat model signed off (week 2). M2 — Slack OAuth + storage of workspace credentials (week 4). M3 — alert worker job + dedup (week 6). M4 — settings UI + LaunchDarkly rollout to pilot tenants (week 8). M5 — GA behind flag (week 10). |
| **Success metrics** | KPI-1: p50 score-crossing-to-Slack-delivery latency < 90 s. KPI-2: Slack delivery success rate >= 99.5% (excluding tenant-side disconnects). KPI-3: false-positive cooldown rate (alerts suppressed by dedup) < 15% — higher implies score wobble around threshold and should escalate to design. KPI-4: per the OKR above, >= 60% paid-tenant adoption in 60 days. |

## 2. Personas

| Persona | Role | Primary need |
|---|---|---|
| **Casey, CSM** | End user; receives the alert in Slack | Know *which* account, *how risky*, *why*, and *one click* to open it |
| **Tara, Tenant Admin** | Configures Aurora for her org | Connect Slack workspace, choose the channel(s), control who can change the config |
| **Owen, Aurora SRE** | Operates the worker | Confidence that the new outbound integration won't blow Slack rate limits or page on tenant-side disconnects |
| **Dana, DPO (tenant-side)** | Compliance owner at the tenant | Know that customer data going to Slack is documented in the DPA and revocable |

## 3. Business goal

Retention. Aurora's pitch is "intervene before churn" — but the median CSM reaction time is bottlenecked by *discovery*, not by *capability*. Pushing high-risk events into the tool CSMs already live in (Slack) is the highest-leverage retention lever we can ship this half. Secondary goal: differentiation against Gainsight/ChurnZero in the upcoming RFP cycle, both of which already ship Slack alerts.

## 4. User stories

### Story 1: Connect a Slack workspace to a tenant
- **As a** tenant admin (Tara), **I want** to authorize Aurora to post into my company's Slack workspace, **so that** my CSMs receive churn alerts where they already work.
- **Acceptance criteria**
  - Given Tara is a tenant admin and on the Integrations page, when she clicks "Connect Slack", then she is redirected through Slack OAuth v2 with scopes `chat:write`, `channels:read`, `groups:read`, `im:write`.
  - Given OAuth completes successfully, when control returns to Aurora, then the bot token is stored encrypted-at-rest in the tenant's schema (never in the shared schema or worker logs) and the workspace name + team_id are visible on the Integrations page.
  - Given a workspace is already connected, when Tara visits Integrations, then she sees "Connected to {workspace_name}" with a "Disconnect" button; the token is never displayed.
  - Given Tara clicks "Disconnect", when she confirms, then the token is purged from storage within the same DB transaction and a `slack.disconnected` audit event is written.
  - Given Slack returns an OAuth error or the user cancels, when control returns to Aurora, then a non-PII error message is shown and no partial credentials are persisted.
- **Edge cases**: user revokes the Aurora app from Slack admin side (must be detected on next post and surfaced as "Reconnect needed"); same Slack workspace connected by two tenants (allowed — workspace-to-tenant is many-to-one from Slack's side); tenant admin loses admin role mid-flow (existing connection stays; new connection blocked).
- **Dependencies**: Slack app registered in Aurora's Slack-developer org with prod + staging redirect URIs; secrets (`SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`) provisioned through existing AWS Secrets Manager + GitGuardian rules.
- **Estimate**: M

### Story 2: Configure alert routing per tenant
- **As a** tenant admin (Tara), **I want** to choose which Slack channel (or channels) churn alerts post to, **so that** the right CSM pod sees alerts for accounts they own.
- **Acceptance criteria**
  - Given a connected workspace, when Tara opens Routing, then she can pick a default channel from the list of channels the bot is a member of.
  - Given Tara wants per-CSM-pod routing, when she adds a routing rule, then she can map an `account.owner_team` value to a specific channel; rules evaluate in order; default channel is the fallback.
  - Given a routing rule references a channel the bot has been removed from, when an alert fires, then Aurora retries via the default channel and surfaces a "channel unreachable" warning on the Routing page.
  - Given no default channel is set, when an alert would fire, then the alert is *not* sent (silent drop is unacceptable — instead emit a `slack.misconfigured` audit event so the admin can see it).
- **Edge cases**: private channel where the bot isn't invited (we cannot list it; documented limitation); channel renamed in Slack (we store `channel_id`, so renames are transparent); tenant has 200+ channels (paginate, don't fetch all).
- **Dependencies**: Story 1.
- **Estimate**: M

### Story 3: Detect upward 0.8 crossing and enqueue an alert
- **As an** Aurora worker, **I want** to detect when an account's daily churn-risk score crosses from `< 0.8` to `>= 0.8`, **so that** exactly one alert is enqueued per upward crossing.
- **Acceptance criteria**
  - Given the daily `score` recompute job writes a new row for `account A` with `score >= 0.8`, when the previous score for the same account was `< 0.8` (or did not exist), then a `slack.alert.enqueued` job is added to BullMQ with `{tenant_id, account_id, new_score, prior_score, score_run_id}`.
  - Given the new score is `>= 0.8` and the previous score was also `>= 0.8`, when the recompute runs, then *no* job is enqueued (no spam on consecutive high days).
  - Given a score backfill job re-writes historical rows, when those rows are written, then the crossing detector does NOT enqueue alerts for backfilled days (gated by a `is_backfill` flag on the score-run).
  - Given the recompute fails partway through a tenant, when it is retried, then idempotency is preserved by `(account_id, score_run_id)` so the same crossing cannot enqueue twice.
- **Edge cases**: score crosses up, then back below, then up again on the same day due to a manual recompute — only the *first* upward crossing per `score_run_id` counts; account paused/archived (skip); tenant on free tier without Slack connected (still detect, but next stage drops with reason "tenant_not_connected").
- **Dependencies**: existing `score` table; addition of `score_run.is_backfill` boolean (migration).
- **Estimate**: M

### Story 4: Deliver the alert to Slack with the right payload
- **As a** CSM (Casey), **I want** the Slack message to tell me *which* account, *how risky*, *why*, and to give me a one-click link, **so that** I can triage in under a minute.
- **Acceptance criteria**
  - Given a `slack.alert.enqueued` job, when the worker processes it, then it posts a Slack `chat.postMessage` with: account name, current score (e.g., "0.84"), prior score, top 3 score drivers from the existing model output, owning CSM (if known), and a deep link to `https://app.aurora.io/t/{tenant_slug}/accounts/{account_id}`.
  - Given the post succeeds, when Slack returns `ok: true`, then a `slack.alert.delivered` audit row is written with `slack_message_ts`.
  - Given Slack returns a 429 / `ratelimited`, when the worker handles it, then the job is retried with BullMQ exponential backoff respecting the `Retry-After` header, up to 5 attempts before dead-lettering.
  - Given Slack returns `account_inactive` or `token_revoked`, when the worker handles it, then the tenant's connection is marked `needs_reconnect` and *no further alerts* are attempted until reconnect.
  - Given the message payload, when constructed, then it MUST NOT contain end-user PII from the account (no contact emails, names of individual end-users, or ticket bodies) — only the account-level identifiers and score data.
- **Edge cases**: account name contains markdown-special characters (escape per Slack `mrkdwn`); score driver text exceeds Slack's 3000-char block limit (truncate); deep link points to a tenant slug that has been changed (resolve by `tenant_id` server-side).
- **Dependencies**: Stories 1, 2, 3; existing score-driver output from the model.
- **Estimate**: M

### Story 5: Dedup / cooldown so a single account can't spam
- **As a** CSM (Casey), **I want** the same account to not re-alert me repeatedly in a short window, **so that** Slack stays useful instead of becoming noise.
- **Acceptance criteria**
  - Given an alert was delivered for `account A` at time T, when another upward crossing occurs for `account A` within 24 hours, then the new alert is suppressed and a `slack.alert.suppressed` audit row is written with reason `cooldown`.
  - Given the cooldown window is configured (default 24h, system-wide for v1), when the window elapses, then the next genuine upward crossing alerts normally.
  - Given the score for `account A` oscillates around 0.8 within a single day (e.g., 0.79 -> 0.81 -> 0.78 -> 0.82), when the recompute generates each row, then *only the first* upward crossing in the cooldown window alerts.
- **Edge cases**: tenant disconnects then reconnects within the cooldown — keep the cooldown (don't reset on reconnect, otherwise reconnect becomes a spam loophole); admin manually triggers a "test alert" — bypasses cooldown but is clearly labeled `[TEST]`.
- **Dependencies**: Story 4; new table or Redis key namespace for cooldown state.
- **Estimate**: S

### Story 6: Tenant admin can audit and replay
- **As a** tenant admin (Tara), **I want** to see which alerts were delivered, suppressed, or failed in the last 30 days, **so that** I can debug "why didn't I get an alert for X?" without filing a ticket.
- **Acceptance criteria**
  - Given audit rows exist, when Tara opens Integrations -> Slack -> Activity, then she sees a paginated list of `delivered / suppressed / failed` events with timestamp, account, channel, and (for failures) a redacted reason code.
  - Given a failed delivery in `needs_reconnect` state, when Tara clicks "Reconnect", then she is taken through OAuth Story 1 again and on success, the previous failed alerts are NOT auto-replayed (would be stale).
  - Given Tara is NOT a tenant admin, when she navigates to the Activity page, then she gets a 403.
- **Edge cases**: audit table grows large per tenant (partition by month, mirror existing `event` partitioning convention); failure reasons that include user-identifying Slack data (channel ID is fine; user IDs must not be surfaced — store the code, not the body).
- **Dependencies**: Stories 1, 4, 5; new `slack_audit` table in tenant schema.
- **Estimate**: M

## 5. Out of scope

- Configurable thresholds (per-tenant or per-account "alert me at 0.7"). Hardcoded 0.8 for v1.
- Downward-recovery alerts ("account A is no longer at risk").
- Microsoft Teams, email, webhooks, or in-app notification center as alternative destinations.
- Two-way interaction (acknowledge / snooze from Slack via interactive components).
- Per-user DM routing based on Aurora<->Slack user mapping.
- Bulk historical replay of alerts that *would have* fired before the feature shipped.
- Slack Enterprise Grid org-level installs (single-workspace install only for v1).

## 6. Risks & open questions

| # | Risk / question | Owner | Resolve by |
|---|---|---|---|
| 1 | Definition of "crossing": is the previous-day score the only baseline, or any prior score within N days? Backfill semantics depend on this. | Eng lead | Design stage |
| 2 | Slack token storage: per-tenant schema vs. central encrypted vault. Schema-per-tenant ADR-0001 favors per-tenant; security may push central with KMS envelope. | Security + Eng lead | Threat-model review |
| 3 | Slack rate limits at scale: a tenant with 5,000 accounts having a bad model day could enqueue thousands of alerts. Need a per-tenant burst cap. | SRE | Design stage |
| 4 | GDPR data-processing addendum: sending tenant's customer data to a Slack workspace owned by the tenant — is Aurora a sub-processor? Likely yes; legal must update DPA template. | Legal + DPO | Before pilot tenants |
| 5 | LaunchDarkly flag scope: per-tenant boolean vs. percentage rollout. Percentage rollout doesn't make sense for an opt-in feature. | PM | Design stage |
| 6 | Score-driver text may include account-attribute names that the tenant considers sensitive. Need an allowlist of driver fields. | PM + Security | Design stage |
| 7 | "Test alert" feature: is it in v1? Risk says yes (admins need to verify before relying on it); Story 5 references it as a cooldown bypass. Confirm in scope. | PM | End of requirements review |

## 7. Compliance flags

This feature triggers `sdlc-security` review. Specific flags below — these are the labels to apply to the Jira stories.

| Story | Flags | Reason |
|---|---|---|
| Story 1 (Connect workspace) | `auth`, `secrets`, `soc2`, `gdpr` | Stores third-party OAuth tokens; auth-touching change per project context requires IriusRisk threat model and SOC 2 evidence. |
| Story 2 (Routing) | `soc2` | Configuration of an outbound data flow; audit trail required. |
| Story 3 (Detect crossing) | (none beyond baseline) | Internal logic, no new data egress. |
| Story 4 (Deliver alert) | `pii`, `gdpr`, `soc2` | Account-level customer data leaves Aurora's boundary to a tenant-controlled Slack workspace. Sub-processor relationship; DPA implications. Strict allowlist on payload fields required. |
| Story 5 (Dedup/cooldown) | (none beyond baseline) | Internal state. |
| Story 6 (Audit + replay) | `pii`, `soc2`, `gdpr` | Surfaces historical delivery records; must redact user-identifying Slack metadata. |

Aggregate label set for the parent Workstream: `auth`, `secrets`, `pii`, `gdpr`, `soc2`.

Accessibility: the Integrations and Activity UI changes (Stories 1, 2, 6) are in the Next.js client and must meet the existing WCAG 2.1 AA bar — flag `accessibility` / `wcag` on those three stories as well.

HIPAA-readiness (FY27): no PHI flows through churn alerts in v1 (Aurora does not yet ingest PHI). Note in the design stage that if HIPAA tenants come online, Slack as a destination requires a Slack BAA — gate via tenant tier rather than retrofitting later.

## 8. Hand-off

Next stage: **Design** (`sdlc-design`). Artifact: `02-design.md`.

Open questions in section 6 (especially #1, #2, #3, #5) are inputs the design stage must resolve.

## Tracker links

Pushed to Jira on 2026-05-05 (site: `dltk-starpoint.atlassian.net`, project: `SDLCTEST`).

- **Workstream**: [SDLCTEST-1](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-1) — Churn-Risk Slack Alerts
- **Stories**:
  - [SDLCTEST-2](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-2) — Story 1: Connect a Slack workspace to a tenant
  - [SDLCTEST-3](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-3) — Story 2: Configure alert routing per tenant
  - [SDLCTEST-4](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-4) — Story 3: Detect upward 0.8 crossing and enqueue an alert
  - [SDLCTEST-5](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-5) — Story 4: Deliver the alert to Slack with the right payload
  - [SDLCTEST-6](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-6) — Story 5: Dedup / cooldown so a single account can't spam
  - [SDLCTEST-7](https://dltk-starpoint.atlassian.net/browse/SDLCTEST-7) — Story 6: Tenant admin can audit and replay Slack alert activity

The local artifact (`docs/sdlc/churn-slack-alerts/01-requirements.md`) is the system of record; Jira mirrors it.
