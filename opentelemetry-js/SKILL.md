---
name: opentelemetry-js
description: Use when adding OpenTelemetry tracing, metrics, or observability to JavaScript/TypeScript applications - covers Cloudflare Workers (V8 isolates), React browser apps, and Node.js with OTLP export, context propagation, and collector configuration
---

# OpenTelemetry for JavaScript

## Overview

OpenTelemetry (OTel) provides vendor-neutral instrumentation for distributed tracing, metrics, and logs. The JS ecosystem has three distinct runtime targets - each with different SDK packages, constraints, and export strategies. This skill covers all three: **Cloudflare Workers**, **React/Browser**, and **Node.js**.

**Core principle:** Choose the right SDK package for your runtime. The wrong package will fail at deploy time or produce no telemetry silently.

## When to Use

- Adding distributed tracing to Cloudflare Workers
- Instrumenting a React SPA or browser app with OTel
- Setting up OTLP export to a collector or backend
- Propagating trace context across service boundaries
- Debugging missing spans, broken context, or silent telemetry failures
- Choosing between direct export vs collector deployment

**When NOT to use:**
- Non-JS runtimes (Go, Python, Java have their own SDKs)
- Log-only observability (OTel JS logs API is still experimental)
- You just need simple `console.log` debugging

**Supporting files:**
- `react-patterns.md` — Extended React patterns: TracingProvider, route tracing with render duration, user journey tracking, Web Vitals, Redux middleware, custom frontend sampler, bundle size estimates
- `collector-reference.md` — Collector configs, backend comparison, Cloudflare native Destinations, Docker Compose, NGINX reverse proxy, deployment patterns

## Quick Reference: Which Packages for Which Runtime

| Runtime | Trace Provider | Span Processor | Exporter | Context Manager |
|---------|---------------|----------------|----------|-----------------|
| **Cloudflare Workers** | `BasicTracerProvider` from `sdk-trace-base` | `SimpleSpanProcessor` | `OTLPTraceExporter` from `exporter-trace-otlp-http` | Manual (no zone.js) |
| **React/Browser** | `WebTracerProvider` from `sdk-trace-web` | `BatchSpanProcessor` | `OTLPTraceExporter` from `exporter-trace-otlp-http` | `ZoneContextManager` from `context-zone` |
| **Node.js** | `NodeSDK` from `sdk-node` | `BatchSpanProcessor` (default) | `OTLPTraceExporter` from `exporter-trace-otlp-http` or `exporter-trace-otlp-grpc` | Built-in AsyncLocalStorage |

### Packages That DON'T Work in Workers/Browser

```
# NEVER install these for Workers or Browser:
@opentelemetry/sdk-trace-node     # Node.js native modules
@opentelemetry/sdk-node           # Node.js native modules
@opentelemetry/auto-instrumentations-node  # Node.js native modules
@opentelemetry/exporter-trace-otlp-grpc   # No gRPC in Workers/Browser
```

## Cloudflare Workers

Workers run on V8 isolates, not Node.js. No `fs`, `net`, `http`, or gRPC. Each request may run in a different isolate. You must use HTTP/JSON OTLP export and flush telemetry via `waitUntil`.

### Install

```sh
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-base \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/exporter-trace-otlp-http
```

### Trace Provider (tracing.js)

```js
import { trace, context, SpanStatusCode } from '@opentelemetry/api';
import { BasicTracerProvider, SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

export function createTracerProvider(env) {
  const resource = new Resource({
    [ATTR_SERVICE_NAME]: 'my-worker',
    [ATTR_SERVICE_VERSION]: '1.0.0',
    'cloud.provider': 'cloudflare',
    'cloud.platform': 'cloudflare_workers',
  });

  const exporter = new OTLPTraceExporter({
    url: env.OTEL_EXPORTER_OTLP_ENDPOINT || 'https://your-collector:4318/v1/traces',
    headers: {
      Authorization: `Bearer ${env.OTEL_API_KEY || ''}`,
    },
  });

  const provider = new BasicTracerProvider({
    resource,
    spanProcessors: [new SimpleSpanProcessor(exporter)],
  });

  provider.register();
  return provider;
}
```

**Why SimpleSpanProcessor:** Workers requests are short-lived. Simple processor exports each span immediately when it ends. BatchSpanProcessor may lose spans if the isolate terminates before the batch interval fires.

### Request Handler (index.js)

