# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) for InventoryFramework.

Each ADR documents a significant architectural decision: the context, the options considered,
the decision taken, and the consequences. They are append-only — superseded decisions are marked
as such rather than deleted.

## Index

| ID | Title | Status |
|----|-------|--------|
| [ADR-001](ADR-001-grpc-signalr-dual-transport.md) | gRPC + SignalR dual-transport | Accepted |
| [ADR-002](ADR-002-snapshot-persistence-over-event-sourcing.md) | Snapshot persistence over event sourcing | Accepted |
| [ADR-003](ADR-003-outbox-pattern-at-least-once-delivery.md) | Outbox pattern for at-least-once event delivery | Accepted |
| [ADR-004](ADR-004-proto-versioning-convention.md) | Proto versioning via csharp_namespace, not package rename | Accepted |

## Format

Each ADR follows the template:
- **Status**: Proposed / Accepted / Deprecated / Superseded by ADR-XXX
- **Context**: Why this decision was needed
- **Options Considered**: What alternatives were evaluated
- **Decision**: What was chosen
- **Consequences**: Trade-offs and follow-up obligations
