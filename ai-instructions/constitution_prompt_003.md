Here is the complete prompt:

---

> Analyze the entire eShop solution codebase — every project, its `.csproj` references, Directory.Packages.props, Directory.Build.props, global.json, .editorconfig, Program.cs files, service registrations, domain model, infrastructure layer, API layer, tests, and AppHost orchestration.
>
> Produce a **developer constitution** in Markdown suitable for use as a copilot-instructions.md file. It must cover:
>
> 1. **Runtime & SDK** — target framework, SDK pinning, build settings
> 2. **Package conventions** — central package management rules
> 3. **Architecture** — list of services with responsibilities, and the patterns used (microservices, CQRS, DDD, event-driven, BFF)
> 4. **Orchestration** — how .NET Aspire is used; rules for declaring infrastructure and service references
> 5. **Service Defaults** — what every service must wire up and what must never be duplicated per-project
> 6. **API Design** — Minimal API conventions, versioning, typed results, OpenAPI metadata, testability rules
> 7. **CQRS & MediatR** — command/query structure, immutability, idempotency, pipeline behavior order, validation
> 8. **DDD** — aggregate roots, entities, value objects, domain events, repository constraints, encapsulation rules
> 9. **Persistence** — EF Core configuration conventions, schema-per-service, HiLo, owned entities, migrations
> 10. **Integration Events** — outbox pattern, event publishing, subscriptions, AOT-safe serialization
> 11. **Authentication** — JWT bearer for APIs, OIDC for the web app, configuration conventions
> 12. **Caching & Storage** — Redis patterns, serialization
> 13. **AI Integration** — abstractions used, provider selection, vector storage, telemetry, optional enablement
> 14. **Observability** — structured logging conventions, scopes, OTel export
> 15. **Testing** — framework, mocking library, test style, unit vs functional test boundaries
> 16. **General C# conventions** — primary constructors, `required`, `ValueTask`, source-gen JSON, collection exposure rules
> 17. **Formatting & Style** — read the .editorconfig at the repo root and derive every rule prescriptively: brace placement, `var` usage, expression-bodied member rules, pattern matching preferences, `this.` qualification, language keywords vs BCL types, `readonly` fields, `const` naming, using directive ordering, null-safety operators, object/collection initializer preferences, indent sizes per file type, newline and charset settings
>
> For every rule in every section, derive it from actual patterns found in the code and configuration files — do not invent guidance not present in the codebase. Write every rule as a prescriptive instruction ("use X", "never do Y", "always Z") so it works as an AI coding assistant instruction file that enforces consistency with the existing codebase.
