---
title: SDK Usage (Plain C#)
parent: Engine Integrations
nav_order: 1
---

# SDK Usage (Plain C#)

`InventoryFramework.SDK` is a `netstandard2.1` client library that works in any .NET project — console apps, ASP.NET Core backends, test harnesses — without requiring Unity or Godot.

---

## Installation

Add a project reference or NuGet package reference to `InventoryFramework.SDK`:

```xml
<PackageReference Include="InventoryFramework.SDK" Version="*" />
```

---

## Creating a client

`GrpcInventoryClient` wraps the gRPC channel and implements `IInventoryClient`.

```csharp
using InventoryFramework.SDK.Grpc;
using InventoryFramework.SDK.Models;

var options = new InventoryClientOptions
{
    ServerAddress = "https://your-server:5001",
    ApiKey        = "sk-game-your-secret-key"
};

IInventoryClient client = new GrpcInventoryClient(options);
```

### InventoryClientOptions

| Property | Type | Default | Description |
|---|---|---|---|
| `ServerAddress` | string | required | Full URL of the gRPC server (e.g. `https://localhost:5001`). |
| `ApiKey` | string | `""` | API key sent in the `x-api-key` gRPC metadata header. Leave empty if `RequireApiKey` is `false` on the server. |
| `TimeoutSeconds` | int | `30` | Per-call deadline in seconds. |

---

## Common operations

### Create an inventory

```csharp
var result = await client.CreateInventoryAsync(ownerActorId: "player-123");

if (result.Succeeded)
    Console.WriteLine($"Inventory id: {result.InventoryId}");
```

### Get inventory

```csharp
var result = await client.GetInventoryAsync(inventoryId: "inv-player-123");

foreach (var container in result.Aggregate.Containers)
{
    Console.WriteLine($"Container {container.Id}: {container.Slots.Count} slots");
}
```

### Grant items (admin key required)

```csharp
var result = await client.GrantItemsAsync(
    inventoryId:    "inv-player-123",
    containerId:    "container-backpack-guid",
    itemDefinitionId: "wood",
    quantity:       10);
```

### Transfer items

Moves items between two containers within the same inventory.

```csharp
var result = await client.TransferItemsAsync(
    inventoryId:         "inv-player-123",
    sourceContainerId:   "container-backpack-guid",
    sourceSlotIndex:     0,
    targetContainerId:   "container-chest-guid",
    quantity:            5);
```

### Quick-store

Automatically moves all stackable items from a source container into the best-matching slots in a target container.

```csharp
var result = await client.QuickStoreItemsAsync(
    inventoryId:       "inv-player-123",
    sourceContainerId: "container-backpack-guid",
    targetContainerId: "container-chest-guid");
```

### Lock / unlock a slot

Prevent automated operations (quick-store, sort) from touching a specific slot. Manual moves and crafting still work.

```csharp
// Lock
await client.LockSlotAsync(inventoryId, containerId, slotIndex: 2, lockSlot: true);

// Unlock
await client.LockSlotAsync(inventoryId, containerId, slotIndex: 2, lockSlot: false);
```

### Split a stack

Split some items from a slot into an adjacent empty slot. Optionally specify the destination.

```csharp
var result = await client.SplitStackAsync(
    inventoryId,
    containerId,
    sourceSlotIndex: 0,
    amount: 5);

Console.WriteLine($"Split landed in slot {result.DestinationSlotIndex}");
```

### Drop items

Remove items from a slot and discard them. The returned data lets the game spawn a world pickup.

```csharp
var result = await client.DropItemsAsync(
    inventoryId,
    containerId,
    slotIndex: 0,
    amount: 3);

if (result.Succeeded)
    SpawnWorldDrop(result.DroppedItemDefinitionId, result.DroppedQuantity, playerPosition);
```

### Sort container

Rearrange all unlocked slots. Locked slots act as fixed anchors the sort routes around.

```csharp
// 0 = ByNameAscending, 1 = ByWeightDescending, 2 = ByTagThenName
await client.SortContainerAsync(inventoryId, containerId, sortMode: 0);
```

### Trade (Pro+)

Atomically swaps items between two inventories.

```csharp
var result = await client.TradeItemsAsync(
    sourceInventoryId:   "inv-player-123",
    targetInventoryId:   "inv-player-456",
    sourceContainerId:   "container-backpack-a",
    sourceSlotIndex:     0,
    targetContainerId:   "container-backpack-b",
    targetSlotIndex:     2,
    quantity:            1);
```

### Craft items

```csharp
var result = await client.CraftItemsAsync(
    inventoryId:       "inv-player-123",
    recipeId:          "plank_recipe",
    craftCount:        2,
    sourceContainerId: "container-backpack-guid",
    targetContainerId: "container-backpack-guid");
```

For full crafting documentation see [Crafting & Recipes](crafting.md).

---

## Error handling

Every operation returns a result type that implements `InventoryClientResult`:

```csharp
public abstract class InventoryClientResult
{
    public bool   Succeeded  { get; }
    public string ErrorCode  { get; }
    public string ErrorMessage { get; }
}
```

Common error codes:

| Code | Meaning |
|---|---|
| `not_found` | The inventory or recipe does not exist. |
| `concurrency_conflict` | The inventory was modified concurrently — retry. |
| `insufficient_ingredients` | Not enough items to craft. |
| `feature_disabled` | The feature is off in server config or the license tier is too low. |
| `unauthorized` | Missing or invalid API key. |
| `permission_denied` | The caller does not have access to this inventory. |

---

## Dependency injection

Register `GrpcInventoryClient` as a singleton in an ASP.NET Core application:

```csharp
builder.Services.AddSingleton<IInventoryClient>(_ =>
    new GrpcInventoryClient(new InventoryClientOptions
    {
        ServerAddress = builder.Configuration["Inventory:ServerAddress"]!,
        ApiKey        = builder.Configuration["Inventory:ApiKey"]!
    }));
```

---

## Real-time updates

For real-time inventory change notifications subscribe to the SignalR hub. See [SignalR Events](signalr-events.md).
