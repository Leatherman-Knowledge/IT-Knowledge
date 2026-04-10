# Task 10: Product-Category Assignment & Product Status

**Execution Plan Item:** #10
**Dependencies:** Products (Task 8), Categories (Task 5)
**Depended on by:** Completeness (Task 11 — channels reference a category tree), Product UI (Task 12)

---

## Objective

Build the product-to-category assignment system and formalize product status management. Products can belong to multiple categories across multiple trees, and categories serve as the organizational and navigation backbone for the catalog. Product status (draft, active, archived) gates visibility and governs which products are considered "publishable."

---

## 1. Database Schema

### Table: `product_categories`

Junction table linking products to categories. A product can belong to any number of categories across any number of trees.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `product_identifier` | `VARCHAR(255)` | NOT NULL, FK → products.identifier, ON DELETE CASCADE | |
| `category_code` | `VARCHAR(255)` | NOT NULL, FK → categories.code, ON DELETE CASCADE | |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | When this assignment was made |
| PRIMARY KEY | | `(product_identifier, category_code)` | A product can only be in a category once |

**Indexes:**

```sql
CREATE INDEX idx_pc_product ON product_categories(product_identifier);
CREATE INDEX idx_pc_category ON product_categories(category_code);
CREATE INDEX idx_pc_category_tree ON product_categories(category_code)
  INCLUDE (product_identifier);
```

### Which Product in the Hierarchy Gets Category Assignments?

Category assignments are made on the **root-level product** (the simple product or the root product model):
- For a **simple product**, categories are assigned directly on the product.
- For a **variant hierarchy**, categories are assigned on the **root product model**. All variants in the hierarchy inherit the root's category assignments. You do not assign categories to individual variants or sub-models.

This matches how categories work in practice: a "Signal Multi-Tool" belongs to the "Knives > Folding Knives" category regardless of which color/blade variant is being viewed.

The API should enforce this: attempts to assign a category to a non-root product (a variant or sub-model) return 400 with: "Categories can only be assigned to root-level products. This product's root is 'PRD-00003'."

---

## 2. API Endpoints

### Category Assignment

**`GET /api/v1/products/:identifier/categories`** — Get all categories assigned to a product.

