# Task 9: Product Attribute Values — Read, Write & Variant Inheritance

**Execution Plan Item:** #9
**Dependencies:** Products table (Task 8), Attribute Values table (Task 3), Families (Task 4), Variant Definitions (Task 4.5)
**Depended on by:** Completeness (Task 11), Product UI (Task 12)

---

## Objective

Build the service layer and API endpoints for reading and writing attribute values on products. This is the integration point between products and the EAV `attribute_values` table created in Task 3.

The critical complexity here is **variant inheritance**: attribute values set at a parent level cascade down to children, and the variant definition controls which attributes can be written at which level in the hierarchy. A variant should never store a value for an attribute that belongs to its parent's level — it inherits that value instead.

### Important: Product Attribute Values vs. Entity Label Translations

This task deals with **product attribute values** — the actual product data stored in the `attribute_values` table (e.g., a product's name in French, its price for the e-commerce channel, its weight). These values are scoped by locale and/or channel depending on the attribute's configuration, and they are managed through the product edit page's channel/locale picker.

This is entirely separate from the **Translations Management System** (Task 7.5), which manages the display labels on structural entities like attributes, families, categories, and channels. Those translations control how the PIM's own UI renders entity names in different locales — they have nothing to do with product data.

To be concrete: the French translation of the *attribute label* "Product Name" → "Nom du produit" is managed on the Translations page. The French *value* of that attribute on a specific product — "Signal Multi-Outil" — is a product attribute value managed here.

---

## 1. Core Concepts

### Attribute Level Ownership

The variant definition (Task 4.5) assigns every family attribute to a level:
- **Level 0 (common):** Values are written on the root product model (or directly on a simple product)
- **Level 1:** Values are written on level-1 children (sub-models or variants)
- **Level 2:** Values are written on level-2 children (leaf variants)

When writing a value on a product, the system must check the variant definition to confirm the attribute belongs to that product's level. Writing a level-0 attribute on a level-2 variant is rejected — that value must be written on the root model.

For **simple products**, all attributes are at level 0 (there is no hierarchy), so all values are written directly on the product.

### Variant Inheritance (Read Path)

When reading attribute values for a product that is part of a variant hierarchy, the system resolves values by walking **up** the hierarchy:

1. Check if a value exists on the product itself
2. If not, check the parent
3. If not, check the grandparent (and so on up to the root)

The result is a merged view: the product's own values overlaid on inherited values from ancestors.

```
Root Model: PRD-00003
  productName = "Signal" (level 0 attribute)
  weight = { amount: 213, unit: "g" } (level 0 attribute)

  Sub-Model: PRD-00003-red
    handleColor = "red" (level 1 axis attribute)
    colorImage = "/images/signal-red.jpg" (level 1 attribute)

    Variant: PRD-00003-red-plain
      bladeType = "plain" (level 2 axis attribute)
      sku = "832-RED-PLAIN" (level 2 attribute)
      price = [{ currency: "USD", amount: 129.99 }] (level 2 attribute)
```

Reading all values for `PRD-00003-red-plain` returns the merged set:
- productName = "Signal" (inherited from root)
- weight = { amount: 213, unit: "g" } (inherited from root)
- handleColor = "red" (inherited from sub-model)
- colorImage = "/images/signal-red.jpg" (inherited from sub-model)
- bladeType = "plain" (own value)
- sku = "832-RED-PLAIN" (own value)
- price = [{ currency: "USD", amount: 129.99 }] (own value)

Each value in the response should indicate whether it's **own** or **inherited** and from which ancestor.

### Scoping Still Applies

Locale and channel scoping (Task 3) applies on top of variant inheritance:
- A localizable attribute at level 0 has one value per locale, stored on the root model
- A channelizable attribute at level 2 has one value per channel, stored on the leaf variant
- A localizable + channelizable attribute has one value per channel per locale at its designated level

---

## 2. Writing Attribute Values

### Endpoint

**`PUT /api/v1/products/:identifier/values`** — Set attribute values on a product.

Request body — a flat array of value entries:
```json
{
  "values": [
    {
      "attributeCode": "productName",
      "channelCode": null,
      "localeCode": "en_US",
      "value": "Signal Multi-Tool"
    },
    {
      "attributeCode": "productName",
      "channelCode": null,
      "localeCode": "fr_FR",
      "value": "Signal Multi-Outil"
    },
    {
      "attributeCode": "weight",
      "channelCode": null,
      "localeCode": null,
      "value": { "amount": 213, "unit": "g" }
    },
    {
      "attributeCode": "price",
      "channelCode": "ecommerce",
      "localeCode": null,
      "value": [{ "currency": "USD", "amount": 129.99 }]
    }
  ]
}
```

This is a **bulk upsert** — each entry either creates a new value or updates an existing one. Values not included in the request are left unchanged (this is not a full replace of all values).

### Write Validation Pipeline

For each value in the request, apply these validations in order:

**Step 1: Attribute exists and belongs to the family**
- Look up the attribute by `attributeCode`
- Confirm it's assigned to the product's family (exists in `family_attributes`)
- Return 400 if the attribute doesn't exist or isn't in the family

**Step 2: Attribute level check (variant hierarchy)**
- Resolve the product's variant definition (walk up to root if needed)
- Resolve the attribute's level in that definition (via `getResolvedAttributeLevels`)
- Confirm the attribute's level matches the product's level
- Return 400 with a clear message if mismatched: "Attribute 'productName' is assigned to level 0 in variant definition 'colorThenBlade'. It must be set on the root product model 'PRD-00003', not on this level-2 variant."
- **Exception for simple products:** All attributes are level 0, and the product is level 0, so all attributes are writable.

**Step 3: Scoping validation**
- If the attribute is `is_localizable: false`, `localeCode` in the request must be `null`. Return 400 otherwise.
- If the attribute is `is_localizable: true`, `localeCode` must be provided and must reference an existing, active locale. Return 400 otherwise.
- Same logic for `is_channelizable` and `channelCode`.
- If the attribute is both localizable and channelizable, both must be provided.

**Step 4: Type-specific value validation**
Validate the `value` field based on the attribute's type:

| Attribute Type | Expected Value Format | Validation |
|---------------|----------------------|------------|
| `text` | `string` | Max length from `validation_rules.maxLength` or `settings.maxLength` |
| `textarea` | `string` | Max length from validation rules |
| `boolean` | `boolean` | Must be `true` or `false` |
| `number` | `number` | Check `validation_rules.minValue`, `maxValue`. Check `settings.decimalsAllowed` |
| `measurement` | `{ "amount": number, "unit": string }` | Unit must be in `settings.allowedUnits` |
| `price` | `[{ "currency": string, "amount": number }]` | Each currency must be in `settings.currencies` |
| `date` | `string` (ISO 8601) | Must be a valid date string |
| `singleSelect` | `string` | Must be a valid option code for this attribute |
| `multiSelect` | `string[]` | Each must be a valid option code |
| `mediaFile` | `string` | File path/URL. Extension check against `settings.allowedExtensions` |
| `assetSingle` | `string` | Asset code (no FK validation — soft reference) |
| `assetList` | `string[]` | Array of asset codes. Check `settings.maxItems` |
| `entityReference` | `string` | Entity code (soft reference) |
| `productReference` | `string` | Product identifier (soft reference) |
| `table` | `object[]` | Array of row objects matching `settings.columns` schema |

Also apply regex validation from `validation_rules.regex` for text/textarea types.

**Step 5: Unique value enforcement**
If the attribute has `is_unique: true`, check that no other product in the same family has the same value for this attribute (in the same scope). Return 409 if a duplicate exists.

**Step 6: Dispatch to correct storage column**
Based on the attribute type, write the value to the appropriate column in `attribute_values`:

| Attribute Type | Storage Column |
|---------------|---------------|
| `text`, `textarea`, `singleSelect`, `assetSingle`, `entityReference`, `productReference` | `value_text` |
| `number` | `value_number` |
| `boolean` | `value_boolean` |
| `date` | `value_date` |
| `measurement`, `price`, `multiSelect`, `assetList`, `table` | `value_json` |
| `mediaFile` | `value_media` |

Use the `setAttributeValue` service function from Task 3 for the actual upsert.

### Write Response

Return the full set of values for the product after the write (same format as the read response). This lets the UI update without a separate read call.

---

## 3. Reading Attribute Values

### Endpoint

**`GET /api/v1/products/:identifier/values`** — Get all attribute values for a product.

Query parameters:
- `?channelCode=ecommerce` — filter to values relevant to a specific channel
- `?localeCode=en_US` — filter to values for a specific locale
- `?includeInherited=true` (default: `true`) — include inherited values from ancestors
- `?attributeCodes=productName,weight,sku` — only return specific attributes

### Read Resolution Pipeline

**Step 1: Get the product and its family**
Load the product, determine its family, and get all attributes assigned to the family.

**Step 2: Resolve variant definition and attribute levels**
If the product is part of a variant hierarchy, resolve the variant definition and determine which level each attribute belongs to.

**Step 3: Load values from this product**
Query `attribute_values` where `entity_type = 'product'` and `entity_id = identifier`, filtered by channel/locale if specified.

**Step 4: Load inherited values (if `includeInherited=true`)**
Walk up the ancestor chain. For each ancestor, load its attribute values. Only include values for attributes that belong to that ancestor's level (according to the variant definition).

**Step 5: Merge and annotate**
Merge own values with inherited values. Own values take precedence if there's any overlap (there shouldn't be if level assignments are correct, but handle it defensively). Annotate each value with its source.

