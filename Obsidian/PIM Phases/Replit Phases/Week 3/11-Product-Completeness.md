# Task 11: Product Completeness Calculation

**Execution Plan Item:** #11
**Dependencies:** Product Attribute Values (Task 9), Product-Category Assignment & Status (Task 10), Family Requirements (Task 4)
**Depended on by:** Product UI (Task 12 — completeness indicators)

---

## Objective

Build the completeness calculation engine. Completeness measures how "filled out" a product is relative to its family's requirements for a given channel and locale. It is the primary data quality signal in the PIM — the percentage tells users at a glance whether a product is ready for a specific channel.

Completeness is **calculated on demand**, not stored. The engine evaluates the product's current attribute values against the family's requirements and returns a result. This avoids stale data and keeps the logic centralized.

---

## 1. Core Concepts

### What Completeness Measures

Completeness answers: "For this product, in this channel, in this locale — what percentage of required attributes have values?"

The formula:

```
completeness = (required attributes with values / total required attributes) × 100
```

A product is **complete** for a channel+locale when completeness = 100%.

### Where Requirements Come From

Requirements are defined in the `family_attribute_requirements` table (Task 4). Each requirement says: "For products in this family, this attribute is required in this channel (and optionally, in this locale)."

### How Scoping Affects Evaluation

The four attribute scoping modes create different evaluation scenarios:

| Attribute Scoping | How It's Evaluated for Channel X, Locale Y |
|-------------------|---------------------------------------------|
| **Global** (not localizable, not channelizable) | Check if a value exists with `channel_code = NULL, locale_code = NULL` |
| **Localizable only** | Check if a value exists with `channel_code = NULL, locale_code = Y` |
| **Channelizable only** | Check if a value exists with `channel_code = X, locale_code = NULL` |
| **Localizable + Channelizable** | Check if a value exists with `channel_code = X, locale_code = Y` |

### How Variant Inheritance Affects Evaluation

For variant products, completeness considers the **merged set** of values — including inherited values from ancestors. A variant does not need to have its own value for a level-0 attribute if the root model has set it.

The evaluation pipeline:
1. Determine all required attributes for the product's family in the given channel+locale
2. For each required attribute, check if a value exists — either owned by this product or inherited from an ancestor (using the same inheritance resolution from Task 9)
3. A value is "present" if the relevant storage column is not NULL (non-empty strings, non-null booleans, non-null numbers all count as present)

### Empty vs. Present

A value is considered **present** (counts toward completeness) if:
- `value_text` is not NULL and not empty string (`''`)
- `value_number` is not NULL
- `value_boolean` is not NULL (both `true` and `false` count as present)
- `value_date` is not NULL
- `value_json` is not NULL and not empty array (`[]`) and not empty object (`{}`)
- `value_media` is not NULL and not empty string

A value is considered **missing** (counts against completeness) if no row exists in `attribute_values` for that attribute+scope combination, or if the row exists but the relevant column is NULL/empty.

---

## 2. Completeness Calculation Function

### Core Engine

```typescript
interface CompletenessResult {
  channelCode: string;
  localeCode: string;
  ratio: {
    complete: number;    // count of required attributes with values
    required: number;    // total count of required attributes
  };
  percentage: number;    // 0–100, rounded to nearest integer
  missingAttributes: Array<{
    attributeCode: string;
    labels: Record<string, string>;
    type: string;
    groupCode: string | null;
  }>;
}

async function calculateCompleteness(
  productIdentifier: string,
  channelCode: string,
  localeCode: string
): Promise<CompletenessResult>
```

### Calculation Steps

**Step 1: Load the product and resolve its family**
```
product = getProduct(identifier)
familyCode = product.familyCode
```

**Step 2: Get the family's requirements for this channel**
```
requirements = getFamilyRequirements(familyCode, channelCode)
```
This returns a list of `(attributeCode, localeCode | null)` pairs. Filter to requirements that match the requested `localeCode`:
- Requirements with `locale_code = NULL` (for non-localizable attributes) always apply
- Requirements with `locale_code = localeCode` apply for the requested locale
- Requirements with a different `locale_code` are excluded

