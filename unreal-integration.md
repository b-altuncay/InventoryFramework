---
title: Unreal Integration
parent: Engine Integrations
nav_order: 4
description: Add server-authoritative inventory and crafting to Unreal Engine 5 using the InventoryFramework C# NuGet adapter. Item stacking, crafting, slot locking, real-time SignalR events.
---

# Unreal Engine Integration Guide

This guide covers adding InventoryFramework to an Unreal Engine project using C# (via the .NET runtime bridge or a standalone service process).

---

## Requirements

- Unreal Engine 5.x
- A running InventoryFramework server (see [server-configuration.md](server-configuration.md))
- .NET 8 runtime accessible from the Unreal project (service process or plugin bridge)

---

## Installation

```bash
dotnet add package InventoryFramework.UnrealAdapter
```

---

## Quick start

```csharp
using InventoryFramework.UnrealAdapter.Models;
using InventoryFramework.UnrealAdapter.Services;

var config = new UnrealInventoryConfiguration
{
    ServerAddress = "https://your-server:5001",
    ApiKey        = "sk-game-your-key",
    ActorId       = "player-001",
    DefaultSourceContainerCapacity = 30,
    DefaultTargetContainerCapacity = 30
};

var facade = new UnrealInventoryFacade(config);
await facade.ConnectAsync();
await facade.CreateDefaultInventoryAsync();
```

---

## Common operations

### Grant items (admin)

```csharp
var result = await facade.GrantItemsAsync(
    containerId: facade.CurrentSourceContainerId,
    itemDefinitionId: "wood",
    quantity: 10);
```

### Transfer between containers

```csharp
var result = await facade.TransferItemsAsync(
    sourceContainerId: facade.CurrentSourceContainerId,
    sourceSlotIndex: 0,
    targetContainerId: facade.CurrentTargetContainerId,
    quantity: 5);
```

### Quick-store (backpack to chest)

```csharp
var result = await facade.QuickStoreToTargetContainerAsync(
    sourceContainerId: facade.CurrentSourceContainerId,
    targetContainerId: facade.CurrentTargetContainerId);
```

### Lock / unlock a slot

Locked slots are skipped by quick-store and sort, useful for pinning a favourite item.

```csharp
await facade.LockSlotAsync(
    containerId: facade.CurrentSourceContainerId,
    slotIndex: 2,
    lockSlot: true);
```

### Split a stack

```csharp
var result = await facade.SplitStackAsync(
    containerId: facade.CurrentSourceContainerId,
    sourceSlotIndex: 0,
    amount: 5);

Console.WriteLine($"Split into slot {result.DestinationSlotIndex}");
```

### Drop items

```csharp
var result = await facade.DropItemsAsync(
    containerId: facade.CurrentSourceContainerId,
    slotIndex: 0,
    amount: 3);

if (result.Succeeded)
    SpawnWorldPickup(result.DroppedItemDefinitionId, result.DroppedQuantity);
```

### Sort container

```csharp
// 0 = ByNameAscending, 1 = ByWeightDescending, 2 = ByTagThenName
await facade.SortContainerAsync(facade.CurrentSourceContainerId, sortMode: 0);
```

### Crafting

```csharp
var preview = await facade.PreviewCraftItemsAsync(
    recipeId: "plank_recipe",
    sourceContainerId: facade.CurrentSourceContainerId,
    targetContainerId: facade.CurrentTargetContainerId,
    requestedCraftCount: 2,
    requiredStation: "workbench");

if (preview.CanCraftRequestedCount)
{
    await facade.CraftItemsAsync(
        recipeId: "plank_recipe",
        sourceContainerId: facade.CurrentSourceContainerId,
        targetContainerId: facade.CurrentTargetContainerId,
        craftCount: 2,
        allowPartial: false,
        requiredStation: "workbench");
}
```

---

## Reading the snapshot

```csharp
var snapshot = await facade.RefreshAsync();

foreach (var container in snapshot.Containers)
{
    foreach (var slot in container.Slots)
    {
        if (!slot.IsEmpty)
        {
            Console.WriteLine(
                $"Slot {slot.Index}: {slot.ItemDefinitionId} x{slot.Quantity}" +
                (slot.IsLocked ? " [LOCKED]" : "") +
                (!string.IsNullOrEmpty(slot.Restriction) ? $" [restricted:{slot.Restriction}]" : ""));
        }
    }
}
```

---

## Session Controller

For longer play sessions, `UnrealInventorySessionController` wraps the facade and maintains a session state object:

```csharp
var controller = new UnrealInventorySessionController(facade);
await controller.StartSessionAsync();
// controller.State.Snapshot    = current inventory
// controller.State.IsConnected = connection health
```

---

## Error Handling

All operations return a result with `Succeeded`, `ErrorCode`, and `ErrorMessage`:

```csharp
var result = await facade.GrantItemsAsync(containerId, "wood", 10);

if (!result.Succeeded)
    Console.Error.WriteLine($"[{result.ErrorCode}] {result.ErrorMessage}");
```

Common error codes:

| Code | Meaning |
|---|---|
| `item_not_found` | `itemDefinitionId` does not exist |
| `container_not_found` | Container ID is not part of this inventory |
| `inventory_aggregate_not_found` | Inventory does not exist |
| `container_full` | No available capacity |
| `weight_limit_exceeded` | Adding items would exceed the container weight limit |
| `split_stack_failed` | Invalid split (empty slot, amount ≥ quantity, no free slot) |
| `drop_items_failed` | Slot is empty |
| `sort_container_failed` | Invalid sort mode |
| `unauthorized` | API key is missing or invalid |
