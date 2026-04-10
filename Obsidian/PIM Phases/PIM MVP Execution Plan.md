# PIM MVP — Execution Plan

**Timeline:** 10 weeks
**Scope:** MVP items only (see [[PIM MVP Prioritization]])
**Last Updated:** 2026-03-10

---

## Status Dashboard

| Week | Focus | Status | Progress | Notes |
|------|-------|--------|----------|-------|
| 1–2 | Foundation | Complete | 9/9 | All tasks 0–7 done per Replit Phases 00–07.5 (see Replit Phases/Week 1-2). |
| 3 | Products & Related Structures | Complete | 8/8 | All implemented except EP 13 (Service-to-Service Auth deferred). Replit Phases 08–12, 12.1, 12.1.1, 12.2–12.3, 12.5–12.9 done. Phase 12.4 deferred; API key auth (20.6) supersedes. |
| 4 | Relationships, Validation & API Hardening | Complete | 6/6 | Phases 15–19, 20, 20.5 complete. Phase 20.6 (API Key Auth) addendum when needed for machine access. |
| 5 | Integration Framework (no scheduling) | Complete | 4/4 | Phases 21–24 complete. |
| 6 | Akeneo as Integration & Real Data | Complete | 6/6 | 25.1-25.5 and 26 complete. |
| 7 | GraphQL, Search & Completeness | Complete | 4/4 | Phases 27–30 complete. |
| 8 | MCP Server | Not Started | 0/1 | Phase 31 (renumbered from 35, moved up — people are waiting on it). Builds on REST (Week 4) and GraphQL (Week 7). |
| 9 | Scheduling | Not Started | 0/3 | Phases 32–34 (renumbered from 31–33). Phase 23 (run engine) gains scheduled full runs. |
| 10 | Final Polish & Hardening | Not Started | 0/1 | Phase 35 (renumbered from 34): UI polish, import verification, workflow consistency. |

**Overall:** 37 complete (through Phase 30); 4 phases remaining (31, 32–34, 35).
**Current Week:** 8 (next: MCP Server — Phase 31)
**Blockers:** None

**Replit Phases (Week 1–2):** 00, 01, 02, 03, 04, 04.5, 05, 06, 07, 07.5 — all complete.

**Replit Phases (Week 3):** 08–12, 12.1, 12.1.1, 12.2–12.3, 12.5–12.9 complete; 12.4 (Service-to-Service Auth) superseded by Phase 20.6 (API Key Auth) for machine access.

**Replit Phases (Week 5):** 21, 22, 23, 24 complete.

**Replit Phases (Week 6):** 25.1, 25.2, 25.3, 25.4, 25.5, 26 complete.

**Replit Phases (Week 7):** 27, 28, 29, 30 complete.

> Update the Status column to `In Progress`, `Complete`, or `Blocked` as work progresses. Increment Progress counts as individual items are checked off below. Add notes for anything that needs attention — scope changes, delays, decisions made with Replit, etc.

---

## Week 1–2: Foundation

Nothing else works without this layer. Auth, the core data model, and the UI shell get built in parallel.

