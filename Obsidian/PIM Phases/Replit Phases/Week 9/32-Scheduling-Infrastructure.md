# Phase 32: Scheduling Infrastructure

**Execution Plan Item:** #32  
**Replit Phase:** 32 (Week 9)  
**Dependencies:** Phase 08 (Products), Phase 09 (Product Attribute Values), Phase 10 (Product Status), Phase 15 (Associations), Phase 19 (RBAC), Phase 23 (Integration Run Engine)  
**Depended on by:** Phase 33 (Scheduled Full Runs), Phase 34 (Scheduling UI)

---

## Objective

Build the **scheduling infrastructure** that allows any data mutation in EARL to be deferred to a future point in time. A user or API caller creates a scheduled action specifying *what* to do, *when* to do it, and *to which entity*. A background job processor runs continuously, picks up due actions, and executes them against the same write paths used for immediate changes — guaranteeing consistency between scheduled and live operations.

Supported action types at MVP: product status change, product attribute value writes, product category assignment changes, and integration run triggers. The data model is designed to accommodate additional action types without schema changes.

---

## 1. Data Model

### Table: `scheduled_actions`

```sql
CREATE TABLE scheduled_actions (
  id              SERIAL PRIMARY KEY,
  entity_type     VARCHAR(50)   NOT NULL,  -- 'product', 'integration_instance'
  entity_id       VARCHAR(255)  NOT NULL,  -- product identifier or integration instance id
  action_type     VARCHAR(80)   NOT NULL,  -- see §2 Action Types
  payload         JSONB         NOT NULL,  -- action-specific parameters (see §2)
  scheduled_at    TIMESTAMPTZ   NOT NULL,  -- when to execute
  status          VARCHAR(20)   NOT NULL DEFAULT 'pending',
  executed_at     TIMESTAMPTZ,             -- when execution started
  completed_at    TIMESTAMPTZ,             -- when execution finished
  execution_error TEXT,                    -- error message if failed
  retry_count     INTEGER       NOT NULL DEFAULT 0,
  max_retries     INTEGER       NOT NULL DEFAULT 3,
  created_by_id   INTEGER       REFERENCES users(id) ON DELETE SET NULL,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ   NOT NULL DEFAULT now()
);

CREATE INDEX idx_scheduled_actions_due
  ON scheduled_actions (scheduled_at, status)
  WHERE status = 'pending';

CREATE INDEX idx_scheduled_actions_entity
  ON scheduled_actions (entity_type, entity_id);
```

### Status Lifecycle

```
pending → executing → completed
                    → failed (retry_count < max_retries → back to pending)
                    → failed (retry_count = max_retries → stays failed)
pending → cancelled  (user-initiated before execution)
```

| Status | Meaning |
|--------|---------|
| `pending` | Waiting to be executed at `scheduled_at` |
| `executing` | Currently being processed by the background job |
| `completed` | Successfully executed |
| `failed` | Execution errored; retry count incremented |
| `cancelled` | Manually cancelled before execution |

---

## 2. Action Types

The `action_type` field is a namespaced string. The `payload` field contains action-specific parameters as a JSON object.

### `product.set_status`

Change a product's status.

```json
{
  "status": "active"
}
```

`status`: `"draft"` | `"active"` | `"archived"`

Executes the same write path as `PATCH /api/v1/products/:identifier` with `{ status }`.

---

### `product.set_values`

Write one or more attribute values to a product.

```json
{
  "values": [
    { "attributeId": 42, "channelId": null, "localeCode": "en_US", "value": "New product name" },
    { "attributeId": 55, "channelId": 2,    "localeCode": null,    "value": [{ "currency": "USD", "amount": 49.99 }] }
  ]
}
```

Executes the same write path as `PUT /api/v1/products/:identifier/values`. All values in the array are applied atomically.

---

### `product.assign_category`

Add or remove a product's category assignment.

```json
{
  "operation": "add",
  "categoryId": 12
}
```

`operation`: `"add"` | `"remove"`

Executes the same write path as `POST /api/v1/products/:identifier/categories` (add) or `DELETE /api/v1/products/:identifier/categories/:id` (remove).

---

### `integration.trigger_run`

Trigger a full run for an integration instance.

```json
{
  "runType": "full"
}
```

`runType`: `"full"` | `"incremental"` (incremental only supported for inbound connectors that support it)

Executes the same run-trigger logic as the existing on-demand run endpoint. Used directly in Phase 32 for one-off scheduled runs; used by Phase 33 for recurring scheduled runs.

---

### Post-execution Integration Push (Optional)

Any action type can include an optional `triggerIntegrationRunsAfter` array in its payload. After the primary action completes successfully, the processor enqueues a run for each listed integration instance ID:

```json
{
  "status": "active",
  "triggerIntegrationRunsAfter": [3, 7]
}
```

This enables workflows like: *"Activate this product at 9am on Friday AND push it to ecommerce immediately after."*

---

## 3. Background Job Processor

### Poll Loop

The processor runs on a configurable interval (default: **30 seconds**) as a background task started on server boot:

```typescript
async function startScheduledActionProcessor(): Promise<void> {
  setInterval(async () => {
    await processDueActions();
  }, SCHEDULER_POLL_INTERVAL_MS);
}
```

Start in `registerRoutes()` or server entry point — fire and forget, with top-level error catching.

### Claiming Actions (Concurrent Safety)

Use `SELECT FOR UPDATE SKIP LOCKED` to claim a batch of due actions without race conditions across multiple server instances or rapid poll cycles:

```sql
SELECT id FROM scheduled_actions
WHERE status = 'pending'
  AND scheduled_at <= now()
ORDER BY scheduled_at ASC
LIMIT 10
FOR UPDATE SKIP LOCKED
```

Immediately update claimed rows to `status = 'executing'` and `executed_at = now()` in the same transaction. This prevents double-execution if two server instances poll simultaneously.

### Execution

For each claimed action:

1. Resolve the entity (load the product by identifier, integration instance by ID, etc.). If not found, mark as `failed` with `"Entity not found"`.
2. Dispatch to the appropriate internal service function based on `action_type`. **Do not call the HTTP route handler** — call the service layer directly (e.g. `storage.updateProduct(...)`, `setProductValues(...)`, `triggerIntegrationRun(...)`).
3. If the action's payload includes `triggerIntegrationRunsAfter`, execute those runs after the primary action succeeds.
4. On success: mark `status = 'completed'`, set `completed_at`.
5. On error: increment `retry_count`. If `retry_count < max_retries`, set `status = 'pending'` and advance `scheduled_at` by an exponential backoff interval (30s, 2m, 10m). If `retry_count >= max_retries`, mark `status = 'failed'` and record `execution_error`.

### Batch Size and Timing

- Process up to 10 actions per poll cycle to avoid blocking the event loop.
- If 10 actions were claimed (batch full), immediately re-poll rather than waiting 30 seconds.
- Log each execution with `[Scheduler]` prefix: claimed action ID, action type, entity, outcome, and duration.

---

## 4. REST API Endpoints

All endpoints require authentication. Creating and cancelling scheduled actions requires `product:write` permission (for product actions) or `integration:manage` (for integration run actions). Listing and viewing requires corresponding read permissions.

### `POST /api/v1/scheduled-actions`

Create a scheduled action.

**Request body:**
```json
{
  "entityType": "product",
  "entityId": "830417",
  "actionType": "product.set_status",
  "payload": { "status": "active" },
  "scheduledAt": "2026-03-15T09:00:00Z"
}
```

**Validation:**
- `scheduledAt` must be in the future (at least 30 seconds from now).
- `entityType` + `entityId` must reference a valid, existing entity.
- `payload` must be valid for the given `actionType` (validated against per-type schemas).
- `actionType` must be a recognized type.

**Response:** `201` with the created `ScheduledAction` object.

---

### `GET /api/v1/scheduled-actions`

List scheduled actions with optional filters.

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `entityType` | string | Filter by entity type: `product`, `integration_instance` |
| `entityId` | string | Filter by specific entity |
| `status` | string | Comma-separated: `pending`, `executing`, `completed`, `failed`, `cancelled` |
| `actionType` | string | Filter by action type |
| `from` | ISO date | Filter by `scheduled_at >= from` |
| `to` | ISO date | Filter by `scheduled_at <= to` |
| `limit` | number | Page size, default 25, max 100 |
| `cursor` | string | Pagination cursor |

**Response:** `{ data: ScheduledAction[], meta: { total, limit, nextCursor } }`

---

### `GET /api/v1/scheduled-actions/:id`

Get a single scheduled action by ID.

**Response:** `{ data: ScheduledAction }`

The `ScheduledAction` response shape:
```json
{
  "id": 47,
  "entityType": "product",
  "entityId": "830417",
  "actionType": "product.set_status",
  "payload": { "status": "active" },
  "scheduledAt": "2026-03-15T09:00:00Z",
  "status": "pending",
  "executedAt": null,
  "completedAt": null,
  "executionError": null,
  "retryCount": 0,
  "maxRetries": 3,
  "createdById": 5,
  "createdAt": "2026-03-10T14:22:00Z",
  "updatedAt": "2026-03-10T14:22:00Z"
}
```

---

### `PATCH /api/v1/scheduled-actions/:id`

Update a pending scheduled action. Only `scheduledAt` and `payload` can be changed; only `pending` actions can be edited.

