# Task 7: UI — Application Shell & Foundation Screens

**Execution Plan Item:** #7
**Dependencies:** All other Week 1-2 tasks (this is the UI for everything built in Tasks 1-6)

---

## Objective

Build the application shell (layout, navigation, auth flow) and admin screens for all the foundation entities created in Tasks 1-6. The UI should be clean, task-oriented, and follow progressive disclosure — making the common case obvious and complex options discoverable without clutter.

This task is the **frontend counterpart** to Tasks 1-6. REST API endpoints should be built alongside each entity's UI — this is the integration point where we verify the data model works end-to-end.

---

## 1. Application Shell

### Layout Structure

```
┌──────────────────────────────────────────────────┐
│  Header: Logo | App Name | User menu (dropdown)  │
├──────────┬───────────────────────────────────────┤
│          │                                       │
│ Sidebar  │  Main Content Area                    │
│ Nav      │                                       │
│          │  ┌─────────────────────────────────┐  │
│ Struct.  │  │  Page header + breadcrumbs       │  │
│  └ Fam.  │  │                                 │  │
│  └ Attr. │  │  Content (tables, forms, etc.)  │  │
│  └ Cat.  │  │                                 │  │
│ Settings │  └─────────────────────────────────┘  │
│  └ Chan. │                                       │
│  └ Loc.  │                                       │
│  └ Roles │                                       │
│ Activity │                                       │
│  └ Audit │                                       │
└──────────┴───────────────────────────────────────┘
```

### Sidebar Navigation

Keep the nav flat and organized into logical groups. Use collapsible sections:

**Structure** (data model configuration — expands by default)
- Families
- Attributes
- Attribute Groups
- Categories

**Settings** (system configuration)
- Channels
- Locales
- Currencies
- Roles & Permissions

**Activity**
- Audit History

### Header
- App logo / name on the left
- User avatar + name on the right with dropdown: Profile, Logout

### Responsive Behavior
- Sidebar collapses to icons on smaller screens
- The app is primarily for desktop use but should not break on tablets

---

## 2. Login Page

**Route:** `/login`

- Centered card layout with Leatherman branding
- "Sign in with Microsoft" button (triggers Entra ID OAuth flow)
- Loading state while auth is in progress
- Error state if auth fails with a retry option
- Successful auth redirects to the dashboard (default: Families page)

---

## 3. Foundation Admin Screens

Each admin screen follows a consistent two-pattern layout:

### Pattern A: List View
- **Page header** with title and "Create New" button
- **Search/filter bar** (if applicable)
- **Data table** using shadcn/ui DataTable (TanStack Table):
  - Sortable columns
  - Row click navigates to detail/edit view
  - Actions column with edit/delete buttons
  - Pagination at the bottom
- **Empty state** when no records exist (helpful message + create button)

### Pattern B: Create/Edit Form
- **Page header** with title and breadcrumb back to list
- **Form fields** using shadcn/ui Form (React Hook Form + Zod validation)
- **Save button** at the bottom (or sticky footer for long forms)
- **Cancel** returns to list without saving
- **Delete button** (on edit only) with confirmation dialog
- **Toast notifications** on success/error

---

## 4. Screen-by-Screen Specifications

### 4.1 Attributes List (`/attributes`)

**Table columns:**
| Column | Description |
|--------|-------------|
| Code | The camelCase attribute code |
| Label | Label in the user's locale (fall back to code) |
| Type | Attribute type with an icon or badge |
| Group | Attribute group label |
| Localizable | Boolean indicator |
| Channelizable | Boolean indicator |

**Filters:** Type dropdown, Group dropdown, Localizable/Channelizable toggles, text search on code/label.

### 4.2 Attribute Create/Edit (`/attributes/new`, `/attributes/:code`)

**Form sections:**
1. **General** — Code (readonly on edit), Type (readonly on edit, dropdown on create), Group (dropdown)
2. **Labels** — One text field per active locale. Show locale code as label.
3. **Scoping** — Localizable toggle, Channelizable toggle. Show warning if changing on edit and values exist.
4. **Properties** — Unique toggle, Read-only toggle
5. **Validation Rules** — Dynamic fields based on type:
   - Text/Textarea: regex, min/max length
   - Number: min/max value
   - Media: allowed extensions, max file size
   - Show only the rules relevant to the selected type
6. **Settings** — Type-specific settings (see Task 2, section 3)
7. **Options** (only for singleSelect/multiSelect) — Inline editable list:
   - Each option: code, labels per locale, sort order
   - Add option button
   - Drag to reorder
   - Delete option (with confirmation if values reference it)

### 4.3 Attribute Groups List (`/attribute-groups`)

Simple table: Code, Label, Attribute Count, Sort Order. Drag-to-reorder if feasible, otherwise sort order as editable field.

### 4.4 Attribute Group Create/Edit (`/attribute-groups/new`, `/attribute-groups/:code`)

Simple form: Code, Labels (per locale), Sort Order.

### 4.5 Families List (`/families`)

**Table columns:** Code, Label, Attribute Count, Label Attribute, Image Attribute.

### 4.6 Family Create/Edit (`/families/new`, `/families/:code`)

