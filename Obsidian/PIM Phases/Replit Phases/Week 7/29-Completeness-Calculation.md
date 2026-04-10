# Phase 29: Completeness Calculation

**Execution Plan Item:** #29  
**Replit Phase:** 29 (Week 7)  
**Dependencies:** Phase 04 (Families & Requirements), Phase 09 (Product Attribute Values), Phase 11 (Initial Completeness), Phase 12.1 (Corrections: locale reset, per-variant, interactive), Phase 12.1.1 (Recursive descendants, optional categories), Phase 03 (Locales with fallback chain)  
**Depended on by:** Phase 28 (completeness filter in search), Phase 30 (Completeness UI), Phase 27 (GraphQL completeness field)

---

## Objective

Harden and complete the completeness calculation engine that was initially built in Phase 11 and corrected in Phase 12.1/12.1.1. This phase is a **completeness pass** — it does not rebuild the engine from scratch. It ensures:

1. Per-variant, per-locale, per-channel completeness with the **locale completeness reset** (Phase 12.1 §3) is fully and correctly implemented.
2. The calculation correctly handles **recursive descendant** hierarchies (Phase 12.1.1).
3. The REST API exposes all completeness queries needed by the Phase 30 UI and Phase 27 GraphQL.
4. The **missing-attribute breakdown** is queryable per variant, per locale/channel, with enough detail to deep-link into the product edit form.
5. An **aggregate completeness score** is available per product model (across all variants, all channels, all locales) for dashboard-level display.
6. Completeness is accurate with the real Akeneo-imported dataset (Phase 26), not just with hand-crafted test products.

If the Phase 11 / Phase 12.1 implementation already satisfies all of the above, this phase documents the verification and any gap fixes. If gaps exist, they are fixed here.

---

## 1. Recap: Completeness Rules (Canonical)

Phase 29 is the canonical reference for completeness behavior. Any future phase that touches completeness should align with this document.

### Rule 1 — Calculated on Demand, Never Stored

Completeness is always computed from the current state of the `attribute_values`, `family_attribute_requirements`, and `attribute` tables. It is never cached in a column on the `products` table. This guarantees accuracy when requirements change (e.g. a family adds a new required attribute) and when values are added.

Exception: The Phase 28 `completeness` filter uses a batch calculation in application code, not a stored score. If materialized completeness becomes necessary for performance, it can be added as a separate optimization with explicit cache invalidation.

### Rule 2 — Completeness Uses the Merged (Inherited) Value Set

For variants, completeness evaluates whether a required attribute has **any value** — own or inherited from an ancestor. This matches Phase 12.1 §1: "first available value = hierarchy + locale (and channel) fallback."

Exception: **Locale completeness reset** (Rule 5 below) overrides this for locale-scoped completeness.

### Rule 3 — Present = Non-Null, Non-Empty

A value is **present** if:
- `value_text` is not NULL and not `''`
- `value_number` is not NULL
- `value_boolean` is not NULL (both `true` and `false` count)
- `value_date` is not NULL
- `value_json` is not NULL and not `[]` and not `{}`
- `value_media` is not NULL and not `''`

A required attribute counts toward completeness only when its value is present.

### Rule 4 — Completeness is Per Channel and Per Locale

The same product may have different completeness scores for different channel+locale combinations because:
- Different channels may have different required attributes (`family_attribute_requirements.channel_id`).
- Attribute scoping (localizable/channelizable flags) determines which rows in `attribute_values` are relevant.

### Rule 5 — Locale Completeness Reset

If a locale `L` has `completeness_resets_after_fallback = true` (Phase 12.1 §3.2):
- For calculating completeness **for locale `L`**, a required localizable attribute is counted as present **only if** a value with `locale_code = L` exists (or with a locale that falls back to `L` without crossing another reset boundary).
- Values found only via `L`'s `fallback_locale_code` do **not** count toward `L`'s completeness.

