# Wrangler Command Reference

Verified against wrangler 4.123.0, 2026-08-15. `wrangler <cmd> --help` is ground truth — check it for any flag or subcommand not listed here.

Binding shapes and wrangler.jsonc fields live in [config.md](config.md).

## Project Setup

```bash
npx wrangler init my-worker           # initialize new project
npx create-cloudflare@latest my-app   # or with a framework
```

## Local Development

```bash
wrangler dev                    # local simulation (the default; there is no --local flag)
wrangler dev --env staging      # specific environment
wrangler dev --port 8787        # custom port
wrangler dev --live-reload      # live reload for HTML changes
wrangler dev --test-scheduled   # test cron handlers, then: curl http://localhost:8787/__scheduled
wrangler dev -r                 # remote/edge dev (legacy)
```

## Deploy

```bash
wrangler deploy                 # deploy to production
wrangler deploy --env staging   # deploy specific environment
wrangler deploy --dry-run       # validate without deploying
wrangler deploy --keep-vars     # keep dashboard-set variables
wrangler deploy --minify        # minify code
```

## Versions, Deployments, Rollback, Triggers

```bash
wrangler versions upload             # upload a new version without deploying it
wrangler versions deploy             # gradual deployment: split traffic between versions
wrangler versions list
wrangler versions view <VERSION_ID>
wrangler deployments list            # deployment history
wrangler rollback                    # roll back to previous version
wrangler rollback <VERSION_ID>       # roll back to specific version
wrangler triggers deploy             # apply routes/schedules from config without a full deploy
```

## Secrets (per-Worker)

> Never pass secret values as command arguments or pipe them via `echo`. Use the
> interactive prompt (preferred), pipe from a file, or use `secret bulk`.

```bash
wrangler secret put API_KEY                     # interactive prompt (preferred)
wrangler secret put PRIVATE_KEY < path/key.pem  # from a file (PEM keys, CI)
wrangler secret list
wrangler secret delete API_KEY
wrangler secret bulk secrets.json               # do not commit this file
```

## Types & Checks

```bash
wrangler types                  # generate worker-configuration.d.ts
wrangler types ./src/env.d.ts   # custom output path
wrangler types --check          # verify types are up to date (CI)
wrangler check startup          # profile Worker startup time
```

## Observability

```bash
wrangler tail                   # stream live logs (add a Worker name to target one)
wrangler tail --status error    # filter by status
wrangler tail --search "term"   # filter by search term
wrangler tail --format json     # JSON output
```

## KV (Key-Value Store)

```bash
wrangler kv namespace create MY_KV
wrangler kv namespace list
wrangler kv namespace rename          # see --help for old/new name flags
wrangler kv namespace delete --namespace-id <ID>

wrangler kv key put --namespace-id <ID> "key" "value"
wrangler kv key put --namespace-id <ID> "key" --path ./file.bin            # value from file
wrangler kv key put --namespace-id <ID> "key" "value" --metadata '{"k":"v"}'
wrangler kv key put --namespace-id <ID> "key" "value" --ttl 3600           # expire N seconds from now
wrangler kv key put --namespace-id <ID> "key" "value" --expiration <unix-epoch>
wrangler kv key get --namespace-id <ID> "key"
wrangler kv key list --namespace-id <ID>
wrangler kv key delete --namespace-id <ID> "key"

wrangler kv bulk put --namespace-id <ID> data.json
wrangler kv bulk get --namespace-id <ID> keys.json
```

## R2 (Object Storage)

```bash
wrangler r2 bucket create my-bucket
wrangler r2 bucket create my-bucket --location weur   # weur|eeur|apac|wnam|enam|oc
wrangler r2 bucket list
wrangler r2 bucket info my-bucket
wrangler r2 bucket update                             # e.g. storage class; see --help
wrangler r2 bucket delete my-bucket

wrangler r2 object put my-bucket/path/file.txt --file ./local-file.txt
wrangler r2 object get my-bucket/path/file.txt
wrangler r2 object delete my-bucket/path/file.txt
```

