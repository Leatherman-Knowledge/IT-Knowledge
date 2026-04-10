# Task 6: Audit History Table Schema

**Execution Plan Item:** #6
**Dependencies:** Project setup (Task 0)

---

## Objective

Create the audit history table and the reusable wiring pattern so that no data change goes untracked from the moment entities start being created. The actual wiring into each entity's create/update/delete paths happens incrementally as those entities are built (Tasks 2–5).

Also define the **wiring pattern** — how audit logging should be implemented so that every task follows the same approach.

---

## 1. Database Schema

### Table: `audit_history`

Append-only log of every data change in the system. This table will grow large — design for write performance and time-range queries.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | |
| `timestamp` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | When the change occurred |
| `user_id` | `VARCHAR(255)` | NULL | FK → pim_users.id. NULL for system/migration changes |
| `action` | `VARCHAR(20)` | NOT NULL | `create`, `update`, `delete` |
| `resource_type` | `VARCHAR(100)` | NOT NULL | `attribute`, `attributeGroup`, `family`, `category`, `channel`, `locale`, `role`, etc. |
| `resource_id` | `VARCHAR(255)` | NOT NULL | The code/identifier of the changed resource |
| `attribute_code` | `VARCHAR(255)` | NULL | For attribute-level changes: which attribute changed. NULL for entity-level actions (create, delete). |
| `channel_code` | `VARCHAR(100)` | NULL | For scoped value changes |
| `locale_code` | `VARCHAR(20)` | NULL | For scoped value changes |
| `old_value` | `JSONB` | NULL | Previous value (NULL on create) |
| `new_value` | `JSONB` | NULL | New value (NULL on delete) |
| `context` | `JSONB` | NULL | Additional context (e.g., `{ "source": "api", "endpoint": "/api/v1/attributes" }`) |

**Indexes** — optimized for the most common query patterns:

```sql
-- "Show me all changes for this resource"
CREATE INDEX idx_audit_resource ON audit_history(resource_type, resource_id, timestamp DESC);

-- "Show me all changes by this user"
CREATE INDEX idx_audit_user ON audit_history(user_id, timestamp DESC);

-- "Show me all changes in a time range"
CREATE INDEX idx_audit_timestamp ON audit_history(timestamp DESC);

-- "Show me all changes to this attribute across all resources"
CREATE INDEX idx_audit_attribute ON audit_history(attribute_code, timestamp DESC) WHERE attribute_code IS NOT NULL;
```

**Design note:** Avoid foreign key constraints FROM this table to other tables. The `user_id`, `resource_id`, etc. are soft references.

---

## 2. Wiring Pattern

Every task that creates, updates, or deletes data should follow this pattern. Define a reusable service function:

### Audit Service (`src/lib/audit.ts`)

```typescript
import { db } from "@/lib/db";
import { auditHistory } from "@drizzle/schema";

interface AuditEntry {
  userId: string | null;
  action: "create" | "update" | "delete";
  resourceType: string;
  resourceId: string;
  attributeCode?: string | null;
  channelCode?: string | null;
  localeCode?: string | null;
  oldValue?: any;
  newValue?: any;
  context?: Record<string, any>;
}

export async function logAudit(entry: AuditEntry): Promise<void> {
  await db.insert(auditHistory).values({
    userId: entry.userId,
    action: entry.action,
    resourceType: entry.resourceType,
    resourceId: entry.resourceId,
    attributeCode: entry.attributeCode ?? null,
    channelCode: entry.channelCode ?? null,
    localeCode: entry.localeCode ?? null,
    oldValue: entry.oldValue ?? null,
    newValue: entry.newValue ?? null,
    context: entry.context ?? null,
  });
}

export async function logAuditBatch(entries: AuditEntry[]): Promise<void> {
  await db.insert(auditHistory).values(
    entries.map(entry => ({
      userId: entry.userId,
      action: entry.action,
      resourceType: entry.resourceType,
      resourceId: entry.resourceId,
      attributeCode: entry.attributeCode ?? null,
      channelCode: entry.channelCode ?? null,
      localeCode: entry.localeCode ?? null,
      oldValue: entry.oldValue ?? null,
      newValue: entry.newValue ?? null,
      context: entry.context ?? null,
    }))
  );
}
```

### How to wire it — two levels of granularity

**1. Entity-level audit (for create/delete of top-level resources):**

```typescript
// When creating an attribute
const [attribute] = await db.insert(attributes).values({ ... }).returning();
await logAudit({
  userId: session.user.id,
  action: "create",
  resourceType: "attribute",
  resourceId: attribute.code,
  newValue: attribute,
});
```

**2. Field-level audit (for updates to individual fields):**

