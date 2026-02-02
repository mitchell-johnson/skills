# Scaffold Guide

Step-by-step instructions to generate a new Cloudflare Workers full-stack app. Replace `<app-name>` with your project name throughout.

## 1. Root Monorepo Setup

### `package.json`

```json
{
  "name": "<app-name>",
  "version": "0.1.0",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "dev": "npm run dev --workspace=packages/api & npm run dev --workspace=packages/web",
    "dev:api": "npm run dev --workspace=packages/api",
    "dev:web": "npm run dev --workspace=packages/web",
    "build": "npm run build:web && npm run build:api",
    "build:api": "npm run build --workspace=packages/api",
    "build:web": "npm run build --workspace=packages/web",
    "deploy": "npm run build:web && npm run deploy --workspace=packages/api",
    "db:migrate": "npm run db:migrate --workspace=packages/api",
    "typecheck": "npm run typecheck --workspaces",
    "test:e2e": "npx playwright test"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.58.0"
  }
}
```

### `.gitignore`

```
node_modules/
dist/
.wrangler/
.dev.vars
.env
.env.local
.env.*.local
*.log
.DS_Store
coverage/
e2e/auth-state/
e2e/test-results/
e2e/playwright-report/
*.tsbuildinfo
test-results/
```

---

## 2. API Package (`packages/api`)

### `packages/api/package.json`

```json
{
  "name": "@<app-name>/api",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "wrangler dev --env dev",
    "build": "wrangler deploy --dry-run --outdir=dist",
    "deploy": "wrangler deploy",
    "db:migrate": "wrangler d1 migrations apply <app-name>-db --local",
    "db:migrate:prod": "wrangler d1 migrations apply <app-name>-db --remote",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "better-auth": "^1.2.0",
    "hono": "^4.4.0",
    "kysely-d1": "^0.4.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20240620.0",
    "typescript": "^5.5.0",
    "vitest": "^1.6.0",
    "wrangler": "^3.60.0"
  }
}
```

### `packages/api/wrangler.toml`

```toml
name = "<app-name>"
main = "src/index.ts"
compatibility_date = "2024-06-01"
compatibility_flags = ["nodejs_compat"]

[assets]
directory = "../web/dist"
binding = "ASSETS"

[vars]
ENVIRONMENT = "production"
BETTER_AUTH_URL = "https://<app-name>.<your-subdomain>.workers.dev"

# BETTER_AUTH_SECRET is set via `wrangler secret put`

[[d1_databases]]
binding = "DB"
database_name = "<app-name>-db"
database_id = "<create-with-wrangler-d1-create>"
migrations_dir = "src/db/migrations"

# Local development overrides
[env.dev]
[env.dev.vars]
ENVIRONMENT = "development"
BETTER_AUTH_SECRET = "development-secret-change-in-production"
BETTER_AUTH_URL = "http://localhost:8787"

[env.dev.assets]
directory = "../web/dist"
binding = "ASSETS"

[[env.dev.d1_databases]]
binding = "DB"
database_name = "<app-name>-db"
database_id = "local"
migrations_dir = "src/db/migrations"
```

