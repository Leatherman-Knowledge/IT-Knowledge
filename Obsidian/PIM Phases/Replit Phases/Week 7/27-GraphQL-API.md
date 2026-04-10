# Phase 27: GraphQL API

**Execution Plan Item:** #27  
**Replit Phase:** 27 (Week 7)  
**Dependencies:** Phase 08 (Products), Phase 09 (Product Attribute Values), Phase 11 (Completeness), Phase 12.1 (Inheritance & Completeness Corrections), Phase 17 (API Consistency), Phase 19 (RBAC), Phase 20.6 (API Key Auth), Phase 26 (real imported data preferred for testing)  
**Depended on by:** Phase 30 (UI data exploration), Phase 35 (MCP Server)

---

## Objective

Build a **GraphQL read-side query layer** on top of the existing REST API and data model. GraphQL is additive — it does not replace REST; all existing REST endpoints remain unchanged. GraphQL provides richer queries for product data consumers that need related data in a single round-trip: products with their variants, resolved attribute values by channel/locale, category assignments, family metadata, and completeness in one request.

Serve a **GraphiQL** in-browser IDE at `/graphql` for development and staging use. Authentication uses the same mechanism as the REST API (Bearer API key or Entra session).

---

## 1. Why GraphQL Here

The PIM data model is deeply relational: products have variants, variants have attribute values, values are scoped by channel and locale, products are assigned to categories, families define the attribute set. REST forces callers to make N+1 requests to assemble a full product view. GraphQL lets the Phase 35 MCP server, the UI, and external consumers request exactly what they need in one query without over- or under-fetching.

This phase builds the **read layer only**. Mutations (create/update products via GraphQL) are out of scope for MVP; use the REST API for writes.

---

## 2. Schema Design

### Root Types

```graphql
type Query {
  product(identifier: String!): Product
  products(
    familyId: Int
    status: ProductStatus
    type: ProductType
    parentIdentifier: String
    root: Boolean
    search: String
    limit: Int
    cursor: String
  ): ProductConnection!

  family(id: Int!): Family
  families(limit: Int, cursor: String): FamilyConnection!

  category(id: Int!): Category
  categories(parentId: Int, root: Boolean, limit: Int, cursor: String): CategoryConnection!

  attribute(id: Int!): Attribute
  attributes(groupId: Int, type: AttributeType, limit: Int, cursor: String): AttributeConnection!

  channel(code: String!): Channel
  channels: [Channel!]!

  locale(code: String!): Locale
  locales: [Locale!]!
}
```

### Product Type

```graphql
type Product {
  identifier: String!
  productNumber: Int
  family: Family!
  variantDefinition: FamilyVariantDefinition
  parent: Product
  children(limit: Int, cursor: String): ProductConnection!
  ancestors: [Product!]!
  type: ProductType!
  level: Int!
  axisValues: JSON
  status: ProductStatus!
  categories: [Category!]!
  values(channel: String, locale: String): [AttributeValue!]!
  completeness(channel: String!, locale: String!): CompletenessResult
  allCompleteness: [CompletenessResult!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum ProductType {
  product
  model
  variant
}

enum ProductStatus {
  draft
  active
  archived
}
```

### Attribute Value Type

```graphql
type AttributeValue {
  attribute: Attribute!
  channelCode: String
  localeCode: String
  value: JSON
  source: ValueSource!
}

enum ValueSource {
  own
  inherited
  fallback
}
```

The `value` field is typed as scalar `JSON` since attribute values are heterogeneous (text, number, boolean, select option, etc.). Consumers receive the raw value and are responsible for interpreting it per the attribute's `type`.

### Family Types

```graphql
type Family {
  id: Int!
  labels: JSON!
  attributes: [FamilyAttribute!]!
  variantDefinitions: [FamilyVariantDefinition!]!
}

type FamilyAttribute {
  attribute: Attribute!
  isRequired: Boolean!
  requiredChannels: [String!]!
}

type FamilyVariantDefinition {
  id: Int!
  labels: JSON!
  levels: [VariantLevel!]!
}

type VariantLevel {
  level: Int!
  axes: [Attribute!]!
  attributes: [Attribute!]!
}
```

### Attribute Types

```graphql
type Attribute {
  id: Int!
  code: String!
  labels: JSON!
  type: AttributeType!
  group: AttributeGroup
  localizable: Boolean!
  scopable: Boolean!
  isUnique: Boolean!
  validationRules: JSON
  options: [AttributeOption!]
}

type AttributeGroup {
  id: Int!
  labels: JSON!
  sortOrder: Int!
  attributes: [Attribute!]!
}

type AttributeOption {
  id: Int!
  code: String!
  labels: JSON!
  sortOrder: Int!
}

enum AttributeType {
  text
  textarea
  boolean
  number
  measurement
  price
  date
  singleSelect
  multiSelect
  assetSingle
  assetList
  entitySingle
  entityList
  table
}
```

