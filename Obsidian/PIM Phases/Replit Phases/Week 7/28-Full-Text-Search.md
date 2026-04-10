# Phase 28: Full-Text Search

**Execution Plan Item:** #28  
**Replit Phase:** 28 (Week 7)  
**Dependencies:** Phase 04 (Families), Phase 08 (Products schema), Phase 09 (Product Attribute Values), Phase 10 (Categories & Status), Phase 11 (Completeness — `calculateCompletenessOverview` for completeness filtering), Phase 12.1 (Product Inheritance — inherited values in variant search documents), Phase 26 (real imported data for meaningful search testing), Phase 27 (user locale preference for `displayName` resolution)  
**Depended on by:** Phase 30 (Search UI), Phase 35 (MCP Server search tool)

---

## Objective

Add **full-text search** across the PIM product catalog using **PostgreSQL's built-in full-text search**. Search must cover product identifiers, **product names**, attribute values (text and textarea types, labels), family labels, and category labels — the data a user would naturally type when looking for a product. Layer **faceted filtering** on top of search so users can narrow results by family, category, channel, status, and completeness.

Because product names are stored as localizable text attribute values (not a direct column on the product), the search index includes text values across **all locales** to maximize recall — a search for a product name works regardless of locale. Search results include a `displayName` field resolved using the requesting user's locale preference (from Phase 27 user settings), with fallback to English if no value exists in the preferred locale.

The implementation uses Postgres `tsvector` / `tsquery` with a **materialized search document** per product. This avoids re-scanning all attribute values on every query and keeps search fast even for catalogs in the tens of thousands of products. If volume demands exceed Postgres capacity in the future, the document model makes it straightforward to sync to Elasticsearch or Typesense without changing the API surface.

---

## 1. Search Document Model

### What Gets Indexed

Each product is represented by a single search document. The document combines:

| Source | Weight | Examples |
|--------|--------|---------|
| Product identifier | A (highest) | `42`, `42_1`, `830417` |
| Attribute values — `text` and `textarea` types, **all locales** | B | Product name (en, de, fr, …), description, subtitle |
| Attribute labels (all locales) | C | "Product Name", "Nom du produit" |
| Family labels (all locales) | C | "Multitool", "Multi-Outil" |
| Category labels (all locales) | D (lowest) | "Hand Tools", "Knives" |

Weights map to Postgres `tsvector` letter weights `'A'`, `'B'`, `'C'`, `'D'`, which `ts_rank` uses to score matches. Identifier matches rank highest; category label matches rank lowest.

**Product name is the primary weight-B target.** Since it is a localizable text attribute, all locale variants are concatenated into the document so that searching for a product's name in any locale returns a match. The `displayName` field in search results is resolved separately at query time using the user's locale preference (see §3).

Numeric values (numbers, prices, measurements) and boolean values are **not** indexed for full-text — they are only used as filter facets. Select option labels can be included at weight D if option labels are meaningful (e.g. "Black", "Stainless Steel") to enable searching by option label.

### Table: `product_search_documents`

```sql
CREATE TABLE product_search_documents (
  product_identifier VARCHAR(255) PRIMARY KEY REFERENCES products(identifier) ON DELETE CASCADE,
  search_vector      TSVECTOR NOT NULL,
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_search_vector ON product_search_documents USING GIN (search_vector);
```

One row per product (including all levels — root models, sub-models, and variants each have their own document). For variant hierarchies, the document for a variant includes its own values AND the inherited values from its ancestors (merged), so searching for a root-level attribute value matches leaf variants that inherit it.

### Document Build Function

```typescript
async function buildSearchDocument(identifier: string): Promise<string>
```

Builds the combined text string for a product. Steps:

1. Load the product record (identifier, family, status, type).
2. Resolve all attribute values for this product using the full inheritance chain (Phase 12.1: hierarchy + locale fallback). Collect `text` and `textarea` values **across all channels and all locales** — do not filter to a single locale. This ensures that searching by a product name in any locale (e.g. a German user searching in German) produces a match.
3. Load family labels and category labels for the product.
4. Concatenate all text fragments, separated by spaces.
5. Convert to `tsvector` using `to_tsvector('simple', ...)` with weighted sections via `setweight()`. Use `'simple'` (no language-specific stemming) since the catalog contains multilingual text values (French, German, etc.) and `'english'` stemming would corrupt non-English terms.