For root products, returns the directly assigned categories. For variants/sub-models, returns the categories inherited from the root model (with an annotation indicating they're inherited).

Response:
```json
{
  "data": [
    {
      "categoryCode": "foldingKnives",
      "treeCode": "masterCatalog",
      "labels": { "en_US": "Folding Knives" },
      "path": "handTools.knives.foldingKnives",
      "inherited": false
    },
    {
      "categoryCode": "edc",
      "treeCode": "webNavigation",
      "labels": { "en_US": "Everyday Carry" },
      "path": "lifestyle.edc",
      "inherited": false
    }
  ]
}
```

When called on a variant (e.g., `PRD-00003-red-plain`), the same categories are returned but with `"inherited": true` and an additional `"rootIdentifier": "PRD-00003"` field.

**`PUT /api/v1/products/:identifier/categories`** — Set the full list of category assignments for a product. This is a **full replace** — the provided list completely replaces all existing assignments.

```json
{
  "categoryCodes": ["foldingKnives", "edc", "newArrivals"]
}
```

Returns 400 if the product is not a root-level product.

**`POST /api/v1/products/:identifier/categories`** — Add one or more categories to a product (additive, does not remove existing).

```json
{
  "categoryCodes": ["newArrivals"]
}
```

Returns 400 if the product is not a root-level product. Silently ignores categories already assigned.

**`DELETE /api/v1/products/:identifier/categories/:categoryCode`** — Remove a single category from a product.

Returns 400 if the product is not a root-level product. Returns 404 if the category is not currently assigned.

### Products by Category

**`GET /api/v1/categories/:categoryCode/products`** — Get all products assigned to a specific category.

Query parameters:
- `?includeDescendants=true` — also include products assigned to any descendant category
- `?status=active` — filter by product status
- `?familyCode=knife` — filter by family
- Standard cursor-based pagination

When `includeDescendants=true`, uses the materialized path on categories (from Task 5) to efficiently find all descendant category codes, then queries the junction table.

This endpoint returns root-level products only (simple products and root models), since category assignments live on roots.

---

## 3. Product Status

### Status Values

Products have a `status` field with three values:

| Status | Meaning |
|--------|---------|
| `draft` | Work in progress. Not visible to downstream channels. Default for new products. |
| `active` | Published and available. Visible to channels and external consumers. |
| `archived` | Retired from active use. Hidden from default views but data is preserved. |

### Status Behavior

- **Default:** New products are created with `status = 'draft'`.
- **Transitions:** Any status can transition to any other status. There are no enforced workflow gates in this task.
- **Variant inheritance:** Status is set independently on each product in a hierarchy. A root model can be `active` while some variants are still `draft`. The API and UI allow this — it's normal for new variants to be added to an existing active model.
- **Filtering:** The product list endpoint (Task 8) already supports `?status=` filtering. Default list views should filter to `draft` and `active` (excluding `archived`).

### Status Update Endpoint

Status is updated via the existing `PATCH /api/v1/products/:identifier` endpoint:

```json
{
  "status": "active"
}
```

### Bulk Status Update

**`POST /api/v1/products/bulk-status`** — Update the status of multiple products at once.

```json
{
  "identifiers": ["PRD-00003", "PRD-00003-red", "PRD-00003-red-plain", "PRD-00003-red-serrated"],
  "status": "active"
}
```

Response:
```json
{
  "data": {
    "updated": 4,
    "identifiers": ["PRD-00003", "PRD-00003-red", "PRD-00003-red-plain", "PRD-00003-red-serrated"]
  }
}
```

Maximum 100 identifiers per request. Returns 400 if any identifier doesn't exist (fails the entire batch).

---

## 4. Channel–Category Relationship

Each channel (Task 3) references a `category_tree_code`. This creates a natural scoping mechanism: a product "belongs to" a channel if it's assigned to any category within that channel's category tree.

Implement a utility function to determine which channels a product belongs to:

```typescript
async function getProductChannels(
  productIdentifier: string
): Promise<string[]> {
  // 1. Get the root product identifier (walk up if needed)
  // 2. Get all category codes assigned to the root product
  // 3. For each category, look up its tree_code
  // 4. Find all channels whose category_tree_code matches any of those trees
  // 5. Return unique channel codes
}
```

This function is used by the completeness engine (Task 11) to determine which channels a product's completeness should be evaluated against.

### Endpoint

**`GET /api/v1/products/:identifier/channels`** — Get all channels this product belongs to (derived from its category assignments).

Response:
```json
{
  "data": [
    {
      "channelCode": "ecommerce",
      "labels": { "en_US": "E-Commerce" },
      "localeCodes": ["en_US", "fr_FR"],
      "currencyCodes": ["USD", "EUR"]
    },
    {
      "channelCode": "retail",
      "labels": { "en_US": "Retail" },
      "localeCodes": ["en_US"],
      "currencyCodes": ["USD"]
    }
  ]
}
```

---

## 5. Storage Layer Methods

### New `IStorage` Methods

| Method | Purpose |
|--------|---------|
| `getProductCategories(identifier)` | Get all categories assigned to a product (with tree/label info) |
| `setProductCategories(identifier, categoryCodes[])` | Full replace of category assignments |
| `addProductCategories(identifier, categoryCodes[])` | Add categories (additive) |
| `removeProductCategory(identifier, categoryCode)` | Remove a single category |
| `getProductsByCategory(categoryCode, options)` | Get products in a category (with descendant and filter options) |
| `getProductChannels(identifier)` | Derive channels from category assignments |
| `bulkUpdateProductStatus(identifiers[], status)` | Update status for multiple products |

---

## 6. Business Rules

1. **Category assignments live on root-level products only.** Variants and sub-models inherit their root's categories. Attempting to assign a category to a non-root product is rejected.

2. **A product can belong to any number of categories across any number of trees.** There is no limit on how many categories a product can have.

3. **Category codes must reference existing, active categories.** Validate on assignment. Return 400 with the specific invalid codes.

4. **Duplicate assignments are silently ignored** on the POST (additive) endpoint. On the PUT (replace) endpoint, duplicates in the request body are deduplicated.

5. **Deleting a category** (Task 5) cascades to the junction table via the ON DELETE CASCADE FK. Products that were in the deleted category are simply no longer in it.

6. **Deleting a product** cascades to the junction table via ON DELETE CASCADE.

7. **Product status defaults to `draft`** on creation. All three statuses are valid for any product type (simple, model, variant).

8. **Bulk status updates are atomic.** If any identifier in the batch doesn't exist, the entire operation fails. This prevents partial updates that could leave hierarchies in inconsistent states.

9. **Channel membership is derived, not stored.** There is no `product_channels` table. A product's channels are computed from its category assignments and the channel-to-tree mapping. This means adding/removing a category can change which channels a product appears in.

---

## 7. Audit History

Wire `logAudit()` calls for:

| Resource Type | Actions | What's Logged |
|---------------|---------|---------------|
| `productCategory` | update | `resourceId` = product identifier, `oldValue` = previous category codes, `newValue` = new category codes (on PUT replace) |
| `productCategory` | create | `resourceId` = product identifier, `newValue` = added category codes (on POST additive) |
| `productCategory` | delete | `resourceId` = product identifier, `oldValue` = removed category code |
| `product` | update | Status changes logged as field-level update with old and new status |