**Step 3: Load the product's merged attribute values (with inheritance)**
```
values = getProductValues(identifier, {
  channelCode,
  localeCode,
  includeInherited: true
})
```

**Step 4: Evaluate each required attribute**

For each required attribute:
1. Look up the attribute definition to determine its scoping (localizable, channelizable)
2. Determine the expected scope key: `(channelCode | null, localeCode | null)` based on the attribute's scoping flags
3. Find the matching value in the merged value set
4. Determine if the value is "present" (see section 1)

**Step 5: Calculate and return**
```
complete = count of required attributes with present values
required = total count of required attributes
percentage = Math.round((complete / required) * 100)
missingAttributes = required attributes without present values
```

If a family has zero requirements for a given channel+locale, completeness is 100% (fully complete by definition — there's nothing required).

---

## 3. Batch Completeness

### Per Product, All Channels and Locales

**Function:**
```typescript
async function calculateAllCompleteness(
  productIdentifier: string
): Promise<CompletenessResult[]>
```

This calculates completeness across all applicable channels and locales:
1. Determine which channels the product belongs to (via `getProductChannels` from Task 10)
2. For each channel, get the channel's active locales
3. Calculate completeness for each channel+locale combination
4. Return the full array of results

### Per Product List (Summary)

For the product list page, completeness needs to be calculated efficiently for many products. Provide a batch function:

```typescript
interface CompletenessOverview {
  productIdentifier: string;
  channels: Array<{
    channelCode: string;
    overallPercentage: number;   // average across all locales in this channel
    locales: Array<{
      localeCode: string;
      percentage: number;
    }>;
  }>;
}

async function calculateCompletenessOverview(
  productIdentifiers: string[]
): Promise<CompletenessOverview[]>
```

This function should be optimized for batch loading:
- Load all products and their families in one query
- Load all family requirements for the involved families in one query
- Load all attribute values for all products in one query
- Compute completeness in application code

This avoids the N+1 query problem when rendering a product list with completeness badges.

**Phase 12.1.1 — Recursive descendants:** For product models, the per-variant breakdown must include **all descendants** (sub-models and their variants), not only direct children. Recursively traverse the full descendant tree when building the completeness overview so that variant-level completeness is included in the rollup. **Category assignment is optional:** products with no category assignments must not cause completeness or the edit page to fail; show an empty categories tab and compute completeness only for channels the product belongs to (or treat no categories gracefully).

---

## 4. API Endpoints

### Single Product Completeness

**`GET /api/v1/products/:identifier/completeness`** — Get completeness for a single product across all applicable channels and locales.

Response:
```json
{
  "data": {
    "identifier": "PRD-00003-red-plain",
    "familyCode": "knife",
    "channels": [
      {
        "channelCode": "ecommerce",
        "labels": { "en_US": "E-Commerce" },
        "locales": [
          {
            "localeCode": "en_US",
            "percentage": 85,
            "ratio": { "complete": 6, "required": 7 },
            "missingAttributes": [
              {
                "attributeCode": "longDescription",
                "labels": { "en_US": "Long Description" },
                "type": "textarea",
                "groupCode": "marketing"
              }
            ]
          },
          {
            "localeCode": "fr_FR",
            "percentage": 57,
            "ratio": { "complete": 4, "required": 7 },
            "missingAttributes": [
              {
                "attributeCode": "productName",
                "labels": { "en_US": "Product Name" },
                "type": "text",
                "groupCode": "marketing"
              },
              {
                "attributeCode": "longDescription",
                "labels": { "en_US": "Long Description" },
                "type": "textarea",
                "groupCode": "marketing"
              },
              {
                "attributeCode": "shortDescription",
                "labels": { "en_US": "Short Description" },
                "type": "text",
                "groupCode": "marketing"
              }
            ]
          }
        ]
      },
      {
        "channelCode": "retail",
        "labels": { "en_US": "Retail" },
        "locales": [
          {
            "localeCode": "en_US",
            "percentage": 100,
            "ratio": { "complete": 4, "required": 4 },
            "missingAttributes": []
          }
        ]
      }
    ]
  }
}
```

**`GET /api/v1/products/:identifier/completeness/:channelCode/:localeCode`** — Get completeness for a specific channel+locale combination. Returns a single `CompletenessResult` instead of the full matrix.

### Batch Completeness (for Product List)

The product list endpoint (`GET /api/v1/products`) should optionally include a completeness summary when requested:

**`GET /api/v1/products?includeCompleteness=true`**

When `includeCompleteness=true`, each product in the response includes a `completeness` field with an overview (overall percentage per channel). This uses the batch completeness function to avoid per-product queries.

```json
{
  "data": [
    {
      "identifier": "PRD-00003",
      "familyCode": "knife",
      "type": "model",
      "status": "active",
      "completeness": {
        "ecommerce": { "en_US": 85, "fr_FR": 57 },
        "retail": { "en_US": 100 }
      }
    }
  ],
  "meta": { "total": 42, "limit": 25, "cursor": null, "nextCursor": "abc" }
}
```

The completeness object is a compact format (channel → locale → percentage) suitable for rendering badges in a table row without overwhelming the response payload.

### Filter Products by Completeness

Extend the `GET /api/v1/products` endpoint with completeness filters:

- `?completeness=100` — only products that are 100% complete (for any channel+locale)
- `?completeness=lt:100` — products with less than 100% completeness
- `?completenessChannel=ecommerce&completenessLocale=en_US&completeness=gte:80` — products with at least 80% completeness for ecommerce/en_US

Completeness filtering is computed at query time. For the initial implementation, this can be done in application code after loading the product list. If performance becomes an issue, a materialized completeness cache can be added later.

---

## 5. Completeness for Product Models vs. Variants

Completeness is calculated for **every product in the hierarchy independently**:

- A **root product model** has completeness based on its level-0 required attributes only
- A **sub-model** has completeness based on its level-0 (inherited) + level-1 (own) required attributes
- A **leaf variant** has completeness based on all levels (inherited + own) required attributes

This means a root model can be 100% complete for its own level while its variants are still incomplete. The product UI should show completeness at each level in the hierarchy.

For the **product list page**, show completeness for the lowest-level products (variants) since those are what downstream systems consume. If a product is a model, the list can show the minimum or average completeness across its variants as a summary.

---

## 6. Storage Layer Methods

### New `IStorage` Methods

| Method | Purpose |
|--------|---------|
| `calculateCompleteness(identifier, channelCode, localeCode)` | Core completeness calculation |
| `calculateAllCompleteness(identifier)` | Completeness across all channels and locales |
| `calculateCompletenessOverview(identifiers[])` | Batch completeness for product list |
| `getFamilyRequirementsForChannel(familyCode, channelCode)` | Get requirements filtered by channel |

---

## 7. Business Rules

1. **Completeness is always calculated, never stored.** This ensures it's always accurate. If requirements change, completeness updates instantly on the next query.

2. **Completeness uses the merged (inherited) value set.** A variant is considered to have a value for a level-0 attribute if the root model has set it, even though the variant doesn't have its own row in `attribute_values`.

3. **A family with zero requirements** for a channel+locale means completeness = 100%. This is intentional — if nothing is required, everything is complete.

4. **Completeness is channel-specific.** A product can be 100% complete for one channel and 50% for another, because different channels can have different requirements (via `family_attribute_requirements`).

5. **Boolean attributes count as present** when the value is `false`. Only NULL/missing counts as absent. A boolean "is this discontinued?" is a valid answer whether true or false.

6. **Empty strings do not count as present.** A text attribute with value `""` is treated the same as a missing value. This prevents users from bypassing completeness by entering blank values.

7. **Locale fallback does NOT apply to completeness.** Completeness checks whether a value exists for the exact requested locale. If `productName` has a value for `en_US` but not `fr_FR`, the `fr_FR` completeness for that attribute is incomplete — even though the read endpoint would fall back to `en_US`. Completeness measures data entry progress, not readability.

8. **Products not in any category tree for a channel** have no completeness for that channel (the channel is irrelevant to them). The completeness endpoint only returns channels the product belongs to.
