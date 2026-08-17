---
name: wrangler
description: Use when running, writing, or reviewing any wrangler CLI command, or when editing wrangler.jsonc/wrangler.toml — deploying Workers, local dev, and managing Cloudflare resources (KV, R2, D1, Queues, Workflows, Containers, Pipelines, Secrets Store, and more) from the CLI. Load before running wrangler commands to ensure correct syntax. Biases towards retrieval from Cloudflare docs over pre-trained knowledge.
---

# Wrangler CLI

Verified against wrangler 4.123.0, 2026-08-15.

Your knowledge of Wrangler CLI flags, config fields, and subcommands may be outdated. **Prefer retrieval over pre-training** for any Wrangler task.

## Retrieval Sources

| Source | How to retrieve | Use for |
|--------|----------------|---------|
| `wrangler <cmd> --help` | Run locally | Ground truth for flags and subcommands |
| Wrangler config schema | `node_modules/wrangler/config-schema.json` | Config fields, binding shapes, allowed values |
| Wrangler docs | `https://developers.cloudflare.com/workers/wrangler/` | CLI commands, flags, config reference |
| Cloudflare docs | Search tool or `https://developers.cloudflare.com/workers/` | API reference, compatibility dates/flags |

This skill ships two reference files — read the relevant one before writing commands or config:

- [references/commands.md](references/commands.md) — per-product command syntax
- [references/config.md](references/config.md) — wrangler.jsonc binding snippets

## FIRST: Check if Wrangler is installed, and if not, install it

```bash
wrangler --version  # Requires v4.x+

# If not installed:
npm install -D wrangler@latest
```

Wherever possible, use Wrangler instead of manually constructing API requests.

## Key Guidelines

### Configuration

- **Use `wrangler.jsonc`**: Prefer JSON config over TOML. Newer features are JSON-only. Set `"$schema": "./node_modules/wrangler/config-schema.json"` and version-control the file as the source of truth.
- **Set a current `compatibility_date`** (e.g. `2026-08-04`). With dates `2026-08-04` and later, Node.js compatibility is enabled by default — no flag needed. For dates `2024-09-23` through `2026-08-03`, add `"compatibility_flags": ["nodejs_compat"]`.
- **Static assets**: the `assets` config field is THE modern way to serve static files from a Worker (Workers Sites is deprecated). See [references/config.md](references/config.md#static-assets).
- **Use environments** (`env.staging`, `env.production`) to separate staging/prod. Caution: bindings and `vars` are NOT inherited by named environments — every binding block must be repeated in each `env.<name>` that needs it. See [references/config.md](references/config.md#environments).
- **Automatic provisioning** (omitting resource IDs so they are created on deploy) is in Beta.

### Dev & Deploy Workflow

- `wrangler dev` runs a local simulation by default (`-r, --remote` runs on the edge and is legacy).
- Set `remote: true` on individual bindings to hit real resources from local dev. Recommended remote bindings: AI (required), Vectorize, Browser Run, mTLS, Images, Dispatch Namespaces.
- `wrangler deploy` — use `--dry-run` to validate first, `--env <name>` for environments.
- Gradual rollouts: `wrangler versions upload`, then `wrangler versions deploy` to split traffic. Revert with `wrangler rollback`.
- `wrangler check startup` profiles Worker startup time and detects scripts exceeding the startup limit.
- Test with `@cloudflare/vitest-pool-workers` (snippet in [references/commands.md](references/commands.md)).

### Secrets

- `wrangler secret put NAME` — interactive prompt (preferred) or pipe from a file; `wrangler secret bulk secrets.json` for many at once (do not commit that file).
- Use `.dev.vars` for local dev secrets.
- Never pass secret values as command arguments, pipe them via `echo`, or output/log/hardcode them.

### Types

- Run `wrangler types` after every config change to regenerate TypeScript bindings.
- Run `wrangler types --check` in CI to catch binding mismatches.

## Command Group Index

| Group | What it's for | Maturity | Details |
|-------|---------------|----------|---------|
| `init`, `setup` | Scaffold / guided project setup | GA | `wrangler setup --help` |
| `dev`, `deploy`, `delete` | Local dev server, deploy, remove Workers | GA | [commands.md](references/commands.md) |
| `versions`, `deployments`, `rollback`, `triggers` | Gradual deployments, history, rollback, deploy triggers | GA | [commands.md](references/commands.md) |
| `secret` | Per-Worker secrets | GA | [commands.md](references/commands.md) |
| `types`, `check` | Type generation, startup profiling | GA | [commands.md](references/commands.md) |
| `tail` | Live logs | GA | [commands.md](references/commands.md) |
| `kv` | Key-value storage | GA | [commands.md](references/commands.md) |
| `r2` | Object storage | GA | [commands.md](references/commands.md) |
| `d1` | Serverless SQL database | GA | [commands.md](references/commands.md) |
| `queues` | Message queues | GA | [commands.md](references/commands.md) |
| `workflows` | Durable execution | GA | [commands.md](references/commands.md) |
| `vectorize` | Vector database | GA | [commands.md](references/commands.md) |
| `hyperdrive` | Database connection accelerator | GA | [commands.md](references/commands.md) |
| `ai` | Workers AI models and finetunes | GA | [commands.md](references/commands.md) |
| `containers` | Container images and instances | GA | [commands.md](references/commands.md) |
| `pages` | Pages projects and deployments | GA | [commands.md](references/commands.md) |
| `pipelines` | Streaming ingestion: streams → SQL pipeline → sinks | Open beta | [commands.md](references/commands.md) |
| `secrets-store` | Account-level shared secrets | Open beta | [commands.md](references/commands.md) |
| `dispatch-namespace` | Workers for Platforms namespaces | GA | `wrangler dispatch-namespace --help` |
| `mtls-certificate` | Client certificates for mTLS bindings | GA | `wrangler mtls-certificate --help` |
| `cert` | CA chain and client cert management | Open beta | `wrangler cert --help` |
| `email` | Email Service (sending and routing) | Open beta | `wrangler email --help` |
| `ai-search` | AI Search | Open beta | `wrangler ai-search --help` |
| `browser` | Browser Run (browser automation) | Open beta | `wrangler browser --help` |
| `flagship` | Feature flags | Open beta | `wrangler flagship --help` |
| `vpc` | Private network connectivity | Open beta | `wrangler vpc --help` |
| `tunnel` | Cloudflare Tunnel | Experimental | `wrangler tunnel --help` |
| `turnstile` | Turnstile widgets | Alpha | `wrangler turnstile --help` |
| `artifacts` | Artifact storage | Private beta | `wrangler artifacts --help` |
| `agent-memory` | Agent memory | Private beta | `wrangler agent-memory --help` |
| `login`, `logout`, `whoami`, `auth` | Authentication; `auth` manages multiple account profiles (`--profile`) | GA | `wrangler auth --help` |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `command not found: wrangler` | `npm install -D wrangler@latest`, run via `npx wrangler` |
| Auth errors | Run `wrangler login`; check with `wrangler whoami` |
| Startup time limit exceeded | Run `wrangler check startup` to profile startup and generate CPU profiles |
| Type errors after config change | Run `wrangler types` |
| Local storage not persisting | Check `.wrangler/state` directory |
| Binding undefined in Worker | Verify binding name matches config exactly; re-run `wrangler types` |
| Unsure of a flag or subcommand | `wrangler <cmd> --help` is ground truth; also `wrangler docs configuration` |
