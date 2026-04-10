# Phase 17: API Consistency & Hardening

**Execution Plan Item:** #17  
**Replit Phase:** 17 (Week 4)  
**Dependencies:** All REST endpoints built in Weeks 1–3 (attributes, families, products, entities, assets, categories, channels, etc.)  
**Depended on by:** Frontend consistency, external API consumers, Phase 20 UI

---

## Objective

Ensure the **entire REST API surface** is consistent in patterns, well-structured, and properly error-handled. Standardize response formats, error codes, and pagination so that clients can rely on predictable behavior across all entity types.

---

## 1. Response Format

### Success — Single Resource

All `GET /:id` and create/update responses that return one resource should follow the same shape:

```json
{
  "data": { ... }
}
```

Optional: include `meta` for timestamps or version if needed.

### Success — List

All list endpoints should return:

```json
{
  "data": [ ... ],
  "meta": {
    "total": 123,
    "limit": 25,
    "cursor": "optional_cursor",
    "nextCursor": "optional_next_cursor_or_null"
  }
}
```

- `total` — total count matching the filter (optional if expensive; document when omitted).
- `limit` — page size used.
- Cursor-based: `cursor` (current), `nextCursor` (next page; null if no more). Offset-based alternatives can be documented if used in a few legacy endpoints.

### Success — Create (with location)

`POST` that creates a resource should return `201 Created` with `Location` header and body:

```json
{
  "data": { "id": 1, ... }
}
```

### Empty / No content

`DELETE` can return `204 No Content` with no body, or `200 OK` with `{ "data": null }` or a small confirmation. Pick one and document.

---

## 2. Error Response Format

All errors should use a consistent structure:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable message.",
    "details": [
      { "field": "labels", "message": "At least one label required." }
    ]
  }
}
```

- `code` — Machine-readable: e.g. `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_ERROR`, `CONFLICT`, `BAD_REQUEST`.
- `message` — Single summary for the client/toast.
- `details` — Optional; for validation errors, list per-field messages.

### HTTP Status Mapping

| Code | When |
|------|------|
| 400 | Bad request (validation, invalid payload). |
| 401 | Unauthenticated (no or invalid session). |
| 403 | Forbidden (authenticated but not allowed). |
| 404 | Resource not found (e.g. invalid id). |
| 409 | Conflict (e.g. duplicate, or "cannot delete because in use"). |
| 500 | Internal server error (log and return generic message). |

Use the same status codes across all endpoints (e.g. 404 for missing entity everywhere).

---

## 3. Pagination

- **Cursor-based** (preferred): `limit`, `cursor` query params. Response includes `nextCursor`; omit or null when no more.
- **Consistent defaults:** e.g. default `limit` 25, max 100 across list endpoints.
- **Sort order:** Document default sort (e.g. `updated_at DESC`, `sort_order ASC`) per endpoint. Optional `sort` param if needed.

---

## 4. Query Parameters

- Use **camelCase** in query params for consistency with JSON bodies: `familyId`, `channelId`, `attributeId`, `limit`, `cursor`, `search`.
- **Filtering:** Same param names across similar resources (e.g. `familyId` for products, entities filtered by `entityTypeId`).
- **Search:** `search` param where applicable; document what it matches (e.g. label, code, identifier).

---

## 5. Validation and Idempotency

- **Path params:** Validate `:id` is numeric where applicable; return 400 if not parseable, 404 if not found.
- **Body validation:** Use Zod (or existing) schemas; return 400 with `details` array for field errors.
- **Idempotency:** POST create is non-idempotent. Document if any endpoint supports idempotency keys later.

---

## 6. Hardening Checklist

- [ ] Every route that accepts `:id` validates and returns 404 when resource is missing.
- [ ] All list endpoints return `data` + `meta` with at least `limit`; cursor where applicable.
- [ ] All errors return the same `error.code` + `error.message` (+ optional `details`) shape.
- [ ] 401/403 used consistently for auth failures vs permission denied.
- [ ] No uncaught exceptions leaking stack traces; 500 returns generic message and logs details.
- [ ] Optional: OpenAPI/Swagger spec updated to reflect response/error shapes and status codes.

---

## 7. Exit Criteria

- [ ] Single-resource and list response formats are consistent across all entity endpoints.
- [ ] Error format and status codes are standardized and documented.
- [ ] Pagination (cursor, limit, nextCursor) is consistent where applicable.
- [ ] Query parameter naming is consistent (camelCase, shared names for filters).
- [ ] Validation and 404/400/409 behavior are predictable; no unhandled 500s in normal use.

---

## 8. Notes

- **Backward compatibility:** If existing clients rely on a different shape, consider a versioned API or a transition period; document in this phase.
- **Rate limiting:** Can be added later; not required for MVP but note in docs if deferred.
