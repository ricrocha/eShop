<!--
SYNC IMPACT REPORT
Version change: (template) → 1.0.0
Added sections: All 19 sections (initial population from full codebase analysis)
Removed sections: Template placeholders replaced with concrete rules
Modified principles: N/A (initial creation)
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ no constitution-specific references found
  - .specify/templates/spec-template.md ✅ no constitution-specific references found
  - .specify/templates/tasks-template.md ✅ no constitution-specific references found
Follow-up TODOs: None — all sections fully populated from codebase evidence.
-->

# eShop Developer Constitution

This document is the **single source of truth** for how all code in the eShop repository MUST
be written — by humans and AI assistants alike. Every rule is derived from evidence in the
actual codebase and its configuration files. Rules are stated as unambiguous prescriptions.

---

## 1. Runtime & SDK

- Always target `net10.0`; never set a `TargetFramework` to an earlier version.
- Always pin the SDK in `global.json` to `10.0.100` with `rollForward: latestFeature` and
  `allowPrerelease: true`; never remove or downgrade this pin.
- Always declare `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` via `Directory.Build.props`;
  never suppress it at the project level.
- Always enable `<ImplicitUsings>enable</ImplicitUsings>` and `<UseArtifactsOutput>true</UseArtifactsOutput>`
  globally in `Directory.Build.props`; never override these per-project.
- Always embed debug symbols (`<DebugType>embedded</DebugType>`, `<DebugSymbols>true</DebugSymbols>`);
  never use `pdb` or `portable` as a project-level override.
- Never suppress `TreatWarningsAsErrors` beyond the `NU1901`–`NU1904` transitive-package security
  warning codes already excluded globally.
- Always set `<SuppressNETCoreSdkPreviewMessage>true</SuppressNETCoreSdkPreviewMessage>` globally.
- Always use `MSTest.Sdk` (version `4.0.2`, declared in `global.json` `msbuild-sdks`) as the SDK
  for all test projects; never use `Microsoft.NET.Sdk` for test projects.
- Always configure `"test": { "runner": "Microsoft.Testing.Platform" }` in `global.json`.
- Always enable `<EventSourceSupport>true</EventSourceSupport>` and `<TrimmerSingleWarn>false</TrimmerSingleWarn>`
  in `Directory.Build.targets` for AOT-published projects.
- Build artifacts go to `artifacts/bin/<ProjectName>/` and `artifacts/obj/<ProjectName>/` via
  `UseArtifactsOutput`; never manually set `OutputPath`.

---

## 2. Package Governance

- Always use Central Package Management: `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>`
  and `<CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>` in
  `Directory.Packages.props`.
- Always declare `<PackageVersion>` entries in `Directory.Packages.props`; never specify a
  `Version=` attribute on a `<PackageReference>` in a project file.
- Always group packages by logical owner (Aspire, ASP.NET, EF, gRPC, OTel, IdentityServer, AI,
  Testing, Miscellaneous) with the version property variables already in `Directory.Packages.props`.
- The introduction of **any new NuGet package** requires committee approval before it is added to
  `Directory.Packages.props`. New packages MUST be justified against packages already in the solution.
- MediatR MUST remain at version `13.0.0`; it was frozen before the license change and MUST NOT
  be upgraded without explicit committee decision.
- Always use `NSubstitute 5.3.0` for mocking in tests; never add Moq or any other mocking library.
- Always use `FluentValidation 12.0.0`; always pair it with
  `FluentValidation.DependencyInjectionExtensions` at the same version.
- Always use `Scalar.AspNetCore` for the OpenAPI UI; never reintroduce Swagger UI.
- Use version property variables (e.g., `$(AspireVersion)`, `$(DotnetPackagesVersion)`) for
  packages that must be kept in sync; never hard-code versions for these families.
- Never add `xunit` to a non-functional-test project; never add `MSTest` to a functional test
  project that already uses `xunit`.

---

## 3. Architecture

### Services and Bounded Contexts