### `packages/api/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ESNext"],
    "types": ["@cloudflare/workers-types"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### `packages/api/src/lib/types.ts`

```typescript
export interface Env {
  DB: D1Database;
  ASSETS: Fetcher;
  ENVIRONMENT: string;
  BETTER_AUTH_SECRET: string;
  BETTER_AUTH_URL: string;
}

export interface PaginationParams {
  page: number;
  limit: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

export interface OrgContext {
  organizationId: string;
  userId: string;
  role: string;
}
```

### `packages/api/src/lib/auth.ts`

```typescript
import { betterAuth } from 'better-auth';
import { D1Dialect } from 'kysely-d1';
import type { Env } from './types';

export function createAuth(env: Env) {
  return betterAuth({
    database: {
      dialect: new D1Dialect({ database: env.DB }),
      type: 'sqlite',
    },
    secret: env.BETTER_AUTH_SECRET,
    baseURL: env.BETTER_AUTH_URL,
    emailAndPassword: {
      enabled: true,
    },
    session: {
      expiresIn: 60 * 60 * 24 * 7, // 7 days
      updateAge: 60 * 60 * 24, // 1 day
    },
    trustedOrigins: [env.BETTER_AUTH_URL],
  });
}

export type Auth = ReturnType<typeof createAuth>;
```

### `packages/api/src/lib/utils.ts`

```typescript
export function generateId(): string {
  return crypto.randomUUID();
}

export function getPaginationParams(url: URL): { page: number; limit: number; offset: number } {
  const page = Math.max(1, parseInt(url.searchParams.get('page') || '1', 10));
  const limit = Math.min(100, Math.max(1, parseInt(url.searchParams.get('limit') || '50', 10)));
  const offset = (page - 1) * limit;
  return { page, limit, offset };
}

export function paginatedResponse<T>(data: T[], total: number, page: number, limit: number) {
  return {
    data,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  };
}

export function formatDate(date: Date): string {
  return date.toISOString().split('T')[0];
}

export function now(): string {
  return new Date().toISOString().replace('T', ' ').substring(0, 19);
}
```

### `packages/api/src/lib/validators.ts`

```typescript
import { z } from 'zod';

// Shared primitives
const isoDateString = z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Must be YYYY-MM-DD');
const nonEmpty = (max = 255) => z.string().trim().min(1).max(max);
const optionalString = (max = 255) => z.string().trim().max(max).optional();
const idRef = nonEmpty(64);

// Validate helper
export type ValidationResult<T> =
  | { success: true; data: T }
  | { success: false; error: string };

export function validate<T>(schema: z.ZodSchema<T>, data: unknown): ValidationResult<T> {
  const result = schema.safeParse(data);
  if (result.success) {
    return { success: true, data: result.data };
  }
  const messages = result.error.issues.map((issue) => {
    const path = issue.path.length > 0 ? issue.path.join('.') : '(root)';
    return `${path}: ${issue.message}`;
  });
  return { success: false, error: messages.join('; ') };
}

// Export primitives for use in route-specific schemas
export { z, isoDateString, nonEmpty, optionalString, idRef };
```

### `packages/api/src/middleware/auth.ts`

```typescript
import { Context, Next } from 'hono';
import { createAuth } from '../lib/auth';
import type { Env } from '../lib/types';

export async function authMiddleware(c: Context<{ Bindings: Env }>, next: Next) {
  const auth = createAuth(c.env);

  const session = await auth.api.getSession({
    headers: c.req.raw.headers,
  });

  if (!session) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  c.set('user', session.user);
  c.set('session', session.session);
  await next();
}
```

### `packages/api/src/middleware/org-context.ts` (optional, for multi-tenant apps)

```typescript
import { Context, Next } from 'hono';
import type { Env } from '../lib/types';

export async function orgContextMiddleware(c: Context<{ Bindings: Env }>, next: Next) {
  const organizationId = c.req.header('X-Organization-Id');

  if (!organizationId) {
    return c.json({ error: 'X-Organization-Id header is required' }, 400);
  }

  const user = c.get('user');

  const member = await c.env.DB.prepare(
    'SELECT role FROM organization_member WHERE organization_id = ? AND user_id = ?'
  )
    .bind(organizationId, user.id)
    .first();

  if (!member) {
    return c.json({ error: 'You do not have access to this organization' }, 403);
  }

  c.set('organizationId', organizationId);
  c.set('memberRole', member.role);
  await next();
}
```

### `packages/api/src/middleware/role-auth.ts` (optional, for role-based access)

```typescript
import { Context, Next } from 'hono';
import type { Env } from '../lib/types';

type Role = 'owner' | 'admin' | 'member' | 'viewer';

const ROLE_HIERARCHY: Record<Role, number> = {
  owner: 100,
  admin: 90,
  member: 50,
  viewer: 10,
};

export function requireRole(...allowedRoles: Role[]) {
  const minLevel = Math.min(...allowedRoles.map(r => ROLE_HIERARCHY[r]));

  return async (c: Context<{ Bindings: Env }>, next: Next) => {
    const memberRole = c.get('memberRole') as Role;
    if (!memberRole) {
      return c.json({ error: 'No role assigned' }, 403);
    }
    const userLevel = ROLE_HIERARCHY[memberRole] ?? 0;
    if (userLevel >= minLevel) {
      await next();
    } else {
      return c.json({ error: 'Insufficient permissions' }, 403);
    }
  };
}
```

### `packages/api/src/routes/auth.ts`

```typescript
import { Hono } from 'hono';
import { createAuth } from '../lib/auth';
import type { Env } from '../lib/types';

const auth = new Hono<{ Bindings: Env }>();

auth.all('/auth/*', async (c) => {
  const authInstance = createAuth(c.env);
  return authInstance.handler(c.req.raw);
});

export default auth;
```

### `packages/api/src/routes/example.ts` (CRUD route template)

```typescript
import { Hono } from 'hono';
import type { Env } from '../lib/types';
import { generateId, getPaginationParams, paginatedResponse, now } from '../lib/utils';
import { validate, z, nonEmpty, optionalString } from '../lib/validators';

const items = new Hono<{ Bindings: Env }>();

// Define schemas
const createSchema = z.object({
  name: nonEmpty(200),
  description: optionalString(1000),
});

// LIST with pagination
items.get('/', async (c) => {
  const organizationId = c.get('organizationId');
  const { page, limit, offset } = getPaginationParams(new URL(c.req.url));

  const countResult = await c.env.DB.prepare(
    'SELECT COUNT(*) as total FROM item WHERE organization_id = ?'
  ).bind(organizationId).first<{ total: number }>();

  const rows = await c.env.DB.prepare(
    'SELECT * FROM item WHERE organization_id = ? ORDER BY created_at DESC LIMIT ? OFFSET ?'
  ).bind(organizationId, limit, offset).all();

  return c.json(paginatedResponse(rows.results, countResult?.total ?? 0, page, limit));
});

// GET by ID
items.get('/:id', async (c) => {
  const organizationId = c.get('organizationId');
  const id = c.req.param('id');

  const row = await c.env.DB.prepare(
    'SELECT * FROM item WHERE id = ? AND organization_id = ?'
  ).bind(id, organizationId).first();

  if (!row) return c.json({ error: 'Not found' }, 404);
  return c.json({ data: row });
});

// CREATE
items.post('/', async (c) => {
  const organizationId = c.get('organizationId');
  const body = await c.req.json();
  const result = validate(createSchema, body);
  if (!result.success) return c.json({ error: result.error }, 400);

  const id = generateId();
  const { name, description } = result.data;

  await c.env.DB.prepare(
    'INSERT INTO item (id, organization_id, name, description, created_at, updated_at) VALUES (?, ?, ?, ?, ?, ?)'
  ).bind(id, organizationId, name, description ?? null, now(), now()).run();

  const row = await c.env.DB.prepare('SELECT * FROM item WHERE id = ?').bind(id).first();
  return c.json({ data: row }, 201);
});

// UPDATE
items.put('/:id', async (c) => {
  const organizationId = c.get('organizationId');
  const id = c.req.param('id');
  const body = await c.req.json();

  const existing = await c.env.DB.prepare(
    'SELECT * FROM item WHERE id = ? AND organization_id = ?'
  ).bind(id, organizationId).first();
  if (!existing) return c.json({ error: 'Not found' }, 404);

  // Build dynamic update
  const updates: string[] = [];
  const values: unknown[] = [];
  if (body.name !== undefined) { updates.push('name = ?'); values.push(body.name); }
  if (body.description !== undefined) { updates.push('description = ?'); values.push(body.description); }

  if (updates.length === 0) return c.json({ data: existing });

  updates.push('updated_at = ?');
  values.push(now());
  values.push(id);
  values.push(organizationId);

  await c.env.DB.prepare(
    `UPDATE item SET ${updates.join(', ')} WHERE id = ? AND organization_id = ?`
  ).bind(...values).run();

  const row = await c.env.DB.prepare('SELECT * FROM item WHERE id = ?').bind(id).first();
  return c.json({ data: row });
});

// DELETE
items.delete('/:id', async (c) => {
  const organizationId = c.get('organizationId');
  const id = c.req.param('id');

  const existing = await c.env.DB.prepare(
    'SELECT * FROM item WHERE id = ? AND organization_id = ?'
  ).bind(id, organizationId).first();
  if (!existing) return c.json({ error: 'Not found' }, 404);

  await c.env.DB.prepare('DELETE FROM item WHERE id = ?').bind(id).run();
  return c.json({ success: true });
});

export default items;
```

### `packages/api/src/index.ts`

```typescript
import { Hono } from 'hono';
import { cors } from 'hono/cors';
import { logger } from 'hono/logger';
import type { Env } from './lib/types';
import { authMiddleware } from './middleware/auth';
// Uncomment for multi-tenant apps:
// import { orgContextMiddleware } from './middleware/org-context';

import authRoutes from './routes/auth';
// import items from './routes/example';

const app = new Hono<{ Bindings: Env }>();

// Global middleware
app.use('*', cors({
  origin: (origin) => {
    if (!origin) return '';
    if (origin.startsWith('http://localhost:')) return origin;
    return '';
  },
  credentials: true,
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization', 'X-Organization-Id'],
}));
app.use('*', logger());

// Health check
app.get('/api/health', (c) => c.json({ status: 'ok', version: '0.1.0' }));

// Auth routes (no auth middleware needed)
app.route('/api', authRoutes);

// Protected routes - require authentication
app.use('/api/*', authMiddleware);

// Add your authenticated routes here:
// app.route('/api/items', items);

// For org-scoped routes (multi-tenant):
// app.use('/api/org/*', orgContextMiddleware);
// app.route('/api/org/items', items);

// API 404 + SPA fallback
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

// Error handler
app.onError((err, c) => {
  console.error('Unhandled error:', err);
  return c.json({ error: 'Internal server error' }, 500);
});

export default app;
```

### `packages/api/src/db/migrations/0001_initial.sql`

```sql
-- Better Auth tables (camelCase columns required)
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

CREATE TABLE IF NOT EXISTS "account" (
    id TEXT PRIMARY KEY,
    "accountId" TEXT NOT NULL,
    "providerId" TEXT NOT NULL,
    "userId" TEXT NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    "accessToken" TEXT,
    "refreshToken" TEXT,
    "idToken" TEXT,
    "accessTokenExpiresAt" TEXT,
    "refreshTokenExpiresAt" TEXT,
    scope TEXT,
    password TEXT,
    "createdAt" TEXT NOT NULL DEFAULT (datetime('now')),
    "updatedAt" TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS "verification" (
    id TEXT PRIMARY KEY,
    identifier TEXT NOT NULL,
    value TEXT NOT NULL,
    "expiresAt" TEXT NOT NULL,
    "createdAt" TEXT NOT NULL DEFAULT (datetime('now')),
    "updatedAt" TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Add your app tables below. Use snake_case for app tables.
-- Example:
-- CREATE TABLE IF NOT EXISTS item (
--     id TEXT PRIMARY KEY,
--     organization_id TEXT NOT NULL,
--     name TEXT NOT NULL,
--     description TEXT,
--     created_at TEXT NOT NULL DEFAULT (datetime('now')),
--     updated_at TEXT NOT NULL DEFAULT (datetime('now'))
-- );
```

---

## 3. Web Package (`packages/web`)

### `packages/web/package.json`

```json
{
  "name": "@<app-name>/web",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.24.0",
    "@tanstack/react-query": "^5.45.0",
    "zustand": "^4.5.2",
    "lucide-react": "^0.396.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38",
    "tailwindcss": "^3.4.4",
    "typescript": "^5.5.0",
    "vite": "^5.3.1"
  }
}
```

### `packages/web/vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8787',
        changeOrigin: true,
      },
    },
  },
});
```

### `packages/web/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["src/**/*.test.ts", "src/**/*.test.tsx"]
}
```

### `packages/web/tailwind.config.js`

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eff6ff',
          100: '#dbeafe',
          200: '#bfdbfe',
          300: '#93c5fd',
          400: '#60a5fa',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          800: '#1e40af',
          900: '#1e3a8a',
          950: '#172554',
        },
      },
    },
  },
  plugins: [],
};
```