- [x] **0. Project Setup & Conventions** — Tech stack, project structure, database conventions, API conventions, TypeScript conventions, and environment configuration established.
- [x] **1. Microsoft Entra ID Auth + RBAC Schema** — Get login working and the permission tables in place. Everything behind this wall from day one, even if permissions are loosely enforced initially. The Entra app registration should happen immediately.
- [x] **2. Attributes, Attribute Types & Attribute Groups** — This is the atomic unit of the whole system. Every other entity (products, entities, assets) stores data through attributes. All 15 types need their storage patterns defined. Complex objects stored as stringified JSON to keep things simple. The validation rule schema (regex, min/max, unique, file type restrictions) should be defined as metadata on attributes here — enforcement in write paths comes later in Week 4.
- [x] **3. Locales, Channels & Scoping Infrastructure** — How values are stored per locale and per channel. Fallback chains (locale, and channel if implemented) go here. **Phase 12.1 adds:** Optional "completeness reset" per locale so completeness for a locale does not count values only from its fallback (e.g. en full does not make de "complete" until de has its own content).
- [x] **4. Families** — Define which attributes a product gets and which are required per channel/locale. Products can't exist without families. (Note: variant level assignment originally lived here but was superseded by Task 4.5.)
- [x] **4.5. Family Variant Definitions** — Replaces the fixed `variant_level` on `family_attributes` with a flexible variant definition system. Each family can have multiple named variant definitions, each specifying its own axis hierarchy (with multi-axis-per-level support) and attribute-level assignments. Products reference both a family and a variant definition.
- [x] **5. Categories & Trees** — The hierarchical taxonomy. Relatively independent from families, so this can be built in parallel.
- [x] **6. Audit History Table Schema** — Create the audit history table structure (resource type, resource ID, attribute code, old value, new value, user, timestamp) now so no changes are lost from the moment data starts being created. Wiring into each entity's create/update/delete paths happens incrementally as those entities are built in subsequent weeks.
- [x] **7. UI: Application Shell & Foundation Screens** — Core layout, navigation structure, and Entra login flow. Admin screens for managing attributes, attribute groups, families, categories, channels, and locales. Keep the interface clean and task-oriented — every screen should make the most common action obvious and the complex actions discoverable without clutter. REST endpoints for these entities should be built alongside the UI to enable immediate testing.

**Exit criteria:** Can create attributes (with validation rule definitions), group them, define families with multiple variant definitions, build category trees, and have channels/locales configured — all behind Entra auth, with the audit table ready to capture changes. A working UI provides access to all foundation management tasks.

---

## Week 3: Products & Related Structures

Now that the foundation exists, build the core entities that sit on top of it. REST endpoints are built alongside each entity — not deferred.

**Built (Replit Phases):** 08 Products, 09 Product Attribute Values, 09.5 Variant Ordering, 09.6 Terminology-Locales-Identifiers, 10 Product-Categories-Status, 11 Completeness, 12 Product UI, **Phase 12.1** ([[12.1-Product-Inheritance-Completeness-Corrections]]), **Phase 12.1.1** ([[12.1.1-Post-Implementation-Fixes-Value-Trace-UX]]), **Phase 12.2** ([[12.2-Entities]]), **Phase 12.3** ([[12.3-Asset-Manager]]), **Phase 12.5** ([[12.5-Entity-Management-UI]]), **Phase 12.6** ([[12.6-Asset-Management-UI]]), **Phase 12.7** ([[12.7-Asset-Type-Management-UI]]), **Phase 12.8** ([[12.8-Entity-Type-Management-UI]]), **Phase 12.9** ([[12.9-Rollback-CamelCase-To-Numeric-Ids]]). **Deferred:** **Phase 12.4** ([[12.4-Service-to-Service-Auth]]).