This means filling the `en_US` locale fully does not make `fr_FR` complete, even though `fr_FR` displays `en_US` values as fallback. The user must enter French content to improve `fr_FR` completeness.

Completeness **display** still uses the full inheritance + fallback chain (first available value). The reset only affects the *scoring* calculation.

### Rule 6 — Family with Zero Requirements = 100% Complete

If a family has no requirements for a given channel+locale, completeness is 100% (nothing required). This is intentional.

### Rule 7 — Boolean Attributes Count as Present for `false`

`value_boolean = false` is a valid present value. Only `NULL` means absent.

### Rule 8 — Empty String Does Not Count

`value_text = ''` is treated as absent. Users must not be able to satisfy completeness by entering blank values.

### Rule 9 — Locale Fallback Does Not Apply to Completeness Scoring (Without Reset)

For locales that do **not** have `completeness_resets_after_fallback = true`, completeness still follows the **full inheritance chain** (hierarchy) — a variant inherits required attribute values from its ancestor. But locale fallback (falling back to a different locale) does not make a localizable attribute "complete" for the requested locale.

Example: `productName` is required for `en_US` and `fr_FR`. The product has `productName` only in `en_US`. Completeness for `en_US` = 100%. Completeness for `fr_FR` = 0% for that attribute (because the value is `locale_code = 'en_US'`, which is not `fr_FR`, even though it displays as `fr_FR` via fallback).

---

## 2. Completeness Calculation API (Canonical)

### Core Function

```typescript
interface CompletenessInput {
  productIdentifier: string;
  channelCode: string;
  localeCode: string;
}

interface CompletenessResult {
  channelCode: string;
  localeCode: string;
  percentage: number;           // 0–100, integer
  complete: number;             // required attrs with present values
  required: number;             // total required attrs for this channel+locale
  missingAttributes: Array<{
    attributeId: number;
    attributeCode: string;
    labels: Record<string, string>;
    type: string;
    groupId: number | null;
  }>;
  localeResetApplied: boolean;  // true if locale has completeness_resets_after_fallback
}

async function calculateCompleteness(input: CompletenessInput): Promise<CompletenessResult>
```

### All Channels and Locales (Per Product)

```typescript
async function calculateAllCompleteness(
  productIdentifier: string
): Promise<CompletenessResult[]>
```

