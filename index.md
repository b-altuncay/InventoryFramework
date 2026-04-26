---
title: Home
nav_order: 1
description: Self-hosted inventory and crafting backend for Unity, Godot, and Unreal Engine. Server-authoritative .NET server, SQLite/PostgreSQL, real-time SignalR events. Free demo available.
---

# InventoryFramework

[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.SDK?label=SDK)](https://www.nuget.org/packages/InventoryFramework.SDK)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnityAdapter?label=Unity)](https://www.nuget.org/packages/InventoryFramework.UnityAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.GodotAdapter?label=Godot)](https://www.nuget.org/packages/InventoryFramework.GodotAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnrealAdapter?label=Unreal)](https://www.nuget.org/packages/InventoryFramework.UnrealAdapter)
[![License](https://img.shields.io/badge/license-Commercial-red)](mailto:mbaltuncay99@gmail.com)

**Server-authoritative inventory and crafting backend for Unity, Godot, and Unreal Engine.**

Item definitions are JSON files. The server starts with `dotnet run`. Connect from Unity, Godot, or Unreal via NuGet in about 10 minutes.

---

## Why InventoryFramework?

Most inventory plugins for game engines keep state on the client. That works for single-player, but once you add multiplayer, leaderboards, or a second target platform, you run into the same problems: players can tamper with client-side data, keeping two engines in sync requires custom code, and swapping engines mid-project means rewriting inventory logic from scratch.

InventoryFramework runs all inventory and crafting logic on a server you control:

- Item state is stored server-side, so players can't tamper with it. No memory editing, no save file manipulation.
- The same server talks to Unity, Godot, and Unreal. Add an engine later or switch mid-project without touching the backend.
- You self-host it. Player data stays on your own servers, no per-MAU costs or third-party dependencies.
- Getting started takes about 10 minutes. Item definitions are JSON files, the server starts with `dotnet run`.

---

## Get it running in 10 minutes

### 1. Start the server

```bash
git clone https://github.com/b-altuncay/InventoryFramework
cd InventoryFramework/InventoryFramework.Server
dotnet run
```

The server starts at `https://localhost:7289`.

### 2. Define your items

Create `Data/Items/items.json`:

```json
[
  { "id": "wood",  "displayName": "Wood",      "maxStackSize": 50, "weight": 1.0 },
  { "id": "sword", "displayName": "Iron Sword", "maxStackSize": 1,  "weight": 3.0,
    "hasDurability": true, "maxDurability": 100 }
]
```

### 3. Install the adapter for your engine

```bash
# Unity
dotnet add package InventoryFramework.UnityAdapter

# Godot
dotnet add package InventoryFramework.GodotAdapter

# Unreal
dotnet add package InventoryFramework.UnrealAdapter

# Plain C# / server-to-server
dotnet add package InventoryFramework.SDK
```

### 4. Connect and add items

```csharp
var facade = new UnityInventoryFacade(new UnityInventoryConfiguration
{
    ServerAddress = "https://localhost:7289",
    ApiKey        = "sk-game-your-key",
    ActorId       = "player-001"
});

await facade.CreateDefaultInventoryAsync();
await facade.GrantItemsAsync(containerId, "wood", 10);

var snapshot = await facade.RefreshAsync();
Debug.Log($"Items in backpack: {snapshot.Containers[0].Slots.Count(s => !s.IsEmpty)}");
```

The same code works on Godot and Unreal; just swap the facade class name.

---

## Feature tiers

| | Demo | Pro | Enterprise |
|---|:---:|:---:|:---:|
| Binary (server + SDK + adapters) | ✓ | ✓ | ✓ |
| File-based persistence | ✓ | ✓ | ✓ |
| SQLite persistence | | ✓ | ✓ |
| SQL Server / PostgreSQL | | | ✓ |
| Unity, Godot, Unreal adapters | ✓ | ✓ | ✓ |
| Item affixes (rolled modifiers) | ✓ | ✓ | ✓ |
| Player progression / recipe unlock keys | | ✓ | ✓ |
| Real-time events via SignalR | ✓ | ✓ | ✓ |
| Max slots per container | 20 | Unlimited | Unlimited |
| Studio license | Single project | Single studio | Multi-studio |
| Support | Community | E-mail | Priority + SLA |
| **Pricing** | Free | [$80 one-time](https://mbaltuncay.gumroad.com/l/qyeyym) | [Contact us](mailto:mbaltuncay99@gmail.com) |

> Enterprise pricing is handled privately. [Email us](mailto:mbaltuncay99@gmail.com) with your studio name and use case.

---

## gRPC endpoints

| RPC | Description |
|---|---|
| `CreateInventory` | Creates a new inventory aggregate |
| `GetInventory` | Returns full inventory state |
| `GrantItems` | Adds items with optional affixes (admin) |
| `TransferItems` | Moves items between containers |
| `QuickStoreItems` | Bulk-transfers matching stacks (skips locked slots) |
| `LockSlot` | Locks or unlocks a specific slot |
| `SplitStack` | Splits a stack into two slots |
| `DropItems` | Removes and discards items from a slot |
| `SortContainer` | Sorts unlocked slots by name, weight, or tag group |
| `CraftItems` | Executes a recipe |
| `PreviewCraftItems` | Checks craftability without modifying state |
| `BrowseRecipes` | Lists recipes by category / station |
| `GetAvailableRecipes` | Returns craftable recipes given current inventory |
| `UnlockRecipeKey` / `RevokeRecipeKey` | Manage player progression keys |
| `TradeItems` | Cross-inventory item transfer (admin) |

All requests require an `x-api-key` header.