```js
import { trace, context, SpanStatusCode, propagation } from '@opentelemetry/api';
import { createTracerProvider } from './tracing.js';

export default {
  async fetch(request, env, ctx) {
    const provider = createTracerProvider(env);
    const tracer = trace.getTracer('my-worker', '1.0.0');

    // Extract incoming trace context for distributed tracing
    const parentContext = propagation.extract(
      context.active(),
      Object.fromEntries(request.headers),
      {
        get(carrier, key) { return carrier[key]; },
        keys(carrier) { return Object.keys(carrier); },
      }
    );

    const span = tracer.startSpan('worker.request', {
      attributes: {
        'http.method': request.method,
        'http.url': request.url,
        'http.target': new URL(request.url).pathname,
        'cf.colo': request.cf?.colo || 'unknown',
      },
    }, parentContext);

    let response;
    try {
      response = await context.with(
        trace.setSpan(parentContext, span),
        () => handleRequest(request, env, tracer)
      );
      span.setAttribute('http.status_code', response.status);
      if (response.status >= 400) {
        span.setStatus({ code: SpanStatusCode.ERROR });
      }
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      response = new Response('Internal Server Error', { status: 500 });
    } finally {
      span.end();
    }

    // CRITICAL: flush telemetry after response, without blocking the user
    ctx.waitUntil(provider.forceFlush());
    return response;
  },
};
```

**`waitUntil` is critical.** Without it, the isolate may terminate before telemetry is exported. It keeps the isolate alive for the flush without adding latency to the response.

### Traced Fetch Wrapper

Wrap outgoing `fetch` calls to create child spans and propagate context:

```js
import { trace, context, SpanStatusCode, propagation } from '@opentelemetry/api';

export async function tracedFetch(url, options = {}) {
  const tracer = trace.getTracer('my-worker');
  const span = tracer.startSpan('fetch', {
    attributes: {
      'http.method': options.method || 'GET',
      'http.url': url,
    },
  });

  return context.with(trace.setSpan(context.active(), span), async () => {
    const headers = new Headers(options.headers || {});
    propagation.inject(context.active(), headers, {
      set(carrier, key, value) { carrier.set(key, value); },
    });

    try {
      const response = await fetch(url, { ...options, headers });
      span.setAttribute('http.status_code', response.status);
      if (response.status >= 400) {
        span.setStatus({ code: SpanStatusCode.ERROR });
      }
      return response;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Wrangler Config

```toml
# wrangler.toml
name = "my-otel-worker"
main = "src/index.js"
compatibility_date = "2024-01-01"

[vars]
OTEL_EXPORTER_OTLP_ENDPOINT = "https://your-collector:4318/v1/traces"

# Set API key as secret (not in plaintext):
# wrangler secret put OTEL_API_KEY
```

### Alternative: Cloudflare Native OTel Destinations (Beta)

Cloudflare offers built-in OTel export without custom SDK code. Configure in `wrangler.jsonc`:

```jsonc
{
  "observability": {
    "traces": {
      "enabled": true,
      "destinations": ["my-trace-destination"],
      "head_sampling_rate": 0.05
    }
  }
}
```

Configure destination endpoints and auth in the Cloudflare dashboard. See `collector-reference.md` for details.

### Sampling for High-Traffic Workers

```js
import { TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-base';

// Sample 10% of traces
const sampler = new TraceIdRatioBasedSampler(0.1);

// Pass to provider
const provider = new BasicTracerProvider({
  resource,
  sampler,
  spanProcessors: [new SimpleSpanProcessor(exporter)],
});
```

## React / Browser

Browser instrumentation uses `WebTracerProvider` and `ZoneContextManager` for async context tracking. Use `BatchSpanProcessor` since the browser is long-lived.

### Install

```sh
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-web \
  @opentelemetry/sdk-trace-base \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/context-zone \
  @opentelemetry/instrumentation \
  @opentelemetry/instrumentation-document-load \
  @opentelemetry/instrumentation-fetch \
  @opentelemetry/instrumentation-xml-http-request \
  @opentelemetry/instrumentation-user-interaction
```

Or use the meta-package for all common browser instrumentations:

```sh
npm install @opentelemetry/auto-instrumentations-web
```

### Browser Trace Provider (instrumentation.ts)

```ts
import { WebTracerProvider } from '@opentelemetry/sdk-trace-web';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { ZoneContextManager } from '@opentelemetry/context-zone';
import { registerInstrumentations } from '@opentelemetry/instrumentation';
import { DocumentLoadInstrumentation } from '@opentelemetry/instrumentation-document-load';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';
import { UserInteractionInstrumentation } from '@opentelemetry/instrumentation-user-interaction';
import { resourceFromAttributes, defaultResource } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const resource = defaultResource().merge(
  resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'my-react-app',
    [ATTR_SERVICE_VERSION]: '1.0.0',
  })
);

