# Task 12: Product UI — List, Edit & Value Management

**Execution Plan Item:** #12
**Dependencies:** All Week 3 backend tasks (Tasks 8–11), UI Shell (Task 7). Post-implementation enhancements in **Phase 12.1.1** ([[12.1.1-Post-Implementation-Fixes-Value-Trace-UX]]) add URL scope, value trace, override reset, and data freshness behavior described below.
**Depended on by:** Everything product-facing in the UI going forward

---

## Objective

Build the full product user interface: the product list page, the product creation flow, and the product edit page with attribute value editing. This is the most complex and important UI in the PIM — it ties together families, variant definitions, attribute values with scoping, categories, completeness, and the variant hierarchy into a cohesive editing experience.

This task adds a **Products** section to the sidebar navigation (at the top, above "Structure") and builds out all the screens needed to create, browse, and edit products.

---

## 1. Sidebar Navigation Update

Add a new top-level section to the sidebar, above "Structure":

**Products** (expands by default)
- All Products
- *(future: saved views/filters will go here)*

**Structure** (existing, unchanged)
- Families
- Attributes
- Attribute Groups
- Categories

**Settings** (existing, unchanged)

**Activity** (existing, unchanged)

---

## 2. Product List Page (`/products`)

### Layout

```
┌──────────────────────────────────────────────────────────┐
│  Products                                    [+ Create]  │
├──────────────────────────────────────────────────────────┤
│  [Search: ________________]                              │
│                                                          │
│  Filters: [Family ▼] [Status ▼] [Completeness ▼]        │
│           [Category ▼] [Type ▼]                          │
│                                                          │
│  Showing 42 products                                     │
├──────────────────────────────────────────────────────────┤
│  ☐ │ Identifier    │ Label      │ Family │ Status │ ... │
│  ──┼───────────────┼────────────┼────────┼────────┼─────│
│  ☐ │ PRD-00003     │ Signal     │ knife  │ active │ 85% │
│  ☐ │ PRD-00001     │ Wingman    │ knife  │ draft  │ 57% │
│  ☐ │ PRD-00002     │ Charge TTi │ multi  │ active │100% │
│  ...                                                     │
├──────────────────────────────────────────────────────────┤
│  ← 1 2 3 ... →    25 per page                           │
└──────────────────────────────────────────────────────────┘
```

### Table Columns

| Column | Content | Notes |
|--------|---------|-------|
| Checkbox | Bulk selection | For bulk status changes |
| Identifier | Product identifier | Links to edit page |
| Label | The product's label attribute value (in the user's locale) | Falls back to identifier if no label value exists |
| Image | Thumbnail from the image attribute | Small square, placeholder if no image |
| Family | Family label | Badge style |
| Type | `simple`, `model`, or `variant` | Icon + label |
| Status | Draft / Active / Archived | Color-coded badge: grey/green/orange |
| Completeness | Percentage for the first channel | Circular progress indicator or colored bar |
| Updated | Last modified date | Relative time ("2 hours ago") |

### Filters

| Filter | Type | Options |
|--------|------|---------|
| Search | Text input | Searches identifier and label attribute value |
| Family | Dropdown (multi-select) | All families |
| Status | Dropdown (multi-select) | Draft, Active, Archived. Default: Draft + Active |
| Type | Dropdown (multi-select) | Simple, Model, Variant |
| Category | Tree picker (single select) | Opens a mini tree browser |
| Completeness | Range | Dropdown: "Complete (100%)", "Incomplete (<100%)", "Empty (0%)" |

### Behavior

- **Default view:** Show root-level products only (simple products and root models). Variants are accessible via the product model's edit page.
- **Row click:** Navigate to the product edit page.
- **Bulk actions toolbar** (appears when checkboxes are selected):
  - "Set Status" dropdown → applies to all selected products
  - "Delete" → with confirmation dialog
  - Show count of selected items
- **Empty state:** "No products yet. Create your first product to get started." with a Create button.
- **Pagination:** Cursor-based, 25 per page default.

---

## 3. Product Create Flow

### Entry Point

The "Create" button on the product list page opens a **multi-step creation dialog** (modal):

### Step 1: Choose Product Type