**Form sections:**
1. **General** — Code, Labels (per locale), Label Attribute (dropdown from family's attributes), Image Attribute (dropdown, filtered to mediaFile/assetSingle types)
2. **Attributes** — The main section. A table/list of attributes assigned to this family:
   - Columns: Attribute Code, Label, Type, Sort Order
   - "Add Attribute" button opens a picker/dialog showing all available attributes
   - Remove button per row
   - Drag-to-reorder for sort order
   - Note: Variant Level is **not** shown here — it is managed per variant definition (see 4.6.1)
3. **Requirements** — Tab per channel:
   - For each channel, show a checklist of the family's attributes
   - Each attribute has a "Required" toggle
   - For localizable attributes, show which locales are required (checkboxes per locale)
   - Non-localizable attributes just have a single required toggle per channel
4. **Variant Definitions** — See 4.6.1

### 4.6.1 Variant Definitions (within Family management)

Manage variant definitions for the family. Each definition is a named variant configuration specifying axes and attribute levels.

**List view:** Shows all variant definitions for the family — code, label, axis level count, summary of axes.

**Create:** Code (camelCase), Labels (per locale).

**Edit a definition:**
1. **Axes** — Ordered list of axis levels. Each level has one or more attribute selects, filtered to `singleSelect`/`boolean` types in the family. Multiple axes at a level produce cartesian-product branches (e.g., color × sheath at level 1).
2. **Attribute Levels** — For each non-axis attribute in the family, a dropdown showing which level it lives at (0 = common, 1 through axis count). Visual grouping by level for easy review.

**Delete:** With confirmation. (Future: blocked if products reference this definition.)

### 4.7 Categories (`/categories`)

**Unique UI: Tree View** instead of a flat table.

- Left panel: Tree navigation showing the hierarchy
  - Expandable/collapsible nodes
  - Drag-to-reorder siblings
  - Drag-to-move to new parent
  - Right-click context menu: Add child, Edit, Delete
- Right panel: Detail/edit form for the selected category
  - Code (readonly on edit), Labels (per locale), Sort Order
  - Breadcrumb showing ancestor path
  - Child count
- Top: Tree selector dropdown (switch between `masterCatalog`, `webNavigation`, etc.)
- "New Tree" and "New Category" buttons

### 4.8 Channels List & Edit (`/channels`)

**List table columns:** Code, Label, Locales (count or badges), Currencies, Category Tree.

**Edit form:**
1. Code, Labels
2. Locales — Multi-select picker showing available locales. Selected locales shown as badges/chips.
3. Currencies — Multi-select picker.
4. Category Tree — Dropdown of available trees.

### 4.9 Locales List & Edit (`/locales`)

**List table columns:** Code, Label, Active, Fallback Locale.

**Edit form:** Code (IETF format, e.g., `en_US`), Label, Active toggle, Fallback Locale (dropdown of other locales — validate no cycles).

### 4.10 Currencies List & Edit (`/currencies`)

Simple form: Code (ISO 4217), Label, Symbol, Active toggle.

### 4.11 Audit History Viewer (`/audit-history`)

- Filterable table: timestamp, user, action, resource type, resource ID, attribute
- Newest first
- Click to expand row and see old/new values

---

## 5. Shared Components to Build

These reusable components will be used across all screens:

| Component | Description | Used In |
|-----------|-------------|---------|
| `DataTable` | shadcn/ui DataTable with sorting, pagination, row actions | Every list view |
| `LocaleLabelsEditor` | Multi-locale label input (one field per active locale) | Attributes, Groups, Families, Categories, Channels |
| `CodeInput` | Text input with camelCase validation and "code already exists" check | All create forms |
| `ConfirmDialog` | "Are you sure?" dialog for deletes | All delete actions |
| `EmptyState` | Helpful empty state with create button | All list views |
| `PageHeader` | Title + breadcrumb + action buttons | Every page |
| `FilterBar` | Composable filter bar with dropdowns, toggles, search | List views |
| `TreeView` | Hierarchical tree component with expand/collapse, drag-drop | Categories |
| `MultiSelectPicker` | Select multiple items from a list with chips | Channels (locales, currencies) |

---

## 6. Design Principles

1. **Task-oriented, not data-oriented.** Screens should be organized around what the user wants to *do*, not just what data exists. The family edit page isn't just a CRUD form — it's where you "configure what a product looks like."
2. **Progressive disclosure.** Show the simple case first. Advanced settings (validation rules, type-specific settings) should be in collapsible sections or secondary tabs.
3. **Consistent patterns.** Every list view works the same way. Every create/edit form works the same way. Users learn the pattern once.
4. **Inline feedback.** Form validation happens on blur and on submit. Errors are shown next to the field, not just in a toast.
5. **Fast navigation.** Breadcrumbs on every page. Sidebar always visible. No more than 2 clicks to reach any entity.
6. **Labels fall back to codes.** If no label exists in the user's locale, display the camelCase code. This keeps the UI functional even before labels are entered.

---

## 7. Styling & Theming

- Use shadcn/ui defaults with Tailwind for customization
- Light mode by default. Dark mode support is nice-to-have but not required for MVP.
- Use a neutral color palette. Accent color for primary actions (Create, Save).
- Use color-coded badges for attribute types (text = blue, number = green, boolean = purple, etc.)
- Keep spacing generous — this is an admin tool, not a marketing site. Readability matters more than density.

