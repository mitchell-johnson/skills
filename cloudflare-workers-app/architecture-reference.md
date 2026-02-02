# Architecture Reference

Comprehensive patterns and conventions for the Cloudflare Workers full-stack app.

## Deployment Model

A single Cloudflare Worker serves both API and frontend:

```
Client Request
    │
    ├── /api/*     → Hono router → JSON responses
    │
    ├── /assets/*  → Static files from web dist (ASSETS binding)
    │
    └── /*         → SPA fallback (serves index.html for client-side routing)
```

The `[assets]` binding in `wrangler.toml` points to `../web/dist`. The Worker's `notFound` handler serves `index.html` for any non-API path, enabling React Router's client-side routing.

## Middleware Chain

Order matters. The chain in `index.ts`:

```
CORS → Logger → Health Check → Auth Routes (public)
  → Auth Middleware (all /api/* after this)
    → User-level routes (e.g., /api/organizations)
      → Org Context Middleware (all /api/org/*)
        → Org-scoped routes (e.g., /api/org/items)
```

Key points:
- Auth routes (`/api/auth/*`) are mounted **before** the auth middleware
- The auth middleware applies to all `/api/*` routes after it
- Org context middleware only applies to `/api/org/*` routes
- Routes mounted before a middleware are not affected by it

## Better Auth + D1 Integration

### Per-Request Auth Instance

Workers are stateless. The `env` object (containing D1 bindings) is only available per-request. Auth must be instantiated per-request:

```typescript
// In middleware or route handler:
const auth = createAuth(c.env);
const session = await auth.api.getSession({ headers: c.req.raw.headers });
```

Never create a global auth instance at module level.

### D1Dialect via Kysely

Better Auth uses Kysely internally. The D1Dialect adapter bridges Better Auth to D1:

```typescript
database: {
  dialect: new D1Dialect({ database: env.DB }),
  type: 'sqlite',
}
```

### Auth Route Handler

Better Auth provides a wildcard handler that handles all `/auth/*` routes (sign-up, sign-in, sign-out, get-session, etc.):

```typescript
auth.all('/auth/*', async (c) => {
  const authInstance = createAuth(c.env);
  return authInstance.handler(c.req.raw);
});
```

### Session Configuration

```typescript
session: {
  expiresIn: 60 * 60 * 24 * 7, // 7 days
  updateAge: 60 * 60 * 24,      // refresh if older than 1 day
},
trustedOrigins: [env.BETTER_AUTH_URL],
```

### Secret Management

- **Production:** `wrangler secret put BETTER_AUTH_SECRET` (stored encrypted)
- **Local dev:** `[env.dev.vars]` section in `wrangler.toml`
- **Never** hardcode production secrets in `wrangler.toml` or source

## Database Patterns

### Raw SQL with D1 Bindings

D1 uses a prepare/bind/execute pattern:

```typescript
// Single row
const row = await c.env.DB.prepare(
  'SELECT * FROM item WHERE id = ? AND organization_id = ?'
).bind(id, orgId).first();

// Multiple rows
const result = await c.env.DB.prepare(
  'SELECT * FROM item WHERE organization_id = ? LIMIT ? OFFSET ?'
).bind(orgId, limit, offset).all();
// Access: result.results (array)

// Count
const count = await c.env.DB.prepare(
  'SELECT COUNT(*) as total FROM item WHERE organization_id = ?'
).bind(orgId).first<{ total: number }>();

// Insert
await c.env.DB.prepare(
  'INSERT INTO item (id, name, created_at) VALUES (?, ?, ?)'
).bind(id, name, now()).run();

// Batch (atomic)
await c.env.DB.batch([
  c.env.DB.prepare('INSERT INTO parent (id, name) VALUES (?, ?)').bind(parentId, name),
  c.env.DB.prepare('INSERT INTO child (id, parent_id, value) VALUES (?, ?, ?)').bind(childId, parentId, val),
]);
```

### Dynamic WHERE Clauses

Build conditions and bind values dynamically:

```typescript
const conditions: string[] = ['organization_id = ?'];
const values: unknown[] = [orgId];

if (status) { conditions.push('status = ?'); values.push(status); }
if (search) { conditions.push('name LIKE ?'); values.push(`%${search}%`); }

const where = conditions.join(' AND ');
const rows = await c.env.DB.prepare(
  `SELECT * FROM item WHERE ${where} ORDER BY created_at DESC LIMIT ? OFFSET ?`
).bind(...values, limit, offset).all();
```

