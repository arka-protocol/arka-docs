# ARKA Engine Environment Profiles

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Operations

---

## Overview

This document defines environment configurations for development, staging, and production deployments of ARKA Engine. Each environment is optimized for its specific use case while maintaining consistency across the deployment pipeline.

---

## Table of Contents

1. [Environment Overview](#environment-overview)
2. [Development Environment](#development-environment)
3. [Staging Environment](#staging-environment)
4. [Production Environment](#production-environment)
5. [Environment Variables](#environment-variables)
6. [Feature Flags](#feature-flags)
7. [Data Management](#data-management)
8. [Promotion Process](#promotion-process)

---

## Environment Overview

### Environment Comparison

| Aspect | Development | Staging | Production |
|--------|-------------|---------|------------|
| **Purpose** | Feature development | Pre-production testing | Live system |
| **Data** | Synthetic/mock | Anonymized production | Real data |
| **Scale** | Minimal | Production-like | Full scale |
| **Access** | All developers | Limited team | Restricted |
| **Monitoring** | Basic | Full | Full + alerting |
| **SSL/TLS** | Self-signed | Valid certs | Valid certs |
| **Backups** | None | Daily | Continuous |
| **SLA** | None | 99% | 99.99% |

### Environment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ENVIRONMENT ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   DEVELOPMENT                STAGING                   PRODUCTION           │
│   ───────────                ───────                   ──────────           │
│                                                                              │
│   ┌─────────┐              ┌─────────┐              ┌─────────────┐        │
│   │  Local  │              │   AWS   │              │    AWS      │        │
│   │ Docker  │──────────────│  EKS    │──────────────│ EKS (Multi- │        │
│   │         │   Promote    │ Single  │   Promote    │   Region)   │        │
│   └─────────┘              │  AZ     │              └─────────────┘        │
│       │                    └─────────┘                    │                 │
│       │                         │                         │                 │
│   ┌───┴───┐               ┌─────┴─────┐           ┌──────┴──────┐         │
│   │SQLite/│               │PostgreSQL │           │ PostgreSQL  │         │
│   │ PG    │               │  Single   │           │ HA Cluster  │         │
│   └───────┘               └───────────┘           └─────────────┘         │
│       │                         │                         │                 │
│   ┌───┴───┐               ┌─────┴─────┐           ┌──────┴──────┐         │
│   │ Redis │               │   Redis   │           │ Redis       │         │
│   │Single │               │   Single  │           │ Cluster     │         │
│   └───────┘               └───────────┘           └─────────────┘         │
│                                                                              │
│   Feature     ──►         Integration  ──►        Blue/Green               │
│   Branches                Testing                 Deployment               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Development Environment

### Configuration

```yaml
# config/environments/development.yaml
environment: development

server:
  host: "0.0.0.0"
  port: 8080
  grpcPort: 9090
  metricsPort: 9091

logging:
  level: debug
  format: pretty
  includeStackTrace: true
  colorize: true

database:
  host: localhost
  port: 5432
  name: arka_dev
  user: arka_dev
  password: arka_dev_password
  pool:
    min: 2
    max: 10
  logging: true
  synchronize: true  # Auto-migrate in dev

redis:
  host: localhost
  port: 6379
  database: 0

kafka:
  enabled: false  # Optional in dev
  brokers:
    - localhost:9092

security:
  jwtSecret: dev-secret-not-for-production
  encryptionKey: dev-encryption-key-32chars!
  corsOrigins:
    - "http://localhost:3000"
    - "http://localhost:8080"
  rateLimit:
    enabled: false

features:
  mockExternalServices: true
  seedTestData: true
  enableSwaggerUI: true
  enableGraphiQL: true
  bypassAuth: false  # Set true for local testing

rtvm:
  workers: 2
  maxExecutionTime: 60000  # Generous timeout for debugging
  enableProfiling: true

ai:
  enabled: true
  provider: mock  # Use mock in dev to avoid API costs
  mockResponses: true

blockchain:
  enabled: false  # Disabled in dev by default
  network: localhost
  mockAnchoring: true

observability:
  tracing:
    enabled: true
    exporter: console
  metrics:
    enabled: true
    exporter: prometheus
```

### Docker Compose (Development)

```yaml
# docker-compose.dev.yml
version: '3.9'

services:
  arka-api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
      - "9090:9090"
      - "9091:9091"
      - "9229:9229"  # Debugger
    environment:
      - NODE_ENV=development
      - ARKA_ENV=development
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - postgres
      - redis
    command: pnpm run dev

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: arka_dev
      POSTGRES_PASSWORD: arka_dev_password
      POSTGRES_DB: arka_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Development tools
  adminer:
    image: adminer
    ports:
      - "8081:8080"

  mailhog:
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres_dev_data:
```

### Local Setup Script

```bash
#!/bin/bash
# scripts/setup-dev.sh

set -e

echo "🚀 Setting up ARKA Engine development environment..."

# Check prerequisites
command -v node >/dev/null 2>&1 || { echo "Node.js is required"; exit 1; }
command -v docker >/dev/null 2>&1 || { echo "Docker is required"; exit 1; }
command -v pnpm >/dev/null 2>&1 || npm install -g pnpm

# Install dependencies
echo "📦 Installing dependencies..."
pnpm install

# Start infrastructure
echo "🐳 Starting Docker services..."
docker-compose -f docker-compose.dev.yml up -d postgres redis

# Wait for services
echo "⏳ Waiting for services..."
sleep 5

# Run migrations
echo "🔄 Running database migrations..."
pnpm run db:migrate

# Seed test data
echo "🌱 Seeding test data..."
pnpm run db:seed

# Generate types
echo "📝 Generating types..."
pnpm run generate:types

echo "✅ Development environment ready!"
echo ""
echo "Run 'pnpm run dev' to start the development server"
echo "API will be available at http://localhost:8080"
echo "Swagger UI at http://localhost:8080/docs"
```

---

## Staging Environment

### Configuration

```yaml
# config/environments/staging.yaml
environment: staging

server:
  host: "0.0.0.0"
  port: 8080
  grpcPort: 9090
  metricsPort: 9091

logging:
  level: info
  format: json
  includeStackTrace: true
  sensitiveFields:
    - password
    - apiKey
    - token

database:
  host: ${DATABASE_HOST}
  port: 5432
  name: pact_staging
  user: ${DATABASE_USER}
  password: ${DATABASE_PASSWORD}
  ssl:
    enabled: true
    rejectUnauthorized: true
  pool:
    min: 5
    max: 50
  logging: true
  synchronize: false

redis:
  host: ${REDIS_HOST}
  port: 6379
  password: ${REDIS_PASSWORD}
  tls: true
  database: 0

kafka:
  enabled: true
  brokers: ${KAFKA_BROKERS}
  ssl: true
  sasl:
    mechanism: SCRAM-SHA-512
    username: ${KAFKA_USER}
    password: ${KAFKA_PASSWORD}

security:
  jwtSecret: ${JWT_SECRET}
  encryptionKey: ${ENCRYPTION_KEY}
  corsOrigins:
    - "https://staging.arka-engine.io"
    - "https://staging-app.arka-engine.io"
  rateLimit:
    enabled: true
    windowMs: 60000
    max: 1000

features:
  mockExternalServices: false
  seedTestData: false
  enableSwaggerUI: true
  enableGraphiQL: true
  bypassAuth: false

rtvm:
  workers: 4
  maxExecutionTime: 30000
  enableProfiling: false

ai:
  enabled: true
  provider: openai
  model: gpt-4
  apiKey: ${OPENAI_API_KEY}

blockchain:
  enabled: true
  network: polygon-mumbai  # Testnet
  rpcUrl: ${POLYGON_MUMBAI_RPC}
  privateKey: ${BLOCKCHAIN_PRIVATE_KEY}

observability:
  tracing:
    enabled: true
    exporter: jaeger
    endpoint: ${JAEGER_ENDPOINT}
    sampleRate: 1.0  # 100% sampling in staging
  metrics:
    enabled: true
    exporter: prometheus

alerts:
  enabled: true
  channels:
    - type: slack
      webhook: ${SLACK_STAGING_WEBHOOK}
```

### Kubernetes Configuration (Staging)

```yaml
# deploy/kubernetes/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: arka-staging

resources:
  - ../../base

patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
    target:
      kind: Deployment
      name: arka-api

  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
    target:
      kind: Deployment
      name: arka-rtvm

configMapGenerator:
  - name: arka-config
    behavior: merge
    literals:
      - ARKA_ENV=staging
      - LOG_LEVEL=info
      - ENABLE_SWAGGER=true

secretGenerator:
  - name: arka-secrets
    behavior: merge
    files:
      - secrets/staging.env

images:
  - name: arka-engine/api
    newTag: staging-latest
```

### Staging Deployment Pipeline

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        run: |
          docker build -t arka-engine/api:staging-${{ github.sha }} .
          docker push $ECR_REGISTRY/arka-engine/api:staging-${{ github.sha }}

      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --name arka-staging
          kubectl set image deployme../arka-api \
            arka-api=$ECR_REGISTRY/arka-engine/api:staging-${{ github.sha }} \
            -n arka-staging

      - name: Wait for rollout
        run: |
          kubectl rollout status deployme../arka-api -n arka-staging --timeout=5m

      - name: Run smoke tests
        run: |
          pnpm run test:smoke --env=staging

      - name: Notify
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Staging deployment ${{ job.status }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Production Environment

### Configuration

```yaml
# config/environments/production.yaml
environment: production

server:
  host: "0.0.0.0"
  port: 8080
  grpcPort: 9090
  metricsPort: 9091
  shutdownTimeout: 30000

logging:
  level: warn
  format: json
  includeStackTrace: false
  sensitiveFields:
    - password
    - apiKey
    - token
    - ssn
    - accountNumber

database:
  host: ${DATABASE_HOST}
  port: 5432
  name: pact_production
  user: ${DATABASE_USER}
  password: ${DATABASE_PASSWORD}
  ssl:
    enabled: true
    rejectUnauthorized: true
    ca: ${DATABASE_CA_CERT}
  pool:
    min: 20
    max: 100
  logging: false
  synchronize: false
  readReplicas:
    - host: ${DATABASE_REPLICA_1}
    - host: ${DATABASE_REPLICA_2}

redis:
  mode: cluster
  nodes:
    - host: ${REDIS_NODE_1}
      port: 6379
    - host: ${REDIS_NODE_2}
      port: 6379
    - host: ${REDIS_NODE_3}
      port: 6379
  password: ${REDIS_PASSWORD}
  tls: true

kafka:
  enabled: true
  brokers: ${KAFKA_BROKERS}
  ssl: true
  sasl:
    mechanism: SCRAM-SHA-512
    username: ${KAFKA_USER}
    password: ${KAFKA_PASSWORD}
  consumer:
    groupId: arka-production
    sessionTimeout: 30000

security:
  jwtSecret: ${JWT_SECRET}
  encryptionKey: ${ENCRYPTION_KEY}
  corsOrigins:
    - "https://arka-engine.io"
    - "https://app.arka-engine.io"
    - "https://api.arka-engine.io"
  rateLimit:
    enabled: true
    windowMs: 60000
    max: 100
    keyGenerator: ip+userId
  helmet:
    enabled: true
    hsts:
      maxAge: 31536000
      includeSubDomains: true
      preload: true

features:
  mockExternalServices: false
  seedTestData: false
  enableSwaggerUI: false
  enableGraphiQL: false
  bypassAuth: false
  maintenanceMode: false

rtvm:
  workers: 8
  maxExecutionTime: 30000
  enableProfiling: false
  caching:
    enabled: true
    ttl: 3600000

ai:
  enabled: true
  provider: openai
  model: gpt-4
  apiKey: ${OPENAI_API_KEY}
  rateLimit:
    requestsPerMinute: 60
    tokensPerMinute: 90000

blockchain:
  enabled: true
  network: polygon-mainnet
  rpcUrl: ${POLYGON_MAINNET_RPC}
  privateKey: ${BLOCKCHAIN_PRIVATE_KEY}
  confirmations: 12
  gasPrice: auto

observability:
  tracing:
    enabled: true
    exporter: datadog
    endpoint: ${DD_AGENT_HOST}
    sampleRate: 0.1  # 10% sampling in production
  metrics:
    enabled: true
    exporter: datadog
  profiling:
    enabled: true
    continuous: true

alerts:
  enabled: true
  channels:
    - type: pagerduty
      serviceKey: ${PAGERDUTY_SERVICE_KEY}
      severity: critical
    - type: slack
      webhook: ${SLACK_PRODUCTION_WEBHOOK}
      severity: warning
    - type: email
      recipients:
        - oncall@arka-engine.io
      severity: error
```

### Production Kubernetes

```yaml
# deploy/kubernetes/overlays/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arka-api
  namespace: arka-production
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: arka-api
              topologyKey: topology.kubernetes.io/zone
      containers:
        - name: arka-api
          resources:
            requests:
              cpu: "1000m"
              memory: "2Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: arka-api
```

### Production HPA

```yaml
# deploy/kubernetes/overlays/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: arka-api-hpa
  namespace: arka-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: arka-api
  minReplicas: 6
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Percent
          value: 10
          periodSeconds: 120
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 50
          periodSeconds: 30
        - type: Pods
          value: 5
          periodSeconds: 30
      selectPolicy: Max
```

---

## Environment Variables

### Complete Variable Reference

```bash
# =============================================================================
# COMMON VARIABLES (All Environments)
# =============================================================================

# Application
ARKA_ENV=production                    # development | staging | production
NODE_ENV=production                    # development | production
LOG_LEVEL=info                         # debug | info | warn | error

# Server
ARKA_API_PORT=8080
ARKA_GRPC_PORT=9090
ARKA_METRICS_PORT=9091

# =============================================================================
# DATABASE
# =============================================================================

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=pact
DATABASE_USER=pact
DATABASE_PASSWORD=secret
DATABASE_SSL=true
DATABASE_CA_CERT=/path/to/ca.crt       # Production only

# Read replicas (Production)
DATABASE_REPLICA_1=replica1.db.example.com
DATABASE_REPLICA_2=replica2.db.example.com

# =============================================================================
# REDIS
# =============================================================================

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=secret
REDIS_TLS=true
REDIS_DATABASE=0

# Cluster mode (Production)
REDIS_MODE=cluster
REDIS_NODES=node1:6379,node2:6379,node3:6379

# =============================================================================
# KAFKA
# =============================================================================

KAFKA_ENABLED=true
KAFKA_BROKERS=kafka1:9092,kafka2:9092
KAFKA_SSL=true
KAFKA_USER=pact
KAFKA_PASSWORD=secret

# =============================================================================
# SECURITY
# =============================================================================

JWT_SECRET=your-jwt-secret-min-32-chars
JWT_EXPIRES_IN=1h
ENCRYPTION_KEY=your-32-character-encryption-key

# =============================================================================
# EXTERNAL SERVICES
# =============================================================================

# AI Provider
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Blockchain
POLYGON_MAINNET_RPC=https://polygon-rpc.com
BLOCKCHAIN_PRIVATE_KEY=0x...

# =============================================================================
# OBSERVABILITY
# =============================================================================

# Datadog
DD_AGENT_HOST=datadog-agent
DD_API_KEY=your-api-key
DD_SERVICE=arka-engine
DD_ENV=production

# Jaeger (Staging)
JAEGER_ENDPOINT=http://jaeger:14268/api/traces

# =============================================================================
# ALERTS
# =============================================================================

PAGERDUTY_SERVICE_KEY=your-service-key
SLACK_WEBHOOK_URL=https://hooks.slack.com/...

# =============================================================================
# FEATURE FLAGS
# =============================================================================

FEATURE_NEW_RULE_ENGINE=false
FEATURE_AI_EXPLANATIONS=true
FEATURE_BLOCKCHAIN_ANCHORING=true
```

---

## Feature Flags

### Feature Flag Configuration

```yaml
# config/feature-flags.yaml
flags:
  # Rule Engine Features
  new_rule_engine:
    default: false
    environments:
      development: true
      staging: true
      production: false
    rollout:
      type: percentage
      value: 0

  # AI Features
  ai_explanations:
    default: true
    environments:
      development: true
      staging: true
      production: true

  ai_rule_suggestions:
    default: false
    environments:
      development: true
      staging: true
      production: false
    rollout:
      type: percentage
      value: 10

  # Blockchain Features
  blockchain_anchoring:
    default: true
    environments:
      development: false
      staging: true
      production: true

  multi_chain_support:
    default: false
    environments:
      development: true
      staging: true
      production: false

  # Performance Features
  query_caching:
    default: true
    environments:
      development: true
      staging: true
      production: true

  compiled_rule_cache:
    default: true
    environments:
      development: false  # Disable for hot-reload
      staging: true
      production: true

  # UI Features
  dark_mode:
    default: false
    environments:
      development: true
      staging: true
      production: false
    rollout:
      type: user_attribute
      attribute: beta_tester
```

### Feature Flag Service

```typescript
// src/services/feature-flags.ts
import { LaunchDarkly } from 'launchdarkly-node-server-sdk';

class FeatureFlagService {
  private client: LaunchDarkly.LDClient;

  async initialize() {
    this.client = LaunchDarkly.init(process.env.LAUNCHDARKLY_SDK_KEY!);
    await this.client.waitForInitialization();
  }

  async isEnabled(
    flag: string,
    context: { userId?: string; tenantId?: string; environment?: string }
  ): Promise<boolean> {
    const ldContext = {
      kind: 'user',
      key: context.userId || 'anonymous',
      custom: {
        tenantId: context.tenantId,
        environment: context.environment || process.env.ARKA_ENV,
      },
    };

    return this.client.variation(flag, ldContext, false);
  }

  async getVariation<T>(
    flag: string,
    context: { userId?: string; tenantId?: string },
    defaultValue: T
  ): Promise<T> {
    const ldContext = {
      kind: 'user',
      key: context.userId || 'anonymous',
    };

    return this.client.variation(flag, ldContext, defaultValue);
  }
}

export const featureFlags = new FeatureFlagService();
```

---

## Data Management

### Data Handling by Environment

| Aspect | Development | Staging | Production |
|--------|-------------|---------|------------|
| Data Source | Synthetic | Anonymized prod | Real data |
| PII | Mock data | Masked/tokenized | Protected |
| Refresh Cycle | On-demand | Weekly | N/A |
| Backup | None | Daily | Continuous |
| Retention | Session | 30 days | Per policy |

### Data Seeding (Development)

```typescript
// scripts/seed-dev-data.ts
import { faker } from '@faker-js/faker';
import { db } from '../src/database';

async function seedDevelopmentData() {
  console.log('🌱 Seeding development data...');

  // Seed customers
  const customers = Array.from({ length: 100 }, () => ({
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    country: faker.location.countryCode(),
    riskScore: faker.number.int({ min: 0, max: 100 }),
    createdAt: faker.date.past(),
  }));

  await db.customers.createMany({ data: customers });

  // Seed rules
  const rules = [
    {
      id: 'rule_dev_001',
      name: 'High Value Transaction',
      category: 'aml',
      severity: 'high',
      status: 'active',
      definition: {
        type: 'condition',
        condition: { field: 'amount', operator: 'gt', value: 10000 },
        actions: [{ type: 'flag', severity: 'high' }],
      },
    },
    // ... more rules
  ];

  await db.rules.createMany({ data: rules });

  // Seed transactions
  const transactions = Array.from({ length: 1000 }, () => ({
    id: faker.string.uuid(),
    type: faker.helpers.arrayElement(['wire_transfer', 'ach', 'card']),
    amount: faker.number.float({ min: 100, max: 100000 }),
    currency: 'USD',
    senderId: faker.helpers.arrayElement(customers).id,
    recipientId: faker.helpers.arrayElement(customers).id,
    timestamp: faker.date.recent(),
  }));

  await db.transactions.createMany({ data: transactions });

  console.log('✅ Development data seeded successfully');
}

seedDevelopmentData().catch(console.error);
```

### Data Anonymization (Staging)

```typescript
// scripts/anonymize-for-staging.ts
import { createHash } from 'crypto';

const anonymizationRules = {
  customers: {
    name: (value: string) => `Customer_${hash(value).substring(0, 8)}`,
    email: (value: string) => `user_${hash(value).substring(0, 8)}@example.com`,
    phone: () => '+1-555-XXX-XXXX',
    ssn: () => 'XXX-XX-XXXX',
    address: (value: string) => '123 Anonymized St',
  },
  transactions: {
    // Preserve structure, anonymize identifiers
    senderId: (value: string) => hash(value),
    recipientId: (value: string) => hash(value),
    // Keep amounts and dates for realistic testing
  },
};

function hash(value: string): string {
  return createHash('sha256').update(value).digest('hex');
}

async function anonymizeData() {
  // Export from production
  // Apply anonymization rules
  // Import to staging
}
```

---

## Promotion Process

### Code Promotion Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CODE PROMOTION FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Feature Branch ──► develop ──► staging ──► main ──► production           │
│        │                │           │          │           │                │
│        ▼                ▼           ▼          ▼           ▼                │
│   ┌─────────┐    ┌──────────┐  ┌────────┐  ┌───────┐  ┌─────────┐         │
│   │  Unit   │    │   CI     │  │ Deploy │  │ Tag   │  │ Deploy  │         │
│   │  Tests  │    │   Build  │  │ to     │  │ Rele- │  │ to      │         │
│   │         │    │   Tests  │  │ Stag-  │  │ ase   │  │ Prod    │         │
│   └─────────┘    └──────────┘  │ ing    │  └───────┘  └─────────┘         │
│                                └────────┘                                   │
│                                     │                                       │
│                                     ▼                                       │
│                              ┌────────────┐                                 │
│                              │  QA Sign-  │                                 │
│                              │  off       │                                 │
│                              └────────────┘                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Release Checklist

```yaml
# .github/RELEASE_CHECKLIST.md
release_checklist:
  pre_release:
    - [ ] All tests passing on staging
    - [ ] Performance benchmarks meet SLOs
    - [ ] Security scan completed
    - [ ] Documentation updated
    - [ ] CHANGELOG updated
    - [ ] Database migrations tested
    - [ ] Rollback plan documented

  staging_validation:
    - [ ] Smoke tests passed
    - [ ] Integration tests passed
    - [ ] Load tests passed
    - [ ] QA sign-off obtained
    - [ ] Stakeholder approval

  production_deploy:
    - [ ] Maintenance window scheduled
    - [ ] On-call team notified
    - [ ] Monitoring dashboards ready
    - [ ] Rollback procedure reviewed
    - [ ] Database backup verified

  post_deploy:
    - [ ] Health checks passing
    - [ ] Metrics within thresholds
    - [ ] No error rate spike
    - [ ] Customer-facing functionality verified
    - [ ] Release notes published
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Platform Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
