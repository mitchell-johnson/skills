---
name: mcp-cloudflare-worker
description: Builds MCP (Model Context Protocol) servers on Cloudflare Workers with OAuth 2.1 authentication (workers-oauth-provider or BetterAuth) and OpenTelemetry observability. Use when creating remote MCP servers, adding OAuth flows to MCP endpoints, storing per-user credentials for multi-tenant MCP servers, or instrumenting MCP Workers with distributed tracing.
---

# MCP Servers on Cloudflare Workers

## Overview

This skill covers building **production-ready MCP servers** on Cloudflare Workers:

- **MCP Protocol**: Remote MCP servers using Streamable HTTP transport
- **Auth**: OAuth 2.1 via `@cloudflare/workers-oauth-provider` or BetterAuth's MCP plugin
- **OpenTelemetry**: Distributed tracing and observability

**Reference implementations:**
- [akahu-mcp](https://github.com/mitchell-johnson/akahu-mcp) — Multi-user banking MCP with custom OAuth consent flow
- [floorsense](https://github.com/mitchell-johnson/floorsense) — Workplace MCP with D1 credential storage

**Related skill:** See [opentelemetry-js](../opentelemetry-js/SKILL.md) for detailed OTel patterns in Workers.

**Supporting files** (read the one that matches the task):

| File | Read when |
|------|-----------|
| `oauth-provider.md` | Adding OAuth with `@cloudflare/workers-oauth-provider`: full setup, custom consent flow that collects API keys, AES-GCM encryption, D1 credential storage |
| `betterauth.md` | Adding OAuth with BetterAuth's MCP plugin (full user accounts with sign-up/sign-in) |
| `observability.md` | Instrumenting the server with OpenTelemetry, or enabling Cloudflare native observability |
| `durable-object-patterns.md` | Caching in Durable Object storage, WebSocket keepalive with alarms |

## When to Use

- Building a remote MCP server (not stdio-based local server)
- Adding OAuth 2.1 authentication to MCP endpoints
- Deploying MCP servers to Cloudflare Workers
- Instrumenting MCP servers with OpenTelemetry
- Multi-tenant MCP servers with per-user credentials

**When NOT to use:**
- Local MCP servers using stdio transport (use the MCP SDK's stdio transport directly)
- Non-MCP Cloudflare Workers (use standard Workers patterns)
- Non-Workers MCP servers (Next.js, Express, etc. have different patterns)

## Quick Start: Authless MCP Server

Fastest path to a working remote MCP server:

```bash
npm create cloudflare@latest -- my-mcp-server \
  --template=cloudflare/ai/demos/remote-mcp-authless
cd my-mcp-server
npm start
# Server at http://localhost:8788/mcp
```

Test with MCP Inspector:
```bash
npx @modelcontextprotocol/inspector@latest
# Open http://localhost:5173, connect to http://localhost:8788/mcp
```

Deploy:
```bash
npx wrangler@latest deploy
```

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   MCP Client    │────▶│  Cloudflare Worker   │────▶│   External API  │
│ (Claude, etc.)  │     │  ┌────────────────┐  │     │  (your service) │
└─────────────────┘     │  │  OAuthProvider │  │     └─────────────────┘
                        │  │  ┌──────────┐  │  │
                        │  │  │ McpAgent │  │  │
                        │  │  │(Durable  │  │  │
                        │  │  │ Object)  │  │  │
                        │  │  └──────────┘  │  │
                        │  └────────────────┘  │
                        │  ┌────────────────┐  │
                        │  │ KV / D1 / R2   │  │
                        │  │ (credentials)  │  │
                        │  └────────────────┘  │
                        └──────────────────────┘
```

**Key components:**
- **OAuthProvider**: Wraps the Worker, handles OAuth 2.1 flows
- **McpAgent**: Durable Object that maintains MCP session state
- **McpServer**: Registers tools, resources, and prompts
- **Storage**: KV for OAuth state, D1 for credentials, Durable Object SQLite for session

**Transport:** Remote MCP uses **Streamable HTTP** (current MCP spec standard) — POST for client→server messages, optional SSE for server→client streaming, endpoint `/mcp` (configurable).

## Choosing an Auth Approach

| Approach | Choose when | Reference |
|----------|-------------|-----------|
| No auth | Internal/demo servers, no per-user state | This file (basic server below) |
| `@cloudflare/workers-oauth-provider` | You need OAuth but not user accounts — e.g. collect an API key at consent time and key everything off an opaque user ID | `oauth-provider.md` |
| BetterAuth MCP plugin | You need real user accounts (sign-up, sign-in, sessions) behind the OAuth flow | `betterauth.md` |

## Core Dependencies

```json
{
  "dependencies": {
    "@cloudflare/workers-oauth-provider": "^0.2.0",
    "@modelcontextprotocol/sdk": "^1.25.2",
    "agents": "^0.3.1",
    "hono": "^4.7.0",
    "zod": "^4.2.1"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20250109.0",
    "wrangler": "^4.56.0",
    "typescript": "^5.9.3"
  }
}
```

**Package roles:**
- `@cloudflare/workers-oauth-provider` — OAuth 2.1 server implementation
- `@modelcontextprotocol/sdk` — MCP protocol implementation
- `agents` — McpAgent Durable Object wrapper (Cloudflare's official)
- `hono` — Web framework for non-MCP routes
- `zod` — Tool input schema validation

## wrangler.jsonc Configuration

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-mcp-server",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-10",
  "compatibility_flags": ["nodejs_compat"],
  
  // Durable Object for MCP session state
  "durable_objects": {
    "bindings": [
      {
        "name": "MCP_OBJECT",
        "class_name": "MyMCP"
      }
    ]
  },
  
  // Migration for Durable Object SQLite
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["MyMCP"]
    }
  ],
  
  // OAuth state storage
  "kv_namespaces": [
    {
      "binding": "OAUTH_KV",
      "id": "your-kv-namespace-id"
    }
  ],
  
  // Optional: D1 for credential storage
  "d1_databases": [
    {
      "binding": "CREDENTIALS_DB",
      "database_name": "mcp-credentials",
      "database_id": "your-d1-database-id",
      "migrations_dir": "migrations"
    }
  ],
  
  // Enable native observability
  "observability": {
    "enabled": true
  }
}
```

Create KV namespace:
```bash
npx wrangler kv namespace create OAUTH_KV
# Copy the id to wrangler.jsonc
```

## Basic MCP Server (No Auth)

```typescript
// src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { McpAgent } from "agents/mcp";
import { Hono } from "hono";
import { z } from "zod";

// Type definitions
interface Env {
  MCP_OBJECT: DurableObjectNamespace;
}

// Hono app for non-MCP routes
const app = new Hono<{ Bindings: Env }>();

app.get("/", (c) => c.json({ name: "My MCP Server", version: "1.0.0" }));
app.get("/health", (c) => c.json({ status: "ok" }));

// MCP Agent Durable Object
export class MyMCP extends McpAgent<Env, Record<string, never>, {}> {
  server = new McpServer({
    name: "My MCP Server",
    version: "1.0.0",
  });

  async init() {
    // Register tools
    this.server.tool(
      "hello",
      "Say hello to someone",
      {
        name: z.string().describe("Name to greet"),
      },
      async ({ name }) => {
        return {
          content: [{ type: "text", text: `Hello, ${name}!` }],
        };
      }
    );

    this.server.tool(
      "get_time",
      "Get current UTC time",
      {},
      async () => {
        return {
          content: [{ type: "text", text: new Date().toISOString() }],
        };
      }
    );
  }
}

// Export Worker
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const url = new URL(request.url);
    
    // Route MCP requests to Durable Object
    if (url.pathname === "/mcp" || url.pathname.startsWith("/mcp/")) {
      const id = env.MCP_OBJECT.idFromName("default");
      const stub = env.MCP_OBJECT.get(id);
      return stub.fetch(request);
    }
    
    // Other routes handled by Hono
    return app.fetch(request, env, ctx);
  },
};
```

To add auth, wrap this Worker with `OAuthProvider` (see `oauth-provider.md`) or mount BetterAuth (see `betterauth.md`). To add tracing, see `observability.md`.

## Testing

### MCP Inspector

```bash
# Start your server
npm run dev

# In another terminal
npx @modelcontextprotocol/inspector@latest
# Open http://localhost:5173
# Connect to http://localhost:8788/mcp
```

### Claude Desktop Config

Claude Desktop connects to remote servers via the `mcp-remote` proxy:

```json
{
  "mcpServers": {
    "my-mcp": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://my-mcp-server.workers.dev/mcp"
      ]
    }
  }
}
```

This works for OAuth-enabled servers too — `mcp-remote` discovers the server's OAuth metadata, performs dynamic client registration, and opens a browser for the authorization flow automatically. No OAuth settings are needed in the config file.

### Vitest Setup

```typescript
// vitest.config.ts
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: "./wrangler.jsonc" },
      },
    },
  },
});
```

```typescript
// src/__tests__/mcp.test.ts
import { describe, it, expect } from "vitest";
import { env } from "cloudflare:test";

describe("MCP Server", () => {
  it("responds to health check", async () => {
    const response = await env.WORKER.fetch("http://localhost/health");
    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.status).toBe("ok");
  });
});
```

## Deployment

### Secrets

```bash
# Set encryption key
npx wrangler secret put ENCRYPTION_KEY
# Enter a 32+ character random string

# Set API keys if needed
npx wrangler secret put API_KEY
```

### Deploy

```bash
npm run deploy
# or
npx wrangler deploy
```

### Monitor

```bash
# Real-time logs
npx wrangler tail

# With filters
npx wrangler tail --format=json | jq 'select(.logs | length > 0)'
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using a stdio transport for a remote server | Use `@modelcontextprotocol/sdk/server/mcp.js` with Streamable HTTP |
| Missing `waitUntil` for OTel flush | Always `ctx.waitUntil(provider.forceFlush())` |
| BatchSpanProcessor in Workers | Use `SimpleSpanProcessor` — Workers are short-lived |
| Forgetting KV namespace for OAuth | Create and bind `OAUTH_KV` in wrangler.jsonc |
| Missing Durable Object migration | Add `new_sqlite_classes` in migrations |
| Storing plaintext credentials | Always encrypt with AES-GCM (see `oauth-provider.md`) |
| Not handling consent denial | Return proper OAuth error redirect (see `oauth-provider.md`) |
| Missing CORS for browser MCP clients | Add `Access-Control-Allow-Origin` headers |
| Using `sdk-trace-node` in Workers | Use `sdk-trace-base` with `BasicTracerProvider` |
| Instantiating BetterAuth at module scope | `env` only exists per-request — use a `createAuth(env)` factory |

## Reference

- [MCP Specification](https://modelcontextprotocol.io/specification/latest)
- [Cloudflare Remote MCP Guide](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)
- [BetterAuth MCP Plugin](https://www.better-auth.com/docs/plugins/mcp)
- [OpenTelemetry JS SDK](https://opentelemetry.io/docs/languages/js/)
- [Cloudflare Workers OAuth Provider](https://github.com/cloudflare/workers-oauth-provider)