### Schema Conventions

- **Primary keys:** `TEXT` type, generated with `crypto.randomUUID()`
- **Dates:** `TEXT` type, stored as ISO format (`datetime('now')` default)
- **Booleans:** `INTEGER` (0/1) - SQLite has no boolean type
- **JSON data:** `TEXT` with JSON strings (e.g., tags, settings)
- **Foreign keys:** `TEXT REFERENCES table(id)` with `ON DELETE CASCADE` where appropriate

### Migration Management

- Files in `src/db/migrations/` numbered sequentially: `0001_initial.sql`, `0002_add_feature.sql`
- Apply locally: `wrangler d1 migrations apply <db-name> --local`
- Apply to production: `wrangler d1 migrations apply <db-name> --remote`

## API Route Patterns

### Route Module Structure

Each route file creates a new Hono instance with the correct type:

```typescript
import { Hono } from 'hono';
import type { Env } from '../lib/types';

const items = new Hono<{ Bindings: Env }>();

// Routes...

export default items;
```

### Context Extraction

```typescript
// User (set by auth middleware)
const user = c.get('user');

// Organization (set by org-context middleware)
const organizationId = c.get('organizationId');
const memberRole = c.get('memberRole');
```

### Validation Pattern

```typescript
const body = await c.req.json();
const result = validate(createSchema, body);
if (!result.success) return c.json({ error: result.error }, 400);
const { name, description } = result.data;
```

### Pagination Pattern

```typescript
const { page, limit, offset } = getPaginationParams(new URL(c.req.url));

const countResult = await c.env.DB.prepare(
  'SELECT COUNT(*) as total FROM item WHERE organization_id = ?'
).bind(orgId).first<{ total: number }>();

const rows = await c.env.DB.prepare(
  'SELECT * FROM item WHERE organization_id = ? LIMIT ? OFFSET ?'
).bind(orgId, limit, offset).all();

return c.json(paginatedResponse(rows.results, countResult?.total ?? 0, page, limit));
```

### Error Handling

```typescript
// 400 - validation error
return c.json({ error: result.error }, 400);

// 401 - not authenticated
return c.json({ error: 'Unauthorized' }, 401);

// 403 - not authorized
return c.json({ error: 'Insufficient permissions' }, 403);

// 404 - not found
if (!row) return c.json({ error: 'Not found' }, 404);

// Global error handler (in index.ts)
app.onError((err, c) => {
  console.error('Unhandled error:', err);
  return c.json({ error: 'Internal server error' }, 500);
});
```

### ID Generation

```typescript
import { generateId } from '../lib/utils';
const id = generateId(); // crypto.randomUUID()
```

## Advanced Route Patterns

### Status Transition Endpoints

For entities with state machines (e.g., draft → approved → paid → voided), use dedicated POST endpoints:

```typescript
// POST /items/:id/approve
items.post('/:id/approve', async (c) => {
  const organizationId = c.get('organizationId');
  const id = c.req.param('id');

  const item = await c.env.DB.prepare(
    'SELECT * FROM item WHERE id = ? AND organization_id = ?'
  ).bind(id, organizationId).first();

  if (!item) return c.json({ error: 'Not found' }, 404);
  if (item.status !== 'draft') return c.json({ error: 'Can only approve draft items' }, 400);

  await c.env.DB.prepare(
    'UPDATE item SET status = ?, updated_at = ? WHERE id = ?'
  ).bind('approved', now(), id).run();

  const updated = await c.env.DB.prepare('SELECT * FROM item WHERE id = ?').bind(id).first();
  return c.json({ data: updated });
});
```

### Child Record Updates (Delete-and-Reinsert)

When updating records with line items, delete existing children then re-insert:

```typescript
items.put('/:id', async (c) => {
  const { lines, ...parentData } = result.data;

  await c.env.DB.batch([
    // Delete old lines
    c.env.DB.prepare('DELETE FROM item_line WHERE item_id = ?').bind(id),
    // Update parent
    c.env.DB.prepare('UPDATE item SET name = ?, updated_at = ? WHERE id = ?')
      .bind(parentData.name, now(), id),
    // Insert new lines
    ...lines.map((line, index) =>
      c.env.DB.prepare(
        'INSERT INTO item_line (id, item_id, description, amount, sort_order) VALUES (?, ?, ?, ?, ?)'
      ).bind(generateId(), id, line.description, line.amount, index)
    ),
  ]);
});
```

