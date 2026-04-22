# InventoryFramework

[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.SDK?label=SDK)](https://www.nuget.org/packages/InventoryFramework.SDK)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnityAdapter?label=Unity)](https://www.nuget.org/packages/InventoryFramework.UnityAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.GodotAdapter?label=Godot)](https://www.nuget.org/packages/InventoryFramework.GodotAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnrealAdapter?label=Unreal)](https://www.nuget.org/packages/InventoryFramework.UnrealAdapter)
[![License](https://img.shields.io/badge/license-Commercial-red)](mailto:mbaltuncay99@gmail.com)

**Server-authoritative inventory and crafting backend for Unity, Godot, and Unreal Engine.**

One server, three engines, zero vendor lock-in. Define your items in JSON, run the server, connect from any engine in under 10 minutes.

> Full documentation: **[https://b-altuncay.github.io/InventoryFramework](https://b-altuncay.github.io/InventoryFramework)**

---

## Why InventoryFramework?

Unity's inventory solutions — whether built-in or from the Asset Store — are client-side. That works fine for single-player. The moment you add multiplayer, leaderboards, or a second platform, you hit the same wall every time: the data lives on the client, players can modify it, and syncing it across engines is a custom project of its own.

InventoryFramework takes a different approach:

- **All state lives on your server.** Players cannot modify their own inventory outside your game logic. No client-side hacks, no save file editing.
- **One backend, any engine.** The same server talks to Unity, Godot, and Unreal simultaneously. Switch engines mid-project or ship on multiple platforms without rewriting your inventory logic.
- **Self-hosted.** Your player data never touches a third-party service. No per-MAU pricing, no outage dependencies, no terms-of-service changes that break your game.
- **Drop-in ready.** Item definitions are plain JSON files. The server runs with `dotnet run`. You don't need to write a single line of backend code to get started.

---

## Get it running in 10 minutes

### 1. Install via NuGet

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

### 2. Define your items

Create `Data/Items/items.json`:

```json
[
  { "id": "wood",  "displayName": "Wood",      "maxStackSize": 50, "weight": 1.0 },
  { "id": "sword", "displayName": "Iron Sword", "maxStackSize": 1,  "weight": 3.0,
    "hasDurability": true, "maxDurability": 100 }
]
```

### 3. Connect and add items

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

Same lines work on Godot and Unreal — just swap the facade class name.

---

## Feature tiers

| | Demo | Pro | Enterprise |
|---|:---:|:---:|:---:|
| Binary (server + SDK + adapters) | ✓ | ✓ | ✓ |
| File-based persistence | ✓ | ✓ | ✓ |
| SQLite persistence | | ✓ | ✓ |
| SQL Server / PostgreSQL | | | ✓ |
| Unity · Godot · Unreal adapters | ✓ | ✓ | ✓ |
| Item affixes (rolled modifiers) | ✓ | ✓ | ✓ |
| Player progression / recipe unlock keys | | ✓ | ✓ |
| Real-time events via SignalR | ✓ | ✓ | ✓ |
| Max slots per container | 20 | Unlimited | Unlimited |
| Studio license | Single project | Single studio | Multi-studio |
| Support | Community | E-mail | Priority + SLA |
| **Pricing** | Free | [Buy license →](https://inventoryframework-license.mbaltuncay99.workers.dev/activate) | [Contact us →](mailto:mbaltuncay99@gmail.com) |

> **Enterprise** pricing is handled privately — [send an email](mailto:mbaltuncay99@gmail.com) with your studio name and use case.

---

## Documentation

| | |
|---|---|
| [Server Configuration](docs/server-configuration.md) | API keys, paths, persistence backends |
| [Item Definitions](docs/item-definitions.md) | JSON schema, durability, weight, tags |
| [Crafting & Recipes](docs/crafting.md) | Recipe format, partial crafting, unlock keys |
| [Player Progression](docs/progression.md) | Recipe unlock keys, grant and revoke |
| [Persistence](docs/persistence.md) | File, SQLite, SQL Server, PostgreSQL |
| [SDK Usage](docs/sdk-usage.md) | Plain C# client, DI, error handling |
| [Unity Integration](docs/unity-integration.md) | Installation, MonoBehaviour setup |
| [Godot Integration](docs/godot-integration.md) | Installation, Node setup |
| [Unreal Integration](docs/unreal-integration.md) | Installation, facade setup |
| [SignalR Events](docs/signalr-events.md) | Real-time inventory change notifications |
| [Testing Guide](docs/testing.md) | Running tests, test projects, writing new tests |