const exporter = new OTLPTraceExporter({
  url: 'https://your-collector:4318/v1/traces',
  headers: { Authorization: 'Bearer YOUR_API_KEY' },
});

const provider = new WebTracerProvider({
  resource,
  spanProcessors: [new BatchSpanProcessor(exporter)],
});

provider.register({
  contextManager: new ZoneContextManager(),
});

registerInstrumentations({
  instrumentations: [
    new DocumentLoadInstrumentation(),
    new FetchInstrumentation({
      // Filter out telemetry export requests to avoid infinite loops
      ignoreUrls: [/\/v1\/traces/],
      // Propagate trace context to your API
      propagateTraceHeaderCorsUrls: [/api\.yourdomain\.com/],
    }),
    new UserInteractionInstrumentation(),
  ],
});
```

**Import this file at the top of your React app entry point (before React renders).**

> **Note:** Browser OTel instrumentation is experimental. The `propagateTraceHeaderCorsUrls` option on `FetchInstrumentation` is critical — without it, `traceparent` headers won't be injected on cross-origin API requests, breaking distributed tracing.

### CORS: Critical for Browser Export

Your OTel collector MUST allow browser origins. Without CORS headers, the browser will silently drop telemetry exports.

Collector config (`otel-collector-config.yaml`):

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: "0.0.0.0:4318"
        cors:
          allowed_origins:
            - "https://yourdomain.com"
            - "http://localhost:*"
          allowed_headers:
            - "Content-Type"
            - "Authorization"
            - "traceparent"
            - "tracestate"
```

### React-Specific Patterns

See `react-patterns.md` for extended patterns: TracingProvider context, component lifecycle tracing, user journey tracking, Web Vitals integration, Redux/Zustand middleware, custom frontend sampler, and bundle size estimates.

**Custom hook for manual spans:**

```tsx
import { trace, SpanStatusCode } from '@opentelemetry/api';

export function useTracer(name: string) {
  const tracer = trace.getTracer(name);

  return {
    traceAsync: async <T>(spanName: string, fn: () => Promise<T>): Promise<T> => {
      return tracer.startActiveSpan(spanName, async (span) => {
        try {
          const result = await fn();
          return result;
        } catch (error) {
          span.recordException(error as Error);
          span.setStatus({ code: SpanStatusCode.ERROR });
          throw error;
        } finally {
          span.end();
        }
      });
    },
  };
}

// Usage in a component:
function UserProfile({ userId }: { userId: string }) {
  const { traceAsync } = useTracer('user-profile');

  useEffect(() => {
    traceAsync('loadUserProfile', async () => {
      const response = await fetch(`/api/users/${userId}`);
      return response.json();
    });
  }, [userId]);
}
```

**Route change tracking (React Router):**

```tsx
import { trace } from '@opentelemetry/api';
import { useLocation } from 'react-router-dom';
import { useEffect, useRef } from 'react';

export function useRouteTracing() {
  const location = useLocation();
  const tracer = trace.getTracer('react-router');
  const prevPath = useRef(location.pathname);

  useEffect(() => {
    if (prevPath.current !== location.pathname) {
      const span = tracer.startSpan('route.change', {
        attributes: {
          'route.from': prevPath.current,
          'route.to': location.pathname,
        },
      });
      span.end();
      prevPath.current = location.pathname;
    }
  }, [location.pathname]);
}
```

**Error boundary integration:**

```tsx
import { trace, SpanStatusCode } from '@opentelemetry/api';

class TracedErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    const tracer = trace.getTracer('error-boundary');
    const span = tracer.startSpan('react.error');
    span.recordException(error);
    span.setAttribute('react.component_stack', info.componentStack || '');
    span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
    span.end();
  }

  render() {
    if (this.state.hasError) return <div>Something went wrong.</div>;
    return this.props.children;
  }
}
```

### Server-to-Browser Trace Linking

Set `traceparent` in your HTML to link server and browser traces:

```html
<meta name="traceparent" content="00-{traceId}-{spanId}-01" />
```

Generate this dynamically on the server with the server request's trace ID and a parent span ID.

## Shared Patterns

### JS API Quick Reference

```js
import { trace, context, SpanStatusCode, propagation } from '@opentelemetry/api';

// Get a tracer
const tracer = trace.getTracer('scope-name', '1.0.0');

// Create spans
tracer.startActiveSpan('operation', (span) => {
  span.setAttribute('key', 'value');
  span.addEvent('something.happened', { detail: 'value' });
  span.setStatus({ code: SpanStatusCode.OK });
  span.end();
});

// Manual context propagation (for Workers/sdk-trace-base)
const parentSpan = tracer.startSpan('parent');
const ctx = trace.setSpan(context.active(), parentSpan);
const childSpan = tracer.startSpan('child', undefined, ctx);

// Get active span (in zone-managed context)
const activeSpan = trace.getActiveSpan();

// Record exceptions
try { doWork(); } catch (e) {
  span.recordException(e);
  span.setStatus({ code: SpanStatusCode.ERROR });
}

// Inject context into outgoing request headers
const headers = {};
propagation.inject(context.active(), headers);

// Extract context from incoming request headers
const parentCtx = propagation.extract(context.active(), incomingHeaders);
```

