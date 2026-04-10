# Phase 16: Audit History Wiring — Completion Pass

**Execution Plan Item:** #16  
**Replit Phase:** 16 (Week 4)  
**Dependencies:** Audit History schema (Phase 06), all entities from Weeks 1–3 (attributes, families, products, entities, assets, categories, channels, etc.)  
**Depended on by:** Phase 20 (Audit history viewer UI), compliance and support workflows

---

## Objective

Verify **full audit coverage** across every create/update/delete path in the system. Audit logging was wired incrementally in earlier phases; this phase is a **completion pass**: audit every write path, close gaps, and align resource types and field names with the current schema (e.g. numeric ids post Phase 12.9).

---

## 1. Scope

### Entities and Write Paths to Verify

| Area | Resource Types | Write Paths |
|------|----------------|-------------|
| Attributes | `attribute`, `attributeGroup`, `attributeOption` | Create/update/delete attribute, group, option |
| Locales / Channels | `locale`, `channel`, `currency` | Create/update/delete |
| Families | `family`, `familyAttribute`, `familyRequirement` | Family CRUD; add/remove attributes; set requirements |
| Variant definitions | `familyVariantDefinition`, `familyVariantAxes`, `familyVariantAttributeLevels` | Definition CRUD; PUT axes; PUT attribute-levels |
| Categories | `categoryTree`, `category` | Tree CRUD; category CRUD and move |
| Products | `product` | Create, PATCH (status, sortOrder), delete, reorder children |
| Product values | `productValue` | PUT values, DELETE value (reset override) |
| Product categories | `productCategory` | Assign/unassign categories to product |
| Entity types | `entityType`, `entityTypeAttribute` | Entity type CRUD; add/remove attributes |
| Entities | `entity` | Entity CRUD |
| Entity values | `entityValue` | PUT entity values |
| Asset types | `assetType`, `assetTypeAttribute` | Asset type CRUD; add/remove attributes |
| Assets | `asset` | Asset CRUD |
| Asset values | `assetValue` | PUT asset values |
| Association types | `associationType` | CRUD (Phase 15) |
| Product associations | `productAssociation` | Add/remove associations (Phase 15) |

### Schema Alignment (Post Phase 12.9)

- `resource_id` stores the entity's primary key (numeric id as string for most structure entities, product identifier for products).
- Where the audit table still has `attribute_code`, `channel_code`, etc., align with current schema: use `attribute_id`, `channel_id` if the codebase was migrated in 12.9. If audit_history was updated in 12.9, no change here; otherwise update column names and log calls to use ids.

---

## 2. Wiring Checklist

For each write path:

1. **Entity-level create:** Log once with `action: 'create'`, `resourceType`, `resourceId`, `newValue` (full record or key fields). No `attributeId`/`channelId`/`localeCode` unless it's an attribute-specific resource.
2. **Entity-level update:** Log with `action: 'update'`, `oldValue` and `newValue` (changed fields only or full before/after). Optionally one log per changed field for granularity.
3. **Entity-level delete:** Log with `action: 'delete'`, `resourceType`, `resourceId`, `oldValue` (full record or key fields). No `newValue`.
4. **Attribute-value writes (products/entities/assets):** Use existing pattern: `resourceType` = productValue/entityValue/assetValue, `resourceId` = product identifier or entity/asset id, `attributeId`, `channelId`, `localeCode` as applicable, `oldValue`/`newValue` with the value payload.
5. **Batch operations:** Either one audit row per logical change (e.g. per value in PUT values) or one row per batch with `new_value` containing the full set; prefer per-change for queryability.

---

## 3. API and Query Behavior

- **GET /api/v1/audit-history** — Ensure filters support `resourceType`, `resourceId`, `userId`, `action`, `attributeId` (or attribute_code if not migrated), `channelId`, `from`/`to` time range, cursor pagination. Response includes user display name (join to pim_users).
- **GET /api/v1/audit-history/resource/:resourceType/:resourceId** — Returns timeline for one resource, newest first. Used by Audit History viewer UI (Phase 20).

---

## 4. Business Rules

- Audit writes must not block the main operation: wrap in try/catch; on failure log to app logs and continue.
- Append-only: no updates or deletes to audit_history.
- System/migration actions: `user_id` = null, `context` = { source: 'migration' } or similar.

---

## 5. Exit Criteria

- [ ] Every create/update/delete path listed above has at least one audit log call.
- [ ] Resource types and resource ids are consistent (ids where schema uses ids).
- [ ] Product/entity/asset value writes are logged with attribute/channel/locale scope.
- [ ] GET audit-history and GET by resource work and return expected data.
- [ ] No regression: existing flows still succeed if audit logging fails.

---

## 6. Notes

- **Performance:** High-volume value writes (e.g. PUT 500 values) may generate many audit rows. Consider batching inserts or sampling if needed; document the choice.
- **Sensitive data:** Avoid logging secrets or PII in old_value/new_value; redact if necessary.