```
┌─────────────────────────────────────────┐
│  Create a Product                       │
├─────────────────────────────────────────┤
│                                         │
│  What type of product?                  │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │ ○  Simple Product               │    │
│  │    A standalone product with    │    │
│  │    no variants                  │    │
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │ ○  Product with Variants        │    │
│  │    A product model with variant │    │
│  │    children (sizes, colors...)  │    │
│  └─────────────────────────────────┘    │
│                                         │
│                          [Cancel] [Next]│
└─────────────────────────────────────────┘
```

### Step 2: Product Details

```
┌─────────────────────────────────────────┐
│  Create a Product                       │
├─────────────────────────────────────────┤
│                                         │
│  Family:     [Select family      ▼]     │
│                                         │
│  (if "Product with Variants" chosen):   │
│  Variant Definition: [Select...  ▼]     │
│  (dropdown filtered to definitions      │
│   in the selected family)               │
│                                         │
│  A unique product ID (e.g., PRD-00042)  │
│  will be assigned automatically.        │
│                                         │
│                        [Back] [Create]  │
└─────────────────────────────────────────┘
```

**Validation:**
- Family: required
- Variant Definition: required if product-with-variants type, shows available definitions for the selected family
- No identifier field — the system generates it automatically from a sequential number (e.g., `PRD-00042`)

**On Create:** Redirect to the product edit page. The auto-generated identifier is shown prominently.

### Creating Variants (Children)

Variant creation happens from **within the product edit page** (section 5), not from the product list. When editing a product model, there's an "Add Variant" button that opens a dialog:

```
┌─────────────────────────────────────────┐
│  Add Variant to PRD-00003               │
├─────────────────────────────────────────┤
│                                         │
│  Axis Values:                           │
│    Handle Color: [Select option  ▼]     │
│                                         │
│  (Pre-fills available options that      │
│   haven't been used yet)                │
│                                         │
│  ID will be generated automatically:    │
│  PRD-00003-{selected option code}       │
│                                         │
│                        [Cancel] [Add]   │
└─────────────────────────────────────────┘
```

For 2-axis definitions, adding a sub-model (level 1) shows level-1 axes. Adding a leaf variant under a sub-model shows level-2 axes.

---

## 4. Product Edit Page — Layout

### Route: `/products/:identifier`

The product edit page is the primary workspace. Its layout adapts based on the product type.

**URL scope (Phase 12.1.1):** Channel and locale can be persisted in the URL for deep linking and scope preservation: `/products/:identifier?channel=ecommerce&locale=en-DE`. When present, the picker initializes from these params; changing picker updates the URL via `history.replaceState`. All in-page links to other products (breadcrumbs, variant rows, categories root link, completeness breakdown) include the current `?channel=&locale=` so scope is preserved when navigating.

### Layout for Simple Products

```
┌──────────────────────────────────────────────────────────────┐
│  ← Products / PRD-00001                                      │
│  Wingman Multi-Tool                          [Draft ▼] [Save]│
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌──────────────────────────────────────┐   │
│  │ Channel/     │  │                                      │   │
│  │ Locale       │  │  Attribute Values                    │   │
│  │ Picker       │  │  (grouped by attribute group)        │   │
│  │              │  │                                      │   │
│  │ Channel:     │  │  ┌── Marketing ──────────────────┐   │   │
│  │ [ecommerce▼] │  │  │ Product Name: [Wingman____]   │   │   │
│  │              │  │  │ Description:  [A versatile..]  │   │   │
│  │ Locale:      │  │  └──────────────────────────────┘   │   │
│  │ [en_US   ▼]  │  │                                      │   │
│  │              │  │  ┌── Technical ──────────────────┐   │   │
│  │ Completeness │  │  │ Weight:       [198] [g ▼]     │   │   │
│  │ ████████ 85% │  │  │ Blade Steel:  [420HC______]   │   │   │
│  │              │  │  └──────────────────────────────┘   │   │
│  │ Missing:     │  │                                      │   │
│  │ • longDesc   │  │  ┌── Categories ────────────────┐   │   │
│  │              │  │  │ • Hand Tools > Knives         │   │   │
│  │              │  │  │ • EDC                [+ Add]  │   │   │
│  │              │  │  └──────────────────────────────┘   │   │
│  └─────────────┘  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Layout for Product Models (with Variants)

```
┌──────────────────────────────────────────────────────────────┐
│  ← Products / PRD-00003                                      │
│  Signal Multi-Tool                           [Active▼] [Save]│
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  [Attributes] [Variants] [Categories]    ← Tab navigation   │
│                                                              │
│  ┌─────────────┐  ┌──────────────────────────────────────┐   │
│  │ Context      │  │  (Active tab content)                │   │
│  │ Panel        │  │                                      │   │
│  │              │  │                                      │   │
│  │ Channel:     │  │                                      │   │
│  │ [ecommerce▼] │  │                                      │   │
│  │              │  │                                      │   │
│  │ Locale:      │  │                                      │   │
│  │ [en_US   ▼]  │  │                                      │   │
│  │              │  │                                      │   │
│  │ Completeness │  │                                      │   │
│  │ ████████ 85% │  │                                      │   │
│  └─────────────┘  └──────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Product Edit Page — Tabs

