# Phase 34: UI: Scheduling Management

**Execution Plan Item:** #34  
**Replit Phase:** 34 (Week 9)  
**Dependencies:** Phase 24 (Integration Management UI), Phase 32 (Scheduling Infrastructure), Phase 33 (Scheduled Integration Runs)  
**Depended on by:** Nothing in MVP.

---

## Objective

Build the scheduling management UI so users can **create, view, edit, and cancel** scheduled actions — and configure recurring schedules on integration instances — without touching the API directly. The UI should make the common case fast: scheduling a product status change or viewing what's coming up should take seconds. The full timeline and history views are secondary.

---

## 1. New Pages & Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/scheduled-actions` | `ScheduledActionsPage` | List view + timeline of all scheduled actions |
| `/scheduled-actions/new` | `ScheduleActionPage` | Create a new one-off scheduled action |
| `/scheduled-actions/:id` | `ScheduledActionDetailPage` | Detail view, edit, and cancel |

Add **"Scheduled Actions"** to the main navigation sidebar, grouped with or near Integrations.

---

## 2. Scheduled Actions List (`/scheduled-actions`)

### Layout

Two tabs:

**Upcoming tab** — Default view. Shows `pending` and `executing` actions sorted by `scheduled_at ASC`. Focused on what's about to happen.

**History tab** — Shows `completed`, `failed`, and `cancelled` actions sorted by `scheduled_at DESC` (most recently executed first). Paginated.

### Table Columns

| Column | Description |
|--------|-------------|
| Scheduled At | Date + time, formatted relative (e.g. "in 2 days") with absolute on hover |
| Entity | Linked entity — product identifier (links to `/products/:identifier`) or integration instance name (links to instance) |
| Action | Human-readable summary of the action (see §2.1 below) |
| Status | Badge: `pending` (neutral), `executing` (blue), `completed` (green), `failed` (red), `cancelled` (muted) |
| Created By | User name or "System" for scheduler-created actions |
| Actions | Edit / Cancel buttons for `pending` actions; retry button for `failed` actions |

### Action Summary Formatting

The raw `action_type` and `payload` should be rendered as a readable sentence:

| action_type | Example summary |
|-------------|----------------|
| `product.set_status` | **Set status** to Active |
| `product.set_values` | **Set 3 attribute values** |
| `product.assign_category` | **Add to category** Hand Tools |
| `integration.trigger_run` | **Run** Akeneo Nightly Import (full) |

### Filters

- Status (multi-select: pending, completed, failed, cancelled)
- Entity type (product / integration)
- Date range (scheduled_at from/to)

---

## 3. Timeline View

A visual timeline of upcoming `pending` actions displayed on the Upcoming tab, above the table. Renders the next 14 days with actions plotted by their `scheduled_at` date.

- Each day column shows a count badge if actions are scheduled that day.
- Clicking a day filters the table to that day's actions.
- Designed to give a quick "what's happening when" overview, not a full calendar — keep it lightweight.

The timeline is powered by `GET /api/v1/scheduled-actions/upcoming` (returns up to 200 actions in a time window, no pagination required).

---

## 4. Create Scheduled Action (`/scheduled-actions/new`)

A focused form for creating a one-off scheduled action. Support the four action types from Phase 32.

### Step 1 — Choose Entity & Action

**Entity type selector** (radio or segmented control):
- Product
- Integration Instance

**Entity picker:**
- For Product: search-as-you-type using `GET /api/v1/products/search?q=` — shows `displayName`, identifier, and status. Required.
- For Integration Instance: dropdown of all integration instances. Required.

**Action type selector** (shown after entity is chosen, filtered to relevant types):

| Entity Type | Available Actions |
|------------|-------------------|
| Product | Set Status, Set Attribute Values, Assign/Remove Category |
| Integration | Trigger Run |

### Step 2 — Configure Payload

Renders a context-specific form based on the chosen action type:

**Set Status:**
- Status select: Draft / Active / Archived
- Optional: "Also trigger integration runs after" — multi-select of integration instances

**Set Attribute Values:**
- Add-value rows: attribute picker (searchable by label), channel selector, locale selector, value input (renders the correct input for the attribute type — same components used in the product edit form)
- Supports adding multiple values in one action

**Assign/Remove Category:**
- Operation: Add / Remove (radio)
- Category picker: searchable tree or flat search of categories

