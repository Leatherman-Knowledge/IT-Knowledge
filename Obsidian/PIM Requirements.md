# PIM System — Replit Build Requirements

Build a Product Information Management (PIM) system. This is a full-stack web application — highly customizable, API-first, and designed to serve multiple downstream platforms. It should match the core functionality of Akeneo PIM while being simpler where possible. The target audience is developers and product data managers.

---

## 1. Product Structure

### 1.1 Product Types

Support three product structures:

- **Simple product** — A standalone product with no variants.
- **Product with one variant axis** — e.g., a T-shirt varying by Size. The parent product holds shared attributes; variant children inherit those values and define their own axis-specific values.
- **Product with two variant axes** — e.g., a T-shirt varying by Color (level 1) and Size (level 2). Attributes cascade downward: root → level 1 → level 2. Each child inherits all attribute values from its parent levels and can override them only where explicitly allowed.

### 1.2 Attribute Inheritance

- Attributes set at a higher level in the variant hierarchy automatically flow down to children.
- Children cannot redefine attributes owned by a parent unless the family configuration explicitly allows overrides at that axis level.

---

## 2. Families

- A **family** defines the set of attributes available to products assigned to it.
- Families control which attributes appear at which variant axis level (root, level 1, level 2).
- Every product must belong to exactly one family.
- Families define **completeness requirements** — which attributes are required per channel and per locale for a product to be considered "complete" (see Section 6).

---

## 3. Categories

- Support **category trees** — hierarchical taxonomies used to organize and classify products.
- A product can belong to multiple categories across one or more category trees.
- Categories are independent from families. They handle classification and navigation, not attribute assignment.

---

## 4. Attributes

### 4.1 General

- Attributes are defined globally and reused across families.
- Each attribute has a code (immutable identifier), a display label, a type, and configuration options.
- Attributes can be grouped into **attribute groups** for organizational clarity (e.g., "Marketing," "Technical," "Logistics").

### 4.2 Attribute Types

Support the following attribute types at minimum:

- **Text** — Short text, single line.
- **Textarea** — Long text, with optional rich text (HTML) support.
- **Boolean** — Yes/no toggle.
- **Number** — Integer or decimal, with optional min/max validation.
- **Measurement** — Numeric value with a unit (e.g., weight in kg/lb, length in cm/in). Support unit families and automatic conversion between units.
- **Price** — Numeric value per currency. A single price attribute supports multiple currencies.
- **Date** — Date picker.
- **Single select** — One value from a predefined option set.
- **Multi select** — Multiple values from a predefined option set.
- **Media / File** — Upload an image or file directly on the attribute.
- **Asset list** — Links to one or more assets from the asset manager (see Section 8).
- **Metaobject reference** — Links to one or more metaobject entries (see Section 7).
- **Product reference** — Links to other products for lightweight associations without defining a full relationship type.

### 4.3 Localization

- Attributes can be marked as **localizable**, meaning they store separate values per locale (e.g., en_US, fr_FR, de_DE).
- Implement a **locale fallback chain** — if a locale value is empty, resolve it by walking a configurable fallback chain (e.g., fr_CA → fr_FR → en_US).
- Fallback logic is configurable at the system level and optionally overridable per channel.

### 4.4 Validation Rules

- Attributes support configurable validation: regex patterns, max length, min/max numeric values, allowed file types, max file size, unique value enforcement, etc.

---

## 5. Channels (Scopes)

- A **channel** represents a distribution destination (e.g., E-commerce, Print Catalog, Retail POS, Amazon).
- Attributes can be marked as **scopable**, meaning they store different values per channel.
- An attribute can be both localizable and scopable simultaneously — its value varies by channel AND locale.
- Each channel defines:
  - Which **locales** are active for it.
  - Which **currencies** are relevant.
  - A **category tree** root that determines which products belong to the channel.
  - **Completeness requirements** — inherited from the family but evaluated per channel.

---

## 6. Completeness & Data Quality

- Calculate **completeness** per product, per channel, per locale — based on required attributes defined in the product's family.
- Surface completeness as a percentage and flag missing required fields in the UI and API.
- Include a **data quality scoring** layer that goes beyond completeness to assess text length, image resolution, spelling, and attribute consistency.
- AI-powered enrichment capabilities feed into completeness and quality workflows (see Section 18).

