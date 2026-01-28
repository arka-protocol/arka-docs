# Deployment Guide

This guide covers deploying ARKA Engine to production environments with high availability, security, and scalability.

## Table of Contents

- [Deployment Overview](#deployment-overview)
- [Prerequisites](#prerequisites)
- [Infrastructure Requirements](#infrastructure-requirements)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Configuration](#configuration)
- [Security Hardening](#security-hardening)
- [High Availability](#high-availability)
- [Monitoring Setup](#monitoring-setup)
- [Backup & Recovery](#backup--recovery)
- [Deployment Checklist](#deployment-checklist)

---

## Deployment Overview

### Architecture for Production

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PRODUCTION ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         INGRESS LAYER                                   │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │ │
│  │  │     CDN      │  │   WAF/DDoS   │  │    Load      │                  │ │
│  │  │ (CloudFlare) │──│  Protection  │──│  Balancer    │                  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       APPLICATION LAYER                                 │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    API Gateway (Kong)                            │   │ │
│  │  │  • Rate limiting  • Auth  • Routing  • TLS termination          │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                  │                                      │ │
│  │         ┌────────────────────────┼────────────────────────┐            │ │
│  │         │                        │                        │            │ │
│  │         ▼                        ▼                        ▼            │ │
│  │  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐    │ │
│  │  │ ARKA Core   │          │ ARKA Core   │          │ ARKA Core   │    │ │
│  │  │  (Pod 1)    │          │  (Pod 2)    │          │  (Pod 3)    │    │ │
│  │  └─────────────┘          └─────────────┘          └─────────────┘    │ │
│  │         │                        │                        │            │ │
│  └─────────┼────────────────────────┼────────────────────────┼────────────┘ │
│            │                        │                        │              │
│            └────────────────────────┼────────────────────────┘              │
│                                     ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       EXECUTION LAYER                                   │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    RTVM Cluster                                  │   │ │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │ │
│  │  │  │ RTVM-1  │  │ RTVM-2  │  │ RTVM-3  │  │ RTVM-N  │            │   │ │
│  │  │  │ (8 CPU) │  │ (8 CPU) │  │ (8 CPU) │  │ (8 CPU) │            │   │ │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘            │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    Plugin Host Cluster                           │   │ │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                          │   │ │
│  │  │  │ Plugin  │  │ Plugin  │  │ Plugin  │                          │   │ │
│  │  │  │ Host-1  │  │ Host-2  │  │ Host-3  │                          │   │ │
│  │  │  └─────────┘  └─────────┘  └─────────┘                          │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│                                     ▼                                        │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         DATA LAYER                                      │ │
│  │                                                                         │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │ │
│  │  │   PostgreSQL    │  │     Redis       │  │    Object       │        │ │
│  │  │   (Primary +    │  │    Cluster      │  │    Storage      │        │ │
│  │  │   2 Replicas)   │  │   (6 nodes)     │  │    (S3)         │        │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │ │
│  │                                                                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

### Required Tools

```bash
# Kubernetes CLI
kubectl version --client  # 1.25+

# Helm
helm version  # 3.10+

# AWS CLI (if using AWS)
aws --version  # 2.0+

# Terraform (optional, for infrastructure)
terraform version  # 1.5+
```

### Required Credentials

- Kubernetes cluster access
- Container registry credentials
- Database credentials
- Cloud provider credentials (if applicable)
- TLS certificates

---

## Infrastructure Requirements

### Compute Resources

| Component | Replicas | CPU | Memory | Storage |
|-----------|----------|-----|--------|---------|
| ARKA Core | 3+ | 4 cores | 8 GB | 50 GB |
| RTVM | 5+ | 8 cores | 16 GB | 20 GB |
| Plugin Host | 3+ | 4 cores | 8 GB | 20 GB |
| PostgreSQL | 3 | 8 cores | 32 GB | 500 GB SSD |
| Redis | 6 | 4 cores | 16 GB | 50 GB SSD |

### Network Requirements

| Port | Service | Protocol |
|------|---------|----------|
| 443 | API Gateway | HTTPS |
| 5432 | PostgreSQL | TCP (internal) |
| 6379 | Redis | TCP (internal) |
| 8080 | ARKA Core | HTTP (internal) |
| 8081 | RTVM | gRPC (internal) |
| 9090 | Prometheus | HTTP (internal) |

---

## Kubernetes Deployment

### 1. Create Namespace

```bash
kubectl create namespace arka-system
kubectl label namespace arka-system istio-injection=enabled  # If using Istio
```

### 2. Configure Secrets

```bash
# Database credentials
kubectl create secret generic arka-db-credentials \
  --namespace arka-system \
  --from-literal=username=pact \
  --from-literal=password='<secure-password>'

# Redis credentials
kubectl create secret generic arka-redis-credentials \
  --namespace arka-system \
  --from-literal=password='<secure-password>'

# API secrets
kubectl create secret generic arka-api-secrets \
  --namespace arka-system \
  --from-literal=jwt-secret='<jwt-secret>' \
  --from-literal=encryption-key='<encryption-key>'

# TLS certificates
kubectl create secret tls arka-tls \
  --namespace arka-system \
  --cert=./certs/tls.crt \
  --key=./certs/tls.key
```

### 3. Helm Values Configuration

Create `production-values.yaml`:

```yaml
# production-values.yaml
global:
  environment: production
  imageTag: "1.0.0"
  imagePullPolicy: Always
  imagePullSecrets:
    - name: registry-credentials

# ═══════════════════════════════════════════════════════════════
# ARKA CORE
# ═══════════════════════════════════════════════════════════════
core:
  replicas: 3

  image:
    repository: arka-engine/core
    tag: "1.0.0"

  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
    limits:
      cpu: "4"
      memory: "8Gi"

  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilization: 70

  podDisruptionBudget:
    minAvailable: 2

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: arka-core
          topologyKey: kubernetes.io/hostname

  env:
    - name: ARKA_ENVIRONMENT
      value: "production"
    - name: ARKA_LOG_LEVEL
      value: "info"
    - name: ARKA_LOG_FORMAT
      value: "json"

  envFrom:
    - secretRef:
        name: arka-db-credentials
    - secretRef:
        name: arka-api-secrets

# ═══════════════════════════════════════════════════════════════
# RTVM
# ═══════════════════════════════════════════════════════════════
rtvm:
  replicas: 5

  image:
    repository: arka-engine/rtvm
    tag: "1.0.0"

  resources:
    requests:
      cpu: "4"
      memory: "8Gi"
    limits:
      cpu: "8"
      memory: "16Gi"

  autoscaling:
    enabled: true
    minReplicas: 5
    maxReplicas: 20
    targetCPUUtilization: 60

  config:
    maxStackDepth: 1000
    maxHeapSize: "8GB"
    maxInstructions: 10000000
    maxExecutionTime: "30s"
    workerThreads: 8

# ═══════════════════════════════════════════════════════════════
# POSTGRESQL
# ═══════════════════════════════════════════════════════════════
postgresql:
  enabled: true

  architecture: replication

  auth:
    existingSecret: arka-db-credentials
    secretKeys:
      adminPasswordKey: password
      userPasswordKey: password
    database: pact

  primary:
    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
      limits:
        cpu: "8"
        memory: "32Gi"

    persistence:
      enabled: true
      size: 500Gi
      storageClass: gp3

    configuration: |
      max_connections = 500
      shared_buffers = 8GB
      effective_cache_size = 24GB
      maintenance_work_mem = 2GB
      checkpoint_completion_target = 0.9
      wal_buffers = 64MB
      default_statistics_target = 100
      random_page_cost = 1.1
      effective_io_concurrency = 200

  readReplicas:
    replicaCount: 2
    resources:
      requests:
        cpu: "2"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
    persistence:
      enabled: true
      size: 500Gi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

# ═══════════════════════════════════════════════════════════════
# REDIS
# ═══════════════════════════════════════════════════════════════
redis:
  enabled: true

  architecture: replication

  auth:
    enabled: true
    existingSecret: arka-redis-credentials
    existingSecretPasswordKey: password

  master:
    resources:
      requests:
        cpu: "2"
        memory: "8Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
    persistence:
      enabled: true
      size: 50Gi

  replica:
    replicaCount: 2
    resources:
      requests:
        cpu: "1"
        memory: "4Gi"
      limits:
        cpu: "2"
        memory: "8Gi"

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

# ═══════════════════════════════════════════════════════════════
# INGRESS
# ═══════════════════════════════════════════════════════════════
ingress:
  enabled: true
  className: nginx

  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"

  hosts:
    - host: api.arka-engine.example.com
      paths:
        - path: /
          pathType: Prefix

  tls:
    - secretName: arka-tls
      hosts:
        - api.arka-engine.example.com

# ═══════════════════════════════════════════════════════════════
# MONITORING
# ═══════════════════════════════════════════════════════════════
monitoring:
  enabled: true

  prometheus:
    enabled: true
    serviceMonitor:
      enabled: true

  grafana:
    enabled: true
    persistence:
      enabled: true
      size: 10Gi

    dashboards:
      default:
        arka-overview:
          gnetId: 12345
          revision: 1
          datasource: Prometheus

# ═══════════════════════════════════════════════════════════════
# SECURITY
# ═══════════════════════════════════════════════════════════════
security:
  podSecurityPolicy:
    enabled: true

  networkPolicy:
    enabled: true

  serviceAccount:
    create: true
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:ro../arka-role
```

### 4. Deploy with Helm

```bash
# Add ARKA Helm repository
helm repo add pact https://charts.arka-engine.io
helm repo update

# Deploy
helm upgrade --install pact pact/arka-engine \
  --namespace arka-system \
  --values production-values.yaml \
  --wait \
  --timeout 10m

# Verify deployment
kubectl get pods -n arka-system
kubectl get services -n arka-system
```

### 5. Run Database Migrations

```bash
# Get a core pod name
CORE_POD=$(kubectl get pods -n arka-system -l app=arka-core -o jsonpath='{.items[0].metadata.name}')

# Run migrations
kubectl exec -n arka-system $CORE_POD -- arka-cli db migrate --confirm

# Verify
kubectl exec -n arka-system $CORE_POD -- arka-cli db status
```

---

## Configuration

### Environment-Specific Configuration

```yaml
# config/production.yaml
environment: production
log_level: info
log_format: json

api:
  host: 0.0.0.0
  port: 8080
  read_timeout: 30s
  write_timeout: 30s
  max_request_size: 10MB

  cors:
    allowed_origins:
      - "https://dashboard.arka-engine.example.com"
    allowed_methods:
      - GET
      - POST
      - PUT
      - DELETE
    allowed_headers:
      - Authorization
      - Content-Type

  rate_limiting:
    enabled: true
    requests_per_minute: 1000
    burst: 100

database:
  host: ${DATABASE_HOST}
  port: 5432
  name: pact
  user: ${DATABASE_USER}
  password: ${DATABASE_PASSWORD}
  ssl_mode: require
  max_connections: 100
  min_connections: 10
  connection_timeout: 10s

redis:
  host: ${REDIS_HOST}
  port: 6379
  password: ${REDIS_PASSWORD}
  database: 0
  max_connections: 50

rtvm:
  host: arka-rtvm
  port: 8081
  timeout: 30s
  max_concurrent: 1000

blockchain:
  enabled: true
  networks:
    ethereum:
      enabled: true
      rpc_url: ${ETHEREUM_RPC_URL}
      contract_address: ${ETHEREUM_CONTRACT_ADDRESS}
      confirmation_blocks: 12
    polygon:
      enabled: true
      rpc_url: ${POLYGON_RPC_URL}
      contract_address: ${POLYGON_CONTRACT_ADDRESS}
      confirmation_blocks: 128

observability:
  metrics:
    enabled: true
    port: 9090
    path: /metrics

  tracing:
    enabled: true
    exporter: jaeger
    endpoint: http://jaeger-collector:14268/api/traces
    sample_rate: 0.1

  logging:
    format: json
    level: info
    output: stdout
```

---

## Security Hardening

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: arka-core-network-policy
  namespace: arka-system
spec:
  podSelector:
    matchLabels:
      app: arka-core
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 9090
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - podSelector:
            matchLabels:
              app: arka-rtvm
      ports:
        - protocol: TCP
          port: 8081
```

### Pod Security Standards

```yaml
# pod-security.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: arka-restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
    - configMap
    - emptyDir
    - secret
    - persistentVolumeClaim
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
```

### Secrets Encryption

```bash
# Enable secrets encryption at rest (AWS EKS example)
aws eks create-addon \
  --cluster-name arka-cluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.25.0 \
  --configuration-values '{"controller":{"extraVolumeTags":{"encrypted":"true"}}}'
```

---

## High Availability

### Multi-AZ Deployment

```yaml
# Spread pods across availability zones
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: arka-core
        topologyKey: topology.kubernetes.io/zone
```

### Pod Disruption Budgets

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: arka-core-pdb
  namespace: arka-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: arka-core
```

### Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30
```

---

## Monitoring Setup

### Prometheus ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: arka-core
  namespace: arka-system
spec:
  selector:
    matchLabels:
      app: arka-core
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

### Grafana Dashboard

Import the ARKA Engine dashboard:

```bash
# Download dashboard
curl -o arka-dashboard.json https://grafana.com/api/dashboards/XXXXX/revisions/1/download

# Import via Grafana API
curl -X POST \
  -H "Content-Type: application/json" \
  -d @arka-dashboard.json \
  http://admin:password@grafana:3000/api/dashboards/import
```

### Alerting Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: arka-alerts
  namespace: arka-system
spec:
  groups:
    - name: pact.rules
      rules:
        - alert: ARKAHighErrorRate
          expr: |
            sum(rate(pact_evaluation_errors_total[5m])) /
            sum(rate(pact_evaluations_total[5m])) > 0.01
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: High error rate in ARKA evaluations
            description: Error rate is {{ $value | humanizePercentage }}

        - alert: ARKAHighLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(pact_evaluation_duration_seconds_bucket[5m])) by (le)
            ) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: High latency in ARKA evaluations
            description: P99 latency is {{ $value | humanizeDuration }}

        - alert: ARKADatabaseConnectionsLow
          expr: |
            pact_database_connections_available < 10
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: Low database connection pool
```

---

## Backup & Recovery

### Database Backup

```bash
# Create backup CronJob
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: arka-db-backup
  namespace: arka-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:14
              command:
                - /bin/bash
                - -c
                - |
                  pg_dump -h \$DATABASE_HOST -U \$DATABASE_USER \$DATABASE_NAME | \
                  gzip | aws s3 cp - s3://arka-backups/db/\$(date +%Y%m%d_%H%M%S).sql.gz
              envFrom:
                - secretRef:
                    name: arka-db-credentials
          restartPolicy: OnFailure
EOF
```

### Restore Procedure

```bash
# 1. Scale down ARKA services
kubectl scale deployment arka-core --replicas=0 -n arka-system

# 2. Download backup
aws s3 cp s3://arka-backups/db/20240115_020000.sql.gz ./backup.sql.gz
gunzip backup.sql.gz

# 3. Restore database
kubectl exec -it arka-postgresql-0 -n arka-system -- \
  psql -U arka -d pact < backup.sql

# 4. Scale up services
kubectl scale deployment arka-core --replicas=3 -n arka-system

# 5. Verify
kubectl exec -n arka-system $CORE_POD -- arka-cli health
```

---

## Deployment Checklist

### Pre-Deployment

- [ ] Infrastructure provisioned and tested
- [ ] Secrets created and verified
- [ ] TLS certificates configured
- [ ] Network policies applied
- [ ] Backup strategy implemented
- [ ] Monitoring configured
- [ ] Alerting rules deployed
- [ ] Load testing completed
- [ ] Security scan passed
- [ ] Disaster recovery tested

### Deployment

- [ ] Helm values reviewed
- [ ] Deploy to staging first
- [ ] Run smoke tests
- [ ] Verify health endpoints
- [ ] Check metrics are flowing
- [ ] Verify logs are captured
- [ ] Test authentication
- [ ] Test core functionality

### Post-Deployment

- [ ] Monitor error rates
- [ ] Check resource utilization
- [ ] Verify backup jobs running
- [ ] Test alerting
- [ ] Document configuration
- [ ] Update runbooks

---

## Next Steps

- [Configuration Reference](./configuration.md) - All configuration options
- [Monitoring & Observability](./monitoring.md) - Detailed monitoring setup
- [Scaling Guide](./scaling.md) - Scaling strategies
- [Security Model](../security/security-model.md) - Security architecture
