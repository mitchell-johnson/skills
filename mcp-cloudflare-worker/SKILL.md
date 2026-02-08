---
name: mcp-cloudflare-worker
description: Build MCP (Model Context Protocol) servers on Cloudflare Workers with BetterAuth for OAuth 2.1 authentication and OpenTelemetry for observability. Use when creating remote MCP servers, adding OAuth flows to MCP endpoints, or instrumenting Workers with distributed tracing.
---

# MCP Servers on Cloudflare Workers

## Overview

This skill covers building **production-ready MCP servers** on Cloudflare Workers with:

- **MCP Protocol**: Remote MCP servers using Streamable HTTP transport
- **BetterAuth**: OAuth 2.1 authentication with MCP plugin support
- **OpenTelemetry**: Distributed tracing and observability

**Reference implementations:**
- [akahu-mcp](https://github.com/mitchell-johnson/akahu-mcp) — Multi-user banking MCP with custom OAuth consent flow
- [floorsense](https://github.com/mitchell-johnson/floorsense) — Workplace MCP with D1 credential storage

**Related skill:** See [opentelemetry-js](../opentelemetry-js/SKILL.md) for detailed OTel patterns in Workers.

## When to Use

- Building a remote MCP server (not stdio-based local server)
- Adding OAuth 2.1 authentication to MCP endpoints
- Deploying MCP servers to Cloudflare Workers
- Instrumenting MCP servers with OpenTelemetry
- Multi-tenant MCP servers with per-user credentials

**When NOT to use:**
- Local MCP servers using stdio transport (use `@modelcontextprotocol/server-stdio`)
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

### MCP on Workers

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

### Transport

Remote MCP uses **Streamable HTTP** transport (current MCP spec standard):
- POST for client→server messages
- Optional SSE for server→client streaming
- Endpoint: `/mcp` (configurable)

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

## OAuth 2.1 with @cloudflare/workers-oauth-provider

The official Cloudflare OAuth provider handles the complete OAuth 2.1 flow.

### Basic OAuth Setup

```typescript
// src/index.ts
import { OAuthProvider } from "@cloudflare/workers-oauth-provider";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { McpAgent } from "agents/mcp";
import { Hono } from "hono";
import { z } from "zod";
import { authorize, confirmConsent, tokenExchangeCallback } from "./auth";
import type { UserProps, Env } from "./types";

// Hono for non-API routes
const app = new Hono<{ Bindings: Env }>();

app.get("/authorize", (c) => authorize(c.req.raw, c.env));
app.post("/authorize/consent", (c) => confirmConsent(c.req.raw, c.env));
app.get("/", (c) => c.json({ name: "My MCP Server", version: "1.0.0" }));
app.get("/health", (c) => c.json({ status: "ok" }));

// MCP Agent with user context
export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  server = new McpServer({
    name: "My MCP Server",
    version: "1.0.0",
  });

  async init() {
    // Access authenticated user via this.props
    this.server.tool(
      "whoami",
      "Get current user info",
      {},
      async () => {
        const userId = this.props?.userId;
        return {
          content: [{
            type: "text",
            text: JSON.stringify({
              userId,
              authenticated: !!userId,
            }, null, 2),
          }],
        };
      }
    );
  }
}

// Wrap with OAuthProvider
export default new OAuthProvider({
  apiHandler: MyMCP.serve("/mcp", { transport: "streamable-http" }),
  apiRoute: "/mcp",
  defaultHandler: app as unknown as ExportedHandler,
  authorizeEndpoint: "/authorize",
  tokenEndpoint: "/token",
  clientRegistrationEndpoint: "/register",
  tokenExchangeCallback,
});
```

### Types

```typescript
// src/types.ts
export interface Env {
  MCP_OBJECT: DurableObjectNamespace;
  OAUTH_KV: KVNamespace;
  CREDENTIALS_DB?: D1Database;
  ENCRYPTION_KEY: string;
  // Add your API keys here
  API_KEY?: string;
}

export interface UserProps {
  userId: string;
  // Add custom user properties
  email?: string;
  siteKey?: string;
}
```

### Custom OAuth Consent Flow

For collecting user credentials during OAuth (like API keys):

```typescript
// src/auth.ts
import type { AuthRequest } from "@cloudflare/workers-oauth-provider";
import type { Env, UserProps } from "./types";
import { encrypt } from "./crypto";

// Consent screen HTML
function renderConsentScreen(queryString: string, error?: string): Response {
  const html = `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Authorize Access</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      background: #f5f5f5;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
    }
    .card {
      background: white;
      border-radius: 12px;
      box-shadow: 0 4px 24px rgba(0,0,0,0.1);
      padding: 40px;
      max-width: 400px;
      width: 100%;
    }
    h1 { font-size: 24px; margin-bottom: 16px; }
    .error {
      background: #fef2f2;
      border: 1px solid #fecaca;
      color: #dc2626;
      padding: 12px;
      border-radius: 8px;
      margin-bottom: 16px;
    }
    label { display: block; margin-bottom: 6px; font-weight: 500; }
    input {
      width: 100%;
      padding: 10px 12px;
      border: 1px solid #d1d5db;
      border-radius: 8px;
      margin-bottom: 16px;
      font-size: 14px;
    }
    input:focus { outline: none; border-color: #3b82f6; }
    .actions { display: flex; gap: 12px; }
    button {
      flex: 1;
      padding: 12px;
      border-radius: 8px;
      font-size: 14px;
      font-weight: 500;
      cursor: pointer;
      border: none;
    }
    .btn-primary { background: #3b82f6; color: white; }
    .btn-secondary { background: #e5e7eb; color: #374151; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Authorize Access</h1>
    ${error ? `<div class="error">${escapeHtml(error)}</div>` : ""}
    <form method="POST" action="/authorize/consent?${escapeHtml(queryString)}">
      <label for="apiKey">API Key</label>
      <input type="password" id="apiKey" name="apiKey" required 
             placeholder="Enter your API key" />
      <div class="actions">
        <button type="button" class="btn-secondary" onclick="deny()">Deny</button>
        <button type="submit" class="btn-primary">Authorize</button>
      </div>
    </form>
  </div>
  <script>
    function deny() {
      const form = document.querySelector('form');
      const input = document.createElement('input');
      input.type = 'hidden';
      input.name = 'action';
      input.value = 'deny';
      form.appendChild(input);
      form.submit();
    }
  </script>
</body>
</html>`;

  return new Response(html, {
    headers: { "Content-Type": "text/html;charset=UTF-8" },
  });
}

function escapeHtml(str: string): string {
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
}

// GET /authorize - Show consent screen
export async function authorize(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  
  const oauthReq = await env.OAUTH_PROVIDER.parseAuthRequest(request);
  if (!oauthReq.clientId) {
    return new Response("Invalid OAuth request: missing client_id", { status: 400 });
  }

  // Store OAuth request in KV for consent handler
  const stateKey = `oauth_state:${oauthReq.clientId}:${crypto.randomUUID()}`;
  await env.OAUTH_KV.put(stateKey, JSON.stringify(oauthReq), {
    expirationTtl: 3600,
  });

  const consentQuery = new URLSearchParams(url.search);
  consentQuery.set("_state_key", stateKey);

  return renderConsentScreen(consentQuery.toString());
}

// POST /authorize/consent - Process consent
export async function confirmConsent(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);
  const form = await request.formData();

  // Handle denial
  if (form.get("action") === "deny") {
    const stateKey = url.searchParams.get("_state_key");
    if (stateKey) {
      const stored = await env.OAUTH_KV.get<AuthRequest>(stateKey, "json");
      await env.OAUTH_KV.delete(stateKey);
      if (stored?.redirectUri) {
        const redirectUrl = new URL(stored.redirectUri);
        redirectUrl.searchParams.set("error", "access_denied");
        if (stored.state) redirectUrl.searchParams.set("state", stored.state);
        return Response.redirect(redirectUrl.toString(), 302);
      }
    }
    return new Response("Authorization denied", { status: 403 });
  }

  // Retrieve stored OAuth request
  const stateKey = url.searchParams.get("_state_key");
  if (!stateKey) {
    return new Response("Missing state parameter", { status: 400 });
  }

  const oauthReq = await env.OAUTH_KV.get<AuthRequest>(stateKey, "json");
  if (!oauthReq) {
    return new Response("Expired or invalid session", { status: 400 });
  }

  // Get credentials from form
  const apiKey = String(form.get("apiKey") || "").trim();
  if (!apiKey) {
    return renderConsentScreen(url.search.slice(1), "API key is required.");
  }

  // Validate API key against external service
  try {
    const valid = await validateApiKey(apiKey);
    if (!valid) {
      return renderConsentScreen(url.search.slice(1), "Invalid API key.");
    }
  } catch {
    return renderConsentScreen(url.search.slice(1), "Failed to validate API key.");
  }

  // Generate user ID and store encrypted credentials
  const userId = crypto.randomUUID();
  const encryptedKey = await encrypt(apiKey, env.ENCRYPTION_KEY);
  
  await env.OAUTH_KV.put(
    `credentials:${userId}`,
    JSON.stringify({ apiKey: encryptedKey, createdAt: new Date().toISOString() }),
    { expirationTtl: 60 * 60 * 24 * 30 } // 30 days
  );

  // Complete OAuth authorization
  const { redirectTo } = await env.OAUTH_PROVIDER.completeAuthorization({
    request: oauthReq,
    userId,
    metadata: { source: "consent" },
    scope: oauthReq.scope || [],
    props: { userId } satisfies UserProps,
  });

  await env.OAUTH_KV.delete(stateKey);
  return Response.redirect(redirectTo, 302);
}

// Token exchange callback
export async function tokenExchangeCallback(options: {
  grantType: "authorization_code" | "refresh_token";
  props: UserProps;
  clientId: string;
  userId: string;
  scope: string[];
}) {
  return {
    accessTokenProps: { ...options.props },
    newProps: { ...options.props },
  };
}

async function validateApiKey(apiKey: string): Promise<boolean> {
  // Implement your API key validation logic
  return apiKey.startsWith("sk_");
}
```

### AES-GCM Encryption

```typescript
// src/crypto.ts

/**
 * Derive AES-256 key from string secret.
 */
async function deriveKey(secret: string): Promise<CryptoKey> {
  const encoder = new TextEncoder();
  let raw = encoder.encode(secret);
  
  // Pad or trim to 32 bytes
  if (raw.length > 32) {
    raw = raw.slice(0, 32);
  } else if (raw.length < 32) {
    const padded = new Uint8Array(32);
    padded.set(raw);
    raw = padded;
  }

  return crypto.subtle.importKey(
    "raw",
    raw,
    { name: "AES-GCM" },
    false,
    ["encrypt", "decrypt"]
  );
}

/**
 * Encrypt plaintext to base64 blob (IV + ciphertext).
 */
export async function encrypt(plaintext: string, encryptionKey: string): Promise<string> {
  const key = await deriveKey(encryptionKey);
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encoded = new TextEncoder().encode(plaintext);

  const ciphertext = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv },
    key,
    encoded
  );

  const combined = new Uint8Array(iv.length + ciphertext.byteLength);
  combined.set(iv, 0);
  combined.set(new Uint8Array(ciphertext), iv.length);

  return btoa(String.fromCharCode(...combined));
}

/**
 * Decrypt base64 blob back to plaintext.
 */
export async function decrypt(encrypted: string, encryptionKey: string): Promise<string> {
  const key = await deriveKey(encryptionKey);
  const combined = Uint8Array.from(atob(encrypted), (c) => c.charCodeAt(0));

  const iv = combined.slice(0, 12);
  const ciphertext = combined.slice(12);

  const decrypted = await crypto.subtle.decrypt(
    { name: "AES-GCM", iv },
    key,
    ciphertext
  );

  return new TextDecoder().decode(decrypted);
}
```

## BetterAuth Integration

BetterAuth provides a dedicated MCP plugin for OAuth 2.1 provider functionality.

### Install

```bash
npm install better-auth @better-auth/oauth-provider
```

### Server Setup

```typescript
// src/auth.ts
import { betterAuth } from "better-auth";
import { mcp } from "better-auth/plugins";

export const auth = betterAuth({
  // Database adapter (see BetterAuth docs for options)
  database: env.DB, // D1, Postgres, etc.
  
  plugins: [
    mcp({
      loginPage: "/sign-in", // Path to your login page
    }),
  ],
  
  // Required for cross-origin MCP clients
  advanced: {
    defaultCookieAttributes: {
      sameSite: "none",
      secure: true,
      partitioned: true,
    },
  },
});
```

### MCP Handler with BetterAuth

```typescript
// src/index.ts
import { Hono } from "hono";
import { auth } from "./auth";
import { withMcpAuth } from "better-auth/plugins";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const app = new Hono<{ Bindings: Env }>();

// Mount BetterAuth handler
app.on(["POST", "GET"], "/api/auth/*", (c) => auth.handler(c.req.raw));

// MCP endpoint with auth
app.all("/mcp/*", async (c) => {
  return withMcpAuth(auth, (req, session) => {
    // session contains access token with scopes and user ID
    const server = new McpServer({
      name: "My MCP Server",
      version: "1.0.0",
    });

    server.tool(
      "whoami",
      "Get current user",
      {},
      async () => ({
        content: [{
          type: "text",
          text: JSON.stringify({
            userId: session.userId,
            scopes: session.scopes,
          }, null, 2),
        }],
      })
    );

    // Return handler
    return server.handler(req);
  })(c.req.raw);
});

export default app;
```

### Well-Known Endpoints

BetterAuth MCP requires these discovery endpoints:

```typescript
// app/.well-known/oauth-authorization-server/route.ts
import { oAuthDiscoveryMetadata } from "better-auth/plugins";
import { auth } from "@/lib/auth";

export const GET = oAuthDiscoveryMetadata(auth);

// app/.well-known/oauth-protected-resource/route.ts  
import { oAuthProtectedResourceMetadata } from "better-auth/plugins";
import { auth } from "@/lib/auth";

export const GET = oAuthProtectedResourceMetadata(auth);
```

## OpenTelemetry Integration

See [opentelemetry-js](../opentelemetry-js/SKILL.md) for complete OTel patterns. Key points for MCP Workers:

### Install

```bash
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-base \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/exporter-trace-otlp-http
```

### Trace Provider

```typescript
// src/tracing.ts
import { trace, type Tracer } from "@opentelemetry/api";
import { BasicTracerProvider, SimpleSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { resourceFromAttributes } from "@opentelemetry/resources";
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from "@opentelemetry/semantic-conventions";

interface Env {
  OTEL_EXPORTER_OTLP_ENDPOINT?: string;
  OTEL_API_KEY?: string;
}

export function createTracerProvider(env: Env): {
  provider: BasicTracerProvider;
  tracer: Tracer;
} {
  const resource = resourceFromAttributes({
    [ATTR_SERVICE_NAME]: "mcp-server",
    [ATTR_SERVICE_VERSION]: "1.0.0",
    "cloud.provider": "cloudflare",
    "cloud.platform": "cloudflare_workers",
  });

  const exporter = new OTLPTraceExporter({
    url: env.OTEL_EXPORTER_OTLP_ENDPOINT || "https://your-collector:4318/v1/traces",
    headers: {
      Authorization: `Bearer ${env.OTEL_API_KEY || ""}`,
    },
  });

  const provider = new BasicTracerProvider({
    resource,
    spanProcessors: [new SimpleSpanProcessor(exporter)],
  });

  const tracer = provider.getTracer("mcp-server", "1.0.0");
  return { provider, tracer };
}
```

### Traced MCP Tools

```typescript
// src/index.ts
import { trace, context, SpanStatusCode } from "@opentelemetry/api";
import { createTracerProvider } from "./tracing";

export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  server = new McpServer({ name: "My MCP Server", version: "1.0.0" });
  
  private tracer = trace.getTracer("mcp-tools");

  async init() {
    this.server.tool(
      "fetch_data",
      "Fetch data from external API",
      { id: z.string() },
      async ({ id }) => {
        return this.tracer.startActiveSpan("tool.fetch_data", async (span) => {
          span.setAttribute("data.id", id);
          span.setAttribute("user.id", this.props?.userId || "unknown");
          
          try {
            const data = await this.fetchFromApi(id);
            span.setStatus({ code: SpanStatusCode.OK });
            return {
              content: [{ type: "text", text: JSON.stringify(data, null, 2) }],
            };
          } catch (error) {
            span.recordException(error as Error);
            span.setStatus({ 
              code: SpanStatusCode.ERROR, 
              message: (error as Error).message 
            });
            throw error;
          } finally {
            span.end();
          }
        });
      }
    );
  }
}

// In the Worker fetch handler
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const { provider, tracer } = createTracerProvider(env);
    
    const span = tracer.startSpan("worker.request", {
      attributes: {
        "http.method": request.method,
        "http.url": request.url,
      },
    });

    try {
      const response = await handleRequest(request, env);
      span.setAttribute("http.status_code", response.status);
      return response;
    } catch (error) {
      span.recordException(error as Error);
      span.setStatus({ code: SpanStatusCode.ERROR });
      throw error;
    } finally {
      span.end();
      // CRITICAL: Flush telemetry without blocking response
      ctx.waitUntil(provider.forceFlush());
    }
  },
};
```

### Cloudflare Native Observability

Alternative to custom OTel: use Cloudflare's built-in observability:

```jsonc
// wrangler.jsonc
{
  "observability": {
    "enabled": true,
    "head_sampling_rate": 1  // Sample 100% (adjust for high traffic)
  }
}
```

This provides automatic logging and tracing in the Cloudflare dashboard without SDK code.

## D1 Credential Storage

For persistent, encrypted credential storage:

### Migration

```sql
-- migrations/0001_create_users.sql
CREATE TABLE IF NOT EXISTS users (
  oauth_user_id TEXT PRIMARY KEY,
  encrypted_api_key TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE INDEX idx_users_created_at ON users(created_at);
```

Run migration:
```bash
npx wrangler d1 migrations apply mcp-credentials
```

### Credential Store

```typescript
// src/credential-store.ts
import { encrypt, decrypt } from "./crypto";

export interface StoredCredentials {
  apiKey: string;
}

export async function storeCredentials(
  db: D1Database,
  encryptionKey: string,
  userId: string,
  apiKey: string
): Promise<void> {
  const encryptedKey = await encrypt(apiKey, encryptionKey);
  const now = new Date().toISOString();

  await db
    .prepare(`
      INSERT INTO users (oauth_user_id, encrypted_api_key, created_at, updated_at)
      VALUES (?1, ?2, ?3, ?4)
      ON CONFLICT (oauth_user_id) DO UPDATE SET
        encrypted_api_key = ?2,
        updated_at = ?4
    `)
    .bind(userId, encryptedKey, now, now)
    .run();
}

export async function getCredentials(
  db: D1Database,
  encryptionKey: string,
  userId: string
): Promise<StoredCredentials | null> {
  const row = await db
    .prepare("SELECT encrypted_api_key FROM users WHERE oauth_user_id = ?1")
    .bind(userId)
    .first<{ encrypted_api_key: string }>();

  if (!row) return null;

  const apiKey = await decrypt(row.encrypted_api_key, encryptionKey);
  return { apiKey };
}

export async function deleteCredentials(
  db: D1Database,
  userId: string
): Promise<void> {
  await db
    .prepare("DELETE FROM users WHERE oauth_user_id = ?1")
    .bind(userId)
    .run();
}
```

## Durable Object Patterns

### Caching with Daily Expiry

```typescript
export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  /** Today's date stamp for cache invalidation */
  private todayStamp(): string {
    return new Date().toISOString().slice(0, 10);
  }

  /** Get from cache if same day */
  private async cacheGet<T>(key: string): Promise<T | undefined> {
    const entry = await this.ctx.storage.get<{ date: string; data: T }>(`cache:${key}`);
    if (entry && entry.date === this.todayStamp()) {
      return entry.data;
    }
    return undefined;
  }

  /** Set cache with today's date */
  private async cacheSet<T>(key: string, data: T): Promise<void> {
    await this.ctx.storage.put(`cache:${key}`, {
      date: this.todayStamp(),
      data,
    });
  }

  /** Get cached or fetch fresh */
  private async cached<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
    const hit = await this.cacheGet<T>(key);
    if (hit !== undefined) return hit;
    const fresh = await fetcher();
    await this.cacheSet(key, fresh);
    return fresh;
  }

  async init() {
    this.server.tool(
      "get_data",
      "Get data (cached daily)",
      {},
      async () => {
        const data = await this.cached("all_data", () => this.fetchAllData());
        return { content: [{ type: "text", text: JSON.stringify(data) }] };
      }
    );
  }
}
```

### WebSocket Keepalive with Alarms

```typescript
const KEEPALIVE_INTERVAL_MS = 30_000;
const IDLE_TIMEOUT_MS = 10 * 60_000;

export class MyMCP extends McpAgent<Env, Record<string, never>, UserProps> {
  private client: ExternalClient | null = null;
  private lastActivity = Date.now();

  private scheduleNextAlarm(): void {
    try {
      this.ctx.storage.setAlarm(Date.now() + KEEPALIVE_INTERVAL_MS);
    } catch {
      // Alarm API may not be available in tests
    }
  }

  override readonly alarm = async (): Promise<void> => {
    if (!this.client) return;

    const idleTime = Date.now() - this.lastActivity;
    if (idleTime > IDLE_TIMEOUT_MS) {
      // Idle too long - disconnect to allow hibernation
      this.client.disconnect();
      this.client = null;
      return;
    }

    // Send keepalive ping
    this.client.ping();
    this.scheduleNextAlarm();
  };

  private async ensureClient(): Promise<ExternalClient> {
    this.lastActivity = Date.now();
    
    if (this.client) return this.client;

    this.client = new ExternalClient();
    await this.client.connect();
    this.scheduleNextAlarm();

    return this.client;
  }
}
```

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

For OAuth-enabled servers:
```json
{
  "mcpServers": {
    "my-mcp": {
      "transport": {
        "type": "http",
        "url": "https://my-mcp-server.workers.dev/mcp"
      },
      "oauth": {
        "client_id": "your-client-id",
        "authorization_endpoint": "https://my-mcp-server.workers.dev/authorize",
        "token_endpoint": "https://my-mcp-server.workers.dev/token"
      }
    }
  }
}
```

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
| Using `@modelcontextprotocol/server-stdio` | Use `@modelcontextprotocol/sdk/server/mcp.js` for remote servers |
| Missing `waitUntil` for OTel flush | Always `ctx.waitUntil(provider.forceFlush())` |
| BatchSpanProcessor in Workers | Use `SimpleSpanProcessor` — Workers are short-lived |
| Forgetting KV namespace for OAuth | Create and bind `OAUTH_KV` in wrangler.jsonc |
| Missing Durable Object migration | Add `new_sqlite_classes` in migrations |
| Storing plaintext credentials | Always encrypt with AES-GCM |
| Not handling consent denial | Return proper OAuth error redirect |
| Missing CORS for browser MCP clients | Add `Access-Control-Allow-Origin` headers |
| Using `sdk-trace-node` in Workers | Use `sdk-trace-base` with `BasicTracerProvider` |

## Reference

- [MCP Specification](https://modelcontextprotocol.io/specification/latest)
- [Cloudflare Remote MCP Guide](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)
- [BetterAuth MCP Plugin](https://www.better-auth.com/docs/plugins/mcp)
- [OpenTelemetry JS SDK](https://opentelemetry.io/docs/languages/js/)
- [Cloudflare Workers OAuth Provider](https://github.com/cloudflare/workers-oauth-provider)
