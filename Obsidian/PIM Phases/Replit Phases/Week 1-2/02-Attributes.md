# Task 2: Attributes, Attribute Types & Attribute Groups

**Execution Plan Item:** #2
**Dependencies:** Project setup (Task 0), auth working (Task 1)
**Depended on by:** Everything — attributes are the atomic unit of the entire system

---

## Objective

Build the complete attribute definition system. Attributes define **what data** can be stored on entities in the system. The attribute layer is foundational — families, scoping, and value storage all depend on it being solid.

This task covers:
- The `attributes` table with all 15 type definitions
- The `attribute_groups` table for organizational grouping
- The `attribute_options` table for select-type attributes
- Validation rule schema stored as metadata on each attribute
- camelCase enforcement on attribute codes
- REST API endpoints and UI for managing all of the above

---

## 1. The 15 Attribute Types

Every attribute has a `type` that determines how its values are stored and validated. The type is set at creation and **cannot be changed** afterward (changing type would require migrating all existing values).

| # | Type Code | Description | Storage Column Used | Value Example |
|---|-----------|-------------|-------------------|---------------|
| 1 | `text` | Single-line text | `value_text` | `"Wingman"` |
| 2 | `textarea` | Multi-line rich text | `value_text` | `"A versatile multi-tool..."` |
| 3 | `boolean` | True/false | `value_boolean` | `true` |
| 4 | `number` | Integer or decimal | `value_number` | `7.25` |
| 5 | `measurement` | Numeric value + unit | `value_json` | `{ "amount": 5.2, "unit": "cm" }` |
| 6 | `price` | Multi-currency price | `value_json` | `[{ "currency": "USD", "amount": 29.99 }, { "currency": "EUR", "amount": 27.50 }]` |
| 7 | `date` | Date/datetime | `value_date` | `"2026-03-15T00:00:00Z"` |
| 8 | `singleSelect` | One option from a list | `value_text` | `"red"` (stores the option code) |
| 9 | `multiSelect` | Multiple options from a list | `value_json` | `["red", "blue", "green"]` (array of option codes) |
| 10 | `mediaFile` | Uploaded file (image, PDF, etc.) | `value_media` | `"/uploads/wingman-hero.jpg"` |
| 11 | `assetSingle` | Reference to one asset | `value_text` | `"wingmanHeroFront"` (asset code) |
| 12 | `assetList` | Reference to multiple assets | `value_json` | `["wingmanHeroFront", "wingmanHeroBack"]` |
| 13 | `entitySingle` | Reference to one entity | `value_text` | `"leatherman"` (entity code) |
| 14 | `entityList` | Reference to multiple entities | `value_json` | `["leatherman", "gerber"]` |
| 15 | `productReference` | Reference to another product | `value_text` | `"WAVE-PLUS"` (product identifier) |
| 16 | `table` | Tabular data (rows × columns) | `value_json` | `[{ "size": "S", "chest": 36 }, { "size": "M", "chest": 40 }]` |

**Key design note:** Types 11-15 are reference types. The attribute type and storage pattern should be defined now. The value is stored as a plain string (code/identifier) — no FK constraint on the value itself. `assetSingle`/`assetList` and `entitySingle`/`entityList` follow the same single-vs-list pattern.

---

## 2. Database Schema

### Table: `attribute_groups`

Organizational grouping for attributes (e.g., "Marketing," "Technical," "Dimensions"). Used in the UI to group attribute fields into tabs or sections.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(100)` | PK | camelCase identifier |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Marketing", "fr_FR": "Marketing" }` |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order in UI |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `attributes`

The core attribute definition table. One row per attribute in the system.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(255)` | PK | camelCase identifier (e.g., `productName`, `weight`) |
| `type` | `VARCHAR(50)` | NOT NULL | One of the 15 type codes listed above |
| `group_code` | `VARCHAR(100)` | FK → attribute_groups.code, NULL | Which group this attribute belongs to |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Product Name" }` |
| `is_localizable` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether values vary per locale |
| `is_channelizable` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether values vary per channel |
| `is_unique` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether values must be unique across all entities of the same type |
| `is_read_only` | `BOOLEAN` | NOT NULL, DEFAULT false | Cannot be edited via UI (system-managed) |
| `validation_rules` | `JSONB` | NOT NULL, DEFAULT '{}' | Validation constraints (see section 4) |
| `settings` | `JSONB` | NOT NULL, DEFAULT '{}' | Type-specific settings (see section 3) |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Default sort within a group |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Index:** `CREATE INDEX idx_attributes_group ON attributes(group_code);`
**Index:** `CREATE INDEX idx_attributes_type ON attributes(type);`

### Table: `attribute_options`

For `singleSelect` and `multiSelect` attribute types. Stores the available options.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `attribute_code` | `VARCHAR(255)` | FK → attributes.code, ON DELETE CASCADE | Which attribute this option belongs to |
| `code` | `VARCHAR(255)` | NOT NULL | camelCase option identifier |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Red", "fr_FR": "Rouge" }` |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| PRIMARY KEY | | `(attribute_code, code)` | Composite PK |

---

## 3. Type-Specific Settings (`settings` JSONB column)

The `settings` column on `attributes` stores configuration specific to each attribute type:

| Attribute Type | Settings Schema |
|---------------|----------------|
| `text` | `{ "maxLength": 255 }` |
| `textarea` | `{ "maxLength": 65535, "richText": false }` |
| `number` | `{ "decimalsAllowed": true, "decimalPlaces": 2 }` |
| `measurement` | `{ "defaultUnit": "cm", "allowedUnits": ["cm", "mm", "in", "m"] }` |
| `price` | `{ "currencies": ["USD", "EUR", "GBP"] }` — which currencies are relevant |
| `date` | `{ "includeTime": false }` |
| `mediaFile` | `{ "allowedExtensions": ["jpg", "png", "pdf"], "maxFileSizeMb": 10 }` |
| `table` | `{ "columns": [{ "code": "size", "type": "text" }, { "code": "chest", "type": "number" }] }` |
| `assetSingle` | `{ "assetFamilyCode": "images" }` — restrict to a specific asset family (optional) |
| `assetList` | `{ "assetFamilyCode": "images", "maxItems": 10 }` — `maxItems` is optional; omit or leave empty for no limit. *(Bug fix: the form coerced empty string to 0 which failed `.positive()` validation; fixed by treating empty/0 as omitted.)* |
| `entitySingle` | `{ "entityTypeCode": "brand" }` — restrict to a specific entity type (optional) |
| `entityList` | `{ "entityTypeCode": "brand", "maxItems": 10 }` — `maxItems` optional; omit or leave empty for no limit. *(Same maxItems fix as assetList.)* |
| `productReference` | `{ "familyCode": "knife" }` — restrict to a specific family (optional) |
| `singleSelect`, `multiSelect` | `{}` — options are stored in `attribute_options` table |
| `boolean` | `{ "defaultValue": false }` |

---

## 4. Validation Rules (`validation_rules` JSONB column)

Stored as metadata on the attribute definition. The schema must be in place now so rules can be defined when attributes are created. Enforcement of these rules on value writes is a separate concern.

```typescript
// Zod schema for validation_rules
const validationRulesSchema = z.object({
  regex: z.string().optional(),          // regex pattern the value must match
  regexMessage: z.string().optional(),   // custom error message for regex failure
  minLength: z.number().optional(),      // for text/textarea
  maxLength: z.number().optional(),      // for text/textarea
  minValue: z.number().optional(),       // for number/measurement
  maxValue: z.number().optional(),       // for number/measurement
  allowedExtensions: z.array(z.string()).optional(),  // for mediaFile
  maxFileSizeMb: z.number().optional(),  // for mediaFile
}).strict();
```

Example: An attribute `sku` might have:
```json
{
  "regex": "^[A-Z0-9-]+$",
  "regexMessage": "SKU must be uppercase alphanumeric with hyphens only",
  "minLength": 3,
  "maxLength": 50
}
```

---

## 5. API Endpoints

### Attribute Groups

**`GET /api/v1/attribute-groups`** — List all groups, ordered by `sort_order`
**`GET /api/v1/attribute-groups/:code`** — Get a single group
**`POST /api/v1/attribute-groups`** — Create a group
```json
{
  "code": "marketing",
  "labels": { "en_US": "Marketing" },
  "sortOrder": 0
}
```
**`PATCH /api/v1/attribute-groups/:code`** — Update a group (code is immutable)
**`DELETE /api/v1/attribute-groups/:code`** — Delete a group. Fails if any attributes reference it, unless `?force=true` (which nulls out the group_code on those attributes).

### Attributes

**`GET /api/v1/attributes`** — List all attributes. Supports filters:
  - `?type=text,number` — filter by type
  - `?groupCode=marketing` — filter by group
  - `?isLocalizable=true` — filter by scoping flags
  - `?search=product` — search labels and codes

**`GET /api/v1/attributes/:code`** — Get a single attribute with its options (if select type)

**`POST /api/v1/attributes`** — Create an attribute
```json
{
  "code": "productName",
  "type": "text",
  "groupCode": "marketing",
  "labels": { "en_US": "Product Name", "fr_FR": "Nom du produit" },
  "isLocalizable": true,
  "isChannelizable": false,
  "isUnique": false,
  "validationRules": { "maxLength": 255 },
  "settings": { "maxLength": 255 }
}
```

**`PATCH /api/v1/attributes/:code`** — Update an attribute. The `code` and `type` fields are **immutable** — cannot be changed after creation. `isLocalizable` and `isChannelizable` can only be changed if no values exist for this attribute yet (changing scoping would invalidate existing values).

**`DELETE /api/v1/attributes/:code`** — Delete an attribute. Fails if it's used in any family or has stored values. Include `?force=true` to cascade-delete all associated values and family references.

### Attribute Options (for select types)

**`GET /api/v1/attributes/:code/options`** — List options for a select attribute
**`POST /api/v1/attributes/:code/options`** — Add an option
```json
{
  "code": "red",
  "labels": { "en_US": "Red", "fr_FR": "Rouge" },
  "sortOrder": 1
}
```
**`PATCH /api/v1/attributes/:code/options/:optionCode`** — Update an option (code is immutable)
**`DELETE /api/v1/attributes/:code/options/:optionCode`** — Delete an option. Consider whether to cascade or fail if values reference it.

---

## 6. Business Rules

1. **Attribute codes are camelCase, immutable, and unique.** Enforce `^[a-z][a-zA-Z0-9]*$` on creation. Return a clear error if the code already exists.
2. **Attribute types are immutable.** Once created, the type cannot change.
3. **Scoping flags (`isLocalizable`, `isChannelizable`) affect value storage.** These determine whether the `attribute_values` table (Task 3) stores values with locale/channel dimensions. Changing them after values exist would invalidate data — block it or require migration.
4. **Option codes are camelCase and unique within an attribute.** Enforce the same regex.
5. **Deleting an attribute group** should not cascade-delete its attributes. Instead, null out the `group_code` on affected attributes (or block deletion if attributes reference it).
6. **Sort order** is user-defined and drives UI display order. Default to 0; duplicates are fine (secondary sort by code).
7. **Labels are optional but encouraged.** If no label is provided for a locale, the UI should fall back to the code.

