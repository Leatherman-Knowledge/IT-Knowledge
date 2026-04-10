# Task 3: Locales, Channels & Scoping Infrastructure

**Execution Plan Item:** #3
**Dependencies:** Attributes (Task 2) — scoping is meaningless without attribute definitions
**Depended on by:** Task 4 (Families)

---

## Objective

Build the locale and channel system and — critically — define **how attribute values are stored** with locale and channel dimensions. This is the most architecturally important task in the entire project. Every attribute value in the system flows through the storage pattern defined here.

The scoping system creates four possible storage modes for any attribute:
1. **Global** — one value, no locale or channel dimension (`is_localizable: false`, `is_channelizable: false`)
2. **Localizable only** — one value per locale (`is_localizable: true`, `is_channelizable: false`)
3. **Channelizable only** — one value per channel (`is_localizable: false`, `is_channelizable: true`)
4. **Localizable + Channelizable** — one value per locale per channel (`is_localizable: true`, `is_channelizable: true`)

---

## 1. Database Schema

### Table: `locales`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(20)` | PK | Locale code: either language-only (`en`, `de`, `fr`) or BCP 47 with region (`en-US`, `de-DE`, `en_US`). Validation accepts both patterns (Phase 12.1.1). |
| `label` | `VARCHAR(100)` | NOT NULL | "English (United States)" |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | Inactive locales are hidden from UI but data is preserved |
| `fallback_locale_code` | `VARCHAR(20)` | FK → locales.code, NULL | The locale to fall back to if a value is missing |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Fallback chain example:** `fr_CA` → `fr_FR` → `en_US` → NULL (no further fallback)

The `fallback_locale_code` creates a singly-linked list. Resolution: try the requested locale, if no value exists follow the chain until a value is found or the chain ends.

### Table: `currencies`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(10)` | PK | ISO 4217: `USD`, `EUR`, `GBP` |
| `label` | `VARCHAR(100)` | NOT NULL | "US Dollar" |
| `symbol` | `VARCHAR(10)` | NOT NULL | "$", "€", "£" |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `channels`

A channel represents a destination for product data (e.g., an e-commerce site, a print catalog, a retail system). Each channel defines which locales, currencies, and category tree are relevant to it.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(100)` | PK | camelCase identifier |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "E-Commerce" }` |
| `locale_codes` | `JSONB` | NOT NULL, DEFAULT '[]' | Array of active locale codes for this channel: `["en_US", "fr_FR"]` |
| `currency_codes` | `JSONB` | NOT NULL, DEFAULT '[]' | Array of active currency codes: `["USD", "EUR"]` |
| `category_tree_code` | `VARCHAR(100)` | FK → category_trees.code, NULL | Root category tree for this channel (Task 5) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

---

## 2. Attribute Value Storage — THE Critical Table

### Table: `attribute_values`

This is the EAV (Entity-Attribute-Value) table that stores **all attribute data**. It's the most queried and written-to table in the entire system.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Surrogate PK for easy reference |
| `entity_type` | `VARCHAR(50)` | NOT NULL | The type of record this value belongs to |
| `entity_id` | `VARCHAR(255)` | NOT NULL | The identifier/code of the record |
| `attribute_code` | `VARCHAR(255)` | NOT NULL, FK → attributes.code | Which attribute this value is for |
| `channel_code` | `VARCHAR(100)` | NULL | FK → channels.code. NULL means "all channels" (global or localizable-only) |
| `locale_code` | `VARCHAR(20)` | NULL | FK → locales.code. NULL means "all locales" (global or channelizable-only) |
| `value_text` | `TEXT` | NULL | For text, textarea, singleSelect, assetSingle, entityReference, productReference |
| `value_number` | `DECIMAL(20,6)` | NULL | For number |
| `value_boolean` | `BOOLEAN` | NULL | For boolean |
| `value_date` | `TIMESTAMPTZ` | NULL | For date |
| `value_json` | `JSONB` | NULL | For measurement, price, multiSelect, assetList, table |
| `value_media` | `VARCHAR(500)` | NULL | For mediaFile — file path or URL |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Critical unique constraint** — prevents duplicate values for the same attribute on the same entity in the same scope:

```sql
CREATE UNIQUE INDEX uq_attribute_values_scope
ON attribute_values (entity_type, entity_id, attribute_code, channel_code, locale_code)
NULLS NOT DISTINCT;
```