Bucket subgroups (each has its own subcommands — run `wrangler r2 bucket <group> --help`):
`catalog` (R2 Data Catalog), `lifecycle`, `cors`, `notification`, `domain`, `dev-url`, `lock`, `sippy`.

## D1 (SQL Database)

```bash
wrangler d1 create my-database
wrangler d1 create my-database --location wnam
wrangler d1 list
wrangler d1 info my-database
wrangler d1 insights my-database        # query performance insights
wrangler d1 delete my-database

wrangler d1 execute my-database --remote --command "SELECT * FROM users"
wrangler d1 execute my-database --remote --file ./schema.sql
wrangler d1 execute my-database --local --command "SELECT * FROM users"

wrangler d1 migrations create my-database create_users_table
wrangler d1 migrations list my-database --local
wrangler d1 migrations apply my-database --local
wrangler d1 migrations apply my-database --remote

wrangler d1 time-travel info my-database
wrangler d1 time-travel restore my-database    # see --help for timestamp/bookmark flags

wrangler d1 export my-database --remote --output backup.sql
wrangler d1 export my-database --remote --output schema.sql --no-data
```

Auto-config flags: `--update-config`, `--binding <NAME>`, and `--use-remote` on `d1 create` (also on `r2 bucket create`, `vectorize create`, `hyperdrive create`) write the new resource's binding into wrangler config automatically; `--use-remote` sets `remote: true` on it.

## Vectorize (Vector Database)

```bash
wrangler vectorize create my-index --dimensions 768 --metric cosine
wrangler vectorize create my-index --preset @cf/baai/bge-base-en-v1.5   # auto-configures dimensions/metric
wrangler vectorize list
wrangler vectorize get my-index
wrangler vectorize delete my-index

wrangler vectorize insert my-index --file vectors.ndjson
wrangler vectorize upsert my-index --file vectors.ndjson
wrangler vectorize query my-index --vector "[0.1, 0.2, ...]" --top-k 10
wrangler vectorize list-vectors my-index
wrangler vectorize get-vectors my-index       # see --help for id flags
wrangler vectorize delete-vectors my-index    # see --help for id flags

wrangler vectorize create-metadata-index my-index   # see --help for property/type flags
wrangler vectorize list-metadata-index my-index
wrangler vectorize delete-metadata-index my-index
```

## Hyperdrive (Database Accelerator)

```bash
wrangler hyperdrive create my-hyperdrive \
  --origin-host db.example.com \
  --origin-port 5432 \
  --database my-database \
  --origin-user db-user \
  --origin-password "$DB_PASSWORD"

# Or using a connection string from an environment variable
wrangler hyperdrive create my-hyperdrive --connection-string "$HYPERDRIVE_CONNECTION_STRING"

wrangler hyperdrive list
wrangler hyperdrive get <HYPERDRIVE_ID>
wrangler hyperdrive update <HYPERDRIVE_ID> --origin-password "$DB_PASSWORD"
wrangler hyperdrive delete <HYPERDRIVE_ID>
```

## Workers AI

```bash
wrangler ai models          # list available models
wrangler ai finetune list   # list finetunes
```

Workers AI always runs remotely and incurs usage charges even in local dev.

## Queues

```bash
wrangler queues create my-queue
wrangler queues list
wrangler queues info my-queue
wrangler queues delete my-queue
wrangler queues pause-delivery my-queue
wrangler queues resume-delivery my-queue
wrangler queues purge my-queue

# Worker (push) consumers — the old alias `queues consumer add|remove` still works
wrangler queues consumer worker add my-queue my-worker
wrangler queues consumer worker remove my-queue my-worker
wrangler queues consumer worker list my-queue

# HTTP (pull) consumers
wrangler queues consumer http add my-queue
wrangler queues consumer http remove my-queue
wrangler queues consumer http list my-queue

# Event subscriptions (create|list|get|delete) — see --help for flags
wrangler queues subscription --help
```

