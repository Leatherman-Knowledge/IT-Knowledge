# PIM Build — MVP vs. Phase 2 Prioritization

**Timeline:** ~7 weeks
**Guiding principles:** Anything that touches database schema/structure goes into MVP to avoid painful migrations later. Anything that's additive UI polish, advanced automation, or AI-powered goes to Phase 2+.

---

## MVP (Must Ship)

### Product Data Model (all DB-structural)

- **Product Types** (§1.1) — Simple products, 1-axis variants, 2-axis variants, and support for additional variant axes beyond two. This is the backbone of the entire system. The parent/child hierarchy and how attributes cascade must be right from day one, and the schema needs to accommodate N-level depth rather than being hardcoded to two.
- **Attribute Inheritance** (§1.2) — Values flow down the variant tree. Children can override inherited values if desired, and those overrides cascade to the variants below them. Baked into how products store and resolve data.
- **Families** (§2) — Define which attributes a product has, which variant level they live at, and which are required per channel/locale. Core structural concept — everything references families.
- **Attributes — All Types** (§4.1, §4.2) — All 15 attribute types: text, textarea, boolean, number, measurement, price, date, single select, multi select, media/file, asset list, entity reference, product reference, and table. These define the column types and storage patterns in the DB. Leaving any out means schema changes later. Complex objects should be stored as stringified objects for simplicity.
- **Attribute Groups** (§4.1) — Organizational grouping for attributes (e.g., "Marketing," "Technical"). Simple to implement now, annoying to retrofit permissions around later.
- **Localization & Locale Fallback** (§4.3) — Attributes can be localizable (separate values per locale). Implement the fallback chain (e.g., fr_CA → fr_FR → en_US) now — this affects how every value is stored and resolved. Confirmed as needed in the Ideas doc.
- **Channelizable Attributes / Channels** (§5) — Attributes can vary per channel. Each channel defines its active locales, currencies, and category tree root. Combined with localizable, this creates the full value matrix. Fundamental to storage design.
- **Categories & Trees** (§3) — Hierarchical taxonomies, products can belong to multiple categories across trees. Tree structure is DB-structural.
- **Entities** (§7) — Standalone structured records (brand profiles, material specs, etc.) with their own schemas. These are referenced by products via a dedicated attribute type — the reference relationship needs to exist in the DB from the start. Entity attributes should be localizable and channelizable.
- **Product Associations & Relationships** (§9) — Named relationship types (cross-sell, upsell, substitute, etc.) with one-way and two-way support. The Ideas doc specifically calls out variant-level and product-level associations with ordering. The relationship table structure needs to be in MVP.
- **Asset Manager — Basic Structure** (§8) — Asset families/types with their own attributes, linked to products via asset list attributes or single asset attributes. The asset storage model and linking mechanism is DB-structural. Full asset attribute richness can grow over time, but the tables need to exist.
- **Product Status Field** (§16) — Draft, Active, Archived/Disabled. Simple enum on the product, but it gates what downstream systems see. Must be in the schema from the start.
- **Audit History Table** (§11.1) — Attribute-level change history (resource type, resource ID, attribute code, old value, new value, user, timestamp). The history table structure must exist from day one so no changes are lost. Queryable via API. *Timing: Table schema created in Week 1–2. Wiring into each entity's write paths happens incrementally as those entities are built (Weeks 2–4), with a completion pass in Week 4.*
- **Enforce camelCase for Codes** (Ideas doc) — Convention enforcement on attribute/family/category codes. Easier to enforce from the start than to migrate later.
- **Auto-incrementing Values** (Ideas doc) — Support for auto-generated sequential values. Needs a DB mechanism (sequence or counter table).
- **Variant Ordering** (Ideas doc) — Ability to order variants within a product. Requires a sort/position field in the variant table.

### Authentication & Permissions

