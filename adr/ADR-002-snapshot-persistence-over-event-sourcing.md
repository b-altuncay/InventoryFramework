# ADR-002: Snapshot Persistence over Event Sourcing

**Status:** Accepted
**Date:** 2026-04-16

## Context

The inventory aggregate accumulates state through domain operations (AddStack, RemoveFromSlot,
Move, etc.). This state must be persisted and reloaded reliably. Two dominant patterns exist:

- **Snapshot persistence**: Serialize the current aggregate state as a single document (JSON).
  On load, deserialize the document and reconstruct the aggregate directly.
- **Event sourcing**: Persist every domain event in an append-only event store. On load,
  replay all events from the beginning (or from a snapshot checkpoint) to reconstruct state.

## Options Considered

### Option A: Event Sourcing
Append `ContainerItemAddedEvent`, `ContainerItemRemovedEvent`, etc. to an event log per
aggregate. Reconstruct state by replaying the log.

Pros:
- Full audit trail, every change is traceable.
- Time-travel debugging: replay state at any point in time.
- Natural fit for CQRS read-model projections.

Cons:
- Replay time grows linearly with aggregate history. A chest modified 10,000 times requires
  replaying 10,000 events on every load.
- Snapshot checkpointing must be added anyway to bound replay time, which partially offsets
  the simplicity advantage.
- Game inventory aggregates have high write frequency (each item pickup is a mutation) but
  low auditability requirements; players do not file "item history" disputes.
- Event schema versioning (upcasting) becomes a permanent maintenance burden as the domain
  model evolves.
- Infrastructure complexity increases significantly (event store, projection workers).

### Option B: Snapshot Persistence (selected)
Serialize the full aggregate state as JSON on every save. Load by deserializing the latest
snapshot.

Pros:
- Load time is O(1) regardless of aggregate history.
- Schema evolution is handled by the `InventorySnapshotMapper` which can apply default values for missing fields, no upcasting pipeline required.
- Provider-agnostic: the same JSON snapshot works with file storage, SQLite, SQL Server, or PostgreSQL without changing domain code.
- Simpler infrastructure: no event store, no projection workers, no replay logic.

Cons:
- No built-in audit trail. Domain events are raised but discarded after dispatch unless the Outbox persists them (see ADR-003).
- Cannot reconstruct state at an arbitrary past point in time.

## Decision

Use **snapshot persistence**. The `InventorySnapshotMapper` translates between the domain
aggregate and a versioned `InventoryAggregateSnapshot` DTO. The snapshot is serialized to
JSON and stored as a single row/file per aggregate.

The snapshot format is versioned by convention: new fields are nullable or have defaults,
old fields are never removed until a major version bump. The mapper enforces this boundary.

## Consequences

- `InventorySnapshotMapper` is the single point of schema migration logic. Any new field
  added to the domain aggregate must have a corresponding nullable or defaulted entry in the
  snapshot DTO.
- Domain events (`ContainerItemAddedEvent`, etc.) are published for real-time notification
  purposes but are **not** the source of truth for aggregate state.
- If a full audit trail is required in the future, an append-only event log can be added
  alongside snapshot persistence without changing the core domain model.
- Performance: snapshots are bounded in size by the aggregate's slot count × max item data.
  A 100-slot inventory snapshot is approximately 10–50 KB. No read amplification concern.
