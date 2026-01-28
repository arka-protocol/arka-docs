# ARKA Engine Administrator Guide

**Version:** 1.0.0
**Last Updated:** December 2024

## Overview

This guide is designed for system administrators responsible for installing, configuring, operating, and maintaining ARKA Engine in production environments.

---

## Table of Contents

1. [Administrator Responsibilities](#administrator-responsibilities)
2. [Initial Setup](#initial-setup)
3. [Configuration Management](#configuration-management)
4. [User & Access Management](#user--access-management)
5. [Plugin Management](#plugin-management)
6. [Monitoring & Alerting](#monitoring--alerting)
7. [Backup & Recovery](#backup--recovery)
8. [Scaling Operations](#scaling-operations)
9. [Maintenance Tasks](#maintenance-tasks)
10. [Security Operations](#security-operations)
11. [Troubleshooting](#troubleshooting)
12. [Operational Runbooks](#operational-runbooks)

---

## Administrator Responsibilities

### Daily Tasks
- Monitor system health dashboards
- Review and acknowledge alerts
- Check backup completion status
- Review security event logs

### Weekly Tasks
- Review system performance metrics
- Audit user access and permissions
- Review and rotate API keys approaching expiration
- Check for available updates

### Monthly Tasks
- Perform disaster recovery testing
- Review and optimize database performance
- Audit compliance rule changes
- Generate operational reports

### Quarterly Tasks
- Security assessment and penetration testing
- Capacity planning review
- Documentation review and updates
- License compliance verification

---

## Initial Setup

### Pre-Installation Checklist

```bash
# 1. Verify system requirements
./scripts/check-requirements.sh

# 2. Configure environment variables
cp .env.example .env
nano .env

# 3. Initialize database
./scripts/init-database.sh

# 4. Generate encryption keys
./scripts/generate-keys.sh

# 5. Configure SSL certificates
./scripts/setup-ssl.sh
```

### Environment Configuration

```bash
# .env - Production Configuration

# Core Settings
NODE_ENV=production
ARKA_VERSION=1.0.0
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://pact:password@db.example.com:5432/pact
DATABASE_POOL_SIZE=50
DATABASE_SSL=true

# Redis
REDIS_URL=redis://redis.example.com:6379
REDIS_PASSWORD=your_redis_password
REDIS_TLS=true

# API
API_PORT=3000
API_HOST=0.0.0.0
API_CORS_ORIGINS=https://app.example.com

# Security
JWT_SECRET=your_jwt_secret_min_32_chars
JWT_EXPIRATION=1h
API_RATE_LIMIT=1000
ENCRYPTION_KEY=your_encryption_key_32_chars

# Blockchain (Optional)
ETHEREUM_RPC_URL=https://mainnet.infura.io/v3/your_key
BLOCKCHAIN_ANCHORING_ENABLED=true

# Monitoring
PROMETHEUS_ENABLED=true
PROMETHEUS_PORT=9090
```

### Service Startup

```bash
# Using systemd
sudo systemctl start arka-core
sudo systemctl start arka-rtvm
sudo systemctl start arka-api

# Using Docker Compose
docker-compose -f docker-compose.prod.yml up -d

# Using Kubernetes
kubectl apply -f k8s/production/
```

---

## Configuration Management

### Configuration Files

| File | Purpose | Location |
|------|---------|----------|
| `pact.config.yaml` | Core configuration | `/etc/pact/` |
| `rules.yaml` | Default rules | `/etc/pact/rules/` |
| `plugins.yaml` | Plugin configuration | `/etc/pact/plugins/` |
| `logging.yaml` | Log configuration | `/etc/pact/` |

### Core Configuration

```yaml
# /etc/pact/pact.config.yaml

core:
  instanceId: arka-prod-01
  environment: production
  timezone: UTC

rtvm:
  workers: auto              # Number of CPU cores
  maxConcurrency: 1000
  executionTimeout: 30000    # ms
  cacheEnabled: true
  cacheTtl: 3600             # seconds

api:
  port: 3000
  rateLimit:
    enabled: true
    windowMs: 60000
    max: 1000
  cors:
    enabled: true
    origins:
      - https://app.example.com
      - https://admin.example.com

database:
  pool:
    min: 10
    max: 50
  ssl: true
  statementTimeout: 30000

cache:
  provider: redis
  prefix: pact:
  defaultTtl: 3600

audit:
  enabled: true
  retentionDays: 2555       # 7 years
  hashChaining: true
```

### Runtime Configuration Updates

```bash
# Reload configuration without restart
arka-cli config reload

# Validate configuration
arka-cli config validate

# View current configuration
arka-cli config show

# Update specific setting
arka-cli config set rtvm.maxConcurrency 2000
```

---

## User & Access Management

### Role-Based Access Control (RBAC)

#### Default Roles

| Role | Permissions |
|------|-------------|
| `admin` | Full system access |
| `compliance_officer` | Rules, events, decisions, reports |
| `analyst` | Read-only access to events and decisions |
| `developer` | Plugin development, testing sandbox |
| `auditor` | Read-only access to audit logs |
| `api_user` | API access only |

#### Managing Users

```bash
# Create user
arka-cli user create \
  --email admin@example.com \
  --name "Admin User" \
  --role admin

# List users
arka-cli user list

# Update user role
arka-cli user update admin@example.com --role compliance_officer

# Disable user
arka-cli user disable admin@example.com

# Reset password
arka-cli user reset-password admin@example.com
```

#### API Key Management

```bash
# Generate API key
arka-cli apikey create \
  --name "Integration Key" \
  --permissions rules:read,events:write \
  --expires 365d

# List API keys
arka-cli apikey list

# Revoke API key
arka-cli apikey revoke key_xxxxxxxxxxxxx

# Rotate API key
arka-cli apikey rotate key_xxxxxxxxxxxxx
```

### Single Sign-On (SSO) Configuration

```yaml
# /etc/pact/auth.yaml

sso:
  enabled: true
  provider: saml2

  saml:
    entryPoint: https://idp.example.com/sso
    issuer: arka-engine
    cert: /etc/pact/certs/idp.crt
    attributeMapping:
      email: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress
      name: http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name
      groups: http://schemas.xmlsoap.org/claims/Group

  # Or OAuth2/OIDC
  oauth2:
    clientId: arka-client
    clientSecret: ${OAUTH_CLIENT_SECRET}
    authorizationUrl: https://auth.example.com/authorize
    tokenUrl: https://auth.example.com/token
    userInfoUrl: https://auth.example.com/userinfo
    scopes:
      - openid
      - profile
      - email
```

---

## Plugin Management

### Installing Plugins

```bash
# List available plugins
arka-cli plugin list --available

# Install plugin
arka-cli plugin install @arka/plugin-kyc-verification

# Install specific version
arka-cli plugin install @arka/plugin-aml-monitor@1.2.0

# Verify installation
arka-cli plugin verify @arka/plugin-kyc-verification
```

### Configuring Plugins

```yaml
# /etc/pact/plugins/kyc-verification.yaml

enabled: true
version: 1.0.0

config:
  realTimeVerification: true
  documentRetentionDays: 365
  highRiskCountries:
    - KP
    - IR
    - SY
  pepScreening:
    enabled: true
    dataSources:
      - dow_jones
      - world_check
```

### Plugin Operations

```bash
# Enable/disable plugin
arka-cli plugin enable @arka/plugin-kyc-verification
arka-cli plugin disable @arka/plugin-aml-monitor

# Update plugin
arka-cli plugin update @arka/plugin-kyc-verification

# View plugin status
arka-cli plugin status

# View plugin logs
arka-cli plugin logs @arka/plugin-kyc-verification --tail 100
```

---

## Monitoring & Alerting

### Health Endpoints

| Endpoint | Purpose | Response |
|----------|---------|----------|
| `/health` | Overall health | Status + dependencies |
| `/health/live` | Liveness probe | 200 OK if alive |
| `/health/ready` | Readiness probe | 200 OK if ready |
| `/metrics` | Prometheus metrics | Metrics data |

### Key Metrics to Monitor

```yaml
# Critical Metrics
- pact_events_processed_total        # Event throughput
- pact_rule_evaluations_total        # Rule executions
- pact_api_request_duration_seconds  # API latency
- pact_database_connections_active   # DB connections
- pact_cache_hit_rate                # Cache efficiency
- pact_error_rate                    # Error percentage

# Business Metrics
- pact_decisions_by_status           # Allow/Deny/Flag counts
- pact_alerts_generated_total        # Alert volume
- pact_rules_active_count            # Active rules

# Resource Metrics
- process_cpu_usage                  # CPU utilization
- process_memory_usage               # Memory usage
- go_goroutines                      # Concurrent operations
```

### Alerting Rules

```yaml
# /etc/pact/alerting.yaml

alerts:
  - name: HighErrorRate
    condition: pact_error_rate > 0.05
    duration: 5m
    severity: critical
    channels:
      - pagerduty
      - slack

  - name: HighLatency
    condition: pact_api_request_duration_seconds_p99 > 1
    duration: 10m
    severity: warning
    channels:
      - slack

  - name: DatabaseConnectionsHigh
    condition: pact_database_connections_active > 40
    duration: 5m
    severity: warning
    channels:
      - email

  - name: DiskSpaceLow
    condition: disk_usage_percent > 80
    duration: 15m
    severity: warning
    channels:
      - email
      - slack
```

### Setting Up Dashboards

```bash
# Import Grafana dashboards
./scripts/import-grafana-dashboards.sh

# Available dashboards:
# - ARKA Overview
# - API Performance
# - Rule Execution
# - Plugin Health
# - Database Performance
# - Security Events
```

---

## Backup & Recovery

### Backup Strategy

| Component | Frequency | Retention | Method |
|-----------|-----------|-----------|--------|
| Database | Hourly | 30 days | pg_dump + WAL |
| Configuration | On change | 90 days | Git |
| Audit Logs | Daily | 7 years | Archive to S3 |
| Encryption Keys | On change | Indefinite | HSM/Vault |

### Backup Commands

```bash
# Manual database backup
arka-cli backup database --output /backups/db-$(date +%Y%m%d).sql.gz

# Backup all configuration
arka-cli backup config --output /backups/config-$(date +%Y%m%d).tar.gz

# Backup audit logs
arka-cli backup audit-logs \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --output /backups/audit-2024-01.tar.gz

# Full system backup
arka-cli backup full --output /backups/full-$(date +%Y%m%d).tar.gz
```

### Automated Backups (Cron)

```bash
# /etc/cron../arka-backup

# Database backup every hour
0 * * * * pact /opt/pact/scripts/backup-database.sh

# Configuration backup daily
0 2 * * * pact /opt/pact/scripts/backup-config.sh

# Audit log backup weekly
0 3 * * 0 pact /opt/pact/scripts/backup-audit.sh

# Upload to S3 daily
0 4 * * * pact /opt/pact/scripts/upload-backups-s3.sh
```

### Recovery Procedures

```bash
# Restore database
arka-cli restore database --input /backups/db-20240115.sql.gz

# Restore configuration
arka-cli restore config --input /backups/config-20240115.tar.gz

# Point-in-time recovery
arka-cli restore database --pit "2024-01-15 14:30:00"

# Verify restore
arka-cli verify --full
```

---

## Scaling Operations

### Horizontal Scaling

```bash
# Scale ARKA Core instances
kubectl scale deployment arka-core --replicas=5

# Scale RTVM workers
kubectl scale deployment arka-rtvm --replicas=10

# Auto-scaling configuration
kubectl apply -f k8s/hpa.yaml
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: arka-core-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: arka-core
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Database Scaling

```bash
# Add read replica
arka-cli db add-replica --host replica2.example.com

# Promote replica to primary
arka-cli db promote-replica replica2.example.com

# Scale connection pool
arka-cli config set database.pool.max 100
arka-cli config reload
```

---

## Maintenance Tasks

### Routine Maintenance

```bash
# Database maintenance
arka-cli maintenance database --vacuum --analyze --reindex

# Clear old cache entries
arka-cli maintenance cache --clear-expired

# Archive old audit logs
arka-cli maintenance audit --archive --older-than 90d

# Clean up temporary files
arka-cli maintenance cleanup --temp --logs-older-than 30d
```

### Rolling Updates

```bash
# Check for updates
arka-cli update check

# Plan update
arka-cli update plan --version 1.1.0

# Execute rolling update
arka-cli update execute --version 1.1.0 --strategy rolling

# Rollback if needed
arka-cli update rollback --to 1.0.0
```

### Maintenance Windows

```bash
# Enable maintenance mode
arka-cli maintenance enable --message "Scheduled maintenance"

# Check maintenance status
arka-cli maintenance status

# Disable maintenance mode
arka-cli maintenance disable
```

---

## Security Operations

### Certificate Management

```bash
# Generate new certificates
arka-cli certs generate --domain api.example.com

# Renew certificates
arka-cli certs renew

# Rotate certificates
arka-cli certs rotate --zero-downtime

# Check certificate expiration
arka-cli certs check
```

### Key Rotation

```bash
# Rotate encryption keys
arka-cli keys rotate --type encryption

# Rotate JWT signing keys
arka-cli keys rotate --type jwt

# Rotate API master key
arka-cli keys rotate --type api-master
```

### Security Auditing

```bash
# Run security scan
arka-cli security scan

# Check for vulnerabilities
arka-cli security check-deps

# Review security events
arka-cli security events --last 24h

# Generate security report
arka-cli security report --output security-report.pdf
```

---

## Troubleshooting

### Common Issues

#### High Memory Usage

```bash
# Check memory usage
arka-cli diag memory

# Force garbage collection
arka-cli diag gc

# Restart with memory limits
arka-cli restart --memory-limit 8g
```

#### Database Connection Issues

```bash
# Check database connectivity
arka-cli diag database

# View connection pool status
arka-cli diag connections

# Reset connection pool
arka-cli diag reset-pool
```

#### Slow Rule Execution

```bash
# Profile rule execution
arka-cli diag profile-rules --rule-id rule-123

# Check RTVM status
arka-cli diag rtvm

# View execution queue
arka-cli diag queue
```

### Diagnostic Commands

```bash
# Full system diagnostic
arka-cli diag full --output diag-report.tar.gz

# Check service connectivity
arka-cli diag connectivity

# View system logs
arka-cli logs --service all --tail 1000

# Export debug bundle
arka-cli debug bundle --output debug-$(date +%Y%m%d).tar.gz
```

---

## Operational Runbooks

### Runbook: Emergency Restart

```bash
#!/bin/bash
# emergency-restart.sh

echo "1. Enable maintenance mode"
arka-cli maintenance enable --message "Emergency restart"

echo "2. Graceful shutdown"
arka-cli shutdown --graceful --timeout 60

echo "3. Verify shutdown"
arka-cli status

echo "4. Start services"
arka-cli start

echo "5. Verify health"
arka-cli health --wait --timeout 120

echo "6. Disable maintenance mode"
arka-cli maintenance disable

echo "7. Notify team"
arka-cli notify --channel ops --message "Emergency restart completed"
```

### Runbook: Database Failover

```bash
#!/bin/bash
# database-failover.sh

echo "1. Check current primary"
arka-cli db status

echo "2. Promote replica"
arka-cli db promote-replica replica1.example.com

echo "3. Update connection strings"
arka-cli config set database.host replica1.example.com

echo "4. Reload configuration"
arka-cli config reload

echo "5. Verify connectivity"
arka-cli diag database

echo "6. Notify team"
arka-cli notify --channel ops --message "Database failover completed"
```

### Runbook: Incident Response

```bash
#!/bin/bash
# incident-response.sh

echo "1. Capture current state"
arka-cli diag full --output /incidents/$(date +%Y%m%d-%H%M%S)/

echo "2. Check error logs"
arka-cli logs --level error --last 1h

echo "3. Check metrics anomalies"
arka-cli metrics anomalies --last 1h

echo "4. Generate incident report"
arka-cli incident report --output incident-report.md
```

---

## Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Check health | `arka-cli health` |
| View status | `arka-cli status` |
| View logs | `arka-cli logs --tail 100` |
| Restart service | `arka-cli restart` |
| Check config | `arka-cli config show` |
| Run diagnostics | `arka-cli diag full` |
| Backup database | `arka-cli backup database` |
| List users | `arka-cli user list` |
| Plugin status | `arka-cli plugin status` |

### Important Files

| File | Purpose |
|------|---------|
| `/etc/pact/pact.config.yaml` | Main configuration |
| `/var/log/pact/` | Log files |
| `/var/lib/pact/` | Data directory |
| `/etc/pact/plugins/` | Plugin configs |
| `/opt/pact/backups/` | Backup location |

### Support Contacts

| Issue Type | Contact |
|------------|---------|
| Critical outage | PagerDuty + Phone |
| Security incident | security@arka-engine.io |
| General support | support@arka-engine.io |
| Documentation | docs@arka-engine.io |

---

## Next Steps

1. Review [User Guide](./USER-GUIDE.md) for end-user documentation
2. Set up [Monitoring](./deployment/monitoring.md)
3. Configure [Backup Schedule](#backup--recovery)
4. Review [Security Model](./security/security-model.md)