| Service            | Bounded Context    | Responsibility                                                          |
| ------------------ | ------------------ | ----------------------------------------------------------------------- |
| `Identity.API`     | Identity           | OIDC provider (Duende IdentityServer + ASP.NET Identity + PostgreSQL)   |
| `Catalog.API`      | Product Catalog    | Product browsing, search (with AI embeddings + pgvector), image serving |
| `Basket.API`       | Shopping Basket    | gRPC basket CRUD, Redis persistence                                     |
| `Ordering.API`     | Ordering           | Order lifecycle; hosts DDD domain model, CQRS, outbox                   |
| `OrderProcessor`   | Ordering Worker    | Processes grace-period and stock-validation events via RabbitMQ         |
| `PaymentProcessor` | Payment Worker     | Processes payment events via RabbitMQ                                   |
| `Webhooks.API`     | Webhooks           | Sends webhook notifications for order status changes                    |
| `WebApp`           | Online Store (BFF) | Blazor Server UI; consumes Catalog, Ordering, Basket via HTTP/gRPC      |
| `WebhookClient`    | Webhook Demo       | Test/demo webhook receiver                                              |
| `mobile-bff`       | Mobile BFF         | YARP reverse proxy for mobile clients                                   |
| `HybridApp`        | Hybrid UI          | .NET MAUI hybrid app                                                    |
| `ClientApp`        | MAUI Client        | .NET MAUI client application                                            |

### Architectural Patterns in Use

- **Microservices**: each service owns its own persistence, schema, and event contracts.
- **DDD**: Ordering uses aggregates, entities, value objects, domain events, and repositories.
- **CQRS with MediatR**: commands go through MediatR pipeline behaviors; queries bypass
  MediatR and use EF Core / Dapper directly.
- **Event-driven architecture**: all cross-service communication uses integration events over
  RabbitMQ via `IEventBus`; direct service-to-service HTTP calls are **forbidden** outside of
  Aspire service discovery.
- **Outbox pattern**: `IntegrationEventLogEF` stores events in the same DB transaction before
  publishing; `TransactionBehavior` dispatches them after commit.
- **Idempotency**: `IdentifiedCommand<T,R>` + `RequestManager` (backed by `ClientRequest` table)
  prevent duplicate command processing.
- **Saga (choreography)**: state changes propagate via integration events (e.g.,
  `GracePeriodConfirmedIntegrationEvent` → OrderProcessor → `OrderStockConfirmedIntegrationEvent`).
- **BFF**: `mobile-bff` (YARP) aggregates Catalog, Ordering, and Identity endpoints for mobile.
- **gRPC**: Basket.API exposes all basket operations exclusively via gRPC; never add REST routes
  to Basket.API.
- **Minimal API**: all REST services (Catalog, Ordering, Webhooks) use Minimal API exclusively;
  never add MVC controllers to these services.

### New Service Requirements

- Any new service MUST conform to all patterns listed above.
- No new service may introduce direct service-to-service HTTP calls outside Aspire service discovery.
- No new service may bypass the event bus for cross-service communication.

---

## 4. Extensibility Contract

When adding a new API service or third-party integration, the following checklist is **mandatory**:

1. **Register in `eShop.AppHost`**: call `builder.AddProject<Projects.NewService>(...)` with
   `WithReference(...)` for every infrastructure dependency, `WaitFor(...)` for ordering, and
   `WithHttpHealthCheck("/health")`.
2. **Add service defaults**: call `builder.AddServiceDefaults()` (or `builder.AddBasicServiceDefaults()`
   for services that make no outbound HTTP calls) as the first line in `Program.cs`.
3. **Expose via Minimal API**: define route groups in a static extension method
   `Map<ServiceName>Api(this IEndpointRouteBuilder app)` in `Apis/<ServiceName>Api.cs`; never use
   MVC controllers.
4. **Publish/subscribe through the event bus**: call `builder.AddRabbitMqEventBus("eventbus")`
   and register subscriptions via `.AddSubscription<TEvent, THandler>()`.
5. **Register AOT-safe JSON context**: for each integration event consumed, add a
   `[JsonSerializable(typeof(TEvent))]` `JsonSerializerContext` and chain it via
   `.ConfigureJsonOptions(options => options.TypeInfoResolverChain.Add(MyContext.Default))`.
6. **Reflect UI changes**: if the new service has user-visible functionality, update `WebApp` and/or
   `WebAppComponents`; never let a new backend service go live without corresponding UI.
