# Phase 18: Attribute Validation Enforcement

**Execution Plan Item:** #18  
**Replit Phase:** 18 (Week 4)  
**Dependencies:** Attributes (Phase 02) — validation_rules and settings schema; Product/Entity/Asset value write paths (Phases 09, 12.2, 12.3)  
**Depended on by:** Data quality, Phase 20 (inline validation feedback in UI)

---

## Objective

**Enforce** the attribute validation rules defined in Phase 02 at every API write path that sets attribute values. Until now, validation rules were stored as metadata only; this phase ensures regex, min/max length, min/max value, unique constraints, and file-type/size restrictions are applied so bad data is rejected at the API boundary.

---

## 1. Validation Rules Schema (Recap)

From Task 2, `validation_rules` JSONB on attributes can include:

| Rule | Applies To | Behavior |
|------|------------|----------|
| `regex` | text, textarea, singleSelect (stored value) | Value must match pattern. |
| `regexMessage` | — | Custom error message when regex fails. |
| `minLength` / `maxLength` | text, textarea | Character length bounds. |
| `minValue` / `maxValue` | number, measurement (amount) | Numeric bounds. |
| `allowedExtensions` | mediaFile | File extension must be in list. |
| `maxFileSizeMb` | mediaFile | File size must not exceed. |

`settings` JSONB can also impose type-specific constraints (e.g. `maxLength` for text, `decimalPlaces` for number). Validation should consider both `validation_rules` and `settings`.

---

## 2. Where to Enforce

| Write Path | Scope |
|------------|--------|
| PUT /api/v1/products/:identifier/values | Each value in the payload: resolve attribute by id, load validation_rules + settings, validate by attribute type. |
| PUT /api/v1/entities/:id/values | Same as above. |
| PUT /api/v1/assets/:id/values | Same as above. |
| Any single-value set (e.g. PATCH that sets one attribute) | Same validation before write. |

Also enforce on **create** flows that set initial values (e.g. product create with default attribute values if supported).

---

## 3. Validation by Attribute Type

- **text / textarea:** minLength, maxLength (from validation_rules or settings), regex if present.
- **number:** minValue, maxValue; decimals from settings if applicable.
- **measurement:** minValue/maxValue on amount; unit in allowed list if defined in settings.
- **singleSelect / multiSelect:** Value(s) must be valid option id(s) for that attribute (already enforced if options are validated). Optionally regex on stored value if used for free text.
- **boolean:** Accept only true/false.
- **date:** Valid ISO date/datetime.
- **price:** Structure and currency validation per settings.
- **mediaFile:** allowedExtensions, maxFileSizeMb; validate on upload or when path is set.
- **assetSingle / assetList:** Referenced asset id(s) must exist and optionally match asset type from settings.
- **entityReference:** Referenced entity id must exist and optionally match entity type from settings.
- **productReference:** Referenced product identifier must exist; optionally family filter from settings.
- **table:** Structure and column types per settings if defined.

Return **400** with `error.code = 'VALIDATION_ERROR'` and `error.details = [{ field: attributeCode/attributeId, message: "..." }]` for each failing attribute.

---

## 4. Unique Constraint

When attribute has `is_unique: true`:

- **Scope:** Uniqueness is per entity type (e.g. per family for products, per entity type for entities). Task 2: "Unique value enforcement spans all products in the same family (within the same scope)."
- **Behavior:** Before writing a value, check that no other entity of the same type (and same channel/locale scope if applicable) has the same value for this attribute. If duplicate, return 409 with a clear message.

---

## 5. Required Attributes

Required-ness is defined per family (and channel/locale) in `family_attribute_requirements`. For products:

- **Option A:** Enforce on PUT values: if the scope has a requirement for an attribute, reject the request if that attribute is missing or empty in the payload (or after applying the write). Return 400 with details.
- **Option B:** Do not block write; completeness calculation already flags missing required. UI shows incomplete; API allows save.

Recommendation: **Enforce on write** (Option A) so API and UI agree that "required" means "must be set before considering the scope complete." Document in this phase.

For entities, if entity_type_requirements exist, apply the same rule.

---

## 6. Error Response

Use the same error shape as Phase 17:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more attribute values failed validation.",
    "details": [
      { "field": "values[0]", "attributeId": 5, "message": "Value must match pattern: ^[A-Z0-9-]+$" }
    ]
  }
}
```

Include enough in `details` for the UI to highlight the right field (e.g. attributeId and channel/locale).

---

## 7. Exit Criteria

- [ ] All value write paths (product, entity, asset) run validation_rules and settings before persisting.
- [ ] Regex, minLength, maxLength, minValue, maxValue, allowedExtensions, maxFileSizeMb are enforced where applicable.
- [ ] is_unique is enforced within the correct scope (e.g. per family for products).
- [ ] Required attributes (per family/entity type and channel/locale) are enforced or explicitly documented as UI-only.
- [ ] Validation errors return 400 with consistent details array for UI display.

---

## 8. Notes

- **Performance:** Validation should be efficient (e.g. single query to load attribute rules per request or batched).
- **Localization:** regexMessage and other messages can be localized later; for MVP a single message per rule type is fine.
