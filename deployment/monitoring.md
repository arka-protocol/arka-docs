# Monitoring and Observability

This guide covers monitoring, logging, tracing, and alerting for ARKA Engine deployments.

## Overview

Comprehensive observability is critical for production compliance systems:

```
┌─────────────────────────────────────────────────────────────┐
│                  Observability Stack                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Metrics   │  │   Logging   │  │     Tracing         │  │
│  │ (Prometheus)│  │  (Loki/ELK) │  │  (Jaeger/Tempo)     │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │                │                     │             │
│         └────────────────┼─────────────────────┘             │
│                          │                                   │
│                    ┌─────▼─────┐                            │
│                    │  Grafana  │                            │
│                    │ Dashboard │                            │
│                    └─────┬─────┘                            │
│                          │                                   │
│                    ┌─────▼─────┐                            │
│                    │ Alerting  │                            │
│                    │(PagerDuty)│                            │
│                    └───────────┘                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Metrics

### Prometheus Metrics

The ARKA Engine exposes comprehensive Prometheus metrics:

#### Core Metrics

```yaml
# Key ARKA Engine metrics
pact_evaluations_total:
  type: counter
  description: Total compliance evaluations
  labels: [jurisdiction, domain, status, rule_set]

pact_evaluation_duration_seconds:
  type: histogram
  description: Evaluation latency
  labels: [jurisdiction, domain]
  buckets: [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

pact_active_evaluations:
  type: gauge
  description: Currently active evaluations

pact_rules_loaded:
  type: gauge
  description: Number of rules loaded
  labels: [jurisdiction, domain, version]

pact_violations_total:
  type: counter
  description: Total violations detected
  labels: [jurisdiction, domain, rule_id, severity]

pact_proofs_generated_total:
  type: counter
  description: Total proofs generated

pact_proofs_anchored_total:
  type: counter
  description: Total proofs anchored to blockchain
  labels: [chain]

pact_rtvm_executions_total:
  type: counter
  description: RTVM bytecode executions

pact_rtvm_execution_duration_seconds:
  type: histogram
  description: RTVM execution time

arka_plugin_calls_total:
  type: counter
  description: Plugin function calls
  labels: [plugin, function, status]

arka_plugin_duration_seconds:
  type: histogram
  description: Plugin call duration
  labels: [plugin, function]
```

#### Agent Metrics

```yaml
# Distributed agent metrics
pact_agent_heartbeat_timestamp:
  type: gauge
  description: Last heartbeat time
  labels: [agent_id, region]

pact_agent_sync_duration_seconds:
  type: histogram
  description: Rule sync duration
  labels: [agent_id]

pact_agent_offline_duration_seconds:
  type: histogram
  description: Offline duration
  labels: [agent_id]

pact_agent_proof_queue_size:
  type: gauge
  description: Queued proofs pending sync
  labels: [agent_id]

pact_agent_rules_cached:
  type: gauge
  description: Rules cached locally
  labels: [agent_id]
```

#### Governance Metrics

```yaml
# Governance workflow metrics
pact_proposals_total:
  type: counter
  description: Total proposals
  labels: [status, priority]

pact_proposal_duration_seconds:
  type: histogram
  description: Time from proposal to deployment
  labels: [priority]

pact_simulations_total:
  type: counter
  description: Total simulations run
  labels: [mode, status]

pact_simulation_duration_seconds:
  type: histogram
  description: Simulation execution time
  labels: [mode]
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'arka-engine'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: arka-engine
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  - job_name: 'arka-agents'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: arka-agent
        action: keep
```

### Grafana Dashboards

#### Main Dashboard

```json
{
  "dashboard": {
    "title": "ARKA Engine Overview",
    "panels": [
      {
        "title": "Evaluation Rate",
        "type": "stat",
        "targets": [{
          "expr": "sum(rate(pact_evaluations_total[5m]))"
        }]
      },
      {
        "title": "Compliance Rate",
        "type": "gauge",
        "targets": [{
          "expr": "sum(rate(pact_evaluations_total{status='compliant'}[5m])) / sum(rate(pact_evaluations_total[5m])) * 100"
        }]
      },
      {
        "title": "P99 Latency",
        "type": "stat",
        "targets": [{
          "expr": "histogram_quantile(0.99, sum(rate(pact_evaluation_duration_seconds_bucket[5m])) by (le))"
        }]
      },
      {
        "title": "Evaluations by Jurisdiction",
        "type": "piechart",
        "targets": [{
          "expr": "sum by (jurisdiction) (rate(pact_evaluations_total[1h]))"
        }]
      },
      {
        "title": "Violations by Severity",
        "type": "timeseries",
        "targets": [{
          "expr": "sum by (severity) (rate(pact_violations_total[5m]))"
        }]
      },
      {
        "title": "Active Agents",
        "type": "stat",
        "targets": [{
          "expr": "count(pact_agent_heartbeat_timestamp > (time() - 60))"
        }]
      }
    ]
  }
}
```

#### SLO Dashboard

```yaml
# slo-dashboard.yaml
panels:
  - title: "Availability SLO (99.9%)"
    type: gauge
    query: |
      (1 - (
        sum(rate(pact_evaluations_total{status="error"}[30d]))
        /
        sum(rate(pact_evaluations_total[30d]))
      )) * 100
    thresholds:
      - value: 99
        color: red
      - value: 99.9
        color: yellow
      - value: 99.99
        color: green

  - title: "Latency SLO (P99 < 500ms)"
    type: gauge
    query: |
      histogram_quantile(0.99, sum(rate(pact_evaluation_duration_seconds_bucket[30d])) by (le)) * 1000
    thresholds:
      - value: 500
        color: green
      - value: 750
        color: yellow
      - value: 1000
        color: red

  - title: "Error Budget Remaining"
    type: stat
    query: |
      1 - (
        sum(increase(pact_evaluations_total{status="error"}[30d]))
        /
        (sum(increase(pact_evaluations_total[30d])) * 0.001)
      )
```

## Logging

### Structured Logging

```yaml
# logging-config.yaml
logging:
  level: info
  format: json

  # Log fields
  fields:
    service: arka-engine
    version: ${VERSION}
    environment: ${ENVIRONMENT}

  # Sampling for high-volume logs
  sampling:
    initial: 100
    thereafter: 100

  # Output configuration
  outputs:
    - type: stdout
      format: json

    - type: file
      path: /var/log/pact/engine.log
      maxSize: 100Mi
      maxBackups: 10
      maxAge: 30

    - type: loki
      endpoint: http://loki:3100/loki/api/v1/push
      labels:
        service: arka-engine
```

### Log Format

```json
{
  "timestamp": "2024-03-15T10:30:00.123Z",
  "level": "info",
  "logger": "pact.evaluation",
  "message": "Evaluation completed",
  "trace_id": "abc123",
  "span_id": "def456",
  "service": "arka-engine",
  "version": "1.2.0",
  "evaluation": {
    "id": "eval-789",
    "transaction_id": "txn-123",
    "jurisdiction": "US",
    "domains": ["AML", "SANCTIONS"],
    "compliant": true,
    "rules_evaluated": 15,
    "duration_ms": 45
  },
  "proof": {
    "id": "proof-xyz",
    "hash": "0xabc..."
  }
}
```

### Log Queries (Loki)

```logql
# Error rate by service
sum by (service) (rate({job="arka-engine"} |= "error" [5m]))

# Slow evaluations
{job="arka-engine"} | json | duration_ms > 500

# Non-compliant transactions
{job="arka-engine"} | json | compliant = "false"

# Violations by rule
{job="arka-engine"} | json | line_format "{{.rule_id}}" | __error__=""

# Agent sync failures
{job="arka-agent"} |= "sync" |= "error"
```

### ELK Stack Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - /var/log/containe../arka-*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  indices:
    - index: "arka-engine-%{+yyyy.MM.dd}"
      when.contains:
        kubernetes.labels.app: "arka-engine"
    - index: "arka-agent-%{+yyyy.MM.dd}"
      when.contains:
        kubernetes.labels.app: "arka-agent"

# Index template
setup.template:
  name: "arka"
  pattern: "arka-*"
  settings:
    number_of_shards: 3
    number_of_replicas: 1
```

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
    timeout: 1s
    send_batch_size: 1000

  attributes:
    actions:
      - key: environment
        value: ${ENVIRONMENT}
        action: upsert

  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow
        type: latency
        latency: {threshold_ms: 1000}
      - name: sample
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes, tail_sampling]
      exporters: [jaeger, otlp/tempo]
