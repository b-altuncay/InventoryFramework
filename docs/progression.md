---
title: Player Progression
parent: Features
nav_order: 2
---

# Player Progression

InventoryFramework uses a lightweight unlock-key system to gate recipe access per actor. Each actor has a `PlayerProgression` record that stores a set of named keys with timestamps.

> **Availability:** Player progression requires the **Pro** license tier or higher.

---

## Concept

A *recipe unlock key* is an arbitrary string (e.g. `"blacksmith_tier2"`, `"alchemy_unlocked"`). Recipes that require a key are hidden and uncraftable until the actor's progression record contains that key.

Keys are:
- **Granted** by a server-side admin call (e.g. after completing a quest).
- **Revoked** individually.
- **Persisted** per actor in the configured progression storage.

---

## Storage

Progression records are stored separately from inventory aggregates. Configure the storage path in `appsettings.json`:

```json
{
  "InventoryFramework": {
    "ProgressionStorageRootPath": "data/progression"
  }
}
```

The file backend writes one JSON file per actor under `ProgressionStorageRootPath`. The SQL backend stores records in the `PlayerProgression` table (created automatically via EF Core migrations).

---

## SDK Operations

### Get current progression

Returns the actor's full progression record, including all unlocked key names and timestamps.

```csharp
var result = await client.GetPlayerProgressionAsync(
    inventoryId: "inv-player-123");

foreach (var key in result.UnlockedKeys)
{
    Console.WriteLine($"{key.Key} unlocked at {key.UnlockedAt}");
}
```

### Unlock a recipe key

Grants a named key to the actor. Requires an **admin API key**.

```csharp
var result = await client.UnlockRecipeKeyAsync(
    inventoryId: "inv-player-123",
    key: "blacksmith_tier2");
```

- If the actor already holds the key the call succeeds without error.
- `result.Succeeded` is `false` only on a server error or if the feature is disabled.

### Revoke a recipe key

Removes a key from the actor's progression. Requires an **admin API key**.

```csharp
var result = await client.RevokeRecipeKeyAsync(
    inventoryId: "inv-player-123",
    key: "blacksmith_tier2");
```

- If the key was not present, the call still succeeds (idempotent).

---

## How keys gate recipes

When an actor calls `BrowseRecipes`, `GetRecipeDetails`, or `CraftItems`, the server evaluates `IRecipeUnlockEvaluator` for each recipe that has `requiredUnlockKeys` set.

The default evaluator (`DefaultRecipeUnlockEvaluator`) loads the actor's progression record from `IPlayerProgressionRepository` and checks that every required key is present. Recipes that fail the check are:

- Hidden from `BrowseRecipes` (when `isHiddenUntilUnlocked: true`).
- Blocked from crafting regardless of `isHiddenUntilUnlocked`.

---

## Customising progression

Override any of the following interfaces via dependency injection:

| Interface | Purpose |
|---|---|
| `IPlayerProgressionRepository` | Custom storage backend. |
| `IRecipeUnlockEvaluator` | Custom unlock logic (e.g. level gates, flag checks). |
| `IRecipeVisibilityPolicy` | Controls which recipes appear in `BrowseRecipes`. |

---

## Feature flag

Player progression is gated by the `PlayerProgression` feature flag (see [Server Configuration](server-configuration.md)). When the flag is off, all progression calls return a `feature_disabled` error.
