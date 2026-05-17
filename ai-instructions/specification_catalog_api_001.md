Based on my exploration of the Catalog.API project and the eShop constitution, here's a comprehensive prompt that could generate a similar microservice:

---

## Prompt: Generate a Catalog-like Microservice for eShop

Create a new REST microservice called `[ServiceName].API` that follows the eShop architecture and conforms to the eShop Developer Constitution. The service should be a data-driven CRUD API with the following characteristics:

### 1. Project Structure & Configuration

**Create the following folder structure:**

```
src/[ServiceName].API/
├── Apis/
├── Extensions/
├── Infrastructure/
│   ├── EntityConfigurations/
│   ├── Exceptions/
│   └── Migrations/
├── IntegrationEvents/
│   ├── EventHandling/
│   └── Events/
├── Model/
├── Services/
├── Setup/
├── Pics/ (if serving static files)
├── Properties/
├── Program.cs
├── Program.Testing.cs
├── GlobalUsings.cs
├── [ServiceName]Options.cs
├── appsettings.json
└── [ServiceName].API.csproj
```

**Project file (`[ServiceName].API.csproj`) must include:**

- `<TargetFramework>net10.0</TargetFramework>`
- `<Nullable>enable</Nullable>`
- Package references:
  - `Asp.Versioning.Http`
  - `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL`
  - `Microsoft.EntityFrameworkCore.Tools` (for migrations)
  - `Microsoft.Extensions.ApiDescription.Server`
- Project references:
  - `eShop.ServiceDefaults`
  - `EventBusRabbitMQ`
  - `IntegrationEventLogEF`
- Link shared files from `Shared/` folder:
  - `ActivityExtensions.cs`
  - `MigrateDbContextExtensions.cs`
- `<InternalsVisibleTo Include="[ServiceName].FunctionalTests" />`

### 2. Program.cs Bootstrapping

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.AddApplicationServices();
builder.Services.AddProblemDetails();

var withApiVersioning = builder.Services.AddApiVersioning(options =>
{
    options.ReportApiVersions = true;
});

builder.AddDefaultOpenApi(withApiVersioning);

var app = builder.Build();

app.MapDefaultEndpoints();
app.UseStatusCodePages();
app.Map[ServiceName]Api();
app.UseDefaultOpenApi();
app.Run();
```

### 3. Extensions/Extensions.cs - Service Configuration

Create `AddApplicationServices()` extension that:

- Registers the DbContext using `builder.AddNpgsqlDbContext<[ServiceName]Context>("dbname")`
- Adds migrations using `builder.Services.AddMigration<[ServiceName]Context, [ServiceName]ContextSeed>()`
- Registers `IntegrationEventLogService<[ServiceName]Context>` as `IIntegrationEventLogService`
- Registers `I[ServiceName]IntegrationEventService` as transient
- Adds RabbitMQ event bus with `builder.AddRabbitMqEventBus("eventbus")` and subscriptions
- Binds configuration options using `builder.Services.AddOptions<[ServiceName]Options>().BindConfiguration(...)`
- Registers any domain-specific services (e.g., AI services, specialized services)

### 4. Domain Model (Model/)

Create entity classes with:

- Primary entity class with required properties
- Navigation properties to related entities (brands, types, categories, etc.)
- Business logic methods (e.g., `RemoveStock`, `AddStock`)
- Domain-specific exceptions deriving from `[ServiceName]DomainException`
- Supporting entities (e.g., Brand, Type, Category)
- DTOs for pagination: `PaginatedItems<T>`, `PaginationRequest`
- Services class: `[ServiceName]Services` record with constructor injection

**Example entity pattern:**

```csharp
public class [EntityName]
{
    public int Id { get; set; }

    [Required]
    public string Name { get; set; }

    public string? Description { get; set; }

    // Business logic methods
    public void DomainOperation()
    {
        if (/* invariant violated */)
        {
            throw new [ServiceName]DomainException("Error message");
        }
        // Update state
    }

    public [EntityName](string name) { Name = name; }
}
```

### 5. Infrastructure Layer

**DbContext (`Infrastructure/[ServiceName]Context.cs`):**

```csharp
public class [ServiceName]Context : DbContext
{
    public [ServiceName]Context(DbContextOptions<[ServiceName]Context> options, IConfiguration configuration)
        : base(options)
    {
    }

    public required DbSet<[Entity]> [Entities] { get; set; }
    // Additional DbSets...

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfiguration(new [Entity]EntityTypeConfiguration());
        // Apply other configurations...

