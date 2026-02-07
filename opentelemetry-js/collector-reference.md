# OTel Collector & Deployment Reference

Extended reference for collector configuration, backend options, and deployment patterns.

## Cloudflare Workers Native Destinations (Beta)

Cloudflare offers native OTel Destinations that export telemetry without custom SDK code:

```jsonc
// wrangler.jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["my-trace-destination"],
      "head_sampling_rate": 0.05
    },
    "logs": {
      "enabled": true,
      "destinations": ["my-log-destination"],
      "head_sampling_rate": 0.6
    }
  }
}
```

Configure destinations in the Cloudflare dashboard with OTLP endpoint URL and auth headers. Metrics export not yet supported via Destinations.

## Deployment Patterns

### Pattern 1: Direct Export (Simplest)

```
[Browser App] --OTLP/HTTP--> [Honeycomb/Axiom/Grafana Cloud]
[CF Worker]   --OTLP/HTTP--> [Honeycomb/Axiom/Grafana Cloud]
```

No infrastructure to manage. API keys exposed in browser code (use ingest-only keys).

### Pattern 2: Collector Gateway

```
[Browser App] --OTLP/HTTP--> [OTel Collector :4318] --OTLP/HTTP--> [Backend]
[CF Worker]   --OTLP/HTTP--> [OTel Collector :4318] --OTLP/HTTP--> [Backend]
```

Unified processing, hides API keys from browser, supports retries/batching.

### Pattern 3: Hybrid (Recommended for Production)

```
[Browser App] --OTLP/HTTP--> [OTel Collector :4318] --OTLP/HTTP--> [Backend]
[CF Worker]   --Destinations--> [Backend directly]
```

Browser routes through collector (hides keys). Workers use native Destinations (no collector needed).

## Backend Comparison

| Backend | Free Tier | Signals | Best For |
|---------|-----------|---------|----------|
| **Honeycomb** | ~20M events/mo | Traces, Metrics, Logs | Trace debugging, high-cardinality analysis |
| **Axiom** | 500 GB/mo ingest | Traces, Metrics, Logs | All-in-one on a budget |
| **Grafana Cloud** | 50 GB traces, 50 GB logs | Traces, Metrics, Logs | Teams familiar with Grafana/Prometheus |
| **Jaeger** (self-hosted) | Free | Traces only | Local dev, zero cost |
| **Datadog** | No free traces | All | Large teams with budget |

**Recommendation:** Honeycomb for getting started (best DX). Axiom for generous free tier. Jaeger for local dev.

## Complete Collector Config

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - https://your-app.com
            - http://localhost:3000
            - http://localhost:5173
          allowed_headers:
            - Content-Type
            - Authorization
          max_age: 7200
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
  batch:
    send_batch_size: 8192
    timeout: 10s
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert

exporters:
  # Honeycomb
  otlphttp/honeycomb:
    endpoint: https://api.honeycomb.io
    headers:
      x-honeycomb-team: ${HONEYCOMB_API_KEY}

  # Axiom
  otlphttp/axiom:
    endpoint: https://api.axiom.co
    headers:
      Authorization: Bearer ${AXIOM_API_TOKEN}
      X-Axiom-Dataset: ${AXIOM_DATASET}

  # Grafana Cloud
  otlphttp/grafana:
    endpoint: https://otlp-gateway-${REGION}.grafana.net/otlp
    headers:
      Authorization: Basic ${GRAFANA_CLOUD_TOKEN}

  # Debug (development)
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/honeycomb]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/honeycomb]
```

## Docker Compose for Local Development

```yaml
version: '3.8'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    environment:
      - HONEYCOMB_API_KEY=${HONEYCOMB_API_KEY}

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # Jaeger UI
      - "14317:4317"   # OTLP gRPC
      - "14318:4318"   # OTLP HTTP
```

Jaeger UI at `http://localhost:16686`.

## NGINX Reverse Proxy (Production)

Put NGINX in front of the collector for SSL termination and CORS:

```nginx
server {
    listen 443 ssl;
    server_name otel.yourdomain.com;

    ssl_certificate /etc/ssl/certs/otel.crt;
    ssl_certificate_key /etc/ssl/private/otel.key;

    location / {
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Access-Control-Allow-Origin' 'https://your-app.com' always;
            add_header 'Access-Control-Allow-Headers' 'Content-Type,Authorization' always;
            add_header 'Access-Control-Allow-Methods' 'POST,OPTIONS' always;
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' 'https://your-app.com' always;
        proxy_pass http://collector:4318;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
