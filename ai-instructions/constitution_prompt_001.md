# eShop Codebase Constitution

## Runtime & SDK

- Target **.NET 10** (`global.json` pins SDK `10.0.100` with `rollForward: latestFeature`)
- All projects use **Central Package Management** (`Directory.Packages.props`). Never add a `Version` attribute to a `<PackageReference>` — always declare versions centrally.
- `TreatWarningsAsErrors = true` globally. Every new warning must be resolved, not suppressed.
- Enable `ImplicitUsings` and collect project-wide `global using` statements in a `GlobalUsings.cs` file at the project root.
- Use `UseArtifactsOutput = true`; build outputs go to the `artifacts/` directory.

## Architecture

This is a **microservices** solution composed of the following bounded contexts:

| Service            | Responsibility                      |
| ------------------ | ----------------------------------- |
| `Catalog.API`      | Product catalog, AI semantic search |
| `Basket.API`       | Shopping cart (Redis + gRPC)        |
| `Ordering.API`     | Orders (DDD + CQRS)                 |
| `Identity.API`     | Auth server (Duende IdentityServer) |
| `WebApp`           | Blazor Server front-end             |
| `OrderProcessor`   | Background saga worker              |
| `PaymentProcessor` | Background payment handler          |
| `Webhooks.API`     | Webhook delivery                    |
| `eShop.AppHost`    | .NET Aspire orchestrator            |

## Orchestration (.NET Aspire)

- All infrastructure (Postgres, Redis, RabbitMQ) is declared in `eShop.AppHost/Program.cs` using `DistributedApplication.CreateBuilder`.
- Use `ContainerLifetime.Persistent` for stateful infrastructure resources.
- Wire service-to-service connections with `.WithReference()` and `.WaitFor()` where ordering matters.
- Expose external endpoints with `.WithExternalHttpEndpoints()` only where needed.
- Do not hard-code URLs; use Aspire service discovery (`https+http://service-name`).

## Service Defaults

Every microservice calls `builder.AddServiceDefaults()` (or `AddBasicServiceDefaults()` for gRPC-only services). This wires:

- **OpenTelemetry** — traces (ASP.NET Core, gRPC, HTTP), metrics (ASP.NET Core, HTTP, runtime), and logs.
- **Service Discovery** — `AddServiceDiscovery()` on `IHttpClientBuilder`.
- **Resilience** — `AddStandardResilienceHandler()` on all outgoing HTTP clients.
- **Health Checks** — `/health` (readiness) and `/alive` (liveness) endpoints, active only in Development.

Never configure OpenTelemetry, resilience, or service discovery per-project; always delegate to `eShop.ServiceDefaults`.

## API Design (Minimal APIs)

- Use **Minimal API** style: `RouteGroupBuilder` returned from a `Map*Api()` extension on `IEndpointRouteBuilder`.
- Apply **API versioning** (`Asp.Versioning`): attach `HasApiVersion(x, y)` to route groups. Report versions with `options.ReportApiVersions = true`.
- Return **strongly-typed results** using `TypedResults` and union return types: `Results<Ok<T>, NotFound, BadRequest<string>, ProblemHttpResult>`.
- Decorate every route with OpenAPI metadata: `.WithName()`, `.WithSummary()`, `.WithDescription()`, `.WithTags()`.
- Use `[AsParameters]` to inject grouped service dependencies (see `OrderServices`).
- Add `builder.Services.AddProblemDetails()` and `app.UseStatusCodePages()` in APIs that serve HTTP errors.
- Expose OpenAPI/Scalar UI with `builder.AddDefaultOpenApi(withApiVersioning)` and `app.UseDefaultOpenApi()`.
- Endpoint handler methods should be **`public static async`** so they are independently unit-testable without standing up the full pipeline.

## CQRS & MediatR

- Commands and queries live in `Application/Commands/` and `Application/Queries/` respectively.
- All commands implement `IRequest<T>` from MediatR.
- Commands must be **immutable**: all setters `private`, data set only through the constructor. Annotate with `[DataContract]`/`[DataMember]`.
- Wrap commands that require idempotency in `IdentifiedCommand<TCommand, TResult>` and handle via `IdentifiedCommandHandler<T>`.
- Register MediatR **pipeline behaviors** in this order: `LoggingBehavior` → `ValidatorBehavior` → `TransactionBehavior`.
- Use `FluentValidation` validators (`AbstractValidator<T>`) for all commands; register via `FluentValidation.DependencyInjectionExtensions`.
- Queries use **EF Core LINQ** (or Dapper for complex projections) directly — they do not go through the domain layer.

## Domain-Driven Design

- The `Ordering.Domain` project is the **pure domain layer** — no infrastructure dependencies.
- Every aggregate implements `IAggregateRoot` (marker) and extends `Entity` (from `SeedWork`).
- Value Objects extend `ValueObject` and implement `GetEqualityComponents()`.
- Expose collections as `IReadOnlyCollection<T>` backed by `private readonly List<T>` — never expose mutable collections.
- All aggregate mutation methods must be on the **aggregate root** to enforce invariants.
- Raise **domain events** (`INotification`) inside aggregate methods using `AddDomainEvent()`. Dispatch them inside `SaveEntitiesAsync()` via `IMediator.DispatchDomainEventsAsync()` before committing.
- Repository interfaces go in the domain layer and are constrained to `IRepository<T> where T : IAggregateRoot`.

## Persistence (EF Core + PostgreSQL)

- Use **Npgsql** with the `pgvector` extension for AI embedding columns.
- Configure the schema per service: `modelBuilder.HasDefaultSchema("ordering")`.
- Place each entity's EF configuration in a dedicated `IEntityTypeConfiguration<T>` class under `Infrastructure/EntityConfigurations/`.
- Use **HiLo sequences** (`UseHiLo`) for integer primary keys.
- Store `enum` values as strings: `.HasConversion<string>().HasMaxLength(30)`.
- Ignore `DomainEvents` from EF mapping: `configuration.Ignore(b => b.DomainEvents)`.
- Persist Value Objects as **owned entity types** (`.OwnsOne()`).
- Add migrations with: `dotnet ef migrations add --startup-project <API> --context <Context> <Name>`
- Register automatic migration execution via `builder.Services.AddMigration<TContext, TSeed>()`.

## Integration Events (Outbox Pattern)

- Cross-service events extend `IntegrationEvent` and live in `Application/IntegrationEvents/Events/`.
- Publish events **after** the local transaction commits via `IOrderingIntegrationEventService.PublishEventsThroughEventBusAsync(transactionId)`.
- Subscribe to events with `.AddSubscription<TEvent, THandler>()` on the `IEventBusBuilder` returned by `AddRabbitMqEventBus("eventbus")`.
- Use `ConfigureJsonOptions` with `JsonSerializerContext` for AOT-compatible event serialization.

## Authentication & Authorization

- APIs secure endpoints with JWT Bearer: configure via the `Identity` section in `appsettings.json`:
  ```json
  { "Identity": { "Url": "https://identity-api", "Audience": "orders" } }
  ```
