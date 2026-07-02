# BetterAuth Integration

BetterAuth provides a dedicated MCP plugin for OAuth 2.1 provider functionality. Use this instead of `@cloudflare/workers-oauth-provider` when you want full user accounts (sign-up, sign-in, sessions) rather than a lightweight consent-only flow.

## Install

```bash
npm install better-auth @better-auth/oauth-provider
```

## Server Setup

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

> **Workers note:** on Cloudflare Workers, `env` is only available inside request handlers — wrap `betterAuth()` in a per-request factory (`createAuth(env)`) rather than instantiating at module scope. BetterAuth also requires the `nodejs_compat` compatibility flag.

## MCP Handler with BetterAuth

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

## Well-Known Endpoints

BetterAuth MCP requires these discovery endpoints (Next.js route-handler example — on Workers, mount equivalent GET routes in Hono):

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
