# Constitution Prompt 005

> Analyze the **entire** eShop solution codebase without exception — every project, every `.csproj` and its package references, Directory.Packages.props, Directory.Build.props, global.json, .editorconfig, every Program.cs, every service registration extension, the full domain model, infrastructure layer, API layer, test projects, and the AppHost orchestration program.
>
> Produce a **Developer Constitution** in Markdown suitable for use as a copilot-instructions.md file. This document is the single source of truth for how all code in this repository must be written — by humans and AI assistants alike. Every rule must be derived from evidence found in the actual codebase or its configuration files. Do not invent guidance that has no basis in the existing code.
>
> Write every rule as a clear, unambiguous prescription: **"always X", "never Y", "use Z not W"**. Avoid vague recommendations.
>
> The constitution must cover the following sections:
>
> 1. **Runtime & SDK** — target framework, SDK pinning, build properties, artifact output conventions
> 2. **Package Governance** — central package management rules; the introduction of any new NuGet package requires committee approval before it is added to Directory.Packages.props; justify new packages against existing ones already in the solution
> 3. **Architecture** — enumerate every service with its bounded context and responsibility; document all architectural patterns in use (microservices, CQRS, DDD, event-driven, BFF, outbox, idempotency, saga); any new service must conform to these patterns
> 4. **Extensibility Contract** — when adding a new API service or third-party integration, state the mandatory checklist: register in `eShop.AppHost`, add service defaults, expose via the established Minimal API pattern, publish/subscribe through the event bus, and reflect any required UI changes in `WebApp` or `WebAppComponents`; new integrations must not bypass the event bus or introduce direct service-to-service HTTP calls outside of service discovery; every new service must ship with its own unit test project and be covered by at least one functional test before merge
> 5. **Orchestration** — .NET Aspire rules for declaring infrastructure, service references, health checks, and container lifetimes
> 6. **Service Defaults** — what every service must wire up through `eShop.ServiceDefaults` and what must never be configured per-project
> 7. **API Design** — Minimal API conventions, versioning, typed results, OpenAPI metadata, `[AsParameters]`, testability requirements for handler methods
> 8. **CQRS & MediatR** — command/query folder structure, command immutability, idempotency via `IdentifiedCommand<T>`, mandatory pipeline behavior registration order (Logging → Validation → Transaction), FluentValidation conventions
> 9. **Domain-Driven Design** — aggregate roots, entities, value objects, domain events, repository interface placement, collection encapsulation, invariant enforcement rules
> 10. **Persistence** — EF Core configuration conventions, schema-per-service, HiLo sequences, owned entities, enum-as-string storage, `IEntityTypeConfiguration<T>` per entity, migration commands
> 11. **Integration Events** — outbox pattern, transactional publishing, subscription registration, AOT-safe `JsonSerializerContext` serialization
> 12. **Authentication & Authorization** — JWT bearer configuration for APIs, OIDC + cookie for the web app, `sub` claim mapping removal, `appsettings.json` `Identity` section convention, route-level `.RequireAuthorization()`
> 13. **Caching & Storage** — Redis key naming, UTF-8 byte serialization, `JsonSerializerContext` source generation for cache values
> 14. **AI Integration** — `Microsoft.Extensions.AI` abstractions, optional enablement pattern, provider selection via configuration, pgvector storage, embedding dimension trimming, OTel tracing for AI calls
> 15. **Observability** — structured log message template conventions, `@` destructuring, `LogLevel.Trace` guard pattern, transaction scope logging, OTLP export detection
> 16. **Testing** — derive the full testing strategy from the existing test projects (`Ordering.UnitTests`, `Basket.UnitTests`, `Catalog.FunctionalTests`, `Ordering.FunctionalTests`, `ClientApp.UnitTests`). The constitution must specify:
>     - **Framework & tooling**: MSTest with `Microsoft.Testing.Platform`; NSubstitute for mocking (never Moq); `NullLogger<T>` for logger dependencies in unit tests; GlobalUsings.cs in every test project; parallel execution via `[assembly: Parallelize]`
>     - **Test project naming**: `<ServiceName>.UnitTests` for unit tests, `<ServiceName>.FunctionalTests` for integration/functional tests
>     - **Unit test targets and minimum coverage**:
>       - **Domain layer** (aggregates, value objects, domain events): 80 % line coverage minimum; every domain invariant that throws a domain exception must have a dedicated negative test
>       - **Command handlers**: 80 % line coverage minimum; test the happy path and every early-exit branch (idempotency check, failed dispatch)
>       - **API endpoint handlers**: 100 % of defined routes must have at least a happy-path test and every typed error result (`BadRequest`, `NotFound`, `ProblemHttpResult`) must have a dedicated test; test handlers directly as `public static` methods — do not use `WebApplicationFactory` for handler logic
>       - **gRPC services**: cover the authenticated, unauthenticated, and empty-data cases for every RPC method
>       - **Validators**: every `AbstractValidator<T>` must have tests for valid input, each invalid field rule, and boundary values
>     - **Functional test targets**:
>       - Functional tests use `WebApplicationFactory<Program>` with real infrastructure via Aspire test containers (Docker required)
>       - Every versioned API endpoint must have at least one functional test per supported version
>       - Functional tests must verify HTTP status codes, response shape, and pagination contracts where applicable
>       - Functional tests are not a substitute for unit tests — both layers are mandatory
>     - **What is explicitly excluded from coverage requirements**: Program.cs, EF Core migrations, generated protobuf code, `AppHost`, seed data classes
>     - **Test style**: always use the Arrange / Act / Assert pattern with comment separators; test method names must describe the scenario and expected outcome in plain English (e.g., `Cancel_order_bad_request`); never assert on implementation details, only on observable outputs (`TypedResults` return types, state changes, published events)
>     - **Coverage enforcement**: configure a minimum coverage gate of 80 % for the `*.Domain` and `*.Application` namespaces in CI; coverage below this threshold must block merge
> 17. **General C# Conventions** — primary constructors, `required` properties, `ValueTask` for hot paths, `u8` string literals, `JsonStringEnumConverter` on cross-boundary enums, `IReadOnlyCollection<T>` for public collection exposure, top-level Program.cs style, local static helper functions
> 18. **Formatting & Style** — derived entirely from .editorconfig: brace placement (`all` new line), `var` everywhere, expression bodies allowed on properties/indexers/accessors but not methods or constructors, always use braces for blocks, pattern matching over casts and null checks, no `this.` qualification, language keywords over BCL types, `readonly` field preference, `const` fields in PascalCase, `System.*` usings sorted first, null propagation and coalescing over explicit null checks, object and collection initializer preference, 4-space indent for `.cs` files, 2-space for `.csproj`/`.props`, `utf-8-bom` charset, files must end with a newline
> 19. **Governance & Evolution** — this constitution is a living document; coding standards may evolve when a change can be demonstrated to improve correctness, maintainability, or alignment with the .NET ecosystem's direction; all proposed standard changes must be reviewed and approved before adoption; no individual pull request may unilaterally introduce a new pattern, library, or architectural deviation without prior committee approval; approved changes must be reflected in this document before implementation begins; coverage thresholds and test requirements are subject to the same approval process
