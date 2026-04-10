# Task 0: Project Setup & Conventions

## Overview

We are building a **Product Information Management (PIM)** system — a centralized platform for managing product data, attributes, translations, and channel-specific content. This is an enterprise internal tool for Leatherman Tool Group. This document establishes the tech stack, project structure, and conventions that every subsequent task must follow.

The PIM will manage:
- 15 attribute types with per-locale and per-channel value scoping
- Attribute groups for organizational structure
- Families (product templates defining which attributes a product has)
- Categories organized in hierarchical trees
- Channels and locales for multi-market content scoping
- Role-based access control via Microsoft Entra ID

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Runtime | Node.js 20+ | Required |
| Language | TypeScript (strict mode) | Required |
| Backend | Express | API server, also serves the frontend in production |
| Frontend | React 18 + Vite | Fast dev server with HMR, runs as Express middleware in dev |
| Routing (client) | wouter | Lightweight client-side routing |
| Database | PostgreSQL 15+ (Neon-backed on Replit) | Needed for JSONB, full-text search, NULLS NOT DISTINCT |
| ORM | Drizzle ORM + node-postgres | SQL-native, type-safe DB access — better fit for the complex EAV queries, recursive CTEs, and bulk upserts this system requires |
| UI Framework | Tailwind CSS + shadcn/ui (40+ components) | Modern, accessible, flexible components |
| Auth | Passport.js + passport-azure-ad + express-session | Entra ID integration with server-side sessions stored in Postgres |
| Validation | Zod | Runtime schema validation, derive TS types from schemas |
| Data Tables | TanStack Table (via shadcn/ui DataTable) | Sorting, filtering, pagination for admin tables |
| State Management | TanStack Query | Server state management, caching, background refetch |

---

## Project Structure

```
pim/
├── client/                          # Frontend React app (Vite)
│   ├── index.html
│   ├── public/
│   └── src/
│       ├── App.tsx                  # Router setup (wouter)
│       ├── main.tsx                 # Entry point
│       ├── index.css                # Tailwind + theme variables
│       ├── components/
│       │   ├── ui/                  # shadcn/ui components (40+ pre-installed)
│       │   ├── layout/              # Shell, Sidebar, Header, NavMenu
│       │   ├── forms/               # Reusable form patterns
│       │   └── data-tables/         # Reusable data table patterns
│       ├── hooks/                   # use-toast, use-mobile, etc.
│       ├── lib/                     # queryClient, utils
│       └── pages/                   # Page components
├── server/                          # Express backend
│   ├── index.ts                     # Express app bootstrap
│   ├── routes.ts                    # Route registration
│   ├── storage.ts                   # DatabaseStorage class (Drizzle-backed)
│   ├── db.ts                        # Drizzle + pg Pool singleton
│   ├── lib/
│   │   └── api.ts                   # Response envelope helpers
│   ├── static.ts                    # Production static file serving
│   └── vite.ts                      # Dev middleware (Vite HMR proxy)
├── shared/                          # Shared between client & server
│   ├── schema.ts                    # Drizzle table definitions
│   └── validation.ts                # Zod validators
├── .env
├── .env.example
├── drizzle.config.ts
├── tailwind.config.ts
├── tsconfig.json                    # Strict mode enabled
├── vite.config.ts
└── package.json
```

**Architecture note:** In dev mode, Vite runs as middleware inside Express — single process, single port (5000). In production, Express serves the built static files. No separate frontend deployment needed.

---

## Database Conventions

### Table Naming
- **snake_case** for all table and column names
- **Plural nouns** for tables: `attributes`, `families`, `products`, `channels`
- **Junction tables** combine both entity names: `family_attributes`, `product_categories`
- Names must be self-explanatory — no abbreviations (`attribute_groups` not `attr_grps`)

### Primary Keys — Meaningful, Not Auto-Incrementing
Every primary entity uses a **meaningful string primary key** instead of auto-incrementing integers:

| Entity | PK Column | Format | Example Values |
|--------|-----------|--------|----------------|
| Attributes | `code` | camelCase | `productName`, `weight`, `longDescription` |
| Attribute Groups | `code` | camelCase | `marketing`, `technical`, `dimensions` |
| Families | `code` | camelCase | `knife`, `multitool`, `sheath` |
| Category Trees | `code` | camelCase | `masterCatalog`, `webNavigation` |
| Categories | `code` | camelCase | `handTools`, `accessories` |
| Channels | `code` | camelCase | `ecommerce`, `retail`, `print` |
| Locales | `code` | IETF format | `en_US`, `fr_FR`, `de_DE` |
| Roles | `code` | camelCase | `admin`, `editor`, `viewer` |

