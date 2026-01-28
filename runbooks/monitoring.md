# Monitoring Runbook

## Overview

This runbook covers monitoring and alerting for the ARKA Engine platform.

## Dashboards

### Key Dashboards

| Dashboard | URL | Purpose |
|-----------|-----|---------|
| Production Overview | `/d/arka-overview` | System health at a glance |
| Service Detail | `../arka-services` | Per-service metrics |
| Database | `../arka-database` | PostgreSQL metrics |
| Redis | `../arka-redis` | Cache performance |
| API Metrics | `../arka-api` | Request/response metrics |

---

## Key Metrics

### Service Level Indicators (SLIs)

| SLI | Target | Query |
|-----|--------|-------|
| Availability | 99.9% | `1 - (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))` |
| Latency P99 | < 200ms | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| Error Rate | < 0.1% | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |

### Service Health Metrics

```promql
# Request rate by service
sum by (service) (rate(http_requests_total[5m]))

# Error rate by service
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))

# Latency P50, P95, P99
histogram_quantile(0.5, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

### Resource Metrics

```promql
# CPU utilization
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="arka-system"}[5m]))

# Memory usage
sum by (pod) (container_memory_usage_bytes{namespace="arka-system"})

# Network I/O
sum by (pod) (rate(container_network_receive_bytes_total{namespace="arka-system"}[5m]))
sum by (pod) (rate(container_network_transmit_bytes_total{namespace="arka-system"}[5m]))
```

### Database Metrics

```promql
# Active connections
pg_stat_activity_count{datname="arka"}

# Query duration
rate(pg_stat_statements_seconds_total[5m])

# Cache hit ratio
pg_stat_database_blks_hit / (pg_stat_database_blks_hit + pg_stat_database_blks_read)

# Transaction rate
rate(pg_stat_database_xact_commit[5m])
```

### Redis Metrics

```promql
# Hit rate
redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)

# Memory usage
redis_memory_used_bytes

# Connected clients
redis_connected_clients

# Commands per second
rate(redis_commands_processed_total[5m])
```

---

## Alerting Rules

### Critical Alerts (Page Immediately)

```yaml
groups:
  - name: arka-critical
    rules:
      - alert: ServiceDown
        expr: up{job=~"arka-.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          runbook: "https://docs.arka.io/runbooks/incident-response"

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: DatabaseDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"

      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"
```

### Warning Alerts

```yaml
groups:
  - name: arka-warning
    rules:
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency on {{ $labels.service }}"
          description: "P99 latency is {{ $value | humanizeDuration }}"

      - alert: HighCpuUsage
        expr: |
          sum(rate(container_cpu_usage_seconds_total{namespace="arka-system"}[5m])) by (pod)
          / sum(container_spec_cpu_quota{namespace="arka-system"}/container_spec_cpu_period{namespace="arka-system"}) by (pod)
          > 0.8
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.pod }}"

      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{namespace="arka-system"}
          / container_spec_memory_limit_bytes{namespace="arka-system"}
          > 0.85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.pod }}"

      - alert: DiskSpaceLow
        expr: |
          (kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes) < 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on PVC {{ $labels.persistentvolumeclaim }}"

      - alert: DatabaseConnectionPoolExhausted
        expr: pg_stat_activity_count > 0.9 * pg_settings_max_connections
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool nearly exhausted"

      - alert: RedisCacheHitRateLow
        expr: |
          redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) < 0.8
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Redis cache hit rate below 80%"
```

---

## Log Analysis

### Viewing Logs

```bash
# All services
kubectl -n arka-system logs -l app.kubernetes.io/name=arka-engine --tail=100

# Specific service
kubectl -n arka-system logs -l app.kubernetes.io/component=license-authority --tail=100

# Follow logs
kubectl -n arka-system logs -f deployment/usage-metering

# JSON parsing with jq
kubectl -n arka-system logs deployment/usage-metering --tail=100 | \
  jq 'select(.level == "error")'