The `NULLS NOT DISTINCT` clause (PostgreSQL 15+) treats NULL values as equal for uniqueness purposes. This means you can't have two rows with the same entity + attribute + NULL channel + NULL locale.

**Performance indexes:**

```sql
CREATE INDEX idx_av_entity ON attribute_values(entity_type, entity_id);
CREATE INDEX idx_av_attribute ON attribute_values(attribute_code);
CREATE INDEX idx_av_channel_locale ON attribute_values(channel_code, locale_code);
CREATE INDEX idx_av_text_search ON attribute_values USING gin(to_tsvector('english', value_text)) WHERE value_text IS NOT NULL;
```

### How the four scoping modes map to this table

| Attribute Scoping | `channel_code` | `locale_code` | Rows per entity |
|-------------------|---------------|---------------|----------------|
| Global | NULL | NULL | 1 |
| Localizable only | NULL | `en_US`, `fr_FR`, etc. | 1 per locale |
| Channelizable only | `ecommerce`, `retail`, etc. | NULL | 1 per channel |
| Localizable + Channelizable | `ecommerce` | `en_US` | 1 per channel × locale |

### Example data for a single product

Product `WINGMAN` with attribute `productName` (localizable, channelizable):
| entity_type | entity_id | attribute_code | channel_code | locale_code | value_text |
|-------------|-----------|----------------|--------------|-------------|-----------|
| product | WINGMAN | productName | ecommerce | en_US | "Wingman Multi-Tool" |
| product | WINGMAN | productName | ecommerce | fr_FR | "Wingman Multi-Outil" |
| product | WINGMAN | productName | retail | en_US | "Wingman" |
| product | WINGMAN | weight | NULL | NULL | *(value_json)* `{ "amount": 198, "unit": "g" }` |
| product | WINGMAN | isDiscontinued | NULL | NULL | *(value_boolean)* `false` |

---

## 3. Value Resolution with Locale Fallback

When reading an attribute value for a specific locale, if no value exists for that locale, follow the fallback chain:

```typescript
async function resolveAttributeValue(
  entityType: string,
  entityId: string,
  attributeCode: string,
  channelCode: string | null,
  localeCode: string
): Promise<AttributeValue | null> {
  let currentLocale: string | null = localeCode;

  while (currentLocale) {
    const value = await findAttributeValue(entityType, entityId, attributeCode, channelCode, currentLocale);
    if (value) return value;

    // Follow fallback chain
    const locale = await getLocale(currentLocale);
    currentLocale = locale?.fallbackLocaleCode ?? null;
  }

  return null; // No value found in any fallback locale
}
```

This should be implemented as a utility function in `src/lib/` since it's used whenever attribute values are read in a locale context.

---

## 4. API Endpoints

### Locales

**`GET /api/v1/locales`** — List all locales. Filter: `?isActive=true`
**`GET /api/v1/locales/:code`** — Get a single locale
**`POST /api/v1/locales`** — Create a locale
```json
{
  "code": "en_US",
  "label": "English (United States)",
  "isActive": true,
  "fallbackLocaleCode": null
}
```
**`PATCH /api/v1/locales/:code`** — Update (label, isActive, fallbackLocaleCode)
**`DELETE /api/v1/locales/:code`** — Delete. Fails if the locale is referenced by any channel or has attribute values. Use `?force=true` to cascade.

### Currencies

**`GET /api/v1/currencies`** — List all currencies
**`POST /api/v1/currencies`** — Create a currency
```json
{
  "code": "USD",
  "label": "US Dollar",
  "symbol": "$",
  "isActive": true
}
```
**`PATCH /api/v1/currencies/:code`** — Update
**`DELETE /api/v1/currencies/:code`** — Delete. Fails if referenced by any channel.

### Channels

**`GET /api/v1/channels`** — List all channels
**`GET /api/v1/channels/:code`** — Get a single channel with its locale and currency details
**`POST /api/v1/channels`** — Create a channel
```json
{
  "code": "ecommerce",
  "labels": { "en_US": "E-Commerce" },
  "localeCodes": ["en_US", "fr_FR", "de_DE"],
  "currencyCodes": ["USD", "EUR"],
  "categoryTreeCode": "masterCatalog"
}
```
**`PATCH /api/v1/channels/:code`** — Update. Removing a locale from a channel does NOT delete attribute values scoped to that locale+channel — the values are preserved but hidden from the channel's view.
**`DELETE /api/v1/channels/:code`** — Delete. Fails if attribute values are scoped to this channel.

