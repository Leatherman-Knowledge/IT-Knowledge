Ω# Task 1: Microsoft Entra ID Auth + RBAC Schema

**Execution Plan Item:** #1
**Dependencies:** Project setup (Task 0)
**Depended on by:** Everything — all routes and API endpoints sit behind auth

---

## Objective

Get Microsoft Entra ID authentication working and create the RBAC (Role-Based Access Control) database tables. Every page and API endpoint must require authentication from day one. Permissions will be **loosely enforced** for now (just check "is authenticated") — strict RBAC enforcement comes later.

---

## 1. Entra ID App Registration

An Azure Entra ID (formerly Azure AD) app registration will be created for this PIM application. The app uses the **Authorization Code Flow with PKCE** for user authentication.

**What Replit needs from Leatherman (env vars):**
- `AZURE_AD_CLIENT_ID` — the Application (client) ID
- `AZURE_AD_CLIENT_SECRET` — a client secret
- `AZURE_AD_TENANT_ID` — Leatherman's Azure tenant ID

**Redirect URI to register:** `{APP_URL}/api/auth/callback`

---

## 2. Passport.js Configuration

Use Passport.js with the `passport-azure-ad` OIDCStrategy for Entra ID authentication. Sessions are stored server-side in PostgreSQL via `connect-pg-simple`.

### Session Setup (`server/lib/auth.ts`)

```typescript
import session from "express-session";
import connectPgSimple from "connect-pg-simple";
import passport from "passport";
import { OIDCStrategy } from "passport-azure-ad";

const PgSession = connectPgSimple(session);

// Session middleware — stores sessions in PostgreSQL
export const sessionMiddleware = session({
  store: new PgSession({
    pool: pool,             // reuse the existing pg Pool from server/db.ts
    tableName: "sessions",  // auto-created by connect-pg-simple
  }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === "production",
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
  },
});

// Passport OIDC strategy for Entra ID
passport.use(new OIDCStrategy({
    identityMetadata: `https://login.microsoftonline.com/${process.env.AZURE_AD_TENANT_ID}/v2.0/.well-known/openid-configuration`,
    clientID: process.env.AZURE_AD_CLIENT_ID!,
    clientSecret: process.env.AZURE_AD_CLIENT_SECRET!,
    responseType: "code",
    responseMode: "form_post",
    redirectUrl: `${process.env.APP_URL || "http://localhost:5000"}/api/auth/callback`,
    scope: ["openid", "profile", "email"],
    passReqToCallback: false,
  },
  async (profile, done) => {
    // profile.oid = Azure Object ID (stable unique identifier)
    // profile.displayName = full name
    // profile._json.email or profile._json.preferred_username = email
    // Look up or create the user in pim_users (see section 3 key behaviors)
    // Attach roles from user_roles
    return done(null, user);
  }
));

// Serialize: store user ID in session
passport.serializeUser((user, done) => {
  done(null, user.id);
});

// Deserialize: load full user with roles from DB on each request
passport.deserializeUser(async (id, done) => {
  const user = await getUserWithRoles(id);
  done(null, user);
});
```

### Key Behaviors
- On first login, auto-create a record in `pim_users` using the Entra ID profile data (OID, email, display name)
- On every request (via `deserializeUser`), the user's roles are loaded from `user_roles` and attached to `req.user`
- The `req.user` object should include: `id`, `email`, `displayName`, `roles[]`, `isActive`

---

## 3. Express Middleware — Protect All Routes

Wire up session + Passport middleware in `server/index.ts`:

```typescript
app.use(sessionMiddleware);
app.use(passport.initialize());
app.use(passport.session());
```

### Auth Routes (`server/routes.ts`)

```typescript
// Initiate Entra ID login
app.get("/api/auth/login", passport.authenticate("azuread-openidconnect", {
  prompt: "select_account",
}));

// Entra ID callback (handles the redirect after login)
app.post("/api/auth/callback",
  passport.authenticate("azuread-openidconnect", {
    failureRedirect: "/login?error=auth_failed",
  }),
  (req, res) => {
    res.redirect("/");
  }
);

// Logout
app.post("/api/auth/logout", (req, res) => {
  req.logout(() => {
    req.session.destroy(() => {
      res.redirect("/login");
    });
  });
});