```

### Common Log Queries (Loki/Kibana)

```logql
# Errors in last hour
{namespace="arka-system"} |= "error" | json | level="error"

# Slow requests (>1s)
{namespace="arka-system"} | json | duration > 1000

# Specific user/tenant
{namespace="arka-system"} | json | tenant_id="abc123"

# Database errors
{namespace="arka-system"} |~ "database|postgres|pg_"

# Authentication failures
{namespace="arka-system"} | json | message=~".*authentication.*fail.*"
```

### Log Aggregation Queries

```logql
# Error count by service
sum by (service) (count_over_time({namespace="arka-system"} | json | level="error" [1h]))

# Request count by endpoint
sum by (path) (count_over_time({namespace="arka-system"} | json | path!="" [1h]))
```

---

## Health Checks

### Manual Health Checks

```bash
# Service health endpoints
kubectl -n arka-system exec -it deploy/license-authority -- curl localhost:3001/health/live
kubectl -n arka-system exec -it deploy/license-authority -- curl localhost:3001/health/ready

# Database health
kubectl -n arka-system exec -it sts/postgresql-0 -- pg_isready -U pact

# Redis health
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli ping

# Full health check script
./scripts/health-check.sh
```

### Health Check Endpoints

| Service | Liveness | Readiness |
|---------|----------|-----------|
| License Authority | `:3001/health/live` | `:3001/health/ready` |
| Usage Metering | `:3002/health/live` | `:3002/health/ready` |
| Policy Engine | `:3003/health/live` | `:3003/health/ready` |
| Attestation | `:3004/health/live` | `:3004/health/ready` |

---

## Performance Analysis

### Identifying Slow Endpoints

```promql
# Top 10 slowest endpoints
topk(10, histogram_quantile(0.99, sum by (path, le) (rate(http_request_duration_seconds_bucket[1h]))))
```

### Database Query Analysis

```sql
-- Slow queries
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- Most called queries
SELECT query, calls, total_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- Queries with high variance
SELECT query, stddev_time, mean_time, calls
FROM pg_stat_statements
WHERE calls > 100
ORDER BY stddev_time DESC
LIMIT 20;
```

### Tracing Analysis

```bash
# View traces in Jaeger
open https://jaeger.pact.internal/search?service=license-authority

# Find slow traces
# Filter: minDuration=1s

# Find error traces
# Filter: tag=error:true
```

---

## Capacity Monitoring

### Current Utilization

```bash
# Cluster-wide resource usage
kubectl top nodes

# Namespace resource usage
kubectl -n arka-system top pods --containers

# PVC usage
kubectl -n arka-system exec -it sts/postgresql-0 -- df -h /bitnami/postgresql
```

### Capacity Alerts

```yaml
- alert: ClusterCapacityLow
  expr: |
    sum(kube_pod_container_resource_requests{resource="cpu",namespace="arka-system"})
    / sum(kube_node_status_allocatable{resource="cpu"})
    > 0.8
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Cluster CPU capacity is running low"

- alert: PVCCapacityLow
  expr: |
    kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.8
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "PVC {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
```

---

## On-Call Procedures

### Daily Checks

1. Review overnight alerts
2. Check error rate trends
3. Review slow query logs
4. Check disk space usage
5. Verify backup completion

### Weekly Checks

1. Review capacity trends
2. Analyze slow endpoint patterns
3. Review database query performance
4. Check for security advisories
5. Update runbooks if needed

### Dashboard Checklist

| Check | Location | Threshold |
|-------|----------|-----------|
| Error Rate | Overview | < 0.1% |
| P99 Latency | API Metrics | < 200ms |
| CPU Usage | Resources | < 70% |
| Memory Usage | Resources | < 80% |
| Disk Usage | Storage | < 80% |
| DB Connections | Database | < 80% of max |
| Cache Hit Rate | Redis | > 90% |
