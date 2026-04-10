# Task 4: Families

**Execution Plan Item:** #4
**Dependencies:** Attributes (Task 2), Locales/Channels (Task 3)

---

## Objective

Build the family system. A **family** defines the "template" for a product: which attributes it has and which attributes are required per channel and locale. Every product belongs to exactly one family.

> **Note:** Variant level assignment was originally part of this task (on `family_attributes`). It has been **superseded by Task 4.5 (Family Variant Definitions)**, which introduces a more flexible system where variant levels are defined per variant definition rather than globally on the family. See `04.5-Family-Variant-Definitions.md`.

---

## 1. Core Concepts

- A **family** is a named collection of attributes (e.g., the "knife" family has `bladeMaterial`, `bladeLength`, `handleColor`, etc.)
- Each attribute in a family can be marked **required** per channel per locale.
- A family has one **label attribute** (the attribute used as the display name for products in this family) and optionally one **image attribute** (the attribute used as the thumbnail).
- ~~Each attribute in a family is assigned to a **variant level**.~~ **Superseded by Task 4.5:** Variant levels are now defined per variant definition, not directly on the family-attribute relationship. See `04.5-Family-Variant-Definitions.md`.

---

## 2. Database Schema

### Table: `families`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(100)` | PK | camelCase family identifier |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Knife", "fr_FR": "Couteau" }` |
| `label_attribute_code` | `VARCHAR(255)` | FK → attributes.code, NULL | Which attribute serves as the product's display name |
| `image_attribute_code` | `VARCHAR(255)` | FK → attributes.code, NULL | Which attribute serves as the product's thumbnail |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `family_attributes`

Junction table: which attributes belong to which family.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `family_code` | `VARCHAR(100)` | FK → families.code, ON DELETE CASCADE | |
| `attribute_code` | `VARCHAR(255)` | FK → attributes.code, ON DELETE CASCADE | |
| ~~`variant_level`~~ | ~~`INTEGER`~~ | ~~NOT NULL, DEFAULT 0~~ | **Removed in Task 4.5** — variant levels now live on variant definitions. See `04.5-Family-Variant-Definitions.md`. |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order within the family |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| PRIMARY KEY | | `(family_code, attribute_code)` | An attribute can only appear once per family |

**Index:** `CREATE INDEX idx_fa_family ON family_attributes(family_code);`

### Table: `family_attribute_requirements`

Defines which attributes are required, per channel, per locale. This is the foundation for completeness calculation.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `family_code` | `VARCHAR(100)` | FK → families.code, ON DELETE CASCADE | |
| `attribute_code` | `VARCHAR(255)` | FK → attributes.code, ON DELETE CASCADE | |
| `channel_code` | `VARCHAR(100)` | FK → channels.code, ON DELETE CASCADE | |
| `locale_code` | `VARCHAR(20)` | NULL | FK → locales.code. NULL = required regardless of locale (for non-localizable attributes) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| PRIMARY KEY | | `(family_code, attribute_code, channel_code, COALESCE(locale_code, '__all__'))` | |

**How requirements work with scoping:**

| Attribute Scoping | Requirement Meaning |
|-------------------|-------------------|
| Global attribute | Required in this channel (locale_code = NULL) |
| Localizable only | Required in this locale across all channels... but typically set per channel+locale |
| Channelizable only | Required in this channel (locale_code = NULL) |
| Localizable + Channelizable | Required in this specific channel + locale combination |

For simplicity, requirements are always defined per channel. If the attribute is localizable, the requirement specifies which locales within that channel require a value.

---

## 3. API Endpoints

### Families

**`GET /api/v1/families`** — List all families. Returns code, labels, attribute count.
**`GET /api/v1/families/:code`** — Get a single family with its full attribute list and requirements.