## Containers

> Never hardcode registry credentials in commands. Use environment variables.

```bash
wrangler containers build -t my-app:latest .          # build image
wrangler containers build -t my-app:latest . --push   # build and push
wrangler containers push my-app:latest                # push existing image

wrangler containers list
wrangler containers info <CONTAINER_ID>
wrangler containers delete <CONTAINER_ID>
wrangler containers instances       # manage running instances; see --help
wrangler containers ssh             # shell into an instance; see --help

wrangler containers images list
wrangler containers images delete my-app:latest

wrangler containers registries list
wrangler containers registries configure <DOMAIN> --aws-access-key-id "$AWS_ACCESS_KEY_ID"
wrangler containers registries configure <DOMAIN> --dockerhub-username "$DOCKERHUB_USERNAME"
wrangler containers registries credentials    # see --help
wrangler containers registries delete <DOMAIN>
```

## Workflows

```bash
wrangler workflows list
wrangler workflows describe my-workflow
wrangler workflows trigger my-workflow
wrangler workflows trigger my-workflow '{"key":"value"}'   # params are a positional JSON argument
wrangler workflows delete my-workflow

wrangler workflows instances list my-workflow
wrangler workflows instances describe my-workflow <INSTANCE_ID>
wrangler workflows instances pause my-workflow <INSTANCE_ID>
wrangler workflows instances resume my-workflow <INSTANCE_ID>
wrangler workflows instances restart my-workflow <INSTANCE_ID>
wrangler workflows instances terminate my-workflow <INSTANCE_ID>
wrangler workflows instances send-event my-workflow <INSTANCE_ID>   # see --help for event type/payload flags
```

Workflows commands accept `--local` to operate against the local `wrangler dev` session instead of the deployed Workflow.

## Pipelines (Open Beta)

Architecture: **streams** (ingest endpoints) → **SQL pipeline** (transform) → **sinks** (destinations, e.g. R2).

```bash
wrangler pipelines setup    # interactive guided creation of stream + pipeline + sink (start here)

wrangler pipelines create my-pipeline --sql "INSERT INTO my_sink SELECT * FROM my_stream"
wrangler pipelines create my-pipeline --sql-file ./query.sql
wrangler pipelines list
wrangler pipelines get my-pipeline
wrangler pipelines delete my-pipeline
# `wrangler pipelines update` applies to legacy pipelines only

wrangler pipelines streams create|list|get|delete   # see --help for flags
wrangler pipelines sinks create|list|get|delete     # see --help for flags
```

## Secrets Store (Open Beta)

**All secrets-store commands run against a LOCAL SIMULATION by default** — including `store create` and `store delete`. Pass `--remote` to operate on the real store.

```bash
wrangler secrets-store store create my-store --remote
wrangler secrets-store store list --remote
wrangler secrets-store store delete <STORE_ID> --remote

# Create a secret (there is no `secret put`). Value is entered at the interactive
# prompt — a --value flag exists but is insecure; avoid it.
wrangler secrets-store secret create <STORE_ID> --name my-secret --scopes workers --remote

# get/update/delete take --secret-id <ID>, not a name
wrangler secrets-store secret list <STORE_ID> --remote
wrangler secrets-store secret get <STORE_ID> --secret-id <SECRET_ID> --remote
wrangler secrets-store secret update <STORE_ID> --secret-id <SECRET_ID> --remote
wrangler secrets-store secret duplicate <STORE_ID> --secret-id <SECRET_ID> --remote   # see --help
wrangler secrets-store secret delete <STORE_ID> --secret-id <SECRET_ID> --remote
```

## Pages (Frontend Deployment)

```bash
wrangler pages project create my-site
wrangler pages deploy ./dist
wrangler pages deploy ./dist --branch main
wrangler pages deployment list --project-name my-site
```

## Testing (Vitest)

```bash
npm install -D @cloudflare/vitest-pool-workers vitest
```

`vitest.config.ts`:

```typescript
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
