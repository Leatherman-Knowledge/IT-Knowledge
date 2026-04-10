# Phase 7 Review Feedback

**Reviewed**: 2026-02-20
**Spec Reference**: `07-UI-Shell-Foundation.md`

---

## Overall

Phase 7 is functionally comprehensive — all 10 admin screens, the application shell, authentication flow, shared components, and styling are implemented. The category tree view with full drag-and-drop exceeds spec expectations. Six items below need attention, roughly ordered by priority.

---

## Should Fix

### 1. Scoping change warning on attribute edit

The spec requires: *"Scoping — Localizable toggle, Channelizable toggle. Show warning if changing on edit and values exist."*

This isn't mentioned in the summary. Changing an attribute from localizable to non-localizable (or vice versa) after product values exist could cause data loss. When editing an existing attribute, toggling localizable or channelizable should show a warning explaining the impact. This doesn't need to block the save — just surface the warning so the user makes an informed decision.

### 2. Audit history missing "attribute" column

The spec lists the audit history table columns as: *"timestamp, user, action, resource type, resource ID, attribute."*

The implementation has Timestamp, User, Action, Resource Type, Resource ID — but no `attributeCode` column. This column matters because attribute-level changes (e.g., updating a specific attribute's value on a product) use `attributeCode` to identify which attribute changed. Without it in the table, users would have to expand every row to see this detail.

Please add `attributeCode` as a visible column (it can be blank for entity-level actions).

### 3. Variant definitions list view — show axis count and axes summary

The spec says: *"List view: Shows all variant definitions for the family — code, label, axis level count, summary of axes."*

The implementation only shows code and label. Please add:
- **Axis level count** (e.g., "2 levels")
- **Summary of axes** (e.g., "L1: color, L2: size") so users can see the structure at a glance without opening each definition

---

## Worth Discussing

### 4. Dialog-based forms vs route-based pages

This is the biggest structural deviation from the spec. The spec defines separate routes for create/edit (e.g., `/attributes/new`, `/attributes/:code`) and describes Pattern B as a full page with breadcrumbs back to the list. The implementation uses dialog-based forms within the list view instead — no entity-level routes exist.

For simple entities like currencies, locales, and roles, dialogs work well. But for complex screens like **Attributes** (7 form sections, 1,614 lines of code) and **Families** (attributes, requirements, variant definitions), a dialog feels cramped and you lose the ability to deep-link to a specific entity's edit page.

**Question**: Was the dialog approach an intentional choice for all screens, or could the complex screens (Attributes, Families) be converted to full-page edit views with their own routes? The simpler entities can stay as dialogs.

---

## Minor / Deferred

### 5. Login page loading state and retry

The spec requires a loading state while auth is in progress and an error state with a retry option. The summary mentions error handling via `?error=` query parameter but doesn't mention a loading spinner during the OAuth redirect or an explicit "Try Again" button on the error state. These are small UX improvements that could be added later.

### 6. Default post-login redirect

The spec says *"Successful auth redirects to the dashboard (default: Families page)."* The implementation redirects to a Dashboard/Home page with quick-link cards. This is a reasonable alternative — the Home page provides an overview rather than dropping users directly into Families. No change needed unless you want to match the spec exactly.