**`POST /api/v1/families`** — Create a family
```json
{
  "code": "knife",
  "labels": { "en_US": "Knife", "fr_FR": "Couteau" }
}
```
Note: `labelAttributeCode` and `imageAttributeCode` are **not accepted on create** — a new family has no attributes assigned yet, so there is nothing to validate against. Set them via PATCH after adding attributes to the family.

**`PATCH /api/v1/families/:code`** — Update family metadata (labels, label/image attribute). Code is immutable.
**`DELETE /api/v1/families/:code`** — Delete a family.

### Family Attributes

**`GET /api/v1/families/:code/attributes`** — List all attributes assigned to this family with their sort orders.

**`POST /api/v1/families/:code/attributes`** — Add attribute(s) to the family
```json
{
  "attributes": [
    { "attributeCode": "productName", "sortOrder": 1 },
    { "attributeCode": "handleColor", "sortOrder": 2 },
    { "attributeCode": "sku", "sortOrder": 3 }
  ]
}
```
Note: `variantLevel` is no longer accepted here. Variant levels are managed per variant definition. See `04.5-Family-Variant-Definitions.md`.

**`PATCH /api/v1/families/:code/attributes/:attributeCode`** — Update sort order for an attribute in this family.

**`DELETE /api/v1/families/:code/attributes/:attributeCode`** — Remove an attribute from the family. Consider: should this delete existing values for products in this family that have this attribute? Probably not — orphaned values should be preserved but hidden from the UI.

### Family Attribute Requirements

**`GET /api/v1/families/:code/requirements`** — Get all requirements. Optionally filter by `?channelCode=ecommerce`.

**`PUT /api/v1/families/:code/requirements`** — Set requirements in bulk. This replaces all requirements for the specified channel:
```json
{
  "channelCode": "ecommerce",
  "requirements": [
    { "attributeCode": "productName", "localeCodes": ["en_US", "fr_FR"] },
    { "attributeCode": "weight", "localeCodes": null },
    { "attributeCode": "shortDescription", "localeCodes": ["en_US"] }
  ]
}
```
Where `localeCodes: null` means "required regardless of locale" (for non-localizable attributes), and `localeCodes: ["en_US"]` means "required only in en_US for this channel."

---

## 4. Business Rules

1. **Family codes are camelCase and immutable.** Standard code regex.
2. **An attribute can only be in a family once.** Enforced by the composite PK on `family_attributes`.
3. ~~**Variant levels are zero-indexed integers.**~~ **Superseded by Task 4.5.** Variant levels are now per variant definition.
4. ~~**An attribute assigned to variant level N means it can only have values at that level.**~~ **Superseded by Task 4.5.**
5. **The label_attribute_code must reference an attribute in this family** (validate on write).
6. **The image_attribute_code must reference a mediaFile or assetSingle type attribute in this family** (validate on write).
7. **Requirements can only reference attributes that are in the family.** Validate on write.
8. **Requirements for a channel can only use locales that the channel has.** Validate that each locale in the requirement exists in the channel's `locale_codes`.

---

## 5. Example: The "Knife" Family

```
Family: knife
Labels: { "en_US": "Knife" }
Label attribute: productName
Image attribute: mainImage

Attributes (pool):
  - productName (text, localizable)
  - longDescription (textarea, localizable, channelizable)
  - mainImage (mediaFile)
  - weight (measurement)
  - bladeSteel (entityReference)
  - productLine (singleSelect)
  - handleColor (singleSelect)
  - colorImage (mediaFile)
  - bladeType (singleSelect)
  - sku (text, unique)
  - price (price, channelizable)

Requirements for "ecommerce" channel:
  - productName: required in [en_US, fr_FR]
  - longDescription: required in [en_US]
  - mainImage: required (non-localizable)
  - sku: required (non-localizable)
  - price: required (non-localizable, channelizable → scoped to ecommerce)
```

> For variant level assignments (which attributes live at which level per variant configuration), see the example in `04.5-Family-Variant-Definitions.md`.