### `packages/web/postcss.config.js`

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### `packages/web/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title><App Name></title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `packages/web/src/index.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  body {
    @apply bg-gray-50 text-gray-900 antialiased;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  }
}

@layer components {
  .btn {
    @apply inline-flex items-center justify-center rounded-md px-4 py-2 text-sm font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed;
  }
  .btn-primary {
    @apply btn bg-brand-600 text-white hover:bg-brand-700 focus:ring-brand-500;
  }
  .btn-secondary {
    @apply btn bg-white text-gray-700 border border-gray-300 hover:bg-gray-50 focus:ring-brand-500;
  }
  .btn-danger {
    @apply btn bg-red-600 text-white hover:bg-red-700 focus:ring-red-500;
  }
  .btn-ghost {
    @apply btn text-gray-600 hover:bg-gray-100 hover:text-gray-900;
  }
  .input {
    @apply block w-full rounded-md border border-gray-300 px-3 py-2 text-sm shadow-sm placeholder:text-gray-400 focus:border-brand-500 focus:outline-none focus:ring-1 focus:ring-brand-500;
  }
  .label {
    @apply block text-sm font-medium text-gray-700 mb-1;
  }
  .card {
    @apply bg-white rounded-lg border border-gray-200 shadow-sm;
  }
}
```

### `packages/web/src/main.tsx`

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App';
import './index.css';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </QueryClientProvider>
  </React.StrictMode>
);
```

