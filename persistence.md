---
title: Persistence
parent: Features
nav_order: 3
---

# Persistence

InventoryFramework ships with two persistence backends and a provider-agnostic EF Core layer for SQL databases.

---

## File-Based (Default)

Inventory aggregates are stored as JSON files, one per aggregate. No external dependencies.

**When to use:** Development, jam games, single-server deployments with low player counts.

Configuration:
```json
"Persistence": { "Type": "File" },
"AggregateStorageRootPath": "data/inventories",
"ProgressionStorageRootPath": "data/progression"
```

The files are written atomically (write to temp file, then rename) to prevent corruption from crashes.

---

## SQLite

Inventory aggregates are stored in a single SQLite database file via EF Core.

**When to use:** Moderate player counts, when you want ACID guarantees without running a separate database server.

Configuration:
```json
"Persistence": {
  "Type": "Sqlite",
  "ConnectionString": "Data Source=data/inventory.db"
}
```

The `Microsoft.EntityFrameworkCore.Sqlite` package is already included in the Server project. Migrations run automatically on startup (`autoMigrate: true`).

---

## SQL Server / PostgreSQL

The `InventoryFramework.Persistence.Sql` project contains no database provider; it only references `Microsoft.EntityFrameworkCore` base packages. This keeps the project portable and avoids bundling providers you don't use.

To add SQL Server or PostgreSQL support:

**SQL Server:**
1. Add `Microsoft.EntityFrameworkCore.SqlServer` to `InventoryFramework.Server.csproj`.
2. Change `Persistence.Type` to `SqlServer` and provide a connection string.
3. Rebuild and run (migrations apply automatically).

**PostgreSQL:**
1. Add `Npgsql.EntityFrameworkCore.PostgreSQL` to `InventoryFramework.Server.csproj`.
2. Change `Persistence.Type` to `PostgreSql` and provide a connection string.
3. Rebuild and run.

If you set the type to `SqlServer` or `PostgreSql` without adding the corresponding NuGet package, the server throws a descriptive error at startup:

```
InvalidOperationException: SqlServer persistence requires the Microsoft.EntityFrameworkCore.SqlServer package.
Add it to InventoryFramework.Server.csproj and rebuild.
```

---

## Concurrency

All SQL writes use **optimistic concurrency** based on the `Revision` column:

1. Read existing revision from the database.
2. Check that stored revision equals aggregate's expected revision.
3. Increment revision and write.
4. If revision mismatch is detected, throw `RepositoryConcurrencyException`.

Application services catch this exception and return an appropriate error to the client. Clients should retry the operation after re-fetching the latest inventory state.

---

## Custom Persistence

Implement `IInventoryAggregateRepository` and register it in the DI container before calling `AddSqlInventoryPersistence`. The interface has two methods:

```csharp
Task<InventoryAggregateLoadResult?> GetAsync(InventoryAggregateId aggregateId, CancellationToken ct = default);
Task<AggregateRevision> SaveAsync(InventoryAggregate aggregate, InventoryAggregateSaveOptions saveOptions, CancellationToken ct = default);
```

Use `InventorySnapshotMapper` (already registered as a singleton) to convert between domain aggregates and JSON-serializable snapshots.
