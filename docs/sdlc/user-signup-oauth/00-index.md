# Feature dossier — Email + Google OAuth signup

**Slug:** `user-signup-oauth`
**Started:** 2026-05-04 17:05 UTC
**Driver:** sevaai-sdlc orchestrator
**Description:**

> Add a self-service signup flow to Aurora. New users sign up with email + password OR Google OAuth. They land on an empty workspace, are tenant-scoped, and trigger a welcome-email job. SOC 2 audit window is active so security and audit logging matter. Target ship: end of next sprint (~2 weeks).

## Stages

| # | Stage | Status | Artifact | Tools touched |
|---|---|---|---|---|
| 1 | Requirements | done | [01-requirements.md](01-requirements.md) | Atlassian Jira, Notion AI |
| 2 | Design | done | [02-design.md](02-design.md) | Eraser, Backstage, Stoplight, IriusRisk |
| 3 | Development | done | [03-development.md](03-development.md) | Cursor, Claude Code, CodeRabbit |
| 4 | Testing | done | [04-testing.md](04-testing.md) | Jest, Playwright, k6, Mabl |
| 5 | Security | done | [05-security.md](05-security.md) | IriusRisk, Snyk, Semgrep, GitGuardian, OWASP ASVS |
| 6 | Deployment | done | [06-deployment.md](06-deployment.md) | GitHub Actions, Argo CD, LaunchDarkly, AWS |
| 7 | Maintenance | done | [07-maintenance.md](07-maintenance.md) | Datadog, Sentry, PagerDuty, Statuspage |

## Top three risks (consolidated across stages)

1. **Tenant scoping at signup** — getting tenant resolution wrong leaks data across customers. Mitigated by middleware-enforced tenant claim in JWT + CI test.
2. **OAuth replay / state CSRF** — Google flow needs strict `state` + `nonce` validation. Pen-test plan covers this.
3. **Email enumeration** — signup endpoints can leak whether an email exists. Mitigated by uniform response timing + uniform error message.

## Top three follow-ups for this sprint

1. Wire LaunchDarkly flag `feature.email_oauth_signup` and seed at 0%.
2. Open Jira epic `AUR-EPIC-44` with the 7 user stories + sub-tasks.
3. File IriusRisk threat-model project for SOC 2 evidence.