### Attribute Values (Generic CRUD)

The underlying value storage logic should be built now as a service layer:

```typescript
// Service layer functions (not exposed as API endpoints yet — products don't exist)
async function setAttributeValue(params: {
  entityType: string;
  entityId: string;
  attributeCode: string;
  channelCode: string | null;
  localeCode: string | null;
  value: any; // dispatched to correct column based on attribute type
}): Promise<AttributeValue>

async function getAttributeValues(params: {
  entityType: string;
  entityId: string;
  channelCode?: string | null;
  localeCode?: string | null;
}): Promise<AttributeValue[]>

async function deleteAttributeValue(params: {
  entityType: string;
  entityId: string;
  attributeCode: string;
  channelCode: string | null;
  localeCode: string | null;
}): Promise<void>
```

The `setAttributeValue` function must:
1. Look up the attribute definition to determine the type
2. Validate that the value matches the expected type (e.g., `value_text` for text attributes)
3. Validate scoping: if the attribute is `is_localizable: false`, `locale_code` must be NULL; if `is_channelizable: false`, `channel_code` must be NULL
4. Upsert the value (insert or update on the unique constraint)

---

## 5. Deferred Items From Phase 2

The following items were deferred during Phase 2 because they depend on the `attribute_values` table introduced in this phase. They **must** be implemented as part of Phase 3 delivery.

### 5a. Scoping Flag Mutability Guard

The `updateAttribute` handler (PATCH `/api/v1/attributes/:code`) must block changes to `isLocalizable` and `isChannelizable` if any rows exist in `attribute_values` for that attribute code. Changing scoping after values exist would invalidate the dimensional structure of stored data. Return 409 with a descriptive error if values exist and a scoping flag change is attempted.

### 5b. Attribute DELETE Safety Checks

The `deleteAttribute` handler (DELETE `/api/v1/attributes/:code`) must check for stored values in `attribute_values` before allowing deletion. Without `?force=true`, return 409 if any values exist. With `?force=true`, cascade-delete all associated values before deleting the attribute. (Family reference checks will be added in Phase 4 when the `family_attributes` table is built.)

### 5c. Option DELETE Value-Reference Handling

When deleting an attribute option (DELETE `/api/v1/attributes/:code/options/:optionCode`), check if any `attribute_values` rows reference the option code. Without `?force=true`, return 409 if values reference it. With `?force=true`, clear the references: set `value_text` to null for `singleSelect` values, remove the option code from the `value_json` array for `multiSelect` values. If removing from a `multiSelect` array leaves it empty, decide whether to preserve `[]` or set to null — document the decision.

### 5d. Default Settings Handling

Phase 2 made all type-specific settings fields optional (e.g., a `text` attribute can be created without specifying `maxLength`). The `setAttributeValue` function and any downstream validation must handle absent settings gracefully by falling back to sensible defaults (e.g., no length limit if `maxLength` is not set, decimals disallowed if `decimalsAllowed` is not set).

---

## 6. Business Rules

1. **Locale codes** may be language-only (`en`, `de`, `fr`) or include region (`en-US`, `en_US`, `fr_FR`, `de_DE`). Validation accepts `^[a-z]{2}$` or `^[a-z]{2}-[A-Z]{2}$` (Phase 12.1.1). Not camelCase; they are a special case for PK format.
2. **Currency codes use ISO 4217** (`USD`, `EUR`) — not camelCase.
3. **Channel codes are camelCase** and follow the standard code regex.
4. **Fallback chains must not create cycles.** Validate on write that setting `fallbackLocaleCode` doesn't create a circular reference (e.g., `fr_CA` → `fr_FR` → `fr_CA`).
5. **A channel's locale_codes must all reference active, existing locales.** Validate on write.
6. **Removing a locale from a channel** does not delete scoped attribute values — it just hides them from that channel's view. Values can be restored by re-adding the locale.
7. **The attribute_values table uses a soft reference** via `entity_type` + `entity_id`. Referential integrity for the entity itself is enforced at the application layer.

