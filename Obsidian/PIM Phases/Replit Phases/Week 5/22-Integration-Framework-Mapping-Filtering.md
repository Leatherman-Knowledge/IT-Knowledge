# Phase 22: Integration Framework — Mapping & Filtering

**Execution Plan Item:** #22  
**Replit Phase:** 22 (Week 5)  
**Dependencies:** Phase 21 (Integration types & instances), Phase 04/04.5 (Families, variant definitions), Phase 02 (Attributes), Phase 08–09 (Products, attribute values)  
**Depended on by:** Phase 23 (Run engine uses mappings/filters), Phase 24 (UI to build mappings), Phase 25 (Akeneo mapping source → PIM)

---

## Objective

Implement **per-instance field mapping** and **filtering** for the integration framework. For **outbound** integrations: map PIM attributes (and product fields) to target system fields, with optional value transformation (unit conversion, concatenation, conditional logic). Define **product filtering rules** per instance so only the right products are pushed (by family, category, channel, status). For **inbound** integrations: map source fields to PIM attributes (and entities); support scope/filter config (e.g. which families or sources to import). The run engine (Phase 23) and connector implementations (e.g. Phase 25 Akeneo) will read these mappings and filters when executing a run.

---

## 1. Core Concepts

- **Field mapping:** A single rule: "source field → target field" with optional **transform**. For outbound: source = PIM (attribute code, product identifier, status, etc.); target = external system field name. For inbound: source = external field name; target = PIM attribute code or product/entity field.
- **Transform (optional):** A small descriptor for value transformation: e.g. unit conversion, string concatenation, conditional (if empty use default), or custom code reference. Stored as JSON so different integration types can interpret as needed.
- **Product filter (outbound only):** Rules that restrict which products are included in a push: e.g. family Id, category Id(s), channel Id, status (Draft/Active/Archived). All conditions can be AND; optional support for "any of these categories" (OR) if needed.
- **Import scope (inbound):** For imports, the instance may have a scope: e.g. which Akeneo families to import, or "full import." Can be stored in instance `config` or in a dedicated table; this phase defines the structure so Phase 25 can use it.

---

## 2. Database Schema

### Table: `integration_field_mappings`

One row per mapped field for an integration instance. Order matters for display and for deterministic export/import.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `integration_instance_id` | `INTEGER` | NOT NULL, FK → integration_instances.id, ON DELETE CASCADE | |
| `source_type` | `VARCHAR(50)` | NOT NULL | For outbound: `attribute`, `product_field`, `constant`. For inbound: `source_field`, `constant`. |
| `source_key` | `VARCHAR(255)` | NOT NULL | Attribute code, product field name (e.g. `identifier`, `status`), or external field key. |
| `target_key` | `VARCHAR(255)` | NOT NULL | Target system field name (outbound) or PIM attribute code / field (inbound). |
| `transform` | `JSONB` | NULL, DEFAULT 'null' | Optional transform descriptor: e.g. `{ "type": "unit", "from": "cm", "to": "in" }`, `{ "type": "concat", "separator": " - ", "fields": ["name", "sku"] }`, `{ "type": "default", "value": "N/A" }`. Interpretation is connector-specific. |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display and processing order |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Index:** `(integration_instance_id)`. Unique constraint optional: e.g. `(integration_instance_id, target_key)` if one target field per instance.

### Table: `integration_product_filters`

Outbound only: which products are included when pushing for this instance. All non-null conditions are ANDed.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `integration_instance_id` | `INTEGER` | NOT NULL, FK → integration_instances.id, ON DELETE CASCADE, UNIQUE | One filter config per instance (one row). If no row, no filter = all products (or define default). |
| `family_ids` | `INTEGER[]` | NULL | If set, only products in these families. NULL = no family filter. |
| `category_ids` | `INTEGER[]` | NULL | If set, only products in any of these categories (or all of them—specify in docs). NULL = no category filter. |
| `channel_id` | `INTEGER` | NULL, FK → channels.id | If set, only products available in this channel. NULL = no channel filter. |
| `statuses` | `VARCHAR(20)[]` | NULL | If set, only products with status in list (e.g. `['active']`). NULL = no status filter. |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Note:** If multiple filter dimensions are set, they are ANDed: e.g. family_ids = [1,2] AND statuses = ['active']. Document behavior clearly.

### Inbound scope (Akeneo / import)

For inbound, "which data to import" can be stored as:

- **Option A:** In the instance `config` JSON, e.g. `{ "importScope": { "families": ["family1"], "fullImport": false } }`. Phase 25 reads this.
- **Option B:** A separate table `integration_import_scopes` with instance_id and scope rules. Prefer Option A for MVP to avoid more tables; add a table later if scope becomes complex.

Define in this phase: "Inbound instances may store scope/filter in `integration_instances.config` under a reserved key (e.g. `importScope`). Structure is type-specific; Phase 25 defines the shape for Akeneo."

---

## 3. Storage Methods

Add to `IStorage` and `DatabaseStorage`:

| Method | Signature | Description |
|--------|-----------|-------------|
| `getFieldMappingsByInstanceId` | `(instanceId: number) → IntegrationFieldMapping[]` | All mappings for an instance, ordered by sort_order. |
| `setFieldMappings` | `(instanceId: number, mappings: InsertIntegrationFieldMapping[]) → void` | Replace all mappings for instance: delete existing, insert new. Use transaction. |
| `getProductFilterByInstanceId` | `(instanceId: number) → IntegrationProductFilter \| undefined` | Single filter row for instance (outbound). |
| `upsertProductFilter` | `(instanceId: number, data: Partial<IntegrationProductFilter>) → IntegrationProductFilter` | Insert or update the one filter row for this instance. Validate instance is outbound. |
| `deleteProductFilter` | `(instanceId: number) → void` | Remove filter (subsequent push = no filter or all products, per product). |

---

## 4. Validation

- **Mappings:** `source_type` and `target_key` must be non-empty. For outbound `source_type = 'attribute'`, `source_key` should match an existing attribute code (warning only or strict per product). For inbound, validation can be connector-specific (Phase 25).
- **Product filter:** If `integration_instance_id` is set, the instance must be outbound (integration_types.direction = 'outbound'). Reject upsert for inbound instances.
- **Scope:** When reading `config.importScope` for an inbound instance, validate structure in the connector (Phase 25); this phase does not enforce a global schema.

---

## 5. REST API Endpoints

### Field mappings

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-instances/:id/field-mappings` | List all field mappings for this instance. Response: `{ data: IntegrationFieldMapping[] }`. |
| PUT | `/api/v1/integration-instances/:id/field-mappings` | Replace all mappings. Body: `{ mappings: { sourceType, sourceKey, targetKey, transform?, sortOrder? }[] }`. Validate instance exists; optionally validate source/target keys. Return 200 + full list. |

### Product filters (outbound only)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-instances/:id/product-filter` | Get product filter for this instance. 404 if instance is inbound or has no filter row. Response: `{ data: IntegrationProductFilter \| null }`. |
| PUT | `/api/v1/integration-instances/:id/product-filter` | Create or update product filter. Body: `{ familyIds?, categoryIds?, channelId?, statuses? }`. Validate instance is outbound; validate family/category/channel ids exist if provided. Return 200 + filter. |
| DELETE | `/api/v1/integration-instances/:id/product-filter` | Remove product filter for this instance. Return 204. |

### Import scope (inbound)

- Stored in `PATCH /api/v1/integration-instances/:id` via `config`. No separate endpoint for MVP. Phase 25 may add a dedicated scope endpoint or use config.importScope in the same PATCH.

---

## 6. Transform Types (Reference)

Document the following transform types so UI and connectors can use them consistently. Connectors implement the actual logic.

| Type | Description | Example `transform` JSON |
|------|-------------|---------------------------|
| `none` / omit | Pass value through | `null` or `{}` |
| `default` | If source is null/empty, use value | `{ "type": "default", "value": "N/A" }` |
| `concat` | Concatenate multiple source values | `{ "type": "concat", "separator": " - ", "sourceKeys": ["name", "sku"] }` (interpretation per connector) |
| `unit` | Unit conversion | `{ "type": "unit", "from": "cm", "to": "in" }` |
| `format` | Date/number format | `{ "type": "format", "format": "YYYY-MM-DD" }` |
| `custom` | Connector-specific | `{ "type": "custom", "name": "trimAndUppercase" }` |

Implement only what Phase 23/25 need; extend in later phases.

---

## 7. Permissions

- Same as Phase 21: require `integration:manage` (or equivalent) for PUT/DELETE on mappings and product filter. Read requires `integration:read` or `integration:manage`.

---

## 8. Audit History

- **Resource types:** `integrationFieldMapping`, `integrationProductFilter`.
- **Actions:** For field mappings, audit at instance level: "field mappings updated" with count or summary (avoid logging full list if large). For product filter: create, update, delete with resource_id = instance_id or filter id.

---

## 9. Exit Criteria

- [x] Tables `integration_field_mappings` and `integration_product_filters` exist; Drizzle and Zod types in place.
- [x] Storage methods implemented; setFieldMappings replaces all mappings in a transaction.
- [x] REST API: GET/PUT field-mappings for an instance; GET/PUT/DELETE product-filter for outbound instances. Validation rejects product-filter for inbound instances.
- [x] Inbound scope documented: stored in `config.importScope`; structure defined by connector (Phase 25).
- [x] Transform types documented; at least `none` and `default` usable in mapping JSON.
- [x] Changes to mappings and product filter are audit-logged as specified.

---

## 10. Notes

- **Mapping UI:** Phase 24 will need to list attributes (Phase 02) and product fields for outbound, and allow building the mapping list. For inbound, Phase 25 will define which source fields exist (e.g. from Akeneo API); the UI can still show target_key as PIM attribute code.
- **Performance:** For large mapping sets, consider pagination or "replace all" only; avoid per-row API calls in UI.
