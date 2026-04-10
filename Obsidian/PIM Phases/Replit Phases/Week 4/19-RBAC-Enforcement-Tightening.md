# Phase 19: RBAC Enforcement Tightening

**Execution Plan Item:** #19  
**Replit Phase:** 19 (Week 4)  
**Dependencies:** Auth & RBAC schema (Phase 01), all API routes (Weeks 1–3), permissions/roles tables and `hasPermission` pattern  
**Depended on by:** Security audit, multi-tenant or multi-role deployments, Phase 20 (UI hiding of forbidden actions)

---

## Objective

Move from **loosely enforced** permissions (any authenticated user can do anything) to **strict enforcement**: every API endpoint checks that the current user has the required permission for the resource type, action, and optional scope (locale, channel, attribute group). Verify that role boundaries are respected and that the UI can rely on API responses (403) to hide or disable actions.

---

## 1. Permission Model (Recap)

From Phase 01:

- **roles** — e.g. admin, editor, viewer (string code PK).
- **permissions** — resource_type + action + optional scope_type + scope_value (e.g. `attribute`, `update`, `locale`, `en-US`).
- **role_permissions** — which permissions each role has.
- **user_roles** — which roles each user has.

`hasPermission(userId, resourceType, action, scope?)` should return true only if the user has at least one role that grants the matching permission.

---

## 2. Resource Types and Actions

Map each API area to resource type and actions:

| Resource Type | Actions | Notes |
|---------------|---------|--------|
| attribute | create, read, update, delete | Attribute definitions |
| attributeGroup | create, read, update, delete | |
| family | create, read, update, delete | |
| categoryTree | create, read, update, delete | |
| category | create, read, update, delete | |
| channel | create, read, update, delete | |
| locale | create, read, update, delete | |
| product | create, read, update, delete | Products and variant hierarchy |
| productValue | read, update, delete | Attribute values on products |
| entityType | create, read, update, delete | |
| entity | create, read, update, delete | |
| entityValue | read, update, delete | |
| assetType | create, read, update, delete | |
| asset | create, read, update, delete | |
| assetValue | read, update, delete | |
| associationType | create, read, update, delete | Phase 15 |
| productAssociation | create, read, update, delete | Phase 15 |
| auditHistory | read | View audit log |

Scoped permissions (optional for MVP but prepare the model):

- **locale:** scope_type = 'locale', scope_value = locale code or '*' for all.
- **channel:** scope_type = 'channel', scope_value = channel id or '*' for all.
- **attributeGroup:** scope_type = 'attributeGroup', scope_value = group id or '*'.

If scope is present, the endpoint must resolve the scope of the request (e.g. which locale/channel is being written) and pass it to `hasPermission`.

---

## 3. Enforcement Pattern

For each route:

1. After `requireAuth`, resolve the resource type and action (e.g. GET attribute = read, POST = create).
2. Optionally resolve scope from path/query/body (e.g. channelId, localeCode).
3. Call `hasPermission(req.user.id, resourceType, action, scope)`.
4. If false, return **403 Forbidden** with `error.code = 'FORBIDDEN'`, `message = 'You do not have permission to perform this action.'`.
5. If true, proceed.

Use a reusable middleware or helper so every route goes through the same check:

```typescript
// Example
function requirePermission(resourceType: string, action: string, scopeResolver?: (req) => Scope | null) {
  return async (req, res, next) => {
    const scope = scopeResolver ? scopeResolver(req) : null;
    const allowed = await hasPermission(req.user.id, resourceType, action, scope);
    if (!allowed) return res.status(403).json({ error: { code: 'FORBIDDEN', message: '...' } });
    next();
  };
}
```

---

## 4. Default Role Permissions

Define and seed default permissions for admin, editor, viewer:

- **admin:** All resource types, all actions, all scopes (scope_value = '*' or equivalent).
- **editor:** create, read, update for most resources; delete where appropriate; no auditHistory delete; no role/permission management if those exist.
- **viewer:** read only for all resources (read on productValue, entityValue, assetValue, auditHistory).

Document the exact matrix (role × resource × action) in this phase or in a separate permissions matrix file.

---

## 5. UI Considerations

- **403 responses:** Frontend should treat 403 as "no permission"; disable or hide the action and optionally show a tooltip.
- **Initial load:** Optionally provide `GET /api/v1/me/permissions` (or include in session) so the UI can hide buttons without calling the API first. Caching permission list per session is acceptable.

---

## 6. Exit Criteria

- [ ] Every write endpoint (create, update, delete) checks permission for the corresponding resource type and action.
- [ ] Every read endpoint checks read permission (except public health check if any).
- [ ] 403 is returned when the user lacks permission; 401 when unauthenticated.
- [ ] Default roles (admin, editor, viewer) are seeded with a documented permission set.
- [ ] Optional: scope (locale/channel/attributeGroup) is passed and enforced where applicable.
- [ ] No endpoint allows an action without a permission check.

---

## 7. Notes

- **Audit:** Log permission denials (user, resource, action) for security auditing if needed.
- **Super-admin:** One role (e.g. admin) can bypass checks for operational recovery; document if used.
