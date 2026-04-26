<div align="center">
  <img src="banner.svg" alt="InventoryFramework" width="100%">
</div>

<br>

<div align="center">

[![SDK](https://img.shields.io/nuget/v/InventoryFramework.SDK?label=SDK&color=059669)](https://www.nuget.org/packages/InventoryFramework.SDK)
[![Unity](https://img.shields.io/nuget/v/InventoryFramework.UnityAdapter?label=Unity&color=059669)](https://www.nuget.org/packages/InventoryFramework.UnityAdapter)
[![Godot](https://img.shields.io/nuget/v/InventoryFramework.GodotAdapter?label=Godot&color=059669)](https://www.nuget.org/packages/InventoryFramework.GodotAdapter)
[![Unreal](https://img.shields.io/nuget/v/InventoryFramework.UnrealAdapter?label=Unreal&color=059669)](https://www.nuget.org/packages/InventoryFramework.UnrealAdapter)
[![Release](https://img.shields.io/github/v/release/b-altuncay/InventoryFramework?label=release&color=0F172A)](https://github.com/b-altuncay/InventoryFramework/releases/latest)
[![License](https://img.shields.io/badge/license-Commercial-dc2626)](mailto:mbaltuncay99@gmail.com)

**[Documentation](https://b-altuncay.github.io/InventoryFramework)** · **[Get a license](https://mbaltuncay.gumroad.com/l/qyeyym)** · **[Discussions](https://github.com/b-altuncay/InventoryFramework/discussions)**

</div>

---

## Why InventoryFramework?

Unity's inventory solutions — and most game engine equivalents — are client-side. That works fine for single-player. The moment you add multiplayer, leaderboards, or a second platform, you hit the same wall: the data lives on the client, players can modify it, and syncing it across engines is a custom project of its own.

InventoryFramework takes a different approach:

- **All state lives on your server.** Players cannot modify their own inventory outside your game logic. No client-side hacks, no save file editing.
- **One backend, any engine.** The same server talks to Unity, Godot, and Unreal simultaneously. Switch engines mid-project or ship on multiple platforms without rewriting your inventory logic.
- **Self-hosted.** Your player data never touches a third-party service. No per-MAU pricing, no outage dependencies, no vendor terms changes that break your game.
- **Drop-in ready.** Item definitions are plain JSON. The server runs with `dotnet run`. No backend code required to get started.

<br>

<div align="center">
  <img src="architecture.svg" alt="Architecture Overview" width="100%">
</div>

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

Same code works on Godot and Unreal — just swap the facade class name.

---

## Feature tiers

| | Demo | Pro | Enterprise |
|---|:---:|:---:|:---:|
| Server binary + SDK + engine adapters | ✓ | ✓ | ✓ |
| File-based persistence | ✓ | ✓ | ✓ |
| SQLite persistence | | ✓ | ✓ |
| SQL Server / PostgreSQL | | | ✓ |
| Unity · Godot · Unreal adapters | ✓ | ✓ | ✓ |
| Item affixes (rolled modifiers) | ✓ | ✓ | ✓ |
| Player progression & recipe unlock keys | | ✓ | ✓ |
| Real-time events via SignalR | ✓ | ✓ | ✓ |
| Max slots per container | 20 | Unlimited | Unlimited |
| Studio license | Single project | Single studio | Multi-studio |
| Support | Community | E-mail | Priority + SLA |
| **Pricing** | **Free** | **[$80 one-time →](https://mbaltuncay.gumroad.com/l/qyeyym)** | **[Contact →](mailto:mbaltuncay99@gmail.com)** |

> **Enterprise** pricing is handled privately — [send an email](mailto:mbaltuncay99@gmail.com) with your studio name and use case.

---

## Documentation

| | |
|---|---|
| [Getting Started](getting-started.md) | First server run, item definitions, first request |
| [Server Configuration](server-configuration.md) | API keys, paths, persistence backends |
| [Item Definitions](item-definitions.md) | JSON schema, durability, weight, tags |
| [Crafting & Recipes](crafting.md) | Recipe format, partial crafting, unlock keys |
| [Player Progression](progression.md) | Recipe unlock keys, grant and revoke |
| [Persistence](persistence.md) | File, SQLite, SQL Server, PostgreSQL |
| [Deployment](deployment.md) | Docker, TLS, reverse proxy, env vars |
| [SDK Usage](sdk-usage.md) | Plain C# client, DI, error handling |
| [Unity Integration](unity-integration.md) | Installation, MonoBehaviour setup |
| [Godot Integration](godot-integration.md) | Installation, Node setup |
| [Unreal Integration](unreal-integration.md) | Installation, facade setup |
| [SignalR Events](signalr-events.md) | Real-time inventory change notifications |
| [Testing Guide](testing.md) | Running tests, test projects, writing new tests |

---

<div align="center">

Questions? Open a thread in **[Discussions](https://github.com/b-altuncay/InventoryFramework/discussions)** or email [mbaltuncay99@gmail.com](mailto:mbaltuncay99@gmail.com).

</div>
