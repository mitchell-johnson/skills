---
name: cloudflare-workers-app
description: Use when building a new web application with Cloudflare Workers, D1 database, Better Auth authentication, Hono API, or React SPA frontend with Tailwind and Zustand - covers project scaffolding, architecture patterns, and deployment
---

# Cloudflare Workers Full-Stack App

## Overview

A production-ready stack for building full-stack web apps deployed as a single Cloudflare Worker: **Hono API + Better Auth + D1 database** on the backend, **React SPA + Tailwind + Zustand + React Query** on the frontend. One Worker serves both API routes and static assets.

## When to Use

- Scaffolding a new web app with auth, database, and a React frontend
- Building on Cloudflare Workers with D1 (SQLite)
- Need email/password auth with session cookies (Better Auth)
- Want a monorepo with API and web packages

**When NOT to use:**
- Static sites without auth or database
- Apps requiring a traditional SQL database (Postgres, MySQL)
- Projects that need SSR (use Next.js/Remix instead)
- Non-web backends (CLI tools, libraries)

## Quick Start

1. **Scaffold the project** - Follow `scaffold.md` for the full monorepo setup with every file template
2. **Understand the architecture** - See `architecture-reference.md` for middleware chain, route patterns, frontend conventions, and deployment model
3. **Avoid known pitfalls** - Read `gotchas.md` before your first deploy (especially the Better Auth D1 camelCase issue)

## Key Decisions This Stack Makes

| Concern | Choice |
|---------|--------|
| Runtime | Cloudflare Workers |
| Database | D1 (SQLite at edge) |
| Auth | Better Auth with email/password |
| API framework | Hono |
| Validation | Zod |
| Frontend | React 18 SPA |
| Styling | Tailwind CSS |
| UI state | Zustand |
| Server state | React Query |
| Build | Vite (web) + Wrangler (API) |
| Monorepo | npm workspaces |
