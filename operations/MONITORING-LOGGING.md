# ARKA Engine Monitoring & Logging Integration Setup

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical Reference

---

## Overview

This document provides comprehensive guidance for setting up monitoring, logging, and observability for ARKA Engine deployments. It covers integration with popular observability platforms and best practices for production environments.

---

## Table of Contents

1. [Observability Architecture](#observability-architecture)
2. [Metrics Collection](#metrics-collection)
3. [Logging Configuration](#logging-configuration)
4. [Distributed Tracing](#distributed-tracing)
5. [Alerting Setup](#alerting-setup)
6. [Dashboard Templates](#dashboard-templates)
7. [Platform Integrations](#platform-integrations)
8. [Troubleshooting](#troubleshooting)

---

## Observability Architecture

### Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ARKA Engine Services                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  API        │  │   RTVM      │  │   Worker    │  │  Plugins    │        │
│  │  Gateway    │  │             │  │             │  │             │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                │
│         └────────────────┴────────────────┴────────────────┘                │
│                                   │                                          │
│                    ┌──────────────┼──────────────┐                          │
│                    │              │              │                          │
│                    ▼              ▼              ▼                          │
│             ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│             │ Metrics  │  │  Logs    │  │  Traces  │                       │
│             └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
└──────────────────┼─────────────┼─────────────┼──────────────────────────────┘
                   │             │             │
                   ▼             ▼             ▼
            ┌──────────┐  ┌──────────┐  ┌──────────┐
            │Prometheus│  │  Loki    │  │  Jaeger  │
            │ /Datadog │  │/Datadog  │  │/Datadog  │
            └────┬─────┘  └────┬─────┘  └────┬─────┘
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │   Grafana    │
                        │   Dashboards │
                        └──────────────┘
```

### Supported Platforms

| Platform | Metrics | Logs | Traces | APM |
|----------|---------|------|--------|-----|
| **Datadog** | ✅ | ✅ | ✅ | ✅ |
| **Prometheus + Grafana** | ✅ | ✅ (Loki) | ✅ (Tempo) | - |
| **New Relic** | ✅ | ✅ | ✅ | ✅ |
| **Splunk** | ✅ | ✅ | ✅ | ✅ |
| **AWS CloudWatch** | ✅ | ✅ | ✅ (X-Ray) | - |
| **Elastic Stack** | ✅ | ✅ | ✅ | ✅ |

---

## Metrics Collection

### Built-in Metrics Endpoint

ARKA Engine exposes Prometheus-compatible metrics:

```yaml
# Metrics endpoint configuration
metrics:
  enabled: true
  endpoint: /metrics
  port: 9090

  # Authentication (optional)
  auth:
    enabled: true
    type: bearer_token

  # Histogram buckets
  histogram_buckets:
    latency: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
    size: [100, 500, 1000, 5000, 10000, 50000, 100000]
```

### Key Metrics

#### API Gateway Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `pact_http_requests_total` | Counter | Total HTTP requests | method, path, status |
| `pact_http_request_duration_seconds` | Histogram | Request latency | method, path |
| `pact_http_request_size_bytes` | Histogram | Request body size | method, path |
| `pact_http_response_size_bytes` | Histogram | Response body size | method, path |
| `pact_http_active_connections` | Gauge | Active connections | - |

#### RTVM Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `pact_rule_evaluations_total` | Counter | Total rule evaluations | rule_id, result |
| `pact_rule_evaluation_duration_seconds` | Histogram | Evaluation latency | rule_id |
| `pact_active_rules` | Gauge | Currently active rules | jurisdiction |
| `pact_decision_total` | Counter | Decisions made | decision_type |
| `pact_rule_cache_hits_total` | Counter | Cache hits | - |
| `pact_rule_cache_misses_total` | Counter | Cache misses | - |

#### Event Processing Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `pact_events_received_total` | Counter | Events received | event_type |
| `pact_events_processed_total` | Counter | Events processed | event_type, status |
| `pact_event_processing_duration_seconds` | Histogram | Processing time | event_type |
| `pact_event_queue_depth` | Gauge | Queue depth | queue_name |
| `pact_event_queue_lag_seconds` | Gauge | Consumer lag | consumer_group |

#### Database Metrics

| Metric | Type | Description | Labels |
|--------|------|-------------|--------|
| `pact_db_connections_active` | Gauge | Active connections | pool_name |
| `pact_db_connections_idle` | Gauge | Idle connections | pool_name |
| `pact_db_query_duration_seconds` | Histogram | Query latency | query_type |
| `pact_db_errors_total` | Counter | Database errors | error_type |

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'arka-api'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: arka-api
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
    metrics_path: /metrics
    scheme: http

  - job_name: 'arka-rtvm'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: arka-rtvm
        action: keep
    metrics_path: /metrics

  - job_name: 'arka-worker'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: arka-worker
        action: keep
    metrics_path: /metrics
```

### Datadog Agent Configuration

```yaml
# datadog-values.yaml (Helm)
datadog:
  apiKey: <DATADOG_API_KEY>
  appKey: <DATADOG_APP_KEY>

  logs:
    enabled: true
    containerCollectAll: true

  apm:
    enabled: true
    portEnabled: true

  processAgent:
    enabled: true
    processCollection: true

  # Custom metrics
  prometheusScrape:
    enabled: true
    serviceEndpoints: true

  # Environment tags
  tags:
    - "env:production"
    - "service:arka-engine"

agents:
  containers:
    agent:
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
```

---

## Logging Configuration

### Log Format

ARKA Engine uses structured JSON logging:

```json
{
  "timestamp": "2024-12-01T12:00:00.000Z",
  "level": "info",
  "service": "arka-api",
  "version": "1.0.0",
  "trace_id": "abc123def456",
  "span_id": "789ghi012",
  "message": "Request processed",
  "context": {
    "method": "POST",
    "path": "/api/v1/events",
    "status": 200,
    "duration_ms": 45,
    "tenant_id": "tenant_123",
    "request_id": "req_456"
  }
}
```

### Log Levels

| Level | Usage | Production |
|-------|-------|------------|
| `error` | Errors requiring attention | Always |
| `warn` | Warning conditions | Always |
| `info` | General operational info | Always |
| `debug` | Detailed debugging | Disabled |
| `trace` | Very detailed tracing | Disabled |

### Application Logging Configuration

```yaml
# config/logging.yaml
logging:
  level: info
  format: json

  # Output destinations
  outputs:
    - type: stdout
      level: info

    - type: file
      path: /var/log/pact/app.log
      level: info
      rotation:
        max_size: 100MB
        max_age: 7d
        max_backups: 10

  # Field masking for sensitive data
  masking:
    enabled: true
    fields:
      - password
      - api_key
      - token
      - ssn
      - account_number
    mask_pattern: "***REDACTED***"

  # Correlation
  correlation:
    enabled: true
    header: X-Request-ID

  # Sampling (for high-volume environments)
  sampling:
    enabled: false
    rate: 0.1  # 10% of debug logs
```

### Fluentd/Fluent Bit Configuration

```yaml
# fluent-bit.conf
[SERVICE]
    Flush         5
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Tag               pact.*
    Path              /var/log/containe../arka-*.log
    Parser            docker
    DB                /var/log/flb_arka.db
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    Refresh_Interval  10

[FILTER]
    Name         kubernetes
    Match        pact.*
    Merge_Log    On
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name    modify
    Match   pact.*
    Add     environment production
    Add     cluster arka-prod

[OUTPUT]
    Name            es
    Match           pact.*
    Host            elasticsearch.logging.svc
    Port            9200
    Index           arka-logs
    Type            _doc
    Logstash_Format On
    Logstash_Prefix pact
    Time_Key        @timestamp

# Alternative: Datadog
[OUTPUT]
    Name        datadog
    Match       pact.*
    apikey      ${DD_API_KEY}
    dd_service  arka-engine
    dd_source   pact
    dd_tags     env:production
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: pact_index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: s3
  aws:
    s3: s3../arka-logs-bucket
    region: us-east-1

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

---

## Distributed Tracing

### OpenTelemetry Configuration

```yaml
# otel-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  # Add resource attributes
  resource:
    attributes:
      - key: service.namespace
        value: pact
        action: insert
      - key: deployment.environment
        value: production
        action: insert

exporters:
  # Jaeger
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true

  # Datadog
  datadog:
    api:
      key: ${DD_API_KEY}
      site: datadoghq.com

  # AWS X-Ray
  awsxray:
    region: us-east-1

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [jaeger, datadog]
```

### Application Tracing Setup

```typescript
// tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'arka-api',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4317',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health', '/metrics'],
      },
      '@opentelemetry/instrumentation-pg': {
        enhancedDatabaseReporting: true,
      },
    }),
  ],
});

sdk.start();
```

### Custom Span Instrumentation

```typescript
// Custom tracing for rule evaluation
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('arka-rtvm');

async function evaluateRule(ruleId: string, event: Event): Promise<Decision> {
  return tracer.startActiveSpan('rule.evaluate', async (span) => {
    span.setAttribute('rule.id', ruleId);
    span.setAttribute('event.type', event.type);
    span.setAttribute('tenant.id', event.tenantId);

    try {
      const decision = await rtvm.evaluate(ruleId, event);

      span.setAttribute('decision.type', decision.type);
      span.setAttribute('decision.confidence', decision.confidence);
      span.setStatus({ code: SpanStatusCode.OK });

      return decision;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

---

## Alerting Setup

### Alert Rules

```yaml
# prometheus-alerts.yaml
groups:
  - name: arka-availability
    rules:
      - alert: ARKAAPIDown
        expr: up{job="arka-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "ARKA API is down"
          description: "ARKA API has been down for more than 1 minute"

      - alert: ARKAHighErrorRate
        expr: |
          sum(rate(pact_http_requests_total{status=~"5.."}[5m]))
          / sum(rate(pact_http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: ARKAHighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(pact_http_request_duration_seconds_bucket[5m])) by (le)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency"
          description: "P95 latency is {{ $value }}s"

  - name: arka-resources
    rules:
      - alert: ARKADatabaseConnectionsHigh
        expr: pact_db_connections_active / pact_db_connections_max > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connections running high"

      - alert: ARKAEventQueueBacklog
        expr: pact_event_queue_depth > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Event queue backlog growing"

      - alert: ARKAMemoryHigh
        expr: |
          container_memory_usage_bytes{pod=~"arka-.*"}
          / container_spec_memory_limit_bytes > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"

  - name: arka-compliance
    rules:
      - alert: ARKADecisionLatencyHigh
        expr: |
          histogram_quantile(0.99,
            sum(rate(pact_rule_evaluation_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Decision latency exceeds SLA"
          description: "P99 decision latency is {{ $value }}s (SLA: 500ms)"

      - alert: ARKAAuditLogFailure
        expr: increase(pact_audit_log_errors_total[5m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Audit logging failures detected"
```

### PagerDuty Integration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'pagerduty-critical'

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true

    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: '<SLACK_WEBHOOK_URL>'
        channel: '#arka-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
```

---

## Dashboard Templates

### Grafana Dashboard - Overview

```json
{
  "dashboard": {
    "title": "ARKA Engine Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(pact_http_requests_total[5m]))",
            "legendFormat": "Requests/s"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(pact_http_requests_total{status=~\"5..\"}[5m])) / sum(rate(pact_http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ]
      },
      {
        "title": "P95 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(pact_http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P95"
          }
        ]
      },
      {
        "title": "Active Rules",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(pact_active_rules)",
            "legendFormat": "Rules"
          }
        ]
      },
      {
        "title": "Decisions/min",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(pact_decision_total[1m])) * 60",
            "legendFormat": "Decisions"
          }
        ]
      },
      {
        "title": "Decision Distribution",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (decision_type) (rate(pact_decision_total[5m]))",
            "legendFormat": "{{decision_type}}"
          }
        ]
      }
    ]
  }
}
```

### Datadog Dashboard Configuration

```yaml
# datadog-dashboard.yaml
title: ARKA Engine Production
widgets:
  - definition:
      title: API Availability
      type: query_value
      requests:
        - q: avg:pact.http.requests.success_rate{env:production}
      precision: 2
      custom_unit: "%"

  - definition:
      title: Request Rate
      type: timeseries
      requests:
        - q: sum:pact.http.requests.total{env:production}.as_rate()
          display_type: line

  - definition:
      title: Latency Distribution
      type: heatmap
      requests:
        - q: avg:pact.http.request.duration{env:production} by {path}

  - definition:
      title: Error Rate by Endpoint
      type: toplist
      requests:
        - q: top(sum:pact.http.requests.errors{env:production} by {path}.as_rate(), 10, 'mean', 'desc')

  - definition:
      title: Active Connections
      type: timeseries
      requests:
        - q: avg:pact.http.active_connections{env:production}
          display_type: area