### `startActiveSpan` vs `startSpan`

- **`startActiveSpan(name, callback)`** — Creates span AND sets it as active context. Child spans created inside the callback are automatically nested. **Preferred in most cases.**
- **`startSpan(name)`** — Creates span WITHOUT setting it on context. Use for independent/sibling spans or when you need manual context control.

**Important for `sdk-trace-base` (Workers):** `startActiveSpan` does NOT set the span as active context without a context manager. You must manually manage context with `context.with(trace.setSpan(...))`.

### Semantic Conventions

Always use the constants from `@opentelemetry/semantic-conventions`:

```js
import {
  ATTR_SERVICE_NAME,
  ATTR_SERVICE_VERSION,
  ATTR_HTTP_REQUEST_METHOD,    // 'http.request.method'
  ATTR_HTTP_RESPONSE_STATUS_CODE,  // 'http.response.status_code'
  ATTR_URL_FULL,               // 'url.full'
  ATTR_URL_PATH,               // 'url.path'
  ATTR_CODE_FUNCTION_NAME,     // 'code.function.name'
  ATTR_CODE_FILE_PATH,         // 'code.filepath'
} from '@opentelemetry/semantic-conventions';
```

### Resource Configuration

```js
import { Resource } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } from '@opentelemetry/semantic-conventions';

const resource = new Resource({
  [ATTR_SERVICE_NAME]: 'my-service',
  [ATTR_SERVICE_VERSION]: '1.0.0',
  'deployment.environment': 'production',
});
```

### Sampling Strategies

| Sampler | Use Case |
|---------|----------|
| `AlwaysOnSampler` | Development, low-traffic services |
| `TraceIdRatioBasedSampler(0.1)` | High-traffic production (sample 10%) |
| `ParentBasedSampler` | Respect upstream sampling decisions |
| `AlwaysOffSampler` | Disable tracing entirely |

## Collector & Backend Setup

See `collector-reference.md` for complete configs, Docker Compose, NGINX proxy, and backend comparison.

### Direct Export (No Collector)

Several backends accept OTLP directly - no collector needed:

| Backend | Endpoint | Auth |
|---------|----------|------|
| **Honeycomb** | `https://api.honeycomb.io/v1/traces` | `x-honeycomb-team: YOUR_KEY` |
| **Grafana Cloud** | `https://otlp-gateway-{region}.grafana.net/otlp/v1/traces` | Basic auth |
| **Axiom** | `https://api.axiom.co/v1/traces` | `Authorization: Bearer YOUR_TOKEN` |

### With Collector

For more control (batching, sampling, multi-backend fan-out), deploy an OTel Collector:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: "0.0.0.0:4318"
        cors:
          allowed_origins: ["https://yourdomain.com", "http://localhost:*"]
          allowed_headers: ["*"]

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  otlphttp:
    endpoint: "https://your-backend/v1/traces"
    headers:
      Authorization: "Bearer YOUR_KEY"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp]
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `sdk-trace-node` in Workers | Use `sdk-trace-base` with `BasicTracerProvider` |
| Using `sdk-node` in browser | Use `sdk-trace-web` with `WebTracerProvider` |
| Forgetting `waitUntil` in Workers | Telemetry silently lost. Always `ctx.waitUntil(provider.forceFlush())` |
| BatchSpanProcessor in Workers | Spans lost before batch fires. Use `SimpleSpanProcessor` |
| Missing CORS on collector | Browser silently drops exports. Configure `cors.allowed_origins` |
| Not filtering telemetry URLs in FetchInstrumentation | Infinite loop: tracing the trace export. Use `ignoreUrls` |
| No `ZoneContextManager` in browser | Async context breaks, child spans become roots. Import `@opentelemetry/context-zone` |
| Initializing SDK after app code | No-op tracers returned. SDK init must be the first import |
| gRPC exporter in Workers/Browser | Not supported. Use HTTP/JSON (`exporter-trace-otlp-http`) |
| Large span attributes | Bloats payload. Keep attributes small; avoid request bodies or large JSON |
| `propagateTraceHeaderCorsUrls` not set | Browser fetch instrumentation won't inject `traceparent` on cross-origin requests |
