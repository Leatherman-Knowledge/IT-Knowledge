# Phase 30: UI — Search, Completeness & Data Exploration

**Execution Plan Item:** #30  
**Replit Phase:** 30 (Week 7)  
**Dependencies:** Phase 12 (Product UI), Phase 12.1.1 (URL-persisted channel/locale, deep links), Phase 28 (Full-Text Search API), Phase 29 (Completeness API — aggregate, missing attributes, batch)  
**Depended on by:** Nothing (end-user feature). Enables Phase 34 (Final Polish).

---

## Objective

Deliver three interlocked UI improvements that turn the PIM into a data exploration tool:

1. **Search interface** — A fast, faceted search bar that works across the product list and returns ranked results. Replaces basic identifier search with full-text + facets.
2. **Completeness dashboard** — A dedicated completeness view that shows per-channel, per-locale, per-variant scores at a glance, with drill-down to individual missing attributes and one-click navigation to the incomplete field.
3. **Product list enhancements** — Completeness indicators inline on the product list rows, visible without navigating into each product.

All screens use the existing design system (brand colors, dark mode, component library from Phase 20.5). No new layout or navigation structure is added — these features slot into the existing Integrations, Products, and Admin navigation.

---

## 1. Search Interface

### Location

The search bar lives at the **top of the product list page** (`/products`). It replaces the current basic text filter (if any) and adds faceted controls alongside it. The URL reflects the search state so results are shareable and bookmarkable: `/products?q=signal&familyId=3&status=active`.

### Search Bar

- **Input:** Single text input labeled "Search products…" with a magnifying glass icon. On type, debounce 300ms and issue `GET /api/v1/products/search?q=<term>`. Clear button (×) clears the query and resets to the unfiltered list.
- **Keyboard:** `Enter` submits immediately without debounce wait. `Escape` clears the search and returns focus to the input.
- **Loading state:** Show a spinner inside the search bar while the request is in flight. Do not clear current results while loading — show new results when they arrive (avoids flicker).
- **Empty state:** If `q` returns zero results: "No products found matching '{query}'. Try adjusting your search or clearing some filters."
- **Result count:** Show "Showing {total} results" below the search bar, or "Showing {filtered} of {total}" when facet filters are active.

### Facet Filters

Display as a **collapsible sidebar** (on desktop) or an expandable filter drawer (on mobile). Facet sections:

| Facet | Control | API parameter |
|-------|---------|---------------|
| Family | Multi-select checkboxes (labels from `facets.families` in response) | `familyId` |
| Status | Multi-select checkboxes (Draft / Active / Archived) | `status` |
| Category | Searchable multi-select tree (shows category hierarchy, selects one node) | `categoryId` |
| Completeness | Threshold selector: "All", "Complete only (100%)", "Incomplete (<100%)", "Custom (≥N%)" | `completeness` + `channelCode` + `localeCode` |
| Channel + Locale | Required context when completeness facet is set; rendered as two selects below the completeness filter | `channelCode`, `localeCode` |

Facet counts are shown next to each option using the `facets` object returned by the API. Counts are from the full matched result set (before pagination), so they remain stable as the user pages through results.

Active filters are shown as **chips** below the search bar. Each chip has a ✕ to remove that filter without clearing others. "Clear all filters" button resets to the default unfiltered view.

### Search Results Display

Search results use the same product list table as the existing product list page. When a search query is active:

- Add a **Relevance** column (optional, can be hidden by default) showing the `ts_rank` score. Not shown in facet-only mode.
- The row order is **rank descending** when `q` is present; **updated_at descending** in facet-only mode.
- Completeness badges (see §3) are shown on each row when a channel+locale is selected in the completeness facet context.

### URL State Persistence

Sync all search and filter state to the URL:

```
/products?q=signal&familyId=3&status=active&categoryId=7&channel=ecommerce&locale=en_US&completeness=lt:100
```

On page load, parse these params and restore the search state. This enables shareable search URLs and browser back/forward navigation.

---

## 2. Completeness Dashboard

### Location

Accessible as a **tab on the product list page** ("Completeness" tab alongside the default "All products" tab), or as a dedicated route `/products/completeness`. Prefer the tab approach to avoid adding a new top-level nav item.

### Dashboard Overview Card

At the top, a summary card showing catalog-wide completeness health:

| Metric | Description |
|--------|-------------|
| Total products | Count of root-level products (simple + models) |
| Fully complete | Count with `isComplete = true` (all applicable channels+locales at 100%) |
| Incomplete | Count with at least one channel+locale below 100% |
| Completion rate | Percentage complete / total |

The card has a **channel + locale selector** so the user can scope the dashboard to a specific channel+locale combination and see the most relevant completeness numbers. Default: first active channel + first active locale.

### Completeness Table

A table of root-level products with their completeness scores. Columns:

| Column | Content |
|--------|---------|
| Product | Identifier + family label |
| Status | Status badge |
| Completeness (per channel+locale) | Compact percentage badge per active channel and locale (one sub-column per channel, with locale rows within). |
| Variants | For models: count of variants, count complete. E.g. "12/15 variants complete". |
| Actions | "Edit" → product edit page. "View completeness" → product completeness panel. |

The completeness columns use color-coded badges:
- 100% — green badge
- 80–99% — yellow badge
- < 80% — red badge

Sort the table by **lowest score ascending** by default so the most incomplete products surface at the top. Allow sorting by any completeness column or by product identifier.

Pagination: cursor-based, 25 rows per page. Use `GET /api/v1/products?includeCompleteness=true&root=true` (Phase 29).

### Product Completeness Panel

Clicking "View completeness" on a row opens a **side panel** (or expands in-place) showing the full completeness breakdown for that product. The panel has two views:

**Summary view** — For models: shows per-variant completeness in a mini-table (variant identifier, channel+locale scores, a "complete" checkmark or ✗). Uses `GET /api/v1/products/:identifier/completeness/aggregate`.

**Detail view** — For a selected variant (or simple product): shows the per-channel, per-locale breakdown with:
- Percentage, complete/required counts.
- Whether locale reset applied.
- **Missing attributes list:** Each missing attribute shows its code, label, group, and a **"Go to field →"** link that navigates to the product edit form scoped to the correct channel+locale and scrolled to the correct attribute group.

The "Go to field →" link uses the deep-link format from Phase 29 §5:
```
/products/{variantIdentifier}?channel={channelCode}&locale={localeCode}#group-{groupId}
```

### Interactive Completeness in Product Edit Form

The completeness panel from Phase 12.1.1 already exists on the product edit page. In this phase, ensure:

- The completeness indicator (progress bar or badge) in the product edit header **updates live** when the user changes channel/locale scope or saves new attribute values — no page reload needed.
- Clicking the completeness indicator opens the missing attributes panel inline.
- Each missing attribute in the panel has a "Go to field →" button that scrolls to and highlights the attribute input in the form.

If these behaviors already exist from Phase 12.1.1, verify they work correctly; fix gaps if found.

---

## 3. Product List Completeness Indicators

### Inline Badges on Existing Product List

The existing product list (Phase 12) shows products in a table. Extend each row to show a completeness badge in a new "Completeness" column:

- The column is **hidden by default** and can be toggled on via a "Columns" button (standard pattern from Phase 17/20).
- When visible, it shows the completeness score for the **currently selected channel+locale** (from the global channel/locale picker at the top of the page, already implemented in Phase 12.1.1).
- Clicking the badge opens the completeness panel (same component as in §2).

### Channel+Locale Context

The completeness column uses the same channel+locale picker that the product edit form uses (URL-persisted `?channel=` and `?locale=` params, Phase 12.1.1). When the user changes the picker, all completeness badges on the current page re-fetch. Use React Query with the channel+locale as part of the query key so invalidation is automatic.

### Batch Loading

To avoid per-row completeness requests, load completeness for the entire current page in one request when the user enables the completeness column:

```
GET /api/v1/products?...(existing list params)...&includeCompleteness=true
```

This returns the same product list with compact `completeness` objects included (Phase 29 §4).

---

## 4. Component Architecture

The following new or updated components are needed:

| Component | Location | Description |
|-----------|----------|-------------|
| `<ProductSearchBar>` | `client/src/components/products/` | Search input with debounce, clear, loading state |
| `<FacetSidebar>` | `client/src/components/products/` | Collapsible sidebar with family, status, category, completeness facets |
| `<FacetChips>` | `client/src/components/products/` | Active filter chips with ✕ and "Clear all" |
| `<CompletenessTab>` | `client/src/pages/products.tsx` | Dashboard tab showing catalog-wide completeness |
| `<CompletenessTable>` | `client/src/components/products/` | Paginated table of products with per-channel completeness columns |
| `<CompletenesspPanel>` | `client/src/components/products/` | Side panel with product/variant completeness breakdown and deep links |
| `<CompletenessBadge>` | `client/src/components/products/` | Compact colored percentage badge (green/yellow/red) |
| `<MissingAttributesList>` | `client/src/components/products/` | List of missing attributes with "Go to field →" links |

