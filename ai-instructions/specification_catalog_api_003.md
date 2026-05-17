# Comprehensive Specification: Generate a Data-Driven Microservice for eShop

This prompt provides a complete specification for creating a new REST microservice following the eShop architecture and conforming to the eShop Developer Constitution. Use this as a blueprint to generate a service called `[ServiceName].API`.

---

## 1. Service Overview & Purpose

Define the **bounded context** and **core responsibilities**:

- **What domain does this service own?** (e.g., Product Catalog, Inventory, Reviews, Promotions)
- **What is the single source of truth?** (e.g., product data, stock levels, customer reviews)
- **What business capabilities does it provide?** (e.g., browsing, CRUD operations, search, validation)
- **How does it integrate with other services?** (define incoming/outgoing events)

---

## 2. Core Business Capabilities

### 2.1 Primary Domain Operations

Define the main business functions:

**Data Management:**

- Browsing/listing with pagination and filtering
- Detail retrieval (single and batch operations)
- CRUD operations (Create, Read, Update, Delete)
- Search capabilities (text-based and/or semantic)

**Domain-Specific Logic:**

- Business rules and invariants (e.g., stock thresholds, validation rules)
- Domain operations encapsulated in entity methods
- State transitions and lifecycle management

**Supporting Data:**

- Lookup tables and reference data (e.g., categories, types, statuses)
- Hierarchical or relational data structures

### 2.2 Advanced Features (Optional)

Consider including:

- **AI-Powered Features**: Semantic search, recommendations, embeddings
- **Media Management**: Images, files, or binary content serving
- **Caching Strategy**: For frequently accessed data
- **Business Analytics**: Aggregations, reporting endpoints

---

## 3. Integration & Event-Driven Communication

### 3.1 Incoming Integration Events (Subscribes To)

Define which events from other services this service responds to:

| Event Name                | Source Service | Trigger Condition | Action Required  | Response Event(s)  |
| ------------------------- | -------------- | ----------------- | ---------------- | ------------------ |
| `[Event]IntegrationEvent` | [Service]      | When [condition]  | [Action to take] | [Events published] |

**Example Pattern:**

- **OrderStatusChangedToAwaitingValidationIntegrationEvent**
  - Source: Ordering.API
  - Trigger: Order submitted, needs validation
  - Action: Validate available stock for items
  - Response: OrderStockConfirmedIntegrationEvent OR OrderStockRejectedIntegrationEvent

### 3.2 Outgoing Integration Events (Publishes)

Define events this service publishes when state changes:

| Event Name                | Trigger Condition | Payload         | Purpose           | Consuming Services    |
| ------------------------- | ----------------- | --------------- | ----------------- | --------------------- |
| `[Event]IntegrationEvent` | When [condition]  | [Data included] | [Business reason] | [Which services care] |

**Example Pattern:**

- **ProductPriceChangedIntegrationEvent**
  - Trigger: Product price updated via API
  - Payload: ProductId, NewPrice, OldPrice
  - Purpose: Notify dependent services of pricing changes
  - Consumers: Ordering.API, Basket.API

### 3.3 Event Handling Patterns

**Outbox Pattern (Mandatory):**

- All events stored in database transaction
- Published after successful commit
- Ensures exactly-once delivery semantics

**Response Events:**

- Some events require immediate response (validation scenarios)
- Some events are fire-and-forget (notifications)

---

## 4. API Design & Endpoints

### 4.1 RESTful Endpoint Structure

**Versioning Strategy:**

- Support multiple API versions (v1, v2)
- Use route groups for version segregation
- Maintain backward compatibility

**Endpoint Categories:**

**Primary Resource Operations:**

```
GET    /api/[resource]/items              - List (paginated, filtered)
GET    /api/[resource]/items/{id}         - Get single item
GET    /api/[resource]/items/by?ids=...   - Batch get
POST   /api/[resource]/items              - Create new
PUT    /api/[resource]/items/{id}         - Update existing
DELETE /api/[resource]/items/{id}         - Delete
```

**Query & Search Operations:**

```
GET    /api/[resource]/items/by/{name}            - Filter by name
GET    /api/[resource]/items/search?q=...         - Text search
GET    /api/[resource]/items/withsemanticrelevance?text=... - AI search
GET    /api/[resource]/items/type/{typeId}/...    - Hierarchical filtering
```

**Lookup Data:**

```
GET    /api/[resource]/types    - Reference data
GET    /api/[resource]/brands   - Reference data
GET    /api/[resource]/categories - Reference data
```

**Media/Assets (if applicable):**

```
GET    /api/[resource]/items/{id}/pic    - Serve images/files
```

### 4.2 Response Patterns

**Success Responses:**

- `200 OK` - Successful query with data
- `201 Created` - Resource created
- `204 No Content` - Successful deletion

**Error Responses:**

- `400 Bad Request` - Validation errors (with ProblemDetails)
- `404 Not Found` - Resource not found
- `409 Conflict` - Business rule violation

**Pagination:**

- All list endpoints support `pageSize` and `pageIndex`
- Response includes total count and current page info

---

