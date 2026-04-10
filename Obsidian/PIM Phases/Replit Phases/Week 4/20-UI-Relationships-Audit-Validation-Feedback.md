# Phase 20: UI — Relationships, Audit & Validation Feedback

**Execution Plan Item:** #20  
**Replit Phase:** 20 (Week 4)  
**Dependencies:** Phase 15 (Product Associations), Phase 16 (Audit History wiring), Phase 17 (API consistency), Phase 18 (Attribute validation enforcement), Phase 19 (RBAC), Product UI (Phase 12)  
**Depended on by:** User workflows for related products, compliance viewing, form UX

---

## Objective

Deliver the **Week 4 UI surface**: (1) **Relationship management** — create and view product associations with clear directionality (one-way vs two-way); (2) **Audit history viewer** — change timeline per entity; (3) **Inline validation feedback** — users see what's wrong and why before submit, aligned with Phase 18 API validation errors.

---

## 1. Relationship Management UI

### Association Types (Admin)

- **List page:** `/association-types` — List all association types with labels, two-way indicator, optional family scope, sort order. "+ Create" opens create dialog.
- **Create/Edit:** Modal or inline form: labels (LocaleLabelsEditor), is two-way (checkbox), optional family (dropdown), sort order. No code; id is auto. Save calls POST/PATCH `/api/v1/association-types`.
- **Delete:** With confirmation; warn if associations exist (cascade delete or block and show count).

### Product Associations (on Product)

- **Location:** Product edit page — new tab or section "Associations" (or "Relationships").
- **List:** For current product (identifier), show associations grouped by type. Each row: type label, target (product identifier/label or entity label), direction indicator (one-way arrow or two-way ↔), sort order, remove button.
- **Add:** "Add association" — choose association type (dropdown), then target type (product vs entity). If product: search/select product by identifier or label. If entity: search/select entity by label. Optional sort order. POST `/api/v1/products/:identifier/associations`.
- **Directionality:** For one-way types, show "Source → Target" so it's clear this product is the source. For two-way, show "↔" and list the link once (or show both sides if API returns symmetric).
- **Reorder:** Optional up/down or drag to set sort order; PATCH association with new sortOrder.

### Permissions

- If user lacks create/update/delete on productAssociation or associationType, hide or disable the corresponding buttons; show 403 message if they attempt via API (e.g. disabled button with tooltip "No permission").

---

## 2. Audit History Viewer

- **Entry point:** From entity detail pages — e.g. "History" tab on product edit, entity edit, asset edit; or "View history" link in sidebar/header for the current resource.
- **Route:** `/audit-history` (global list) and/or per-resource: e.g. product edit page tab "History" that shows only that product's history.
- **Global list:** Filters: resource type, resource id, user, action, date range. Uses GET `/api/v1/audit-history` with query params. Table: timestamp, user, action, resource type, resource id, attribute (if applicable), old value → new value (truncated or expandable). Cursor pagination.
- **Per-resource:** GET `/api/v1/audit-history/resource/:resourceType/:resourceId`. Timeline or table: newest first. Show user name, action, changed fields (old → new), attribute and scope if applicable. Optional: expand JSON diff for complex values.
- **Display:** Human-readable labels for resource types and actions. User name from API. Format dates with date-fns. Truncate long values with "Show more."

---

## 3. Inline Validation Feedback

- **When:** On blur and/or on submit for all forms that write attribute values or entity data (product edit attributes, entity edit, asset edit, create dialogs).
- **Source of truth:** Phase 18 API returns 400 with `error.details = [{ field, attributeId, message }]`. Map these to the correct form fields.
- **Display:** Inline error message below the field (or under the attribute row in product edit). Use existing form library (e.g. react-hook-form) error state so the field is highlighted and the message is visible.
- **Before submit:** Where feasible, run client-side validation (same rules as server: regex, min/max length, required) so users get immediate feedback without a round trip. Server remains the authority; if server returns validation errors, display them and optionally focus the first invalid field.
- **Consistency:** Use the same error component and message style across attribute groups, product create, entity create, asset create, and association forms.

---

## 4. RBAC and UI

- **403 handling:** On any API call, if response is 403, show a toast or inline message: "You don't have permission to perform this action." Do not refresh entire page; allow user to continue viewing if they have read access.
- **Hiding actions:** If backend exposes permissions (e.g. GET /api/v1/me/permissions), hide Create/Edit/Delete buttons for resources the user cannot act on. Otherwise, show buttons and rely on 403 after click (with clear message).

---

## 5. Exit Criteria

- [ ] Users can create and manage association types (list, create, edit, delete).
- [ ] Users can add, view, reorder, and remove product associations from the product edit page; directionality (one-way vs two-way) is clear.
- [ ] Audit history is viewable per resource (and optionally globally); timeline shows user, action, and changes in a readable format.
- [ ] All relevant forms show inline validation errors from the API (and optionally client-side) before and after submit; messages are clear and field-specific.
- [ ] 403 responses are handled gracefully; actions that the user cannot perform are disabled or hidden when permission info is available.

---

## 6. Notes

- **Performance:** Audit history for a single resource should be paginated; avoid loading thousands of rows at once.
- **Localization:** Validation messages and audit labels can be localized in a later pass; MVP can use server-returned or client messages in a single language.
