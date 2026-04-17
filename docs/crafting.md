---
title: Crafting & Recipes
parent: Features
nav_order: 1
---

# Crafting & Recipes

InventoryFramework provides a server-authoritative crafting system. All recipe validation runs on the server — clients cannot spoof ingredients or outputs.

---

## Recipe Definition Format

Recipe files are JSON arrays. By default the server loads every `*.json` file under the directory specified by `RecipeDefinitionsRootPath`.

```json
[
  {
    "id": "plank_recipe",
    "displayName": "Wood Plank",
    "craftingCategory": "woodworking",
    "requiredStation": "workbench",
    "ingredients": [
      { "itemDefinitionId": "wood", "quantity": 2 }
    ],
    "outputs": [
      { "itemDefinitionId": "plank", "quantity": 1 }
    ]
  },
  {
    "id": "sword_recipe",
    "displayName": "Iron Sword",
    "craftingCategory": "smithing",
    "requiredStation": "forge",
    "isHiddenUntilUnlocked": true,
    "requiredUnlockKeys": ["blacksmith_tier2"],
    "ingredients": [
      { "itemDefinitionId": "iron_ingot", "quantity": 3 },
      { "itemDefinitionId": "wood",       "quantity": 1 }
    ],
    "outputs": [
      { "itemDefinitionId": "sword", "quantity": 1 }
    ]
  }
]
```

### Required fields

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique recipe identifier. |
| `displayName` | string | Human-readable name shown in UI. |
| `ingredients` | array | At least one ingredient required. |
| `outputs` | array | At least one output required. |

### Optional fields

| Field | Type | Default | Description |
|---|---|---|---|
| `craftingCategory` | string | `""` | Groups recipes in the browser (e.g. `"smithing"`). |
| `requiredStation` | string | `""` | Station the player must be at to craft (e.g. `"forge"`). |
| `isHiddenUntilUnlocked` | bool | `false` | Hides the recipe from the browser until the player has all `requiredUnlockKeys`. |
| `requiredUnlockKeys` | array | `[]` | Unlock keys the actor must own to see and use this recipe. See [Player Progression](progression.md). |

---

## Crafting API Operations

All crafting operations are accessed through `IInventoryClient` in the SDK.

### Browse recipes

Returns a flat list of all recipes the actor is allowed to see.

```csharp
var result = await client.BrowseRecipesAsync(
    inventoryId: "inv-player-123",
    category: "woodworking"); // optional filter
```

### Get recipe details

Returns full ingredient and output information for a single recipe.

```csharp
var result = await client.GetRecipeDetailsAsync(
    inventoryId: "inv-player-123",
    recipeId: "plank_recipe");
```

### Check available recipes (Pro+)

Returns all craftable recipes for the actor's current inventory, including per-ingredient availability.

```csharp
var result = await client.GetAvailableRecipesAsync(
    inventoryId: "inv-player-123",
    requestedCraftCount: 1);
```

Each `AvailableRecipeItemResult` reports:
- `CanCraftRequestedCount` — whether the actor has enough ingredients for `requestedCraftCount` runs.
- `MaximumCraftableCount` — maximum number of times this recipe could be crafted with current stock.
- `Ingredients` — per-ingredient `RequiredQuantity`, `AvailableQuantity`, `MissingQuantity`, `IsSatisfied`.

### Preview craft (Pro+)

Simulates what would happen without committing any changes.

```csharp
var result = await client.PreviewCraftItemsAsync(
    inventoryId: "inv-player-123",
    recipeId: "plank_recipe",
    craftCount: 2,
    sourceContainerId: "container-backpack",
    targetContainerId: "container-backpack");
```

### Craft items

Commits the craft. Ingredients are consumed and outputs placed into the target container.

```csharp
var result = await client.CraftItemsAsync(
    inventoryId: "inv-player-123",
    recipeId: "plank_recipe",
    craftCount: 1,
    sourceContainerId: "container-backpack",
    targetContainerId: "container-backpack");
```

`result.Succeeded` is `false` if ingredients are missing, the station is wrong, the feature is disabled, or the actor lacks the required unlock keys.

---

## Station Enforcement

If a recipe has a non-empty `requiredStation`, the server calls `IRecipeStationPolicy.IsAtRequiredStation(actorContext, recipe)`. The default implementation (`DefaultRecipeStationPolicy`) always returns `true` — override it via DI to add real station proximity checks.

---

## Visibility and Access

- Recipes with `isHiddenUntilUnlocked: true` are hidden from `BrowseRecipes` and blocked from crafting until the actor holds every key in `requiredUnlockKeys`.
- The default `IRecipeVisibilityPolicy` (`DefaultRecipeVisibilityPolicy`) checks the actor's `PlayerProgression` unlock keys.
- Override `IRecipeUnlockEvaluator` via DI to customise access logic.

---

## Feature Flags

Crafting operations are individually gated by feature flags (see [Server Configuration](server-configuration.md)):

| Operation | Feature key |
|---|---|
| BrowseRecipes | `RecipeBrowser` |
| GetRecipeDetails | `RecipeDetails` |
| GetAvailableRecipes | `RecipeAvailability` (Pro+) |
| PreviewCraftItems | `CraftPreview` (Pro+) |
| CraftItems | `Crafting` |