### Tab 1: Attributes (Value Editing)

This is the primary tab. It shows all attribute values for the current product, organized by attribute group.

**For a root product model:** Only shows level-0 attributes (the ones owned by the root). Level-1 and level-2 attributes are edited on the respective children.

**For a sub-model or variant:** Shows only the attributes owned by this product's level. Inherited attributes from ancestors are shown in a read-only "Inherited Values" section above the editable section.

#### Channel & Locale Picker (Context Panel)

The left sidebar context panel has two dropdowns:
- **Channel:** Lists channels the product belongs to (derived from category assignments). If the product isn't in any channel's category tree, show a message: "Assign this product to a category to see channel-specific values."
- **Locale:** Lists locales available in the selected channel.

Changing the channel or locale reloads the attribute values shown in the main area (and updates the URL query params). This is how users switch between editing the English vs. French product name, or the e-commerce vs. retail description. Product-specific queries use `staleTime: 0` so scope changes always fetch fresh data; the edited-values state is synced from the server only when values for the current scope have loaded (`productValues.length > 0`) to avoid showing the wrong locale's data during the transition.

**For global (non-scoped) attributes:** Always shown regardless of channel/locale selection. Displayed in a "General" section at the top.

**For localizable-only attributes:** Shown when a locale is selected. The channel doesn't affect them.

**For channelizable-only attributes:** Shown when a channel is selected. The locale doesn't affect them.

**For localizable + channelizable attributes:** Shown when both channel and locale are selected.

#### Attribute Group Sections

Attributes are grouped into collapsible sections by attribute group (e.g., "Marketing", "Technical", "Dimensions"). Groups are ordered by their `sort_order`. Attributes within each group are ordered by their `sort_order` in the family.

Ungrouped attributes (those without an attribute group) appear in an "Other" section at the bottom.

#### Attribute Value Input Components

Each attribute type has a dedicated input component:

| Attribute Type | Input Component |
|---------------|----------------|
| `text` | Single-line text input. Shows character count if `maxLength` is set. |
| `textarea` | Multi-line textarea. Rich text editor if `settings.richText` is true. Shows character count. |
| `boolean` | Toggle switch (on/off). |
| `number` | Number input with step based on `settings.decimalPlaces`. Shows min/max if set. |
| `measurement` | Number input + unit dropdown. Unit options from `settings.allowedUnits`. |
| `price` | One row per currency (from `settings.currencies`). Each row: currency label + amount input. |
| `date` | Date picker. Include time picker if `settings.includeTime` is true. |
| `singleSelect` | Dropdown populated from attribute options. Show option labels in current locale. |
| `multiSelect` | Multi-select dropdown or checkbox list. Options from attribute options. |
| `mediaFile` | File upload zone with preview. Shows current file if exists. Accept filter from `settings.allowedExtensions`. |
| `assetSingle` | Text input showing the asset code. (Full asset picker comes later with the Asset Manager.) |
| `assetList` | Tag-style input for multiple asset codes. (Full picker later.) |
| `entityReference` | Text input showing the entity code. (Full picker later.) |
| `productReference` | Text input showing the product identifier. (Full picker later.) |
| `table` | Inline spreadsheet-style grid based on `settings.columns`. Add row / remove row buttons. |

#### Inherited Values Display

For products that are part of a variant hierarchy (sub-models and variants), show inherited values in a distinct read-only style:

