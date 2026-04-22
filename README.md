# InventoryFramework

[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.SDK?label=SDK)](https://www.nuget.org/packages/InventoryFramework.SDK)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnityAdapter?label=Unity)](https://www.nuget.org/packages/InventoryFramework.UnityAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.GodotAdapter?label=Godot)](https://www.nuget.org/packages/InventoryFramework.GodotAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnrealAdapter?label=Unreal)](https://www.nuget.org/packages/InventoryFramework.UnrealAdapter)
[![License](https://img.shields.io/badge/license-Commercial-red)](mailto:mbaltuncay99@gmail.com)

**Server-authoritative inventory and crafting backend for Unity, Godot, and Unreal Engine.**

One server, three engines, zero vendor lock-in. Define items in JSON, run the server, connect from any engine in under 10 minutes.

> Documentation: **[https://b-altuncay.github.io/InventoryFramework](https://b-altuncay.github.io/InventoryFramework)**

---

## Why InventoryFramework?

Most inventory solutions — whether Unity built-in or from the Asset Store — are client-side. That's fine for single-player. The moment you add multiplayer or a second platform, you hit the same wall: inventory data lives on the client, players can tamper with it, and syncing across engines is its own project.

InventoryFramework moves all state to the server:

- **Cheat-resistant by default.** Players cannot modify their inventory outside your game logic — no save file editing, no memory patching.
- **One backend, any engine.** The same server instance talks to Unity, Godot, and Unreal simultaneously. Switch engines mid-project without rewriting inventory logic.
- **Self-hosted.** Player data never touches a third-party service. No per-MAU fees, no external dependencies, no terms-of-service surprises.
- **Drop-in.** Items are plain JSON files. The server ships as a single binary. No backend code required to get started.

---

## Architecture

```mermaid
graph TD
    subgraph Engines["Game Engines"]
        Unity["Unity\nUnityAdapter"]
        Godot["Godot\nGodotAdapter"]
        Unreal["Unreal\nUnrealAdapter"]
        CSharp["Plain C#\n/ Other"]
    end

    SDK["InventoryFramework.SDK\nnetstandard2.1 · gRPC client · retry · mapping"]

    subgraph Server["InventoryFramework Server  (ASP.NET Core · gRPC)"]
        GrpcLayer["gRPC Endpoints"]
        AppLayer["Application Layer\nuse cases · feature gate · license checks"]
        Domain["Domain Layer\ninventory · crafting · trading · progression"]
    end

    subgraph Storage["Persistence"]
        File[("Files\nJSON snapshots")]
        SQLite[("SQLite")]
        SQL[("SQL Server\n/ PostgreSQL")]
    end

    SignalR["SignalR\nreal-time events"]

    Unity & Godot & Unreal & CSharp --> SDK
    SDK -- "gRPC + TLS" --> GrpcLayer
    GrpcLayer --> AppLayer --> Domain
    Domain --> Storage
    Domain --> SignalR
    SignalR -.->|"ItemAdded · ItemRemoved\nSlotMoved · CraftCompleted"| Unity & Godot & Unreal
```

---

## Get started in 10 minutes

### 1. Download the server

Download the latest binary from [GitHub Releases](https://github.com/b-altuncay/InventoryFramework/releases):

```
InventoryFramework-Server-Demo-win-x64.zip    (Windows)
InventoryFramework-Server-Demo-linux-x64.zip  (Linux)
```

Extract and run:

```bash
# Windows
start-demo.bat

# Linux
chmod +x start-demo.sh && ./start-demo.sh
```

The server starts at `https://localhost:7289`.

### 2. Define your items

Create `Data/Items/items.json` next to the binary:

```json
[
  { "id": "wood",  "displayName": "Wood",       "maxStackSize": 50, "weight": 1.0 },
  { "id": "sword", "displayName": "Iron Sword",  "maxStackSize": 1,  "weight": 3.0,
    "hasDurability": true, "maxDurability": 100 }
]
```

### 3. Install the client SDK

```bash
dotnet add package InventoryFramework.UnityAdapter   # Unity
dotnet add package InventoryFramework.GodotAdapter   # Godot
dotnet add package InventoryFramework.UnrealAdapter  # Unreal
dotnet add package InventoryFramework.SDK            # Plain C#
```

### 4. Connect

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

The API is identical across all three adapters — only the facade class name differs.

---

## Feature tiers

| | Demo | Pro | Enterprise |
|---|:---:|:---:|:---:|
| Pre-built server binary (Windows & Linux) | ✓ | ✓ | ✓ |
| File-based persistence | ✓ | ✓ | ✓ |
| SQLite persistence | | ✓ | ✓ |
| SQL Server / PostgreSQL | | | ✓ |
| Unity · Godot · Unreal adapters | ✓ | ✓ | ✓ |
| Full inventory management (CRUD, merge, split, sort, drop) | ✓ | ✓ | ✓ |
| Crafting system | ✓ | ✓ | ✓ |
| Craft preview & recipe availability | | ✓ | ✓ |
| Item affixes (per-instance rolled modifiers) | ✓ | ✓ | ✓ |
| Player progression / recipe unlock keys | | ✓ | ✓ |
| Trading system | | ✓ | ✓ |
| QuickStore | | ✓ | ✓ |
| Real-time events via SignalR | ✓ | ✓ | ✓ |
| Docker + docker-compose | ✓ | ✓ | ✓ |
| Rate limiting & API key auth | ✓ | ✓ | ✓ |
| Max actors | 3 | Unlimited | Unlimited |
| Max items per actor | 50 | Unlimited | Unlimited |
| Max slots per container | 20 | Unlimited | Unlimited |
| Studio license | — | Single studio | Multi-studio |
| Support | — | E-mail | Priority + SLA |
| **Pricing** | Free | [Buy license →](https://inventoryframework-license.mbaltuncay99.workers.dev/activate) | [Contact →](mailto:mbaltuncay99@gmail.com) |

> Enterprise pricing is handled privately — [send an email](mailto:mbaltuncay99@gmail.com) with your studio name and use case.

---

## License activation

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant Gumroad
    participant Worker as License Worker (Cloudflare)
    participant Server as Your Server

    Customer->>Gumroad: Purchase Pro license
    Gumroad-->>Customer: License key via e-mail
    Customer->>Worker: POST /activate  (key + studio name)
    Worker->>Gumroad: Verify license key
    Gumroad-->>Worker: Valid  (tier: Pro v1)
    Worker-->>Customer: license.json  (RSA-signed)
    Customer->>Server: Place license.json next to binary
    Server->>Server: Validate signature on startup
    Server-->>Customer: License accepted — Pro features unlocked
```

---

## Item affixes

Affixes are per-instance modifiers rolled onto individual item stacks — fire damage, move speed, crit chance. They are defined in JSON and flow from server to SDK to engine adapter with no extra mapping code.

```json
[
  { "id": "fire_damage", "displayName": "Fire Damage", "statKey": "damage", "minValue": 10, "maxValue": 50 }
]
```

```csharp
await facade.GrantItemsAsync(containerId, "sword", 1, affixes: new[]
{
    new ItemAffixRequest { AffixDefinitionId = "fire_damage", Value = 38.5f }
});
```

---

## Crafting

```mermaid
flowchart LR
    JSON["recipes.json"] --> Registry["Recipe Registry\n(loaded at startup)"]

    subgraph CraftFlow["Craft request"]
        direction TB
        Check["Check ingredients\n+ unlock keys"] --> Consume["Consume inputs"]
        Consume --> Produce["Place outputs"]
        Produce --> Events["Emit CraftCompleted\nvia SignalR"]
    end

    Registry --> CraftFlow

    subgraph ProOnly["Pro tier only"]
        Preview["PreviewCraft\n(no state change)"]
        Available["GetAvailableRecipes\n(filter by inventory)"]
    end
```

Recipes support multiple outputs, partial crafting, and per-player unlock keys. `PreviewCraft` lets players check what they can build without consuming ingredients.

---

## Inventory operations

### Slot locking

Locks a slot so it is skipped by QuickStore and auto-sort. Manual moves and crafting still work on locked slots.

```csharp
await facade.LockSlotAsync(containerId, slotIndex: 2, lockSlot: true);
```

### Stack splitting

Splits a stack into two within the same container.

```csharp
var result = await facade.SplitStackAsync(containerId, sourceSlotIndex: 0, amount: 5);
// result.DestinationSlotIndex — where the split portion landed
```

### Dropping items

Removes items from a slot and returns the definition id and quantity, so the game can spawn a world pickup.

```csharp
var result = await facade.DropItemsAsync(containerId, slotIndex: 0, amount: 3);
// result.DroppedItemDefinitionId — use this to spawn the world object
```

### Auto-sort

Sorts all unlocked slots in a container. Locked slots stay in place and the sort routes around them.

```csharp
await facade.SortContainerAsync(containerId, sortMode: 0);
// 0 = ByNameAscending  1 = ByWeightDescending  2 = ByTagThenName
```

### Slot restrictions

Equipment slots can be restricted to a specific item tag. Mismatched items are rejected during transfers and sorting.

```json
{ "id": "helmet", "displayName": "Iron Helmet", "maxStackSize": 1, "weight": 3.0, "tags": ["helmet"] }
```

---

## gRPC endpoints

| RPC | Description |
|---|---|
| `CreateInventory` | Creates a new inventory aggregate |
| `GetInventory` | Returns full state — affixes, `IsLocked`, `Restriction` per slot |
| `GrantItems` | Adds items with optional affixes (admin) |
| `TransferItems` | Moves items between containers |
| `QuickStoreItems` | Bulk-transfers matching stacks, skips locked slots |
| `CraftItems` | Executes a recipe |
| `PreviewCraftItems` | Checks craftability without modifying state |
| `BrowseRecipes` | Lists recipes by category / station |
| `GetAvailableRecipes` | Returns craftable recipes given current inventory |
| `UnlockRecipeKey` / `RevokeRecipeKey` | Manage per-player progression keys |
| `GetPlayerProgression` | Returns all unlocked keys for an actor |
| `TradeItems` | Cross-inventory item transfer (admin) |
| `LockSlot` | Locks or unlocks a specific slot |
| `SplitStack` | Splits a stack into two slots |
| `DropItems` | Removes and discards items from a slot |
| `SortContainer` | Sorts unlocked slots by name, weight, or tag group |

All requests require an `x-api-key` header. Admin endpoints additionally require `IsAdmin: true`.

---

## Documentation

| | |
|---|---|
| [Getting Started](https://b-altuncay.github.io/InventoryFramework/deployment.html) | Download, run, connect |
| [Server Configuration](https://b-altuncay.github.io/InventoryFramework/server-configuration.html) | API keys, paths, persistence backends |
| [Item Definitions](https://b-altuncay.github.io/InventoryFramework/item-definitions.html) | JSON schema, durability, weight, tags |
| [Crafting & Recipes](https://b-altuncay.github.io/InventoryFramework/crafting.html) | Recipe format, partial crafting, unlock keys |
| [Player Progression](https://b-altuncay.github.io/InventoryFramework/progression.html) | Recipe unlock keys, grant and revoke |
| [Persistence](https://b-altuncay.github.io/InventoryFramework/persistence.html) | File, SQLite, SQL Server, PostgreSQL |
| [SDK Usage](https://b-altuncay.github.io/InventoryFramework/sdk-usage.html) | Plain C# client, DI, error handling |
| [Unity Integration](https://b-altuncay.github.io/InventoryFramework/unity-integration.html) | Installation, MonoBehaviour setup |
| [Godot Integration](https://b-altuncay.github.io/InventoryFramework/godot-integration.html) | Installation, Node setup |
| [Unreal Integration](https://b-altuncay.github.io/InventoryFramework/unreal-integration.html) | Installation, facade setup |
| [SignalR Events](https://b-altuncay.github.io/InventoryFramework/signalr-events.html) | Real-time inventory change notifications |

---

## License

Commercial — one-time purchase per major version. A single studio license covers all projects within your studio.

[**Buy Pro →**](https://inventoryframework-license.mbaltuncay99.workers.dev/activate) · [Enterprise inquiry →](mailto:mbaltuncay99@gmail.com)
