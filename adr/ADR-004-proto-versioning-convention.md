# ADR-004: Proto Versioning via csharp_namespace, not Package Rename

**Status:** Accepted
**Date:** 2026-04-16

## Context

The gRPC contract is defined in `inventory.proto` files (one in `InventoryFramework.Server`,
one mirrored in `InventoryFramework.SDK`). As the API evolves, breaking changes must be
managed without forcing all SDK consumers to update simultaneously.

Two versioning mechanisms are available in protobuf:

1. **Proto package versioning**: Change `package inventory;` to `package inventory.v1;`.
   gRPC service URLs become `/inventory.v1.InventoryService/GrantItems`. Old and new versions
   can coexist in the same server.

2. **csharp_namespace option**: Override the generated C# namespace independently of the
   proto package. The proto package (`inventory`) controls service URLs; `csharp_namespace`
   controls generated class names.

## Options Considered

### Option A: Rename proto package to `inventory.v1`
Change both proto files from `package inventory;` to `package inventory.v1;`.

**Problem:** The proto `package` determines the gRPC service URL path. Renaming it would
change all endpoint paths from `/inventory.InventoryService/GrantItems` to
`/inventory.v1.InventoryService/GrantItems`. Every client connecting to the old URL would
immediately receive `UNIMPLEMENTED`. This is a hard breaking change with no graceful migration.

The current protos already use `option csharp_namespace = "InventoryFramework.Server.Grpc";`
(server) and `option csharp_namespace = "InventoryFramework.SDK.Grpc";` (SDK), which already
decouple the generated C# namespaces from the proto package name. Renaming the package
provides no C# benefit.

### Option B: Keep package name, version via csharp_namespace + URL routing (selected)
Keep `package inventory;` stable. Version the API at the HTTP routing layer (API gateway or
reverse proxy) if needed: `/v1/` prefix routes to the current server, `/v2/` to a new
deployment. The proto `csharp_namespace` is already versioned by convention (server and SDK
have different namespaces).

**For breaking proto changes:**
- Add new fields with new field numbers (backward compatible by default in proto3).
- For true breaking changes (remove a field, change a type), deploy a new service alongside
  the old one and migrate clients over a deprecation window.

## Decision

**Keep `package inventory;` permanently.** Do not rename the proto package for versioning.

API versioning is achieved through:
1. **Proto3 field addition rules**: New optional fields with new numbers are backward
   compatible. SDK consumers that do not know the field ignore it.
2. **Separate service deployment**: For breaking changes, a new server instance is deployed.
   An API gateway (nginx, Envoy, or Kubernetes ingress) routes `/v2/` traffic to it.
3. **SDK package version (SemVer)**: The NuGet package version communicates breaking changes
   to SDK consumers. A major version bump (v1.x.x to v2.0.0) signals a proto-level breaking
   change and requires an SDK upgrade.

## Consequences

- The gRPC endpoint URL (`/inventory.InventoryService/GrantItems`) is stable for all v1.x.x
  SDK releases. Clients do not need to update their channel address on minor upgrades.
- Future maintainers must not add required fields to existing proto messages (proto3 has no required fields, but default values must never carry semantic meaning; zero is not the same as "not set" for business logic purposes). Use wrapper types or the `optional` keyword when semantics matter.
- When a true breaking change is needed, create `inventory_v2.proto` with a new service name
  (`InventoryServiceV2`) and register it alongside the existing service in `Program.cs`.
  Deprecation notices go into release notes, not into the proto file.