- Group inherited values in an **"Inherited from [PARENT]"** section at the top, visually distinct (e.g., slightly greyed out background, lock icon)
- Each inherited value shows which ancestor it comes from. **Phase 12.1.1:** Also show **locale fallback** source when the value came from a different locale in the fallback chain (e.g. "From locale en"); treat both hierarchy inheritance and locale fallback as "inherited" for styling and tooltips.
- **Rich per-type rendering:** The `InheritedValueDisplay` component renders human-readable output based on attribute type rather than raw stored values:
  - **singleSelect / multiSelect:** Option label (e.g. "Red", "Large") instead of the raw option ID.
  - **entityReference / entityList:** Entity label and ID (e.g. "Stainless Steel (#5)") fetched from the API via a dedicated `EntityValueLabel` sub-component.
  - **assetSingle / assetList:** Actual asset thumbnail image instead of the code/ID.
  - **boolean:** "Yes" / "No" instead of `true` / `false`, with robust string/number coercion.
  - **measurement:** Formatted value with unit (e.g. "150 mm").
  - **price:** Currency badges with amounts (e.g. "USD: 29.99").
  - **date:** Localized date format instead of raw ISO string, with invalid-date guard.
  - **All other types:** Plain text display fallback.
- Inherited values are **not editable** here — clicking on one shows a tooltip: "This value is set on [PRD-00003]. Click to edit it there." with a link to the parent's edit page. **Phase 12.1.1:** A **"Trace"** control (or click on the source badge) opens the **Value Resolution Modal** to show the full resolution path (locale chain × hierarchy) and which step provided the winning value; ancestor links in the modal are clickable.
- **Override:** Where the backend allows override at this level, the user can edit to set a value on the current product. **Phase 12.1.1:** When a value is overridden, a **Reset** button (e.g. RotateCcw icon) is shown; clicking it calls `DELETE /api/v1/products/:identifier/values/:attributeCode` (with channel/locale scope) to remove the override and revert to the inherited value.
- **Axis attributes** are excluded from the editable attribute groups and shown only as read-only badges in the product header; they are non-editable through the values form. Axis values also use the rich per-type rendering (e.g. select axis shows option label, not ID).
- **Override stats (Phase 12.1.1):** On model/sub-model products, show a badge (e.g. GitBranch) next to attribute labels when descendants have overridden that attribute (e.g. "3 overrides"), using `GET /api/v1/products/:identifier/override-stats`.

#### Save Behavior

- **Auto-save is NOT used.** Attribute values are saved explicitly via the "Save" button.
- On save, send all changed values to `PUT /api/v1/products/:identifier/values`.
- Only send values that have actually changed (diff against the loaded state).
- Show a saving indicator (spinner on the Save button).
- On success, show a toast: "Product saved."
- On validation error, scroll to the first invalid field and show inline error messages.
- Track dirty state: if the user navigates away with unsaved changes, show a "You have unsaved changes" confirmation dialog.

#### Completeness Panel

The context panel shows completeness for the currently selected channel+locale:
- Circular progress indicator with percentage
- Count: "6 of 7 required attributes filled"
- List of missing required attributes, each clickable to scroll to that field in the form
- Completeness updates in real-time as values are entered (recalculated client-side based on which required fields have values)

### Tab 2: Variants (only for product models)

This tab shows the variant hierarchy and allows managing variants.

#### For a Root Model with 1 Axis Level

```
┌──────────────────────────────────────────────────────────┐
│  Variants                                   [+ Add Variant]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Variant Definition: Color then Blade                    │
│  Axis: Handle Color                                      │
│                                                          │
│  ┌──────────────────────────────────────────────────────┐│
│  │ Identifier       │ Handle Color │ Status │ Complete  ││
│  ├──────────────────┼──────────────┼────────┼───────────┤│
│  │ PRD-00002-black   │ Black        │ Active │ 100%      ││
│  │ PRD-00002-silver  │ Silver       │ Active │ 100%      ││
│  │ PRD-00002-red     │ Red          │ Draft  │ 72%       ││
│  └──────────────────────────────────────────────────────┘│
│                                                          │
│  Click a variant to edit its attribute values.           │
└──────────────────────────────────────────────────────────┘
```

#### For a Root Model with 2 Axis Levels

Show a tree/accordion structure:

