# [Architecture](README.md)

## Observability

Observability is the ability to understand the internal state of a system by examining its outputs. The three pillars of observability are **Logs**, **Metrics**, and **Traces**. Together they provide a complete picture of system behavior.

## The Three Pillars

```
┌─────────────────────────────────────────────────────────────┐
│                      OBSERVABILITY                          │
├───────────────────┬───────────────────┬────────────────────┤
│       LOGS        │      METRICS      │      TRACES        │
│   (What happened) │   (Aggregated     │  (Request flow     │
│                   │    measurements)  │   across services) │
├───────────────────┼───────────────────┼────────────────────┤
│ • Debug info      │ • Request rate    │ • Distributed      │
│ • Error details   │ • Error rate      │   transactions     │
│ • Audit trail     │ • Latency         │ • Service          │
│ • Business events │ • Resource usage  │   dependencies     │
└───────────────────┴───────────────────┴────────────────────┘
```

## Logging

### Structured Logging

Always use structured (JSON) logging instead of plain text.

```php
// Bad: Plain text logs
$this->logger->info("User 123 created order 456 for $99.99");

// Good: Structured logs
$this->logger->info('Order created', [
    'user_id' => $user->id()->toString(),
    'order_id' => $order->id()->toString(),
    'total' => $order->total()->amount(),
    'currency' => $order->total()->currency()->value,
    'items_count' => $order->itemsCount(),
]);

// Output (JSON)
{
    "timestamp": "2024-01-15T10:30:00.000Z",
    "level": "info",
    "message": "Order created",
    "context": {
        "user_id": "usr_abc123",
        "order_id": "ord_def456",
        "total": 9999,
        "currency": "USD",
        "items_count": 3
    }
}
```

### Log Levels

| Level | When to Use |
|-------|-------------|
| **DEBUG** | Detailed diagnostic information (disabled in production) |
| **INFO** | Business events, state changes |
| **WARNING** | Unexpected but recoverable situations |
| **ERROR** | Failures that need attention |
| **CRITICAL** | System-wide failures |

### Correlation IDs

Track requests across services using correlation IDs.

```php
final readonly class CorrelationIdMiddleware
{
    private const HEADER = 'X-Correlation-ID';

    public function handle(Request $request, callable $next): Response
    {
        $correlationId = $request->headers->get(self::HEADER) ?? Uuid::v4()->toString();

        // Store in context for all subsequent logs
        $this->logger->pushProcessor(fn($record) => [
            ...$record,
            'extra' => [...$record['extra'], 'correlation_id' => $correlationId],
        ]);

        $response = $next($request);

        return $response->withHeader(self::HEADER, $correlationId);
    }
}

// All logs now include correlation_id
{
    "message": "Processing payment",
    "extra": {
        "correlation_id": "550e8400-e29b-41d4-a716-446655440000"
    }
}
```

### Contextual Logging

```php
final readonly class OrderService
{
    public function __construct(
        private LoggerInterface $logger,
        private OrderRepository $repository,
    ) {
    }

    public function process(Order $order): void
    {
        $context = [
            'order_id' => $order->id()->toString(),
            'user_id' => $order->userId()->toString(),
        ];

        $this->logger->info('Starting order processing', $context);

        try {
            $this->validateOrder($order);
            $this->logger->debug('Order validation passed', $context);

            $this->reserveInventory($order);
            $this->logger->debug('Inventory reserved', [
                ...$context,
                'items_count' => $order->itemsCount(),
            ]);

            $this->processPayment($order);
            $this->logger->info('Order processed successfully', $context);

        } catch (PaymentFailedException $e) {
            $this->logger->error('Payment failed', [
                ...$context,
                'error' => $e->getMessage(),
                'payment_method' => $order->paymentMethod()->value,
            ]);
            throw $e;
        }
    }
}
```

## Metrics

### Types of Metrics

**Counter** - Cumulative value that only increases
```php
$this->metrics->counter('http_requests_total', [
    'method' => $request->getMethod(),
    'path' => $request->getPathInfo(),
    'status' => $response->getStatusCode(),
])->increment();
```

**Gauge** - Value that can go up or down
```php
$this->metrics->gauge('queue_size', ['queue' => 'orders'])
    ->set($queue->count());

$this->metrics->gauge('active_connections')
    ->set($connectionPool->activeCount());
```

**Histogram** - Distribution of values
```php
$startTime = microtime(true);
$result = $this->processOrder($order);
$duration = microtime(true) - $startTime;

$this->metrics->histogram('order_processing_duration_seconds', [
    'status' => $result->isSuccess() ? 'success' : 'failure',
])->observe($duration);
```

### RED Method (Request-Driven)

For services - measure:
- **R**ate: requests per second
- **E**rrors: failed requests per second
- **D**uration: latency distribution

```php
final readonly class HttpMetricsMiddleware
{
    public function handle(Request $request, callable $next): Response
    {
        $startTime = microtime(true);

        try {
            $response = $next($request);

            $this->recordMetrics($request, $response->getStatusCode(), $startTime);

            return $response;
        } catch (\Throwable $e) {
            $this->recordMetrics($request, 500, $startTime);
            throw $e;
        }
    }

    private function recordMetrics(Request $request, int $status, float $startTime): void
    {
        $labels = [
            'method' => $request->getMethod(),
            'path' => $this->normalizePath($request->getPathInfo()),
            'status' => (string) $status,
        ];

        // Rate
        $this->metrics->counter('http_requests_total', $labels)->increment();

        // Errors
        if ($status >= 400) {
            $this->metrics->counter('http_requests_errors_total', $labels)->increment();
        }

        // Duration
        $duration = microtime(true) - $startTime;
        $this->metrics->histogram('http_request_duration_seconds', $labels)
            ->observe($duration);
    }
}
```

