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

A typical domain test looks like this:

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
// From TestData/TestInventoryFactory.cs
var aggregate = TestInventoryFactory.CreateWithItems("wood", 10);
```

A typical application test:

```csharp
[Fact]
public async Task GrantItems_WhenContainerHasSpace_AddsItemToSlot()
{
    var repo = new InMemoryInventoryAggregateRepository();
    var aggregate = TestInventoryFactory.CreateEmpty();
    await repo.SaveAsync(aggregate, ...);

    var service = new GrantItemsApplicationService(repo, itemRegistry, ...);
    var result = await service.ExecuteAsync(new GrantItemsCommand
    {
        AggregateId     = aggregate.Id.Value,
        ContainerId     = aggregate.Containers[0].Id.Value,
        ItemDefinitionId = "wood",
        Quantity        = 5
    });

    Assert.True(result.Succeeded);
}
```

---

## Infrastructure tests

These test JSON file loading and policy evaluation.

Location: `InventoryFramework.Infrastructure.Tests/`

JSON source tests write temp files to disk and clean them up in `Dispose`:

```csharp
public class JsonItemDefinitionSourceTests : IDisposable
{
    private readonly string _tempDir = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());

    [Fact]
    public async Task LoadAsync_ValidFile_ReturnsDefinitions() { ... }

    public void Dispose() => Directory.Delete(_tempDir, recursive: true);
}
```

---

## Persistence tests

These cover the snapshot mapper and the file-based repositories.

Location: `InventoryFramework.Persistence.Tests/`

The SQL repository tests (`Persistence.Tests/Sql/`) use an in-memory SQLite database — no database server needed:

```csharp
var options = new DbContextOptionsBuilder<InventoryDbContext>()
    .UseSqlite("Data Source=:memory:")
    .Options;
```

Each test gets a fresh database. Tests are independent and can run in any order.

---

## Server tests

These test the gRPC layer in isolation — mapping, auth, licensing — without starting a real HTTP server.

Location: `InventoryFramework.Server.Tests/`

The auth interceptor tests build a minimal gRPC `ServerCallContext` by hand:

```csharp
var context = TestServerCallContext.Create(method: "GrantItems",
    requestHeaders: new Metadata { { "x-api-key", "valid-key" } });
```

---

## Integration tests

These spin up the full server in-process and send real gRPC requests through it.

Location: `InventoryFramework.IntegrationTests/`

The key class is `InventoryWebApplicationFactory`:

```csharp
public class InventoryWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Swap the real repository for an in-memory one
            services.AddSingleton<IInventoryAggregateRepository,
                InMemoryInventoryAggregateRepository>();
        });
    }
}
```

A full integration test:

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

## Writing a new test

**Domain or Application test** — just add an `[Fact]` or `[Theory]` to the relevant test project. No configuration needed.

**Integration test** — inject `InventoryWebApplicationFactory` via `IClassFixture`. Use `factory.CreateGrpcClient()` to get a connected client.

**Naming convention:**

```
MethodName_WhenCondition_ExpectedOutcome

// Examples:
AddStack_WhenContainerIsFull_ReturnsFailed
CraftItems_WhenIngredientsAreMissing_ReturnsInsufficientIngredients
GetInventory_WhenAggregateDoesNotExist_ReturnsNotFound
```

This makes failing tests immediately readable in the output — you know what broke and under what condition without opening the file.

---

## Code coverage

To generate a coverage report locally:

```bash
dotnet test --collect:"XPlat Code Coverage" --results-directory ./TestResults
```

This creates `coverage.cobertura.xml` files under `TestResults/`. To view them as HTML, install the report generator tool:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"TestResults/**/coverage.cobertura.xml" -targetdir:"TestResults/html"
```

Then open `TestResults/html/index.html` in a browser.

Coverage is also uploaded automatically on every CI run (see `.github/workflows/ci.yml`).

---

## A few notes

- **Never share state between tests.** Each test must set up its own data. If a test depends on the order of execution, it will break in CI.
- **In-memory SQLite resets between tests.** Each `SqlInventoryAggregateRepositoryTests` instance gets a clean database.
- **Integration tests are slower** (~5 seconds total) because they start an HTTP server. Keep the heavy setup in `IClassFixture` shared across tests in the same class.
- **Don't mock the domain.** The domain has no external dependencies — test it directly. Mocking `InventoryAggregate` will mask bugs that real usage would catch.