### `packages/web/src/api/client.ts`

```typescript
const API_BASE = '/api';

interface RequestOptions {
  method?: string;
  body?: unknown;
  headers?: Record<string, string>;
}

class ApiClient {
  private organizationId: string | null = null;

  setOrganizationId(id: string | null) {
    this.organizationId = id;
  }

  getOrganizationId() {
    return this.organizationId;
  }

  private async request<T>(path: string, options: RequestOptions = {}): Promise<T> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...options.headers,
    };

    if (this.organizationId) {
      headers['X-Organization-Id'] = this.organizationId;
    }

    const response = await fetch(`${API_BASE}${path}`, {
      method: options.method || 'GET',
      headers,
      body: options.body ? JSON.stringify(options.body) : undefined,
      credentials: 'include',
    });

    if (!response.ok) {
      const error = await response.json().catch(() => ({ error: 'Request failed' }));
      throw new Error((error as { error: string }).error || `HTTP ${response.status}`);
    }

    return response.json();
  }

  // Auth
  auth = {
    signUp: (data: { name: string; email: string; password: string }) =>
      this.request('/auth/sign-up/email', { method: 'POST', body: data }),
    signIn: (data: { email: string; password: string }) =>
      this.request('/auth/sign-in/email', { method: 'POST', body: data }),
    signOut: () => this.request('/auth/sign-out', { method: 'POST' }),
    getSession: () => this.request<{ session: any; user: any }>('/auth/get-session'),
  };

  // Add resource-specific methods here
}

export const api = new ApiClient();
```