```
┌──────────────────────────────────────────────────────────┐
│  Variants                              [+ Add Sub-Model]  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Variant Definition: Color then Blade                    │
│  Level 1 Axis: Handle Color                              │
│  Level 2 Axis: Blade Type                                │
│                                                          │
│  ▼ PRD-00003-red (Handle Color: Red)          Draft   72%   │
│    ├── PRD-00003-red-plain (Blade: Plain)     Active  100%  │
│    └── PRD-00003-red-serrated (Blade: Serr.)  Draft   85%  │
│                                         [+ Add Variant]     │
│                                                             │
│  ▼ PRD-00003-blue (Handle Color: Blue)        Draft   57%   │
│    └── PRD-00003-blue-plain (Blade: Plain)    Draft   57%  │
│                                      [+ Add Variant]     │
│                                                          │
│  Click any item to edit its attribute values.            │
└──────────────────────────────────────────────────────────┘
```

**Behaviors:**
- Clicking a variant/sub-model navigates to its edit page (`/products/PRD-00003-red-plain`) **with the current channel/locale scope preserved** in the URL (e.g. `?channel=ecommerce&locale=en-DE`) so the user does not lose their scope selection. **Phase 12.1.1:** All variant and breadcrumb links receive the current `scopeQuery` and append it to the target path.
- "Add Sub-Model" at the top creates a level-1 child under the root
- "Add Variant" within a sub-model section creates a level-2 child under that sub-model
- Each row shows: identifier, axis value(s), status badge, completeness percentage
- Delete button on each row (with confirmation: "This will also delete all attribute values for this variant.")

### Tab 3: Categories

Shows and manages category assignments for this product (root-level only).

