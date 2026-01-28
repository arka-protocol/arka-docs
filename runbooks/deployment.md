# Deployment Runbook

## Overview

This runbook covers deployment procedures for the ARKA Engine platform.

## Pre-Deployment Checklist

- [ ] All tests passing in CI
- [ ] Code review approved
- [ ] Security scan completed
- [ ] Database migrations reviewed
- [ ] Feature flags configured (if applicable)
- [ ] Rollback plan documented
- [ ] On-call engineer notified

---

## Standard Deployment

### 1. Prepare for Deployment

```bash
# Pull latest changes
git checkout main
git pull origin main

# Check CI status
gh run list --limit 5

# Verify image exists
aws ecr describe-images \
  --repository-name pact/license-authority \
  --image-ids imageTag=v1.2.3
```

### 2. Deploy to Staging

```bash
# Update staging environment
cd deploy/helm/arka-engine

# Dry run
helm upgrade arka . \
  --namespace arka-staging \
  -f values.yaml \
  -f values-staging.yaml \
  --set image.tag=v1.2.3 \
  --dry-run

# Apply
helm upgrade arka . \
  --namespace arka-staging \
  -f values.yaml \
  -f values-staging.yaml \
  --set image.tag=v1.2.3 \
  --wait \
  --timeout 10m
```

### 3. Verify Staging

```bash
# Check deployment status
kubectl -n arka-staging rollout status deployment/license-authority

# Run smoke tests
./scripts/smoke-test.sh --env staging

# Check logs for errors
kubectl -n arka-staging logs -l app.kubernetes.io/name=arka-engine --tail=100 | grep error

# Verify metrics
curl -s https://staging-api.arka.io/metrics | head -20
```

### 4. Deploy to Production

```bash
# Deploy with canary (10% traffic)
helm upgrade arka . \
  --namespace arka-system \
  -f values.yaml \
  -f values-production.yaml \
  --set image.tag=v1.2.3 \
  --set licenseAuthority.replicaCount=1 \
  --wait \
  --timeout 10m

# Monitor canary for 10 minutes
watch kubectl -n arka-system get pods -l app.kubernetes.io/component=license-authority

# If healthy, complete rollout
helm upgrade arka . \
  --namespace arka-system \
  -f values.yaml \
  -f values-production.yaml \
  --set image.tag=v1.2.3 \
  --wait \
  --timeout 10m
```

### 5. Post-Deployment Verification

```bash
# Verify all pods running
kubectl -n arka-system get pods

# Check HPA status
kubectl -n arka-system get hpa

# Run smoke tests
./scripts/smoke-test.sh --env production

# Monitor error rate
# Open Grafana dashboard: Production Overview
```

---

## Database Migration

### Before Migration

```bash
# Backup database
kubectl -n arka-system exec sts/postgresql-0 -- \
  pg_dump -U pact pact | gzip > backup-$(date +%Y%m%d-%H%M%S).sql.gz

# Upload to S3
aws s3 cp backup-*.sql.gz s3://arka-backups/database/

# Review migration
cat migrations/20240101_add_new_column.sql
```

### Run Migration

```bash
# For simple migrations (no downtime)
kubectl -n arka-system exec -it deploy/license-authority -- \
  npm run db:migrate

# For complex migrations (with downtime)
# 1. Enable maintenance mode
kubectl -n arka-system scale deployment --all --replicas=0

# 2. Run migration
kubectl -n arka-system exec -it sts/postgresql-0 -- \
  psql -U arka -d pact -f /migrations/20240101_complex_migration.sql

# 3. Verify migration
kubectl -n arka-system exec -it sts/postgresql-0 -- \
  psql -U arka -d pact -c "\d+ users"

# 4. Resume services
kubectl -n arka-system scale deployment/license-authority --replicas=2
kubectl -n arka-system scale deployment/usage-metering --replicas=3
kubectl -n arka-system scale deployment/policy-engine --replicas=2
kubectl -n arka-system scale deployment/attestation-service --replicas=2
```

### Rollback Migration

```bash
# Apply down migration
kubectl -n arka-system exec -it sts/postgresql-0 -- \
  psql -U arka -d pact -f /migrations/20240101_add_new_column_down.sql

# Or restore from backup
gunzip -c backup-20240101-120000.sql.gz | \
  kubectl -n arka-system exec -i sts/postgresql-0 -- \
  psql -U arka -d pact
```

---

## Rollback Procedures

### Quick Rollback (Kubernetes)

```bash
# View rollout history
kubectl -n arka-system rollout history deployment/license-authority

# Rollback to previous version
kubectl -n arka-system rollout undo deployment/license-authority

# Rollback to specific revision
kubectl -n arka-system rollout undo deployment/license-authority --to-revision=5

# Verify rollback
kubectl -n arka-system rollout status deployment/license-authority
```