// Session check (used by the frontend to get current user)
app.get("/api/auth/session", (req, res) => {
  if (req.isAuthenticated()) {
    res.json({ user: req.user });
  } else {
    res.json({ user: null });
  }
});
```

### API Route Protection

Create an Express middleware function that protects all `/api/v1/*` routes:

```typescript
export function requireAuth(req, res, next) {
  if (!req.isAuthenticated()) {
    return res.status(401).json({
      error: {
        code: "UNAUTHORIZED",
        message: "Authentication required",
      },
    });
  }
  next();
}

// Apply to all API routes
app.use("/api/v1", requireAuth);
```

For now, any authenticated user can access any endpoint (loose enforcement).

---

## 4. Database Schema

**Important:** Replace the existing starter `users` table with the `pim_users` table below. All auth-related tables should be added to `shared/schema.ts`.

### Table: `pim_users`

Tracks users who have logged in. Linked to Entra ID via the Azure Object ID.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `VARCHAR(255)` | PK | Azure Object ID (OID) from Entra — this is the meaningful, stable identifier |
| `email` | `VARCHAR(255)` | NOT NULL, UNIQUE | User's email from Entra claims |
| `display_name` | `VARCHAR(255)` | NOT NULL | Full name from Entra claims |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | Can be deactivated without deleting |
| `last_login_at` | `TIMESTAMPTZ` | NULL | Updated on each login |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

### Table: `roles`

Defines available roles in the system.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `code` | `VARCHAR(100)` | PK | camelCase role identifier |
| `label` | `VARCHAR(255)` | NOT NULL | Human-readable name |
| `description` | `TEXT` | NULL | What this role can do |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Seed data:**
| code | label |
|------|-------|
| `admin` | Administrator |
| `editor` | Editor |
| `viewer` | Viewer |

### Table: `permissions`

Defines granular permissions. A permission is a combination of resource type + action + optional scope.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | |
| `resource_type` | `VARCHAR(100)` | NOT NULL | e.g., `attribute`, `family`, `category`, `channel`, `locale` |
| `action` | `VARCHAR(50)` | NOT NULL | `create`, `read`, `update`, `delete` |
| `scope_type` | `VARCHAR(50)` | NULL | Scope dimension: `locale`, `channel`, `attributeGroup`, or NULL for unscoped |
| `scope_value` | `VARCHAR(255)` | NULL | Specific scope value (e.g., locale code `en_US`, channel code `ecommerce`, or `*` for all) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| UNIQUE | | `(resource_type, action, scope_type, scope_value)` | |

### Table: `role_permissions`

Maps roles to permissions.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `role_code` | `VARCHAR(100)` | FK → roles.code, ON DELETE CASCADE | |
| `permission_id` | `UUID` | FK → permissions.id, ON DELETE CASCADE | |
| PRIMARY KEY | | `(role_code, permission_id)` | |

### Table: `user_roles`

Maps users to roles.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `VARCHAR(255)` | FK → pim_users.id, ON DELETE CASCADE | |
| `role_code` | `VARCHAR(100)` | FK → roles.code, ON DELETE CASCADE | |
| `assigned_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `assigned_by` | `VARCHAR(255)` | FK → pim_users.id, NULL | Who assigned this role |
| PRIMARY KEY | | `(user_id, role_code)` | |

---

## 5. Auth Helper Functions

Create utility functions in `server/lib/auth.ts` that later tasks will use:

```typescript
// Get the current authenticated user with roles (from req.user, populated by Passport)
function getCurrentUser(req): PimUser & { roles: string[] }

// Check if user has a specific permission (loose for now — returns true for any authenticated user)
function hasPermission(userId: string, resourceType: string, action: string, scope?: { type: string, value: string }): Promise<boolean>

// Express middleware: require auth and attach user to request
function requireAuth(req, res, next): void
```

The `hasPermission` function should be **structured correctly now** (query the permission tables) but enforcement can be loose — it's acceptable to return `true` for any authenticated user. The important thing is that the function signature and calling pattern are established so enforcement can be tightened later without touching every endpoint.

---

## 6. API Endpoints

### `GET /api/v1/users/me`
Returns the current user's profile and roles.

### `GET /api/v1/roles`
List all roles.

### `GET /api/v1/roles/:code`
Get a single role with its permissions.

### `POST /api/v1/users/:id/roles`
Assign a role to a user. Body: `{ "roleCode": "editor" }`

### `DELETE /api/v1/users/:id/roles/:roleCode`
Remove a role from a user.

---

## 7. Login UI

### Login Page (`/login` — client-side route via wouter)
- Clean, branded page with a "Sign in with Microsoft" button
- The button links to `/api/auth/login` which triggers the Passport OIDC flow
- Redirects to Entra ID, then back to the dashboard on success
- Show error state if auth fails (check for `?error=` query param)

### Session Behavior
- Sessions are stored server-side in PostgreSQL via `connect-pg-simple`
- The frontend checks auth state by calling `GET /api/auth/session`
- If `user` is null, redirect to `/login`
- Logout calls `POST /api/auth/logout` and redirects to `/login`

### Frontend Auth Hook

Create a React hook for auth state that components can use:

```typescript
// client/src/hooks/use-auth.ts
function useAuth() {
  // Uses TanStack Query to fetch GET /api/auth/session
  // Returns { user, isLoading, isAuthenticated }
  // Redirects to /login if not authenticated (except on the login page itself)
}
```