**Request body:**
```json
{
  "scheduledAt": "2026-03-15T10:00:00Z"
}
```

Returns `409 Conflict` if the action is not in `pending` status.

---

### `DELETE /api/v1/scheduled-actions/:id`

Cancel a scheduled action. Sets `status = 'cancelled'`. Only `pending` actions can be cancelled.

Returns `409 Conflict` if the action is not in `pending` status.

---

### `GET /api/v1/scheduled-actions/upcoming`

Returns all pending scheduled actions sorted by `scheduled_at ASC`, optionally limited to a time window. Designed for the UI timeline view. No pagination — returns up to 200 records.

**Query parameters:**
- `from` (ISO date, default: now)
- `to` (ISO date, default: now + 30 days)
- `entityType` (optional filter)

---

## 5. Storage Methods

Add to `IStorage` and `DatabaseStorage`:

| Method | Signature |
|--------|-----------|
| `createScheduledAction` | `(input: CreateScheduledActionInput) → Promise<ScheduledAction>` |
| `getScheduledAction` | `(id: number) → Promise<ScheduledAction \| undefined>` |
| `getScheduledActions` | `(params: ScheduledActionFilters) → Promise<{ actions, total, nextCursor }>` |
| `updateScheduledAction` | `(id: number, updates: PartialScheduledAction) → Promise<ScheduledAction>` |
| `cancelScheduledAction` | `(id: number) → Promise<ScheduledAction>` |
| `claimDueActions` | `(limit: number) → Promise<ScheduledAction[]>` | Uses `SELECT FOR UPDATE SKIP LOCKED` |
| `getUpcomingActions` | `(from: Date, to: Date, entityType?: string) → Promise<ScheduledAction[]>` |

---

## 6. Drizzle Schema

```typescript
export const scheduledActions = pgTable("scheduled_actions", {
  id:             serial("id").primaryKey(),
  entityType:     varchar("entity_type", { length: 50 }).notNull(),
  entityId:       varchar("entity_id", { length: 255 }).notNull(),
  actionType:     varchar("action_type", { length: 80 }).notNull(),
  payload:        jsonb("payload").notNull(),
  scheduledAt:    timestamp("scheduled_at", { withTimezone: true }).notNull(),
  status:         varchar("status", { length: 20 }).notNull().default("pending"),
  executedAt:     timestamp("executed_at", { withTimezone: true }),
  completedAt:    timestamp("completed_at", { withTimezone: true }),
  executionError: text("execution_error"),
  retryCount:     integer("retry_count").notNull().default(0),
  maxRetries:     integer("max_retries").notNull().default(3),
  createdById:    integer("created_by_id").references(() => users.id, { onDelete: "set null" }),
  createdAt:      timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt:      timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

---

## 7. Exit Criteria

- [ ] `scheduled_actions` table created with correct indexes.
- [ ] `POST /api/v1/scheduled-actions` creates valid actions; rejects actions scheduled in the past or with invalid payloads.
- [ ] `GET /api/v1/scheduled-actions` returns filtered, paginated list.
- [ ] `DELETE /api/v1/scheduled-actions/:id` cancels a pending action; returns 409 for non-pending.
- [ ] Background processor starts on server boot and polls every 30 seconds.
- [ ] `SELECT FOR UPDATE SKIP LOCKED` prevents double-execution — verified by creating two actions due at the same time and confirming each executes exactly once.
- [ ] `product.set_status` action executes correctly: a product scheduled to become `active` at a future time is `draft` before that time and `active` after.
- [ ] `product.set_values` action writes all values atomically using the existing value write path.
- [ ] `product.assign_category` correctly adds or removes a category assignment.
- [ ] Failed actions increment `retry_count` and reschedule with backoff; actions at `max_retries` stay `failed`.
- [ ] `triggerIntegrationRunsAfter` triggers integration runs after successful action completion.
- [ ] All endpoints require authentication and enforce correct permissions.

---

## 8. Notes

- **Idempotency.** Action execution calls the same service functions as immediate writes, which are already idempotent for most operations (set_status, set_values). Category assignment checks for duplicates. This makes retries safe.
- **No cron expressions in Phase 32.** Recurring schedules (cron-based) are added in Phase 33 for integration runs only. Phase 32 handles one-off scheduled actions only.
- **Scheduler precision.** The 30-second poll interval means actions may execute up to 30 seconds late. This is acceptable for MVP. Document as a known limitation.
- **Single server assumption.** `SELECT FOR UPDATE SKIP LOCKED` handles concurrent execution correctly even across multiple server instances. Replit runs a single instance, but this is future-safe.
- **Audit logging.** Actions executed by the scheduler should be logged in the audit history with `created_by = null` (or a special "scheduled" system actor), so the audit trail shows that a change was made by a scheduled action rather than a user.