- **Microsoft Entra ID Auth** (§17) — All authentication through a dedicated Entra app registration. JWT validation, token refresh, claim mapping. Non-negotiable for a Leatherman internal tool.
- **Service-to-Service Auth** (§17.3) — OAuth 2.0 Client Credentials Flow for machine-to-machine access (import jobs, integrations). Needs its own Entra app registration. *Timing: Week 3, so it's available before integrations or the Akeneo import hit the API.*
- **RBAC — Basic** (§13) — Role-based access control. At minimum: permissions by resource type, by locale, by channel, and by attribute group. The Ideas doc calls out simplified permissions and attribute group viewer/edit permissions. The permission model is DB-structural. *Timing: Schema and loose enforcement in Week 1–2. Strict enforcement tightened in Week 4 after the full API surface exists.*

### API Layer

- **REST API** (§12.2) — CRUD on all core entities (products, families, attributes, categories, assets, entities, channels, locales). This is how the UI and integrations talk to the system.
- **GraphQL API — Core Queries** (§12.1) — Read-side API. Must support querying a parent product with variants, querying by channel/locale, and defaulting to all channels/locales. The Ideas doc is very specific about these query patterns. Cursor-based pagination and field selection.

### Import — Akeneo Migration

- **Akeneo Data Import** — Ability to quickly and accurately run imports from Akeneo to populate data tables in the new PIM. Some advanced cleaning and transformation logic will be required to massage values into the new schema properly. Advanced and thorough logging for this process is critical for success — every skipped record, transformed value, and validation failure needs to be traceable.

### Search — Basic

- **Full-Text Search** (§14) — Search across product attributes, labels, and SKUs. Basic faceted filtering by family, category, completeness, channel, status. Doesn't need Elasticsearch on day one — a solid Postgres full-text search can work for MVP and be swapped later.

### Completeness — Basic

- **Completeness Calculation** (§6) — Per product, per channel, per locale. Based on required attributes from the family. Surface as a percentage and flag missing fields. The calculation logic and storage for completeness scores is MVP.

### Validation Rules — Basic

- **Attribute Validation** (§4.4) — Regex, max length, min/max numeric, allowed file types, max file size, unique value enforcement. These constrain what goes into the DB, so the validation framework should be in place. *Timing: Validation rule schema defined as metadata on attributes in Week 1–2. Enforcement in API write paths in Week 4.*

### Scheduling System

- **Scheduled Actions Infrastructure** — Scheduled actions table storing entity type, entity ID, action type, change payload, target datetime, execution status, and created-by user. A background job processor evaluates pending actions at their target time and executes them. This is DB-structural — the concept of future-dated changes needs its own storage, and retrofitting it later means every write path has to be revisited. *Timing: Week 6.*
- **Schedulable Data Changes** — Users can schedule any data mutation: product status activation/deactivation, attribute value changes, association modifications, category reassignments. The scheduling system wraps the same write paths used for immediate changes, just with deferred execution. This means a user can prepare a full product launch (activate product, update descriptions, assign categories) and have it all go live at a precise time.
- **Integration Push Triggers** — Scheduled actions can optionally trigger integration pushes on execution. For example: activate a SKU at a scheduled time AND push it to the relevant Shopify stores automatically. Connects the scheduling system to the integration framework. *Phase 2 extensions: recurring schedules, conditional triggers (e.g., only push if completeness > 90%).*

### Integration Framework

- **Integration Type Registry** — Pluggable integration types (Shopify, FBA, etc.) registered in the system. Each type defines its connection requirements, available field mappings, and push/pull capabilities. New types can be added without modifying the core framework. The registry pattern is DB-structural — it determines how all future integrations are stored and configured.
- **Integration Instances & Configuration** — Each instance represents a single connection to a single store or endpoint. Multiple instances of the same type are fully independent — 8 Shopify stores means 8 instances, each with its own credentials, endpoint URLs, and instance-specific settings. This multi-instance architecture must be in the schema from the start.
- **Field Mapping Engine** — Per-instance mapping from PIM attributes to target system fields. Supports value transformation (unit conversion, field concatenation, conditional logic). Product filtering rules per instance determine which products flow to which integration — by family, category, channel, status, or custom criteria. The mapping and filtering tables are DB-structural.
- **Run Engine, History & Logging** — On-demand single product push, on-demand full run, and scheduled full runs (using the scheduling infrastructure). Run history table tracking execution status, timestamps, and record counts per instance. Detailed per-record logs capturing every success, failure, and skip with reasons. Retry logic for transient failures. The run/log tables must exist from the start so no integration activity goes untracked.