Example SQL construction:

```sql
setweight(to_tsvector('simple', coalesce(identifier, '')), 'A') ||
setweight(to_tsvector('simple', coalesce(text_attribute_values, '')), 'B') ||
setweight(to_tsvector('simple', coalesce(family_labels, '')), 'C') ||
setweight(to_tsvector('simple', coalesce(category_labels, '')), 'D')
```

---

## 2. Index Population & Maintenance

### Initial Build (Migration)

Add a migration step that builds the `product_search_documents` table and runs `buildSearchDocument` for every existing product. For a catalog of tens of thousands of products this runs as a background migration — do not block the server startup.

### Incremental Updates (Triggers or Hooks)

The search document must be kept fresh. Update a product's document when:

- A product attribute value is written (`PUT /api/v1/products/:identifier/values`).
- A product's status, family, or category assignment changes.
- A family's labels change (updates all products in that family).
- A category's labels change (updates all products in that category).

**Implementation:** Add an `invalidateSearchDocument(identifier)` call at the end of each relevant write path. `invalidateSearchDocument` queues the document for rebuild. For MVP, execute the rebuild **synchronously within the same request** (after the primary write commits) using a simple update:

```typescript
await storage.rebuildSearchDocument(identifier);
```

For large-scale updates (e.g. renaming a family label affects thousands of products), batch the rebuild in the background. Accept brief staleness for bulk attribute renames. Document this as a known limitation.

### Ancestor/Descendant Propagation

When a root model's attribute values change, leaf variants' search documents must also be rebuilt (since leaf documents include inherited values). When `invalidateSearchDocument` is called for a root model, also enqueue rebuilds for all its descendants (up to depth 2 for MVP — direct children and grandchildren). If a catalog has very deep trees or huge fan-out, this can be deferred to a background job.

---

## 3. Search API Endpoint

### `GET /api/v1/products/search`

