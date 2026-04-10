# Phase 23: Integration Run Engine & Logging

**Execution Plan Item:** #23  
**Replit Phase:** 23 (Week 5)  
**Dependencies:** Phase 21 (Integration types & instances), Phase 22 (Field mappings & product filters), Phase 19 (RBAC), Phase 20.6 (API key auth for machine-triggered runs)  
**Depended on by:** Phase 24 (UI: trigger runs, view history/logs), Phase 25 (Akeneo import runs), Phase 32 (Scheduled full runs in Week 8)

---

## Objective

Build the **integration run engine** and **run logging** so that integration instances can execute on-demand runs (full push, full import, or single-record push for outbound) with full visibility. Each run is recorded in a **run history** table with status, timestamps, and record counts. Each record processed (product pushed, row imported, etc.) is logged in a **per-record log** table with success/failure/skip and a reason message. Retry logic for transient failures is supported. **Scheduled full runs** (triggered by Phase 31 scheduling) are out of scope for this phase and added in Phase 32.

---

## 1. Core Concepts

- **Run:** A single execution of an integration instance. Has a **run type**: `on_demand_full` (full sync/push or full import), `on_demand_single` (single product push — outbound only). Triggered by a user (via UI or API) or by an API key; stored as `triggered_by` (user id or `apikey:<id>`).
- **Run status:** `pending`, `running`, `completed`, `failed`, `cancelled`. `running` while the connector is executing; `completed` when finished successfully; `failed` when run aborted with error; `cancelled` if explicitly cancelled (optional for MVP).
- **Record log:** One row per record processed in a run (e.g. one product pushed, one Akeneo product imported). Status: `success`, `failure`, `skip`. Message and optional details JSON explain the outcome. Enables "drill into run and see every record" in Phase 24.

---

## 2. Database Schema

### Table: `integration_runs`

One row per run (full or single).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `integration_instance_id` | `INTEGER` | NOT NULL, FK → integration_instances.id, ON DELETE CASCADE | |
| `run_type` | `VARCHAR(50)` | NOT NULL | `on_demand_full`, `on_demand_single` |
| `status` | `VARCHAR(20)` | NOT NULL | `pending`, `running`, `completed`, `failed`, `cancelled` |
| `triggered_by` | `VARCHAR(255)` | NULL | User id (e.g. Entra OID) or `apikey:<api_key_id>` for API key. NULL if system/scheduled (Phase 32). |
| `single_record_id` | `VARCHAR(255)` | NULL | For `on_demand_single`: product identifier or entity id. NULL for full runs. |
| `started_at` | `TIMESTAMPTZ` | NULL | Set when run actually starts (worker picks up). |
| `finished_at` | `TIMESTAMPTZ` | NULL | Set when run ends (success or failure). |
| `record_count_success` | `INTEGER` | NOT NULL, DEFAULT 0 | Number of records processed successfully. |
| `record_count_failure` | `INTEGER` | NOT NULL, DEFAULT 0 | Number of records that failed. |
| `record_count_skip` | `INTEGER` | NOT NULL, DEFAULT 0 | Number of records skipped (e.g. filter excluded, duplicate). |
| `error_message` | `TEXT` | NULL | Top-level error if run failed (e.g. connection refused). |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | When run was created (queued). |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | Last status update. |

**Indexes:**
- `(integration_instance_id, created_at DESC)` for "list runs for this instance."
- `(status)` for "find pending runs" if using a background worker.
- `(created_at)` for global run history.

### Table: `integration_run_logs`

One row per record processed within a run.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `integration_run_id` | `INTEGER` | NOT NULL, FK → integration_runs.id, ON DELETE CASCADE | |
| `record_identifier` | `VARCHAR(255)` | NOT NULL | Product identifier, entity id, or external id — enough to identify the record. |
| `status` | `VARCHAR(20)` | NOT NULL | `success`, `failure`, `skip` |
| `message` | `TEXT` | NULL | Human-readable reason (e.g. "Validation failed: missing required attribute"). |
| `details` | `JSONB` | NULL | Optional extra context (e.g. validation errors, HTTP status). |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Order within run (e.g. 1, 2, 3…). |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Index:** `(integration_run_id, sort_order)` for "get all logs for this run" in order.