## 5. Domain Model & Entities

### 5.1 Primary Entity

Define the main domain entity:

**Properties:**

- Identity (Id)
- Required attributes
- Optional attributes
- Relationships to other entities
- Calculated/derived properties
- Special types (embeddings, JSON, etc.)

**Business Methods:**

- Domain operations that enforce invariants
- State transitions
- Validation logic

**Example Structure:**

```
- Id: int
- Name: string (required)
- Description: string (optional)
- [Domain-specific properties]
- [Relationships to brands/types/categories]
- [Stock/inventory properties if applicable]
- [Embedding vector if AI-enabled]
- Business methods that enforce rules
```

### 5.2 Supporting Entities

**Reference/Lookup Entities:**

- Brands, Types, Categories, Statuses
- Simple structure: Id, Name/Code

**Value Objects (if applicable):**

- Complex attributes encapsulated as value objects
- No identity, compared by value

### 5.3 Domain Exceptions

**Custom Exceptions:**

- `[ServiceName]DomainException` - Base exception
- Thrown when business rules violated
- Contains meaningful error messages

---

## 6. Business Rules & Invariants

Define enforceable business rules:

**Data Validation:**

- ✅ Required fields must be present
- ✅ IDs must be valid (> 0)
- ✅ String lengths and formats

**Domain Constraints:**

- ✅ [Domain-specific rule example: Stock cannot be negative]
- ✅ [Example: Cannot exceed maximum threshold]
- ✅ [Example: State transitions must be valid]

**State Management:**

- ✅ [Example: Status changes trigger events]
- ✅ [Example: Flags cleared/set automatically]

**Integration Rules:**

- ✅ [Example: Price changes must publish events]
- ✅ [Example: Stock reservations must be atomic]

---

## 7. Data Persistence & Infrastructure

### 7.1 Database Technology

**PostgreSQL with EF Core:**

- Use Aspire for configuration
- Schema isolation per service
- Entity Framework Core for ORM

**Special Extensions (if needed):**

- pgvector - for AI embeddings
- PostGIS - for geospatial data
- JSONB - for flexible schemas

### 7.2 Data Organization

**Schema:**

- Dedicated schema per service
- Clear table naming conventions

**Entity Configurations:**

- One configuration class per entity
- Table names, indexes, constraints
- Relationships and navigation

**Migrations:**

- EF Core migrations for versioning
- Auto-apply in development
- Manual in production

### 7.3 Seed Data

**Initial Data:**

- Load from JSON files in Setup/ folder
- Populate lookup tables
- Provide sample/demo data

---

## 8. Project Structure & Organization

### 8.1 Folder Structure

```
src/[ServiceName].API/
├── Apis/                    # Minimal API endpoint definitions
├── Extensions/              # Service configuration extensions
├── Infrastructure/          # Data access layer
│   ├── EntityConfigurations/
│   ├── Exceptions/
│   └── Migrations/
├── IntegrationEvents/       # Event-driven integration
│   ├── EventHandling/
│   └── Events/
├── Model/                   # Domain entities and DTOs
├── Services/                # Domain services (AI, specialized logic)
├── Setup/                   # Seed data files
├── Pics/ or Assets/         # Static files (if applicable)
├── Properties/
├── Program.cs               # Entry point
├── Program.Testing.cs       # Test support
├── GlobalUsings.cs          # Global namespace imports
├── [ServiceName]Options.cs  # Configuration options
├── appsettings.json
└── [ServiceName].API.csproj
```

### 8.2 Key Files Purpose

**Program.cs:**

- Service registration and middleware pipeline
- Call `AddServiceDefaults()` and `AddApplicationServices()`
- Configure API versioning and OpenAPI
- Map endpoints

**Extensions.cs:**

- `AddApplicationServices()` method
- Database context registration
- Event bus subscriptions
- Domain service registration
- Options binding

**GlobalUsings.cs:**

- Common namespace imports
- Reduces verbosity

---

## 9. Architecture Patterns (Mandatory)

### 9.1 Microservices Patterns

**Service Isolation:**

- Owns its database and schema
- No direct database access from other services
- Communicates via events and APIs

**Event-Driven Architecture:**

- RabbitMQ for async messaging
- Outbox pattern for reliability
- Event sourcing for audit trail

**API Gateway Pattern:**

- Services registered in Aspire AppHost
- Service discovery via Aspire
- Health checks for monitoring

### 9.2 eShop-Specific Patterns

**Minimal API (Required):**

- Static extension methods for route groups
- No MVC controllers
- Typed results (Ok<T>, NotFound, etc.)

**Service Defaults (Required):**

- `AddServiceDefaults()` for telemetry and health
- `AddDefaultOpenApi()` for API documentation
- `MapDefaultEndpoints()` for health/metrics

**Outbox Pattern (Required):**

- `IntegrationEventLogEF` for event persistence
- `ResilientTransaction` for atomicity
- Publish after commit

**API Versioning (Required):**

- Route-based versioning
- Multiple versions supported
- Backward compatibility

---

## 10. Technical Requirements

### 10.1 Framework & SDK

