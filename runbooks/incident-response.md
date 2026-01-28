# Incident Response Runbook

## Overview

This runbook provides step-by-step procedures for responding to incidents in the ARKA Engine platform.

## Severity Levels

| Level | Description | Response Time | Escalation |
|-------|-------------|---------------|------------|
| SEV1 | Complete system outage | 15 minutes | Immediate |
| SEV2 | Major feature unavailable | 30 minutes | Within 1 hour |
| SEV3 | Minor feature impacted | 2 hours | Within 4 hours |
| SEV4 | Low impact issue | Next business day | As needed |

## Initial Response

### 1. Acknowledge the Incident

```bash
# Check monitoring dashboards
open https://grafana.pact.internal/d/arka-overview

# Check Slack alerts channel
# #arka-alerts
```

### 2. Initial Assessment

**Questions to answer:**
- What services are affected?
- What is the customer impact?
- When did it start?
- Any recent deployments?

```bash
# Check recent deployments
kubectl -n arka-system get deployments -o wide

# Check pod status
kubectl -n arka-system get pods

# Check recent events
kubectl -n arka-system get events --sort-by='.lastTimestamp' | tail -20
```

### 3. Communication

**For SEV1/SEV2:**
1. Create incident channel: `#incident-YYYY-MM-DD-brief-name`
2. Post initial update in #arka-incidents
3. Notify stakeholders via PagerDuty

**Template:**
```
🚨 INCIDENT: [Brief Description]
Severity: SEVx
Impact: [Customer impact]
Status: Investigating
Lead: @[name]
```

---

## Common Scenarios

### Service Not Responding

**Symptoms:**
- Health checks failing
- 5xx errors in logs
- Requests timing out

**Diagnosis:**

```bash
# Check service health
kubectl -n arka-system exec -it deploy/license-authority -- curl localhost:3001/health

# Check logs
kubectl -n arka-system logs deploy/license-authority --tail=100

# Check resource usage
kubectl -n arka-system top pods

# Check for OOMKilled
kubectl -n arka-system describe pod -l app.kubernetes.io/component=license-authority | grep -A5 "Last State"
```

**Resolution:**

```bash
# Restart the service (rolling restart)
kubectl -n arka-system rollout restart deployment/license-authority

# If OOMKilled, scale up resources
kubectl -n arka-system set resources deployment/license-authority \
  --containers=license-authority \
  --limits=memory=2Gi
```

---

### Database Connection Issues

**Symptoms:**
- "Connection refused" errors
- "Too many connections" errors
- Slow queries

**Diagnosis:**

```bash
# Check PostgreSQL status
kubectl -n arka-system exec -it sts/postgresql-0 -- pg_isready

# Check connection count
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT count(*) FROM pg_stat_activity;"

# Check for blocking queries
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT pid, now() - pg_stat_activity.query_start AS duration, query
   FROM pg_stat_activity
   WHERE state = 'active' AND now() - pg_stat_activity.query_start > interval '5 seconds';"
```

**Resolution:**

```bash
# Kill blocking query (get PID from above)
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT pg_terminate_backend(PID);"

# If connection pool exhausted, restart services
kubectl -n arka-system rollout restart deployment --selector=app.kubernetes.io/name=arka-engine
```

---

### Redis Issues

**Symptoms:**
- Cache misses increasing
- "LOADING" errors
- Connection timeouts

**Diagnosis:**

```bash
# Check Redis health
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli ping

# Check memory usage
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli info memory

# Check connected clients
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli info clients

# Check replication status
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli info replication
```

**Resolution:**

```bash
# Clear cache if corrupted
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli FLUSHDB

# If slave out of sync
kubectl -n arka-system delete pod redis-replicas-0
```

---

### High Error Rate

**Symptoms:**
- Error rate > 1%
- Specific error codes increasing

**Diagnosis:**

```bash
# Check error distribution in logs
kubectl -n arka-system logs deploy/usage-metering --tail=1000 | \
  grep '"level":"error"' | \
  jq -r '.error' | sort | uniq -c | sort -rn

# Check error rate in Prometheus
curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{status=~'5..'}[5m])"
```

**Resolution:**

1. Identify the error pattern
2. Check for recent code changes
3. Roll back if needed:

```bash
# Get previous revision
kubectl -n arka-system rollout history deployment/usage-metering

# Rollback
kubectl -n arka-system rollout undo deployment/usage-metering

# Rollback to specific revision
kubectl -n arka-system rollout undo deployment/usage-metering --to-revision=2
```

---

### High Latency

**Symptoms:**
- P99 latency > SLA threshold
- Slow API responses

**Diagnosis:**

```bash
# Check latency metrics
curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.99,rate(http_request_duration_seconds_bucket[5m]))"

# Check for slow database queries
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT query, calls, mean_time, total_time
   FROM pg_stat_statements
   ORDER BY mean_time DESC LIMIT 10;"

# Check CPU throttling
kubectl -n arka-system top pods
```

**Resolution:**

```bash
# Scale up if CPU bound
kubectl -n arka-system scale deployment/usage-metering --replicas=5

# Or increase HPA limits
kubectl -n arka-system patch hpa usage-metering-hpa -p '{"spec":{"maxReplicas":30}}'
```

---

### Disk Space Issues

**Symptoms:**
- PVC at capacity
- Write failures

**Diagnosis:**

```bash
# Check PVC usage
kubectl -n arka-system exec -it sts/postgresql-0 -- df -h /bitnami/postgresql

# Check for large tables
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
   FROM pg_catalog.pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;"
```

**Resolution:**

```bash
# Vacuum to reclaim space
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c "VACUUM FULL;"

# Expand PVC (if storage class supports it)
kubectl -n arka-system patch pvc data-postgresql-0 -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

---

## Post-Incident

### 1. Verify Recovery

```bash
# Run smoke tests
./scripts/smoke-test.sh

# Check all services healthy
kubectl -n arka-system get pods
kubectl -n arka-system get hpa
```

### 2. Update Status

```
✅ RESOLVED: [Brief Description]
Duration: X hours Y minutes
Impact: [Summary of customer impact]
Root Cause: [Brief description]
Follow-up: [JIRA ticket number]
```

### 3. Schedule Post-Mortem

- Schedule within 48 hours for SEV1/SEV2
- Use blameless post-mortem template
- Document timeline, root cause, and action items

---

## Escalation Contacts

| Role | Primary | Secondary |
|------|---------|-----------|
| On-Call Engineer | PagerDuty Schedule | - |
| Platform Lead | @platform-lead | @platform-backup |
| Database Admin | @dba-oncall | @dba-backup |
| Security | @security-oncall | @security-backup |

---

## Useful Commands Reference

```bash
# Quick health check
kubectl -n arka-system get pods,svc,hpa,pdb

# Describe all issues
kubectl -n arka-system describe pods | grep -A10 "Events:"

# Get all logs from namespace
kubectl -n arka-system logs --all-containers --prefix --timestamps --tail=100 -l app.kubernetes.io/name=arka-engine

# Port forward for local debugging
kubectl -n arka-system port-forward svc/license-authority 3001:3001

# Execute command in pod
kubectl -n arka-system exec -it deploy/license-authority -- /bin/sh
```
