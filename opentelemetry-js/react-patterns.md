# React OpenTelemetry Patterns Reference

Extended patterns for instrumenting React applications with OpenTelemetry.

## TracingProvider (React Context)

Wrap your app in a tracing context to provide consistent span creation across components:

```tsx
import React, { createContext, useContext, useCallback, useMemo } from 'react';
import { trace, Span, SpanStatusCode, context as otelContext } from '@opentelemetry/api';

interface TracingContextValue {
  tracer: ReturnType<typeof trace.getTracer>;
  withSpan: <T>(name: string, fn: (span: Span) => Promise<T> | T, attributes?: Record<string, any>) => Promise<T>;
}

const TracingContext = createContext<TracingContextValue | null>(null);

export function TracingProvider({ children }: { children: React.ReactNode }) {
  const tracer = useMemo(() => trace.getTracer('react-app', '1.0.0'), []);

  const withSpan = useCallback(async <T,>(
    name: string, fn: (span: Span) => Promise<T> | T, attributes: Record<string, any> = {}
  ): Promise<T> => {
    const span = tracer.startSpan(name, { attributes: { component: 'react', ...attributes } });
    try {
      const result = await otelContext.with(
        trace.setSpan(otelContext.active(), span), () => fn(span)
      );
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error: any) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      throw error;
    } finally {
      span.end();
    }
  }, [tracer]);

  const value = useMemo(() => ({ tracer, withSpan }), [tracer, withSpan]);
  return <TracingContext.Provider value={value}>{children}</TracingContext.Provider>;
}

export function useTracing() {
  const ctx = useContext(TracingContext);
  if (!ctx) throw new Error('useTracing must be used within TracingProvider');
  return ctx;
}
```

## Component Lifecycle Tracing

```tsx
import { trace } from '@opentelemetry/api';
import { useEffect } from 'react';

const tracer = trace.getTracer('react-components');

export function useComponentTracer(componentName: string) {
  useEffect(() => {
    const span = tracer.startSpan(`${componentName}.lifecycle`);
    span.addEvent('component.mounted');
    return () => {
      span.addEvent('component.unmounted');
      span.end();
    };
  }, [componentName]);
}
```

## Route Change Tracing with Render Duration

```tsx
import { useEffect, useRef } from 'react';
import { useLocation, useNavigationType } from 'react-router-dom';
import { trace, Span, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('spa-routing', '1.0.0');

export function RouteTracer({ children }: { children: React.ReactNode }) {
  const location = useLocation();
  const navigationType = useNavigationType();
  const activeSpanRef = useRef<Span | null>(null);
  const navigationStartRef = useRef(performance.now());

  useEffect(() => {
    if (activeSpanRef.current) {
      activeSpanRef.current.setStatus({ code: SpanStatusCode.OK });
      activeSpanRef.current.end();
    }

    navigationStartRef.current = performance.now();

    const span = tracer.startSpan('route.change', {
      attributes: {
        'route.path': location.pathname,
        'route.search': location.search,
        'route.navigation_type': navigationType,
      },
    });
    activeSpanRef.current = span;

    // Double rAF to measure actual paint time
    requestAnimationFrame(() => {
      requestAnimationFrame(() => {
        const renderTime = performance.now() - navigationStartRef.current;
        span.setAttribute('route.render_duration_ms', renderTime);
      });
    });

    return () => {
      if (activeSpanRef.current) {
        activeSpanRef.current.end();
        activeSpanRef.current = null;
      }
    };
  }, [location.pathname]);

  return <>{children}</>;
}

// Usage:
// <BrowserRouter>
//   <RouteTracer>
//     <Routes>...</Routes>
//   </RouteTracer>
// </BrowserRouter>
```

## User Journey Tracking (Multi-Step Flows)

Track complex user flows (checkout, onboarding) as a single long-lived span with step events:

```tsx
import { trace, type Span } from '@opentelemetry/api';
import { useState, useEffect, useRef } from 'react';

const tracer = trace.getTracer('user-journeys');

export function useJourneyTracker(journeyName: string) {
  const spanRef = useRef<Span | null>(null);
  const [currentStep, setCurrentStep] = useState<string>('');

  useEffect(() => {
    const span = tracer.startSpan(`user.${journeyName}.journey`);
    spanRef.current = span;
    return () => {
      if (currentStep !== 'complete') {
        span.addEvent('journey.abandoned', { last_step: currentStep });
      }
      span.end();
    };
  }, []);

  const completeStep = (step: string, nextStep: string) => {
    spanRef.current?.addEvent('step.completed', { step });
    spanRef.current?.addEvent('step.started', { step: nextStep });
    setCurrentStep(nextStep);
  };

  const completeJourney = () => {
    spanRef.current?.addEvent('journey.completed');
    setCurrentStep('complete');
  };

  return { completeStep, completeJourney, currentStep };
}
```