**Exceptions** where UUIDs or composite keys are appropriate:
- `attribute_values` — UUID primary key with a unique composite constraint
- `family_attributes` — composite PK of (family_code, attribute_code)

### Standard Columns on Every Table
```
created_at    TIMESTAMPTZ  NOT NULL  DEFAULT now()
updated_at    TIMESTAMPTZ  NOT NULL  DEFAULT now()
```

### Column Naming
- **snake_case** for all columns
- **Boolean columns**: prefix with `is_` (e.g., `is_localizable`, `is_required`, `is_channelizable`)
- **Foreign keys**: use the referenced table's PK column name with table context (e.g., `family_code`, `channel_code`, `parent_code`)
- **JSON columns**: use `_json` suffix only if ambiguous; otherwise name by content (e.g., `labels`, `validation_rules`, `settings`)

### Code Enforcement
All `code` primary keys must be validated as **camelCase** on write:
- Regex: `^[a-z][a-zA-Z0-9]*$`
- Must start with a lowercase letter
- No spaces, hyphens, or underscores
- Enforced at the API layer via Zod validation

Exception: `locale` codes use IETF format (`en_US`) and `product` identifiers allow uppercase and hyphens.

---

## API Conventions

### REST Endpoints
- **Base path**: `/api/v1/`
- **Resource naming**: plural, kebab-case in URLs (e.g., `/api/v1/attributes`, `/api/v1/attribute-groups`)
- **Standard HTTP methods**:
  - `GET /resources` — list with filtering and pagination
  - `GET /resources/:code` — read single
  - `POST /resources` — create
  - `PATCH /resources/:code` — partial update
  - `DELETE /resources/:code` — delete (soft or hard depending on entity)

### Response Envelope
Every API response uses a consistent envelope:

```typescript
// Success — single item
{
  "data": { ... },
}

// Success — list
{
  "data": [ ... ],
  "meta": {
    "total": 142,
    "limit": 25,
    "cursor": "abc123",
    "nextCursor": "def456" // null if no more pages
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Attribute code is required",
    "details": [
      { "field": "code", "message": "Required", "rule": "required" }
    ]
  }
}
```

### Pagination
- Cursor-based using `limit` and `cursor` query params
- Default limit: 25, max limit: 100
- Cursor is an opaque string (base64-encoded PK or offset)

### Filtering
- Query params for common filters: `?familyCode=knife&channel=ecommerce`
- Array filters with comma separation: `?status=active,draft`

---

## TypeScript Conventions

- **Strict mode** enabled in `tsconfig.json`
- **Zod schemas** are the single source of truth for validation — derive TypeScript types from them using `z.infer<typeof schema>`
- **No `any` types** — use `unknown` and narrow
- **Interfaces** for object shapes, **types** for unions and intersections
- **Enums** defined in Zod and as `pgEnum` in Drizzle schema, not TypeScript enums (use `as const` objects for application code)

Example pattern:
```typescript
import { z } from "zod";

export const attributeTypeEnum = z.enum([
  "text", "textarea", "boolean", "number", "measurement",
  "price", "date", "singleSelect", "multiSelect", "mediaFile",
  "assetSingle", "assetList", "entityReference", "productReference", "table"
]);

export const createAttributeSchema = z.object({
  code: z.string().regex(/^[a-z][a-zA-Z0-9]*$/, "Must be camelCase"),
  type: attributeTypeEnum,
  groupCode: z.string().optional(),
  labels: z.record(z.string()).optional(), // { "en_US": "Product Name" }
  isLocalizable: z.boolean().default(false),
  isChannelizable: z.boolean().default(false),
  isUnique: z.boolean().default(false),
  validationRules: z.object({ ... }).optional(),
});

export type CreateAttributeInput = z.infer<typeof createAttributeSchema>;
```

---

## Environment Variables

```env
# Database
DATABASE_URL=postgresql://user:pass@host:5432/pim

# Azure Entra ID / Auth
AZURE_AD_CLIENT_ID=
AZURE_AD_CLIENT_SECRET=
AZURE_AD_TENANT_ID=
SESSION_SECRET=  # generate with: openssl rand -base64 32
APP_URL=http://localhost:5000  # public URL, used for auth callback redirect

# App
NODE_ENV=development
PORT=5000
```

---

## Labels Pattern

Many entities (attributes, attribute groups, families, categories, channels) have human-readable labels that vary by locale. These are stored as a JSONB column:

```json
{
  "en_US": "Product Name",
  "fr_FR": "Nom du produit",
  "de_DE": "Produktname"
}
```

This avoids a separate labels table and keeps reads fast. The API accepts and returns this format directly.