- [x] **8. Product Types & Variant Hierarchy** — All products use a variant definition (0-axis = Top Model with no children; 1+ axes = variant hierarchy). N-level variant axes, parent/child relationships. The schema supports arbitrary depth without being hardcoded. Attribute inheritance and override cascading — children can override values, and those overrides cascade to variants below them. Audit history wiring for products starts here. *(Replit Phase 08; simple product type removed per 09.6.)*
- [x] **9. Product Attribute Values — Read, Write & Variant Inheritance** — Service layer and API endpoints for reading, writing, and deleting attribute values on products. Integration point between products and the EAV system. Variant inheritance resolution (first available value: self → parent → … with locale/channel fallback), type-specific validation for all 15 attribute types, axis value immutability, transactional writes with audit logging. *(Replit Phase 09.)* **Corrections in Phase 12.1:** Override at any level below owner (not only at owner level); first-available value in unison with locale and channel fallback; source annotation (own / inherited / fallback).
- [x] **9.5. Variant Ordering, camelCase Enforcement & Auto-Incrementing Sort Orders** — Persistent display order for child products (sort_order column + reorder endpoint), camelCase regex validation on all entity code fields, and auto-incrementing default sort orders across all ordered collections. *(Completed in Replit Phase 09.5; spec [[09.5-Variant-Ordering-CamelCase-AutoIncrement]].)*
- [x] **10. Product Status Field** — Draft/Active/Archived. Simple to add alongside products. *(Replit Phase 10: Product-Categories-Status covers status + product–category assignment.)*
- [x] **11. Entities** — Same pattern as products (schema with typed attributes, localizable/channelizable) but standalone. Audit history wired in. *(Replit Phase **12.2** [[12.2-Entities]] completed 2026-02-25.)*
- [x] **12. Asset Manager — Basic Structure** — Asset families/types, attribute system, single and list linking to products. Leverages the attribute infrastructure already built. Audit history wired in. *(Replit Phase **12.3** [[12.3-Asset-Manager]] completed 2026-02-25.)*
- [ ] ~~**13. Service-to-Service Auth**~~ — **Deferred.** OAuth 2.0 Client Credentials Flow for machine-to-machine access. Superseded by Phase 20.6 (API Key Auth) for Akeneo import and other machine consumers. Revisit only if OAuth2 client credentials are required later. *(Replit Phase **12.4** [[12.4-Service-to-Service-Auth]].)*
- [x] **14. UI: Product, Entity & Asset Management** — Product creation and editing with a clear variant hierarchy view that makes N-level depth manageable. Entity and asset management screens following the same interaction patterns as products for consistency. The variant editor should make inheritance visible — users need to see at a glance which values are inherited vs. overridden. REST endpoints for all Week 3 entities are testable through the UI by end of week.
  - [x] **Product UI (Replit Phase 12)** — Product list (filters, bulk actions, pagination), create flow (variant definition: 0-axis Top Model vs 1+ axes with variants), edit page (attributes, variants tab, categories tab), channel/locale picker, completeness indicators, inherited values, all 15 attribute input types. Verified complete per [[12-Product-UI-Verification]]. **Phase 12.1:** First available fallback value with inheritance/fallback indicator; override where permitted; interactive completeness. **Phase 12.1.1:** State reset on product navigation; override reset button + override-stats badge; value resolution trace modal; URL-persisted channel/locale (`?channel=&locale=`) and scope-preserving links; fresh data on scope change (staleTime: 0, sync guard); axis attributes filtered from editable list; locale fallback source indicators; variant definition edit protection and level migration; optional categories; recursive descendant completeness. See [[12.1.1-Post-Implementation-Fixes-Value-Trace-UX]].
  - [x] **Entity management UI** — List, create, edit entities with attribute values and channel/locale. *(Replit Phase **12.5** [[12.5-Entity-Management-UI]]; depends on 12.2.)*
  - [x] **Asset management UI** — List, create, edit assets with attribute values (including media upload) and channel/locale. *(Replit Phase **12.6** [[12.6-Asset-Management-UI]]; depends on 12.3.)*
  - *Structure admin UI:* Entity Types ([[12.8-Entity-Type-Management-UI]]), Asset Types ([[12.7-Asset-Type-Management-UI]]). **Phase 12.9** ([[12.9-Rollback-CamelCase-To-Numeric-Ids]]): All structure entities use numeric `id` PK; no code in UI/API.*

**Exit criteria:** Can create products with N-level variants, entities, and assets via UI and API. Inheritance and overrides work correctly. All three entity types use the shared attribute system. UI supports full CRUD on products, entities, and assets. Entity Types and Asset Types admin UIs complete. Structure entities (attributes, families, categories, channels, entity types, asset types, etc.) use numeric IDs; no code in UI. ~~Service-to-service auth is functional.~~ *(Deferred to pre–Week 7.)* **Week 3 complete.**

---

## Week 4: Relationships, Validation & API Hardening

Wire everything together, enforce data quality, and tighten access control.