```

### Application Instrumentation

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Configure tracing
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Add OTLP exporter
otlp_exporter = OTLPSpanExporter(endpoint="otel-collector:4317")
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

# Custom span example
async def evaluate_transaction(transaction: dict):
    with tracer.start_as_current_span("evaluate_transaction") as span:
        span.set_attribute("transaction.id", transaction["id"])
        span.set_attribute("transaction.amount", transaction["amount"])

        # Rule evaluation span
        with tracer.start_span("rule_evaluation"):
            result = await rtvm.evaluate(transaction)

        span.set_attribute("evaluation.compliant", result.compliant)
        span.set_attribute("evaluation.rules_evaluated", len(result.rules))

        # Proof generation span
        with tracer.start_span("proof_generation"):
            proof = await generate_proof(result)

        span.set_attribute("proof.id", proof.id)

        return result
```

### Trace Visualization

```
Trace: eval-123 (125ms)
├── evaluate_transaction (125ms)
│   ├── input_validation (2ms)
│   ├── rule_selection (5ms)
│   │   └── jurisdiction_filter (3ms)
│   ├── rule_evaluation (85ms)
│   │   ├── rule: aml/high-value (15ms)
│   │   ├── rule: aml/structuring (20ms)
│   │   ├── rule: sanctions/ofac (35ms)
│   │   │   └── plugin: sanctions_check (30ms)
│   │   │       └── http_request (25ms)
│   │   └── rule: pep/check (15ms)
│   ├── proof_generation (20ms)
│   │   ├── hash_computation (5ms)
│   │   └── signature (15ms)
│   └── response_formatting (3ms)
```