Returns an array of `CompletenessResult`, one per (channel, locale) combination applicable to this product (channels the product belongs to, plus each channel's active locales). Products with no category assignments have no applicable channels; return an empty array.

### Batch Overview (For Product List)

```typescript
interface CompletenessOverview {
  productIdentifier: string;
  scores: Record<string, Record<string, number>>;  // channel → locale → percentage
  lowestScore: number;    // minimum score across all channel+locale combos
  isComplete: boolean;    // true if all applicable channel+locale combos are 100%
}

async function calculateCompletenessOverview(
  identifiers: string[]
): Promise<CompletenessOverview[]>
```

Used by the product list for batch rendering of completeness badges. Must use batched DB queries — no per-product loops. Return `isComplete = true` when all applicable combos are 100% (or the product has no applicable channels).

### Aggregate Score (Per Root Model)

```typescript
interface ModelCompletenessAggregate {
  modelIdentifier: string;
  variantCount: number;
  completeVariantCount: number;   // variants at 100% for ALL channel+locale combos
  lowestVariantScore: number;     // lowest score across all variants × channels × locales
  perChannel: Array<{
    channelCode: string;
    averagePercentage: number;    // average across variants and locales
    locales: Array<{
      localeCode: string;
      averagePercentage: number;  // average across variants
    }>;
  }>;
}

async function calculateModelCompleteness(
  modelIdentifier: string
): Promise<ModelCompletenessAggregate>
```

Provides the roll-up completeness for a product model based on all its leaf variants. Used in the Phase 30 completeness dashboard to show "% of variants complete" for a model.

---

## 3. Locale Reset Implementation Detail

When `locales.completeness_resets_after_fallback = true` for the requested `localeCode`:

**Step 1:** Get the family's requirements for this channel+locale (same as normal).

**Step 2:** For each required attribute, determine if it is localizable:
- **Global (not localizable):** Check for a value with `locale_code IS NULL`. Inherited global values count (hierarchy walk applies). Reset does not affect global attributes.
- **Localizable:** Check for a value with `locale_code = localeCode`. Walk the hierarchy (inherit from parent), but **only count** values stored with `locale_code = localeCode` exactly. Do **not** count values stored with the fallback locale code.

**Step 3:** Calculate percentage using only the "locale-reset-aware" set of present values.

**Step 4:** Set `localeResetApplied: true` in the result. The missing attributes list includes any attribute counted as missing due to the reset (even if a fallback value would display in the UI).

---

## 4. REST API Endpoints

These endpoints should already exist from Phase 11. Verify they are correct and add any that are missing:

### `GET /api/v1/products/:identifier/completeness`

Returns completeness across all applicable channels and locales. Response structure:

```json
{
  "data": {
    "identifier": "42_1_1",
    "channels": [
      {
        "channelCode": "ecommerce",
        "locales": [
          {
            "localeCode": "en_US",
            "percentage": 85,
            "complete": 6,
            "required": 7,
            "localeResetApplied": false,
            "missingAttributes": [
              {
                "attributeId": 42,
                "attributeCode": "longDescription",
                "labels": { "en_US": "Long Description" },
                "type": "textarea",
                "groupId": 3
              }
            ]
          },
          {
            "localeCode": "fr_FR",
            "percentage": 43,
            "complete": 3,
            "required": 7,
            "localeResetApplied": true,
            "missingAttributes": [...]
          }
        ]
      }
    ]
  }
}
```

### `GET /api/v1/products/:identifier/completeness/:channelCode/:localeCode`

Returns a single `CompletenessResult` for the exact channel+locale. Used by the product edit form to update the completeness indicator when the user switches channel/locale scope.

### `GET /api/v1/products/:identifier/completeness/aggregate`

Returns `ModelCompletenessAggregate` for a root model. Returns 400 if called on a variant or simple product. Used by the Phase 30 completeness dashboard.

### `GET /api/v1/products?includeCompleteness=true`

Extends the existing product list endpoint to include a compact `completeness` summary (`{ [channelCode]: { [localeCode]: percentage } }`) for each product. Uses `calculateCompletenessOverview` for the batch. Default: off (omit completeness unless requested, to keep the list response fast).

---

## 5. Deep-Link Support for Missing Attributes

The `missingAttributes` list in each completeness result must include enough information for the Phase 30 UI to deep-link to the exact field in the product edit form. The consumer needs:

- The attribute code (to identify which form field)
- The channel and locale for which it is missing (to switch the edit form scope)
- The attribute group (to know which section of the form)

The `CompletenessResult` already carries `channelCode` and `localeCode` at the top level. Each `missingAttribute` carries `attributeCode` and `groupId`. The Phase 30 UI constructs the deep link as:

```
/products/{identifier}?channel={channelCode}&locale={localeCode}#group-{groupId}
```

This requires the product edit form (Phase 12.1.1) to support URL-persisted channel/locale and group-anchor scrolling. Verify this is already in place from Phase 12.1.1; document if not.

---

## 6. Batch Optimization (Performance)

The `calculateCompletenessOverview` function for the product list must avoid N+1 queries. Verified implementation approach:

1. Collect all unique family IDs from the product list.
2. Load all family requirements for those families in one query.
3. Load all attribute values for all product identifiers in one query.
4. Load all channels and locales in one query (likely already cached).
5. Compute completeness in application code.

For a product list of 25 products spanning 5 families, this should result in at most 4 DB queries regardless of product count.

---

## 7. Verification Checklist

These items may already be correctly implemented from Phase 11 / Phase 12.1. Verify each:

- [ ] `calculateCompleteness` applies locale reset when `completeness_resets_after_fallback = true` on the locale.
- [ ] `calculateAllCompleteness` iterates all applicable channels (from category assignments) and their active locales, using the locale reset rule where configured.
- [ ] `calculateCompletenessOverview` executes at most O(N) DB queries (not per-product), where N is a small constant (families, values, channels, locales).
- [ ] `calculateModelCompleteness` recurses through all leaf variants (not just direct children) — uses the recursive descendant logic from Phase 12.1.1.
- [ ] Products with no category assignments return empty completeness arrays (no divide-by-zero or error).
- [ ] The `missingAttributes` list correctly identifies attributes that are present via locale fallback but missing in the requested locale when `localeResetApplied = true`.
- [ ] The `GET /api/v1/products/:identifier/completeness` endpoint returns the correct structure.
- [ ] The `GET /api/v1/products/:identifier/completeness/:channelCode/:localeCode` single-scope endpoint works.
- [ ] The `GET /api/v1/products/:identifier/completeness/aggregate` endpoint works for root models.

---

## 8. Storage Methods

All completeness methods live in the service layer, not directly in `DatabaseStorage`. They call storage methods for raw data fetching and compute in application code.

Methods that `IStorage` / `DatabaseStorage` must expose for completeness calculation (add if missing):

| Method | Description |
|--------|-------------|
| `getFamilyRequirementsForChannel(familyId, channelId)` | Requirements for a family × channel combination |
| `getAllFamilyRequirements(familyId)` | All requirements for a family (all channels) |
| `getFamilyRequirementsBatch(familyIds)` | Batch fetch for multiple families |
| `getProductAttributeValuesMerged(identifier, channelCode, localeCode)` | Resolved values with full inheritance + locale fallback |
| `getProductAttributeValuesBatch(identifiers)` | All raw values for a list of identifiers (for batch completeness) |
| `getProductChannels(identifier)` | Channels applicable to this product via category assignments |
| `getDescendantIdentifiers(identifier)` | All descendant identifiers (recursive); used for model aggregate |

---

## 9. Exit Criteria

- [ ] All 9 canonical rules in §1 are correctly implemented and verified with tests.
- [ ] Locale completeness reset works: a locale with `completeness_resets_after_fallback = true` does not count inherited locale values as complete.
- [ ] `calculateAllCompleteness` and `calculateCompletenessOverview` batch functions are efficient (no N+1 queries).
- [ ] `calculateModelCompleteness` returns correct aggregate across all leaf variants (recursive descendants).
- [ ] REST endpoints for single product, single scope, and model aggregate are functional and return the specified response structure.
- [ ] `missingAttributes` includes sufficient data for the Phase 30 UI to construct a deep link to the incomplete field.
- [ ] Completeness is accurate against the real Akeneo-imported dataset from Phase 26 (spot-check 5–10 products with known completeness state).
- [ ] `GET /api/v1/products?includeCompleteness=true` returns compact completeness summaries using the batch function.

---

## 10. Notes

- **No stored completeness.** If future phases add a completeness cache (e.g. a `product_completeness_cache` table for faster filter queries), it is additive. The on-demand calculation functions remain the source of truth and the cache is kept warm by the write hooks.
- **Standalone products.** A standalone product (type: `product`) is its own "leaf" — there are no variants to aggregate. `calculateModelCompleteness` called on a standalone product returns a `variantCount` of 1 (the product itself) and the product's own completeness scores.
- **Phase 30 dependency.** The completeness UI (Phase 30) depends on this phase's `calculateModelCompleteness` aggregate and the deep-link-ready `missingAttributes` shape. Ensure those are stable before building the UI.
- **Phase 28 dependency.** The search completeness filter (Phase 28 §8) uses `calculateCompletenessOverview` in application code. The batch function's efficiency directly affects search response time when the `completeness` filter is applied.