- [x] **15. Product Associations & Relationships** — One-way, two-way, variant-level and product-level, with ordering. Needs products and entities to exist first. Audit history wired in. *(Replit Phase **15** [[15-Product-Associations-Relationships]]; completed 2026-02-27.)*
- [x] **16. Audit History Wiring — Completion Pass** — By this point, audit logging should already be wired into most entities incrementally. This is the week to verify full coverage across every create/update/delete path and close any gaps. *(Replit Phase **16** [[16-Audit-History-Wiring-Completion]]; completed 2026-02-28.)*
- [x] **17. API Consistency & Hardening** — REST endpoints have been built alongside each entity in Weeks 1–3. This week is for ensuring the full surface area is consistent in its patterns, well-structured, and properly error-handled. Standardize response formats, error codes, and pagination. *(Replit Phase **17** [[17-API-Consistency-Hardening]]; completed 2026-02-28.)*
- [x] **18. Attribute Validation Enforcement** — The validation rule schema was defined in Weeks 1–2 as metadata on attributes. Now enforce those rules in all API write paths: regex, min/max, unique constraints, file type restrictions, max file size. *(Replit Phase **18** [[18-Attribute-Validation-Enforcement]]; completed 2026-02-28.)*
- [x] **19. RBAC Enforcement Tightening** — Move from loosely enforced permissions to strict enforcement. Lock down access by resource type, locale, channel, and attribute group. Verify that every API endpoint respects role boundaries. *(Replit Phase **19** [[19-RBAC-Enforcement-Tightening]]; completed 2026-02-28.)*
- [x] **20. UI: Relationships, Audit & Validation Feedback** — Relationship management UI for creating and viewing product associations with clear directionality indicators for one-way vs. two-way. Audit history viewer showing change timeline per entity. Inline validation feedback on all forms — users should see exactly what's wrong and why before they submit, not after. *(Replit Phase **20** [[20-UI-Relationships-Audit-Validation-Feedback]]; Phase **20.5** [[20.5-Brand-Colors-Dark-Mode]]: Brand color system, three-state dark mode toggle, EARL branding — complete.)*

**Exit criteria:** Full CRUD via REST on all entity types with consistent patterns. Relationships work in both directions. Every data change is audit-logged. Validation rules block bad data at the API boundary. RBAC is strictly enforced. UI provides clear feedback on validation errors and relationship state. **Week 4 complete through Phase 20.5.**

---

## Week 5: Integration Framework (no scheduling)

Pluggable connectors — inbound and outbound — with type registry, instances, mapping, and run engine. Built before Akeneo so the importer is the first integration type. Scheduling (Phase 32) and scheduled runs in the run engine come in Week 9.

- [x] **21. Integration Framework — Type Registry & Instances** — Integration types are pluggable connectors registered in the system. Each type defines its **direction** (inbound e.g. Akeneo Import, or outbound e.g. Shopify, FBA), connection requirements, available field mappings, and run capabilities. Integration instances represent a single connection — one instance per source or endpoint. Multiple instances of the same type are fully independent. Design the registry and run engine to support both inbound (import) and outbound (push) from the start. *(Replit Phase **21** [[21-Integration-Framework-Type-Registry-Instances]].)*
- [x] **22. Integration Framework — Mapping & Filtering** — Per-instance field mapping: for **outbound** types, PIM attributes → target system fields; for **inbound** types, source fields → PIM attributes. Support value transformation (unit conversion, concatenation, conditional logic, cleaning). For outbound, product filtering rules per instance (family, category, channel, status). For inbound, scope/filter config as needed (e.g. which families or sources to import). *(Replit Phase **22** [[22-Integration-Framework-Mapping-Filtering]].)*
- [x] **23. Integration Run Engine & Logging** — On-demand runs first: single-record push (outbound), full run (outbound), and full (or incremental) import (inbound). Run history table: execution status, timestamps, record counts per instance. Per-record logs: success, failure, skip with reasons. Retry logic for transient failures. **Scheduled full runs** (using Phase 32) are added in Week 9 when scheduling exists. *(Replit Phase **23** [[23-Integration-Run-Engine-Logging]]; completed 2026-03-03.)*
- [x] **24. UI: Integration Management** — Manage integration instances and their configurations, build field mappings visually, trigger on-demand runs, and drill into run history and per-record logs. No scheduling UI here — that is Phase 34 in Week 9. *(Replit Phase **24** [[24-UI-Integration-Management]]; completed 2026-03-04.)*

**Exit criteria:** Integration framework supports creating typed instances (inbound and outbound), configuring per-instance mappings and filters, and running on-demand pushes/imports with full logging. UI provides clear visibility into instances, runs, and logs. *(Phases 21–24 complete. **Week 5 complete.**)*

