# Phase 33: Scheduled Integration Runs

**Execution Plan Item:** #33  
**Replit Phase:** 33 (Week 9)  
**Dependencies:** Phase 23 (Integration Run Engine), Phase 32 (Scheduling Infrastructure)  
**Depended on by:** Phase 34 (Scheduling UI — schedule configuration for integration instances)

---

## Objective

Wire the Phase 23 integration run engine to the Phase 32 scheduling infrastructure so integration instances can run on a **recurring schedule** in addition to on-demand. An Akeneo import, for example, can be configured to run every night at 2am without manual intervention.

This phase is intentionally narrow: it adds **schedule configuration** to integration instances and a **schedule manager** that creates the next `scheduled_action` when a run completes. Everything else — the run engine, logging, status tracking — is unchanged from Phase 23.

---

## 1. Integration Instance Schedule Configuration

### Schema Change

Add a `schedule` column to `integration_instances`:

```sql
ALTER TABLE integration_instances
  ADD COLUMN schedule_cron    VARCHAR(100),   -- cron expression, e.g. "0 2 * * *"
  ADD COLUMN schedule_enabled BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN schedule_run_type VARCHAR(20) NOT NULL DEFAULT 'full',
  ADD COLUMN next_scheduled_run_at TIMESTAMPTZ; -- denormalized next run time for quick querying
```

| Column | Description |
|--------|-------------|
| `schedule_cron` | Standard 5-field cron expression (`minute hour dom month dow`). `null` = no recurring schedule. |
| `schedule_enabled` | Whether the recurring schedule is active. Allows pausing without clearing the cron expression. |
| `schedule_run_type` | `"full"` or `"incremental"`. Must be supported by the connector type. |
| `next_scheduled_run_at` | Denormalized: computed from `schedule_cron` when the schedule is saved or after each run. Used by the schedule manager for efficient polling and UI display. |

### Cron Expression Validation

Use the `cron-parser` or `cronstrue` npm package to:
1. Validate that the cron expression is syntactically valid on save.
2. Compute `next_scheduled_run_at` from the current time.
3. Provide a human-readable description for the UI (e.g. `"Every day at 2:00 AM"`).

Reject expressions with a sub-minute interval (e.g. `*/10 * * * * *` — 6-field second-level cron) — these are unsupported for MVP.

---

## 2. Schedule Manager

A lightweight module (`server/lib/schedule-manager.ts`) responsible for keeping the `scheduled_actions` table current with upcoming integration runs.

### `ensureNextRunScheduled(instanceId)`

Called after every integration run completes (success or failure) and after schedule configuration changes. Logic:

```
1. Load the integration instance.
2. If schedule_enabled = false OR schedule_cron = null → do nothing.
3. Compute next_run_at = nextCronDate(schedule_cron, now).
4. Check if a pending scheduled_action of type 'integration.trigger_run' already exists
   for this instance with scheduled_at = next_run_at (±1 min tolerance).
   - If yes → do nothing (already scheduled).
   - If no  → create the scheduled_action and update next_scheduled_run_at on the instance.
```

This function is idempotent — calling it multiple times for the same instance has no side effects.

### `syncAllSchedules()`

Called on server startup. Iterates all integration instances with `schedule_enabled = true` and calls `ensureNextRunScheduled(instanceId)` for each. This recovers from any missed schedule registrations after a server restart or crash.

```typescript
async function syncAllSchedules(): Promise<void> {
  const instances = await storage.getScheduledIntegrationInstances();
  for (const instance of instances) {
    await ensureNextRunScheduled(instance.id);
  }
}
```

---

## 3. Integration with Phase 23 Run Engine

### After Run Completion

In the Phase 23 run engine, at the point where a run's final status is written (`completed` or `failed`), add:

```typescript
// After writing final run status:
await ensureNextRunScheduled(instanceId);
```

This ensures the next scheduled run is always registered immediately after the current one finishes, regardless of success or failure.

### On Schedule Configuration Change

In the integration instance update route (`PUT /api/v1/integration-instances/:id`), when `scheduleCron` or `scheduleEnabled` changes:

1. Cancel any existing `pending` scheduled actions of type `integration.trigger_run` for this instance.
2. Call `ensureNextRunScheduled(instanceId)` to register the next run under the new schedule.

---

## 4. REST API Changes

