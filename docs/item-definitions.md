---
title: Item Definitions
parent: Getting Started
nav_order: 2
---

# Item Definitions

Item definitions are the static metadata for every item in your game. They are loaded from JSON files on server startup and are never modified at runtime.

---

## File Format

Each JSON file may contain a single object or an array of objects. All files matching the configured `SearchPattern` inside `ItemDefinitionsRootPath` are loaded (recursively if configured).

### Full Schema

```json
{
  "id":            "sword",
  "displayName":   "Iron Sword",
  "maxStackSize":  1,
  "weight":        3.0,
  "hasDurability": true,
  "maxDurability": 100.0,
  "tags":          ["weapon", "melee", "craftable"]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique identifier. Case-insensitive for lookups. |
| `displayName` | string | Yes | Human-readable name shown in UI. |
| `maxStackSize` | integer > 0 | Yes | Maximum quantity per inventory slot. |
| `weight` | float ≥ 0 | Yes | Item weight per unit. Used for container weight limits. |
| `hasDurability` | bool | No | Whether individual stacks track durability. Default: `false`. |
| `maxDurability` | float > 0 | Conditional | Required when `hasDurability` is `true`. Starting and maximum durability value. |
| `tags` | string[] | No | Arbitrary classification labels. Useful for filtering in UI or scripting. |

---

## Durability

When `hasDurability` is `true`:

- New stacks are initialized at `maxDurability`.
- The `CurrentDurability` field appears on every slot in the inventory snapshot returned by `GetInventory`.
- Durability is reduced by calling a damage application service on the server (or directly through your game's server-side logic).
- Stacks with different durability values **cannot be merged** — they are treated as distinct items.
- `WithDurability(newValue)` produces a new immutable stack with the updated value.

### Example — durability in inventory snapshot

```json
{
  "index": 0,
  "itemDefinitionId": "sword",
  "quantity": 1,
  "currentDurability": 67.5
}
```

---

## Weight

`weight` defines the per-unit weight of the item. If a container has a `WeightCapacity`, the server enforces that the total weight of all stacks does not exceed it.

- `WeightCapacity = null` means unlimited (the default for all containers created via `CreateInventory`).
- Container `CurrentWeight` is returned in every inventory snapshot.
- Weight checks happen at `AddStack` / `PreviewAdd` time. The preview failure reason `WeightLimitExceeded` is returned when an add would push the total over the limit.

---

## Tags

Tags are free-form strings. The server does not filter or interpret them — they are stored on the definition and surfaced in snapshots for client-side use.

Common patterns:
- `"weapon"`, `"armor"`, `"consumable"` — item category
- `"quest"` — flag for quest items that shouldn't be dropped
- `"craftable"` — items that can appear as recipe outputs

---

## Example Files

### `data/items/resources.json`

```json
[
  { "id": "wood",  "displayName": "Wood",  "maxStackSize": 50, "weight": 1.0, "tags": ["resource"] },
  { "id": "stone", "displayName": "Stone", "maxStackSize": 30, "weight": 2.0, "tags": ["resource"] },
  { "id": "fiber", "displayName": "Fiber", "maxStackSize": 50, "weight": 0.5, "tags": ["resource"] }
]
```

### `data/items/weapons.json`

```json
[
  {
    "id": "sword",
    "displayName": "Iron Sword",
    "maxStackSize": 1,
    "weight": 3.0,
    "hasDurability": true,
    "maxDurability": 100,
    "tags": ["weapon", "melee", "craftable"]
  },
  {
    "id": "bow",
    "displayName": "Short Bow",
    "maxStackSize": 1,
    "weight": 1.5,
    "hasDurability": true,
    "maxDurability": 80,
    "tags": ["weapon", "ranged", "craftable"]
  }
]
```
