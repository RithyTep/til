# Distributed Tracing Architecture

<div align="center">

![Tracing](https://img.shields.io/badge/Observability-Distributed_Tracing-purple?style=for-the-badge&logo=jaeger&logoColor=white)
![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-000000?style=for-the-badge&logo=opentelemetry&logoColor=white)

*End-to-end request visibility across microservices.*

</div>

## Trace Propagation

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRACE STRUCTURE                               │
│                                                                 │
│  Trace ID: abc123 (entire request flow)                        │
│  │                                                              │
│  ├─ Span: API Gateway (span-1, parent: null)                   │
│  │   ├─ Start: 0ms                                              │
│  │   ├─ Duration: 250ms                                         │
│  │   └─ Tags: http.method=POST, http.url=/orders               │
│  │                                                              │
│  ├─ Span: Order Service (span-2, parent: span-1)               │
│  │   ├─ Start: 10ms                                             │
│  │   ├─ Duration: 150ms                                         │
│  │   ├─ Events: [order.validated, inventory.checked]           │
│  │   │                                                          │
│  │   └─ Span: Database Query (span-3, parent: span-2)          │
│  │       ├─ Start: 20ms                                         │
│  │       ├─ Duration: 30ms                                      │
│  │       └─ Tags: db.type=postgresql, db.statement=SELECT...   │
│  │                                                              │
│  └─ Span: Payment Service (span-4, parent: span-2)             │
│      ├─ Start: 80ms                                             │
│      ├─ Duration: 100ms                                         │
│      └─ Tags: payment.provider=stripe, payment.status=success  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## OpenTelemetry Implementation

```typescript
import {
  trace,
  context,
  SpanKind,
  SpanStatusCode,
  propagation,
} from '@opentelemetry/api';
import { W3CTraceContextPropagator } from '@opentelemetry/core';
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';

// Initialize tracer
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: 'production',
  }),
});

provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({ url: 'http://otel-collector:4317' })
  )
);

provider.register({
  propagator: new W3CTraceContextPropagator(),
});

const tracer = trace.getTracer('order-service');

// Manual instrumentation
class OrderService {
  async createOrder(orderData: CreateOrderRequest): Promise<Order> {
    return tracer.startActiveSpan(
      'OrderService.createOrder',
      { kind: SpanKind.INTERNAL },
      async (span) => {
        try {
          span.setAttributes({
            'order.customer_id': orderData.customerId,
            'order.items_count': orderData.items.length,
          });

          // Validate order
          span.addEvent('validating_order');
          await this.validateOrder(orderData);
          span.addEvent('order_validated');

          // Check inventory (external call)
          const inventory = await tracer.startActiveSpan(
            'InventoryService.check',
            { kind: SpanKind.CLIENT },
            async (inventorySpan) => {
              try {
                const result = await this.inventoryClient.check(orderData.items);
                inventorySpan.setStatus({ code: SpanStatusCode.OK });
                return result;
              } catch (error) {
                inventorySpan.recordException(error);
                inventorySpan.setStatus({
                  code: SpanStatusCode.ERROR,
                  message: error.message,
                });
                throw error;
              } finally {
                inventorySpan.end();
              }
            }
          );

          // Process payment
          const payment = await this.processPayment(orderData);
          span.setAttribute('payment.id', payment.id);

          // Create order record
          const order = await this.orderRepository.create({
            ...orderData,
            paymentId: payment.id,
            status: 'confirmed',
          });

          span.setStatus({ code: SpanStatusCode.OK });
          span.setAttribute('order.id', order.id);

          return order;
        } catch (error) {
          span.recordException(error);
          span.setStatus({
            code: SpanStatusCode.ERROR,
            message: error.message,
          });
          throw error;
        } finally {
          span.end();
        }
      }
    );
  }
}

// Context propagation in HTTP client
async function callService(url: string, data: unknown): Promise<Response> {
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
  };

  // Inject trace context into headers
  propagation.inject(context.active(), headers);

  return tracer.startActiveSpan(
    `HTTP POST ${url}`,
    {
      kind: SpanKind.CLIENT,
      attributes: {
        'http.method': 'POST',
        'http.url': url,
      },
    },
    async (span) => {
      try {
        const response = await fetch(url, {
          method: 'POST',
          headers,
          body: JSON.stringify(data),
        });

        span.setAttributes({
          'http.status_code': response.status,
          'http.response_content_length': response.headers.get('content-length'),
        });

        if (!response.ok) {
          span.setStatus({ code: SpanStatusCode.ERROR });
        }

        return response;
      } finally {
        span.end();
      }
    }
  );
}

// Extract context on receiving side
function extractTraceContext(req: Request): Context {
  return propagation.extract(context.active(), req.headers);
}

// Server middleware
app.use((req, res, next) => {
  const extractedContext = extractTraceContext(req);

  context.with(extractedContext, () => {
    tracer.startActiveSpan(
      `${req.method} ${req.path}`,
      {
        kind: SpanKind.SERVER,
        attributes: {
          'http.method': req.method,
          'http.url': req.url,
          'http.route': req.path,
          'http.user_agent': req.headers['user-agent'],
        },
      },
      (span) => {
        // Store span in request for access in handlers
        req.span = span;

        res.on('finish', () => {
          span.setAttribute('http.status_code', res.statusCode);
          if (res.statusCode >= 400) {
            span.setStatus({ code: SpanStatusCode.ERROR });
          }
          span.end();
        });

        next();
      }
    );
  });
});
```

## Sampling Strategies

```typescript
import { Sampler, SamplingDecision, SamplingResult } from '@opentelemetry/sdk-trace-base';

// Adaptive sampling based on error rate
class AdaptiveSampler implements Sampler {
  private errorCount = 0;
  private totalCount = 0;
  private baseSampleRate = 0.01; // 1% base rate

  shouldSample(
    context: Context,
    traceId: string,
    spanName: string,
    spanKind: SpanKind,
    attributes: Attributes
  ): SamplingResult {
    this.totalCount++;

    // Always sample errors
    if (attributes['error'] === true) {
      this.errorCount++;
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }

    // Increase sampling during high error rates
    const errorRate = this.errorCount / this.totalCount;
    const adjustedRate = Math.min(1, this.baseSampleRate * (1 + errorRate * 10));

    // Consistent sampling using trace ID
    const traceIdNum = parseInt(traceId.slice(-8), 16);
    const shouldSample = (traceIdNum / 0xffffffff) < adjustedRate;

    return {
      decision: shouldSample
        ? SamplingDecision.RECORD_AND_SAMPLED
        : SamplingDecision.NOT_RECORD,
    };
  }

  toString(): string {
    return 'AdaptiveSampler';
  }
}

// Priority-based sampling
class PrioritySampler implements Sampler {
  shouldSample(
    context: Context,
    traceId: string,
    spanName: string,
    spanKind: SpanKind,
    attributes: Attributes
  ): SamplingResult {
    // High priority endpoints always sampled
    const highPriorityPaths = ['/api/payments', '/api/checkout', '/api/auth'];
    const path = attributes['http.route'] as string;

    if (highPriorityPaths.some(p => path?.startsWith(p))) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }

    // Sample based on user tier
    const userTier = attributes['user.tier'] as string;
    if (userTier === 'premium') {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }

    // Sample slow requests
    const duration = attributes['http.request.duration'] as number;
    if (duration > 1000) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }

    // Default: 1% sampling
    return {
      decision: Math.random() < 0.01
        ? SamplingDecision.RECORD_AND_SAMPLED
        : SamplingDecision.NOT_RECORD,
    };
  }
}

// Tail-based sampling (collector-side)
/*
// otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 10000
    policies:
      - name: errors-policy
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: latency-policy
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic-policy
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
*/
```

## Trace Analysis & Debugging

```typescript
// Trace analysis utilities
class TraceAnalyzer {
  // Find bottlenecks
  findBottlenecks(trace: Trace): Bottleneck[] {
    const bottlenecks: Bottleneck[] = [];
    const spans = this.flattenSpans(trace.rootSpan);

    for (const span of spans) {
      // Self-time calculation
      const childDuration = span.children.reduce(
        (sum, child) => sum + child.duration,
        0
      );
      const selfTime = span.duration - childDuration;
      const selfTimeRatio = selfTime / trace.duration;

      if (selfTimeRatio > 0.2) {
        bottlenecks.push({
          span,
          selfTime,
          selfTimeRatio,
          type: this.classifyBottleneck(span),
        });
      }
    }

    return bottlenecks.sort((a, b) => b.selfTimeRatio - a.selfTimeRatio);
  }

  // Critical path analysis
  findCriticalPath(trace: Trace): Span[] {
    const path: Span[] = [];
    let current = trace.rootSpan;

    while (current) {
      path.push(current);

      if (current.children.length === 0) break;

      // Find child that ends last
      current = current.children.reduce((latest, child) =>
        (child.startTime + child.duration) > (latest.startTime + latest.duration)
          ? child
          : latest
      );
    }

    return path;
  }

  // Service dependency map
  buildDependencyGraph(traces: Trace[]): DependencyGraph {
    const edges: Map<string, Map<string, EdgeMetrics>> = new Map();

    for (const trace of traces) {
      this.walkSpans(trace.rootSpan, (span, parent) => {
        if (parent && span.serviceName !== parent.serviceName) {
          const fromService = parent.serviceName;
          const toService = span.serviceName;

          if (!edges.has(fromService)) {
            edges.set(fromService, new Map());
          }

          const existingEdge = edges.get(fromService)!.get(toService);
          if (existingEdge) {
            existingEdge.callCount++;
            existingEdge.totalDuration += span.duration;
            existingEdge.errors += span.status.code === 'ERROR' ? 1 : 0;
          } else {
            edges.get(fromService)!.set(toService, {
              callCount: 1,
              totalDuration: span.duration,
              errors: span.status.code === 'ERROR' ? 1 : 0,
            });
          }
        }
      });
    }

    return { edges };
  }

  // Anomaly detection
  detectAnomalies(trace: Trace, baseline: TraceBaseline): Anomaly[] {
    const anomalies: Anomaly[] = [];

    this.walkSpans(trace.rootSpan, (span) => {
      const baselineMetrics = baseline.getMetrics(span.operationName);

      if (!baselineMetrics) return;

      // Duration anomaly
      const durationZScore = (span.duration - baselineMetrics.meanDuration) /
                             baselineMetrics.stdDevDuration;

      if (durationZScore > 3) {
        anomalies.push({
          span,
          type: 'duration',
          expected: baselineMetrics.meanDuration,
          actual: span.duration,
          severity: durationZScore > 5 ? 'high' : 'medium',
        });
      }

      // Missing expected child spans
      for (const expectedChild of baselineMetrics.expectedChildren) {
        const hasChild = span.children.some(
          c => c.operationName === expectedChild
        );
        if (!hasChild) {
          anomalies.push({
            span,
            type: 'missing_span',
            expected: expectedChild,
            severity: 'low',
          });
        }
      }
    });

    return anomalies;
  }

  private walkSpans(
    span: Span,
    callback: (span: Span, parent: Span | null) => void,
    parent: Span | null = null
  ): void {
    callback(span, parent);
    for (const child of span.children) {
      this.walkSpans(child, callback, span);
    }
  }
}
```

## Trace Storage & Querying

```typescript
// Jaeger-style trace storage schema
/*
CREATE TABLE traces (
  trace_id UUID PRIMARY KEY,
  root_service_name TEXT,
  root_operation_name TEXT,
  start_time TIMESTAMPTZ,
  duration_us BIGINT,
  num_spans INT,
  has_error BOOLEAN,
  INDEX idx_traces_time (start_time),
  INDEX idx_traces_service (root_service_name, start_time)
);

CREATE TABLE spans (
  trace_id UUID REFERENCES traces(trace_id),
  span_id UUID,
  parent_span_id UUID,
  service_name TEXT,
  operation_name TEXT,
  start_time TIMESTAMPTZ,
  duration_us BIGINT,
  status_code INT,
  tags JSONB,
  events JSONB,
  PRIMARY KEY (trace_id, span_id),
  INDEX idx_spans_service (service_name, operation_name, start_time)
);
*/

// Query interface
class TraceStore {
  async findTraces(query: TraceQuery): Promise<Trace[]> {
    const sql = `
      SELECT t.*, json_agg(s.*) as spans
      FROM traces t
      JOIN spans s ON t.trace_id = s.trace_id
      WHERE t.start_time BETWEEN $1 AND $2
        ${query.service ? 'AND t.root_service_name = $3' : ''}
        ${query.operation ? 'AND t.root_operation_name = $4' : ''}
        ${query.minDuration ? 'AND t.duration_us >= $5' : ''}
        ${query.hasError ? 'AND t.has_error = true' : ''}
        ${query.tags ? 'AND s.tags @> $6' : ''}
      GROUP BY t.trace_id
      ORDER BY t.start_time DESC
      LIMIT $7
    `;

    const params = [
      query.startTime,
      query.endTime,
      query.service,
      query.operation,
      query.minDuration,
      JSON.stringify(query.tags),
      query.limit || 100,
    ].filter(Boolean);

    return this.db.query(sql, params);
  }

  // Service metrics aggregation
  async getServiceMetrics(
    service: string,
    timeRange: TimeRange
  ): Promise<ServiceMetrics> {
    const sql = `
      SELECT
        operation_name,
        COUNT(*) as call_count,
        AVG(duration_us) as avg_duration,
        PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY duration_us) as p50,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_us) as p95,
        PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY duration_us) as p99,
        SUM(CASE WHEN status_code = 2 THEN 1 ELSE 0 END)::FLOAT / COUNT(*) as error_rate
      FROM spans
      WHERE service_name = $1
        AND start_time BETWEEN $2 AND $3
      GROUP BY operation_name
    `;

    return this.db.query(sql, [service, timeRange.start, timeRange.end]);
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Observability, Tracing, OpenTelemetry, Microservices, Debugging*