**Step 6: Apply locale fallback**
If a `localeCode` is specified and a localizable attribute has no value for that locale, walk the locale fallback chain (from Task 3) to find a value. Annotate the fallback source (e.g. `source.fromLocale` when the value came from a different locale in the chain). Phase 12.1.1 adds **`GET /api/v1/products/:identifier/values/trace/:attributeCode`** (query: `channelCode`, `localeCode`) to return the full resolution path (locale chain × hierarchy) with a step-by-step trace and a `winner` flag for the step that provided the effective value — used by the Value Resolution Modal in the Product UI.

### Read Response Structure

```json
{
  "data": {
    "identifier": "PRD-00003-red-plain",
    "familyCode": "knife",
    "values": [
      {
        "attributeCode": "productName",
        "type": "text",
        "channelCode": null,
        "localeCode": "en_US",
        "value": "Signal Multi-Tool",
        "source": {
          "type": "inherited",
          "fromIdentifier": "PRD-00003",
          "fromLevel": 0
        }
      },
      {
        "attributeCode": "productName",
        "type": "text",
        "channelCode": null,
        "localeCode": "fr_FR",
        "value": "Signal Multi-Outil",
        "source": {
          "type": "inherited",
          "fromIdentifier": "PRD-00003",
          "fromLevel": 0
        }
      },
      {
        "attributeCode": "handleColor",
        "type": "singleSelect",
        "channelCode": null,
        "localeCode": null,
        "value": "red",
        "source": {
          "type": "inherited",
          "fromIdentifier": "PRD-00003-red",
          "fromLevel": 1
        }
      },
      {
        "attributeCode": "sku",
        "type": "text",
        "channelCode": null,
        "localeCode": null,
        "value": "832-RED-PLAIN",
        "source": {
          "type": "own",
          "fromIdentifier": "PRD-00003-red-plain",
          "fromLevel": 2
        }
      },
      {
        "attributeCode": "price",
        "type": "price",
        "channelCode": "ecommerce",
        "localeCode": null,
        "value": [{ "currency": "USD", "amount": 129.99 }],
        "source": {
          "type": "own",
          "fromIdentifier": "PRD-00003-red-plain",
          "fromLevel": 2
        }
      }
    ]
  }
}
```

