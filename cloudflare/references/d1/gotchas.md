# D1 Gotchas & Troubleshooting

## Common Errors

### "SQL Injection Vulnerability"

**Cause:** Using string interpolation instead of prepared statements with bind()  
**Solution:** ALWAYS use prepared statements: `env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(userId).all()` instead of string interpolation which allows attackers to inject malicious SQL

### "no such table"

**Cause:** Table doesn't exist because migrations haven't been run, or using wrong database binding  
**Solution:** Run migrations using `wrangler d1 migrations apply <db-name> --remote` and verify binding name in wrangler.jsonc matches code

### "UNIQUE constraint failed"

**Cause:** Attempting to insert duplicate value in column with UNIQUE constraint  
**Solution:** Catch error and return 409 Conflict status code

### "Query Timeout (30s exceeded)"

**Cause:** Query execution exceeds 30 second timeout limit  
**Solution:** Break into smaller queries, add indexes to speed up queries, or reduce dataset size

### "N+1 Query Problem"

**Cause:** Making multiple individual queries in a loop instead of single optimized query  
**Solution:** Use JOIN to fetch related data in single query or use `batch()` method for multiple queries

### "Missing Indexes"

**Cause:** Queries performing full table scans without indexes  
**Solution:** Use `EXPLAIN QUERY PLAN` to check if index is used, then create index with `CREATE INDEX idx_users_email ON users(email)`

### "Boolean Type Issues"

**Cause:** SQLite uses INTEGER (0/1) not native boolean type  
**Solution:** Bind 1 or 0 instead of true/false when working with boolean values

### "Date/Time Type Issues"

**Cause:** SQLite doesn't have native DATE/TIME types  
**Solution:** Use TEXT (ISO 8601 format) or INTEGER (unix timestamp) for date/time values

## Plan Tier Limits

| Limit | Free Tier | Paid Plans | Notes |
|-------|-----------|------------|-------|
| Database size | 500 MB | 10 GB | Design for multiple DBs per tenant on paid |
| Row size | 1 MB | 1 MB | Store large files in R2, not D1 |
| Query timeout | 30s | 30s | Split long-running work into smaller statements |
| Batch size | 1,000 statements | 10,000 statements | Split large batches accordingly |
| Time Travel | 7 days | 30 days | Point-in-time recovery window |
| Read replication | ✅ Available | ✅ Available | Enable per database; no extra storage or compute cost |
| Sessions API | ✅ Available | ✅ Available | `withSession()` for read-replica consistency |
| Concurrent requests | 10,000/min | Higher | Contact support for custom limits |

## Production Gotchas

### "Batch size exceeded"

**Cause:** Attempting to send >1,000 statements on free tier or >10,000 on paid  
**Solution:** Chunk batches: `for (let i = 0; i < stmts.length; i += MAX_BATCH) await env.DB.batch(stmts.slice(i, i + MAX_BATCH))`

### "Replication lag causing stale reads"

**Cause:** Reading from a replica immediately after write - replication lag can be 100ms-2s  
**Solution:** Start the session at the primary for read-after-write: `env.DB.withSession('first-primary')`, or pass a previous `session.getBookmark()` value

### "Migration applied to local but not remote"

**Cause:** Forgot `--remote` flag when applying migrations  
**Solution:** Always run `wrangler d1 migrations apply <db-name> --remote` for production

### "Foreign key constraint failed"

**Cause:** Inserting row with FK to non-existent parent, or deleting parent before children  
**Solution:** Enable FK enforcement: `PRAGMA foreign_keys = ON;` and use ON DELETE CASCADE in schema

### "BLOB data corrupted on export"

**Cause:** D1 export may not handle BLOB correctly  
**Solution:** Store binary files in R2, only store R2 URLs/keys in D1

### "Database size approaching limit"

**Cause:** Storing too much data in single database  
**Solution:** Horizontal scale-out: create per-tenant/per-user databases, archive old data, or upgrade to paid plan

### "Local dev vs production behavior differs"

**Cause:** Local uses SQLite file, production uses distributed D1 - different performance/limits  
**Solution:** Always test migrations on remote with `--remote` flag before production rollout
