# Scaling Runbook

## Overview

This runbook covers scaling procedures for the ARKA Engine platform to handle varying loads.

## Auto-Scaling Configuration

### Current HPA Settings

| Service | Min | Max | CPU Target | Memory Target |
|---------|-----|-----|------------|---------------|
| License Authority | 2 | 10 | 70% | 80% |
| Usage Metering | 3 | 20 | 70% | 80% |
| Policy Engine | 2 | 10 | 70% | - |
| Attestation Service | 2 | 8 | 70% | - |

### View Current State

```bash
# Check all HPAs
kubectl -n arka-system get hpa

# Detailed HPA status
kubectl -n arka-system describe hpa usage-metering-hpa

# Check current resource usage
kubectl -n arka-system top pods
```

---

## Manual Scaling

### Scale Up (Immediate Need)

```bash
# Scale single deployment
kubectl -n arka-system scale deployment/usage-metering --replicas=10

# Scale all ARKA services
kubectl -n arka-system scale deployment -l app.kubernetes.io/name=arka-engine --replicas=5

# Verify scaling
kubectl -n arka-system get pods -w
```

### Scale Down

```bash
# Scale down to normal
kubectl -n arka-system scale deployment/usage-metering --replicas=3

# Let HPA manage (set to HPA minimum)
kubectl -n arka-system scale deployment/usage-metering --replicas=3
```

---

## Adjusting HPA Limits

### Increase Maximum Replicas

```bash
# For expected high traffic event
kubectl -n arka-system patch hpa usage-metering-hpa -p \
  '{"spec":{"maxReplicas":50}}'

# Or via Helm
helm -n arka-system upgrade arka ./arka-engine \
  --reuse-values \
  --set usageMetering.autoscaling.maxReplicas=50
```

### Adjust Scaling Thresholds

```bash
# Lower CPU threshold for faster scaling
kubectl -n arka-system patch hpa usage-metering-hpa -p \
  '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":50}}}]}}'
```

### Add Custom Metrics

```yaml
# HPA with custom metrics (requests per second)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: usage-metering-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: usage-metering
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

---

## Database Scaling

### Connection Pool Adjustment

```bash
# Check current connections
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "SELECT count(*) FROM pg_stat_activity;"

# Update max connections (requires restart)
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "ALTER SYSTEM SET max_connections = 500;"

# Restart PostgreSQL
kubectl -n arka-system delete pod postgresql-0
```

### Add Read Replicas

```bash
# Scale PostgreSQL replicas
helm -n arka-system upgrade postgresql bitnami/postgresql \
  --reuse-values \
  --set readReplicas.replicaCount=3
```

### PostgreSQL Performance Tuning

```sql
-- Check for slow queries
SELECT query, calls, mean_time, total_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_usage_events_tenant_date
ON usage_events(tenant_id, created_at);

-- Vacuum and analyze
VACUUM ANALYZE usage_events;
```

---

## Redis Scaling

### Add Redis Replicas

```bash
# Scale replicas
helm -n arka-system upgrade redis bitnami/redis \
  --reuse-values \
  --set replica.replicaCount=3

# Verify replication
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli info replication
```

### Redis Cluster Mode

For very high throughput, migrate to Redis Cluster:

```bash
# Deploy Redis Cluster
helm -n arka-system install redis-cluster bitnami/redis-cluster \
  --set cluster.nodes=6 \
  --set cluster.replicas=1

# Update application connection strings
```

### Memory Optimization

```bash
# Check memory usage
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli info memory

# Set memory limit
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli CONFIG SET maxmemory 2gb

# Set eviction policy
kubectl -n arka-system exec -it sts/redis-master-0 -- redis-cli CONFIG SET maxmemory-policy allkeys-lru
```

---

## Preparing for High Traffic Events

### 1. Pre-Scale (1 Day Before)

```bash
# Scale up services
kubectl -n arka-system scale deployment/usage-metering --replicas=10
kubectl -n arka-system scale deployment/license-authority --replicas=5

# Increase HPA limits
helm -n arka-system upgrade arka ./arka-engine \
  --reuse-values \
  --set usageMetering.autoscaling.maxReplicas=50 \
  --set licenseAuthority.autoscaling.maxReplicas=30

