# Aurora — High-level architecture

```
[Next.js client] --> [ALB] --> [API Service (Fastify)] --> [Postgres RDS]
                                  |--> [Worker Service (BullMQ)] --> [Postgres / external APIs]
                                  |--> [Redis ElastiCache]
                                  |--> [Datadog APM]
```

## Services

- **API service** — public REST API consumed by the client and external integrations
- **Worker service** — background jobs (event ingest, churn-score recompute, email)
- **Scheduler** — cron-style jobs in EventBridge -> Worker

## Data model (high level)

- `tenant` — top-level customer org
- `account` — a customer of the tenant (e.g., a SaaS company that Aurora's customer is selling to)
- `user` — individual logging in; bound to one tenant via `tenant_user`
- `event` — product-usage signal (high write volume, partitioned by month)
- `ticket` — support ticket pulled from Zendesk/Intercom
- `score` — daily churn-risk score per account

## Existing ADRs

- ADR-0001: Schema-per-tenant for hard isolation
- ADR-0002: BullMQ over SQS for queues (latency, in-process retry)
- ADR-0003: Auth0 selected as IdP (not yet implemented)