### `PUT /api/v1/integration-instances/:id`

Extend the existing update endpoint to accept schedule fields:

```json
{
  "scheduleCron": "0 2 * * *",
  "scheduleEnabled": true,
  "scheduleRunType": "full"
}
```

**Response** now includes schedule fields:
```json
{
  "id": 3,
  "name": "Akeneo Nightly Import",
  "scheduleCron": "0 2 * * *",
  "scheduleEnabled": true,
  "scheduleRunType": "full",
  "nextScheduledRunAt": "2026-03-11T02:00:00Z",
  "scheduleCronHuman": "Every day at 2:00 AM"
}
```

The `scheduleCronHuman` field is a server-computed human-readable description of the cron expression. Include it in every response that includes `scheduleCron`.

### `GET /api/v1/integration-instances/:id`

Response extended with same schedule fields.

### `GET /api/v1/integration-instances/:id/next-run`

Returns the next scheduled run time for an integration instance, useful for the UI status display.

```json
{
  "instanceId": 3,
  "nextScheduledRunAt": "2026-03-11T02:00:00Z",
  "scheduleCronHuman": "Every day at 2:00 AM",
  "scheduledActionId": 47
}
```

Returns `{ "nextScheduledRunAt": null }` if no schedule is configured or it is disabled.

---

## 5. Drizzle Schema

Add columns to the existing `integrationInstances` table definition in `shared/schema.ts`:

```typescript
scheduleCron:          varchar("schedule_cron", { length: 100 }),
scheduleEnabled:       boolean("schedule_enabled").notNull().default(false),
scheduleRunType:       varchar("schedule_run_type", { length: 20 }).notNull().default("full"),
nextScheduledRunAt:    timestamp("next_scheduled_run_at", { withTimezone: true }),
```

---

## 6. Storage Methods

Add to `IStorage` and `DatabaseStorage`:

| Method | Signature | Description |
|--------|-----------|-------------|
| `updateIntegrationInstanceSchedule` | `(id: number, schedule: ScheduleConfig) → Promise<IntegrationInstance>` | Updates schedule fields and `next_scheduled_run_at` |
| `getScheduledIntegrationInstances` | `() → Promise<IntegrationInstance[]>` | Returns all instances with `schedule_enabled = true` |

---

## 7. Exit Criteria

- [ ] `schedule_cron`, `schedule_enabled`, `schedule_run_type`, and `next_scheduled_run_at` columns added to `integration_instances`.
- [ ] `PUT /api/v1/integration-instances/:id` accepts and validates cron expressions; rejects syntactically invalid expressions with a clear error.
- [ ] `scheduleCronHuman` is returned in all integration instance responses that include a `scheduleCron`.
- [ ] `ensureNextRunScheduled` creates a `scheduled_action` for the next cron occurrence after a run completes.
- [ ] On server startup, `syncAllSchedules` registers missing next-run actions for all enabled schedules (recovery from server restart).
- [ ] Disabling a schedule (`scheduleEnabled: false`) cancels the pending `scheduled_action` for that instance.
- [ ] Changing the cron expression cancels the old pending action and creates a new one for the updated schedule.
- [ ] A full end-to-end test: configure an instance with a cron that fires ~1 minute in the future; confirm a run is triggered automatically at that time; confirm the next run is scheduled afterward.
- [ ] `GET /api/v1/integration-instances/:id/next-run` returns the correct next run time.

---

## 8. Notes

- **One pending action per instance.** The schedule manager ensures at most one pending `integration.trigger_run` action exists per instance at any time. If two are somehow created (e.g. rapid config change), the second `ensureNextRunScheduled` call detects the existing one and skips creation.
- **No cron for product actions.** Recurring schedules apply only to integration runs in Phase 33. Product actions (set_status, set_values, etc.) are always one-off in Phase 32. Recurring product actions (e.g. "activate every Monday") are post-MVP.
- **Cron timezone.** For MVP, evaluate cron expressions in UTC. Add a `schedule_timezone` column in a future phase if users need local-time scheduling.
- **Phase 34 dependency.** The scheduling UI (Phase 34) surfaces the schedule configuration fields added here in the integration instance edit form. Phase 34 uses `scheduleCronHuman` for display, `scheduleCron` for editing, and `nextScheduledRunAt` for the timeline view.