7. **Unit test project**: every new service MUST ship with a `<ServiceName>.UnitTests` project
   before merge.
8. **Functional test project**: every new service MUST be covered by at least one functional test
   (`<ServiceName>.FunctionalTests`) before merge.
9. **`Program.Testing.cs`**: add `public partial class Program { }` so that
   `WebApplicationFactory<Program>` works in functional tests.
10. **`InternalsVisibleTo`**: add `<InternalsVisibleTo Include="<ServiceName>.FunctionalTests" />`
    to the service `.csproj`.

---

## 5. Orchestration

- Always declare infrastructure (Redis, RabbitMQ, PostgreSQL) in `eShop.AppHost/Program.cs` and
  pass references to services; never hard-code connection strings in service code.
- Always use `ContainerLifetime.Persistent` for RabbitMQ and PostgreSQL containers so they survive
  restarts during development.
- Always use the `ankane/pgvector:latest` image for PostgreSQL to support the `vector` extension.
- Always call `.WaitFor(rabbitMq)` on every service that subscribes to or publishes events; never
  allow a service to start before its infrastructure is ready.
- Always call `.WaitFor(orderingApi)` on `OrderProcessor` because `Ordering.API` owns the EF
  migrations for the shared schema.
- Always call `.WithExternalHttpEndpoints()` for services that must be reachable from outside the
  Aspire network (Identity, mobile-bff, WebApp).
- Always call `.WithHttpHealthCheck("/health")` for services that expose health endpoints.
- Use the `launchProfileName` pattern (`ShouldUseHttpForEndpoints()`) to support CI/CD over plain
  HTTP while keeping HTTPS as the default for local development.
- Never declare infrastructure resources in individual service projects; all resources are
  declared once in `AppHost` and shared via Aspire references.
- Never configure service discovery URIs manually; always use Aspire's
  `https+http://<service-name>` scheme so the runtime resolves addresses.

---

## 6. Service Defaults

Every service MUST wire up via `eShop.ServiceDefaults`:

| Concern                        | Method                                                                     |
| ------------------------------ | -------------------------------------------------------------------------- |
| Health checks + OpenTelemetry  | `builder.AddServiceDefaults()`                                             |
| Service discovery + resilience | `builder.AddServiceDefaults()`                                             |
| Health endpoints               | `app.MapDefaultEndpoints()`                                                |
| OpenAPI (API services only)    | `builder.AddDefaultOpenApi(withApiVersioning)` + `app.UseDefaultOpenApi()` |
| Authentication (when needed)   | `builder.AddDefaultAuthentication()`                                       |

Services that make **no outbound HTTP calls** (e.g., Basket.API with gRPC only) MUST call
`builder.AddBasicServiceDefaults()` instead of `AddServiceDefaults()` to avoid loading
Polly resilience pipelines.

**Never** configure per-project:

- OpenTelemetry exporters or instrumentation (owned entirely by `ServiceDefaults`).
- Service discovery middleware (owned entirely by `ServiceDefaults`).
- Standard resilience handlers (owned entirely by `ServiceDefaults`).
- Health check infrastructure beyond the self liveness check (owned by `ServiceDefaults`).

The OTLP exporter is activated automatically when `OTEL_EXPORTER_OTLP_ENDPOINT` is set in the
environment; never add conditional OTLP configuration inside individual services.

---

## 7. API Design

- Always use Minimal API; never use MVC controllers in API services.
- Always define routes in a static class named `<ServiceName>Api` inside `Apis/<ServiceName>Api.cs`.
- Always expose route registration via a static extension method
  `Map<ServiceName>Api(this IEndpointRouteBuilder app)` or
  `Map<ServiceName>ApiV1(this RouteGroupBuilder api)`.
- Always use API versioning (`Asp.Versioning.Http`): call `NewVersionedApi("<Tag>")` and
  `.HasApiVersion(major, minor)` on every route group; always set `options.ReportApiVersions = true`.
- Always use `TypedResults` for return types: `Ok<T>`, `NotFound`, `BadRequest<string>`,
  `ProblemHttpResult`; never return `IActionResult` or `object`.