        // Add the outbox table
        builder.UseIntegrationEventLogs();
    }
}
```

**Entity configurations (`Infrastructure/EntityConfigurations/`):**

- One configuration class per entity implementing `IEntityTypeConfiguration<T>`
- Configure table names, column constraints, indexes
- Configure relationships

**Seed data (`Infrastructure/[ServiceName]ContextSeed.cs`):**

- Implement `IDbSeeder<[ServiceName]Context>`
- Load seed data from JSON file in `Setup/` folder
- Populate lookup tables and main entities

### 6. Minimal API Endpoints (Apis/[ServiceName]Api.cs)

```csharp
public static class [ServiceName]Api
{
    public static IEndpointRouteBuilder Map[ServiceName]Api(this IEndpointRouteBuilder app)
    {
        var vApi = app.NewVersionedApi("[ServiceName]");
        var api = vApi.MapGroup("api/[servicename]").HasApiVersion(1, 0);
        var v1 = vApi.MapGroup("api/[servicename]").HasApiVersion(1, 0);

        // Define routes
        v1.MapGet("/items", GetAllItems)
            .WithName("ListItems")
            .WithSummary("List items")
            .WithDescription("Get a paginated list of items")
            .WithTags("Items");

        api.MapGet("/items/{id:int}", GetItemById)
            .WithName("GetItem")
            .WithSummary("Get item")
            .WithDescription("Get an item by ID")
            .WithTags("Items");

        api.MapPost("/items", CreateItem)
            .WithName("CreateItem")
            .WithSummary("Create item")
            .WithTags("Items");

        api.MapPut("/items/{id:int}", UpdateItem)
            .WithName("UpdateItem")
            .WithSummary("Update item")
            .WithTags("Items");

        api.MapDelete("/items/{id:int}", DeleteItem)
            .WithName("DeleteItem")
            .WithSummary("Delete item")
            .WithTags("Items");

        return app;
    }

    // Handler methods with proper response types
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status400BadRequest, "application/problem+json")]
    public static async Task<Ok<PaginatedItems<[Entity]>>> GetAllItems(
        [AsParameters] PaginationRequest paginationRequest,
        [AsParameters] [ServiceName]Services services)
    {
        // Implementation
    }

    // Additional handlers...
}
```

### 7. Integration Events

**Events (`IntegrationEvents/Events/`):**

- Define integration event records inheriting from `IntegrationEvent`
- Use past tense naming (e.g., `ProductPriceChangedIntegrationEvent`)
- Events represent "something that has happened"

**Event Handlers (`IntegrationEvents/EventHandling/`):**

- Implement `IIntegrationEventHandler<TEvent>` for each subscribed event
- Inject `[ServiceName]Context`, `I[ServiceName]IntegrationEventService`, and logger
- Process event, update state, publish response events

**Integration Event Service:**

```csharp
public sealed class [ServiceName]IntegrationEventService : I[ServiceName]IntegrationEventService, IDisposable
{
    public async Task SaveEventAndContextChangesAsync(IntegrationEvent evt)
    {
        await ResilientTransaction.New([serviceName]Context).ExecuteAsync(async () =>
        {
            await [serviceName]Context.SaveChangesAsync();
            await integrationEventLogService.SaveEventAsync(evt, [serviceName]Context.Database.CurrentTransaction);
        });
    }

    public async Task PublishThroughEventBusAsync(IntegrationEvent evt)
    {
        await integrationEventLogService.MarkEventAsInProgressAsync(evt.Id);
        await eventBus.PublishAsync(evt);
        await integrationEventLogService.MarkEventAsPublishedAsync(evt.Id);
    }
}
```

### 8. Global Usings (GlobalUsings.cs)

Include all commonly used namespaces:

```csharp
global using Asp.Versioning;
global using eShop.[ServiceName].API;
global using eShop.[ServiceName].API.Infrastructure;
global using eShop.[ServiceName].API.Model;
global using eShop.EventBus.Abstractions;
global using eShop.EventBus.Events;
global using eShop.IntegrationEventLogEF;
global using eShop.IntegrationEventLogEF.Services;
global using eShop.ServiceDefaults;
global using Microsoft.EntityFrameworkCore;
global using Microsoft.Extensions.Options;
// Add other commonly used namespaces
```

### 9. Configuration (appsettings.json)

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "OpenApi": {
    "Endpoint": {
      "Name": "[ServiceName].API V1"
    },
    "Document": {
      "Description": "The [ServiceName] Microservice HTTP API",
      "Title": "eShop - [ServiceName] HTTP API",
      "Version": "v1"
    }
  },
  "ConnectionStrings": {
    "EventBus": "amqp://localhost"
  },
  "EventBus": {
    "SubscriptionClientName": "[ServiceName]"
  },
  "[ServiceName]Options": {
    // Service-specific options
  }
}
```

### 10. Testing Support (Program.Testing.cs)

```csharp
public partial class Program { }
```

### 11. AppHost Registration

In `eShop.AppHost/Program.cs`, add:

```csharp
var [serviceName]Api = builder.AddProject<Projects.[ServiceName]_API>("[servicename]-api")
    .WithReference(postgres)
    .WaitFor(postgres)
    .WithReference(rabbitMq)
    .WaitFor(rabbitMq)
    .WithHttpHealthCheck("/health");
```

### 12. Mandatory Companion Projects

Create:

- `tests/[ServiceName].UnitTests/` - Unit tests project
- `tests/[ServiceName].FunctionalTests/` - Functional tests using `WebApplicationFactory<Program>`

### Key Architectural Rules

1. **Always** use Minimal API - never MVC controllers
2. **Always** use API versioning with route groups
3. **Always** use PostgreSQL with EF Core via Aspire
4. **Always** implement the Outbox pattern for integration events
5. **Always** subscribe to events via RabbitMQ event bus
6. **Always** use `[AsParameters]` for service injection in handlers
7. **Always** return typed results (`Ok<T>`, `NotFound`, `BadRequest<ProblemDetails>`)
8. **Always** add `[ProducesResponseType]` attributes for error responses
9. **Always** use proper HTTP status codes and problem details
10. **Always** follow the constitution rules for package versions, SDK settings, and patterns

Would you like me to generate any specific part of this structure in more detail, or would you like to see a concrete example for a specific domain (e.g., Inventory.API, Reviews.API)?
