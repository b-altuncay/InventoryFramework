---
title: Server Configuration
parent: Getting Started
nav_order: 1
description: Configure the InventoryFramework server. API keys, database connection strings, item/recipe paths, feature flags, container capacity, weight limits, and HTTPS settings.
---

# Server Configuration

All server settings live in `appsettings.json` (or environment variables / user secrets in development).

---

## Full Example

```json
{
  "Kestrel": {
    "EndpointDefaults": {
      "Protocols": "Http2"
    }
  },
  "InventoryFramework": {
    "Auth": {
      "RequireApiKey": true,
      "ApiKeys": [
        { "Key": "sk-game-xxxxxxxxxxxxxxxx", "IsAdmin": false, "Description": "Game server key" },
        { "Key": "sk-admin-yyyyyyyyyyyyyy",  "IsAdmin": true,  "Description": "Admin key" }
      ]
    },
    "ItemDefinitionsRootPath": "data/items",
    "RecipeDefinitionsRootPath": "data/recipes",
    "AggregateStorageRootPath": "data/inventories",
    "ProgressionStorageRootPath": "data/progression",
    "Persistence": {
      "Type": "File",
      "ConnectionString": ""
    }
  }
}
```

---

## Settings Reference

### `Auth.RequireApiKey`

When `true`, every inbound gRPC request must include the `x-api-key` metadata header. When `false`, authentication is disabled (useful for local development).

### `Auth.ApiKeys`

An array of API key entries.

| Field | Description |
|---|---|
| `Key` | The secret key string. Use a long random value (e.g. `sk-` prefix + 32 random hex chars). |
| `IsAdmin` | `true` grants admin privileges: bypass recipe unlock key checks, access all actors' data. |
| `Description` | Optional human-readable label for the key (not validated). |

Generate a key:
```bash
node -e "console.log('sk-' + require('crypto').randomBytes(16).toString('hex'))"
```

### `ItemDefinitionsRootPath`

Directory (relative to the working directory) where item definition JSON files are stored. Subdirectories are scanned recursively by default.

### `RecipeDefinitionsRootPath`

Directory where recipe definition JSON files are stored.

### `AggregateStorageRootPath`

Root directory for file-based inventory storage. One file per inventory aggregate. Only used when `Persistence.Type` is `File`.

### `ProgressionStorageRootPath`

Root directory for player progression (recipe unlock key) storage. One file per actor ID. Only used with file-based persistence.

### `Persistence.Type`

Controls the inventory storage backend.

| Value | Description |
|---|---|
| `File` | JSON files on disk. Zero dependencies. Good for development and small deployments. |
| `Sqlite` | EF Core with SQLite. Set `ConnectionString` to the database file path. |
| `SqlServer` | Requires adding `Microsoft.EntityFrameworkCore.SqlServer` to the Server project. |
| `PostgreSql` | Requires adding `Npgsql.EntityFrameworkCore.PostgreSQL` to the Server project. |

For `SqlServer` and `PostgreSql`, the server throws a descriptive `InvalidOperationException` at startup listing the exact NuGet package to add. The framework is provider-agnostic; no provider is bundled for these.

### `Persistence.ConnectionString`

Database connection string. Required when `Persistence.Type` is `Sqlite`, `SqlServer`, or `PostgreSql`.

Example for SQLite:
```
Data Source=data/inventory.db
```

---

## Environment Variable Overrides

ASP.NET Core configuration supports environment variable overrides using double-underscore (`__`) as the section separator:

```bash
InventoryFramework__Persistence__Type=Sqlite
InventoryFramework__Persistence__ConnectionString="Data Source=/var/data/inventory.db"
```

---

## Development vs Production

For development, use `appsettings.Development.json` and .NET user secrets to keep API keys out of source control:

```bash
dotnet user-secrets set "InventoryFramework:Auth:ApiKeys:0:Key" "sk-dev-localkey"
dotnet user-secrets set "InventoryFramework:Auth:ApiKeys:0:IsAdmin" "true"
```