- Always declare handler methods as `public static async Task<Results<...>>` — testable without
  `WebApplicationFactory`.
- Always use `[AsParameters]` to group service dependencies into a service parameter record
  (e.g., `OrderServices`) rather than listing them individually in handler signatures.
- Always add OpenAPI metadata to every route: `.WithName(...)`, `.WithSummary(...)`,
  `.WithDescription(...)`, `.WithTags(...)`.
- Always call `.RequireAuthorization()` on route groups that require authentication; never rely
  on attribute-based authorization in Minimal API.
- Always call `builder.Services.AddProblemDetails()` in services that return `ProblemHttpResult`.
- Always activate OpenAPI only when an `OpenApi` configuration section exists
  (guard inside `AddDefaultOpenApi` and `UseDefaultOpenApi`).

---

## 8. CQRS & MediatR

### Folder Structure

```
Application/
  Behaviors/           # Pipeline behaviors (Logging, Validator, Transaction)
  Commands/            # XxxCommand.cs + XxxCommandHandler.cs
  DomainEventHandlers/ # Handlers for domain events
  IntegrationEvents/
    EventHandling/     # Integration event handlers
    Events/            # Integration event types
  Models/              # DTOs used by commands and queries
  Queries/             # IXxxQueries interface + XxxQueries implementation
  Validations/         # XxxCommandValidator.cs (one per command)
```

### Commands

- Always make commands immutable: all setters `private`, data set only through the constructor.
- Always annotate commands with `[DataContract]` and each property with `[DataMember]`.
- Always implement commands as `IRequest<TResult>`.
- Always implement command handlers as `IRequestHandler<TCommand, TResult>`.
- Always wrap mutation commands in `IdentifiedCommand<TCommand, TResult>` at the API handler
  level to guarantee idempotency; never call the inner command directly from API endpoints.
- Always implement a concrete `IdentifiedCommandHandler<T,R>` subclass that overrides
  `CreateResultForDuplicateRequest()`.

### Pipeline Behavior Registration Order (MUST NOT change)

1. `LoggingBehavior<,>` — logs command entry and exit.
2. `ValidatorBehavior<,>` — runs all `IValidator<T>` implementations; throws on failure.
3. `TransactionBehavior<,>` — wraps the handler in a DB transaction and publishes events
   after commit.

Register via:

```csharp
cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
cfg.AddOpenBehavior(typeof(ValidatorBehavior<,>));
cfg.AddOpenBehavior(typeof(TransactionBehavior<,>));
```

### Queries

- Always use `OrderQueries`-style classes (EF Core or Dapper) for queries; never route queries
  through MediatR.
- Always define a `IXxxQueries` interface in the `Queries/` folder; register as `Scoped`.

### FluentValidation

- Always derive validators from `AbstractValidator<TCommand>`.
- Always register via `services.AddValidatorsFromAssemblyContaining<AnyValidatorInProject>()`.
- Always guard `LogLevel.Trace` logging inside validators:
  `if (logger.IsEnabled(LogLevel.Trace)) { logger.LogTrace(...); }`.

---

## 9. Domain-Driven Design

- Always derive aggregate root classes from `Entity` (in `Ordering.Domain.Seedwork`) and implement
  `IAggregateRoot`; never create aggregates that do not carry both.
- Always derive value objects from `ValueObject` (in `Ordering.Domain.SeedWork`) and implement
  `GetEqualityComponents()`.
- Always define repository interfaces in the `AggregatesModel/<Aggregate>/` folder alongside the
  aggregate they serve (e.g., `IOrderRepository` in `OrderAggregate/`).
- Always constrain repositories to `IRepository<T> where T : IAggregateRoot`; never inject a
  DbContext directly from a command handler.
- Always use `IUnitOfWork` (not `DbContext` directly) to coordinate saves in command handlers;
  call `SaveEntitiesAsync()` which dispatches domain events before persisting.
- Always back internal collections with a `private readonly List<T>` field; always expose them as
  `public IReadOnlyCollection<T>` (via `.AsReadOnly()`); never expose `List<T>` publicly.
- Always raise domain events by calling `AddDomainEvent(new XxxDomainEvent(...))` inside
  aggregate methods; never dispatch events from outside the aggregate.
