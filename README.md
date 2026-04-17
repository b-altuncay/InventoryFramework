# InventoryFramework

[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.SDK?label=SDK)](https://www.nuget.org/packages/InventoryFramework.SDK)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnityAdapter?label=Unity)](https://www.nuget.org/packages/InventoryFramework.UnityAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.GodotAdapter?label=Godot)](https://www.nuget.org/packages/InventoryFramework.GodotAdapter)
[![NuGet](https://img.shields.io/nuget/v/InventoryFramework.UnrealAdapter?label=Unreal)](https://www.nuget.org/packages/InventoryFramework.UnrealAdapter)
[![License](https://img.shields.io/badge/license-Commercial-red)](mailto:mbaltuncay99@gmail.com)

**Server-authoritative inventory and crafting backend for Unity, Godot, and Unreal Engine.**

📖 **[Full documentation →](https://b-altuncay.github.io/InventoryFramework)**

---

## Install

```bash
# Unity
dotnet add package InventoryFramework.UnityAdapter

# Godot
dotnet add package InventoryFramework.GodotAdapter

# Unreal
dotnet add package InventoryFramework.UnrealAdapter

# Plain C#
dotnet add package InventoryFramework.SDK
```

## Quick start

```csharp
var facade = new UnityInventoryFacade(new UnityInventoryConfiguration
{
    ServerAddress = "https://localhost:7289",
    ApiKey        = "sk-game-your-key",
    ActorId       = "player-001"
});

await facade.CreateDefaultInventoryAsync();
await facade.GrantItemsAsync(containerId, "wood", 10);
```

## Licensing

Commercial license required. Contact [mbaltuncay99@gmail.com](mailto:mbaltuncay99@gmail.com)