Full-text search with faceted filters. All parameters are optional; omitting `q` returns all products (pure facet filter mode).

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | `string` | Full-text search query. Supports plain text; treated as prefix match (e.g. `q=signal` matches "Signal Multi-Tool"). |
| `familyId` | `number` | Filter by family ID (exact match). |
| `categoryId` | `number` | Filter by category ID. Includes products in descendants of the category (recursive). |
| `status` | `string` | Filter by status: `draft`, `active`, `archived`. Comma-separated for multiple. |
| `locale` | `string` | Locale code for resolving `displayName` in the response (e.g. `en`, `de`). If omitted, the user's locale preference from their account settings (Phase 27) is used. Falls back to English if neither is set or has a value. Does **not** restrict which products are returned — search always queries across all locales. |
| `channelCode` | `string` | Combined with `completeness` filter to scope completeness check. |
| `localeCode` | `string` | Combined with `completeness` filter. |
| `completeness` | `string` | Completeness filter: `100` (complete only), `lt:100` (incomplete), `gte:80` (at least 80%). Requires `channelCode` and `localeCode`. |
| `type` | `string` | Product type: `product`, `model`, `variant`. Comma-separated. Defaults to `product,model` (root-level only) if not specified. Note: standalone products use type `product` (renamed from `simple` in Phase 09.6). |
| `root` | `boolean` | If `true`, limit to root-level products only (`level = 0`). Overrides `type`. |
| `limit` | `number` | Page size. Default 25, max 100. |
| `cursor` | `string` | Cursor for pagination (from previous response's `meta.nextCursor`). |

**Response:**

```json
{
  "data": [
    {
      "identifier": "42",
      "productNumber": 42,
      "displayName": "Signal Multi-Tool",
      "type": "model",
      "level": 0,
      "status": "active",
      "familyId": 3,
      "familyLabels": { "en_US": "Multitool" },
      "categories": [
        { "id": 7, "labels": { "en_US": "Hand Tools" } }
      ],
      "rank": 0.8231,
      "matchedOn": "identifier"
    }
  ],
  "meta": {
    "total": 14,
    "limit": 25,
    "cursor": null,
    "nextCursor": null,
    "query": "signal"
  },
  "facets": {
    "families": [
      { "id": 3, "labels": { "en_US": "Multitool" }, "count": 9 },
      { "id": 5, "labels": { "en_US": "Knife" }, "count": 5 }
    ],
    "statuses": [
      { "value": "active", "count": 11 },
      { "value": "draft", "count": 3 }
    ],
    "categories": [
      { "id": 7, "labels": { "en_US": "Hand Tools" }, "count": 14 }
    ]
  }
}
```

**`displayName`** is the product's name resolved for the effective locale (see below). `null` if no text attribute value exists for the product in the effective locale or any fallback.  
**`rank`** is the `ts_rank` score from Postgres. Omit if no `q` was provided (facet-only queries).  
**`matchedOn`** is an optional hint about what the top match was (identifier, attribute value, family label, category label). Omit for MVP if complex to compute.  
**`facets`** are counts across the **entire result set** (before pagination), allowing the UI to render facet counts without a separate request.

### `displayName` Resolution

`displayName` is resolved at query time (not stored in the search document) using the following logic:

1. Determine the **effective locale**: use the `locale` query parameter if provided; otherwise use the authenticated user's locale preference from their account settings (Phase 27); otherwise fall back to `en`.
2. Find the product's **label attribute** — the text/textarea attribute designated as the display name for the product's family, if one is tracked in the data model. If no label attribute is designated, use the first text or textarea attribute value with a non-empty string.
3. Return the value in the effective locale. If no value exists in the effective locale, fall back to English. If no English value exists, return the first non-empty text value found in any locale.
4. Apply inheritance (Phase 12.1): if the product (e.g. a variant) has no name value of its own, walk up to the parent model.

This resolution happens in application code after the Postgres search query returns identifiers. Batch the attribute value lookups to avoid N+1 queries.

### Search Query Handling

- Convert `q` to a Postgres `tsquery` using `plainto_tsquery('english', $q)` for plain text or `websearch_to_tsquery('english', $q)` for web-style queries (supports `"phrase search"`, `-exclusion`).
- **Prefix matching:** Append `:*` to the last word in the tsquery to support incremental search (e.g. typing "sign" matches "signal"). Use `to_tsquery('english', 'sign:*')` for single-word prefix.
- **Ranking:** Use `ts_rank_cd(search_vector, query)` (cover density ranking). Sort by rank DESC, then identifier ASC as a stable tiebreaker.

### Facet Computation

Compute facets as aggregations on the filtered result set (before pagination):

```sql
SELECT family_id, COUNT(*) FROM ...
WHERE <search and filter conditions>
GROUP BY family_id
```

Run facet aggregation queries in parallel with the main result query to minimize latency.

---

## 4. Storage Methods

Add to `IStorage` and `DatabaseStorage`:

| Method | Signature | Description |
|--------|-----------|-------------|
| `searchProducts` | `(params: SearchParams) → { products: Product[], total: number, nextCursor: string \| null, facets: SearchFacets }` | Full search + facets. |
| `rebuildSearchDocument` | `(identifier: string) → void` | Build and upsert the `product_search_documents` row for one product. |
| `rebuildSearchDocumentsBatch` | `(identifiers: string[]) → void` | Batch rebuild for multiple products. |
| `getSearchDocumentStatus` | `() → { total: number, indexed: number }` | Admin/diagnostic: how many products are indexed. |

---

## 5. REST API — Existing Endpoints

The existing `GET /api/v1/products` endpoint is **not replaced**. The new search endpoint is additive. The existing endpoint continues to serve the product list page and simple filters. The search endpoint is the recommended path when a `q` query is present or when facet counts are needed.

Optionally, extend `GET /api/v1/products` with a `?q=` parameter that delegates to the search backend when present. This keeps the API surface minimal and avoids two separate endpoints in the UI. Evaluate during implementation — if routing complexity is low, prefer a single endpoint; otherwise keep them separate.

---

## 6. Faceted Filtering Without Full-Text

When `q` is absent, the search endpoint acts as a **faceted filter**: return all products matching the applied facets, with facet counts for all dimensions. This is the "browse by category + family" use case in Phase 30.

SQL for facet-only mode (no `WHERE search_vector @@ query` clause):

```sql
SELECT p.*, ts_rank(...) as rank
FROM products p
JOIN product_search_documents psd ON psd.product_identifier = p.identifier
WHERE
  ($familyId IS NULL OR p.family_id = $familyId)
  AND ($status IS NULL OR p.status = ANY($status))
  AND <category filter via product_categories join>
ORDER BY p.updated_at DESC
LIMIT $limit
```

---

## 7. Category Filter (Recursive)

When `categoryId` is provided, match products assigned to that category **or any of its descendants**. Use a recursive CTE to collect all descendant category IDs, then join to `product_categories`:

```sql
WITH RECURSIVE cat_tree AS (
  SELECT id FROM categories WHERE id = $categoryId
  UNION ALL
  SELECT c.id FROM categories c
  JOIN cat_tree ct ON c.parent_id = ct.id
)
SELECT DISTINCT product_identifier FROM product_categories
WHERE category_id IN (SELECT id FROM cat_tree)
```

---

## 8. Completeness Filter

When `completeness`, `channelCode`, and `localeCode` are provided, filter the result set by completeness score. Since completeness is calculated on demand (not stored), this filter operates in application code after fetching the matched product set:

1. Run the search/filter query to get matching product identifiers (without completeness filter).
2. Batch-calculate completeness for those identifiers using `calculateCompletenessOverview` (Phase 11).
3. Filter the identifiers to those meeting the completeness threshold.
4. Fetch and return the filtered product records.

**Limitation:** This means completeness filtering is not reflected in `total` count or facet counts. Document this as a known limitation for MVP. A future optimization can materialize completeness scores for frequent channel+locale pairs.

---

## 9. Exit Criteria

- [ ] `product_search_documents` table and GIN index created in migration. All existing products have search documents built.
- [ ] `GET /api/v1/products/search?q=<term>` returns ranked results matching product identifiers, text attribute values (including product names in all locales), family labels, and category labels.
- [ ] Searching by product name works: `q=Signal` returns products whose name attribute contains "Signal" in any indexed locale.
- [ ] `displayName` in each result reflects the product's name in the effective locale (from `locale` param → user preference → English fallback).
- [ ] Prefix matching works: `q=sign` matches products containing "Signal".
- [ ] Facet counts (`families`, `statuses`, `categories`) returned in every response and reflect the full matched set (before pagination).
- [ ] `familyId`, `categoryId` (recursive), `status`, `type`, and `root` filters all work independently and in combination.
- [ ] Search documents are updated when product values, status, family assignment, or category assignment changes.
- [ ] `completeness` + `channelCode` + `localeCode` filter correctly narrows results (with documented limitation on total count accuracy).
- [ ] Search endpoint requires authentication (same as REST API).
- [ ] For a catalog of ~2,000 products, `q=signal` returns results in under 200ms (GIN index is used — verify with `EXPLAIN ANALYZE`).

---

## 10. Notes

- **Elasticsearch / Typesense migration path.** The `product_search_documents` table is a clean abstraction: the sync layer calls `rebuildSearchDocument`, and the search layer queries it. Swapping the backend means replacing only the rebuild and query implementations, not the API or the UI.
- **Language configuration.** The document build uses `to_tsvector('simple', ...)` (no stemming) because the catalog contains multilingual text values. `'simple'` treats every word as a token without language-specific stemming, which is correct for a multilingual index — `'english'` would mangle French and German terms. The trade-off is that `'simple'` doesn't collapse English variants (e.g. "running" and "run" are distinct tokens), but this is acceptable for an MVP where exact-word and prefix matching are the primary use cases.
- **Axis values not indexed.** Variant axis values (e.g. `{ "handleColor": "red" }`) are stored as JSON. The axis option labels (e.g. "Red", "Black") are included in attribute values at the variant level and therefore indexed. The JSON key names (attribute codes) are not indexed as text — they are not useful search terms.
- **Inherited values in variant documents.** Leaf variant documents include inherited values (e.g. a root-level product name). This means searching for a product name always returns the leaf variants, not just the root. The `root: true` filter or `type: model` filter can be used to restrict to top-level results only.