- Always enforce invariants inside aggregate methods; throw a `<ServiceName>DomainException` when
  an invariant is violated; never place invariant logic in application services or command handlers.
- Always place domain exceptions in `Ordering.Domain/Exceptions/`.
- Always declare domain event types in `Ordering.Domain/Events/`.

---

## 10. Persistence

- Always use `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` for EF Core + PostgreSQL;
  always call `builder.EnrichNpgsqlDbContext<TContext>()` after `AddDbContext` to activate
  Aspire telemetry and health checks.
- Always define a schema per service using `modelBuilder.HasDefaultSchema("<schema>")` in
  `OnModelCreating` (e.g., `"ordering"` for `OrderingContext`).
- Always use HiLo sequences for primary keys: `.UseHiLo("<sequenceName>")`.
- Always store enum properties as strings: `.HasConversion<string>().HasMaxLength(30)`.
- Always map value objects as owned entities: `.OwnsOne(o => o.Address)`.
- Always create one `IEntityTypeConfiguration<TEntity>` class per entity in
  `Infrastructure/EntityConfigurations/`; never configure entities inline in `OnModelCreating`.
- Always apply configurations via `modelBuilder.ApplyConfiguration(new XxxEntityTypeConfiguration())`.
- Always ignore domain event collections in EF configuration:
  `entityConfiguration.Ignore(b => b.DomainEvents)`.
- Always include the outbox table via `builder.UseIntegrationEventLogs()`.
- Always run migrations from the `Infrastructure` project, specifying the startup project:
  ```
  dotnet ef migrations add --startup-project ../Ordering.API --context OrderingContext <MigrationName>
  ```
- In development, always auto-apply migrations using the `AddMigration<TContext, TSeed>()`
  extension; never run `Database.EnsureCreated()`.
- Never use `DbContext` pooling for `OrderingContext` because it has multiple constructors.

---

## 11. Integration Events

- Always store integration events in the outbox (`IntegrationEventLogEF`) within the same
  database transaction as the domain change; never publish events without an associated
  transaction record.
- Always publish stored events after transaction commit via
  `_orderingIntegrationEventService.PublishEventsThroughEventBusAsync(transactionId)` inside
  `TransactionBehavior`.
- Always define integration event types as records or classes that inherit from `IntegrationEvent`.
- Always register event subscriptions in a dedicated `AddEventBusSubscriptions()` private method
  inside the service's `Extensions.cs`; never scatter subscription registrations across `Program.cs`.
- Always use AOT-safe JSON serialization: declare a `[JsonSerializable(typeof(TEvent))]`
  `partial class XxxContext : JsonSerializerContext` and chain it via
  `.ConfigureJsonOptions(options => options.TypeInfoResolverChain.Add(XxxContext.Default))`.
- Always use `IEventBus.PublishAsync(IntegrationEvent)` as the sole public API for publishing;
  never invoke RabbitMQ client APIs directly from application code.
- Always use the `"eshop_event_bus"` exchange name (declared in `RabbitMQEventBus`); never
  create additional exchanges.
- Always use `direct` exchange type; route by event type name.

---

## 12. Authentication & Authorization

### API Services (JWT Bearer)

- Always call `builder.AddDefaultAuthentication()` from `eShop.ServiceDefaults` in any service
  that requires JWT authentication.
- Always configure the `Identity` section in `appsettings.json`:
  ```json
  "Identity": {
    "Url": "http://identity-api",
    "Audience": "<service-audience>",
    "Scopes": { "<scope-name>": "<scope-description>" }
  }
  ```
- Always remove the `"sub"` claim type mapping:
  `JsonWebTokenHandler.DefaultInboundClaimTypeMap.Remove("sub")`.
- Always set `RequireHttpsMetadata = false` (Aspire handles TLS at the infrastructure level);
  never set it to `true` in the service itself.
- Always validate the audience implicitly via the `Audience` property; set
  `ValidateAudience = false` only when the provider does not embed the audience.
- Always call `.RequireAuthorization()` on route groups that need authentication; never leave
  protected endpoints without an explicit authorization requirement.

### WebApp (OIDC + Cookie)

