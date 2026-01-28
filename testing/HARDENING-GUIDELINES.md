# ARKA Engine Hardening Guidelines

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Security / Operations

---

## Overview

This document provides comprehensive security hardening guidelines for ARKA Engine deployments. Following these guidelines ensures a robust security posture across all components.

---

## Table of Contents

1. [Security Baseline](#security-baseline)
2. [Infrastructure Hardening](#infrastructure-hardening)
3. [Application Hardening](#application-hardening)
4. [Container Hardening](#container-hardening)
5. [Kubernetes Hardening](#kubernetes-hardening)
6. [Database Hardening](#database-hardening)
7. [Network Hardening](#network-hardening)
8. [Secrets Management](#secrets-management)
9. [Logging & Monitoring](#logging--monitoring)
10. [Compliance Validation](#compliance-validation)

---

## Security Baseline

### CIS Benchmark Compliance

ARKA Engine follows CIS Benchmarks for:

| Benchmark | Version | Compliance Level |
|-----------|---------|------------------|
| CIS Docker | v1.6.0 | Level 2 |
| CIS Kubernetes | v1.8.0 | Level 2 |
| CIS PostgreSQL | v1.0.0 | Level 1 |
| CIS Amazon Linux | v3.0.0 | Level 2 |

### Security Controls Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SECURITY CONTROLS MATRIX                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Control Category          │ Implementation Status │ Compliance            │
│   ─────────────────────────┼──────────────────────┼───────────────────────│
│   Access Control            │ ✅ Implemented        │ SOC2, ISO27001        │
│   Authentication            │ ✅ Implemented        │ SOC2, NIST            │
│   Authorization             │ ✅ Implemented        │ SOC2, ISO27001        │
│   Encryption at Rest        │ ✅ Implemented        │ PCI-DSS, GDPR         │
│   Encryption in Transit     │ ✅ Implemented        │ PCI-DSS, SOC2         │
│   Audit Logging             │ ✅ Implemented        │ SOC2, HIPAA           │
│   Vulnerability Management  │ ✅ Implemented        │ SOC2, ISO27001        │
│   Incident Response         │ ✅ Implemented        │ SOC2, ISO27001        │
│   Business Continuity       │ ✅ Implemented        │ SOC2, ISO27001        │
│   Data Loss Prevention      │ ✅ Implemented        │ GDPR, CCPA            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure Hardening

### Host OS Hardening

```bash
#!/bin/bash
# scripts/harden-host.sh

set -e

echo "🔒 Hardening host operating system..."

# 1. Disable unnecessary services
systemctl disable --now rpcbind
systemctl disable --now avahi-daemon
systemctl disable --now cups

# 2. Configure kernel parameters
cat >> /etc/sysctl.d/99-arka-security.conf << 'EOF'
# Network security
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_syncookies = 1

# Kernel hardening
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 2
kernel.perf_event_paranoid = 3
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2

# File system
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.suid_dumpable = 0
EOF

sysctl -p /etc/sysctl.d/99-arka-security.conf

# 3. Configure firewall
firewall-cmd --set-default-zone=drop
firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-port=8080/tcp  # ARKA API
firewall-cmd --permanent --add-port=9090/tcp  # gRPC
firewall-cmd --permanent --add-port=9091/tcp  # Metrics
firewall-cmd --reload

# 4. SSH hardening
cat > /etc/ssh/sshd_config.d/99-arka-security.conf << 'EOF'
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
MaxAuthTries 3
MaxSessions 2
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 60
EOF

systemctl restart sshd

# 5. Configure file permissions
chmod 600 /etc/ssh/sshd_config
chmod 700 /root
chmod 644 /etc/passwd
chmod 640 /etc/shadow
chmod 644 /etc/group
chmod 640 /etc/gshadow

# 6. Enable audit logging
systemctl enable --now auditd

cat >> /etc/audit/rules.d/pact.rules << 'EOF'
# Monitor privileged commands
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged

# Monitor authentication events
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/group -p wa -k group_changes

# Monitor SSH
-w /etc/ssh/sshd_config -p wa -k sshd_config

# Monitor Docker
-w /usr/bin/docker -p wa -k docker
-w /var/lib/docker -p wa -k docker
-w /etc/docker -p wa -k docker

# Monitor Kubernetes
-w /etc/kubernetes -p wa -k kubernetes
EOF

augenrules --load

# 7. Configure automatic security updates
cat > /etc/dnf/automatic.conf << 'EOF'
[commands]
upgrade_type = security
random_sleep = 360
download_updates = yes
apply_updates = yes

[emitters]
emit_via = stdio

[email]
email_from = root@localhost
email_to = security@arka-engine.io
EOF

systemctl enable --now dnf-automatic.timer

echo "✅ Host hardening complete"
```

### Cloud Provider Hardening

#### AWS Hardening

```yaml
# terraform/aws-hardening.tf
# VPC Configuration
resource "aws_vpc" "arka" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "arka-vpc"
  }
}

# VPC Flow Logs
resource "aws_flow_log" "arka" {
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.pact.id
}

# Security Group - API
resource "aws_security_group" "pact_api" {
  name        = "arka-api-sg"
  description = "ARKA API Security Group"
  vpc_id      = aws_vpc.pact.id

  # HTTPS only
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS"
  }

  # Egress restricted
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound"
  }

  egress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = [aws_subnet.database.cidr_block]
    description = "PostgreSQL"
  }

  tags = {
    Name = "arka-api-sg"
  }
}

# KMS Key for encryption
resource "aws_kms_key" "arka" {
  description             = "ARKA Engine encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      }
    ]
  })
}

# S3 Bucket with encryption and versioning
resource "aws_s3_bucket" "pact_data" {
  bucket = "arka-engine-data-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "pact_data" {
  bucket = aws_s3_bucket.pact_data.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "pact_data" {
  bucket = aws_s3_bucket.pact_data.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.pact.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "pact_data" {
  bucket = aws_s3_bucket.pact_data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM Role with least privilege
resource "aws_iam_role" "pact_api" {
  name = "arka-api-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "pact_api" {
  name = "arka-api-policy"
  role = aws_iam_role.pact_api.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = aws_kms_key.pact.arn
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "${aws_s3_bucket.pact_data.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = "arn:aws:secretsmanager:*:*:secret:pact/*"
      }
    ]
  })
}
```

---

## Application Hardening

### Security Headers Configuration

```typescript
// src/middleware/security-headers.ts
import helmet from 'helmet';
import { Express } from 'express';

export function configureSecurityHeaders(app: Express) {
  app.use(helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", "data:", "https:"],
        fontSrc: ["'self'"],
        objectSrc: ["'none'"],
        mediaSrc: ["'none'"],
        frameSrc: ["'none'"],
        baseUri: ["'self'"],
        formAction: ["'self'"],
        frameAncestors: ["'none'"],
        upgradeInsecureRequests: [],
      },
    },

    // Cross-Origin policies
    crossOriginEmbedderPolicy: true,
    crossOriginOpenerPolicy: { policy: "same-origin" },
    crossOriginResourcePolicy: { policy: "same-origin" },

    // DNS Prefetch Control
    dnsPrefetchControl: { allow: false },

    // Frameguard
    frameguard: { action: "deny" },

    // HSTS
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },

    // IE No Open
    ieNoOpen: true,

    // No Sniff
    noSniff: true,

    // Origin Agent Cluster
    originAgentCluster: true,

    // Permitted Cross Domain Policies
    permittedCrossDomainPolicies: { permittedPolicies: "none" },

    // Referrer Policy
    referrerPolicy: { policy: "strict-origin-when-cross-origin" },

    // XSS Filter
    xssFilter: true,
  }));

  // Additional custom headers
  app.use((req, res, next) => {
    res.setHeader('X-Request-ID', req.id);
    res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate, proxy-revalidate');
    res.setHeader('Pragma', 'no-cache');
    res.setHeader('Expires', '0');
    res.removeHeader('X-Powered-By');
    next();
  });
}
```

### Input Validation & Sanitization

```typescript
// src/middleware/input-validation.ts
import { z } from 'zod';
import xss from 'xss';
import { Request, Response, NextFunction } from 'express';

// Transaction schema with strict validation
const transactionSchema = z.object({
  id: z.string().uuid(),
  type: z.enum(['wire_transfer', 'ach', 'card_payment', 'cash_deposit']),
  amount: z.number().positive().max(1000000000),
  currency: z.string().length(3).regex(/^[A-Z]{3}$/),
  timestamp: z.string().datetime(),
  sender: z.object({
    id: z.string().max(100),
    name: z.string().max(200).transform(sanitizeString),
    country: z.string().length(2).regex(/^[A-Z]{2}$/),
  }),
  recipient: z.object({
    id: z.string().max(100),
    name: z.string().max(200).transform(sanitizeString),
    country: z.string().length(2).regex(/^[A-Z]{2}$/),
  }),
  metadata: z.record(z.unknown()).optional(),
});

function sanitizeString(value: string): string {
  return xss(value.trim());
}

export function validateTransaction(
  req: Request,
  res: Response,
  next: NextFunction
) {
  try {
    const validated = transactionSchema.parse(req.body.transaction);
    req.body.transaction = validated;
    next();
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'VALIDATION_ERROR',
        message: 'Invalid transaction data',
        details: error.errors,
      });
    }
    next(error);
  }
}

// SQL Injection prevention (parameterized queries)
export function buildSafeQuery(
  template: string,
  params: Record<string, unknown>
): { text: string; values: unknown[] } {
  const values: unknown[] = [];
  let paramIndex = 1;

  const text = template.replace(/:(\w+)/g, (_, key) => {
    if (!(key in params)) {
      throw new Error(`Missing parameter: ${key}`);
    }
    values.push(params[key]);
    return `$${paramIndex++}`;
  });

  return { text, values };
}

// Path traversal prevention
export function sanitizePath(userPath: string): string {
  const normalized = path.normalize(userPath);
  const resolved = path.resolve(ALLOWED_BASE_DIR, normalized);

  if (!resolved.startsWith(ALLOWED_BASE_DIR)) {
    throw new Error('Path traversal attempt detected');
  }

  return resolved;
}
```

### Rate Limiting & Throttling

```typescript
// src/middleware/rate-limiting.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import slowDown from 'express-slow-down';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

// Standard API rate limit
export const apiRateLimit = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: {
    error: 'RATE_LIMITED',
    message: 'Too many requests, please try again later',
    retryAfter: 60,
  },
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => {
    // Use user ID if authenticated, otherwise IP
    return req.user?.id || req.ip;
  },
  skip: (req) => {
    // Skip rate limiting for health checks
    return req.path === '/health';
  },
});

// Strict limit for authentication endpoints
export const authRateLimit = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per 15 minutes
  message: {
    error: 'AUTH_RATE_LIMITED',
    message: 'Too many authentication attempts',
    retryAfter: 900,
  },
});

// Compliance check rate limit (resource intensive)
export const complianceRateLimit = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 60 * 1000,
  max: 50, // 50 compliance checks per minute
  keyGenerator: (req) => `compliance:${req.user?.tenantId}`,
});

// Speed limiter for gradual slowdown
export const speedLimiter = slowDown({
  windowMs: 60 * 1000,
  delayAfter: 50, // Allow 50 requests per minute at full speed
  delayMs: (hits) => hits * 100, // Add 100ms per request over limit
  maxDelayMs: 2000, // Max delay of 2 seconds
});

// Concurrent request limiter
const concurrentRequests = new Map<string, number>();
const MAX_CONCURRENT = 10;

export function concurrentLimiter(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const key = req.user?.tenantId || req.ip;
  const current = concurrentRequests.get(key) || 0;

  if (current >= MAX_CONCURRENT) {
    return res.status(429).json({
      error: 'CONCURRENT_LIMIT',
      message: 'Too many concurrent requests',
    });
  }

  concurrentRequests.set(key, current + 1);

  res.on('finish', () => {
    const count = concurrentRequests.get(key) || 1;
    if (count <= 1) {
      concurrentRequests.delete(key);
    } else {
      concurrentRequests.set(key, count - 1);
    }
  });

  next();
}
```

---

## Container Hardening

### Dockerfile Security

```dockerfile
# Dockerfile - Production hardened
# Stage 1: Build
FROM node:20-alpine AS builder

# Don't run as root during build
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Install dependencies first (caching)
COPY --chown=appuser:appgroup package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# Copy source and build
COPY --chown=appuser:appgroup . .
RUN pnpm run build && pnpm prune --prod

# Stage 2: Production
FROM node:20-alpine AS production

# Security: Install security updates
RUN apk update && \
    apk upgrade && \
    apk add --no-cache tini dumb-init && \
    rm -rf /var/cache/apk/*

# Security: Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Security: Remove unnecessary packages
RUN apk del --purge apk-tools

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

# Security: Set restrictive permissions
RUN chmod -R 550 /app && \
    chmod -R 440 /app/dist

# Security: Create necessary directories with proper permissions
RUN mkdir -p /app/tmp /app/logs && \
    chown -R appuser:appgroup /app/tmp /app/logs && \
    chmod 750 /app/tmp /app/logs

# Security: Make filesystem read-only where possible
RUN chmod 555 /app

# Environment
ENV NODE_ENV=production \
    HOME=/app \
    NPM_CONFIG_PREFIX=/app/.npm

# Security: Drop all capabilities
USER appuser:appgroup

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD node -e "require('http').get('http://localhost:8080/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Use tini as init
ENTRYPOINT ["/sbin/tini", "--"]

CMD ["node", "dist/index.js"]

# Security labels
LABEL org.opencontainers.image.source="https://github.com/ARKA-SYSTEMS/ARKA-ENGINE" \
      org.opencontainers.image.description="ARKA Engine - Hardened Production Image" \
      org.opencontainers.image.licenses="Apache-2.0" \
      security.hardened="true"
```

### Container Security Scanning

```yaml
# .github/workflows/container-security.yml
name: Container Security Scan

on:
  push:
    branches: [main, develop]
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t arka-engine:scan .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'arka-engine:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Run Snyk container scan
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: arka-engine:scan
          args: --severity-threshold=high

      - name: Run Dockle linter
        uses: erzz/dockle-action@v1
        with:
          image: arka-engine:scan
          failure-threshold: high
          exit-code: 1

      - name: Run Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          failure-threshold: warning

      - name: Fail on critical vulnerabilities
        run: |
          CRITICAL=$(cat trivy-results.sarif | jq '.runs[0].results | length')
          if [ "$CRITICAL" -gt 0 ]; then
            echo "Critical vulnerabilities found!"
            exit 1
          fi
```

---

## Kubernetes Hardening

### Pod Security Standards

```yaml
# deploy/kubernetes/pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: arka-production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arka-api
  namespace: arka-production
spec:
  template:
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault

      # Service account
      serviceAccountName: arka-api
      automountServiceAccountToken: false

      containers:
        - name: arka-api
          image: arka-engine/api:1.0.0

          # Container-level security
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            capabilities:
              drop:
                - ALL

          # Resource limits (prevent DoS)
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
              ephemeral-storage: "100Mi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
              ephemeral-storage: "500Mi"

          # Writable volumes for temp files
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-tmp
              mountPath: /app/tmp

      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: app-tmp
          emptyDir:
            sizeLimit: 100Mi

      # Node affinity for isolation
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In
                    values:
                      - arka-workload

      # Tolerations
      tolerations:
        - key: "arka-workload"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
```

### Network Policies

```yaml
# deploy/kubernetes/network-policies.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: arka-api-network-policy
  namespace: arka-production
spec:
  podSelector:
    matchLabels:
      app: arka-api
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow from ingress controller only
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 9090

    # Allow from monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9091

  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # Allow database
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432

    # Allow Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow external HTTPS (for AI providers, blockchain)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443

---
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: arka-production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

---

## Database Hardening

### PostgreSQL Security Configuration

```sql
-- postgresql-security.sql

-- 1. Authentication configuration
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
ALTER SYSTEM SET ssl = 'on';
ALTER SYSTEM SET ssl_min_protocol_version = 'TLSv1.3';
ALTER SYSTEM SET ssl_ciphers = 'HIGH:!aNULL:!MD5';

-- 2. Connection security
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
ALTER SYSTEM SET log_hostname = 'off';

-- 3. Statement logging (for audit)
ALTER SYSTEM SET log_statement = 'ddl';
ALTER SYSTEM SET log_min_duration_statement = 1000;  -- Log slow queries

-- 4. Row-level security
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON customers
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation_insert ON customers
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

-- 5. Least privilege roles
CREATE ROLE pact_readonly;
GRANT CONNECT ON DATABASE pact TO pact_readonly;
GRANT USAGE ON SCHEMA public TO pact_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO pact_readonly;

CREATE ROLE pact_readwrite;
GRANT CONNECT ON DATABASE pact TO pact_readwrite;
GRANT USAGE ON SCHEMA public TO pact_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO pact_readwrite;

CREATE ROLE pact_admin;
GRANT ALL PRIVILEGES ON DATABASE pact TO pact_admin;

-- 6. Revoke public permissions
REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC;

-- 7. Audit logging table
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  user_name TEXT,
  client_addr INET,
  action TEXT,
  table_name TEXT,
  old_data JSONB,
  new_data JSONB
);

-- Audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (user_name, client_addr, action, table_name, old_data, new_data)
  VALUES (
    current_user,
    inet_client_addr(),
    TG_OP,
    TG_TABLE_NAME,
    CASE WHEN TG_OP IN ('UPDATE', 'DELETE') THEN row_to_json(OLD) END,
    CASE WHEN TG_OP IN ('INSERT', 'UPDATE') THEN row_to_json(NEW) END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Apply to sensitive tables
CREATE TRIGGER audit_rules
  AFTER INSERT OR UPDATE OR DELETE ON rules
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_customers
  AFTER INSERT OR UPDATE OR DELETE ON customers
  FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

### pg_hba.conf Security

```
# pg_hba.conf - PostgreSQL Client Authentication

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Reject all by default
local   all             all                                     reject
host    all             all             0.0.0.0/0               reject
host    all             all             ::/0                    reject

# Allow local socket for superuser only
local   all             postgres                                peer

# Allow from application servers only (with SSL)
hostssl pact            pact_app        10.0.1.0/24             scram-sha-256
hostssl pact            pact_readonly   10.0.2.0/24             scram-sha-256

# Allow from admin network (with SSL and client certs)
hostssl all             pact_admin      10.0.0.0/24             cert clientcert=verify-full

# Replication (with SSL)
hostssl replication     replicator      10.0.3.0/24             scram-sha-256
```

---

## Secrets Management

### HashiCorp Vault Integration

```yaml
# vault-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-agent-config
  namespace: arka-production
data:
  vault-agent-config.hcl: |
    exit_after_auth = false
    pid_file = "/home/vault/pidfile"

    auto_auth {
      method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
          role = "arka-api"
        }
      }

      sink "file" {
        config = {
          path = "/home/vault/.vault-token"
        }
      }
    }

    template {
      destination = "/vault/secrets/database.env"
      contents = <<EOT
      {{- with secret "pact/data/database" }}
      DATABASE_URL={{ .Data.data.url }}
      DATABASE_PASSWORD={{ .Data.data.password }}
      {{- end }}
      EOT
    }

    template {
      destination = "/vault/secrets/encryption.env"
      contents = <<EOT
      {{- with secret "pact/data/encryption" }}
      ENCRYPTION_KEY={{ .Data.data.key }}
      JWT_SECRET={{ .Data.data.jwt_secret }}
      {{- end }}
      EOT
    }
```

### Secret Rotation

```typescript
// src/services/secret-rotation.ts
import { SecretsManager } from '@aws-sdk/client-secrets-manager';
import { schedule } from 'node-cron';

class SecretRotationService {
  private secretsManager = new SecretsManager({});
  private rotationInterval = '0 0 * * 0'; // Weekly

  constructor() {
    this.scheduleRotation();
  }

  private scheduleRotation() {
    schedule(this.rotationInterval, async () => {
      await this.rotateSecrets();
    });
  }

  async rotateSecrets() {
    const secrets = ['pact/database', 'pact/encryption', 'pact/api-keys'];

    for (const secretId of secrets) {
      try {
        await this.secretsManager.rotateSecret({
          SecretId: secretId,
          RotationLambdaARN: process.env.ROTATION_LAMBDA_ARN,
        });
        logger.info(`Rotated secret: ${secretId}`);
      } catch (error) {
        logger.error(`Failed to rotate secret: ${secretId}`, error);
        await this.alertOnFailure(secretId, error);
      }
    }
  }

  private async alertOnFailure(secretId: string, error: Error) {
    // Send alert to security team
  }
}
```

---

## Compliance Validation

### Security Audit Checklist

```yaml
# security-audit-checklist.yaml
security_audit:
  infrastructure:
    - id: INFRA-001
      control: "OS patches applied"
      frequency: weekly
      automated: true

    - id: INFRA-002
      control: "Firewall rules reviewed"
      frequency: monthly
      automated: false

    - id: INFRA-003
      control: "SSL certificates valid"
      frequency: daily
      automated: true

  application:
    - id: APP-001
      control: "Dependency vulnerabilities scanned"
      frequency: daily
      automated: true

    - id: APP-002
      control: "SAST scan completed"
      frequency: per_commit
      automated: true

    - id: APP-003
      control: "DAST scan completed"
      frequency: weekly
      automated: true

  access_control:
    - id: ACCESS-001
      control: "User access reviewed"
      frequency: quarterly
      automated: false

    - id: ACCESS-002
      control: "Service accounts audited"
      frequency: monthly
      automated: true

    - id: ACCESS-003
      control: "Privileged access reviewed"
      frequency: monthly
      automated: false

  data_protection:
    - id: DATA-001
      control: "Encryption at rest verified"
      frequency: monthly
      automated: true

    - id: DATA-002
      control: "Backup integrity tested"
      frequency: weekly
      automated: true

    - id: DATA-003
      control: "Data classification reviewed"
      frequency: quarterly
      automated: false
```

### Automated Compliance Checks

```bash
#!/bin/bash
# scripts/compliance-check.sh

set -e

echo "🔍 Running compliance checks..."

# CIS Kubernetes Benchmark
echo "Running CIS Kubernetes Benchmark..."
kube-bench run --targets=node,master,controlplane,etcd,policies

# Container security
echo "Scanning containers..."
trivy image arka-engine/api:latest --severity HIGH,CRITICAL

# Secret detection
echo "Checking for secrets in code..."
trufflehog filesystem . --only-verified

# Infrastructure compliance
echo "Checking Terraform compliance..."
tfsec .
checkov -d ./terraform

# Generate compliance report
echo "Generating compliance report..."
cat > compliance-report.json << EOF
{
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "checks": {
    "cis_kubernetes": "$CIS_RESULT",
    "container_scan": "$CONTAINER_RESULT",
    "secret_detection": "$SECRET_RESULT",
    "infrastructure": "$INFRA_RESULT"
  },
  "overall_status": "$([ "$FAILED" -eq 0 ] && echo 'PASS' || echo 'FAIL')"
}
EOF

echo "✅ Compliance check complete"
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Security Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
