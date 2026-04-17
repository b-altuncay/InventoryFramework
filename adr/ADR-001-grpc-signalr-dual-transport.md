# ADR-001: gRPC + SignalR Dual-Transport

**Status:** Accepted  
**Date:** 2026-04-16

## Context

The inventory server needs to serve two distinct communication patterns:

1. **Request/response**: Client sends a command (grant items, transfer, craft) and waits for a
   deterministic result. Failure modes, retry semantics, and error codes must be precise.

2. **Server push**: After a mutation, all clients watching the same inventory must receive a
   real-time notification without polling. Latency matters here — a 5-second polling loop is
   unacceptable in a multiplayer trading scenario.

The system targets Unity, Godot, and Unreal Engine game clients. These environments have
varying levels of HTTP/2 support and very different threading models.

## Options Considered

### Option A: Pure gRPC (server streaming for push)
gRPC server streaming can push inventory change events over a long-lived HTTP/2 stream.

**Pros:** Single protocol, single port, full type safety end-to-end via protobuf.  
**Cons:**
- gRPC streaming in Unity WebGL and Godot GDScript is poorly supported without heavy wrappers.
- Server-streaming connections require careful flow-control and reconnect logic per client.
- Game engines often drop HTTP/2 connections on scene transitions; reconnect handling becomes
  application-level responsibility.

### Option B: Pure REST + WebSocket
Standard REST for commands, raw WebSocket for push.

**Pros:** Universal support in all environments.  
**Cons:** Loses protobuf type safety on the command path. REST error handling is weaker than
gRPC status codes. Two separate client implementations to maintain.

### Option C: gRPC for commands + SignalR for push (selected)
Use gRPC for all state-mutating and query operations. Use SignalR for real-time server-push
notifications only.

**Pros:**
- gRPC gives precise status codes, proto-typed requests/responses, and built-in deadline support.
- SignalR handles reconnect, transport negotiation (WebSocket → Long Polling fallback), and
  connection grouping automatically.
- SignalR's hub group model maps naturally to "all clients watching inventory X".
- SDK can expose push via a clean callback interface, hiding the SignalR complexity.

**Cons:**
- Two protocols increase the conceptual surface area.
- API key must be passed as a query parameter for SignalR (WebSocket handshake cannot carry
  Authorization headers in all environments). This is logged at WARN level and documented.

## Decision

Use **gRPC for all command/query traffic** and **SignalR for server-push notifications only**.

The SignalR hub (`InventoryHub`) is restricted to push — clients cannot invoke server methods
through it. All mutations go through gRPC. This avoids business logic duplication.

## Consequences

- Clients must establish two connections: one gRPC channel and one SignalR connection.
- The SDK abstracts this behind `IInventoryClient` (gRPC) and a separate push subscription
  interface. Engine adapters (Unity/Godot/Unreal facades) wire both together.
- API keys in SignalR query parameters must be transmitted over HTTPS only. The startup
  validator enforces HTTPS in Production environments.
- At-least-once delivery of push notifications is handled by the Outbox pattern (see ADR-003).