---

## Week 6: Akeneo as Integration & Real Data

Akeneo Data Import is implemented as an **integration type** (inbound). Same run engine, run history, and per-record logging as other integrations. Gets real data into the PIM for REST testing and GraphQL work in Week 7. Prerequisite: machine auth (Phase 20.6 API Key Auth or Phase 12.4 Service-to-Service Auth).

- [x] **25. Akeneo Import (integration type)** — Split into 5 sub-phases for accuracy. Every run is a full replacement — the PIM is wiped and re-imported fresh. Runs via API key auth (Phase 20.6).
  - [x] **25.1** Connector scaffold, auth & pre-import wipe
  - [x] **25.2** Product attribute structure (attribute groups, attributes, families) — [[25.2-Akeneo-Product-Attribute-Structure]]
  - [x] **25.3** Entity types & entities (Reference Entities — EE) — [[25.3-Akeneo-Entity-Types-Entities]]
  - [x] **25.4** Asset types & assets (Asset Manager — EE)
  - [x] **25.5** Categories, products & variants with full attribute values
- [x] **26. Import monitoring** — Import progress and monitoring are provided by the Integration UI (Phase 24: run history, per-record logs). No separate import dashboard required unless Akeneo-specific views are needed; otherwise use the same "run history and logs" UX as for other integrations.

**Exit criteria:** Akeneo data can be imported through the integration framework. Run history and logs are visible in the Integration UI. Real data is available for REST and GraphQL testing. **Week 6 complete.**

---

## Week 7: GraphQL, Search & Completeness

The read-side experience and data quality layer. Real data from the Week 6 Akeneo import improves testing and UX.

- [x] **27. GraphQL API** — The read-side query layer built on top of the REST/data layer. Specific query patterns: parent with variants, by channel/locale, default to all channels/locales. Cursor-based pagination and field selection. **GraphiQL** in-browser IDE served at `/graphql` (dev + staging; disabled or auth-gated in production).
- [x] **28. Full-Text Search** — Postgres full-text search across attributes, labels, SKUs. Faceted filtering by family, category, completeness, channel, status. Can swap to Elasticsearch later if needed.
- [x] **29. Completeness Calculation** — Per **variant**, per channel, per locale; locale fallback chain may have a "completeness reset" so a locale only counts values stored in that locale (not from fallback). Depends on families (required attributes), channels, and locale config (Phase 12.1). Surface as percentage; aggregate per-variant/per-locale scores for overall product score. Flag missing fields in the UI and API.
- [x] **30. UI: Search, Completeness & Data Exploration** — Search interface with faceted filtering. Completeness dashboard: per-channel/per-locale (and per-variant) scores, **interactive** — user can identify what is incomplete (which variant, locale, channel, attributes) and drill into or deep-link to the missing fields (Phase 12.1). Deep links and variant breakdown links preserve channel/locale scope (Phase 12.1.1). Completeness indicators visible wherever products are listed.

**Exit criteria:** GraphQL queries return deeply nested product data in a single request. Search returns relevant results with faceted filters. Completeness scores are calculated and surfaced. UI provides a fast, filterable product search and clear completeness visibility. **Week 7 complete.**

---

## Week 8: MCP Server

Expose the PIM as a first-class data source for LLMs and AI tooling. Moved up from Week 10 (renumbered from Phase 35 → **Phase 31**) — builds directly on the REST (Week 4) and GraphQL (Week 7) layers that are now complete, and demand is high.

- [ ] **31. Built-in MCP Server** — Implement a Model Context Protocol (MCP) server embedded in the EARL Express app that serves as an **API reference and schema guide** for LLMs. Does not return live data — exposes the documentation, schemas, and structural definitions that allow an LLM to write correct REST and GraphQL calls against EARL without guessing. Via:
  - **Resources** — REST endpoint docs (`earl://rest/endpoints/...`), full GraphQL SDL and per-type definitions (`earl://graphql/...`), data model guides for all 15 attribute types, scoping rules, pagination conventions, and curated working examples (`earl://examples/...`).
  - **Tools** — `list_rest_endpoints`, `get_rest_endpoint`, `get_graphql_type`, `list_graphql_queries`, `get_attribute_type_spec`, `search_api_docs`, `get_example`, `validate_graphql_query`, `get_scoping_guide`, `get_pagination_guide`.
  - **No auth required** — equivalent to serving API documentation. No live data is exposed.
  - **Transport** — HTTP/SSE (Streamable HTTP transport from `@modelcontextprotocol/sdk`).