```
┌──────────────────────────────────────────────────────────┐
│  Categories                                    [+ Assign] │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Master Catalog                                          │
│    └ Hand Tools > Knives > Folding Knives       [✕]      │
│                                                          │
│  Web Navigation                                          │
│    └ Lifestyle > Everyday Carry                 [✕]      │
│    └ New Arrivals                               [✕]      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**"Assign" button** opens a category tree browser dialog:
- Shows all category trees with their hierarchies
- Checkboxes on each category node
- Already-assigned categories are pre-checked
- Search bar to find categories by label/code
- Save applies the changes

**For variants/sub-models:** Show the categories inherited from the root in a read-only display with a note: "Categories are managed on the root product model [PRD-00003]." with a link. **Phase 12.1.1:** The root product link includes the current scope query so channel/locale is preserved. **Category assignment is optional:** products with no categories show an empty categories tab without error.

---

## 6. Breadcrumb Navigation

Products within a variant hierarchy should show a breadcrumb that reflects the hierarchy:

- **Simple product:** `Products / PRD-00001`
- **Root model:** `Products / PRD-00003`
- **Sub-model:** `Products / PRD-00003 / PRD-00003-red`
- **Leaf variant:** `Products / PRD-00003 / PRD-00003-red / PRD-00003-red-plain`

Each breadcrumb segment is clickable and navigates to that product's edit page. This makes it easy to navigate up and down the hierarchy. **Phase 12.1.1:** Breadcrumb links include the current channel/locale query params so scope is preserved when navigating.

---

## 7. Shared Components to Build

| Component | Description | Used In |
|-----------|-------------|---------|
| `ProductCreateDialog` | Multi-step modal for creating new products | Product list page |
| `VariantCreateDialog` | Modal for adding a variant/sub-model to a product model | Product edit page (Variants tab) |
| `ChannelLocalePicker` | Combined channel + locale selector with scoping awareness | Product edit page sidebar |
| `CompletenessIndicator` | Circular/bar progress indicator with percentage | Product list, product edit sidebar |
| `AttributeValueInput` | Polymorphic input component that renders the correct input based on attribute type | Product edit (Attributes tab) |
| `MeasurementInput` | Number + unit dropdown combo | For measurement attributes |
| `PriceInput` | Multi-currency price editor (rows per currency) | For price attributes |
| `MediaUploadInput` | File upload with preview, drag-and-drop | For mediaFile attributes |
| `OptionSelect` | Dropdown populated from attribute options with locale-aware labels | For singleSelect/multiSelect |
| `TableAttributeEditor` | Inline grid editor for table-type attributes | For table attributes |
| `InheritedValueDisplay` | Read-only display for inherited attribute values with rich per-type rendering (select→label, entity→name, asset→thumbnail, boolean→Yes/No, measurement→value+unit, price→currency badges, date→localized), source annotation (including locale fallback), and optional Reset / Trace | Product edit for variants |
| `ValueResolutionModal` | Modal showing the full value resolution path (locale chain × hierarchy) with winner highlight and clickable ancestor links | Product edit — triggered from inherited/source badges |
| `CategoryAssignmentDialog` | Tree browser with checkboxes for assigning categories | Product edit (Categories tab) |
| `VariantTreeView` | Expandable tree showing the variant hierarchy with inline summaries; links include scope query | Product edit (Variants tab) |

---

## 8. Design Specifications

### Status Badge Colors
| Status | Background | Text |
|--------|-----------|------|
| Draft | Grey (neutral) | Dark grey |
| Active | Green (success) | Dark green |
| Archived | Amber/Orange (warning) | Dark amber |

### Completeness Indicator Colors
| Range | Color |
|-------|-------|
| 100% | Green |
| 50–99% | Amber/Yellow |
| 1–49% | Orange |
| 0% | Red/Grey |

### Inherited Value Styling
- Background: slightly tinted (e.g., very light blue or grey)
- Left border: accent color indicating "inherited"
- Lock icon next to the field label
- Tooltip on hover: "Inherited from [PARENT-IDENTIFIER]"

### Required Field Indicators
- Required attributes show a red asterisk (*) next to the label
- Missing required values show a subtle red left border on the input
- The completeness panel highlights missing required fields

---

## 9. Data Loading Strategy

### Product Edit Page — Initial Load

When the product edit page loads, read channel/locale from URL query params if present (`?channel=&locale=`), then fetch in parallel:
1. `GET /api/v1/products/:identifier` — product details, ancestors, children
2. `GET /api/v1/products/:identifier/values?includeInherited=true&channelCode=…&localeCode=…` — attribute values for the selected scope
3. `GET /api/v1/products/:identifier/completeness` — completeness across all channels
4. `GET /api/v1/products/:identifier/categories` — category assignments
5. `GET /api/v1/products/:identifier/override-stats` — (Phase 12.1.1) descendant override counts per attribute (for model/sub-model)
6. `GET /api/v1/families/:familyCode` — family details with attributes and requirements
7. `GET /api/v1/locales?isActive=true` — active locales (for the locale picker)
8. `GET /api/v1/channels` — channels (for the channel picker)

Use TanStack Query for all fetches. The global TanStack Query default `staleTime` is `0`, so all queries refetch fresh data on every page navigation by default. Individual queries may set explicit overrides where appropriate (e.g. `staleTime: 60000` for inline asset thumbnails in pickers). ~~Previously, global reference data (families, locales, channels, attributes) used `staleTime: Infinity`, but this caused stale data to persist after edits on other pages; the global default was changed to `0` to fix this.~~

### Channel/Locale Switch and Edited-Values Sync

When the user switches channel or locale, invalidate/refetch the values query so the UI gets fresh data for the new scope. **Phase 12.1.1:** Sync the server response into the local `editedValues` map only when `productValues.length > 0` and when the current `valuesQueryString` (channel+locale) has not already been synced for this identifier. Use a ref (`lastSyncedScopeRef`) to track the last synced scope and avoid a race where the effect runs during loading (empty values) and marks the new scope as synced before data arrives, which would cause the wrong locale's data to display. When the product `identifier` changes, reset this ref so the new product gets a fresh sync.

### Save

On save, compute a diff of changed values and send only the changes to `PUT /api/v1/products/:identifier/values`. On success, invalidate the TanStack Query cache for product values and completeness to trigger a background refetch.

---

## 10. Error Handling

| Scenario | Behavior |
|----------|----------|
| Product not found | Redirect to `/products` with a toast: "Product not found." |
| Save validation error | Scroll to first error, show inline error messages on each invalid field. Toast: "Some values could not be saved. Check the highlighted fields." |
| Level mismatch on save | Should not happen if the UI is correct, but if the API returns a level error, show a toast with the API error message. |
| Uniqueness conflict | Show inline error on the specific field: "This value is already used by product [OTHER-ID]." |
| Network error | Toast: "Could not save. Check your connection and try again." Keep the form in its current state. |
| Concurrent edit conflict | If the API returns a version conflict (if implemented), show: "This product was modified by another user. Reload to see the latest version." |
