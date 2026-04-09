---
name: Architecture Review — Event Sourcing Decision
description: Evaluated event sourcing vs outbox pattern for Relay v3 notification pipeline
type: session
---

# Architecture Review — Event Sourcing Decision

**Date:** 2024-11-15
**Participants:** Dana, Marcus, Priya, Tomás

## Decisions Made

### 1. Outbox pattern, not event sourcing

We considered full event sourcing for the notification pipeline. Rejected it. Our write patterns are simple (create notification, mark delivered, mark read) and don't benefit from event replay. The outbox pattern gives us reliable event publishing — a transactional outbox table that a poller or CDC process reads — without the complexity of event stores, projections, and eventual consistency across read models.

The deciding factor: our team has zero event sourcing experience, and the migration timeline is 6 sprints. Learning a new paradigm mid-project is a risk we don't need to take.

### 2. Message serialization format: Protocol Buffers

JSON was the default assumption. Marcus benchmarked Protobuf vs JSON for our message shapes — 40% smaller payloads, 3x faster serialization. At 10k concurrent connections, that's meaningful. The trade-off is developer ergonomics (Protobuf requires schema definitions and code generation), but Marcus will own the schema and CI pipeline for code gen.

### 3. Reconnection strategy: client-driven with server hints

Priya proposed a hybrid: the client manages reconnection timing (exponential backoff), but the server provides a "resume token" on each message so the client can request only missed messages on reconnect. This avoids full replay on transient disconnects.

## Left Open

- **Exactly-once vs at-least-once delivery.** We deferred this. Current leaning is at-least-once with client-side deduplication (idempotency keys on each message). But the mobile team hasn't weighed in on whether client-side dedup is practical on their end. Follow up scheduled for Sprint 5 kickoff.
- **Monitoring strategy for WebSocket connections.** Tomás will draft a proposal for connection-level metrics (duration, message throughput, error rates) before Sprint 5.