### UI — Core Experience

The UI is built incrementally alongside the backend, not as a separate phase. Every backend feature should have a corresponding UI by the end of its week.

- **Application Shell & Navigation** — Core layout, navigation structure, and Entra login flow. The navigation should be simple and flat — users shouldn't need to click through multiple layers to reach common tasks. Built in Week 1–2.
- **Foundation Admin Screens** — Management interfaces for attributes, attribute groups, families, categories, channels, and locales. Task-oriented design where the most common action is obvious and complex actions are discoverable without clutter. Built in Week 1–2.
- **Product, Entity & Asset Management** — Product creation and editing with a clear variant hierarchy view that makes N-level depth manageable. Entity and asset screens follow the same interaction patterns for consistency. The variant editor must make inheritance visible — users see at a glance which values are inherited vs. overridden. Built in Week 3.
- **Relationship & Audit Views** — Relationship management with clear directionality indicators for one-way vs. two-way associations. Audit history viewer showing change timeline per entity. Built in Week 4.
- **Inline Validation Feedback** — All forms surface validation errors inline — users see what's wrong and why before submission, not after. Built in Week 4.
- **Search & Completeness** — Search interface with faceted filtering that allows narrowing to the right product in a few clicks. Completeness indicators visible wherever products are listed, with drill-down into what's missing per channel/locale. Built in Week 5.
- **Scheduling Management** — Create, view, edit, and cancel scheduled actions with a timeline view of upcoming changes. Users should see at a glance what's pending and when. The common case — schedule one change for one product — should take seconds. Built in Week 6.
- **Integration Management** — Manage integration instances and their configurations, build field mappings visually, trigger on-demand runs (single product or full), and drill into run history and per-record logs. Each instance's status (last run, success rate, pending items) should be visible at a glance. Built in Week 6.
- **Import Monitoring** — Real-time import progress dashboard with error counts and detailed logs. Built in Week 7.
- **Design principles:** Progressive disclosure (simple case by default, advanced options on demand), consistent interaction patterns across all entity types, and fast navigation to the most common workflows: find a product, edit it, check completeness, schedule a change, push to an integration, view history.

---

## Phase 2 (Post-MVP)

### Import / Export — General Purpose

- **Bulk Import** (§10) — Products, families, attributes, categories, and assets via CSV, XLSX, or JSON. Support create/update/delete, dry-run validation, and line-level error reporting. The Akeneo migration import covers the initial data load; this is the ongoing general-purpose import system.
- **Bulk Export** (§10) — Export in the same formats, filterable by family, category, channel, completeness, locale.
- **Scheduled/Automated Exports** (§10) — Cron-based or triggered exports.

### Enhanced Data Management

- **Bulk Operations UI** (§15) — Bulk edit, bulk category assignment, bulk family reassignment, bulk delete from the UI. The API should support this in MVP, but the polished UI workflows can come in Phase 2.
- **Saved Filters / Views** (§14) — Save and name common search queries. Nice productivity feature, not structural.
- **Moving Products Between Families** (Ideas doc) — Family reassignment with attribute remapping. Complex edge cases, can be Phase 2.
- **Improved Product Viewing** (Ideas doc) — Enhanced UI for browsing/viewing products.
- **External Catalogs** (Ideas doc) — Support for external catalog usage. Needs more scoping.

### Workflow & Automation

- **Approval Workflow** (§16) — Configurable review/approval before publishing to a channel. Adds workflow state machine complexity. Status field is MVP; the workflow engine is Phase 2.
- **Webhooks & Events** (§12.3) — Real-time event notifications (product created/updated/deleted, completeness changed, etc.) with retry logic and delivery logging. Very useful for integrations but not required for core PIM functionality.

### Integration Connectors

The integration framework (MVP) provides the type registry, instance configuration, mapping engine, run engine, and logging. Phase 2 is where specific connectors are built to plug into that framework. The first connector will likely surface framework adjustments — budget time for that.

