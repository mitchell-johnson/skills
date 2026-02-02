# Gotchas

Critical pitfalls when building with this stack. Read this before deploying.

## 1. Better Auth D1 camelCase Columns

**Problem:** Better Auth expects camelCase column names (`createdAt`, `userId`, `emailVerified`) but SQL convention is snake_case. If you create tables with snake_case columns, Better Auth will silently fail to read/write session and user data.

**Solution:** Create Better Auth tables with camelCase columns from the start. Quote them with double quotes in SQL since they contain uppercase letters:

```sql
CREATE TABLE IF NOT EXISTS "user" (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    "emailVerified" INTEGER NOT NULL DEFAULT 0,
    image TEXT,
    "createdAt" TEXT NOT NULL DEFAULT (datetime('now')),
    "updatedAt" TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS "session" (
    id TEXT PRIMARY KEY,
    "expiresAt" TEXT NOT NULL,
    token TEXT NOT NULL UNIQUE,
    "ipAddress" TEXT,
    "userAgent" TEXT,
    "userId" TEXT NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    "createdAt" TEXT NOT NULL DEFAULT (datetime('now')),
    "updatedAt" TEXT NOT NULL DEFAULT (datetime('now'))
);
```

**If you already created tables with snake_case:** Use ALTER TABLE RENAME COLUMN:

```sql
ALTER TABLE "user" RENAME COLUMN "email_verified" TO "emailVerified";
ALTER TABLE "session" RENAME COLUMN "user_id" TO "userId";
-- etc.
```

**App tables should still use snake_case.** Only Better Auth tables (`user`, `session`, `account`, `verification`) need camelCase.

## 2. Per-Request Auth Instantiation

**Problem:** Workers are stateless. Creating `betterAuth()` at module scope won't work because `env.DB` is only available inside request handlers.

**Wrong:**
```typescript
// Module scope - env doesn't exist here
const auth = betterAuth({ database: { dialect: new D1Dialect({ database: env.DB }) } });
```

**Right:**
```typescript
// Factory function called per-request
export function createAuth(env: Env) {
  return betterAuth({
    database: { dialect: new D1Dialect({ database: env.DB }), type: 'sqlite' },
    secret: env.BETTER_AUTH_SECRET,
    // ...
  });
}

// In handler:
const auth = createAuth(c.env);
```

## 3. nodejs_compat Flag Required

**Problem:** Better Auth uses Node.js crypto APIs internally. Without the compat flag, you get runtime errors like `crypto.randomBytes is not a function`.

**Solution:** Add to `wrangler.toml`:

```toml
compatibility_flags = ["nodejs_compat"]
```

## 4. CORS Credentials

**Problem:** Auth cookies don't get sent/received without proper CORS + fetch configuration.

**Both sides must be configured:**

Backend (Hono CORS middleware):
```typescript
cors({
  credentials: true,
  // ...
})
```

Frontend (fetch calls):
```typescript
fetch(url, {
  credentials: 'include',
  // ...
});
```

If either side is missing, session cookies won't work and auth will silently fail.

## 5. Static Asset Serving

**Problem:** How does one Worker serve both API responses and React SPA files?

**Solution:** Use the `[assets]` binding in `wrangler.toml`:

```toml
[assets]
directory = "../web/dist"
binding = "ASSETS"
```

This makes the built web files available as static assets. The Worker handles API routes through Hono, and Cloudflare automatically serves matched static files.

## 6. SPA Fallback for Client-Side Routing

**Problem:** Navigating directly to `/dashboard` returns 404 because there's no `dashboard.html` file.

**Solution:** The `notFound` handler serves `index.html` for any non-API path:

```typescript
app.notFound(async (c) => {
  if (c.req.path.startsWith('/api/')) {
    return c.json({ error: 'Not found' }, 404);
  }
  try {
    return await c.env.ASSETS.fetch(new Request(new URL('/index.html', c.req.url)));
  } catch {
    return c.json({ error: 'Not found' }, 404);
  }
});
```

This ensures React Router handles all client-side routes while API 404s return proper JSON errors.

