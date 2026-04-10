# Phase 24: UI — Integration Management

**Execution Plan Item:** #24  
**Replit Phase:** 24 (Week 5)  
**Dependencies:** Phase 21 (Integration types & instances API), Phase 22 (Field mappings & product filter API), Phase 23 (Run engine & run history API), Phase 19 (RBAC), Phase 17 (API consistency), existing app shell and patterns (Phase 07, 20)  
**Depended on by:** Phase 25 (Akeneo import — users create instance and trigger runs from this UI), Phase 33 (Scheduling UI is separate; this phase is integration-only)

---

## Objective

Deliver the **Integration Management UI**: (1) list integration types and instances and create/edit integration instances with connection config; (2) build and edit **field mappings** per instance; (3) for **outbound** instances, configure **product filters**; (4) **trigger on-demand runs** and view **run history** with drill-down into **per-record logs**. No scheduling UI in this phase — that is Phase 33. The UI should feel consistent with the rest of the app (navigation, forms, tables, permissions).

---

## 1. Navigation and Entry Points

- **Nav item:** Add "Integrations" (or "Integrations" under Settings/Admin) to the main navigation. Route: `/integrations` (or `/admin/integrations` if admin routes are namespaced).
- **Landing:** `/integrations` shows a list of **integration instances** (not types). Optionally a secondary view or tab for "Integration types" (read-only) to see available connector types and their direction. Primary focus: manage instances and runs.
- **Permissions:** Show Integrations nav and pages only if user has `integration:read` or `integration:manage`. Hide or disable create/edit/run actions if user has read-only.

---

## 2. Integration Types (Read-Only)

- **List:** Page or section "Available connector types" — table or cards: type name, description, direction (badge: Inbound / Outbound), "Full run" / "Single record" capabilities. Data from GET `/api/v1/integration-types`. Optional filter by direction.
- **Detail (optional):** Clicking a type can show connection_schema (field list) and description so admins know what config an instance will need. No edit; types are seeded.

---

## 3. Integration Instances — List

- **Route:** `/integrations` or `/integrations/instances`.
- **Table:** Columns: Name, Type (e.g. "Akeneo Import"), Direction (Inbound/Outbound), Status (Active/Inactive), Last run (timestamp or "Never"), Actions (Edit, View runs, Delete). Use GET `/api/v1/integration-instances` with optional `?direction=inbound`.
- **Actions:** "Add instance" button → go to create flow. Row actions: Edit, "View runs" (go to run history for this instance), Delete (with confirmation; warn if runs exist).
- **Empty state:** If no instances, show message and "Create your first integration instance" CTA.

---

## 4. Integration Instance — Create / Edit

- **Route:** `/integrations/instances/new`, `/integrations/instances/:id/edit`.
- **Create flow:**
  1. Select integration type (dropdown or cards from GET `/api/v1/integration-types`). Filter by direction if needed.
  2. Form: Name (required), Description (optional), **Connection config** — render fields from the type's `connection_schema`. For each key in connection_schema: label, input type (text, password for secret fields), required indicator. Do not display or store raw secrets in client state longer than needed; mask on load if editing.
  3. Submit → POST `/api/v1/integration-instances`. On success, redirect to instance detail or to mappings.
- **Edit:** Same form; load instance with GET `/api/v1/integration-instances/:id`. PATCH on save. Optionally "Test connection" button that calls a dedicated endpoint (can be stubbed in MVP).
- **Validation:** Client-side: required fields, format. Server returns 400 with details; display field-level errors under inputs.
- **Secrets:** For fields marked `secret: true` in connection_schema, use password input; on edit, show placeholder (e.g. "••••••••") and allow "Leave blank to keep current" so users can update other config without re-entering secret.

---

## 5. Field Mappings

- **Route:** `/integrations/instances/:id/mappings` or tab "Mappings" on instance detail.
- **Data:** GET `/api/v1/integration-instances/:id/field-mappings`. Instance type gives direction (inbound vs outbound).
- **Outbound:** Two columns (or two dropdowns per row): **Source** = PIM attribute (list from attributes API) or product field (e.g. identifier, status, familyId). **Target** = free text (target system field name). Optional: transform selector (none, default value, concat, unit, etc.) and transform config (e.g. default value string). "Add mapping" adds a row; "Remove" per row. Save → PUT `/api/v1/integration-instances/:id/field-mappings` with full list. Sort order = row order.
- **Inbound:** Same idea: **Source** = free text (source system field name; Phase 25 may provide a list for Akeneo). **Target** = PIM attribute code (dropdown from attributes API) or product/entity field. Transform optional. Save same PUT.
- **Empty state:** "No mappings yet. Add mappings to define how data is transformed between systems."
- **Validation:** Duplicate target_key per instance can be warned or blocked. Required source/target in each row.

---

## 6. Product Filter (Outbound Only)

- **Route:** `/integrations/instances/:id/filter` or tab "Product filter" on instance detail. **Show this section only when instance type is outbound.**
- **Data:** GET `/api/v1/integration-instances/:id/product-filter`. If 404 or null, show "No filter — all products included" and allow creating one.
- **Form:** Family (multi-select from families API), Category (multi-select from categories API), Channel (single select), Status (multi-select: Draft, Active, Archived). All conditions ANDed. "Clear filter" → DELETE product-filter. Save → PUT product-filter.
- **Help text:** "Only products matching all selected criteria will be included in pushes. Leave all empty to include all products."

