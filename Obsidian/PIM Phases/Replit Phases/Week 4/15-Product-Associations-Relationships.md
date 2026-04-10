# Phase 15: Product Associations & Relationships

**Execution Plan Item:** #15  
**Replit Phase:** 15 (Week 4)  
**Dependencies:** Products (Phase 08), Entities (Phase 12.2), Families & Variant Definitions (Phases 04, 04.5)  
**Depended on by:** Integration Framework (mapping product links), UI Phase 20 (relationship management UI)

---

## Objective

Build **product associations and relationships**: structured links between products (and optionally entities) that support one-way and two-way associations, variant-level and product-level assignment, and ordering. Every create/update/delete is audit-logged.

Use cases: "Related products," "Upsell," "Cross-sell," "Accessories," "Same designer" (entity link), "Replacement part." Associations can be product-to-product or product-to-entity.

---

## 1. Core Concepts

- An **association type** defines a named relationship (e.g. "Related", "Upsell", "Accessories"). It can be one-way (A → B) or two-way (A ↔ B). Optionally scoped by family (e.g. only products in family "knife" can have "Accessories").
- A **product association** is a single link: one product (or variant) is associated with another product or with an entity, under a given association type. Each association has a sort order for display.
- **Product-level vs variant-level:** Associations can be defined at the Top Model (product) level (all variants inherit) or at a specific variant level. The schema and API support both; the UI can restrict to one or the other per association type if needed.
- **Directionality:** For one-way types, the "source" product has the association; the "target" does not show the reverse. For two-way types, both sides see the link.

---

## 2. Database Schema

### Table: `association_types`

Defines the kinds of relationships that can exist.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | Auto-increment |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | Display name per locale |
| `is_two_way` | `BOOLEAN` | NOT NULL, DEFAULT false | If true, link appears on both source and target |
| `family_id` | `INTEGER` | NULL, FK → families.id | If set, only products in this family can use this type. NULL = all families. |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order in UI |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `product_associations`

Junction table: one row per link. Source is always a product (by identifier). Target can be a product (identifier) or an entity (id).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `association_type_id` | `INTEGER` | NOT NULL, FK → association_types.id, ON DELETE CASCADE | |
| `source_product_identifier` | `VARCHAR(255)` | NOT NULL, FK → products.identifier | The product that "has" this association |
| `target_type` | `VARCHAR(20)` | NOT NULL | `product` or `entity` |
| `target_product_identifier` | `VARCHAR(255)` | NULL, FK → products.identifier | Set when target_type = 'product' |
| `target_entity_id` | `INTEGER` | NULL, FK → entities.id | Set when target_type = 'entity' |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Order among associations of same type on same source |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Constraints:**
- Exactly one of `target_product_identifier` or `target_entity_id` must be set, according to `target_type`.
- Unique on `(association_type_id, source_product_identifier, target_type, target_product_identifier | target_entity_id)` so the same link cannot be added twice.

**Indexes:**
- `(source_product_identifier, association_type_id)` for "list associations for this product"
- `(target_product_identifier, association_type_id)` for two-way lookup when target_type = 'product'
- `(target_entity_id, association_type_id)` for "products linking to this entity"

---

## 3. API Endpoints

### Association Types

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/association-types` | List all association types (optional `familyId` filter). Ordered by sort_order. |
| GET | `/api/v1/association-types/:id` | Get one with optional usage counts. |
| POST | `/api/v1/association-types` | Create. Body: `labels`, `isTwoWay`, `familyId` (optional), `sortOrder` (optional). |
| PATCH | `/api/v1/association-types/:id` | Update labels, isTwoWay, familyId, sortOrder. |
| DELETE | `/api/v1/association-types/:id` | Delete; cascades to all product_associations of this type. |

### Product Associations

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/products/:identifier/associations` | List all associations for this product. Query: `?associationTypeId=1`. Returns target product or entity summary + sortOrder. |
| POST | `/api/v1/products/:identifier/associations` | Add association. Body: `associationTypeId`, `targetType` ('product' \| 'entity'), `targetProductIdentifier` or `targetEntityId`, optional `sortOrder`. Validate type allows source product's family; validate target exists. |
| PATCH | `/api/v1/products/:identifier/associations/:id` | Update sortOrder only (or delete and re-add if needed). |
| DELETE | `/api/v1/products/:identifier/associations/:id` | Remove this association. |

**Two-way behavior:** When `is_two_way` is true, creating A → B should automatically create B → A (or store once and return both directions in GET). Specify in implementation: either two rows (simpler) or one row with symmetric query (single source of truth).

---

## 4. Validation & Business Rules

- **Association type:** `familyId` optional; if set, only products in that family can be source for this type.
- **Target validation:** For target_type = 'product', target must exist and not be the same as source. For target_type = 'entity', entity must exist.
- **Duplicate:** Same (type, source, target) cannot be added twice.
- **Audit:** Log create/update/delete on both `association_type` and `product_associations` (resource types `associationType`, `productAssociation`).

---

## 5. Audit History

- **Resource types:** `associationType`, `productAssociation`.
- **Actions:** create, update, delete.
- **resource_id:** For association type use type id; for product association use association id (or composite key in context).
- **Context:** Include source product identifier and target type/id for productAssociation.

---

## 6. Exit Criteria

- [x] Association types can be created, listed, updated, deleted.
- [x] Product associations can be added from a product to another product or to an entity, with association type and sort order.
- [x] One-way and two-way behavior is correct (directionality visible in API and later in UI).
- [x] All changes are audit-logged.
- [x] Family scoping on association type is enforced when specified.

---

## 7. Notes

- **Variant-level:** Schema above is product-identifier based (so any product in the hierarchy can be source/target). If the product model uses "root product" for associations, document that convention (e.g. only Top Model identifier as source) and enforce in API.
- **Bidirectional sync:** If using two rows for two-way, keep them in sync on create/delete to avoid orphaned reverse links.