- Always use `CookieAuthenticationDefaults.AuthenticationScheme` as `DefaultScheme` and
  `OpenIdConnectDefaults.AuthenticationScheme` as `DefaultChallengeScheme`.
- Always remove `"sub"` from `JsonWebTokenHandler.DefaultInboundClaimTypeMap` in WebApp as well.
- Always configure `ResponseType = "code"` (authorization code flow); never use implicit flow.
- Always save tokens: `options.SaveTokens = true`.

---

## 13. Caching & Storage

- Always use the key prefix pattern `/basket/<userId>` for Redis basket keys.
- Always declare the key prefix as a `private static RedisKey` using a UTF-8 byte literal:
  ```csharp
  private static RedisKey BasketKeyPrefix = "/basket/"u8.ToArray();
  ```
- Always serialize cache values to UTF-8 bytes using `JsonSerializer.SerializeToUtf8Bytes`
  with a `JsonSerializerContext`; never use `JsonSerializer.Serialize(string)` for Redis storage.
- Always deserialize from `Span<byte>` (`StringGetLeaseAsync`) for zero-copy reads.
- Always declare a dedicated `[JsonSerializable(typeof(T))]` `JsonSerializerContext` for each
  cached type; never use reflection-based JSON for cache read/write paths.
- Always set `[JsonSourceGenerationOptions(PropertyNameCaseInsensitive = true)]` on cache
  serialization contexts.
- Always inject `IConnectionMultiplexer` (via `builder.AddRedisClient("redis")`) and obtain
  `IDatabase` lazily in the repository constructor.

---

## 14. AI Integration

- Always use `Microsoft.Extensions.AI` abstractions (`IEmbeddingGenerator<string, Embedding<float>>`)
  never reference provider SDKs (OpenAI, Ollama) directly in application code.
- Always gate AI features with an `IsEnabled` property that returns `_embeddingGenerator is not null`;
  never throw when the generator is absent — degrade gracefully.
- Always resolve AI providers from configuration, not from compile-time switches; use
  `builder.AddOpenAIClientFromConfiguration("textEmbeddingModel")` or
  `builder.AddOllamaApiClient("embedding")`.
- Always register the feature in `AppHost` as an optional block guarded by a `bool useOpenAI`
  or `bool useOllama` flag; never make AI mandatory for the service to start.
- Always store embeddings as `Pgvector.Vector` in PostgreSQL; use the `ankane/pgvector` image.
- Always trim embeddings to the target dimension immediately after generation:
  `embedding[0..EmbeddingDimensions]` — the constant is local to the service.
- Always use `ValueTask` for AI operation return types on hot paths.
- Always guard `LogLevel.Trace` logging around embedding operations:
  `if (_logger.IsEnabled(LogLevel.Trace)) { _logger.LogTrace(...); }`.
- Always enable OpenAI OpenTelemetry via the runtime configuration option
  `OpenAI.Experimental.EnableOpenTelemetry = true` (declared globally in `Directory.Build.props`).

---

## 15. Observability

- Always use structured log message templates with named properties; never use string
  interpolation in log messages.
- Always use `@` destructuring for complex objects: `{@Command}`, `{@Command}`, `{@Response}`.
- Always guard `LogLevel.Trace` calls with `if (logger.IsEnabled(LogLevel.Trace))` before
  calling `logger.LogTrace(...)`.
- Always use `[LoggerMessage]` source-generated methods for high-frequency log events (see
  `OrderingApiTrace`); never call `logger.Log` directly for trace-level events in hot paths.
- Always open a logging scope with `TransactionContext` when beginning a database transaction:
  ```csharp
  using (_logger.BeginScope(new List<KeyValuePair<string, object>>
      { new("TransactionContext", transaction.TransactionId) }))
  ```
- Always configure OpenTelemetry in `eShop.ServiceDefaults.Extensions.ConfigureOpenTelemetry()`;
  never add OTel instrumentation per-service.
- Always add the `Experimental.Microsoft.Extensions.AI` meter and activity source in OTel
  configuration to capture AI calls.
- Always activate the OTLP exporter conditionally: check `OTEL_EXPORTER_OTLP_ENDPOINT`;
  never hard-code an OTLP endpoint.