### Helm Rollback

```bash
# List releases
helm -n arka-system history pact

# Rollback to previous release
helm -n arka-system rollback pact

# Rollback to specific revision
helm -n arka-system rollback pact 5

# Verify
helm -n arka-system status pact
```

### Full Rollback (All Services)

```bash
# Find last stable revision
helm -n arka-system history pact

# Rollback entire stack
helm -n arka-system rollback pact <revision>

# Verify all services
kubectl -n arka-system get pods
kubectl -n arka-system get hpa

# Run smoke tests
./scripts/smoke-test.sh --env production
```

---

## Blue-Green Deployment

### Setup Blue-Green

```bash
# Current (blue) is running
kubectl -n arka-system get svc arka-gateway -o yaml | grep selector

# Create green deployment
helm install arka-green ./arka-engine \
  --namespace arka-system \
  -f values-production.yaml \
  --set nameOverride=arka-green \
  --set image.tag=v1.3.0 \
  --wait

# Verify green is healthy
kubectl -n arka-system get pods -l app.kubernetes.io/instance=arka-green
./scripts/smoke-test.sh --target arka-green
```

### Switch Traffic

```bash
# Update gateway to point to green
kubectl -n arka-system patch svc arka-gateway -p \
  '{"spec":{"selector":{"app.kubernetes.io/instance":"arka-green"}}}'

# Monitor for errors
watch kubectl -n arka-system logs -l app.kubernetes.io/instance=arka-green --tail=10 | grep error

# If issues, quick rollback
kubectl -n arka-system patch svc arka-gateway -p \
  '{"spec":{"selector":{"app.kubernetes.io/instance":"arka"}}}'
```

### Cleanup

```bash
# After successful deployment, remove old blue
helm -n arka-system uninstall pact

# Rename green to pact
helm -n arka-system upgrade arka-green ./arka-engine \
  --reuse-values \
  --set nameOverride=pact
```

---

## Canary Deployment

### Deploy Canary (10% traffic)

```bash
# Deploy canary version
kubectl -n arka-system apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: license-authority-canary
  labels:
    app.kubernetes.io/name: arka-engine
    app.kubernetes.io/component: license-authority
    app.kubernetes.io/version: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: license-authority
  template:
    metadata:
      labels:
        app.kubernetes.io/component: license-authority
        version: canary
    spec:
      containers:
      - name: license-authority
        image: pact/license-authority:v1.3.0
        # ... rest of container spec
EOF
```

### Monitor Canary

```bash
# Compare error rates (Prometheus)
# Canary errors
curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{version='canary',status=~'5..'}[5m])"

# Stable errors
curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{version!='canary',status=~'5..'}[5m])"

# Compare latency
curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.99,rate(http_request_duration_seconds_bucket{version='canary'}[5m]))"
```

### Promote or Rollback

```bash
# If canary healthy, promote
kubectl -n arka-system set image deployment/license-authority \
  license-authority=pact/license-authority:v1.3.0

# Cleanup canary
kubectl -n arka-system delete deployment license-authority-canary

# If canary unhealthy, rollback
kubectl -n arka-system delete deployment license-authority-canary
```

---

## Emergency Procedures

### Complete Service Restart

```bash
# Restart all ARKA services
kubectl -n arka-system rollout restart deployment -l app.kubernetes.io/name=arka-engine

# Wait for rollout
kubectl -n arka-system rollout status deployment -l app.kubernetes.io/name=arka-engine
```

### Disable Service (Maintenance Mode)

```bash
# Scale down
kubectl -n arka-system scale deployment license-authority --replicas=0

# Update ingress with maintenance page
kubectl -n arka-system annotate ingress arka-ingress \
  nginx.ingress.kubernetes.io/custom-http-errors="503" \
  nginx.ingress.kubernetes.io/default-backend="maintenance-backend"

# Re-enable
kubectl -n arka-system scale deployment license-authority --replicas=2
kubectl -n arka-system annotate ingress arka-ingress \
  nginx.ingress.kubernetes.io/custom-http-errors- \
  nginx.ingress.kubernetes.io/default-backend-
```

### Force Pod Termination

```bash
# If pod is stuck
kubectl -n arka-system delete pod <pod-name> --force --grace-period=0

# Delete all pods of a deployment
kubectl -n arka-system delete pods -l app.kubernetes.io/component=license-authority
```

---

## Deployment Schedule

| Environment | Deploy Window | Approval Required |
|-------------|---------------|-------------------|
| Development | Anytime | No |
| Staging | Anytime | No |
| Production | Tue-Thu, 10am-4pm | Yes |
| Hotfix | Anytime | Tech Lead |

**Deployment Freeze:**
- Last week of each quarter
- Major holidays
- During active incidents