## Alerting

### Alert Rules

```yaml
# alerting-rules.yaml
groups:
  - name: arka-engine-alerts
    rules:
      # High error rate
      - alert: ARKAHighErrorRate
        expr: |
          sum(rate(pact_evaluations_total{status="error"}[5m]))
          /
          sum(rate(pact_evaluations_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
          team: compliance
        annotations:
          summary: "High error rate in ARKA Engine"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # High latency
      - alert: ARKAHighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(pact_evaluation_duration_seconds_bucket[5m])) by (le)) > 0.5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High P99 latency in ARKA Engine"
          description: "P99 latency is {{ $value | humanizeDuration }}"

      # Agent offline
      - alert: ARKAAgentOffline
        expr: |
          time() - pact_agent_heartbeat_timestamp > 120
        for: 2m
        labels:
          severity: warning
          team: operations
        annotations:
          summary: "ARKA Agent offline"
          description: "Agent {{ $labels.agent_id }} has not sent heartbeat"

      # Proof queue backup
      - alert: ARKAProofQueueBackup
        expr: |
          pact_agent_proof_queue_size > 1000
        for: 10m
        labels:
          severity: warning
          team: operations
        annotations:
          summary: "Proof queue backing up"
          description: "Agent {{ $labels.agent_id }} has {{ $value }} queued proofs"

      # Blockchain anchoring failure
      - alert: ARKAAnchoringFailure
        expr: |
          increase(pact_proofs_anchored_total{status="error"}[15m]) > 10
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Blockchain anchoring failures"
          description: "{{ $value }} anchoring failures in last 15 minutes"

      # Rule sync failure
      - alert: ARKARuleSyncFailure
        expr: |
          time() - pact_agent_last_sync_timestamp > 3600
        for: 10m
        labels:
          severity: warning
          team: operations
        annotations:
          summary: "Rule sync failure"
          description: "Agent {{ $labels.agent_id }} hasn't synced in 1 hour"

  - name: arka-slo-alerts
    rules:
      # SLO breach risk
      - alert: ARKASLOBurnRateHigh
        expr: |
          (
            sum(rate(pact_evaluations_total{status="error"}[1h]))
            /
            sum(rate(pact_evaluations_total[1h]))
          ) > 14.4 * 0.001
        for: 1h
        labels:
          severity: critical
          team: sre
        annotations:
          summary: "SLO burn rate too high"
          description: "At current error rate, monthly error budget will be exhausted"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'team']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true

    - match:
        team: compliance
      receiver: 'slack-compliance'

    - match:
        team: platform
      receiver: 'slack-platform'

receivers:
  - name: 'default'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK}'
        channel: '#alerts-pact'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_KEY}'
        severity: critical

  - name: 'slack-compliance'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK}'
        channel: '#compliance-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'slack-platform'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK}'
        channel: '#platform-alerts'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
```

## Health Checks

### Liveness and Readiness

