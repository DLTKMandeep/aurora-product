# Aurora

A B2B customer success analytics SaaS. Aggregates product-usage events, support tickets, and contract data to predict churn and surface "save plays" for CS teams.

## Stack

- **Runtime**: Node 20 + TypeScript
- **API**: Fastify 4 + JSON Schema
- **Database**: Postgres 16 (multi-tenant, schema-per-tenant)
- **Cache**: Redis 7
- **Queue**: BullMQ on Redis
- **Frontend**: Next.js 14 (separate repo)
- **Deploy target**: AWS — ECS Fargate behind ALB, RDS Postgres, ElastiCache Redis
- **CI/CD**: GitHub Actions -> Argo CD on EKS for non-customer-data services
- **Feature flags**: LaunchDarkly
- **Observability**: Datadog (APM + logs + RUM), Sentry, PagerDuty
- **Auth**: Auth0 (planned), email/password fallback

## Compliance posture

- SOC 2 Type II (in audit window)
- GDPR (EU customers)
- Targeting HIPAA-readiness for FY27

## Repo layout

- `src/` — server source
- `docs/` — architecture docs, ADRs, runbooks
- `tests/` — Jest test suites
- `migrations/` — node-pg-migrate SQL migrations

## Engineering norms

- Trunk-based development; feature branches < 5 days old
- PR size budget 400 lines net diff
- Conventional Commits
- 80% line coverage gate on changed files
- No-deploy windows: Fri >= 14:00, weekends, US public holidays