# Scale database connections
kubectl -n arka-system exec -it sts/postgresql-0 -- psql -U arka -c \
  "ALTER SYSTEM SET max_connections = 500;"
kubectl -n arka-system delete pod postgresql-0
```

### 2. Monitor During Event

```bash
# Watch scaling activity
watch kubectl -n arka-system get hpa,pods

# Monitor resource usage
kubectl -n arka-system top pods --containers

# Check error rates
# Open Grafana: Production Dashboard
```

### 3. Post-Event Scale Down

```bash
# Wait for traffic to normalize (usually 1-2 hours)

# Let HPA manage scale down naturally
# Or force scale down
kubectl -n arka-system scale deployment/usage-metering --replicas=3

# Restore HPA limits
helm -n arka-system upgrade arka ./arka-engine \
  --reuse-values \
  --set usageMetering.autoscaling.maxReplicas=20
```

---

## Capacity Planning

### Current Capacity Estimates

| Service | Per Pod Capacity | Current Pods | Total Capacity |
|---------|-----------------|--------------|----------------|
| Usage Metering | 1,000 req/s | 3 | 3,000 req/s |
| License Authority | 500 req/s | 2 | 1,000 req/s |
| Policy Engine | 200 req/s | 2 | 400 req/s |

### Capacity Testing

```bash
# Run load test
k6 run --vus 100 --duration 5m scripts/load-test.js

# Monitor during test
kubectl -n arka-system top pods
kubectl -n arka-system get hpa -w
```

### Scaling Formula

```
Required Pods = (Expected RPS / Per Pod Capacity) * 1.3 (headroom)

Example:
- Expected: 5,000 req/s
- Per Pod: 1,000 req/s
- Required: (5000 / 1000) * 1.3 = 6.5 → 7 pods
```

---

## Vertical Scaling (Resource Limits)

### Increase Pod Resources

```bash
# Check current limits
kubectl -n arka-system get deployment usage-metering -o yaml | grep -A10 resources

# Update resources
kubectl -n arka-system set resources deployment/usage-metering \
  --containers=usage-metering \
  --requests=cpu=1,memory=2Gi \
  --limits=cpu=2,memory=4Gi

# Via Helm
helm -n arka-system upgrade arka ./arka-engine \
  --reuse-values \
  --set usageMetering.resources.requests.cpu=1 \
  --set usageMetering.resources.requests.memory=2Gi \
  --set usageMetering.resources.limits.cpu=2 \
  --set usageMetering.resources.limits.memory=4Gi
```

### Node Pool Scaling

```bash
# Check node capacity
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory

# Scale node pool (EKS)
aws eks update-nodegroup-config \
  --cluster-name arka-cluster \
  --nodegroup-name arka-workers \
  --scaling-config minSize=3,maxSize=20,desiredSize=5

# Scale node pool (GKE)
gcloud container clusters resize arka-cluster \
  --node-pool arka-workers \
  --num-nodes 5
```

---

## Troubleshooting Scaling Issues

### HPA Not Scaling

```bash
# Check HPA status
kubectl -n arka-system describe hpa usage-metering-hpa

# Common issues:
# 1. Metrics server not running
kubectl -n kube-system get pods -l k8s-app=metrics-server

# 2. Resource requests not set
kubectl -n arka-system get deployment usage-metering -o yaml | grep -A5 resources

# 3. Metrics not available
kubectl -n arka-system get --raw /apis/metrics.k8s.io/v1beta1/namespaces/arka-system/pods
```

### Pods Not Starting

```bash
# Check pending pods
kubectl -n arka-system get pods --field-selector=status.phase=Pending

# Check events
kubectl -n arka-system describe pod <pending-pod-name>

# Common issues:
# 1. Insufficient resources - scale node pool
# 2. PVC not bound - check storage class
# 3. Image pull issues - check registry credentials
```

### Scale Down Not Working

```bash
# Check PDB (Pod Disruption Budget)
kubectl -n arka-system get pdb

# PDB might be blocking scale down
# Temporarily adjust if needed
kubectl -n arka-system patch pdb usage-metering-pdb -p '{"spec":{"minAvailable":1}}'
```
