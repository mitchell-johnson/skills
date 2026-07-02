# OAuth 2.1 with @cloudflare/workers-oauth-provider

Full setup for the official Cloudflare OAuth provider, including a custom consent flow that collects user credentials (e.g. API keys) during authorization, AES-GCM encryption, and D1 credential storage.

## Basic OAuth Setup

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

## Types

`OAUTH_PROVIDER` is injected into `env` by the `OAuthProvider` wrapper — include it in your `Env` so the auth handlers type-check:

```typescript
// src/types.ts
import type { OAuthHelpers } from "@cloudflare/workers-oauth-provider";

export interface Env {
  MCP_OBJECT: DurableObjectNamespace;
  OAUTH_KV: KVNamespace;
  OAUTH_PROVIDER: OAuthHelpers; // injected by OAuthProvider wrapper
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

## Custom OAuth Consent Flow

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

## AES-GCM Encryption

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

## D1 Credential Storage

For persistent, encrypted credential storage (instead of the KV-with-TTL approach above):

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
