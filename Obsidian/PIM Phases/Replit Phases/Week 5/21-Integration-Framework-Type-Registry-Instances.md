# Phase 21: Integration Framework — Type Registry & Instances

**Execution Plan Item:** #21  
**Replit Phase:** 21 (Week 5)  
**Dependencies:** Phase 17 (API consistency), Phase 19 (RBAC), Phase 20.6 (API key auth for machine-triggered runs)  
**Depended on by:** Phase 22 (Mapping & Filtering), Phase 23 (Run Engine), Phase 24 (Integration UI), Phase 25 (Akeneo Import as integration type)

---

## Objective

Build the **integration type registry** and **integration instances** so the PIM can support pluggable connectors. Each **integration type** defines a kind of connector (e.g. Akeneo Import, Shopify, FBA) with a **direction** (inbound = data into PIM, outbound = data from PIM to external system), connection requirements, and run capabilities. Each **integration instance** is a single configured connection — one instance per source or endpoint. Multiple instances of the same type (e.g. 8 Shopify stores) are fully independent. Design the registry and data model to support **both inbound and outbound** from the start so Phase 25 (Akeneo) can be the first inbound type without rework.

---

## 1. Core Concepts

- **Integration type:** A registered connector kind. Has a unique `code` (e.g. `akeneo_import`, `shopify`), display name, **direction** (`inbound` | `outbound`), and a **connection schema** describing what configuration an instance needs (e.g. API base URL, credentials, file path). Types are registered in code or via seed/migration; they are not created by end users at runtime.
- **Integration instance:** A concrete connection of a given type. Has a name, the type it belongs to, and a **config** object (JSON) holding type-specific settings (e.g. Akeneo API URL, Shopify store domain). Instances are created and edited by admins via API and UI. Each instance can have its own credentials (stored securely; see §4).
- **Direction:** `inbound` = data flows into the PIM (e.g. import from Akeneo); `outbound` = data flows from the PIM to an external system (e.g. push to Shopify). The run engine (Phase 23) uses direction to decide whether to run an "import" or a "push."

**Authentication — two directions:** (1) **Integration → PIM:** When a run needs to call the PIM API (e.g. inbound import writing products, or a connector reading from the PIM), it authenticates using a **managed API key** from Phase 20.6. There is no separate authentication system for integrations; 20.6 is the single mechanism for machine access to the PIM. An instance can optionally be linked to a specific API key (`pim_api_key_id`) so runs execute "as" that key. (2) **PIM → external system:** When the PIM calls an external system (e.g. Akeneo API, Shopify), the instance's **config/credentials** hold that system's secrets (see §4). Those are not 20.6 keys — they are the external system's credentials only.

---

## 2. Database Schema

### Table: `integration_types`

Defines the kinds of connectors available. Populated by seed or migration; not CRUD by end users in MVP.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | Auto-increment |
| `code` | `VARCHAR(100)` | NOT NULL, UNIQUE | Machine-readable identifier (e.g. `akeneo_import`, `shopify`) |
| `name` | `VARCHAR(255)` | NOT NULL | Display name (e.g. "Akeneo Import", "Shopify") |
| `description` | `TEXT` | NULL | Short description for UI |
| `direction` | `VARCHAR(20)` | NOT NULL | `inbound` or `outbound` |
| `connection_schema` | `JSONB` | NOT NULL, DEFAULT '{}' | JSON Schema or key-value spec describing required/optional config fields for an instance (e.g. `{ "baseUrl": { "type": "string", "required": true }, "apiKey": { "type": "string", "required": true, "secret": true } }`). Used by UI to render connection form and validate config. |
| `supports_full_run` | `BOOLEAN` | NOT NULL, DEFAULT true | Whether this type supports "run full sync/import" |
| `supports_single_record` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether this type supports "push single product" (typically outbound only) |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order in type list |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Index:** `(direction)` for filtering types by direction in UI.

### Table: `integration_instances`

One row per configured connection (e.g. one Akeneo source, one Shopify store).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `SERIAL` | PK | |
| `integration_type_id` | `INTEGER` | NOT NULL, FK → integration_types.id, ON DELETE RESTRICT | |
| `name` | `VARCHAR(255)` | NOT NULL | Human-readable label (e.g. "Production Akeneo", "Shopify US Store") |
| `description` | `TEXT` | NULL | Optional notes |
| `config` | `JSONB` | NOT NULL, DEFAULT '{}' | Type-specific connection/config for **PIM → external system** (e.g. Akeneo base URL, Shopify store domain). Do **not** store raw external secrets here if avoidable; use credentials table or env. See §4. |
| `pim_api_key_id` | `INTEGER` | NULL, FK → api_keys.id, ON DELETE SET NULL | Optional. When set, runs that need to call the PIM API use this managed API key (Phase 20.6) for authentication. Enables per-instance identity, audit ("triggered by" can show key name), and easy rotation/revocation. |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | Soft-disable without deleting |
| `created_by` | `VARCHAR(255)` | NULL, FK → pim_users.id | Audit |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:** `(integration_type_id)`, `(is_active)`, `(pim_api_key_id)` (optional, for "list instances using this key").

