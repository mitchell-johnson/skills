# OpenTelemetry for MCP Workers

Distributed tracing for MCP servers on Cloudflare Workers. See the [opentelemetry-js](../opentelemetry-js/SKILL.md) skill for the full runtime-by-runtime OTel guide; this file covers the MCP-specific application.

Workers constraints that shape everything here:

- Use `sdk-trace-base` + `BasicTracerProvider` (never `sdk-trace-node` — Workers are V8 isolates, not Node)
- Use `SimpleSpanProcessor` (never `BatchSpanProcessor` — isolates terminate before batches flush)
- Always flush via `ctx.waitUntil(provider.forceFlush())` after the response

## Install

```bash
npm install @opentelemetry/api \
  @opentelemetry/sdk-trace-base \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/exporter-trace-otlp-http
```

## Trace Provider

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

## Traced MCP Tools

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

## Cloudflare Native Observability

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
