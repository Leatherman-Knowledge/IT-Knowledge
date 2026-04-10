# Phase 26: Import Monitoring & Verification

**Execution Plan Item:** #26  
**Replit Phase:** 26 (Week 6)  
**Dependencies:** Phase 24 (Integration Management UI), Phase 25.1–25.5 (Akeneo Import connector — all sub-phases complete)  
**Depended on by:** Phase 27 (GraphQL — requires real imported data), Phase 28 (Search — requires real imported data)

---

## Objective

Verify that the **end-to-end Akeneo import flow** works correctly using the integration framework built in Phases 21–25. Import monitoring is provided by the Phase 24 Integration Management UI — run history, per-record logs, status badges, and drill-down into failures are all already built. This phase is not a UI rebuild; it is a **verification and hardening pass** that:

1. Confirms the Phase 24 UI surfaces everything a user needs to monitor an Akeneo import.
2. Identifies and fixes any gaps in run log quality, data accuracy, or UI usability specific to inbound/import runs.
3. Verifies imported data is structurally correct in the PIM (families, categories, products, attribute values).
4. Ensures real data is available and queryable for the GraphQL and search work in Phase 27–28.

If the Phase 24 UI covers the Akeneo use case fully with no gaps, this phase is a verification pass only. If Akeneo-specific gaps exist (e.g., import-specific log context, summary statistics, or data quality issues), fix them here.

---

## 1. End-to-End Workflow Verification

Walk through the full user workflow using the Phase 24 UI and the Phase 25 connector. Verify each step works as expected:

**Setup:**
- [ ] Navigate to Integrations → Add instance → Select type "Akeneo Import". Form renders all connection config fields (base URL, client ID, client secret, username, password, scope filters).
- [ ] Enter valid Akeneo credentials and save. Instance appears in the instances list with direction "Inbound" and status "Active".
- [ ] Optionally configure field mappings on the instance if attribute codes differ between Akeneo and PIM.

**Run trigger:**
- [ ] From the instance detail or run history page, click "Run full import". Run appears in run history with status "Pending" or "Running".
- [ ] Run progresses to "Completed" (or "Failed" if credentials are wrong — verify error is shown clearly on the run record).

**Run history:**
- [ ] Completed run shows: started/finished timestamps, record counts (success / failure / skip), status badge. All columns render correctly for an inbound run.
- [ ] If some records failed or were skipped, counts are non-zero and the "View details" action is available.

**Per-record logs:**
- [ ] Clicking into a completed run shows the per-record log table. Columns: Record identifier, Status (badge), Message, Details (expandable). Pagination works for large imports.
- [ ] Filtering logs by status (success / failure / skip) works and narrows the list.
- [ ] Failed records show a meaningful message (e.g., "No PIM attribute found for Akeneo code: custom_data_field") that a user can act on.
- [ ] Skipped records show a clear reason (e.g., "Unsupported Akeneo attribute type: pim_reference_data_simpleselect").

---

## 2. Import-Specific Log Enhancements

Review the per-record logs from a real Akeneo import. If the following are missing or unclear, add them:

**Record identifier clarity:** The `record_identifier` field in `integration_run_logs` should make it obvious what record the log refers to. For Akeneo import:
- Attribute groups: use attribute group code (e.g., `marketing`).
- Attributes: use attribute code (e.g., `product_description`).
- Families: use family code (e.g., `tools`).
- Categories: use category code (e.g., `hand_tools`).
- Products / product models: use Akeneo product identifier / code (e.g., `LTG-001`).

If the current logs use numeric IDs or unclear identifiers, update the connector (Phase 25) to use the above format.

**Run summary enhancement (optional):** If the run detail view in Phase 24 does not already show a breakdown by entity type (e.g., "45 attributes imported, 12 families, 1,430 products"), consider adding a `details` JSON field on the `integration_runs` record itself to store a run-level summary. Populate this in the Akeneo connector at run completion:

```json
{
  "summary": {
    "attribute_groups": { "success": 8, "failure": 0, "skip": 0 },
    "attributes": { "success": 92, "failure": 3, "skip": 5 },
    "families": { "success": 14, "failure": 0, "skip": 0 },
    "categories": { "success": 156, "failure": 0, "skip": 0 },
    "products": { "success": 1430, "failure": 12, "skip": 4 }
  }
}
```

If `integration_runs` does not have a `details` JSONB column, add one in this phase. Surface the summary in the Phase 24 run detail view if time permits (otherwise log it and surface later).

---

## 3. Data Quality Verification

After a successful import, verify that the imported data is structurally correct in the PIM. Perform these checks via the existing PIM UI and REST API:

**Families:**
- [ ] All imported Akeneo families exist in PIM with the correct `code` and `label`.
- [ ] Each family has the correct attributes assigned.
- [ ] Families with Akeneo family variants have the corresponding `FamilyVariantDefinition` with correct axis attributes and levels.
- [ ] Families without variants have a zero-axis variant definition.

**Attributes:**
- [ ] All imported attributes exist in PIM with the correct code, type, and attribute group.
- [ ] `singleSelect` and `multiSelect` attributes have their option sets imported.
- [ ] Localizable and scopable flags are correctly set based on Akeneo `localizable` and `scopable` fields.