### USE Method (Resource-Driven)

For resources (CPU, memory, queues) - measure:
- **U**tilization: % time resource is busy
- **S**aturation: queued work
- **E**rrors: error count

```php
// Database connection pool
$this->metrics->gauge('db_pool_utilization')
    ->set($pool->usedConnections() / $pool->maxConnections());

$this->metrics->gauge('db_pool_saturation')
    ->set($pool->waitingRequests());

$this->metrics->counter('db_pool_errors_total')
    ->increment(); // On connection error
```

### Business Metrics

```php
// Order metrics
$this->metrics->counter('orders_created_total', [
    'payment_method' => $order->paymentMethod()->value,
    'currency' => $order->currency()->value,
])->increment();

$this->metrics->histogram('order_value', [
    'currency' => $order->currency()->value,
])->observe($order->total()->amount());

// User metrics
$this->metrics->counter('user_registrations_total', [
    'source' => $registration->source(),
])->increment();
```

## Distributed Tracing

### Trace Structure

```
Trace (end-to-end request)
└── Span: API Gateway (parent)
    ├── Span: Auth Service
    ├── Span: Order Service (parent)
    │   ├── Span: Database Query
    │   ├── Span: Inventory Service (HTTP)
    │   └── Span: Payment Service (HTTP)
    │       └── Span: External Payment API
    └── Span: Notification Service (async)
```

### OpenTelemetry Implementation

```php
final readonly class TracingMiddleware
{
    public function __construct(
        private TracerInterface $tracer,
    ) {
    }

    public function handle(Request $request, callable $next): Response
    {
        // Extract context from incoming headers
        $context = $this->propagator->extract($request->headers->all());

        // Start span
        $span = $this->tracer
            ->spanBuilder('HTTP ' . $request->getMethod() . ' ' . $request->getPathInfo())
            ->setParent($context)
            ->setSpanKind(SpanKind::SERVER)
            ->startSpan();

        $scope = $span->activate();

        try {
            // Add attributes
            $span->setAttribute('http.method', $request->getMethod());
            $span->setAttribute('http.url', $request->getUri());
            $span->setAttribute('http.user_agent', $request->headers->get('User-Agent'));

            $response = $next($request);

            $span->setAttribute('http.status_code', $response->getStatusCode());

            return $response;
        } catch (\Throwable $e) {
            $span->setStatus(StatusCode::ERROR, $e->getMessage());
            $span->recordException($e);
            throw $e;
        } finally {
            $span->end();
            $scope->detach();
        }
    }
}
```

### Tracing Database Queries

```php
final readonly class TracingConnection implements Connection
{
    public function __construct(
        private Connection $inner,
        private TracerInterface $tracer,
    ) {
    }

    public function executeQuery(string $sql, array $params = []): Result
    {
        $span = $this->tracer
            ->spanBuilder('SQL Query')
            ->setSpanKind(SpanKind::CLIENT)
            ->startSpan();

        $span->setAttribute('db.system', 'postgresql');
        $span->setAttribute('db.statement', $this->sanitizeSql($sql));

        try {
            $result = $this->inner->executeQuery($sql, $params);
            $span->setAttribute('db.rows_affected', $result->rowCount());
            return $result;
        } catch (\Throwable $e) {
            $span->setStatus(StatusCode::ERROR, $e->getMessage());
            throw $e;
        } finally {
            $span->end();
        }
    }
}
```

### Tracing HTTP Clients

```php
final readonly class TracingHttpClient implements HttpClientInterface
{
    public function request(string $method, string $url, array $options = []): Response
    {
        $span = $this->tracer
            ->spanBuilder("HTTP {$method}")
            ->setSpanKind(SpanKind::CLIENT)
            ->startSpan();

        // Inject trace context into outgoing headers
        $headers = $options['headers'] ?? [];
        $this->propagator->inject($headers);
        $options['headers'] = $headers;

        $span->setAttribute('http.method', $method);
        $span->setAttribute('http.url', $url);

        try {
            $response = $this->client->request($method, $url, $options);
            $span->setAttribute('http.status_code', $response->getStatusCode());
            return $response;
        } catch (\Throwable $e) {
            $span->setStatus(StatusCode::ERROR, $e->getMessage());
            throw $e;
        } finally {
            $span->end();
        }
    }
}
```

## Alerting

### Alert Rules

```yaml
# Prometheus alerting rules
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_errors_total[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile latency is {{ $value }}s"

      - alert: QueueBacklog
        expr: queue_size > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Queue backlog growing"
```

## Tools

| Category | Tools |
|----------|-------|
| **Logging** | ELK Stack, Loki, Splunk, Datadog |
| **Metrics** | Prometheus, Grafana, Datadog, New Relic |
| **Tracing** | Jaeger, Zipkin, Datadog APM, New Relic |
| **All-in-One** | Datadog, New Relic, Dynatrace |

## Summary

* Use structured logging with correlation IDs
* Apply RED method for services, USE method for resources
* Implement distributed tracing for microservices
* Set up alerting on key metrics (error rate, latency)
* Choose tools that integrate well with your stack

### Read
* [The Three Pillars of Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)
* [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
* [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
* [RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
* [USE Method](https://www.brendangregg.com/usemethod.html)