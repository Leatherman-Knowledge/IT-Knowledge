# Phase 31: MCP Documentation API

**Execution Plan Item:** #31  
**Replit Phase:** 31 (Week 8)  
**Dependencies:** Phase 17 (API Consistency — REST endpoints), Phase 19 (RBAC), Phase 20.6 (API Key Auth), Phase 27 (GraphQL API), Phase 29 (Completeness rules)  
**Depended on by:** External MCP server (connects Cursor, Claude Desktop, and custom agents to this API)

---

## Objective

Expose a **structured documentation API** at `/mcp-docs` that serves as a machine-readable reference for LLMs. The API does not return live product data — it exposes the schemas, endpoint definitions, attribute type specs, and examples that allow an LLM to write correct REST and GraphQL API calls against EARL without guessing.

The **MCP protocol layer** (resources, tools, handshake) lives in an **external MCP server** that calls these REST endpoints. The EARL app's responsibility is to serve accurate, structured documentation content. The external server's responsibility is to speak the MCP protocol and route LLM tool calls to the appropriate endpoint here.

When an LLM (in Cursor, Claude Desktop, or a custom agent) needs to interact with EARL programmatically, it consults the MCP server to answer questions like:

- "What are the available REST endpoints for products, and what parameters do they accept?"
- "What does a GraphQL query for a product with its variants look like?"
- "How do I format a `price` attribute value in a PUT request?"
- "What does the completeness endpoint return?"
- "How does channel/locale scoping work in API calls?"

The result is an LLM that can write reliable, correctly-structured API calls on the first attempt — no trial and error, no hallucinated parameters.

---

## 1. Architecture

```
┌─────────────────────────────────┐      ┌──────────────────────────────────┐
│  LLM Client                     │      │  EARL PIM App                    │
│  (Cursor, Claude Desktop, agent)│      │                                  │
│                                 │      │  GET /mcp-docs/rest-endpoints    │
│  ─── MCP tool call ────────────►│      │  GET /mcp-docs/graphql-types/:t  │
│                                 │      │  GET /mcp-docs/attribute-types/:t│
│         ┌───────────────────┐   │      │  POST /mcp-docs/validate-graphql │
│         │ External MCP      │───┼─────►│  GET /mcp-docs/search            │
│         │ Server            │   │      │  GET /mcp-docs/examples          │
│         │ (speaks MCP       │◄──┼──────│  GET /mcp-docs/scoping-guide     │
│         │  protocol)        │   │      │  GET /mcp-docs/entity-model/:e   │
│         └───────────────────┘   │      │  ...etc                          │
└─────────────────────────────────┘      └──────────────────────────────────┘
```

The external MCP server:
- Speaks the MCP protocol (resources, tools, handshake)
- Maps MCP tool calls to `GET /mcp-docs/...` requests
- Returns structured JSON responses to the LLM client

The EARL app:
- Serves accurate, structured JSON documentation
- Has no dependency on `@modelcontextprotocol/sdk`
- Is stateless and DB-free (except `get_scoping_guide` with a specific attribute ID)

---

## 2. What This API Is (and Is Not)

| Is | Is Not |
|----|--------|
| API reference — endpoint paths, methods, parameters, request/response schemas | Live data reader — no product records, no search results |
| GraphQL schema browser — SDL, types, query examples | GraphQL executor — does not run queries |
| Data model guide — attribute types, scoping, identifier formats, pagination conventions | ORM or query builder |
| Example generator — working request/response examples for every endpoint | Proxy or middleware for live API calls |

LLMs that need to **read live data** from EARL do so directly via the REST API or GraphQL endpoint using an API key — they look up the schema here first to know how to make those calls correctly.

---

## 3. Endpoints

All endpoints are under `/mcp-docs`. No authentication required — this is a developer reference tool, equivalent to serving API docs.

### `GET /mcp-docs/rest-endpoints`

Returns a structured array of all REST endpoints.

**Query parameters:**
- `tag` (optional) — filter by resource type: `products`, `families`, `categories`, `attributes`, `entities`, `assets`, `channels`, `locales`, `integrations`, `search`, `completeness`

**Response shape (per endpoint):**
```json
{
  "method": "GET",
  "path": "/api/v1/products/:identifier/values",
  "summary": "Get resolved attribute values for a product",
  "tag": "products",
  "parameters": [...],
  "response_type": "AttributeValue[]"
}
```

---

### `GET /mcp-docs/rest-endpoint`

Full structured documentation for a specific REST endpoint.

**Query parameters:**
- `method` — HTTP method (GET, POST, PUT, PATCH, DELETE)
- `path` — endpoint path, e.g. `/api/v1/products/:identifier/values`

**Response includes:**
- Inline authentication notes
- Structured request body schema (real JSON schema objects, not prose)
- Structured response schema
- Matched example requests/responses
- Relevant error codes

---