**Consideration:** For very large runs (e.g. 100k records), consider batching logs or capping stored rows per run and summarizing the rest. Document limit if any (e.g. last 10k logs per run); for MVP, store all and optimize later if needed.

---

## 3. Storage Methods

Add to `IStorage` and `DatabaseStorage`:

| Method | Signature | Description |
|--------|-----------|-------------|
| `createIntegrationRun` | `(data: InsertIntegrationRun) → IntegrationRun` | Insert run with status `pending`. Return run with id. |
| `getIntegrationRunById` | `(id: number) → IntegrationRunWithInstance \| undefined` | Run plus instance and type info (name, direction). |
| `getIntegrationRunsByInstanceId` | `(instanceId: number, options?: { limit?, cursor?, status? }) → { runs: IntegrationRun[], nextCursor? }` | Paginated list for instance. |
| `getIntegrationRuns` | `(options?: { instanceId?, limit?, cursor?, status? }) → { runs: IntegrationRunWithInstance[], nextCursor? }` | Global or filtered run list for UI. |
| `updateIntegrationRun` | `(id: number, data: Partial<IntegrationRun>) → IntegrationRun` | Update status, started_at, finished_at, record counts, error_message. |
| `appendIntegrationRunLog` | `(runId: number, log: InsertIntegrationRunLog) → void` | Insert one log row; optionally increment run's record_count_* in same transaction. |
| `appendIntegrationRunLogs` | `(runId: number, logs: InsertIntegrationRunLog[]) → void` | Batch insert logs; update run counts. Use transaction. |
| `getIntegrationRunLogs` | `(runId: number, options?: { limit?, offset? }) → IntegrationRunLog[]` | Logs for a run, ordered by sort_order or id. Paginate for large runs. |

Types:

```ts
type IntegrationRunWithInstance = IntegrationRun & {
  integrationInstance: { name: string; integrationType: { name: string; direction: 'inbound' | 'outbound' } };
};
```

---

## 4. Run Execution Flow

1. **Trigger:** Client calls `POST /api/v1/integration-instances/:id/runs` with body `{ runType: 'on_demand_full' }` or `{ runType: 'on_demand_single', recordIdentifier: 'PRD-001' }`.
2. **Create run:** Insert row in `integration_runs` with status `pending`, run_type, triggered_by (from req.user), single_record_id if applicable.
3. **Execute (synchronous or asynchronous):**
   - **Option A (MVP):** Execute in the same request. Set status to `running`, call connector (Phase 25 for Akeneo; outbound connector stub for now). For each record: append log row, update record_count_* on run. On finish: set status `completed` or `failed`, set finished_at, set error_message if failed.
   - **Option B:** Enqueue job (e.g. background worker); worker picks up pending run, executes, updates run and logs. Prefer Option A for MVP if runs are short-lived; move to Option B when runs become long (e.g. full catalog import).
4. **Connector contract:** The run engine does not implement Akeneo or Shopify logic. It provides: get instance + type, get mappings + filter (Phase 22), create run, append logs, update run. A **connector** (e.g. Phase 25) is invoked with (runId, instanceId) and is responsible for reading config, fetching/iterating data, and calling back to append logs and update counts. Document this contract in this phase.

---

## 5. Retry Logic

- **Transient failures:** If a single record fails with a retryable error (e.g. network timeout, 503), the connector may retry up to N times (e.g. 2) before logging as `failure`. Document: "Connectors should implement retry for transient errors; run engine does not auto-retry runs."
- **Run-level retry:** Optional: allow "retry run" (create a new run with same parameters). No automatic retry of failed runs in MVP; UI can offer "Retry" button that triggers a new run.

---

## 6. REST API Endpoints