### Value Deletion

**`DELETE /api/v1/products/:identifier/values`** — Delete specific attribute values (bulk).

Request body:
```json
{
  "values": [
    {
      "attributeCode": "productName",
      "channelCode": null,
      "localeCode": "fr_FR"
    }
  ]
}
```

Only values **owned** by this product can be deleted. Attempting to delete an inherited value returns 400 with: "Cannot delete inherited value for 'productName'. This value is owned by ancestor 'PRD-00003'."

**`DELETE /api/v1/products/:identifier/values/:attributeCode`** (Phase 12.1.1) — Delete the stored value for a single attribute in a given scope (query params: `channelCode`, `localeCode`). Used to **reset an override**: remove the product's own value so the attribute reverts to showing the inherited value. Same ownership rule: only removes a value stored on this product for that attribute+channel+locale.

---

## 4. Axis Values as Attribute Values

When a child product is created with `axisValues` (Task 8), those axis values should also be written to the `attribute_values` table as regular attribute values. This ensures:
- Axis values are queryable and readable like any other attribute value
- The read endpoint returns axis values alongside all other values
- Completeness calculations can see axis values

When creating a child product with `axisValues: { "handleColor": "red" }`:
1. Create the product row in `products` with `axis_values` JSONB
2. For each axis attribute, call `setAttributeValue` to write it to `attribute_values`:
   - `entity_type: "product"`, `entity_id: "PRD-00003-red"`, `attribute_code: "handleColor"`, `value_text: "red"`, `channel_code: null`, `locale_code: null`