### `packages/web/src/store/index.ts`

```typescript
import { create } from 'zustand';
import { api } from '../api/client';

interface User {
  id: string;
  name: string;
  email: string;
}

interface AppState {
  user: User | null;
  sidebarOpen: boolean;

  setUser: (user: User | null) => void;
  toggleSidebar: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  user: null,
  sidebarOpen: true,

  setUser: (user) => set({ user }),
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}));
```

### `packages/web/src/App.tsx`

```tsx
import { useEffect, useState } from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import { useAppStore } from './store';
import { api } from './api/client';
import Login from './pages/Login';
import Register from './pages/Register';
// import Layout from './components/Layout';
// import Dashboard from './pages/Dashboard';

function App() {
  const { user, setUser } = useAppStore();
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function checkSession() {
      try {
        const session = await api.auth.getSession();
        if (session?.user) {
          setUser(session.user);
        }
      } catch {
        // Not authenticated
      } finally {
        setLoading(false);
      }
    }
    checkSession();
  }, [setUser]);

  if (loading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-brand-600" />
      </div>
    );
  }

  if (!user) {
    return (
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
        <Route path="*" element={<Navigate to="/login" replace />} />
      </Routes>
    );
  }

  return (
    <Routes>
      {/* Add authenticated routes here */}
      <Route path="/" element={<div className="p-8"><h1 className="text-2xl font-bold">Welcome, {user.name}</h1></div>} />
      <Route path="/login" element={<Navigate to="/" replace />} />
      <Route path="/register" element={<Navigate to="/" replace />} />
    </Routes>
  );
}

export default App;
```

### `packages/web/src/pages/Login.tsx`

```tsx
import { useState } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { api } from '../api/client';
import { useAppStore } from '../store';

export default function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();
  const { setUser } = useAppStore();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      await api.auth.signIn({ email, password });
      const session = await api.auth.getSession();
      if (session?.user) {
        setUser(session.user);
        navigate('/');
      }
    } catch (err: any) {
      setError(err.message || 'Invalid email or password');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="w-full max-w-md">
        <div className="text-center mb-8">
          <h1 className="text-2xl font-bold text-gray-900">Sign in</h1>
        </div>

        <form onSubmit={handleSubmit} className="card p-6 space-y-4">
          {error && (
            <div className="p-3 text-sm text-red-700 bg-red-50 border border-red-200 rounded-md">
              {error}
            </div>
          )}

          <div>
            <label className="label">Email</label>
            <input type="email" className="input" value={email}
              onChange={(e) => setEmail(e.target.value)} placeholder="you@example.com" required />
          </div>

          <div>
            <label className="label">Password</label>
            <input type="password" className="input" value={password}
              onChange={(e) => setPassword(e.target.value)} placeholder="Your password" required />
          </div>

          <button type="submit" className="btn-primary w-full" disabled={loading}>
            {loading ? 'Signing in...' : 'Sign in'}
          </button>

          <p className="text-sm text-center text-gray-500">
            Don't have an account?{' '}
            <Link to="/register" className="text-brand-600 hover:text-brand-700 font-medium">
              Create one
            </Link>
          </p>
        </form>
      </div>
    </div>
  );
}
```