```

---

## Platform Integrations

### AWS CloudWatch

```yaml
# CloudWatch agent configuration
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/pact/*.log",
            "log_group_name": "/pact/application",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "PACT",
    "metrics_collected": {
      "prometheus": {
        "prometheus_config_path": "/etc/prometheus/prometheus.yml",
        "emf_processor": {
          "metric_namespace": "PACT"
        }
      }
    }
  }
}
```

### Splunk Integration

```yaml
# splunk-forwarder inputs.conf
[monitor:///var/log/pact/*.log]
disabled = false
index = pact
sourcetype = pact:json
host_segment = 1

[http../arka-metrics]
disabled = false
token = <SPLUNK_HEC_TOKEN>
index = pact_metrics
sourcetype = pact:metrics
```

### Elastic APM

```typescript
// Elastic APM agent initialization
import apm from 'elastic-apm-node';

apm.start({
  serviceName: 'arka-api',
  serverUrl: process.env.ELASTIC_APM_SERVER_URL,
  secretToken: process.env.ELASTIC_APM_SECRET_TOKEN,
  environment: process.env.NODE_ENV,
  captureBody: 'errors',
  transactionSampleRate: 1.0,
  metricsInterval: '30s',
});
```

---

## Troubleshooting

### Common Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Missing metrics | Grafana shows "No data" | Check scrape config, verify /metrics endpoint |
| High cardinality | Prometheus memory issues | Review label usage, add recording rules |
| Log ingestion lag | Delayed logs in search | Check Fluent Bit buffer, increase resources |
| Trace sampling | Missing traces | Verify sampling configuration |
| Alert fatigue | Too many alerts | Tune thresholds, add alert grouping |

### Diagnostic Commands

```bash
# Check metrics endpoint
curl -s http../arka-api:9090/metrics | head -50

# Verify Prometheus targets
curl -s http://prometheus:9090/api/v1/targets | jq '.data.activeTargets'

# Check log shipper status
kubectl logs -n logging ds/fluent-bit --tail=100

# Verify tracing
curl -s http://jaeger:16686/api/services | jq

# Test alertmanager
curl -X POST http://alertmanager:9093/-/reload
```

### Health Check Endpoints

| Endpoint | Purpose | Expected Response |
|----------|---------|-------------------|
| `/health` | Basic health | `{"status": "ok"}` |
| `/health/live` | Liveness probe | 200 OK |
| `/health/ready` | Readiness probe | 200 OK |
| `/metrics` | Prometheus metrics | Prometheus format |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Platform Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
