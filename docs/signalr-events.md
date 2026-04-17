---
title: SignalR Events
parent: Features
nav_order: 4
---

# SignalR Real-Time Events

InventoryFramework broadcasts domain events to connected clients via SignalR. This lets game clients keep their local inventory view in sync without polling.

---

## Hub endpoint

The hub is mounted at `/inventory-hub` on the same host as the gRPC server.

```
wss://your-server/inventory-hub
```

---

## Subscribing to an inventory

After connecting, a client joins the group for a specific inventory aggregate:

```javascript
// JavaScript / TypeScript example
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/inventory-hub")
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

Fired when items are transferred between two containers (inventory transfer or crafting output).

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

---

## Unity example

```csharp
using Microsoft.AspNetCore.SignalR.Client;

var connection = new HubConnectionBuilder()
    .WithUrl("https://your-server/inventory-hub")
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

```gdscript
extends Node

var connection: SignalRClient  # use a GDScript SignalR library

func _ready():
    connection.connect_to_server("wss://your-server/inventory-hub")
    connection.subscribe("ItemAdded", _on_item_added)
    connection.invoke("SubscribeToInventory", inventory_id)

func _on_item_added(data: Dictionary):
    print("Added %d x %s" % [data["quantityAdded"], data["itemDefinitionId"]])
```

---

## Notes

- Events are dispatched **after** the inventory aggregate is saved. If the save fails, no events are emitted.
- Events are delivered on a best-effort basis. If the client is disconnected at the moment of dispatch, it will miss the event. Use `GetInventoryAsync` to resync after reconnection.
- Unknown event types are silently skipped by the server dispatcher.
