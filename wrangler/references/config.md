# Wrangler Configuration Reference

Verified against wrangler 4.123.0, 2026-08-15.

`node_modules/wrangler/config-schema.json` is the **authoritative** source for config fields, binding shapes, and allowed values — read it before guessing a key name. Setting `"$schema": "./node_modules/wrangler/config-schema.json"` at the top of `wrangler.jsonc` also gives editor validation and autocomplete. If a field is not covered here, check the schema or run `wrangler docs configuration`.

## JSONC vs TOML

Cloudflare recommends `wrangler.jsonc` for new projects; some newer features are JSON-only. `wrangler.toml` still works for existing Workers, and keys are identical across formats. Only one config file per project — do not keep both.

## Minimal Config

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-worker",
  "main": "src/index.ts",
  "compatibility_date": "2026-08-04"
}
```

## Core Top-Level Fields

| Field | Purpose |
|-------|---------|
| `name` | Worker name (also the `workers.dev` subdomain) |
| `main` | Entry point, e.g. `src/index.ts` |
| `compatibility_date` | Pins runtime behavior — see below |
| `compatibility_flags` | Opt into/out of individual runtime behaviors |
| `account_id` | Optional; also settable via `CLOUDFLARE_ACCOUNT_ID` |
| `vars` | Plaintext environment variables (never secrets) |
| `assets` | Static asset directory + optional binding |
| `observability` | Worker Logs; enable this on every Worker |
| `limits` | `cpu_ms`, `subrequests` |
| `placement` | `{ "mode": "smart" }` (also `off`, `targeted`) |
| `workers_dev` / `preview_urls` | Booleans controlling `*.workers.dev` and per-version preview URLs |
| `routes` / `route` | Zone routes and custom domains |
| `triggers` | `{ "crons": [...] }` |
| `env.<name>` | Named environments |
| `secrets.required` | List of secret names the Worker needs — drives type generation and local-dev warnings |

```jsonc
{
  "observability": { "enabled": true, "head_sampling_rate": 1 },
  "limits": { "cpu_ms": 30000 },
  "placement": { "mode": "smart" },
  "vars": { "ENVIRONMENT": "production" },
  "triggers": { "crons": ["0 * * * *"] },
  "workers_dev": false,
  "routes": [
    { "pattern": "example.com/api/*", "zone_name": "example.com" },
    { "pattern": "api.example.com", "custom_domain": true }   // custom domain
  ]
}
```

Run `wrangler triggers deploy` to apply route/cron changes without a full redeploy.

## Compatibility Date and Node.js Compat

Use a current `compatibility_date` (`2026-08-04` or later) for new Workers.

| Compatibility date | Node.js compatibility |
|--------------------|-----------------------|
| `2026-08-04` and later | Enabled **by default** — no flag needed |
| `2024-09-23` – `2026-08-03` | Requires `"compatibility_flags": ["nodejs_compat"]` |
| Before `2024-09-23` | Older/partial flags; raise the date instead |

```jsonc
{
  "compatibility_date": "2026-08-04"
  // no compatibility_flags needed for Node.js APIs
}
```

## Static Assets

The `assets` field is **the** modern way to serve static files from a Worker. Workers Sites (`site`) is deprecated — do not use it for new projects.

```jsonc
{
  "assets": {
    "directory": "./public",
    "binding": "ASSETS"
  }
}
```

- `directory` alone serves the files. Add `binding` only if the Worker needs to fetch assets itself (`env.ASSETS.fetch(request)`).
- SPA and custom 404 behavior is configured with `not_found_handling` (`single-page-application`, `404-page`, `none`), and URL trailing-slash behavior with `html_handling`. Routing precedence between the Worker and assets is controlled by `run_worker_first`. Confirm exact semantics in the schema or `wrangler docs` before relying on them.
- A Worker with `assets` and no `main` is a pure static site.

## Environments

```jsonc
{
  "name": "my-worker",
  "vars": { "ENVIRONMENT": "production" },
  "env": {
    "staging": {
      "name": "my-worker-staging",
      "vars": { "ENVIRONMENT": "staging" },
      "kv_namespaces": [{ "binding": "CACHE", "id": "<STAGING_KV_ID>" }]
    }
  }
}
```

**Bindings and `vars` are NOT inherited by named environments.** Every binding block (`kv_namespaces`, `r2_buckets`, `d1_databases`, `queues`, `durable_objects`, `services`, `containers`, `pipelines`, `ai`, `browser`, `images`, `ratelimits`, …) must be repeated in each `env.<name>` that needs it. `containers` in particular is easy to miss — a named environment silently deploys without it. Fields like `name`, `main`, `compatibility_date`, and `routes` are inherited.

Deploy with `wrangler deploy --env staging`.

## Remote Bindings

`remote: true` makes `wrangler dev` hit the real remote resource instead of the local simulation.

```jsonc
{
  "ai": { "binding": "AI" },
  "vectorize": [{ "binding": "INDEX", "index_name": "my-index", "remote": true }],
  "r2_buckets": [{ "binding": "BUCKET", "bucket_name": "my-bucket", "remote": true }]
}
```

Workers AI **always** runs remotely (and bills) even in local dev. Other good candidates for `remote: true`: Vectorize, Browser Run, mTLS certificates, Images, and Dispatch Namespaces — resources with no meaningful local simulation. Keep KV/R2/D1 local unless you specifically need production data.

## Automatic Provisioning (Beta)

Omitting a resource's ID lets Wrangler create it at deploy time and wire up the binding:

```jsonc
{ "kv_namespaces": [{ "binding": "CACHE" }], "r2_buckets": [{ "binding": "BUCKET" }] }
```

This is in **Beta**. For reproducible deploys, prefer creating resources explicitly (see [commands.md](commands.md)) and committing the returned IDs.

## Binding Snippets

### KV

```jsonc
{ "kv_namespaces": [{ "binding": "CACHE", "id": "<KV_NAMESPACE_ID>" }] }
```

### R2

```jsonc
{ "r2_buckets": [{ "binding": "BUCKET", "bucket_name": "my-bucket" }] }
```

### D1

```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-database",
      "database_id": "<DATABASE_ID>",
      "migrations_dir": "./migrations"
    }
  ]
}
```

### Queues (producer + consumer)

```jsonc
{
  "queues": {
    "producers": [{ "binding": "MY_QUEUE", "queue": "my-queue" }],
    "consumers": [
      {
        "queue": "my-queue",
        "max_batch_size": 10,
        "max_batch_timeout": 30,
        "max_retries": 3,
        "dead_letter_queue": "my-queue-dlq"
      }
    ]
  }
}
```

### Durable Objects

Current guidance: declare classes with the declarative `exports` map, and bind them with `durable_objects.bindings`.

```jsonc
{
  "durable_objects": {
    "bindings": [{ "name": "COUNTER", "class_name": "Counter" }]
  },
  "exports": {
    "Counter": { "type": "durable-object", "storage": "sqlite" }
  }
}
```

- `storage` is `"sqlite"` (use this) or `"legacy-kv"`.
- Legacy `migrations` arrays (`new_sqlite_classes`, `renamed_classes`, `deleted_classes`) still work for existing Workers, but `exports` and `migrations` are **mutually exclusive** and switching to `exports` is **one-way**.
- Binding a class defined in another Worker: add `"script_name": "other-worker"` to the binding.

### Workflows

```jsonc
{
  "workflows": [
    { "binding": "MY_WORKFLOW", "name": "my-workflow", "class_name": "MyWorkflow" }
  ]
}
```

### Service Bindings (Worker-to-Worker)

```jsonc
{
  "services": [
    { "binding": "AUTH", "service": "auth-worker", "entrypoint": "AuthEntrypoint" }
  ]
}
```

`entrypoint` is optional — omit it to call the target's default export.

### Vectorize, Hyperdrive, AI, Browser Run, mTLS, Analytics Engine, Rate Limiting, Images, Dispatch Namespaces

```jsonc
{
  "ai": { "binding": "AI" },                                   // always remote
  "browser": { "binding": "BROWSER" },                         // Browser Run
  "images": { "binding": "IMAGES" },
  "vectorize": [{ "binding": "SEARCH_INDEX", "index_name": "my-index" }],
  "hyperdrive": [{ "binding": "HYPERDRIVE", "id": "<HYPERDRIVE_ID>" }],
  "mtls_certificates": [{ "binding": "MTLS_CERT", "certificate_id": "<CERT_ID>" }],
  "analytics_engine_datasets": [{ "binding": "ANALYTICS", "dataset": "my_dataset" }],
  "dispatch_namespaces": [{ "binding": "DISPATCH", "namespace": "my-namespace" }],
  "ratelimits": [
    // period must be 10 or 60 (seconds); namespace_id is any unique string per limiter
    { "name": "RATE_LIMITER", "namespace_id": "1001", "simple": { "limit": 100, "period": 60 } }
  ]
}
```

### Containers

A container app is paired with a Durable Object class that manages its lifecycle.

```jsonc
{
  "durable_objects": {
    "bindings": [{ "name": "MY_CONTAINER", "class_name": "MyContainer" }]
  },
  "exports": { "MyContainer": { "type": "durable-object", "storage": "sqlite" } },
  "containers": [
    {
      "class_name": "MyContainer",
      "image": "./Dockerfile",
      "max_instances": 5,
      "instance_type": "dev"
    }
  ]
}
```

`image` is a Dockerfile path or a Cloudflare registry image URI. `instance_type` accepts named sizes (`dev`, `basic`, `standard`, `standard-1`…`standard-4`, `lite`) or an explicit `{ "vcpu", "memory_mib", "disk_mb" }` object — check the schema for the current list. **`containers` is not inherited by named environments.**

### Secrets Store

```jsonc
{
  "secrets_store_secrets": [
    { "binding": "MY_SECRET", "store_id": "<STORE_ID>", "secret_name": "my-secret" }
  ]
}
```

### Pipelines

```jsonc
{ "pipelines": [{ "binding": "STREAM", "stream": "<STREAM_ID>" }] }
```

The value is a **Stream ID**, not a pipeline name. The older `pipeline` key is **deprecated** — use `stream`. `remote: true` is supported.

## After Any Config Change

Run `wrangler types` to regenerate `worker-configuration.d.ts` so binding types match the config.