**Trigger Run:**
- Run type: Full / Incremental (disabled if connector doesn't support incremental)

### Step 3 — Set Time

- Date and time picker with timezone display (UTC for MVP)
- Validation: must be at least 30 seconds in the future
- Quick presets: "Tomorrow at 9am", "Monday at 8am", "Next month, 1st at 6am"

### Step 4 — Review & Confirm

Summary of what will happen and when, before submitting. Clear action summary (using same formatting as the list view).

---

## 5. Scheduled Action Detail (`/scheduled-actions/:id`)

Displays the full action details and allows editing or cancelling pending actions.

### Sections

**Summary card:**
- Status badge, entity, action type, scheduled time, created by, created at
- Execution details (executed at, completed at, error message) when status is completed/failed

**Payload:**
- Human-readable description of the full payload (not raw JSON for normal users; raw JSON toggle for debugging)

**Actions (pending only):**
- **Edit** — opens an inline form to change `scheduledAt` or the payload
- **Cancel** — confirmation dialog before cancelling

**Retry (failed only):**
- **Retry Now** — clones the action with `scheduledAt = now + 30s` and `retryCount = 0`
- **Retry Later** — opens the time picker to reschedule

---

## 6. Integration Instance Scheduling (Phase 33 UI)

Extend the existing integration instance edit form (Phase 24 UI) with a **Schedule** section. This is not a new page — it's an additional section in the existing instance edit form at `/integrations/:id/edit` (or equivalent).

### Schedule Section

**Schedule toggle** — "Enable recurring schedule" (on/off switch). When off, the rest of the section is greyed out.

**Cron expression input:**
- Free-text input accepting a 5-field cron expression (e.g. `0 2 * * *`)
- Live validation: show "Invalid expression" for syntactically incorrect input
- Live human-readable preview below the input: *"Every day at 2:00 AM (UTC)"* — rendered from the `scheduleCronHuman` field returned by the API

**Run type selector:** Full / Incremental (greyed out if the connector doesn't support incremental)

**Next scheduled run** — read-only display: *"Next run: Tuesday, March 11 at 2:00 AM UTC"* — pulled from `nextScheduledRunAt` on the instance.

**Quick presets** (below the cron input):
- Every day at 2am → `0 2 * * *`
- Every Monday at 6am → `0 6 * * 1`
- Every hour → `0 * * * *`
- Weekdays at 8am → `0 8 * * 1-5`

Clicking a preset populates the cron input and triggers the live preview update.

---

## 7. Product Edit Form Integration

Add a **"Schedule a Change"** shortcut to the product edit page, accessible from the action menu (the `...` or kebab menu near the save button). Clicking it navigates to `/scheduled-actions/new` pre-populated with the current product identifier, so the user doesn't need to search for it again.

This is a lightweight integration — no embedded scheduler UI on the product page itself. The full scheduling flow stays on the dedicated scheduling page.

---

## 8. Exit Criteria

- [ ] `/scheduled-actions` lists pending and historical actions with correct status badges and action summaries.
- [ ] Timeline shows upcoming actions grouped by day; clicking a day filters the table.
- [ ] Create form guides users through entity → action → payload → time in 4 clear steps.
- [ ] Product entity picker uses live search (`GET /api/v1/products/search`).
- [ ] Set Attribute Values action renders the correct attribute input type (text, number, select, etc.) using the shared attribute input components.
- [ ] Time picker validates that the scheduled time is in the future.
- [ ] Detail page shows full action info; Edit and Cancel are available for pending actions.
- [ ] Cancel shows a confirmation dialog before sending the DELETE request.
- [ ] Retry clones the failed action and redirects to the new pending action's detail page.
- [ ] Integration instance edit form includes schedule section with cron input, live human-readable preview, and quick presets.
- [ ] Enabling/disabling the schedule saves correctly and reflects the updated `nextScheduledRunAt`.
- [ ] "Schedule a Change" shortcut on the product edit page pre-populates the create form with the correct product.
- [ ] All scheduling UI actions require appropriate permissions (`product:write` for product actions, `integration:manage` for integration actions).

---

## 9. Notes

- **Cron help text.** Many users won't know cron syntax. The quick presets cover 80% of use cases. For the remaining 20%, a link to crontab.guru opens in a new tab.
- **UTC only for MVP.** All times are displayed and entered in UTC. Add timezone support in a future phase if users request it. Make UTC explicit everywhere (label all times with "UTC").
- **Attribute input components reuse.** The "Set Attribute Values" payload editor reuses the same attribute input components from the product edit form. This is the most complex part of Phase 34 — budget implementation time accordingly.
- **Empty states.** When there are no scheduled actions, the list shows a helpful empty state: "No scheduled actions yet. Schedule a product status change or configure an integration to run on a recurring schedule."
- **Error handling.** If an action fails (status `failed`), show the `executionError` message in the detail page with appropriate context ("The action failed with the following error:"). Don't surface raw stack traces.