### `GET /mcp-docs/graphql-queries`

Returns GraphQL operations split into `queries`, `mutations`, and `subscriptions` arrays. Each entry includes `type`, `description`, structured `arguments`, and `return_type`.

---

### `GET /mcp-docs/graphql-types`

Flat list of all GraphQL type names with `kind` and `description`.

---

### `GET /mcp-docs/graphql-types/:typeName`

Full definition for a single GraphQL type.

**Response includes:**
- All fields with types and descriptions
- `related_type` pointers for schema graph traversal
- `used_in_operations` listing which queries reference this type

---

### `GET /mcp-docs/graphql-schema`

Returns the complete Phase 27 SDL schema string as a single JSON document. Use this when the LLM needs the full type system in one shot.

---

### `GET /mcp-docs/attribute-types`

Flat list of all 14 attribute types with `description` and `value_format` summary.

---

### `GET /mcp-docs/attribute-types/:type`

Full spec for a specific attribute type.

**Supported types:** `text`, `textarea`, `boolean`, `number`, `measurement`, `price`, `date`, `singleSelect`, `multiSelect`, `assetSingle`, `assetList`, `entitySingle`, `entityList`, `table`

**Note:** 14 types are supported. `mediaFile` does not exist as a distinct type — Akeneo file/image types map to `assetSingle`.

**Response includes:**
- Value format for REST (`PUT /api/v1/products/:id/values`) and GraphQL (`AttributeValue.value`)
- Scoping notes (localizable/scopable behavior for this type)
- Validation rules
- `common_mistakes` array — inline LLM guardrails for this type

---

### `GET /mcp-docs/search`

Full-text search across all API documentation.

**Query parameters:**
- `q` — search term, e.g. `completeness`, `cursor pagination`, `price value format`

**Response:** Ranked results with `score`, typed `category` (`rest_endpoint`, `graphql_type`, `attribute_type`, `example`, `guide`), and `follow_up_tool` routing hints pointing to the endpoint to call for full content.

---

### `GET /mcp-docs/examples`

Returns curated working examples.

**Query parameters:**
- `operation` — natural language description of what you want to do, e.g. `"create a product"`, `"query product variants in GraphQL"`, `"write a price value"`, `"filter products by completeness"`

**Response:** Structured example objects with both `rest` and `graphql` versions where available. Includes full curl commands, request/response pairs, and notes.

**Example categories:** `products`, `families`, `categories`, `attributes`, `search`, `completeness`, `graphql`, `integration`, `auth`

---

### `POST /mcp-docs/validate-graphql`

Validates a GraphQL query against the EARL schema without executing it.

**Request body:**
```json
{ "query": "...", "variables": { ... } }
```

**Response:**
- `"Valid query."` if no errors
- Structured error list with `line`, `column`, `path`, and fuzzy suggestions for misspelled field/type names
- `warnings` array for future deprecation support

**Implementation:** Uses `graphql` package's `parse()` + `validate()` against the Phase 27 schema.

---

### `GET /mcp-docs/scoping-guide`

Returns the channel/locale scoping reference.

**Query parameters (both optional):**
- `attributeId` (number) — direct DB lookup for a specific attribute's `localizable`/`scopable` flags. Fast and unambiguous.
- `attributeLabel` (string) — label substring search. Returns a disambiguation list if multiple attributes match.

**Global mode** (no parameters): Returns `dimensions`, full scoping matrix, and both REST `write_format` and `read_format` examples.

**Attribute-specific mode**: Returns `scoping_matrix` with `dimension`/`scoped`/`required`, plus full write and read format examples for both REST and GraphQL APIs.

**Note:** EARL has no `code` column on attributes (removed in Phase 12.9). Pass `attributeId` for reliable results; use `attributeLabel` only when the ID is unknown. This is the **one endpoint that performs a live DB lookup**.

---

### `GET /mcp-docs/pagination-guide`

Structured pagination reference covering:
- REST cursor-based pagination (`limit`, `cursor`, `meta.nextCursor` pattern)
- GraphQL Relay-style connections (`edges`, `node`, `pageInfo`, `endCursor`)
- How to detect the last page (`nextCursor === null`)
- Working examples in both REST and GraphQL

---

### `GET /mcp-docs/auth-guide`

Full authentication model documentation.

**Response includes:**
- RBAC roles (`admin`, `editor`, `viewer`) with their specific permissions
- API key properties and lifecycle
- Example `Authorization: Bearer` headers for both REST and GraphQL
- `common_mistakes` array

---

### `GET /mcp-docs/entity-model/:entityName`

Cross-cutting view of a PIM entity tying together all its relevant API surface.

**Supported entities:** `product`, `family`, `attribute`, `category`, `channel`, `locale`, `asset`, `entity`

**Response includes:**
- REST endpoints for this entity
- GraphQL types related to this entity
- GraphQL operations that query or return this entity
- `attribute_types_used` (where applicable)
- `related_entities` pointers