**PIM authentication:** If `pim_api_key_id` is set, the run engine (or the connector, e.g. Phase 25) uses that key's token when making requests to the PIM. If null, runs triggered via UI use the current user's session; runs triggered by an external scheduler or script must either pass a Bearer token (the caller's responsibility) or the instance should have a linked key. Document the chosen default (e.g. "scheduled runs require a linked API key").

### Drizzle and Types

- Add both tables to `shared/schema.ts` using existing Drizzle patterns.
- Export Zod insert/select schemas and TypeScript types (e.g. `IntegrationType`, `IntegrationInstance`, `InsertIntegrationType`, `InsertIntegrationInstance`).
- Run migrations to create tables.

---

## 3. Storage Methods

Add to `IStorage` and `DatabaseStorage` in `server/storage.ts`:

| Method | Signature | Description |
|--------|-----------|-------------|
| `getAllIntegrationTypes` | `() → IntegrationType[]` | List all types, ordered by sort_order. Used by UI and run engine. |
| `getIntegrationTypeByCode` | `(code: string) → IntegrationType \| undefined` | Lookup by code (for run engine and Phase 25). |
| `getIntegrationTypeById` | `(id: number) → IntegrationType \| undefined` | Single type by id. |
| `getAllIntegrationInstances` | `(options?: { typeId?: number, direction?: 'inbound' \| 'outbound', activeOnly?: boolean }) → IntegrationInstanceWithType[]` | List instances, optionally filtered. Join with integration_types for type name and direction. |
| `getIntegrationInstanceById` | `(id: number) → IntegrationInstanceWithType \| undefined` | Single instance with type info. |
| `createIntegrationInstance` | `(data: InsertIntegrationInstance) → IntegrationInstance` | Insert instance. Validate config against type's connection_schema if implemented. |
| `updateIntegrationInstance` | `(id: number, data: Partial<IntegrationInstance>) → IntegrationInstance` | Update name, description, config, is_active. |
| `deleteIntegrationInstance` | `(id: number) → void` | Delete instance. Phase 23 may require no runs in progress; otherwise CASCADE or restrict as appropriate. |

Type:

```ts
type IntegrationInstanceWithType = IntegrationInstance & {
  integrationType: { code: string; name: string; direction: 'inbound' | 'outbound'; connectionSchema: unknown };
  pimApiKey?: { id: number; name: string; keyPrefix: string } | null;  // optional join for display; never include raw key
};
```

---

## 4. Credentials and Secrets

**Two kinds of credentials — do not confuse:**

1. **External system credentials (PIM → external):** Used when the PIM calls Akeneo, Shopify, etc. (fetch data, push products). Stored per instance in `config` or a separate credentials store. These are **not** Phase 20.6 API keys; they are the external system's API keys, tokens, or secrets.
2. **PIM API key (integration → PIM):** Used when an integration run needs to call the PIM's own API (e.g. inbound import creating products via REST). This **is** a managed API key from Phase 20.6. The instance optionally references one via `pim_api_key_id`. No separate authentication system for integrations — 20.6 is the single mechanism for machine access to the PIM.

**External secrets (option A or B):**

- **Option A (MVP):** Store non-secret config in `config` (e.g. `baseUrl`, `storeDomain`). Store external secrets (Akeneo API key, Shopify token) in a separate `integration_instance_credentials` table with encrypted values, or in environment variables keyed by instance id (e.g. `INTEGRATION_1_AKENEO_API_KEY`). Document the chosen approach.
- **Option B:** Store a single encrypted blob in `config` for the instance and decrypt only when running. Prefer a minimal approach for MVP so Phase 25 can proceed; refine in a later phase if needed.
- **API:** Never return raw external secrets in GET/POST responses. Mask in API (e.g. `"apiKey": "••••••••"`) and allow "update credentials" without re-sending if unchanged. The `pim_api_key_id` is just an id reference; the actual key value is never returned (it lives in `api_keys` and is only used at runtime by the run engine/connector with the key's token).

---

## 5. REST API Endpoints

### Integration Types (read-only in MVP)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-types` | List all integration types. Query: `?direction=inbound` or `?direction=outbound` (optional). Response: `{ data: IntegrationType[] }`. |
| GET | `/api/v1/integration-types/:id` | Get one type by id. Response: `{ data: IntegrationType }`. |
| GET | `/api/v1/integration-types/code/:code` | Get one type by code (e.g. `akeneo_import`). Response: `{ data: IntegrationType }`. |

No POST/PATCH/DELETE on types in MVP; types are seeded or registered in code.

### Integration Instances

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/integration-instances` | List instances. Query: `?typeId=1`, `?direction=inbound`, `?activeOnly=true`. Response: `{ data: IntegrationInstanceWithType[], meta: { total, limit } }`. |
| GET | `/api/v1/integration-instances/:id` | Get one instance with type. Include `pimApiKey` summary (id, name, keyPrefix) if set; never include raw key. Mask external secrets in config. |
| POST | `/api/v1/integration-instances` | Create. Body: `integrationTypeId`, `name`, `description?`, `config`, `pimApiKeyId?`, `isActive?`. Validate type exists; validate config shape against type's connection_schema if implemented. If `pimApiKeyId` provided, validate key exists and is active. Return 201 + Location. |
| PATCH | `/api/v1/integration-instances/:id` | Update name, description, config, pim_api_key_id, is_active. Mask external secrets in config; never return raw key. For `pimApiKeyId`, accept id or null to unlink. |
| DELETE | `/api/v1/integration-instances/:id` | Delete instance. Return 204 or 200. Consider blocking if runs exist (Phase 23); document behavior. |

Use consistent error codes (404 for missing type/instance, 400 for validation, 409 if delete not allowed).

---

## 6. Permissions

- **Integration types:** Read-only; any authenticated user with permission to manage integrations can list types. Use a permission such as `integration:read` or `integration:manage` (Phase 19 RBAC). Define one permission that covers "view and manage integration instances and runs."
- **Integration instances:** Create, update, delete require `integration:manage` (or equivalent). List and get require `integration:read` or `integration:manage`.
- Document the permission code in this phase so Phase 24 UI can gate actions.

---

## 7. Seed: Register at Least One Type (Placeholder)

To unblock UI and Phase 23, seed at least one integration type:

- **Option 1:** Seed `akeneo_import` (inbound) with a minimal `connection_schema` so Phase 25 can add the real implementation later.
- **Option 2:** Seed a generic `generic_outbound` or `webhook` type for testing outbound runs.

Example seed row:

```sql
INSERT INTO integration_types (code, name, description, direction, connection_schema, supports_full_run, supports_single_record, sort_order)
VALUES (
  'akeneo_import',
  'Akeneo Import',
  'Import product data from Akeneo into the PIM.',
  'inbound',
  '{"baseUrl":{"type":"string","required":true,"label":"Akeneo API URL"},"apiKey":{"type":"string","required":true,"secret":true,"label":"API Key"}}',
  true,
  false,
  10
);
```

Phase 25 will implement the actual Akeneo import logic; this phase only ensures the type and instance exist so an instance can be created and runs can be triggered.

---

## 8. Audit History

- **Resource types:** `integrationType` (read-only, optional audit on config changes if types ever become editable), `integrationInstance`.
- **Actions:** create, update, delete on **integration instances** only. Log who created/updated/deleted and when. Do not log raw secrets in audit payload.

---

## 9. Exit Criteria

- [ ] Tables `integration_types` and `integration_instances` exist with schema above (including `pim_api_key_id` FK to `api_keys`); Drizzle and Zod types in place.
- [ ] Storage methods implemented for types and instances; instances can be filtered by type and direction. When returning instance with type, optionally join `pimApiKey` (id, name, keyPrefix) for display.
- [ ] REST API: list/get integration types (with optional direction filter); full CRUD on integration instances including optional `pimApiKeyId` on create/update. External secrets masked in config; PIM key never returned (only id/name/prefix when linked).
- [ ] At least one integration type (e.g. `akeneo_import`) seeded so instances can be created.
- [ ] Permissions documented; endpoints protected by RBAC (Phase 19).
- [ ] Create/update/delete on instances are audit-logged (resource type `integrationInstance`).
- [ ] Document when runs use the instance's linked API key (e.g. scheduled or machine-triggered runs) vs session (UI-triggered); Phase 23/25 use the linked key when calling the PIM API if set.

---

## 10. Notes

- **No separate integration auth system.** Integrations that need to call the PIM use the same managed API keys from Phase 20.6. Linking an instance to a key via `pim_api_key_id` gives per-instance identity, easier rotation, and a single place (20.6 admin UI) to manage all machine access.
- **Connection schema:** The `connection_schema` JSON can be a simple key-value spec for MVP (field name → { type, required, label, secret }). A full JSON Schema can be adopted later for richer validation.
- **Direction:** Keeping direction on the type (not the instance) ensures every instance of e.g. Akeneo is always inbound; no mixed use.
- **Phase 25:** Will add the Akeneo import runner that reads an instance of type `akeneo_import`, reads its config (and uses the instance's `pim_api_key_id` when calling the PIM API to write products), and performs the import using the run engine from Phase 23.
