# Modern Observability Stack

Metrics, logs, and traces for production systems.

## OpenTelemetry Instrumentation

```typescript
// Unified telemetry with OpenTelemetry
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-grpc';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'api-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.VERSION,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4317',
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: 'http://otel-collector:4317',
    }),
    exportIntervalMillis: 10000,
  }),
  logRecordProcessor: new BatchLogRecordProcessor(
    new OTLPLogExporter({
      url: 'http://otel-collector:4317',
    })
  ),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },
      '@opentelemetry/instrumentation-http': {
        requestHook: (span, request) => {
          span.setAttribute('http.request_id', request.headers['x-request-id']);
        },
      },
    }),
  ],
});

sdk.start();
```

## Custom Metrics & Business KPIs

```typescript
import { metrics } from '@opentelemetry/api';

const meter = metrics.getMeter('business-metrics');

// Counters
const ordersCounter = meter.createCounter('orders.created', {
  description: 'Total orders created',
  unit: '1',
});

const revenueCounter = meter.createCounter('orders.revenue', {
  description: 'Total revenue',
  unit: 'USD',
});

// Histograms
const orderValueHistogram = meter.createHistogram('orders.value', {
  description: 'Order value distribution',
  unit: 'USD',
  boundaries: [10, 50, 100, 250, 500, 1000, 5000],
});

const latencyHistogram = meter.createHistogram('http.server.duration', {
  description: 'Request latency',
  unit: 'ms',
  boundaries: [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000],
});

// Gauges (Observable)
const activeConnections = meter.createObservableGauge('connections.active', {
  description: 'Active database connections',
});

activeConnections.addCallback((result) => {
  result.observe(connectionPool.activeCount, { pool: 'postgres' });
});

// Usage in business logic
async function createOrder(order: Order): Promise<void> {
  const startTime = Date.now();

  try {
    await processOrder(order);

    ordersCounter.add(1, {
      'order.type': order.type,
      'customer.tier': order.customer.tier,
      'payment.method': order.paymentMethod,
    });

    revenueCounter.add(order.total.amount, {
      currency: order.total.currency,
      'order.type': order.type,
    });

    orderValueHistogram.record(order.total.amount, {
      'customer.tier': order.customer.tier,
    });
  } finally {
    latencyHistogram.record(Date.now() - startTime, {
      'http.method': 'POST',
      'http.route': '/orders',
    });
  }
}
```

## Distributed Tracing with Context Propagation

```typescript
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('order-service');

class OrderService {
  async createOrder(command: CreateOrderCommand): Promise<Order> {
    return tracer.startActiveSpan('OrderService.createOrder', async (span) => {
      try {
        span.setAttributes({
          'order.customer_id': command.customerId,
          'order.items_count': command.items.length,
        });

        // Validate inventory (child span created automatically via instrumentation)
        const inventory = await this.inventoryClient.checkAvailability(command.items);
        span.addEvent('inventory_checked', { available: inventory.allAvailable });

        if (!inventory.allAvailable) {
          span.setStatus({ code: SpanStatusCode.ERROR, message: 'Insufficient inventory' });
          throw new InsufficientInventoryError(inventory.unavailableItems);
        }

        // Process payment
        const payment = await this.paymentService.charge(command);
        span.addEvent('payment_processed', { paymentId: payment.id });
        span.setAttribute('payment.id', payment.id);

        // Create order
        const order = await this.orderRepository.save(
          Order.create(command, payment)
        );

        span.setStatus({ code: SpanStatusCode.OK });
        span.setAttribute('order.id', order.id);

        return order;
      } catch (error) {
        span.recordException(error);
        span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
        throw error;
      } finally {
        span.end();
      }
    });
  }
}

// HTTP client with trace propagation
import { W3CTraceContextPropagator } from '@opentelemetry/core';

const propagator = new W3CTraceContextPropagator();

async function callExternalService(url: string, data: unknown): Promise<Response> {
  const headers: Record<string, string> = {};

  // Inject trace context into headers
  propagator.inject(context.active(), headers, {
    set: (carrier, key, value) => { carrier[key] = value; },
  });

  return fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...headers, // Includes traceparent, tracestate
    },
    body: JSON.stringify(data),
  });
}
```

## Prometheus Alerting Rules

```yaml
# prometheus-rules.yaml
groups:
  - name: api-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"
          runbook_url: "https://runbooks.example.com/high-error-rate"

      # Latency SLO breach
      - alert: LatencySLOBreach
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency SLO breach on {{ $labels.service }}"
          description: "P99 latency is {{ $value | humanizeDuration }}"

      # Pod not ready
      - alert: PodNotReady
        expr: |
          sum by (namespace, pod) (
            kube_pod_status_phase{phase=~"Pending|Unknown"} == 1
          ) > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} not ready"

      # Memory pressure
      - alert: HighMemoryUsage
        expr: |
          (
            container_memory_working_set_bytes
            / container_spec_memory_limit_bytes
          ) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage in {{ $labels.pod }}"
          description: "Memory usage is at {{ $value | humanizePercentage }}"

      # Error budget burn rate
      - alert: ErrorBudgetBurnRate
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status!~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            )
          ) / (1 - 0.999) > 14.4
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning too fast"
          description: "At current rate, monthly error budget will be exhausted"
```

## Grafana Dashboard as Code

```typescript
// dashboard.ts - Generate Grafana dashboards programmatically
import { Dashboard, Row, Graph, SingleStat, Table } from 'grafana-foundation-sdk';

const apiDashboard = new Dashboard({
  title: 'API Service Overview',
  uid: 'api-overview',
  tags: ['api', 'production'],
  refresh: '30s',
  time: { from: 'now-1h', to: 'now' },
})
  .addRow(new Row({ title: 'Overview' })
    .addPanel(new SingleStat({
      title: 'Request Rate',
      datasource: 'prometheus',
      targets: [{
        expr: 'sum(rate(http_requests_total{service="api"}[5m]))',
        legendFormat: 'Requests/sec',
      }],
      format: 'short',
      sparkline: { show: true },
    }))
    .addPanel(new SingleStat({
      title: 'Error Rate',
      datasource: 'prometheus',
      targets: [{
        expr: `
          sum(rate(http_requests_total{service="api",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="api"}[5m]))
        `,
      }],
      format: 'percentunit',
      thresholds: '0.01,0.05',
      colors: ['green', 'yellow', 'red'],
    }))
    .addPanel(new SingleStat({
      title: 'P99 Latency',
      datasource: 'prometheus',
      targets: [{
        expr: `
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{service="api"}[5m])) by (le)
          )
        `,
      }],
      format: 'dtdurations',
      thresholds: '0.1,0.5',
    }))
  )
  .addRow(new Row({ title: 'Traffic' })
    .addPanel(new Graph({
      title: 'Request Rate by Endpoint',
      datasource: 'prometheus',
      targets: [{
        expr: 'sum(rate(http_requests_total{service="api"}[5m])) by (endpoint)',
        legendFormat: '{{ endpoint }}',
      }],
      yAxes: [{ format: 'reqps' }],
    }))
  );

export default apiDashboard;
```

---

*Learned: 2024*
*Tags: Observability, OpenTelemetry, Prometheus, Grafana, Tracing*