Reuse existing components where possible: `<DataTable>`, `<Badge>`, `<Select>`, `<Pagination>`, `<Sheet>` (for the side panel), from the existing component library.

---

## 5. React Query Patterns

All data fetching uses React Query with consistent patterns established in earlier phases.

| Data | Query key | Endpoint |
|------|-----------|----------|
| Search results | `["/api/v1/products/search", { q, familyId, status, categoryId, channelCode, localeCode, completeness, cursor }]` | `GET /api/v1/products/search` |
| Product completeness (all scopes) | `["/api/v1/products", identifier, "completeness"]` | `GET /api/v1/products/:id/completeness` |
| Product completeness (single scope) | `["/api/v1/products", identifier, "completeness", channelCode, localeCode]` | `GET /api/v1/products/:id/completeness/:channel/:locale` |
| Model aggregate completeness | `["/api/v1/products", identifier, "completeness", "aggregate"]` | `GET /api/v1/products/:id/completeness/aggregate` |
| Product list with completeness | `["/api/v1/products", { ...filters, includeCompleteness: true }]` | `GET /api/v1/products?includeCompleteness=true&...` |

Set `staleTime: 0` for completeness queries (same as attribute values in Phase 12.1.1) to always refetch when the scope changes.

---

## 6. Navigation and Entry Points

- **Product list page (`/products`):** Add "Search" bar at top. Add "Completeness" tab. Add "Columns" toggle for completeness column.
- **Product edit page (`/products/:identifier`):** Verify completeness panel updates live on scope change (from Phase 12.1.1). Add "Go to field" links from missing attributes.
- No new top-level navigation items needed.

---

## 7. RBAC and Permissions

- **product:read:** Can view search results, completeness dashboard, and completeness panels. Cannot edit.
- **product:write:** Can use "Go to field" links to navigate to and edit missing attributes.
- The completeness dashboard and search are read-only by default; no write actions in this phase. The "Go to field" deep link navigates to the product edit page, which enforces its own RBAC.

---

## 8. Exit Criteria

- [ ] Search bar on `/products` calls `GET /api/v1/products/search`, debounces 300ms, shows ranked results with facet counts.
- [ ] Facet filters (family, status, category, completeness + channel+locale) work independently and in combination; active filters displayed as chips.
- [ ] Facet filter state and search query synced to URL; browser back/forward navigates search state.
- [ ] Completeness tab on `/products` (or `/products/completeness`) shows catalog-wide summary card and paginated completeness table with color-coded badges.
- [ ] Product completeness panel shows per-variant breakdown and missing attributes with "Go to field →" deep links.
- [ ] Clicking "Go to field" navigates to the product edit form with correct channel+locale and scrolls to the correct attribute group.
- [ ] Product list "Completeness" column (toggle via Columns button) shows inline completeness badges for the current channel+locale context; batch-loaded for the full page.
- [ ] Completeness badge in product edit header updates live on scope change (no page reload).
- [ ] All new components match existing design system (brand colors, dark mode from Phase 20.5).

---

## 9. Notes

- **No new top-level nav item.** The search and completeness features slot into the existing Products section. This keeps the nav clean and avoids Phase 34 polish work.
- **Facet sidebar collapse.** On mobile, the sidebar should collapse to a "Filters" button that opens a bottom drawer. On desktop, it is always visible in a collapsible panel on the left of the product table.
- **Category facet.** The category facet is a single-select tree (pick one category node, which also includes all descendants via the Phase 28 recursive filter). Multi-category filter would require OR logic in the backend — defer to post-MVP if needed.
- **Completeness column performance.** Using `GET /api/v1/products?includeCompleteness=true` triggers `calculateCompletenessOverview` on the server for the current page. With Phase 29's batch optimization, this adds one extra round-trip (per page) but no per-product queries. Monitor response time; if > 500ms for 25 products, investigate the batch function.
- **Phase 34 (Final Polish) dependency.** This phase delivers the feature. Phase 34 handles edge cases, accessibility, and flow polish across the whole app — including refining these screens if needed.