### Category Type

```graphql
type Category {
  id: Int!
  labels: JSON!
  parent: Category
  children: [Category!]!
  products(limit: Int, cursor: String): ProductConnection!
}
```

### Completeness Types

```graphql
type CompletenessResult {
  channelCode: String!
  localeCode: String!
  percentage: Int!
  complete: Int!
  required: Int!
  missingAttributes: [MissingAttribute!]!
}

type MissingAttribute {
  attribute: Attribute!
}
```

### Connection / Pagination Types

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
  total: Int!
}

type ProductEdge {
  node: Product!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

Use the same cursor-based pagination pattern for `FamilyConnection`, `CategoryConnection`, `AttributeConnection`, etc.

### Scalar Types

```graphql
scalar JSON
scalar DateTime
```

---

## 3. Key Query Patterns

The following patterns are explicitly required in the spec and should be tested end-to-end:

### Pattern 1 — Product with full variant tree

```graphql
query ProductTree($identifier: String!) {
  product(identifier: $identifier) {
    identifier
    type
    status
    family {
      id
      labels
    }
    children {
      edges {
        node {
          identifier
          type
          axisValues
          children {
            edges {
              node {
                identifier
                type
                axisValues
                status
              }
            }
          }
        }
      }
    }
  }
}
```

### Pattern 2 — Product values by channel and locale

```graphql
query ProductValues($identifier: String!, $channel: String!, $locale: String!) {
  product(identifier: $identifier) {
    identifier
    values(channel: $channel, locale: $locale) {
      attribute {
        code
        type
        labels
      }
      value
      source
      channelCode
      localeCode
    }
  }
}
```

When `channel` and/or `locale` are omitted from the `values` field, return all stored values (all channels, all locales) without inheritance resolution. When they are provided, apply the full variant inheritance + locale fallback resolution chain as implemented in Phase 09 / Phase 12.1.

### Pattern 3 — Product list by family with completeness

```graphql
query ProductsByFamily($familyId: Int!, $channel: String!, $locale: String!) {
  products(familyId: $familyId, root: true, limit: 25) {
    edges {
      node {
        identifier
        status
        completeness(channel: $channel, locale: $locale) {
          percentage
          complete
          required
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    total
  }
}
```

### Pattern 4 — Default: all channels and all locales

When no `channel` or `locale` is specified in a `values` query, return the raw stored values across all scopes. This is the "export all" view for integrations and MCP access. When `channel` and `locale` are specified, resolve with full inheritance + fallback (Phase 12.1 semantics).

---

## 4. Implementation Approach

### Library

Use **`graphql-yoga`** (or **`apollo-server-express`**) as the GraphQL server. Both integrate cleanly with the existing Express app. `graphql-yoga` is preferred for its simpler setup and modern defaults; it can be swapped later if needed.

Add to `server/graphql/`:
- `schema.ts` — type definitions (SDL strings or GraphQL schema object)
- `resolvers/index.ts` — root resolver map
- `resolvers/product.ts` — `Query.product`, `Query.products`, `Product.*` field resolvers
- `resolvers/family.ts`, `resolvers/attribute.ts`, `resolvers/category.ts`, etc.
- `resolvers/completeness.ts` — delegates to existing completeness service
- `context.ts` — builds resolver context from request (user/API key, storage instance)

Mount the GraphQL endpoint in `server/routes.ts`:

```typescript
app.use('/graphql', graphqlHandler);
```

The handler enforces the same RBAC as the REST API (see §6).

### DataLoader Pattern (N+1 Prevention)

Resolver fields that fan out (e.g., `Product.family`, `Product.values`, `AttributeValue.attribute`) must use **DataLoader** to batch database calls per request. Without DataLoaders, fetching a list of 25 products where each requests its family results in 25 separate family queries.

Required DataLoaders (add to context per request):

| DataLoader | Key | Batches |
|---|---|---|
| `familyLoader` | family id | `getFamiliesByIds(ids)` |
| `attributeLoader` | attribute id | `getAttributesByIds(ids)` |
| `attributeGroupLoader` | group id | `getAttributeGroupsByIds(ids)` |
| `categoryLoader` | category id | `getCategoriesByIds(ids)` |
| `productLoader` | product identifier | `getProductsByIdentifiers(identifiers)` |
| `productValuesLoader` | `${identifier}:${channel}:${locale}` | Batch value loads |

Add the required batch storage methods (`getAttributesByIds`, `getFamiliesByIds`, etc.) to `IStorage` if they do not already exist.

### Resolver for `Product.values`

The `values` resolver on a product must invoke the same inheritance-resolution logic used by the REST endpoint `GET /api/v1/products/:identifier/values`. It should call the existing service function rather than re-implementing it, to ensure GraphQL and REST return identical resolved values.

### Resolver for `Product.completeness`

Delegates directly to the existing `calculateCompleteness(identifier, channelCode, localeCode)` service function (Phase 11 / Phase 12.1). No duplication of completeness logic.

---

## 5. GraphiQL IDE

Serve GraphiQL at `/graphql` in non-production environments (development and staging). In production, gate it behind `integration:manage` permission or disable entirely.

The GraphiQL UI should be pre-loaded with example queries for:
- Product by identifier
- Products by family
- Product with values by channel+locale
- Product with completeness

Configure `graphql-yoga` (or Apollo) to enable the built-in GraphiQL IDE with `NODE_ENV !== 'production'` guard.

---

## 6. Authentication & Authorization

GraphQL requests use the same auth as REST:

- **Browser sessions:** Entra ID session cookie / token (same as existing routes).
- **API key (Bearer token):** API key from Phase 20.6, passed as `Authorization: Bearer <key>`. The GraphQL handler reads the same `req.user` / API key context built by the existing auth middleware.
- **RBAC:** Require `integration:read` or `product:read` (or the equivalent read permission) for any GraphQL query. No public unauthenticated access. Return a GraphQL error (not an HTTP 401) if the request is unauthenticated, formatted as:

```json
{
  "errors": [{ "message": "Unauthorized", "extensions": { "code": "UNAUTHORIZED" } }]
}
```

Per-field authorization is not needed for MVP. A single "authenticated with read access" check on the request is sufficient.

---

## 7. Error Handling

- **Not found:** If `product(identifier: "99999")` resolves to null, return `null` in the response (not an error). Consumers check for null.
- **Validation errors:** Invalid enum values or malformed arguments return a GraphQL validation error (before execution).
- **Internal errors:** Unexpected server errors return a generic `"Internal server error"` message with `extensions.code: "INTERNAL_ERROR"`. Do not expose stack traces or DB error messages in production.

---

## 8. REST API Batch Methods (New Storage Methods)

The following storage methods should be added if they do not already exist, to support DataLoader batching:

| Method | Signature | Description |
|--------|-----------|-------------|
| `getProductsByIdentifiers` | `(identifiers: string[]) → Product[]` | Batch product fetch by PK |
| `getAttributesByIds` | `(ids: number[]) → Attribute[]` | Batch attribute fetch |
| `getFamiliesByIds` | `(ids: number[]) → Family[]` | Batch family fetch |
| `getAttributeGroupsByIds` | `(ids: number[]) → AttributeGroup[]` | Batch group fetch |
| `getCategoriesByIds` | `(ids: number[]) → Category[]` | Batch category fetch |
| `getProductCategoriesByProductIdentifiers` | `(identifiers: string[]) → Map<string, Category[]>` | Batch category assignments |

---

## 9. Exit Criteria

- [ ] GraphQL endpoint at `/api/graphql` (or `/graphql`) responds to POST requests with valid GraphQL queries.
- [ ] `product(identifier)` query returns a product with family, variant tree (children + grandchildren), category assignments, and attribute values with channel/locale parameters.
- [ ] `products(familyId, status, root, limit, cursor)` query returns a paginated list matching the filters. `total` field is accurate.
- [ ] `Product.values(channel, locale)` applies full inheritance + locale fallback resolution (Phase 12.1 semantics). Omitting channel/locale returns raw stored values.
- [ ] `Product.completeness(channel, locale)` returns percentage, counts, and missing attributes. Delegates to existing completeness service.
- [ ] DataLoaders prevent N+1 queries: fetching 25 products with family data results in at most 2 DB queries (1 for products, 1 for families), not 26.
- [ ] GraphiQL IDE accessible at `/graphql` in development; requires auth or disabled in production.
- [ ] Authentication enforced: unauthenticated requests return `UNAUTHORIZED` GraphQL error.
- [ ] No schema definition duplication with REST: value resolution and completeness calculation delegate to shared service functions.

---

## 10. Notes

- **Mutations out of scope.** All writes use REST. GraphQL is read-only in this phase.
- **Subscriptions out of scope.** Real-time event streaming is post-MVP.
- **N+1 is the primary risk.** Every resolver that accesses a related entity must use a DataLoader. Verify with a query that fetches 25 products with family and attribute values and confirm DB query count is bounded.
- **`JSON` scalar.** Attribute values, labels (multi-locale), and axis values are all JSON blobs. Using a `JSON` scalar avoids explosion of the type system for the MVP. Future phases can introduce typed value union types if needed.
- **Phase 35 (MCP Server)** will use this GraphQL layer as one of its transport mechanisms. Keep the schema clean and the resolvers stateless.