### Trigger run

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/integration-instances/:id/runs` | Start a run. Body: `{ runType: 'on_demand_full' }` or `{ runType: 'on_demand_single', recordIdentifier: string }`. Validate instance exists and is active; validate runType for instance's type (e.g. single only for outbound). Create run (pending), then execute (sync or async). Response: `201 Created` with `{ data: IntegrationRun }` and Location header. If sync and run completes in request, run may already be `completed` or `failed`. |

### Run history

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-instances/:id/runs` | List runs for this instance. Query: `?limit=20`, `?cursor=...`, `?status=completed`. Response: `{ data: IntegrationRun[], meta: { nextCursor, total? } }`. |
| GET | `/api/v1/integration-runs` | List all runs (optional filter). Query: `?instanceId=1`, `?limit=20`, `?cursor=...`, `?status=failed`. For admin/UI dashboard. |
| GET | `/api/v1/integration-runs/:id` | Get one run with instance and type info. Response: `{ data: IntegrationRunWithInstance }`. |

### Run logs (per-record)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-runs/:id/logs` | List logs for this run. Query: `?limit=100`, `?offset=0` (or cursor). Response: `{ data: IntegrationRunLog[], meta: { total, limit, offset } }`. Order by sort_order or id. |

Use consistent error responses (404 for missing run/instance, 400 for invalid runType or recordIdentifier, 409 if instance already has a run in progress and you enforce one-at-a-time).

---

## 7. Connector Contract (for Phase 25 and future connectors)

Document the following so Phase 25 (Akeneo) and future outbound connectors can plug in:

1. **Input:** Run engine provides `runId`, `instanceId`. Connector loads instance (config, type), loads field mappings and (for outbound) product filter.
2. **Execution:** Connector performs the actual sync/import: fetch data, transform using mappings, call PIM API (inbound) or external API (outbound). For each record: call `appendIntegrationRunLog(runId, { recordIdentifier, status, message?, details? })` and update run's record_count_success / record_count_failure / record_count_skip (or let run engine do it from log insert).
3. **Lifecycle:** Connector must call `updateIntegrationRun(runId, { status: 'running' })` at start and `updateIntegrationRun(runId, { status: 'completed'|'failed', finished_at, error_message? })` at end.
4. **Idempotency:** Same instance can have multiple runs over time; each run is independent. No requirement for run to be idempotent; that is connector-specific.

---

## 8. Permissions

- Trigger run: require `integration:manage` or a dedicated `integration:run` permission.
- List runs / get run / get logs: require `integration:read` or `integration:manage`.
- Enforce that user/API key can only trigger runs for instances they can access (same permission scope).

---

## 9. Audit History

- **Resource type:** `integrationRun`. Optionally log "run triggered" with run id and instance id. Per-record logs are not duplicated into audit; the run_log table is the audit trail for the run.

---

## 10. Exit Criteria

- [ ] Tables `integration_runs` and `integration_run_logs` exist with schema above; Drizzle and Zod types in place.
- [ ] Storage methods implemented; create run, update run, append logs (single and batch), list runs with pagination, get run logs with pagination.
- [ ] REST API: POST to trigger run (on_demand_full, on_demand_single where applicable); GET runs by instance and GET runs global; GET run by id; GET run logs. Errors and validation as specified.
- [ ] Run execution flow documented; connector contract documented for Phase 25.
- [ ] Retry behavior documented (per-record retry in connector; no automatic run retry in engine).
- [ ] All endpoints protected by RBAC; triggered_by populated from req.user.

---

## 11. Notes

- **Scheduled runs:** Phase 32 will add the ability to schedule a run (e.g. "full run every night"). This phase only handles on-demand triggers. When Phase 31 (scheduling) exists, the scheduler can call the same "create run + execute" path with triggered_by = null or "scheduler".
- **Long-running runs:** If runs take minutes/hours, move execution to a background job and return 202 Accepted with run id; client polls GET run until status is completed/failed. MVP can keep sync execution if runs are short.
- **One run at a time:** Optionally enforce "only one run per instance in status pending/running" to avoid overlapping full runs; document the choice.