**Exit criteria:** Cursor or Claude Desktop connects, asks how to write a GraphQL query for product variants, uses `get_graphql_type` + `validate_graphql_query`, and produces a correct query without hallucinating field names or types.

---

## Week 9: Scheduling

Scheduling infrastructure and UI. Integration run engine (Phase 23) gains scheduled full runs once this is in place. *(Renumbered: was Phases 31–33, now **Phases 32–34**.)*

- [ ] **32. Scheduling Infrastructure** — Scheduled actions table (entity type, entity ID, action type, change payload, scheduled datetime, execution status, created by). A background job processor evaluates pending actions at their target time and executes them against the same write paths used for immediate changes. Users can schedule any data mutation: product status activation/deactivation, attribute value updates, association changes, category reassignments. Scheduled actions can optionally trigger integration pushes on execution — e.g., activate a product at a scheduled time AND push it to the relevant channels.
- [ ] **33. Scheduled full runs** — Wire Phase 23 (run engine) to the scheduling infrastructure so integration instances can run "scheduled full runs" in addition to on-demand runs.
- [ ] **34. UI: Scheduling Management** — Create, view, edit, and cancel scheduled actions with a timeline view of upcoming changes. Users see at a glance what's pending and when. Lightweight — common case (schedule one change, push one product) takes seconds.

**Exit criteria:** Any data change can be scheduled for future execution with optional integration push triggers. UI provides clear visibility into what's scheduled.

---

## Week 10: Final Polish & Hardening

*(Renumbered: was Phase 34, now **Phase 35**.)*

- [ ] **35. UI: Final Polish** — Verify that all screens flow logically, navigation is consistent, and the most common workflows (find a product, edit it, check completeness, schedule a change, push to an integration, view history) feel fast and intuitive. Bug fixes, edge cases, and verifying the Akeneo import produces clean data.

**Exit criteria:** Core system is stable and ready for use. UI is polished and all primary workflows are smooth.

---

## Key Risks

- **Weeks 1–2 are the highest risk.** If the attribute storage model and scoping infrastructure aren't rock solid, everything downstream wobbles. Worth spending extra time here even if it means compressing later weeks.
- **Integration framework must support inbound from day one.** The type registry and run engine are designed for both inbound (e.g. Akeneo Import) and outbound (e.g. Shopify) connectors. Building Akeneo as the first integration type will stress-test the framework early — budget time for framework adjustments as the import surfaces requirements (mapping, filtering, run semantics) that outbound-only design might have missed.
- **N-level variant depth** is a meaningful complexity increase over a fixed 2-level model. The recursive/tree-based approach needs careful design in week 3.
- **UI simplicity vs. data model complexity.** The underlying data model (N-level variants, localizable + channelizable attributes, inheritance cascading) is inherently complex. The UI must absorb that complexity without exposing it all at once. Progressive disclosure — showing the simple case by default and revealing advanced options on demand — will be critical. If the UI becomes confusing, adoption will suffer regardless of how solid the backend is.
- **Scheduling and integration share infrastructure but have distinct failure modes.** A failed scheduled action (e.g., activate a product) is recoverable — retry it. A failed integration run that partially succeeded (e.g., 3 of 8 stores updated, or import partially complete) is harder to recover from. The run engine needs clear per-instance, per-record status tracking so partial failures are visible and retryable without re-running everything. *(Scheduling is Phases 32–34, Week 9.)*
- **Integration framework scope creep.** The framework is connector-agnostic; the first real connector (Akeneo Import, then later e.g. Shopify) will surface requirements the framework didn't anticipate. Budget time for framework adjustments when building the first connectors.