Axis attribute values are **permanently read-only** through the values endpoint. Since axis values are baked into the product's identifier (Task 8, Section 3), they cannot be changed after creation. The PUT values endpoint should reject writes to axis attributes with: "Attribute 'handleColor' is an axis attribute. Axis values are immutable because the product identifier is derived from them."

---

## 5. Performance Considerations

### Batch Reads
The read endpoint for a variant product requires loading values from multiple products (the product itself and all ancestors). Optimize this:

- **Single query approach:** Load all attribute values for the entire ancestry chain in one query:
  ```sql
  SELECT av.* FROM attribute_values av
  WHERE av.entity_type = 'product'
    AND av.entity_id IN ('PRD-00003', 'PRD-00003-red', 'PRD-00003-red-plain')
  ORDER BY av.attribute_code, av.channel_code, av.locale_code;
  ```
  Then merge in application code, giving precedence to deeper-level values.

- **Cache the ancestor chain:** The ancestor identifiers for a product don't change (products can't be re-parented). Cache the list of ancestor identifiers for each product to avoid walking the tree on every read.

### Batch Writes
The PUT endpoint accepts multiple values. Process all writes in a single database transaction to ensure atomicity. If any value fails validation, reject the entire batch and return all errors.

---

## 6. Service Layer Functions

```typescript
interface ProductValueEntry {
  attributeCode: string;
  channelCode: string | null;
  localeCode: string | null;
  value: any;
}

interface ProductValueResult {
  attributeCode: string;
  type: string;
  channelCode: string | null;
  localeCode: string | null;
  value: any;
  source: {
    type: "own" | "inherited";
    fromIdentifier: string;
    fromLevel: number;
    fromLocale?: string;   // set when value came via locale fallback (Phase 12.1.1)
    fromChannel?: string; // reserved for future channel fallback
  };
}

// Write values to a product (with full validation pipeline)
async function setProductValues(
  identifier: string,
  values: ProductValueEntry[],
  userId: string
): Promise<ProductValueResult[]>

// Read all values for a product (with inheritance and fallback)
async function getProductValues(
  identifier: string,
  options?: {
    channelCode?: string;
    localeCode?: string;
    includeInherited?: boolean;
    attributeCodes?: string[];
  }
): Promise<ProductValueResult[]>

// Delete specific values from a product
async function deleteProductValues(
  identifier: string,
  values: Array<{ attributeCode: string; channelCode: string | null; localeCode: string | null }>,
  userId: string
): Promise<void>

// Get the ancestor chain for a product (cached)
async function getProductAncestorChain(
  identifier: string
): Promise<string[]>

// Resolve which level an attribute belongs to for a product's variant definition
async function getAttributeLevelForProduct(
  identifier: string,
  attributeCode: string
): Promise<number>
```

---

## 7. Audit History

Wire `logAudit()` calls for all value writes and deletes:

| Resource Type | Actions | What's Logged |
|---------------|---------|---------------|
| `productValue` | update | `resourceId` = product identifier, `attributeCode` = the attribute, `channelCode` and `localeCode` for scoped values, `oldValue` and `newValue` |
| `productValue` | delete | Same as above, `newValue` = null |

For bulk writes (multiple values in one PUT), log one audit entry per value changed. Use `logAuditBatch()` for efficiency.

---

## 8. Business Rules

1. **Attribute values can only be written at the correct level.** The variant definition determines which product in the hierarchy owns each attribute. Enforce this on every write.

2. **Axis attribute values are managed through the product entity, not the values endpoint.** Reject writes to axis attributes via PUT values.

3. **Inherited values are read-only from a child's perspective.** A variant cannot modify or delete an attribute value that belongs to its parent or grandparent. The UI will make this clear by showing inherited values as non-editable.

4. **Scoping rules are strictly enforced.** A non-localizable attribute cannot have a `localeCode` value. A non-channelizable attribute cannot have a `channelCode` value.

5. **Type-specific validation is enforced on every write.** Invalid values are rejected with descriptive error messages including the attribute code, expected type, and the specific validation that failed.

6. **Unique value enforcement spans all products in the same family** (within the same scope). A unique `sku` attribute means no two products in the same family can have the same SKU value.

7. **Locale fallback is read-only behavior.** Fallback resolution is applied when reading values. It does not affect storage — values are always stored against the specific locale they were entered for.

8. **Deleting a product's parent does not orphan values.** Cascading product deletion (Task 8) also deletes all attribute values. If a parent is deleted without cascade, children still exist with their own values, but inherited values from the deleted parent are lost.
