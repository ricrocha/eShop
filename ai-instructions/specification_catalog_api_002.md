Let me analyze the complete API functionality by examining all the endpoints and their implementations.

Read [](file:///c%3A/001_Dev/SPC_KT_TRNNG/eShop/src/Catalog.API/Apis/CatalogApi.cs#100-100), lines 100 to 400

Read [](file:///c%3A/001_Dev/SPC_KT_TRNNG/eShop/src/Catalog.API/Model/CatalogItem.cs#60-60), lines 60 to 107

Read [](file:///c%3A/001_Dev/SPC_KT_TRNNG/eShop/src/Catalog.API/IntegrationEvents/EventHandling/OrderStatusChangedToPaidIntegrationEventHandler.cs)

Read [](file:///c%3A/001_Dev/SPC_KT_TRNNG/eShop/src/Catalog.API/Model/CatalogBrand.cs)

Read [](file:///c%3A/001_Dev/SPC_KT_TRNNG/eShop/src/Catalog.API/Model/CatalogType.cs)

## Catalog.API Functional Description

The **Catalog.API** is a data-driven microservice that manages the product catalog for the eShop e-commerce platform. Here's what it does:

---

### Core Business Capabilities

#### 1. **Product Catalog Management**

- **Product Browsing**: Provides paginated lists of products with filtering by name, brand, and type
- **Product Details**: Returns complete product information including name, description, price, brand, type, and stock levels
- **Product CRUD Operations**:
  - Create new products
  - Update existing products (with special handling for price changes)
  - Delete products
  - Batch retrieval of multiple products by IDs

#### 2. **Product Organization**

- **Brands**: Manages product brands (e.g., "Nike", "Adidas", ".NET")
- **Types/Categories**: Manages product types/categories (e.g., "T-Shirt", "Mug", "Pin")
- Products are categorized by both brand and type for structured browsing

#### 3. **Inventory Management**

Each product tracks:

- **Available Stock**: Current quantity on hand
- **Restock Threshold**: Minimum stock level before reordering
- **Max Stock Threshold**: Maximum warehouse capacity
- **On Reorder Status**: Flag indicating if item is being restocked

**Stock Operations**:

- `RemoveStock(quantity)`: Decreases inventory when orders are paid
- `AddStock(quantity)`: Increases inventory (respects max threshold)
- Enforces business rules (can't remove from empty stock, prevents negative quantities)

#### 4. **AI-Powered Semantic Search** 🤖

- **Embedding Generation**: Automatically generates vector embeddings for product descriptions using:
  - Azure OpenAI (text embedding model), OR
  - Ollama (local embedding model)
- **Semantic Search**: Finds products by meaning/relevance rather than just keyword matching
  - Uses pgvector extension in PostgreSQL for vector similarity search
  - Calculates cosine distance to find most semantically similar products
  - Falls back to name-based search if AI is disabled
- Example: Search for "warm beverage container" finds "coffee mug" even without exact keyword match

#### 5. **Product Images**

- Serves product images (stored as .webp files in the `Pics/` folder)
- Supports multiple image formats (PNG, JPEG, GIF, WebP, SVG, etc.)
- Returns images with proper MIME types and last-modified headers for caching

#### 6. **API Versioning**

- **Version 1.0**: Original API endpoints
- **Version 2.0**: Enhanced endpoints with improved query parameters
  - V1: `/items/by/{name}` - name in route
  - V2: `/items?name=xxx&type=1&brand=2` - query string filters
- Backward compatibility maintained for existing clients

---

### Integration with Other Services (Event-Driven)

#### **Subscribes To** (Incoming Events):

1. **OrderStatusChangedToAwaitingValidationIntegrationEvent**
   - **When**: Order is submitted and needs stock validation
   - **What it does**:
     - Checks if sufficient stock exists for each ordered item
     - Publishes **OrderStockConfirmedIntegrationEvent** if all items in stock
     - Publishes **OrderStockRejectedIntegrationEvent** if any item lacks stock
   - **Purpose**: Validates inventory before order processing

2. **OrderStatusChangedToPaidIntegrationEvent**
   - **When**: Order payment is confirmed
   - **What it does**:
     - Removes the ordered quantity from available stock for each item
     - Updates inventory levels
   - **Purpose**: Commits inventory reservation after payment

#### **Publishes** (Outgoing Events):

1. **ProductPriceChangedIntegrationEvent**
   - **When**: Product price is updated via PUT/UPDATE endpoint
   - **What it does**: Notifies other services (like Ordering) about price changes
   - **Purpose**: Keep pricing consistent across services; may trigger order adjustments

2. **OrderStockConfirmedIntegrationEvent**
   - **When**: All items in an order have sufficient stock
   - **Purpose**: Allows order processing to continue

3. **OrderStockRejectedIntegrationEvent**
   - **When**: One or more items in an order lack sufficient stock
   - **Includes**: List of items with their stock status
   - **Purpose**: Triggers order cancellation or partial fulfillment

---

### RESTful API Endpoints

#### Product Queries (Read Operations)

- `GET /api/catalog/items` - List all products (paginated, with optional filters)
- `GET /api/catalog/items/{id}` - Get single product by ID
- `GET /api/catalog/items/by?ids=1,2,3` - Batch get products
- `GET /api/catalog/items/by/{name}` - Search by name (V1)
- `GET /api/catalog/items/withsemanticrelevance?text={query}` - AI semantic search (V2)
- `GET /api/catalog/items/type/{typeId}/brand/{brandId}` - Filter by type and brand
- `GET /api/catalog/items/{id}/pic` - Get product image

#### Lookup Data

- `GET /api/catalog/catalogtypes` - List all product types
- `GET /api/catalog/catalogbrands` - List all product brands

#### Product Modifications (Write Operations)

- `POST /api/catalog/items` - Create new product
- `PUT /api/catalog/items/{id}` - Update existing product (V2)
- `PUT /api/catalog/items` - Update product (V1 - ID in body)
- `DELETE /api/catalog/items/{id}` - Delete product

---

### Key Technical Features

1. **PostgreSQL with pgvector**: Stores products with vector embeddings for semantic search
2. **Outbox Pattern**: Ensures reliable event publishing (events stored in DB transaction, then published)
3. **RabbitMQ Event Bus**: Asynchronous communication with other microservices
4. **OpenAPI/Scalar**: Auto-generated API documentation with interactive UI
5. **Health Checks**: `/health` endpoint for monitoring
6. **Pagination**: All list endpoints support page size and page index
7. **Database Seeding**: Auto-loads initial catalog from JSON file
8. **EF Core Migrations**: Schema versioning and updates

---

### Business Rules Enforced

- ✅ Cannot remove stock from items with zero inventory (throws `CatalogDomainException`)
- ✅ Cannot add negative quantities to stock
- ✅ Stock cannot exceed maximum warehouse threshold
- ✅ Price changes trigger integration events to notify dependent services
- ✅ Product validation (required name, valid IDs)
- ✅ Automatic reorder flag clearing when stock is added

---

### Use Cases

1. **Customer Browsing**: WebApp calls catalog to display products, filter by category/brand, search semantically
2. **Order Validation**: Ordering service requests stock verification before processing orders
3. **Inventory Updates**: After payment, catalog decrements stock automatically
4. **Price Management**: Admin updates prices; other services are notified to handle impact on pending orders
5. **Semantic Discovery**: "I want something to drink coffee from" → finds mugs, cups, tumblers using AI embeddings

This API serves as the **source of truth for product data and inventory** in the eShop platform, coordinating with the Ordering and Basket services through asynchronous events.