---

## 4. Content: Curated Examples

The examples served by `GET /mcp-docs/examples` cover the most common operations:

| Category | Examples |
|----------|---------|
| `products` | List all active products; get a product with variants; get a product's attribute values by channel+locale; create a product; update product status; write a text value; write a price value; write a select value |
| `families` | List families; get a family with attributes; get family requirements per channel |
| `categories` | Get category tree; assign a product to a category |
| `attributes` | List attributes; get attribute options for a singleSelect |
| `search` | Full-text search; filter by family + status; filter by completeness |
| `completeness` | Get completeness for a product; get missing attributes for a channel+locale |
| `graphql` | Query product with variants; query values by channel+locale; query product completeness; paginate products by family |
| `integration` | Trigger a full import run; check run status; get run logs filtered by failure |
| `auth` | How to include the API key header in REST; how to include it in GraphQL; how to create an API key in EARL |

---

## 5. Implementation Notes

### Static vs. Dynamic Content

Almost all content is **static** — loaded from source definitions at server startup, not queried from the database. The only exception is `GET /mcp-docs/scoping-guide` when called with a specific `attributeId`, which performs a single DB lookup.

This means the documentation API:
- Adds no meaningful DB load
- Stays consistent even during heavy imports or writes
- Is always fast (sub-millisecond for most responses)
- Must be manually updated when the API changes (treat the docs definitions as a changelog)

### Keeping Docs in Sync

When adding or changing REST endpoints or GraphQL types, update the MCP documentation definitions alongside the code change. The accuracy of the MCP server depends on it.

For MVP, a manual update process is acceptable. A future enhancement could auto-generate endpoint docs from Zod schemas or route registrations.

### External MCP Server Configuration

The external MCP server connects to this API and exposes it to LLM clients. Example Cursor configuration:

```json
{
  "mcpServers": {
    "earl-api": {
      "url": "https://<external-mcp-host>/mcp"
    }
  }
}
```

The external MCP server maps each MCP tool call (e.g. `get_rest_endpoint`) to the corresponding `GET /mcp-docs/...` request on the EARL app.

---

## 6. Exit Criteria

- [x] All documentation endpoints under `/mcp-docs` respond with structured JSON.
- [x] `GET /mcp-docs/rest-endpoints` returns all REST endpoints grouped by tag.
- [x] `GET /mcp-docs/rest-endpoint` returns full structured documentation for any endpoint, including real JSON schemas (not prose) for request/response bodies.
- [x] `GET /mcp-docs/graphql-schema` returns the complete Phase 27 SDL string.
- [x] `GET /mcp-docs/graphql-types/:typeName` returns the full type definition with `related_type` pointers and `used_in_operations`.
- [x] `GET /mcp-docs/attribute-types/:type` returns correct format, validation, and `common_mistakes` for all 14 attribute types.
- [x] `POST /mcp-docs/validate-graphql` returns `"Valid query."` for a valid query and structured errors with line/column/fuzzy suggestions for an invalid one.
- [x] `GET /mcp-docs/examples` returns working examples for `"query product variants in GraphQL"` and `"write a price value"`.
- [x] `GET /mcp-docs/search?q=completeness` returns ranked results pointing to the completeness endpoint doc and completeness data model content.
- [x] `GET /mcp-docs/scoping-guide` (no params) returns the full scoping reference. With `attributeId`, performs a live DB lookup and returns scoping flags with write/read format examples.
- [x] `GET /mcp-docs/auth-guide` returns RBAC roles, key properties, and example headers.
- [x] `GET /mcp-docs/entity-model/product` returns the full cross-cutting product model view.
- [x] External MCP server connects to these endpoints and is reachable by Cursor.
- [x] End-to-end verified: Cursor connected to the external MCP server can answer "How do I get a product's attribute values filtered to the ecommerce channel in English?" without hallucinating parameters.

---

## 7. Notes

- **No live data access.** If an LLM needs to run a query or check actual product data, it does so directly against the EARL REST or GraphQL API using an API key — the MCP documentation API only told it how. This keeps the docs API stateless, DB-free (except `scoping-guide`), and zero-risk for data exposure.
- **`scoping-guide` is the one exception.** Looking up live attribute scoping flags is worth a DB call because scoping flags are attribute-specific and cannot be reliably guessed. All other endpoints are fully static.
- **Update discipline is the main risk.** If endpoint docs drift from actual behavior, LLMs will write incorrect calls. Treat documentation definition changes as part of the definition of done for any API change.
- **Structured schemas over prose.** Response schemas are real JSON schema objects, not prose descriptions. This allows the external MCP server and LLM clients to programmatically inspect request/response shapes.
- **`common_mistakes` arrays.** Every attribute type spec includes an explicit list of common LLM mistakes for that type, providing inline guardrails without requiring the LLM to infer them.