---

## 7. Metaobjects (Reference Entities)

- Metaobjects are standalone structured records with their own attribute definitions — similar to Akeneo's Reference Entities or Shopify's Metaobjects.
- Each metaobject type has its own schema (a defined set of typed attributes).
- Metaobject entries link to products or variants via a **metaobject reference** attribute type.
- Metaobject attributes can be localizable and scopable.
- Example use cases: brand profiles, material specs, care instructions, compliance records, designer bios.

---

## 8. Asset Manager

### 8.1 Asset Types

- Support multiple **asset families/types** (e.g., "Product Photo," "Lifestyle Image," "Technical Drawing," "Video," "Document").
- Each asset type has its own set of attributes (e.g., alt text, photographer, usage rights, expiration date).

### 8.2 Asset Attributes & Linking

- Assets use the same attribute type system as products (text, date, single select, etc.).
- Assets link to products via an **asset list** attribute assigned to a family.
- Asset attributes can be localizable and scopable.

---

## 9. Product Associations & Relationships

- Support **named relationship types** between products (e.g., "Cross-sell," "Upsell," "Substitute," "Accessory," "Spare Part").
- Relationships can be configured as:
  - **One-way** — Product A relates to Product B, but not vice versa.
  - **Two-way (bidirectional)** — If A relates to B, B automatically relates to A.
- Relationships are manageable in bulk.
- Relationships can optionally carry their own attributes (e.g., sort order, relationship note).

---

## 10. Import / Export

- **Bulk import** products, families, attributes, categories, and assets via CSV, XLSX, or JSON.
- **Bulk export** in the same formats, filterable by family, category, channel, completeness, etc.
- Import supports:
  - Create, update, and delete operations.
  - Dry-run / validation mode before committing changes.
  - Error reporting with line-level detail.
- Export supports:
  - Scheduled/automated exports.
  - Filtered exports per channel, locale, or category.

---

## 11. Audit History, Logging & Versioning

### 11.1 Attribute-Level Change History

- Maintain a complete **history table** for every attribute change on every resource type (product, variant, metaobject, asset, category, etc.).
- Each history entry records: resource type, resource ID, attribute code, old value, new value, user/API client, and timestamp.
- History is queryable via the API.

### 11.2 Product Version Snapshots

Beyond attribute-level history, support full product version snapshots — capturing the complete state of a product (all attributes, all locales, all channels, all linked assets and metaobjects) at a specific point in time.

- **Named snapshots** — Users can manually create a named version (e.g., "Pre-Holiday 2026 Update") that freezes the entire product state.
- **Automatic snapshots** — The system creates a version automatically on key events: first publish, status change, bulk import overwrite, etc.
- **Diff and compare** — Side-by-side comparison of any two versions, showing what changed across all attributes, locales, and channels.
- **Restore / rollback** — Revert a product to a previous version. The rollback itself is logged as a new entry in the audit history.
- **Version retention policy** — Configurable rules for how long versions are retained and when old snapshots are pruned to manage storage.

---

## 12. API

### 12.1 GraphQL API

- Build a comprehensive **GraphQL API** as the primary interface for reading PIM data.
- Support deep querying: fetch a product with its variants, associated assets, metaobject references, and relationship targets in a single request.
- Support filtering, sorting, cursor-based pagination, and field selection.

### 12.2 REST API (Admin / Write Operations)

- Build a **REST API** for administrative and write operations: CRUD on products, families, attributes, categories, assets, metaobjects, channels, locales, etc.
- Follow consistent endpoint conventions and return clear, structured error responses.

### 12.3 Webhooks & Events

- Implement **webhook support** for reacting to changes in real time: product created, product updated, product deleted, completeness changed, asset uploaded, etc.
- Webhooks are configurable per event type, with retry logic and delivery logging.

### 12.4 API Documentation

- Auto-generate interactive API documentation (e.g., GraphQL Playground for GraphQL, Swagger/OpenAPI for REST).
- Every endpoint and field must be documented with descriptions, types, and examples.

---

## 13. User Management & Permissions