### Audit Fields

Track who created/modified records using the authenticated user from context:

```typescript
const user = c.get('user');
await c.env.DB.prepare(
  'INSERT INTO item (id, organization_id, name, created_by, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?)'
).bind(id, organizationId, name, user.id, now(), now()).run();
```

### Route File Naming Convention

Use kebab-case for route files: `user-profiles.ts`, `task-lists.ts`, `project-settings.ts`. Mount with matching URL paths:

```typescript
app.route('/api/org/user-profiles', userProfiles);
app.route('/api/org/task-lists', taskLists);
```

## Multi-Tenancy Pattern (optional)

For apps with organizations/workspaces:

### Org Context Header

Frontend sends `X-Organization-Id` header with every org-scoped request. The org-context middleware validates membership before allowing access.

### Role Hierarchy

```typescript
const ROLE_HIERARCHY: Record<Role, number> = {
  owner: 100,
  admin: 90,
  member: 50,
  viewer: 10,
};
```

Usage: `requireRole('admin')` allows admin and owner (level >= 90).

### Route Organization

```
/api/auth/*           - Public auth routes
/api/organizations    - User-level (list orgs the user belongs to)
/api/org/*            - Org-scoped (requires X-Organization-Id header)
```

## Frontend Patterns

### API Client Singleton

A class-based singleton that:
- Manages the current organization context
- Injects `X-Organization-Id` header automatically
- Always sends `credentials: 'include'` for session cookies
- Provides typed methods per resource

### State Management Split

| Concern | Tool | Example |
|---------|------|---------|
| UI state | Zustand | sidebar open, current org, theme |
| Server state | React Query | list of items, user data |
| Form state | useState | input values during editing |

### Session Check on Load

`App.tsx` calls `api.auth.getSession()` on mount. If authenticated, sets user in Zustand store. Otherwise, redirects to `/login`.

### Protected Route Pattern

```tsx
if (!user) {
  return (
    <Routes>
      <Route path="/login" element={<Login />} />
      <Route path="/register" element={<Register />} />
      <Route path="*" element={<Navigate to="/login" replace />} />
    </Routes>
  );
}
// Authenticated routes below
```

### Form Pattern

```tsx
const [formState, setFormState] = useState({ name: '', description: '' });
const mutation = useMutation({
  mutationFn: (data) => api.items.create(data),
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['items'] });
    navigate('/items');
  },
});
```

### Tailwind Custom Component Classes

Defined in `index.css` using `@layer components`:
- `.btn` - base button styles
- `.btn-primary` / `.btn-secondary` / `.btn-danger` / `.btn-ghost` - button variants
- `.input` - form input styling
- `.label` - form label styling
- `.card` - white card with border and shadow

## Development Workflow

```bash
# Start both API and web dev servers
npm run dev
# API: http://localhost:8787
# Web: http://localhost:5173 (proxies /api → :8787)

# Apply DB migrations locally
npm run db:migrate

# Type check both packages
npm run typecheck

# Deploy (builds web, then deploys Worker)
npm run deploy

# Apply migrations to production
cd packages/api
npm run db:migrate:prod
```

### Vite Proxy

During development, Vite proxies `/api` requests to the Wrangler dev server:

```typescript
proxy: {
  '/api': {
    target: 'http://localhost:8787',
    changeOrigin: true,
  },
}
```

This means the frontend always uses relative paths (`/api/...`), which works in both dev and production.

## CORS Configuration

Production uses same-origin (Worker serves both API and SPA), so no CORS needed. The CORS middleware only allows `localhost:*` for development:

```typescript
origin: (origin) => {
  if (!origin) return '';                              // same-origin (production)
  if (origin.startsWith('http://localhost:')) return origin; // dev
  return '';                                            // block all others
},
credentials: true,
allowHeaders: ['Content-Type', 'Authorization', 'X-Organization-Id'],
```

`credentials: true` is required for session cookies. Add any custom headers (like `X-Organization-Id`) to `allowHeaders`.