**Shopify Connector**

- **Shopify Integration Type** (Ideas doc) — The first connector built on the integration framework. Defines Shopify-specific connection requirements (API credentials, store URL), available field mappings (PIM attributes → Shopify product/variant/metafield), and push logic. Once the type exists, each of the 8 Shopify stores becomes an instance with its own configuration.
- **Connect to Existing Shopify Exporter** (Ideas doc) — Minimal changes to work with existing exporter. Depends on how the current exporter works and whether it can be wrapped as an integration instance.
- **"View on Shopify Store" Buttons** (Ideas doc) — Deep links from PIM products to their Shopify counterparts. Requires the integration instance to track external IDs.
- **Third-Party SKU Management** (Ideas doc) — Feature for third parties to enable/remove SKUs from their Shopify stores.
- **Identical SKU Problem Detection** (Ideas doc) — Identify problematic SKU setups for "identical" SKUs going into Shopify.

**Future Connectors**

- **Amazon FBA Connector** — Integration type for pushing product data to Amazon FBA. Same framework pattern as Shopify: type definition, per-marketplace instances, field mapping, and run engine.
- **Additional Connectors** — The framework is designed so new connector types (distributors, retailers, internal systems) can be added without modifying the core. Each follows the same pattern: define the type, create instances, configure mappings, run pushes.

### Versioning & Snapshots

- **Product Version Snapshots** (§11.2) — Named snapshots, automatic snapshots on key events, diff/compare, rollback. Heavy feature. The audit history table (MVP) preserves all change data, so snapshots can be reconstructed or built on top later without losing anything.
- **Version Retention Policy** (§11.2) — Configurable pruning rules for old snapshots.

### Data Quality & AI

- **Data Quality Scoring** (§6) — Beyond completeness: text length, image resolution, spelling, attribute consistency. Additive layer on top of completeness.
- **AI Content Generation** (§18.1) — Description generation per channel/locale, translation assistance.
- **AI Image Intelligence** (§18.2) — Auto-tagging, alt text generation, category suggestions from images.
- **AI Classification & Suggestions** (§18.3) — Category suggestions, completeness acceleration via similar-product inference.
- **Anomaly Detection** (§18.4) — Flag outlier attribute values (prices, weights, etc.).
- **Duplicate Detection** (§18.4) — Surface potential duplicate products by attribute similarity.
- **Self-Serve Chat UI** (Ideas doc) — Natural language data querying. Cool but not MVP.

### Polish & Advanced Features

- **API Documentation — Auto-Generated** (§12.4) — GraphQL Playground, Swagger/OpenAPI. Helpful but can be generated after the APIs stabilize.
- **Relationship Attributes** (§9) — Attributes on relationships themselves (sort order, notes). The relationship table is MVP; enriching it with its own attributes is Phase 2.
- **Data Team Self-Service** (Ideas doc) — Enable data team to add attributes, locales, countries without developer involvement. Partially covered by MVP admin UI, but a polished self-service experience is Phase 2.

---

## Summary

| Category | MVP Items | Phase 2 Items |
|----------|-----------|---------------|
| Data Model | 16 | 3 |
| Auth & Permissions | 3 | 0 |
| API | 2 | 3 |
| Scheduling | 3 | 2 |
| Integration Framework | 4 | 0 |
| Import/Export | 1 (Akeneo migration) | 3 |
| Search | 1 | 1 |
| Completeness & Quality | 2 | 5 |
| UI | 10 (built alongside backend) | 4 (polish & advanced) |
| Integration Connectors | 0 | 7+ (Shopify, FBA, future) |
| Versioning | 1 (audit table) | 2 |
| AI | 0 | 7 |

The MVP is intentionally heavy on data model, schema decisions, infrastructure, and a working UI. The data model is the hardest thing to change later. The scheduling system and integration framework are in MVP because their table structures touch every entity and write path — retrofitting them means revisiting every API endpoint. The UI is how users validate that the data model and workflows are correct. Everything in Phase 2 is either additive (connectors that plug into the framework, new features on top of a solid foundation) or advanced UI polish that doesn't require schema migrations.