---

## 7. Trigger Run

- **Location:** Instance detail page or run history page for that instance.
- **Actions:** "Run full sync" (or "Run full import" for inbound) → POST `/api/v1/integration-instances/:id/runs` with `{ runType: 'on_demand_full' }`. For outbound types that support single-record: "Push single product" → open modal/drawer to pick product (search by identifier or name), then POST with `{ runType: 'on_demand_single', recordIdentifier: '...' }`. Show these buttons only if type supports the run type (from integration_types.supports_full_run, supports_single_record).
- **Feedback:** On trigger, show loading state. If sync execution: on response, show success toast and run status (e.g. "Run completed: 150 success, 2 failed"). If async (202): "Run started" toast and link to run detail; user can navigate to run to see progress.
- **Errors:** 400/409 (e.g. invalid runType, run already in progress) → show message. 403 → "You don't have permission to run this integration."

---

## 8. Run History

- **Route:** `/integrations/instances/:id/runs` or tab "Run history" on instance detail.
- **List:** GET `/api/v1/integration-instances/:id/runs` with pagination (limit, cursor). Table: Run id (or date), Started, Finished, Status (badge: Pending, Running, Completed, Failed), Record counts (success / failure / skip), Triggered by, Actions (View details).
- **Status display:** Color or icon for status (e.g. green completed, red failed, blue running). "Running" can show a spinner or "In progress…".
- **Global run list (optional):** Route `/integrations/runs` — GET `/api/v1/integration-runs` with optional filter by instanceId. Same table columns plus "Instance" column. Useful for admin overview.

---

## 9. Run Detail and Per-Record Logs

- **Route:** `/integrations/runs/:id` (or `/integrations/instances/:instanceId/runs/:runId`).
- **Data:** GET `/api/v1/integration-runs/:id` for run summary (instance name, type, status, started_at, finished_at, record_count_success/failure/skip, error_message). GET `/api/v1/integration-runs/:id/logs` for per-record logs (paginated).
- **Summary block:** Run type, status, started/finished times, counts, error message (if failed). "Triggered by" (user or API key).
- **Logs table:** Columns: Record identifier, Status (success/failure/skip), Message, Details (expandable if details JSON present). Sort by sort_order or id. Pagination: "Load more" or page size selector. Filter by status (success/failure/skip) optional.
- **Export (optional):** "Export logs as CSV" for current page or full run — defer to later if time-constrained.
- **Empty state:** If run has no logs yet (e.g. run just started), show "No record logs yet. Run in progress."

---

## 10. RBAC and Permissions

- **integration:read:** Can view integration types, instances, runs, and logs. Cannot create/edit instances, edit mappings/filter, or trigger runs.
- **integration:manage:** Full access: create/edit/delete instances, edit mappings and filter, trigger runs, view all runs and logs.
- **UI:** Hide "Add instance", "Edit", "Delete", "Run", "Save mappings", "Save filter" when user lacks manage. Show 403 toast or inline message if they attempt action (e.g. from another tab). Run history and logs are visible to read users.
- **Consistency:** Reuse existing permission checks and 403 handling from Phase 20.

---

## 11. Consistency and UX

- **Design system:** Use existing components (tables, forms, buttons, badges, modals, toasts). Match spacing, typography, and colors from Phase 20.5 (brand, dark mode).
- **Loading states:** Skeleton or spinner for list and detail loads. Disable "Run" button while a run is in progress for that instance if one-run-at-a-time is enforced.
- **Errors:** Network errors or 500 → generic toast. Validation errors → inline under fields. 404 → "Integration not found" and link back to list.
- **Empty states:** Clear copy for no instances, no mappings, no runs, no logs.

---

## 12. Exit Criteria

- [x] Users with integration:read can open Integrations and view types, instances, runs, and run logs.
- [x] Users with integration:manage can create and edit integration instances; connection config form is driven by type's connection_schema; secrets are masked on edit.
- [x] Users can add, edit, and remove field mappings per instance (outbound and inbound); save persists via PUT field-mappings.
- [x] For outbound instances, users can set and clear product filter (family, category, channel, status); save persists via PUT product-filter.
- [x] Users can trigger on-demand full run (and single-record run where type supports it); run appears in run history; run detail shows summary and per-record logs with pagination.
- [x] Navigation, permissions, loading and error states are consistent with the rest of the app. No scheduling UI in this phase.

---

## 13. Notes

- **Phase 25 (Akeneo):** When Akeneo import is implemented, the same UI is used: create an instance of type "Akeneo Import", enter base URL and API key in connection config, optionally set mappings and import scope (in config or future scope UI), trigger "Run full import", view run and logs here.
- **Scheduling:** Phase 33 will add scheduling UI (create scheduled run, timeline of upcoming runs). This phase does not include scheduling; only on-demand trigger and run history.
- **Connection test:** A "Test connection" button that calls a type-specific endpoint can be added in a later phase; MVP can omit or stub (e.g. return success if config has required fields).
