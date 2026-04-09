---
name: Relay v3 Migration
description: WebSocket push replacing polling for real-time notification delivery
type: project
---

# Relay v3 Migration

WebSocket-based push notifications replacing the current polling architecture. Target: 50% reduction in API traffic, sub-second delivery latency.

## Team

| Person | Role | Focus |
|--------|------|-------|
| Dana Chen | Tech Lead | Architecture, WebSocket server, connection management |
| Marcus Webb | Backend | Event pipeline, outbox pattern, message serialization |
| Priya Sharma | Frontend | Client SDK, reconnection logic, offline queue |
| Tomás Rivera | Infrastructure | Load balancing, health checks, deployment strategy |

## Current Sprint (Sprint 4 of 6)

**Goal:** Auth integration and tenant isolation for WebSocket connections.

- [x] Connection authentication via short-lived tokens
- [x] Tenant-scoped channels (messages never cross tenant boundaries)
- [ ] Token refresh without connection drop (in progress — Dana)
- [ ] Load test at 10k concurrent connections per node (blocked)

## Key Decisions

- **Outbox pattern over event sourcing.** Decided in architecture review (2024-11-15). Event sourcing added complexity without clear benefit for our write patterns. Outbox gives us reliable event publishing with a simpler mental model.
- **Sticky sessions for WebSocket nodes.** Connection state lives in-memory on the node. Sticky routing via consistent hashing on tenant ID. Failover reconnects to a new node and replays from the outbox.
- **6-month parallel run.** Polling and WebSocket will coexist for 6 months. Clients opt in to WebSocket via feature flag. Polling is the fallback, not deprecated until WebSocket proves stable.

## Blockers

- **Auth token refresh:** Current token library doesn't support refresh without a full re-auth. Dana is evaluating whether to patch the library or implement a custom refresh flow. Decision needed by end of sprint.
- **Load testing environment:** The staging cluster maxes out at 2k connections. Tomás is provisioning a dedicated load-test environment but is waiting on infrastructure budget approval.
