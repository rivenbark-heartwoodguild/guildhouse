# Relay — Project Memory

> Working knowledge for the Relay notification platform. Organized by type, linked to detail files.

## Active Feedback
- [No mocking in integration tests](feedback-example.md) — Use real database connections; mocks hid the connection pool bug for 3 weeks
- [Error messages are UI](feedback-error-messages.md) — Every user-facing error needs a next-action suggestion, not just a status code
- [Batch endpoints need idempotency keys](feedback-idempotency.md) — Retry storms after the March outage; now mandatory on all batch writes

## Infrastructure
- [Database connection pooling](infra-connection-pool.md) — PgBouncer config, max 50 connections per service, pool_mode=transaction
- [Deployment pipeline](infra-deploy-pipeline.md) — GitHub Actions → staging (auto) → production (manual gate), rollback under 90s
- [External resources and dashboards](reference-example.md) — Bug tracker, monitoring, runbooks, status page

## Active Projects
- [Relay v3 migration](project-example.md) — WebSocket push replacing polling; Sprint 4 of 6, blockers on auth integration
- [Rate limiter redesign](project-rate-limiter.md) — Moving from per-IP to per-tenant with sliding window; design approved, implementation started

## Sessions
- [Architecture review: event sourcing decision](session-example.md) — Decided against full event sourcing, adopted outbox pattern instead
- [Post-mortem: March 12 connection pool exhaustion](session-postmortem.md) — Root cause was missing connection release in error path; PgBouncer masked it for weeks

## Reference
- [Team roster and on-call rotation](reference-team.md) — 4 engineers, weekly rotation, escalation path documented
- [API versioning strategy](reference-api-versioning.md) — URL-based (v1, v2), 6-month deprecation window, sunset headers required