- Always set `AlwaysOnSampler` only in development; rely on default sampling in production.

---

## 16. Testing

### Framework & Tooling

- **MSTest** with `Microsoft.Testing.Platform` is the MANDATORY framework for all **unit tests**.
  Use the `MSTest.Sdk` project SDK (declared in `global.json`).
- **xunit v3** (`xunit.v3.mtp-v2`) is the MANDATORY framework for all **functional tests** that
  use `IClassFixture<T>`, `[Theory]`, or `[InlineData]`.
- Always use **NSubstitute** for mocking; never use Moq or any other mocking library.
- Always use `NullLogger<T>.Instance` for logger dependencies in unit tests; never create real
  loggers or use `Substitute.For<ILogger<T>>()` unless the test asserts on log output.
- Every test project MUST contain a `GlobalUsings.cs` that imports all commonly used namespaces.
- Every MSTest project MUST declare parallel execution at assembly level:
  ```csharp
  [assembly: Parallelize(Workers = 0, Scope = ExecutionScope.MethodLevel)]
  ```

### Test Project Naming

- Unit test projects: `<ServiceName>.UnitTests` (e.g., `Ordering.UnitTests`).
- Functional test projects: `<ServiceName>.FunctionalTests` (e.g., `Catalog.FunctionalTests`).

### Unit Test Targets and Minimum Coverage

**Domain layer** (aggregates, value objects, domain events):

- MUST achieve 80% line coverage minimum.
- Every domain invariant that throws a domain exception MUST have a dedicated negative test.
- Test file locations: `Domain/` folder inside the unit test project.

**Command handlers**:

- MUST achieve 80% line coverage minimum.
- MUST test the happy path and every early-exit branch (idempotency check, failed dispatch).

**API endpoint handlers**:

- 100% of defined routes MUST have at least one happy-path test.
- Every typed error result (`BadRequest<TDetail>`, `NotFound`, `ProblemHttpResult`) MUST have a
  dedicated test.
- Always test handlers **directly** as `public static` methods; never use `WebApplicationFactory`
  for handler logic unit tests.

**gRPC services**:

- MUST cover the authenticated, unauthenticated, and empty-data cases for every RPC method.
- Use `TestServerCallContext.Create(...)` from `Grpc.Core.Testing`; set user state via
  `serverCallContext.SetUserState("__HttpContext", httpContext)`.

**Validators** (`AbstractValidator<T>`):

- MUST have tests for valid input, each invalid field rule, and boundary values.

### Functional Test Targets

- Functional tests MUST use `WebApplicationFactory<Program>` combined with real infrastructure
  provisioned via Aspire test containers (Docker is required in CI).
- Every versioned API endpoint MUST have at least one functional test per supported version.
- Functional tests MUST verify HTTP status codes, response body shape, and pagination contracts
  where applicable.
- Functional tests are **NOT** a substitute for unit tests; both layers are mandatory.
- Use `IAsyncLifetime` (xunit) on `WebApplicationFactory` fixtures to start/stop Aspire
  infrastructure before/after the test class runs.

### Coverage Exclusions

The following are explicitly excluded from coverage requirements:

- `Program.cs` and `Program.Testing.cs`
- EF Core migration files
- Generated protobuf code (`.proto`-generated files)
- `AppHost` projects
- Seed data classes

### Test Style

- Always use the **Arrange / Act / Assert** pattern with comment separators (`// Arrange`,
  `// Act`, `// Assert`).
- Always name test methods to describe the scenario and expected outcome in plain English,
  using underscores as separators: `Cancel_order_bad_request`, `GetBasket_returns_empty_for_no_user`.
- Never assert on implementation details; always assert on observable outputs (TypedResults
  return types, state changes, published events).
- Always assert on the `.Result` property of discriminated union returns:
  `Assert.IsInstanceOfType<Ok>(result.Result)`.

### Coverage Enforcement

- Configure a minimum coverage gate of **80%** for `*.Domain` and `*.Application` namespaces in CI.
- Coverage below this threshold MUST block merge.

---

## 17. General C# Conventions

- Always use **primary constructors** for classes whose only initialization is field assignment
  from constructor parameters (e.g., `OrderQueries(OrderingContext context)`).
