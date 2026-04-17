# ADR-003: Outbox Pattern for At-Least-Once Event Delivery

**Status:** Accepted  
**Date:** 2026-04-16

## Context

When a domain operation succeeds (e.g., items transferred between containers), the server must:

1. Persist the updated aggregate snapshot.
2. Dispatch domain events to SignalR so connected clients receive real-time notifications.

These two operations are not atomic. If the server persists the snapshot but crashes before
dispatching the event, connected clients miss the notification and their UI falls out of sync.

This is specifically a concern for the SQL persistence path (`SqlInventoryAggregateRepository`).
The file-based path is used in single-node development scenarios where this race window is
acceptable.

## Options Considered

### Option A: Dispatch before persist
Publish the SignalR notification before committing the snapshot.

**Rejected:** If persist fails after dispatch, clients see a notification for a change that
never happened. Phantom events cause client-side inconsistency.

### Option B: Dispatch in the same DB transaction (Transaction Hook)
Override `SaveChangesAsync`, detect pending domain events, and dispatch them inside the
commit callback.

**Rejected:** SignalR `DispatchAsync` is I/O-bound and cannot safely participate in a DB
transaction. If dispatch throws, the transaction has already committed — we lose both
transactional atomicity and clarity about what happened.

### Option C: Outbox Pattern with Background Poller (selected)
Write domain events to an `OutboxMessages` table in the **same DB transaction** as the
aggregate snapshot. A separate `BackgroundService` polls for unprocessed outbox records,
dispatches them via `IDomainEventDispatcher`, and marks them as dispatched.

**Pros:**
- Aggregate snapshot + outbox record are committed atomically. No phantom events, no lost
  notifications after crash.
- Background poller is simple: read pending rows, dispatch, mark done.
- At-least-once delivery: if dispatch succeeds but the "mark done" write fails, the event
  is dispatched again on the next poll cycle. SignalR push is idempotent from the client's
  perspective (duplicate notifications are harmless — the client re-fetches state anyway).

**Cons:**
- Notification delay equals polling interval (default: 5 seconds). For real-time trading
  this is acceptable; for sub-second reaction games a different transport would be needed.
- Requires an additional DB table (`OutboxMessages`) and a background service.

## Decision

Implement the **Outbox Pattern with a Background Poller** for the SQL persistence path.

Design specifics:
- `OutboxMessages` table is part of the same `InventoryDbContext` — same DB, same schema,
  same migration.
- `SqlInventoryAggregateRepository.SaveAsync` writes outbox records inside the existing
  transaction before `CommitAsync`.
- `OutboxDispatcherBackgroundService` polls every 5 seconds, processes up to 50 pending
  messages per cycle (prevents thundering herd after downtime).
- Poison messages (unknown event type after deserialization) are marked dispatched and logged
  as errors rather than blocking the queue.
- The file-based persistence path (`FileInventoryAggregateRepository`) is unaffected — it
  continues dispatching synchronously after save (best-effort, at-most-once).

## Consequences

- **Client-side idempotency is required.** SDK consumers and engine adapter facades must
  treat SignalR push notifications as triggers to re-fetch inventory state, not as deltas.
  Duplicate notifications must not cause double-processing.
- `OutboxDispatcherBackgroundService` is only registered when SQL persistence is active.
  The DI registration is inside `SqlPersistenceServiceCollectionExtensions`.
- The 5-second polling interval is configurable via `OutboxOptions.PollingInterval` in
  `appsettings.json`. Reduce for latency-sensitive deployments; increase to reduce DB load.
- `OutboxMessages` table grows indefinitely. A cleanup job (or TTL-based delete) must be
  added for long-running production deployments. This is tracked as a follow-up task.
