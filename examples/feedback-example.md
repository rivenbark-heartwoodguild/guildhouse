---
name: No mocking databases in integration tests
description: Use real database connections in integration tests to catch connection management bugs
type: feedback
---

# No Mocking Databases in Integration Tests

**Rule:** Integration tests must use real database connections (via test containers or a dedicated test database). Mock the HTTP layer, never the data layer.

**Why:** A connection pool exhaustion bug in the notification service went undetected for 3 weeks because all integration tests used mocked database clients. The mock returned instantly, so the test never exercised the connection acquire/release lifecycle. In production, connections leaked on error paths, eventually exhausting the pool during a traffic spike on March 12.

**How to apply:** Use testcontainers to spin up a real PostgreSQL instance for integration test suites. Each test gets a transaction that rolls back on teardown. This adds ~8 seconds to the test suite startup but catches an entire class of bugs that mocks structurally cannot detect.