- Always mark properties that must be set at construction time as `required` (e.g.,
  `public required DbSet<CatalogItem> CatalogItems { get; set; }`).
- Always use `ValueTask` for hot-path async operations (AI generation, cache reads); use `Task`
  for infrequent operations.
- Always use UTF-8 string literals (`"prefix"u8.ToArray()`) for byte-array constants; never
  call `Encoding.UTF8.GetBytes(...)` for compile-time constant values.
- Always apply `[JsonStringEnumConverter]` (or `HasConversion<string>()` in EF) to enum types
  that cross service or persistence boundaries.
- Always expose collection properties as `IReadOnlyCollection<T>`; never expose `List<T>`,
  `IList<T>`, or arrays as public API.
- Always use top-level `Program.cs` (no explicit `Program` class, no `static void Main`);
  add `public partial class Program { }` in `Program.Testing.cs` for functional test support.
- Always use local `static` helper functions inside `Program.cs` for pure utility logic
  (e.g., `ShouldUseHttpForEndpoints()`); never add unnecessary helper classes.
- Always use `[DoesNotReturn]` on methods that always throw (e.g., `ThrowNotAuthenticated()`).
- Always use `ArgumentNullException.ThrowIfNull(...)` instead of manual null checks in
  constructors.

---

## 18. Formatting & Style

All rules are derived from `.editorconfig` at the repository root.

### Indentation & Encoding

- `.cs` files: 4-space indent, `utf-8-bom` charset, `insert_final_newline = true`.
- `.csproj`, `.props`, `.targets`, `.config`: 2-space indent.
- All files: `indent_style = space`; never use tabs.

### Usings

- Always sort `System.*` usings first: `dotnet_sort_system_directives_first = true`.
- Use `ImplicitUsings`; minimize explicit usings in individual files.

### `var`

- Always use `var` everywhere (`csharp_style_var_for_built_in_types`,
  `csharp_style_var_when_type_is_apparent`, `csharp_style_var_elsewhere` all `true`).

### Expression Bodies

- **Allowed** on properties, indexers, and accessors.
- **Forbidden** on methods and constructors.

### Braces

- Always use braces for every block (`csharp_prefer_braces = true`).
- Always place `{` on a new line (`csharp_new_line_before_open_brace = all`).
- Always place `else`, `catch`, `finally` on a new line.

### Qualifiers

- Never use `this.` qualification; `dotnet_style_qualification_for_*` are all `false`.
- Always use language keywords over BCL type names (`string` not `String`,
  `int` not `Int32`, etc.).

### Fields

- Always prefer `readonly` fields: `dotnet_style_readonly_field = true`.
- Always name `const` fields in PascalCase.

### Pattern Matching

- Always use pattern matching over `is` with cast and `as` with null check.
- Always use null propagation (`?.`) and null coalescing (`??`) over explicit null checks.
- Always prefer `throw` expressions and conditional delegate calls.

### Initializers

- Always prefer object initializers and collection initializers.

### Modifier Order

```
public private protected internal static extern new virtual abstract sealed
override readonly unsafe volatile async
```

---

## 19. Governance & Evolution

- This constitution is a **living document**; it MUST be updated whenever a coding standard
  changes.
- Coding standards MAY evolve when a change can be demonstrated to improve correctness,
  maintainability, or alignment with the .NET ecosystem's direction.
- All proposed standard changes MUST be reviewed and approved by the project committee before
  adoption; no individual pull request may unilaterally introduce a new pattern, library, or
  architectural deviation without prior committee approval.
- Approved changes MUST be reflected in this document before implementation begins.
- Coverage thresholds and test requirements are subject to the same approval process.
- When a new external package is proposed, the author MUST demonstrate that no existing package
  in `Directory.Packages.props` satisfies the requirement before the proposal is considered.
- Breaking changes to integration event schemas require a deprecation period; old and new event
  handlers MUST coexist for at least one release cycle.
- Any deviation from a rule in this document discovered during code review MUST be treated as a
  blocking defect; it may not be merged until either the code is fixed or the constitution is
  amended through the approval process.

---

**Version**: 1.0.0 | **Ratified**: 2026-05-16 | **Last Amended**: 2026-05-16