```typescript
// When updating an attribute's labels
const oldAttribute = await getAttribute(code);
const updatedAttribute = await updateAttribute(code, { labels: newLabels });
await logAudit({
  userId: session.user.id,
  action: "update",
  resourceType: "attribute",
  resourceId: code,
  oldValue: { labels: oldAttribute.labels },
  newValue: { labels: updatedAttribute.labels },
});
```

### Alternative: Drizzle Transaction Wrapper

You can create a wrapper function that automatically logs audit entries as part of the same database transaction:

```typescript
export async function withAudit<T>(
  userId: string | null,
  fn: (tx: typeof db) => Promise<{ result: T; auditEntries: AuditEntry[] }>
): Promise<T> {
  return db.transaction(async (tx) => {
    const { result, auditEntries } = await fn(tx);
    if (auditEntries.length > 0) {
      await tx.insert(auditHistory).values(
        auditEntries.map(e => ({ ...e, userId }))
      );
    }
    return result;
  });
}
```

This ensures the audit log and the data change succeed or fail together. However, it adds complexity to every call site. The **explicit `logAudit()` call pattern is recommended** for simplicity and clarity in most cases, with the transaction wrapper reserved for operations where atomicity between the data change and the audit entry is critical.

---

## 3. What Gets Audited (by task)

| Task | Resource Type | What's Logged |
|------|--------------|---------------|
| Task 1 (Auth) | `role`, `userRole` | Role created/deleted, role assigned/removed from user |
| Task 2 (Attributes) | `attribute`, `attributeGroup`, `attributeOption` | Created, updated (each changed field), deleted |
| Task 3 (Locales/Channels) | `locale`, `channel`, `currency` | Created, updated, deleted |
| Task 4 (Families) | `family`, `familyAttribute`, `familyRequirement` | Created, updated, deleted, attributes added/removed |
| Task 4.5 (Variant Definitions) | `familyVariantDefinition`, `familyVariantAxes`, `familyVariantAttributeLevels` | Definition created/updated/deleted, axes set/replaced, attribute levels set/replaced |
| Task 5 (Categories) | `categoryTree`, `category` | Created, updated (including moves), deleted |

---

## 4. API Endpoints

**`GET /api/v1/audit-history`** — Query audit history. Supports filters:
  - `?resourceType=product` — filter by resource type
  - `?resourceId=WINGMAN` — filter by specific resource
  - `?userId=abc123` — filter by user
  - `?action=update` — filter by action
  - `?attributeCode=productName` — filter by attribute
  - `?from=2026-01-01T00:00:00Z&to=2026-02-01T00:00:00Z` — time range
  - Standard cursor-based pagination

**`GET /api/v1/audit-history/resource/:resourceType/:resourceId`** — Shortcut: get full change history for a specific resource, newest first.

Response:
```json
{
  "data": [
    {
      "id": "uuid",
      "timestamp": "2026-02-19T15:30:00Z",
      "userId": "azure-oid-123",
      "userName": "Cody Luth",
      "action": "update",
      "resourceType": "attribute",
      "resourceId": "productName",
      "attributeCode": null,
      "oldValue": { "labels": { "en_US": "Name" } },
      "newValue": { "labels": { "en_US": "Product Name" } }
    }
  ],
  "meta": { "total": 47, "limit": 25, "cursor": "...", "nextCursor": "..." }
}
```

Note: The response enriches `userId` with `userName` by joining to `pim_users`. The API should do this join at query time.

---

## 5. Business Rules

1. **Audit history is append-only.** No updates, no deletes. Once a record is written, it's permanent.
2. **Audit writes should not block the main operation.** If the audit insert fails, the main operation should still succeed. Use a try-catch around the audit log call and log the failure to application logs. Synchronous inserts are fine.
3. **The `old_value` and `new_value` columns store JSONB.** This handles all value types uniformly. For simple values, wrap them: `{ "value": "old text" }`. For complex changes (multiple fields updated on an entity), store the full diff.
4. **System actions** use `user_id = NULL` and include a `context` field identifying the source.

---

## 6. Wiring Priority

The audit wiring should focus on entity-level creates/updates/deletes:

- **Task 2 (Attributes):** Log when attributes, groups, and options are created, updated, or deleted
- **Task 3 (Locales/Channels):** Log when locales, channels, and currencies are created, updated, or deleted
- **Task 4 (Families):** Log when families are created/updated/deleted and when attributes are added/removed from families
- **Task 4.5 (Variant Definitions):** Log when variant definitions are created/updated/deleted, when axes are set/replaced, and when attribute levels are set/replaced
- **Task 5 (Categories):** Log when trees and categories are created, updated (including moves), or deleted

