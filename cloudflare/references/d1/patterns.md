# D1 Patterns & Best Practices

## Pagination

```typescript
async function getUsers({ page, pageSize }: { page: number; pageSize: number }, env: Env) {
  const offset = (page - 1) * pageSize;
  const [countResult, dataResult] = await env.DB.batch([
    env.DB.prepare('SELECT COUNT(*) as total FROM users'),
    env.DB.prepare('SELECT * FROM users ORDER BY created_at DESC LIMIT ? OFFSET ?').bind(pageSize, offset)
  ]);
  return { data: dataResult.results, total: countResult.results[0].total, page, pageSize, totalPages: Math.ceil(countResult.results[0].total / pageSize) };
}
```

## Conditional Queries

```typescript
async function searchUsers(filters: { name?: string; email?: string; active?: boolean }, env: Env) {
  const conditions: string[] = [], params: (string | number | boolean | null)[] = [];
  if (filters.name) { conditions.push('name LIKE ?'); params.push(`%${filters.name}%`); }
  if (filters.email) { conditions.push('email = ?'); params.push(filters.email); }
  if (filters.active !== undefined) { conditions.push('active = ?'); params.push(filters.active ? 1 : 0); }
  const whereClause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';
  return await env.DB.prepare(`SELECT * FROM users ${whereClause}`).bind(...params).all();
}
```

## Bulk Insert

```typescript
async function bulkInsertUsers(users: Array<{ name: string; email: string }>, env: Env) {
  const stmt = env.DB.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
  const batch = users.map(user => stmt.bind(user.name, user.email));
  return await env.DB.batch(batch);
}
```

## Caching with KV

```typescript
async function getCachedUser(userId: number, env: { DB: D1Database; CACHE: KVNamespace }) {
  const cacheKey = `user:${userId}`;
  const cached = await env.CACHE?.get(cacheKey, 'json');
  if (cached) return cached;
  const user = await env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(userId).first();
  if (user) await env.CACHE?.put(cacheKey, JSON.stringify(user), { expirationTtl: 300 });
  return user;
}
```

## Query Optimization

```typescript
// ✅ Use indexes in WHERE clauses
const users = await env.DB.prepare('SELECT * FROM users WHERE email = ?').bind(email).all();

// ✅ Limit result sets
const recentPosts = await env.DB.prepare('SELECT * FROM posts ORDER BY created_at DESC LIMIT 100').all();

// ✅ Use batch() for multiple independent queries
const [user, posts, comments] = await env.DB.batch([
  env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(userId),
  env.DB.prepare('SELECT * FROM posts WHERE user_id = ?').bind(userId),
  env.DB.prepare('SELECT * FROM comments WHERE user_id = ?').bind(userId)
]);

// ❌ Avoid N+1 queries
for (const post of posts) {
  const author = await env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(post.user_id).first(); // Bad: multiple round trips
}

// ✅ Use JOINs instead
const postsWithAuthors = await env.DB.prepare(`
  SELECT posts.*, users.name as author_name
  FROM posts
  JOIN users ON posts.user_id = users.id
`).all();
```

## Multi-Tenant SaaS

```typescript
// Each tenant gets own database
export default {
  async fetch(request: Request, env: { [key: `TENANT_${string}`]: D1Database }) {
    const tenantId = request.headers.get('X-Tenant-ID');
    const data = await env[`TENANT_${tenantId}`].prepare('SELECT * FROM records').all();
    return Response.json(data.results);
  }
}
```

## Session Storage

```typescript
async function createSession(userId: number, token: string, env: Env) {
  const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString();
  return await env.DB.prepare('INSERT INTO sessions (user_id, token, expires_at) VALUES (?, ?, ?)').bind(userId, token, expiresAt).run();
}

async function validateSession(token: string, env: Env) {
  return await env.DB.prepare('SELECT s.*, u.email FROM sessions s JOIN users u ON s.user_id = u.id WHERE s.token = ? AND s.expires_at > CURRENT_TIMESTAMP').bind(token).first();
}
```

## Analytics/Events

```typescript
async function logEvent(event: { type: string; userId?: number; metadata: object }, env: Env) {
  return await env.DB.prepare('INSERT INTO events (type, user_id, metadata) VALUES (?, ?, ?)').bind(event.type, event.userId || null, JSON.stringify(event.metadata)).run();
}

async function getEventStats(startDate: string, endDate: string, env: Env) {
  return await env.DB.prepare('SELECT type, COUNT(*) as count FROM events WHERE timestamp BETWEEN ? AND ? GROUP BY type ORDER BY count DESC').bind(startDate, endDate).all();
}
```

## Read Replication Pattern

Enable replication per database (dashboard or REST API) - there is no separate replica binding. Route reads through the Sessions API on the same `DB` binding.

```typescript
interface Env { DB: D1Database; }

export default {
  async fetch(request: Request, env: Env) {
    if (request.method === 'GET') {
      // Reads: session may be served by the nearest replica for lower latency
      const session = env.DB.withSession('first-unconstrained');
      const users = await session.prepare('SELECT * FROM users WHERE active = 1').all();
      return Response.json(users.results);
    }
    
    if (request.method === 'POST') {
      const { name, email } = await request.json();
      // Read-after-write: start the session at the primary for consistency (replication lag <100ms-2s)
      const session = env.DB.withSession('first-primary');
      const result = await session.prepare('INSERT INTO users (name, email) VALUES (?, ?)').bind(name, email).run();
      
      const user = await session.prepare('SELECT * FROM users WHERE id = ?').bind(result.meta.last_row_id).first();
      return Response.json(user, { status: 201 });
    }
  }
}
```

**Use `first-unconstrained` for**: Analytics dashboards, search results, public queries (eventual consistency OK)  
**Use `first-primary` for**: Read-after-write, financial transactions, authentication (consistency required)  
**Use a bookmark for**: Carrying consistency across requests - `session.getBookmark()` on one request, `env.DB.withSession(bookmark)` on the next

## Time Travel & Backups

```bash
wrangler d1 time-travel restore <db-name> --timestamp="2024-01-15T14:30:00Z"  # Point-in-time
wrangler d1 time-travel info <db-name>  # List restore points (7 days free, 30 days paid)
wrangler d1 export <db-name> --remote --output=./backup.sql  # Full export
wrangler d1 export <db-name> --remote --no-schema --output=./data.sql  # Data only
wrangler d1 execute <db-name> --remote --file=./backup.sql  # Import
```
