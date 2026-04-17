---
title: Testing Guide
parent: Development
nav_order: 1
---

# Testing Guide

This document explains how to run the test suite, what each test project covers, and how to add new tests.

---

## Running the tests

From the solution root:

```bash
dotnet test
```

That's it. All test projects run in one go. You should see output like:

```
Passed!  - Failed: 0, Passed: 1150, Skipped: 0
```

To run a single project:

```bash
dotnet test InventoryFramework.Domain.Tests
```

To filter by test name:

```bash
dotnet test --filter "FullyQualifiedName~ItemStack"
```

---

## Test projects at a glance

| Project | What it covers |
|---|---|
| `Domain.Tests` | Business rules — item stacks, containers, aggregates, affixes |
| `Application.Tests` | Use-case services — grant, transfer, craft, lock slot, split stack, drop items, sort, progression |
| `Infrastructure.Tests` | JSON loaders, recipe registry, station/visibility policies |
| `Persistence.Tests` | Snapshot mapper, file repositories, SQL repositories |
| `Server.Tests` | gRPC mapping, auth interceptor, license validation, SignalR |
| `SDK.Tests` | Client options, gRPC mapper, result types |
| `GodotAdapter.Tests` | Godot facade and session controller |
| `UnityAdapter.Tests` | Unity facade and session controller |
| `UnrealAdapter.Tests` | Unreal facade and session controller |
| `IntegrationTests` | Full server stack via in-process WebApplicationFactory |

---

## Domain tests

These test the core business logic with no external dependencies at all — just pure C#.

Location: `InventoryFramework.Domain.Tests/`

A typical domain test:

```csharp
[Fact]
public void AddStack_WhenSlotIsEmpty_PlacesItemInSlot()
{
    var container = ContainerBuilder.WithCapacity(10).Build();
    var item = ItemStack.Create("wood", 5);

    var result = container.AddStack(item);

    Assert.True(result.Succeeded);
    Assert.Equal(1, container.OccupiedSlotCount);
}
```

No mocks, no setup. If a domain test needs a dependency, the design is probably off.

---

## Application tests

These test the use-case services (`GrantItemsApplicationService`, `CraftItemsApplicationService`, etc.). Instead of hitting a real database they use the in-memory repository from `InventoryFramework.Persistence`.

Location: `InventoryFramework.Application.Tests/`

Helper classes in `TestData/` build ready-to-use aggregates:

```csharp
var aggregate = TestInventoryFactory.CreateWithItems("wood", 10);
```

---

## Integration tests

These spin up the full server in-process and send real gRPC requests through it.

Location: `InventoryFramework.IntegrationTests/`

```csharp
public class GrpcInventoryClientIntegrationTests : IClassFixture<InventoryWebApplicationFactory>
{
    private readonly IInventoryClient _client;

    public GrpcInventoryClientIntegrationTests(InventoryWebApplicationFactory factory)
    {
        _client = factory.CreateGrpcClient();
    }

    [Fact]
    public async Task GrantItems_ReturnsSuccess()
    {
        var created = await _client.CreateInventoryAsync(...);
        var granted = await _client.GrantItemsAsync(created.InventoryId, ...);

        Assert.True(granted.Succeeded);
    }
}
```

The server starts fresh for each test class (`IClassFixture`). Tests within the same class share one server instance.

---

## Naming convention

```
MethodName_WhenCondition_ExpectedOutcome

// Examples:
AddStack_WhenContainerIsFull_ReturnsFailed
CraftItems_WhenIngredientsAreMissing_ReturnsInsufficientIngredients
GetInventory_WhenAggregateDoesNotExist_ReturnsNotFound
```

---

## Code coverage

```bash
dotnet test --collect:"XPlat Code Coverage" --results-directory ./TestResults
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"TestResults/**/coverage.cobertura.xml" -targetdir:"TestResults/html"
```

Then open `TestResults/html/index.html`.

---

## Notes

- **Never share state between tests.** Each test must set up its own data.
- **In-memory SQLite resets between tests.** Each test instance gets a clean database.
- **Integration tests are slower** (~5 seconds total). Keep heavy setup in `IClassFixture`.
- **Don't mock the domain.** The domain has no external dependencies — test it directly.