## Web Vitals Integration

Report Core Web Vitals as OTel spans. Uses `web-vitals` v5+ (v5 removed `onFID`; use `onINP` instead):

```tsx
import { useEffect } from 'react';
import { trace } from '@opentelemetry/api';

interface WebVitalMetric {
  name: string;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
}

export function useWebVitals(): void {
  useEffect(() => {
    const tracer = trace.getTracer('web-vitals');
    const reportMetric = (metric: WebVitalMetric): void => {
      const span = tracer.startSpan(`web-vital.${metric.name.toLowerCase()}`, {
        attributes: {
          'web_vital.name': metric.name,
          'web_vital.value': metric.value,
          'web_vital.rating': metric.rating,
        },
      });
      span.end();
    };
    // web-vitals v5: onFID removed, use onINP for responsiveness
    import('web-vitals').then(({ onCLS, onFCP, onLCP, onTTFB, onINP }) => {
      onCLS(reportMetric); onFCP(reportMetric);
      onLCP(reportMetric); onTTFB(reportMetric); onINP(reportMetric);
    });
  }, []);
}
```

## Redux/Zustand Middleware

```ts
import { trace } from '@opentelemetry/api';
import type { Middleware } from 'redux';

const tracer = trace.getTracer('state-management');

// Redux middleware
export const otelReduxMiddleware: Middleware = (_store) => (next) => (action) => {
  const { type } = action as { type: string };
  const span = tracer.startSpan(`redux.${type}`);
  span.setAttribute('action.type', type);
  try {
    const result = next(action);
    span.end();
    return result;
  } catch (error) {
    span.recordException(error as Error);
    span.end();
    throw error;
  }
};
```

## Custom Frontend Sampler

Sample errors at 100%, user interactions at 50%, general traces at 10%:

```ts
import {
  type Sampler,
  type SamplingResult,
  SamplingDecision,
  type Context,
  type SpanKind,
  type Attributes,
  type Link,
} from '@opentelemetry/api';

// SamplingDecision has 3 values:
//   NOT_RECORD (0) — don't record or export
//   RECORD (1) — record but don't export (isRecording=true, Sampled flag off)
//   RECORD_AND_SAMPLED (2) — record AND export

interface SamplerConfig {
  defaultRate: number;
  errorRate: number;
  interactionRate: number;
}

export class FrontendSampler implements Sampler {
  private config: SamplerConfig;

  constructor(config: Partial<SamplerConfig> = {}) {
    this.config = {
      defaultRate: 0.1,
      errorRate: 1.0,
      interactionRate: 0.5,
      ...config,
    };
  }

  shouldSample(
    _context: Context,
    _traceId: string,
    spanName: string,
    _spanKind: SpanKind,
    attributes: Attributes,
    _links: Link[],
  ): SamplingResult {
    let rate = this.config.defaultRate;
    if (spanName.startsWith('error.') || attributes?.['error'] === true) {
      rate = this.config.errorRate;
    } else if (spanName.startsWith('ui.click') || spanName.startsWith('route.')) {
      rate = this.config.interactionRate;
    }
    return {
      decision: Math.random() < rate
        ? SamplingDecision.RECORD_AND_SAMPLED
        : SamplingDecision.NOT_RECORD,
    };
  }

  toString(): string { return 'FrontendSampler'; }
}
```

## Bundle Size Estimates

| Package | Gzipped Size |
|---------|-------------|
| `@opentelemetry/api` | ~5-7 KB |
| `@opentelemetry/sdk-trace-web` | ~15-20 KB |
| `@opentelemetry/context-zone` (Zone.js) | ~15 KB |
| `@opentelemetry/exporter-trace-otlp-http` | ~10 KB |
| `@opentelemetry/core` | ~10-15 KB |
| Minimal setup total | ~50-60 KB |
| Full `@opentelemetry/auto-instrumentations-web` bundle | ~60 KB |

Use individual instrumentation packages for production to minimize bundle size. SDK 2.x improved tree-shakability.