- Implement **role-based access control (RBAC)** with granular permissions.
- Permissions are configurable by:
  - Resource type (products, assets, categories, etc.).
  - Locale (e.g., a user can only edit fr_FR values).
  - Channel (e.g., a user can only manage the "E-commerce" channel).
  - Category tree / subtree.
  - Attribute group (e.g., the marketing team can only edit "Marketing" attributes).
- API tokens support scoped permissions matching the same RBAC model.

---

## 14. Search & Filtering

- Implement full-text search across product attributes, labels, and SKUs.
- Support faceted filtering by family, category, completeness, channel, status, and attribute values.
- Support saved filters / views for common queries.
- Search must be fast and scalable — use Elasticsearch or a similar search engine under the hood.

---

## 15. Bulk Operations

- Bulk edit attribute values across a filtered set of products.
- Bulk category assignment and removal.
- Bulk family reassignment.
- Bulk delete with confirmation safeguards.
- Log all bulk operations in the audit history.

---

## 16. Product Status & Workflow

- Products have a **status field** with at minimum: Draft, Active, and Archived/Disabled.
- Support a configurable **approval workflow** where products must be reviewed before being published to a channel.

---

## 17. Authentication — Microsoft Entra ID

All users authenticate via **Microsoft Entra ID** using a **custom app registration** dedicated to this PIM. Do not route authentication through the standard Leatherman APIM login flow — the PIM owns its own Entra integration directly.

### 17.1 Token & Session Handling

- The PIM backend validates access tokens issued by Entra ID on every API request (JWT signature, audience, issuer, expiration).
- Map the access token's claims (oid, preferred_username, roles, groups) to PIM-internal roles and permissions (see Section 13).
- Support **token refresh** via refresh tokens so users are not forced to re-authenticate during active sessions.
- Session lifetimes should align with Leatherman's security policies.

### 17.2 Role Mapping

- Roles are managed entirely within the PIM. Entra ID provides authentication (identity verification and access to the app) — the PIM handles authorization internally.

### 17.3 API Authentication

- **Service-to-service / machine-to-machine** access (e.g., import jobs, scheduled exports, external integrations) uses the **OAuth 2.0 Client Credentials Flow** with a separate Entra app registration or a client secret/certificate scoped to the PIM.
- API tokens issued via client credentials carry scoped permissions just like user tokens (see Section 13).
- All API authentication goes through Entra ID — no standalone API key system that bypasses Entra.

### 17.4 Security Considerations

- Enforce **HTTPS everywhere**.
- Support Entra ID **Conditional Access Policies** (MFA, device compliance, location restrictions) without requiring changes to the PIM itself — the PIM trusts Entra's enforcement.
- Log all authentication events (successful login, failed login, token refresh, token rejection) in the audit history (see Section 11).

---

## 18. AI Enrichment

Build a pluggable AI enrichment layer to accelerate product data entry, improve quality, and reduce manual effort. Enrichment runs on demand or as part of automated pipelines, and integrates with the completeness and data quality systems (Section 6).

All AI enrichment is **assistive, not autonomous** — suggestions are surfaced for human review and approval, never written directly to published product data without confirmation.

### 18.1 Content Generation

- **Description generation** — Given a product's attributes (name, category, key specs), generate marketing descriptions tailored per channel and locale. Users review and approve before publishing.
- **Translation assistance** — Auto-translate localizable attribute values to fill in missing locales, flagged for human review. Use the locale fallback chain as context for what needs translating.

### 18.2 Image Intelligence

- **Image tagging and alt text** — Analyze product images to generate descriptive tags, alt text, and suggested categories. Useful for accessibility compliance and search optimization.

### 18.3 Classification & Suggestions

- **Category suggestion** — Based on a product's name, description, and attributes, suggest the most appropriate categories from existing category trees.
- **Completeness acceleration** — Identify products with low completeness scores and suggest attribute values based on similar products in the same family or category.

### 18.4 Data Quality & Anomaly Detection

- **Anomaly detection** — Flag products with attribute values that look like outliers compared to their family (e.g., a price that's 10x the average, a weight that seems implausible).
- **Duplicate detection** — Surface potential duplicate products based on attribute similarity, helping keep the catalog clean.
