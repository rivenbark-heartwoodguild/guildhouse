---
name: External Resources and Dashboards
description: Where to find bug tracking, monitoring, runbooks, and status information
type: reference
---

# External Resources and Dashboards

Quick reference for where things live outside the codebase.

## Bug Tracking

**Linear workspace:** https://linear.app/relay-team
- Triage happens Monday mornings. Untriaged bugs go to the `Incoming` state.
- Critical bugs (P0/P1) get assigned immediately via Slack alert in #relay-incidents.
- Bug labels: `bug`, `regression`, `performance`, `security`. Every bug gets exactly one.

## Monitoring

**Grafana dashboards:** https://grafana.internal.example.com/d/relay-overview
- **Relay Overview** — Request rate, error rate, p50/p95/p99 latency, active connections.
- **Connection Pool** — PgBouncer stats, active/waiting/idle connections per service. This is the one that caught the March 12 outage (after the fact).
- **WebSocket Health** — Connection count, message throughput, reconnection rate. Added in Sprint 3.

**PagerDuty service:** https://relay-team.pagerduty.com/services/P1A2B3C
- On-call rotation: weekly, Sunday to Sunday. Current schedule in the team roster.
- Escalation: if primary doesn't acknowledge in 10 minutes, escalates to secondary, then to Dana.

## Runbooks

**Runbook repository:** https://github.com/example-org/relay-runbooks
- `connection-pool-exhaustion.md` — Written after March 12. Steps to diagnose, mitigate, and recover.
- `websocket-node-failover.md` — How sticky sessions re-route when a node goes down.
- `deployment-rollback.md` — Emergency rollback procedure (under 90 seconds if pre-staged).

## Status Page

**Public status:** https://status.example.com
- Relay notification delivery is a tracked component.
- Incidents auto-create from PagerDuty P0 alerts. Manual updates required for P1+.