## 7. D1 Batch Limitations

**Problem:** Need to insert a parent record and multiple child records atomically.

**Solution:** Use `DB.batch()` for atomic multi-statement operations:

```typescript
const statements = [
  c.env.DB.prepare('INSERT INTO "order" (id, ...) VALUES (?, ...)').bind(orderId, ...),
  ...lines.map(line =>
    c.env.DB.prepare('INSERT INTO order_item (id, order_id, ...) VALUES (?, ?, ...)')
      .bind(itemId, orderId, ...)
  ),
];
await c.env.DB.batch(statements);
```

**Limitation:** Batch returns are limited. Don't rely on return values from individual batch statements for complex logic. If you need the results, query after the batch.

## 8. Secret Management

**Problem:** `BETTER_AUTH_SECRET` must never be in source control but is required for production.

| Environment | Method |
|-------------|--------|
| Local dev | `[env.dev.vars]` in `wrangler.toml` |
| Production | `wrangler secret put BETTER_AUTH_SECRET` |

The `[vars]` section in the top-level `wrangler.toml` is for non-secret values like `ENVIRONMENT` and `BETTER_AUTH_URL`.

## 9. Custom Headers in CORS

**Problem:** `X-Organization-Id` header gets blocked by CORS preflight.

**Solution:** Explicitly list custom headers in `allowHeaders`:

```typescript
cors({
  allowHeaders: ['Content-Type', 'Authorization', 'X-Organization-Id'],
  // ...
})
```

Any custom header used by the frontend must be listed here.

## 10. D1 Migrations Directory

**Problem:** Migrations not found or applied from wrong location.

**Solution:** The `migrations_dir` in `wrangler.toml` must match the actual directory path:

```toml
[[d1_databases]]
migrations_dir = "src/db/migrations"
```

Files are numbered sequentially: `0001_initial.sql`, `0002_add_feature.sql`. Wrangler tracks which migrations have been applied and only runs new ones.

## 11. Build Order for Deployment

**Problem:** `wrangler deploy` needs the web `dist/` directory to exist for the `[assets]` binding.

**Solution:** Always build web first, then deploy the worker:

```bash
npm run build:web && npm run deploy --workspace=packages/api
```

The root `deploy` script handles this: `"deploy": "npm run build:web && npm run deploy --workspace=packages/api"`

## 12. D1 Database Creation

**Problem:** `wrangler.toml` needs a `database_id` but you don't have one yet.

**Solution:** Create the database first, then copy the ID:

```bash
npx wrangler d1 create <app-name>-db
# Output includes: database_id = "abc-123-..."
# Copy this into wrangler.toml
```

For local dev, use `database_id = "local"` in the `[env.dev]` section.

## 13. Better Auth `account` Table Name Conflict

**Problem:** Better Auth creates a table named `account` for OAuth provider data (Google, GitHub, credential accounts). If your app has its own "accounts" concept, you can't also name that table `account`.

**Solution:** Name your app's account table differently. For example: `user_account`, `app_account`, `ledger_account`, `billing_account`, etc.

The Better Auth `account` table has an `accountId` column that refers to the external provider's identifier (not your app's account ID).

## 14. Local Development Secrets with `.dev.vars`

**Problem:** The scaffold puts `BETTER_AUTH_SECRET` in `[env.dev.vars]` in `wrangler.toml` and uses `wrangler dev --env dev` to pick it up. An alternative approach is the `.dev.vars` file.

**Alternative:** Create a `.dev.vars` file in `packages/api/` (gitignored) with local secrets:

```
BETTER_AUTH_SECRET=development-secret-change-in-production
BETTER_AUTH_URL=http://localhost:8787
```

If using `.dev.vars`, the dev script can be just `wrangler dev` (without `--env dev`), since wrangler automatically reads `.dev.vars` in local development.

**Which approach to use:**
- `--env dev` with `[env.dev.vars]` in `wrangler.toml`: Simpler setup, dev config is version-controlled
- `.dev.vars` file: Better separation, each developer can have their own config
