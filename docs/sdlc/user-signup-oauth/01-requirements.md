# Requirements — Email + Google OAuth signup

**Feature slug:** `user-signup-oauth`
**Status:** Draft
**Author:** sevaai-sdlc / sdlc-requirements
**Date:** 2026-05-04

## 1. Summary

Aurora needs a self-service signup so prospects from the website can create a workspace without sales touch. Two paths: email + password, and Google OAuth. New users land in an empty tenant workspace and get a welcome email. The flow must capture enough metadata for product analytics and feed Aurora's own churn model later.

## 2. Personas

| Persona | Role | Primary need |
|---|---|---|
| Sam — CS Ops Manager | Self-serve trial signup from marketing site | Spin up a trial workspace in <60 seconds without contacting sales |
| Maya — IT Admin | Org-wide adoption with Google Workspace | Use existing Google credentials, no new password to manage |
| Internal Sales Eng | Demo accounts | Provision throwaway test workspaces quickly |

## 3. Business goal

- **Primary KPI**: signup-to-activation conversion >= 35% within 30 days of first login
- **Secondary KPI**: time-to-first-workspace <= 60 seconds (95th percentile)
- **Outcome**: unblock self-serve trial program; remove "request a demo" friction in funnel

## 4. User stories

### Story 1: Email + password signup
- **As a** prospect, **I want** to sign up with email + password, **so that** I can try Aurora without contacting sales.
- **Acceptance criteria**
  - Given a valid email and password >=12 chars, when I submit signup, then a tenant + user are created and I'm logged in.
  - Given an email already in use, when I submit signup, then I get a uniform "if this email exists, check your inbox" message (anti-enumeration).
  - Given a weak password (<12 chars or in HIBP top-100k), when I submit, then signup is rejected with a guidance message.
- **Edge cases**: unicode/emoji in name, RTL display names, SMTP bounce on welcome email.
- **Dependencies**: SMTP relay (SES) configured; HIBP password-check service.
- **Estimate**: M

### Story 2: Google OAuth signup
- **As a** Google Workspace user, **I want** to sign up with my Google account, **so that** I don't manage a new password.
- **Acceptance criteria**
  - Given I click "Continue with Google", when I authorize Aurora, then a tenant + user are created using my Google profile.
  - Given my Google email matches an existing user, when I sign in, then I'm linked to that user (no duplicate tenant).
  - Given the OAuth `state` value doesn't match, when I return to the callback, then the request is rejected with `403 csrf_mismatch`.
- **Edge cases**: Google denies consent; Google returns no email scope; user revokes Aurora later.
- **Estimate**: M

### Story 3: Welcome email job
- **As a** new user, **I want** a welcome email after signup, **so that** I know what to do next.
- **Acceptance criteria**
  - Given signup succeeds, when the request returns 201, then a `welcome_email` job is enqueued.
  - Given the SMTP send fails, when the worker retries, then it backs off exponentially and gives up after 5 attempts.
- **Estimate**: S

### Story 4: Tenant isolation guarantees
- **As a** security engineer, **I want** every signup-created object to carry the new tenant_id, **so that** there is no path for new users to read another tenant's data.
- **Acceptance criteria**
  - Given a signup completes, when the user issues any subsequent request, then the JWT carries `tenant_id` and middleware rejects any request to another tenant.
  - Given a unit test that issues a token for tenant A and reads tenant B's data, when run in CI, then it fails closed.
- **Estimate**: S
- **Compliance**: SOC 2 CC6.1 — logical access; tagged

### Story 5: Audit logging
- **As a** SOC 2 auditor, **I want** every signup event logged with timestamp, IP, user agent, IdP, **so that** I can produce evidence on demand.
- **Acceptance criteria**
  - Given signup occurs (success or failure), when the response is returned, then an `audit_event` row is written with the listed fields.
  - Audit log is retained 7 years (Aurora default).
- **Estimate**: S

### Story 6: Rate limiting
- **As an** SRE, **I want** signup endpoints rate-limited per IP and per email, **so that** abuse and credential stuffing are bounded.
- **Acceptance criteria**
  - Given >5 signup attempts from one IP / 60s, when a 6th arrives, then it returns 429.
  - Given >3 signup attempts for one email / 1h, when a 4th arrives, then it returns 429.
- **Estimate**: S

### Story 7: Welcome screen + first workspace
- **As a** new user, **I want** to land on a non-empty welcome screen, **so that** I know what Aurora does and what to do first.
- **Acceptance criteria**
  - Given signup completes, when I'm redirected, then I see an empty workspace with 3 quick-start cards.
- **Estimate**: M (frontend work; tracked in Aurora frontend repo)

## 5. Out of scope

- SAML / SCIM (deferred; covered later when we go upmarket)
- Magic-link login
- Multi-factor enrollment at signup (added in a follow-up story)
- Importing existing CRM data on signup

## 6. Risks & open questions

| # | Risk / question | Owner | Resolve by |
|---|---|---|---|
| 1 | Do we tie tenant to email domain (Google Workspace) or always create a new tenant? | Eng + PM | end of design stage |
| 2 | HIBP rate-limit at our anticipated signup volume? | SRE | before merge |
| 3 | Auth0 still planned; do we ship now via in-house auth and migrate later, or wait? | Eng leadership | already decided — ship in-house, migrate later |

## 7. Compliance flags

- **PII**: yes — email, name. Treat per Aurora data-classification policy.
- **Auth**: yes — touches authentication primitives. Triggers `sdlc-security` deep review and IriusRisk model.
- **Audit log**: yes — required for SOC 2 CC6.1, CC6.6.
- **GDPR**: yes — must support data-deletion request via existing `DELETE /v1/users/me` flow.

## 8. Hand-off

Next stage: **Design** (`sdlc-design`). Artifact: `02-design.md`.
