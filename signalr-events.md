---
title: SignalR Events
parent: Features
nav_order: 4
---

# SignalR Real-Time Events

InventoryFramework broadcasts domain events to connected clients via SignalR. This lets game clients keep their local inventory view in sync without polling.

---

## Hub endpoint

The hub is mounted at `/hubs/inventory` on the same host as the gRPC server.

```
wss://your-server/hubs/inventory
```

---

## Subscribing to an inventory

After connecting, a client joins the group for a specific inventory aggregate:

```javascript
// JavaScript / TypeScript example
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/inventory")
    .build();

await connection.start();

// Subscribe to events for a specific inventory
await connection.invoke("SubscribeToInventory", "inv-player-123");

// Unsubscribe when done
await connection.invoke("UnsubscribeFromInventory", "inv-player-123");
```

The server adds the connection to a group named `inventory:{aggregateId}`. All events for that aggregate are broadcast to the group.

---

## Events

### `ItemAdded`

Fired when items are placed into a container slot.

```json
{
  "inventoryAggregateId": "inv-player-123",
  "containerId": "container-backpack-guid",
  "slotIndex": 2,
  "itemDefinitionId": "wood",
  "quantityAdded": 5,
  "occurredAtUtc": "2026-04-15T10:00:00Z"
}
```

### `ItemRemoved`

Fired when items are removed from a container slot.

```json
{
  "inventoryAggregateId": "inv-player-123",
  "containerId": "container-backpack-guid",
  "slotIndex": 2,
  "itemDefinitionId": "wood",
  "quantityRemoved": 3,
  "occurredAtUtc": "2026-04-15T10:00:01Z"
}
```

### `SlotMoved`

Fired when items are moved from one slot to another within the same container.

```json
{
  "inventoryAggregateId": "inv-player-123",
  "containerId": "container-backpack-guid",
  "fromSlotIndex": 0,
  "toSlotIndex": 4,
  "itemDefinitionId": "sword",
  "quantityMoved": 1,
  "occurredAtUtc": "2026-04-15T10:00:02Z"
}
```

### `TransferCompleted`

Fired when items are transferred between two containers.

```json
{
  "inventoryAggregateId": "inv-player-123",
  "sourceContainerId": "container-chest-guid",
  "sourceSlotIndex": 1,
  "targetContainerId": "container-backpack-guid",
  "itemDefinitionId": "plank",
  "quantityTransferred": 2,
  "occurredAtUtc": "2026-04-15T10:00:03Z"
}
```

### `CraftCompleted`

Fired once after a craft operation completes and all outputs have been placed in the target container. Emitted in addition to the per-slot `ItemRemoved` and `ItemAdded` events, so clients can react to the craft as a single high-level action.

```json
{
  "inventoryAggregateId": "inv-player-123",
  "recipeId": "plank_recipe",
  "actualCraftCount": 2,
  "sourceContainerId": "container-backpack-guid",
  "targetContainerId": "container-backpack-guid",
  "occurredAtUtc": "2026-04-15T10:00:04Z"
}
```

---

## Unity example

```csharp
using Microsoft.AspNetCore.SignalR.Client;

var connection = new HubConnectionBuilder()
    .WithUrl("https://your-server/hubs/inventory")
    .WithAutomaticReconnect()
    .Build();

connection.On<ItemAddedNotification>("ItemAdded", n =>
{
    Debug.Log($"Added {n.QuantityAdded}x {n.ItemDefinitionId} to slot {n.SlotIndex}");
});

await connection.StartAsync();
await connection.InvokeAsync("SubscribeToInventory", inventoryId);
```

---

## Godot example

Godot 4 C# projects can use the official `Microsoft.AspNetCore.SignalR.Client` NuGet package directly, no third-party library needed.

Add to your `.csproj`:

```xml
<PackageReference Include="Microsoft.AspNetCore.SignalR.Client" Version="8.*" />
```

Then connect the same way as the Unity example:

```csharp
using Microsoft.AspNetCore.SignalR.Client;
using Godot;

public partial class InventorySignalR : Node
{
    private HubConnection _connection;

    public override async void _Ready()
    {
        _connection = new HubConnectionBuilder()
            .WithUrl("https://your-server/hubs/inventory")
            .WithAutomaticReconnect()
            .Build();

        _connection.On<ItemAddedNotification>("ItemAdded", n =>
        {
            GD.Print($"Added {n.QuantityAdded}x {n.ItemDefinitionId} to slot {n.SlotIndex}");
        });

        _connection.On<CraftCompletedNotification>("CraftCompleted", n =>
        {
            GD.Print($"Crafted {n.ActualCraftCount}x {n.RecipeId}");
        });

        await _connection.StartAsync();
        await _connection.InvokeAsync("SubscribeToInventory", inventoryId);
    }

    public override async void _ExitTree()
    {
        if (_connection != null)
            await _connection.DisposeAsync();
    }
}
```

---

## Notes

- Events are dispatched **after** the inventory aggregate is saved. If the save fails, no events are emitted.
- Events are delivered on a best-effort basis. If the client is disconnected at the moment of dispatch, it will miss the event. Use `GetInventoryAsync` to resync after reconnection.
- Unknown event types are silently skipped by the server dispatcher.
