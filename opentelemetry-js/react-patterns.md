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
import { trace } from '@opentelemetry/api';
import { useState, useEffect, useRef } from 'react';

const tracer = trace.getTracer('user-journeys');

export function useJourneyTracker(journeyName: string) {
  const spanRef = useRef<any>(null);
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

Report Core Web Vitals as OTel spans:

```tsx
import { useEffect } from 'react';
import { trace } from '@opentelemetry/api';

export function useWebVitals() {
  useEffect(() => {
    const tracer = trace.getTracer('web-vitals');
    const reportMetric = (metric: any) => {
      const span = tracer.startSpan(`web-vital.${metric.name.toLowerCase()}`, {
        attributes: {
          'web_vital.name': metric.name,
          'web_vital.value': metric.value,
          'web_vital.rating': metric.rating, // 'good' | 'needs-improvement' | 'poor'
        },
      });
      span.end();
    };
    import('web-vitals').then(({ onCLS, onFID, onFCP, onLCP, onTTFB, onINP }) => {
      onCLS(reportMetric); onFID(reportMetric); onFCP(reportMetric);
      onLCP(reportMetric); onTTFB(reportMetric); onINP(reportMetric);
    });
  }, []);
}
```

## Redux/Zustand Middleware

```ts
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('state-management');

// Redux middleware
export const otelReduxMiddleware = (store: any) => (next: any) => (action: any) => {
  const span = tracer.startSpan(`redux.${action.type}`);
  span.setAttribute('action.type', action.type);
  try {
    const result = next(action);
    span.end();
    return result;
  } catch (error: any) {
    span.recordException(error);
    span.end();
    throw error;
  }
};
```

## Custom Frontend Sampler

Sample errors at 100%, user interactions at 50%, general traces at 10%:

```ts
import { Sampler, SamplingResult, SamplingDecision } from '@opentelemetry/api';

export class FrontendSampler implements Sampler {
  constructor(private config = {
    defaultRate: 0.1,
    errorRate: 1.0,
    interactionRate: 0.5,
  }) {}

  shouldSample(context: any, traceId: string, spanName: string, spanKind: any, attributes: any) {
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
    } as SamplingResult;
  }

  toString() { return 'FrontendSampler'; }
}
```

## Bundle Size Estimates

| Package | Gzipped Size |
|---------|-------------|
| `@opentelemetry/api` | ~5 KB |
| `@opentelemetry/sdk-trace-web` | ~15 KB |
| `@opentelemetry/context-zone` (Zone.js) | ~15 KB |
| `@opentelemetry/exporter-trace-otlp-http` | ~10 KB |
| Minimal setup total | ~45-50 KB |
| `@opentelemetry/auto-instrumentations-web` | Significantly larger |

Use individual instrumentation packages for production to minimize bundle size.