```yaml
# Kubernetes health check configuration
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

### Health Endpoint Response

```json
{
  "status": "healthy",
  "timestamp": "2024-03-15T10:30:00Z",
  "version": "1.2.0",
  "components": {
    "rtvm": {
      "status": "healthy",
      "latency_ms": 5
    },
    "database": {
      "status": "healthy",
      "latency_ms": 10
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 2
    },
    "control_plane": {
      "status": "healthy",
      "latency_ms": 25
    }
  },
  "metrics": {
    "active_evaluations": 45,
    "rules_loaded": 1250,
    "proof_queue_size": 15
  }
}
```

### Deep Health Check

```python
from pact.health import HealthChecker, ComponentCheck

class ARKAHealthChecker(HealthChecker):
    def __init__(self):
        self.checks = [
            ComponentCheck("rtvm", self.check_rtvm),
            ComponentCheck("database", self.check_database),
            ComponentCheck("redis", self.check_redis),
            ComponentCheck("control_plane", self.check_control_plane),
            ComponentCheck("blockchain", self.check_blockchain),
        ]

    async def check_rtvm(self) -> dict:
        """Test RTVM with sample evaluation."""
        start = time.time()
        try:
            result = await self.rtvm.evaluate_test()
            return {
                "status": "healthy",
                "latency_ms": (time.time() - start) * 1000
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "error": str(e)
            }

    async def check_database(self) -> dict:
        """Test database connectivity."""
        start = time.time()
        try:
            await self.db.execute("SELECT 1")
            return {
                "status": "healthy",
                "latency_ms": (time.time() - start) * 1000
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "error": str(e)
            }

    async def deep_check(self) -> dict:
        """Run comprehensive health check."""
        results = {}
        overall_healthy = True

        for check in self.checks:
            result = await check.execute()
            results[check.name] = result
            if result["status"] != "healthy":
                overall_healthy = False

        return {
            "status": "healthy" if overall_healthy else "unhealthy",
            "timestamp": datetime.utcnow().isoformat(),
            "components": results
        }
```

## Audit Logging

### Audit Log Format

```json
{
  "timestamp": "2024-03-15T10:30:00.123Z",
  "event_type": "evaluation.completed",
  "actor": {
    "type": "service",
    "id": "payment-service",
    "ip": "10.0.1.50"
  },
  "resource": {
    "type": "evaluation",
    "id": "eval-789"
  },
  "action": "evaluate",
  "outcome": "success",
  "details": {
    "transaction_id": "txn-123",
    "compliant": true,
    "rules_evaluated": 15,
    "proof_id": "proof-xyz"
  },
  "metadata": {
    "trace_id": "abc123",
    "request_id": "req-456"
  }
}
```

### Audit Log Storage

```yaml
# audit-log-config.yaml
audit:
  enabled: true

  # Log destinations
  destinations:
    - type: elasticsearch
      endpoint: https://audit-es:9200
      index: arka-audit
      retention: 7y

    - type: s3
      bucket: arka-audit-logs
      region: us-east-1
      prefix: audit/
      encryption: AES256

  # Events to capture
  events:
    - evaluation.completed
    - evaluation.failed
    - rule.created
    - rule.updated
    - rule.deleted
    - proposal.submitted
    - proposal.approved
    - proposal.rejected
    - proof.anchored
    - agent.registered
    - agent.offline

  # Data retention
  retention:
    hot: 30d
    warm: 90d
    cold: 365d
    delete: 7y
```

## Best Practices

### Monitoring Strategy

1. **USE Method**: Monitor Utilization, Saturation, Errors
2. **RED Method**: Monitor Rate, Errors, Duration for services
3. **Golden Signals**: Focus on latency, traffic, errors, saturation

### Alert Design

1. **Actionable**: Every alert should have a clear response
2. **Meaningful**: Avoid alert fatigue with relevant thresholds
3. **Documented**: Include runbook links in alert annotations
4. **Tiered**: Use severity levels appropriately

### Log Management

1. **Structured**: Use JSON format for machine parsing
2. **Contextual**: Include trace IDs and correlation IDs
3. **Sampled**: Sample high-volume logs to control costs
4. **Retained**: Follow compliance requirements for retention

## Related Documentation

- [Deployment Guide](./deployment-guide.md)
- [Scaling Guide](./scaling.md)
- [Security Model](../security/security-model.md)
- [Troubleshooting](./troubleshooting.md)
