---
title: Unity Integration
parent: Engine Integrations
nav_order: 2
---

# Unity Integration Guide

This guide covers adding InventoryFramework to a Unity project using the `InventoryFramework.UnityAdapter` library.

---

## Requirements

- Unity 2021.3 LTS or later
- .NET Standard 2.1 scripting backend (IL2CPP or Mono)
- A running InventoryFramework server (see [server-configuration.md](server-configuration.md))

---

## Installation

### Option A: NuGetForUnity (recommended)

Install [NuGetForUnity](https://github.com/GlitchEnzo/NuGetForUnity) from the Unity Package Manager, then search for and install:

- `InventoryFramework.UnityAdapter`

This pulls in `InventoryFramework.SDK` and all gRPC dependencies automatically.

### Option B: Manual DLL install

Download `InventoryFramework.UnityAdapter` from [NuGet.org](https://www.nuget.org/packages/InventoryFramework.UnityAdapter). A `.nupkg` file is a zip archive; rename it to `.zip`, open it, and copy the contents of `lib/netstandard2.1/` to `Assets/Plugins/InventoryFramework/` in your Unity project.

Required DLLs:
- `InventoryFramework.SDK.dll`
- `InventoryFramework.UnityAdapter.dll`
- `Grpc.Net.Client.dll`, `Grpc.Core.Api.dll`, `Grpc.Net.Common.dll`, `Google.Protobuf.dll`
- `Polly.dll`, `System.Memory.dll`

---

## Basic Setup

Create a MonoBehaviour (or ScriptableObject) that owns the facade:

```csharp
using InventoryFramework.UnityAdapter.Models;
using InventoryFramework.UnityAdapter.Services;
using UnityEngine;

public class InventoryManager : MonoBehaviour
{
    private UnityInventoryFacade _facade;

    private async void Start()
    {
        var config = new UnityInventoryConfiguration
        {
            ServerAddress = "https://localhost:7289",
            ActorId      = SystemInfo.deviceUniqueIdentifier,
            ApiKey       = "sk-game-your-key"
        };

        _facade = new UnityInventoryFacade(config);

        var created = await _facade.CreateDefaultInventoryAsync();
        if (!created)
        {
            Debug.LogError("Failed to create inventory.");
            return;
        }

        var snapshot = await _facade.RefreshAsync();
        Debug.Log($"Inventory loaded. Containers: {snapshot?.Containers.Count}");
    }

    private void OnDestroy() => _facade?.Dispose();
}
```

---

## Common Operations

### Grant items

```csharp
var result = await _facade.GrantItemsAsync(
    targetContainerId: _facade.CurrentSourceContainerId,
    itemDefinitionId: "wood",
    quantity: 10);

if (!result.Succeeded)
    Debug.LogError($"Grant failed: {result.ErrorMessage}");
```

### Transfer between containers

```csharp
var result = await _facade.TransferItemsAsync(
    sourceContainerId: _facade.CurrentSourceContainerId,
    sourceSlotIndex: 0,
    targetContainerId: _facade.CurrentTargetContainerId,
    quantity: 5);
```

### Quick-store (backpack to chest)

```csharp
var result = await _facade.QuickStoreToTargetContainerAsync(
    sourceContainerId: _facade.CurrentSourceContainerId,
    targetContainerId: _facade.CurrentTargetContainerId,
    options: new UnityQuickStoreOptions
    {
        OnlyMatchExistingStacks = false,
        AllowCreatingNewStacks  = true
    });
```

### Lock / unlock a slot

Protect a slot from automated bulk operations (quick-store, auto-sort). Manual moves still work.

```csharp
// Lock slot 2 so quick-store skips it
await _facade.LockSlotAsync(_facade.CurrentSourceContainerId, slotIndex: 2, lockSlot: true);

// Unlock
await _facade.LockSlotAsync(_facade.CurrentSourceContainerId, slotIndex: 2, lockSlot: false);
```

Locked state is visible on every slot in the snapshot:

```csharp
var snapshot = await _facade.RefreshAsync();
bool isLocked = snapshot.Containers[0].Slots[2].IsLocked;
```

### Split a stack

```csharp
var result = await _facade.SplitStackAsync(
    containerId: _facade.CurrentSourceContainerId,
    sourceSlotIndex: 0,
    amount: 5);

if (result.Succeeded)
    Debug.Log($"Split into slot {result.DestinationSlotIndex}");
```

### Drop items

Remove items from a slot. The server returns the item id and quantity so the client can spawn a world pickup.

```csharp
var result = await _facade.DropItemsAsync(
    containerId: _facade.CurrentSourceContainerId,
    slotIndex: 0,
    amount: 3);

if (result.Succeeded)
    SpawnWorldPickup(result.DroppedItemDefinitionId, result.DroppedQuantity);
```

### Sort container

```csharp
// sortMode: 0 = ByNameAscending, 1 = ByWeightDescending, 2 = ByTagThenName
await _facade.SortContainerAsync(_facade.CurrentSourceContainerId, sortMode: 0);
```

### Crafting

```csharp
// Preview first (shows ingredient availability)
var preview = await _facade.PreviewCraftItemsAsync(
    recipeId: "plank_recipe",
    sourceContainerId: _facade.CurrentSourceContainerId,
    targetContainerId: _facade.CurrentTargetContainerId,
    requestedCraftCount: 3,
    requiredStation: "workbench");

Debug.Log($"Can craft: {preview.CanCraftRequestedCount}, max: {preview.MaximumCraftableCount}");

// Confirm craft
if (preview.CanCraftRequestedCount)
{
    await _facade.CraftItemsAsync(
        recipeId: "plank_recipe",
        sourceContainerId: _facade.CurrentSourceContainerId,
        targetContainerId: _facade.CurrentTargetContainerId,
        craftCount: 3,
        allowPartial: false,
        requiredStation: "workbench");
}
```

---

## Durability

Durability is exposed on each slot in the inventory snapshot:

```csharp
var snapshot = await _facade.RefreshAsync();

foreach (var container in snapshot.Containers)
{
    foreach (var slot in container.Slots)
    {
        if (!slot.IsEmpty && slot.CurrentDurability.HasValue)
        {
            Debug.Log($"Slot {slot.Index}: {slot.ItemDefinitionId} — " +
                      $"durability {slot.CurrentDurability.Value:F1}");
        }
    }
}
```

---

## Weight

Container weight is available on every container snapshot:

```csharp
foreach (var container in snapshot.Containers)
{
    var capacity = container.WeightCapacity.HasValue
        ? container.WeightCapacity.Value.ToString("F1")
        : "∞";

    Debug.Log($"Container {container.ContainerId}: " +
              $"{container.CurrentWeight:F1} / {capacity} kg");
}
```

---

## Player Progression (Recipe Keys)

```csharp
// Unlock a recipe key for a player
await _facade.UnlockRecipeKeyAsync(actorId: "player-123", unlockKey: "tier2_crafting");

// Check progression
var progression = await _facade.GetPlayerProgressionAsync("player-123");
foreach (var entry in progression.UnlockedKeys)
    Debug.Log($"Unlocked: {entry.UnlockKey} at {entry.UnlockedAtUtc}");
```

---

## Session Controller

For longer play sessions, use `UnityInventorySessionController` which wraps the facade and maintains a session state object suitable for binding to UI:

```csharp
var controller = new UnityInventorySessionController(facade);
await controller.StartSessionAsync();
// controller.State.Snapshot    = current inventory
// controller.State.IsConnected = connection health
```

---

## Error Handling

All operation results expose `Succeeded`, `ErrorCode`, and `ErrorMessage`. Common error codes:

| Code | Meaning |
|---|---|
| `ITEM_NOT_FOUND` | `itemDefinitionId` does not exist in the item registry |
| `CONTAINER_NOT_FOUND` | Container ID is not part of this inventory |
| `INSUFFICIENT_ITEMS` | Not enough items in source slot |
| `CONTAINER_FULL` | Target container has no available capacity |
| `WEIGHT_LIMIT_EXCEEDED` | Adding items would exceed the container weight limit |
| `RECIPE_NOT_FOUND` | `recipeId` does not exist |
| `MISSING_INGREDIENTS` | Not enough ingredients to craft |
| `RECIPE_LOCKED` | Actor has not unlocked the required recipe key |
| `UNAUTHORIZED` | API key is missing or invalid |
