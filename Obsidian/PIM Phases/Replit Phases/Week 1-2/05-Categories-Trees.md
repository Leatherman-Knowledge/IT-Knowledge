# Task 5: Categories & Trees

**Execution Plan Item:** #5
**Dependencies:** Project setup (Task 0), auth (Task 1)
**Depended on by:** Channels (Task 3 — channels reference a category tree)

---

## Objective

Build the hierarchical category system. Categories are organized into **trees** (e.g., "Master Catalog," "Web Navigation"). Each tree has a single root, and categories form a parent-child hierarchy within each tree. Products can belong to multiple categories across multiple trees.

This task is relatively independent from attributes and families, so it can be built in parallel.

---

## 1. Database Schema

### Table: `category_trees`

A tree is a named root for a hierarchy of categories.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(100)` | PK | camelCase identifier |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Master Catalog" }` |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | Inactive trees are hidden from UI |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `categories`

Each category belongs to exactly one tree and has at most one parent (except root categories which have no parent).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(255)` | PK | camelCase identifier (globally unique across all trees) |
| `tree_code` | `VARCHAR(100)` | NOT NULL, FK → category_trees.code | Which tree this category belongs to |
| `parent_code` | `VARCHAR(255)` | FK → categories.code, NULL | Parent category. NULL = root of its tree |
| `labels` | `JSONB` | NOT NULL, DEFAULT '{}' | `{ "en_US": "Hand Tools" }` |
| `sort_order` | `INTEGER` | NOT NULL, DEFAULT 0 | Display order among siblings |
| `depth` | `INTEGER` | NOT NULL, DEFAULT 0 | Computed: 0 for roots, 1 for first-level children, etc. |
| `path` | `TEXT` | NOT NULL, DEFAULT '' | Materialized path: `handTools.knives.foldingKnives` |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
```sql
CREATE INDEX idx_categories_tree ON categories(tree_code);
CREATE INDEX idx_categories_parent ON categories(parent_code);
CREATE INDEX idx_categories_path ON categories(path);
CREATE INDEX idx_categories_tree_sort ON categories(tree_code, parent_code, sort_order);
```

### Tree Storage Strategy: Adjacency List + Materialized Path

We use a **hybrid approach**:
- **Adjacency list** (`parent_code` column) — simple, easy to maintain, supports moves and deletes
- **Materialized path** (`path` column) — enables fast "find all descendants" queries without recursive CTEs

The `path` column stores the full ancestor chain as dot-separated codes: `handTools.knives.foldingKnives`. To find all descendants of `handTools`, query: `WHERE path LIKE 'handTools.%'`.

The `depth` and `path` columns are **computed on write** — whenever a category is created or moved, recalculate its depth and path (and all descendants' paths if it was moved).

---

## 2. API Endpoints

### Category Trees

**`GET /api/v1/category-trees`** — List all trees
**`GET /api/v1/category-trees/:code`** — Get a tree with its root categories
**`POST /api/v1/category-trees`** — Create a tree
```json
{
  "code": "masterCatalog",
  "labels": { "en_US": "Master Catalog" }
}
```
**`PATCH /api/v1/category-trees/:code`** — Update tree metadata
**`DELETE /api/v1/category-trees/:code`** — Delete. Fails if any channels reference this tree or if it contains categories.

### Categories

**`GET /api/v1/categories`** — List categories. Supports filters:
  - `?treeCode=masterCatalog` — filter by tree (commonly used)
  - `?parentCode=handTools` — get direct children of a category
  - `?root=true` — get only root categories
  - `?includeDescendants=true` — include all descendants in response (flat list with depth/path)

**`GET /api/v1/categories/:code`** — Get a single category with:
  - Its ancestors (breadcrumb path)
  - Its direct children

**`POST /api/v1/categories`** — Create a category
```json
{
  "code": "foldingKnives",
  "treeCode": "masterCatalog",
  "parentCode": "knives",
  "labels": { "en_US": "Folding Knives" },
  "sortOrder": 1
}
```

**`PATCH /api/v1/categories/:code`** — Update labels, sortOrder. To **move** a category to a different parent:
```json
{
  "parentCode": "newParent"
}
```
Moving a category must recalculate `depth` and `path` for the category and all its descendants.

**`DELETE /api/v1/categories/:code`** — Delete a category.
  - If the category has children: fail unless `?cascade=true` (deletes all descendants)

### Full Tree Endpoint

**`GET /api/v1/category-trees/:code/tree`** — Returns the complete tree as a nested JSON structure:
```json
{
  "data": {
    "code": "masterCatalog",
    "labels": { "en_US": "Master Catalog" },
    "children": [
      {
        "code": "handTools",
        "labels": { "en_US": "Hand Tools" },
        "children": [
          {
            "code": "knives",
            "labels": { "en_US": "Knives" },
            "children": [
              { "code": "foldingKnives", "labels": { "en_US": "Folding Knives" }, "children": [] }
            ]
          },
          { "code": "multitools", "labels": { "en_US": "Multi-Tools" }, "children": [] }
        ]
      }
    ]
  }
}
```

This nested response is built from the flat `categories` table. Use the `path` column for efficient tree construction: query all categories for a tree, sort by path, and build the nested structure in application code.

---

## 3. Business Rules

1. **Category codes are camelCase and globally unique** (across all trees). This simplifies references — you can identify a category by code alone without needing the tree code.
2. **A category belongs to exactly one tree.** Moving between trees is not supported (delete and recreate).
3. **A category can have at most one parent.** No multi-parent hierarchies.
4. **Root categories have `parent_code = NULL` and `depth = 0`.**
5. **The `path` column** is automatically computed. For root categories: `code`. For children: `parentPath.code`. Must be recalculated on move.
6. **No circular references.** A category cannot be its own ancestor. Validate on move.
7. **Sort order** determines display order among siblings. Default 0, secondary sort by code.
8. **Deleting a category** that has children should fail by default. Require explicit `?cascade=true` to delete a subtree.

---

## 4. Path Recalculation on Move

When a category is moved to a new parent, you need to update its `path`, `depth`, and the `path` and `depth` of all descendants:

```sql
-- Example: moving "knives" from parent "handTools" to parent "bladeTools"
-- Old path: "handTools.knives"
-- New path: "bladeTools.knives"
-- All descendants need path prefix updated:
UPDATE categories
SET
  path = REPLACE(path, 'handTools.knives', 'bladeTools.knives'),
  depth = depth + (new_depth - old_depth)
WHERE path LIKE 'handTools.knives.%' OR code = 'knives';
```

This is more complex in practice (need to handle depth changes correctly). Implement as a transaction that:
1. Updates the moved category's parent, path, and depth
2. Updates all descendants' paths and depths
3. Validates no circular reference was created