### `packages/web/src/pages/Register.tsx`

```tsx
import { useState } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { api } from '../api/client';
import { useAppStore } from '../store';

export default function Register() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();
  const { setUser } = useAppStore();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      await api.auth.signUp({ name, email, password });
      const session = await api.auth.getSession();
      if (session?.user) {
        setUser(session.user);
        navigate('/');
      }
    } catch (err: any) {
      setError(err.message || 'Registration failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="w-full max-w-md">
        <div className="text-center mb-8">
          <h1 className="text-2xl font-bold text-gray-900">Create your account</h1>
        </div>

        <form onSubmit={handleSubmit} className="card p-6 space-y-4">
          {error && (
            <div className="p-3 text-sm text-red-700 bg-red-50 border border-red-200 rounded-md">
              {error}
            </div>
          )}

          <div>
            <label className="label">Full Name</label>
            <input type="text" className="input" value={name}
              onChange={(e) => setName(e.target.value)} placeholder="Your name" required />
          </div>

          <div>
            <label className="label">Email</label>
            <input type="email" className="input" value={email}
              onChange={(e) => setEmail(e.target.value)} placeholder="you@example.com" required />
          </div>

          <div>
            <label className="label">Password</label>
            <input type="password" className="input" value={password}
              onChange={(e) => setPassword(e.target.value)} placeholder="Min 8 characters"
              minLength={8} required />
          </div>

          <button type="submit" className="btn-primary w-full" disabled={loading}>
            {loading ? 'Creating account...' : 'Create account'}
          </button>

          <p className="text-sm text-center text-gray-500">
            Already have an account?{' '}
            <Link to="/login" className="text-brand-600 hover:text-brand-700 font-medium">
              Sign in
            </Link>
          </p>
        </form>
      </div>
    </div>
  );
}
```

### `packages/web/src/components/Layout.tsx` (app shell template)

```tsx
import { Outlet } from 'react-router-dom';

export default function Layout() {
  return (
    <div className="flex h-screen overflow-hidden">
      {/* Sidebar */}
      <aside className="w-64 bg-white border-r border-gray-200 flex flex-col">
        <div className="p-4 border-b border-gray-200">
          <h1 className="text-lg font-bold text-gray-900">App Name</h1>
        </div>
        <nav className="flex-1 p-4 space-y-1">
          {/* Add nav links */}
        </nav>
      </aside>

      {/* Main content */}
      <div className="flex flex-col flex-1 overflow-hidden">
        <header className="h-14 border-b border-gray-200 flex items-center px-6">
          {/* Header content */}
        </header>
        <main className="flex-1 overflow-y-auto p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
}
```

---

## 4. Post-Scaffold Steps

After generating all files:

```bash
# 1. Install dependencies
npm install

# 2. Create the D1 database
cd packages/api
npx wrangler d1 create <app-name>-db
# Copy the database_id into wrangler.toml

# 3. Run migrations locally
npm run db:migrate

# 4. Set the production secret
npx wrangler secret put BETTER_AUTH_SECRET
# Enter a strong random string

# 5. Start development
cd ../..
npm run dev
# API: http://localhost:8787
# Web: http://localhost:5173

# 6. Deploy
npm run deploy
```

---

## 5. E2E Test Setup (optional)

### `playwright.config.ts` (root)

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
  },
  webServer: {
    command: 'npm run dev',
    port: 5173,
    reuseExistingServer: true,
  },
});
```

### `e2e/auth.spec.ts` (example)

```typescript
import { test, expect } from '@playwright/test';

test('can register and login', async ({ page }) => {
  await page.goto('/register');
  await page.fill('input[type="text"]', 'Test User');
  await page.fill('input[type="email"]', `test-${Date.now()}@example.com`);
  await page.fill('input[type="password"]', 'password123');
  await page.click('button[type="submit"]');
  await expect(page).toHaveURL('/');
});
```