**Categories:**
- [ ] All imported categories exist in PIM with correct `code`, `label`, and parent/child relationships.
- [ ] Root categories appear at the top of the category tree; child categories appear nested under correct parents.

**Products:**
- [ ] Imported product models appear as parent products in PIM with the correct family, variant definition, and status.
- [ ] Leaf products appear as children under the correct parent, with the correct axis attribute values.
- [ ] Simple products (no Akeneo parent) appear as standalone top-model products.
- [ ] Attribute values on products match the source Akeneo data (spot-check 5–10 products: verify text values, select values, localized values, numeric values).
- [ ] Product category assignments are correct (products are assigned to the correct imported categories).
- [ ] `status` is `active` for enabled products and `draft` for disabled products.

**Wipe-and-replace check:**
- [ ] Trigger the full import a second time. Verify that the `pre-import-wipe` log entry appears as the first log row and shows a `success` status with deletion counts.
- [ ] After the second run completes, confirm the total product, family, attribute, and category counts match the first run — not doubled. The wipe cleared the previous data and the fresh import repopulated it cleanly.
- [ ] Spot-check that no stale or orphaned records remain (e.g., a product from the first run that was deleted in Akeneo between runs should not exist after the second run).

---

## 4. Gap Fixes

If verification reveals issues, fix them in this phase. Common expected gaps:

| Issue | Fix |
|-------|-----|
| Auth failure shows confusing error in run detail | Improve `error_message` text in the connector (Phase 25) to say `"Akeneo authentication failed: {HTTP status} — check credentials in instance config."` |
| Run detail page shows "0 records" even though logs exist | Verify `record_count_*` is being updated on the `integration_runs` row after batch log inserts. |
| Per-record logs missing for some entity types | Ensure every import step (attribute groups, attributes, families, categories, products) calls `appendIntegrationRunLog` for each record. |
| Duplicate products after second run | Verify upsert logic uses `identifier`/`code` correctly in all write paths. |
| Attribute values missing for some locales/channels | Check locale and channel resolution. Add skip log with reason for unmatched locale/channel rather than silently dropping values. |
| Category tree is flat (parent/child not set) | Verify topological sort in category tree rebuild (§8 of Phase 25). |
| Family variant definition not created | Verify the family variants fetch and mapping logic. |

Document any gaps found and the fixes applied as a brief note in the phase implementation summary.

---

## 5. Real Data Availability

Once import is verified:

- [ ] Confirm that at least one product with full attribute values, a category assignment, and a correct parent/child hierarchy is visible in the PIM Product list.
- [ ] Confirm that the PIM REST API can return the product with variants: `GET /api/v1/products?familyId={id}` returns results.
- [ ] Note the count of imported products, families, categories, and attributes in the implementation summary — this establishes the baseline dataset for Phase 27 (GraphQL) and Phase 28 (Search) testing.

---

## 6. UI Consistency Check (Inbound-Specific)

Phase 24 was built to handle both inbound and outbound. Verify inbound-specific UX is correct:

- [ ] "Run full import" is the label used (not "Run full sync" or "Run full push") for inbound instances. If the label is wrong, update Phase 24 UI: for `direction === 'inbound'`, show "Run full import"; for `direction === 'outbound'`, show "Run full sync" or "Run full push".
- [ ] "Push single product" button is hidden for inbound instances (Akeneo Import does not support `on_demand_single`). Verify `supports_single_record: false` hides the single-record trigger UI.
- [ ] The product filter section (Phase 24, §6) is hidden for inbound instances — product filters apply to outbound only.
- [ ] Field mappings section is visible and usable for inbound (source = Akeneo field, target = PIM attribute). Verify the labels make sense for inbound direction; update if they currently say "Source (PIM attribute)" when the PIM attribute is the target.

---

## 7. Exit Criteria

- [ ] A user can create an Akeneo Import instance, trigger a full import, and see completed run history and per-record logs entirely within the Phase 24 Integration Management UI.
- [ ] Run detail shows correct record counts (success / failure / skip) matching the per-record log entries.
- [ ] Per-record logs have meaningful record identifiers and human-readable messages for failures and skips.
- [ ] Imported data (families, attributes, categories, products with values) is verifiably correct in the PIM via UI and REST API spot checks.
- [ ] Wipe-and-replace works: second run shows a `pre-import-wipe` success log entry, and record counts after the second run match the first (no duplicates, no stale data).
- [ ] Inbound-specific UX is correct: "Run full import" label, no "Push single product" button, no product filter section, field mapping direction labels correct.
- [ ] Any identified gaps from §4 are fixed and documented in the implementation summary.
- [ ] Real data is available and confirmed queryable for Phase 27 (GraphQL) and Phase 28 (Search) testing.

---

## 8. Notes

- **No separate import dashboard.** Import monitoring is fully covered by Phase 24. This phase does not build a new screen — only fixes gaps in the existing one.
- **Media/file attributes.** Phase 25 skips importing binary file/image values. If no file values were expected in the spot-check, this is expected behavior. Document it in the summary.
- **Data volume.** If the Akeneo instance is large, the import may surface performance issues (slow run, timeout). Document if this occurs and note the record counts; optimization is a post-MVP concern.
- **Phase 27 dependency.** GraphQL work (Phase 27) depends on real product data with families, attribute values, and variants. This phase's verification pass is the gate: do not start Phase 27 until this exit criteria is confirmed.