- **Target Framework:** net10.0
- **Nullable Reference Types:** Enabled
- **Implicit Usings:** Enabled

### 10.2 Key Dependencies

**Infrastructure:**

- `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL`
- `Asp.Versioning.Http`
- `Microsoft.EntityFrameworkCore.Tools`

**eShop Shared:**

- `eShop.ServiceDefaults`
- `EventBusRabbitMQ`
- `IntegrationEventLogEF`

**Optional (Based on Features):**

- `Aspire.Azure.AI.OpenAI` - for AI capabilities
- `CommunityToolkit.Aspire.OllamaSharp` - for local AI
- `Pgvector` + `Pgvector.EntityFrameworkCore` - for embeddings

### 10.3 Configuration

**appsettings.json Structure:**

- Logging configuration
- OpenAPI metadata
- EventBus connection and client name
- Service-specific options

**Environment-Specific:**

- Development vs. Production settings
- Feature flags (e.g., AI enabled)
- Connection strings managed by Aspire

---

## 11. Aspire AppHost Integration

### 11.1 Service Registration

```csharp
var [serviceName]Api = builder.AddProject<Projects.[ServiceName]_API>("[servicename]-api")
    .WithReference(postgres)
    .WaitFor(postgres)
    .WithReference(rabbitMq)
    .WaitFor(rabbitMq)
    .WithHttpHealthCheck("/health");
```

### 11.2 Dependencies

**Infrastructure Dependencies:**

- PostgreSQL (with pgvector if AI-enabled)
- RabbitMQ for event bus
- Redis (if caching required)

**Service Dependencies:**

- Wait for dependencies before starting
- Service discovery for inter-service calls

---

## 12. Testing Strategy

### 12.1 Unit Tests

**Project:** `tests/[ServiceName].UnitTests/`

**Focus:**

- Domain entity business logic
- Validation rules
- Domain exceptions
- Business rule enforcement

### 12.2 Functional Tests

**Project:** `tests/[ServiceName].FunctionalTests/`

**Focus:**

- API endpoint integration
- Database interactions
- Event publishing/handling
- End-to-end scenarios

**Pattern:**

- Use `WebApplicationFactory<Program>`
- Requires Program.Testing.cs with `public partial class Program { }`
- Requires `<InternalsVisibleTo>` in csproj

---

## 13. Observability & Operations

### 13.1 Health Checks

- `/health` endpoint via `MapDefaultEndpoints()`
- Database connectivity check
- Event bus connectivity check

### 13.2 OpenTelemetry

- Automatic tracing via ServiceDefaults
- Distributed tracing across services
- Metrics and logging

### 13.3 OpenAPI Documentation

- Scalar UI (not Swagger)
- Auto-generated from endpoint metadata
- Includes descriptions, examples, response types

---

## 14. Constitution Compliance Checklist

**Mandatory Compliance with eShop Developer Constitution:**

- [ ] Uses net10.0 target framework
- [ ] Minimal API only (no MVC controllers)
- [ ] API versioning implemented
- [ ] PostgreSQL via Aspire
- [ ] Outbox pattern for events
- [ ] RabbitMQ event bus subscriptions
- [ ] Service Defaults integration
- [ ] Health checks configured
- [ ] Proper global usings
- [ ] Entity configurations separated
- [ ] Testing projects created
- [ ] Program.Testing.cs for test support
- [ ] InternalsVisibleTo for test access
- [ ] Follows Central Package Management
- [ ] No new packages without committee approval

---

## 15. Use Cases & Scenarios

Define the primary use cases this service enables:

1. **[User/Service] browsing/searching [resources]**
   - Description of the scenario
   - API endpoints involved
   - Expected behavior

2. **[System] validating [business process]**
   - Integration event flow
   - Validation logic
   - Response handling

3. **[Process] updating and notifying**
   - State change triggers
   - Events published
   - Downstream impacts

4. **[Feature] with advanced capabilities**
   - Special features (AI, analytics, etc.)
   - Technical implementation notes

---

## Implementation Phases (Future Work)

**Phase 1: Foundation**

- Project structure and configuration
- Database context and migrations
- Basic CRUD endpoints

**Phase 2: Domain Logic**

- Entity business methods
- Event handling implementation
- Business rule enforcement

**Phase 3: Advanced Features**

- AI/ML integration (if applicable)
- Search optimization
- Performance tuning

**Phase 4: Production Readiness**

- Comprehensive testing
- Documentation
- Monitoring and alerting

---

## Example: Catalog.API Reference

This specification is modeled after the **Catalog.API** service, which:

- Manages product catalog (name, description, price, stock)
- Organizes products by brands and types
- Implements inventory management with business rules
- Provides AI-powered semantic search using embeddings
- Serves product images
- Integrates with Ordering service via events
- Validates stock before order processing
- Decrements inventory after payment

Use this as a reference implementation pattern for your new service.

---

## Conclusion

This comprehensive specification provides the blueprint for creating a robust, scalable, event-driven microservice that integrates seamlessly into the eShop ecosystem. Focus on defining clear boundaries, business capabilities, and integration points before implementation.
