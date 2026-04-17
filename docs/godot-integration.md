---
title: Godot Integration
parent: Engine Integrations
nav_order: 3
---

# Godot Integration Guide

This guide covers adding InventoryFramework to a Godot 4 project using C#.

---

## Requirements

- Godot 4.x with .NET (Mono) enabled
- .NET 6 or .NET 7 (Godot 4.x uses .NET 6 by default)
- A running InventoryFramework server (see [Server Configuration](server-configuration.md))

---

## Installation

### Add the NuGet Package

In your Godot project's `.csproj`, add:

```xml
<ItemGroup>
  <Reference Include="InventoryFramework.SDK">
    <HintPath>addons/InventoryFramework/InventoryFramework.SDK.dll</HintPath>
  </Reference>
  <Reference Include="InventoryFramework.GodotAdapter">
    <HintPath>addons/InventoryFramework/InventoryFramework.GodotAdapter.dll</HintPath>
  </Reference>
</ItemGroup>
```

Copy the DLLs from the `dist/godot/` folder to `res://addons/InventoryFramework/`.

---

## Autoload Setup

Create an autoload node `InventoryManager.cs`:

```csharp
using Godot;
using InventoryFramework.GodotAdapter.Models;
using InventoryFramework.GodotAdapter.Services;
using System.Threading.Tasks;

public partial class InventoryManager : Node
{
    public GodotInventoryFacade Facade { get; private set; }
    public GodotInventorySnapshot Snapshot { get; private set; }

    public override async void _Ready()
    {
        var config = new GodotInventoryConfiguration
        {
            ServerAddress = "https://localhost:7289",
            ActorId       = OS.GetUniqueId(),
            ApiKey        = "sk-game-your-key"
        };

        Facade = new GodotInventoryFacade(config);

        var created = await Facade.CreateDefaultInventoryAsync();
        if (!created)
        {
            GD.PrintErr("Failed to create inventory.");
            return;
        }

        Snapshot = await Facade.RefreshAsync();
        GD.Print($"Inventory loaded. Aggregate ID: {Facade.CurrentInventoryAggregateId}");
    }

    protected override void Dispose(bool disposing)
    {
        Facade?.Dispose();
        base.Dispose(disposing);
    }
}
```

Register `InventoryManager` as an autoload in **Project → Project Settings → Autoload**.

---

## Common Operations

### Grant items

```csharp
var result = await manager.Facade.GrantItemsAsync(
    targetContainerId: "source-container-id",
    itemDefinitionId: "wood",
    quantity: 10);

if (!result.Succeeded)
    GD.PrintErr($"Grant failed: {result.ErrorMessage}");
```

### Transfer between containers

```csharp
await manager.Facade.TransferItemsAsync(
    sourceContainerId: "src-id",
    sourceSlotIndex: 0,
    targetContainerId: "dst-id",
    quantity: 5);
```

### Quick-store

```csharp
await manager.Facade.QuickStoreToTargetContainerAsync(
    sourceContainerId: "src-id",
    targetContainerId: "dst-id",
    options: new GodotQuickStoreOptions
    {
        OnlyMatchExistingStacks = false,
        AllowCreatingNewStacks  = true
    });
```

### Lock / unlock a slot

```csharp
await manager.Facade.LockSlotAsync("src-id", slotIndex: 2, lockSlot: true);
// Slot 2 is now skipped by QuickStore and SortContainer

// Check lock state from snapshot
bool isLocked = snapshot.Containers[0].Slots[2].IsLocked;
```

### Split a stack

```csharp
var result = await manager.Facade.SplitStackAsync(
    containerId: "src-id",
    sourceSlotIndex: 0,
    amount: 5);

GD.Print($"Split landed in slot {result.DestinationSlotIndex}");
```

### Drop items

```csharp
var result = await manager.Facade.DropItemsAsync(
    containerId: "src-id",
    slotIndex: 0,
    amount: 3);

if (result.Succeeded)
    SpawnPickupNode(result.DroppedItemDefinitionId, result.DroppedQuantity);
```

### Sort container

```csharp
// sortMode: 0 = ByNameAscending, 1 = ByWeightDescending, 2 = ByTagThenName
await manager.Facade.SortContainerAsync("src-id", sortMode: 0);
```

### Crafting

```csharp
var preview = await manager.Facade.PreviewCraftItemsAsync(
    recipeId: "plank_recipe",
    sourceContainerId: "src-id",
    targetContainerId: "dst-id",
    requestedCraftCount: 2,
    requiredStation: "workbench");

GD.Print($"Max craftable: {preview.MaximumCraftableCount}");

if (preview.CanCraftRequestedCount)
{
    await manager.Facade.CraftItemsAsync(
        recipeId: "plank_recipe",
        sourceContainerId: "src-id",
        targetContainerId: "dst-id",
        craftCount: 2,
        allowPartial: false,
        requiredStation: "workbench");
}
```

---

## Error Codes

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
