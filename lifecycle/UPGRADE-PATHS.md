# ARKA Engine Upgrade Paths

**Version:** 1.0.0
**Last Updated:** 2024-01-15
**Document Owner:** ARKA Engineering

---

## Table of Contents

1. [Upgrade Philosophy](#upgrade-philosophy)
2. [Upgrade Types](#upgrade-types)
3. [Pre-Upgrade Planning](#pre-upgrade-planning)
4. [Major Version Upgrades](#major-version-upgrades)
5. [Minor Version Upgrades](#minor-version-upgrades)
6. [Patch & Hotfix Upgrades](#patch--hotfix-upgrades)
7. [Upgrade Procedures](#upgrade-procedures)
8. [Rollback Procedures](#rollback-procedures)
9. [Troubleshooting](#troubleshooting)

---

## Upgrade Philosophy

### Core Principles

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ARKA Upgrade Principles                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   SAFETY FIRST                                                               │
│   ────────────                                                               │
│   • Always have a rollback plan                                              │
│   • Test in non-production first                                             │
│   • Backup before upgrading                                                  │
│   • Verify after each step                                                   │
│                                                                              │
│   MINIMAL DISRUPTION                                                         │
│   ─────────────────                                                          │
│   • Zero-downtime upgrades when possible                                     │
│   • Rolling deployments                                                      │
│   • Backward compatibility maintained                                        │
│   • Graceful degradation                                                     │
│                                                                              │
│   CLEAR COMMUNICATION                                                        │
│   ────────────────────                                                       │
│   • Documented upgrade paths                                                 │
│   • Breaking changes announced                                               │
│   • Migration guides provided                                                │
│   • Support available                                                        │
│                                                                              │
│   AUTOMATED VALIDATION                                                       │
│   ────────────────────                                                       │
│   • Pre-upgrade checks                                                       │
│   • Post-upgrade validation                                                  │
│   • Health monitoring                                                        │
│   • Automated rollback triggers                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Supported Upgrade Paths

```yaml
Direct Upgrade Paths:
  Within Major Version:
    - 1.0.x → 1.1.x → 1.2.x → 1.3.x  ✓ Sequential
    - 1.0.x → 1.3.x                   ✓ Skip minors OK
    - 1.0.x → any 1.x.x              ✓ Always supported

  Across Major Versions:
    - 1.x.x → 2.0.x                  ✓ With migration
    - 1.x.x → 2.1.x                  ✓ With migration
    - 1.x.x → 3.x.x                  ✗ Must go through 2.x

  Patch Versions:
    - 1.0.0 → 1.0.5                  ✓ Skip patches OK
    - Any patch → latest patch       ✓ Always supported

Unsupported Paths:
  - Skipping major versions (1.x → 3.x)
  - Downgrading major versions
  - Downgrading with data changes
```

---

## Upgrade Types

### Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Upgrade Type Comparison                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Type      │ Risk   │ Downtime │ Planning │ Testing  │ Rollback            │
│   ──────────┼────────┼──────────┼──────────┼──────────┼──────────────────── │
│   Hotfix    │ Low    │ None     │ Minimal  │ Basic    │ Immediate           │
│   Patch     │ Low    │ None     │ Minimal  │ Standard │ Same-day            │
│   Minor     │ Medium │ Minimal  │ 1-2 weeks│ Full     │ Within hours        │
│   Major     │ High   │ Planned  │ 4+ weeks │ Extensive│ Complex             │
│                                                                              │
│   Recommended Approach by Type:                                              │
│                                                                              │
│   Hotfix:   Apply immediately, verify, monitor                               │
│   Patch:    Apply within 1 week, standard testing                           │
│   Minor:    Plan 2-week window, full test cycle                             │
│   Major:    Plan 4-8 week project, comprehensive testing                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hotfix Upgrades

```yaml
Purpose: Emergency fixes for critical production issues
Risk Level: Low
Expected Downtime: None (rolling update)

Characteristics:
  - Single issue focus
  - Minimal code change
  - Extensively tested fix
  - Immediate deployment recommended

When to Apply:
  - Security vulnerabilities (CVE)
  - Data corruption risks
  - Service outages
  - Critical functionality broken

Upgrade Process:
  1. Review hotfix notes
  2. Apply to production directly (if approved)
  3. Or: Quick staging test → production
  4. Verify fix effectiveness
  5. Monitor for regressions

Example:
  Current: 1.0.2
  Hotfix:  1.0.2-hotfix.1
  Result:  1.0.3 (absorbed into next patch)
```

### Patch Upgrades (1.0.X)

```yaml
Purpose: Bug fixes and security updates
Risk Level: Low
Expected Downtime: None (rolling update)

Characteristics:
  - Bug fixes only
  - Security patches
  - No new features
  - Backward compatible
  - No configuration changes

When to Apply:
  - Monthly maintenance window
  - Security patch releases
  - Accumulated bug fixes
  - Performance improvements

Upgrade Process:
  1. Review release notes
  2. Test in development environment
  3. Test in staging environment
  4. Apply to production (rolling)
  5. Verify and monitor

Timeline:
  - Security patches: Within 1 week
  - Regular patches: Within 1 month
  - Can skip intermediate patches
```

### Minor Upgrades (1.X.0)

```yaml
Purpose: New features and enhancements
Risk Level: Medium
Expected Downtime: Minimal (minutes)

Characteristics:
  - New features added
  - Enhancements to existing features
  - Backward compatible APIs
  - May include configuration additions
  - Database migrations (non-breaking)

When to Apply:
  - Quarterly maintenance window
  - When new features are needed
  - Recommended within 3 months

Upgrade Process:
  1. Review release notes thoroughly
  2. Review new configuration options
  3. Test in development (1 week)
  4. Test in staging (1 week)
  5. Plan production upgrade window
  6. Execute with monitoring
  7. Verify all features

Pre-requisites:
  - On latest patch of current minor
  - All customizations tested
  - Integrations validated
  - Team trained on new features
```

### Major Upgrades (X.0.0)

```yaml
Purpose: Significant platform updates
Risk Level: High
Expected Downtime: Planned maintenance window

Characteristics:
  - Breaking API changes possible
  - Database schema changes
  - Configuration format changes
  - New architecture components
  - Extensive new features

When to Apply:
  - Annual major upgrade cycle
  - When current version approaches EOL
  - When major features are required

Upgrade Process:
  1. Read migration guide completely
  2. Assess impact on customizations
  3. Plan 4-8 week upgrade project
  4. Set up parallel environment
  5. Migrate configuration
  6. Migrate data
  7. Update integrations
  8. Comprehensive testing
  9. User acceptance testing
  10. Production cutover
  11. Post-upgrade validation

Pre-requisites:
  - On LTS or latest minor of current major
  - Full backup completed
  - Rollback plan documented
  - Extended maintenance window approved
  - Support team on standby
```

---

## Pre-Upgrade Planning

### Upgrade Assessment Checklist

```yaml
General Assessment:
  □ Current version documented
  □ Target version identified
  □ Release notes reviewed
  □ Breaking changes identified
  □ Migration guide reviewed
  □ Support timeline verified

Technical Assessment:
  □ Hardware requirements checked
  □ Software dependencies verified
  □ Database compatibility confirmed
  □ Network requirements reviewed
  □ Storage requirements assessed

Integration Assessment:
  □ Custom plugins inventoried
  □ Third-party integrations listed
  □ API usage analyzed
  □ Webhook configurations documented
  □ SSO/authentication impact assessed

Operational Assessment:
  □ Backup procedures verified
  □ Rollback plan documented
  □ Maintenance window scheduled
  □ Stakeholders notified
  □ Support contacts confirmed

Team Readiness:
  □ Team trained on changes
  □ Documentation updated
  □ Support escalation path clear
  □ Post-upgrade verification plan
```

### Environment Preparation

```yaml
Development Environment:
  Purpose: Initial testing and validation
  Data: Subset of production data (anonymized)
  Configuration: Production-like

  Preparation:
    1. Clone production configuration
    2. Load test data set
    3. Document current state
    4. Create baseline metrics

Staging Environment:
  Purpose: Full integration testing
  Data: Production data copy (anonymized)
  Configuration: Production mirror

  Preparation:
    1. Sync with production config
    2. Refresh data from production
    3. Verify all integrations connected
    4. Establish performance baseline

Production Environment:
  Purpose: Live system upgrade

  Preparation:
    1. Complete full backup
    2. Verify backup restoration works
    3. Document current configuration
    4. Capture baseline metrics
    5. Clear maintenance window
    6. Notify all stakeholders
```

### Backup Requirements

```yaml
Backup Scope:
  Database:
    - PostgreSQL full backup
    - Transaction logs
    - Point-in-time recovery enabled
    Retention: 30 days minimum

  Configuration:
    - All YAML/JSON config files
    - Environment variables
    - Secrets (encrypted)
    - Custom rules and policies
    Retention: Versioned in git

  Application Data:
    - Uploaded files/documents
    - Generated reports
    - Audit logs
    Retention: Per retention policy

  Infrastructure:
    - Kubernetes manifests
    - Helm values
    - Infrastructure as Code
    Retention: Versioned in git

Backup Commands:
  # Database backup
  pg_dump -Fc pact_db > pact_backup_$(date +%Y%m%d).dump

  # Configuration backup
  kubectl get configmap arka-config -o yaml > config_backup.yaml
  kubectl get secret arka-secrets -o yaml > secrets_backup.yaml

  # Full cluster state
  velero backup create pre-upgrade-$(date +%Y%m%d)

Verification:
  □ Backup completed successfully
  □ Backup size is reasonable
  □ Test restore in dev environment
  □ Verify data integrity
```

---

## Major Version Upgrades

### v1.x to v2.x Upgrade Guide

```yaml
Overview:
  Source: Any v1.x version (recommend v1.3.x LTS)
  Target: v2.0.x or later
  Estimated Duration: 4-8 hours (production)
  Downtime Required: Yes (30-60 minutes)

Breaking Changes in v2.0:
  API Changes:
    - REST API v1 deprecated (still functional)
    - New authentication method (OAuth 2.0 default)
    - Response format changes for some endpoints
    - Pagination default changed

  Database Changes:
    - Schema version 2.0
    - New tables for AI features
    - Index optimizations
    - Partitioning introduced

  Configuration Changes:
    - YAML format updated
    - New required fields
    - Deprecated fields removed
    - Environment variable renames

  Plugin API Changes:
    - Plugin SDK v2 required
    - Lifecycle hooks updated
    - Configuration schema changes
```

### Major Upgrade Procedure

```yaml
Phase 1: Preparation (Week 1-2)
  Day 1-3: Assessment
    □ Complete upgrade checklist
    □ Review all breaking changes
    □ Identify affected customizations
    □ Create upgrade project plan

  Day 4-7: Environment Setup
    □ Set up v2.x staging environment
    □ Configure parallel database
    □ Prepare migration scripts
    □ Set up monitoring

  Day 8-10: Development Testing
    □ Test upgrade in development
    □ Validate all integrations
    □ Test custom plugins
    □ Document issues found

Phase 2: Staging Validation (Week 3-4)
  Day 11-14: Staging Upgrade
    □ Execute upgrade in staging
    □ Run full test suite
    □ Performance testing
    □ Security validation

  Day 15-18: User Acceptance
    □ UAT with key users
    □ Validate business workflows
    □ Confirm reporting accuracy
    □ Sign-off from stakeholders

  Day 19-21: Final Preparation
    □ Finalize production plan
    □ Schedule maintenance window
    □ Notify all stakeholders
    □ Prepare rollback procedures

Phase 3: Production Upgrade (Week 5)
  Pre-Upgrade (Day before):
    □ Final backup completed
    □ Verify backup integrity
    □ Confirm team availability
    □ Send final notifications

  Upgrade Day:
    □ Begin maintenance window
    □ Stop application services
    □ Run database migration
    □ Deploy new application version
    □ Run post-upgrade scripts
    □ Verify core functionality
    □ End maintenance window
    □ Monitor closely

  Post-Upgrade (Following week):
    □ Monitor for issues
    □ Address any problems
    □ Collect user feedback
    □ Update documentation
    □ Close upgrade project
```

### Migration Script Example

```bash
#!/bin/bash
# ARKA v1.x to v2.x Migration Script
# Run this script in the production environment

set -e  # Exit on error

echo "=== ARKA v1.x to v2.x Migration ==="
echo "Started at: $(date)"

# Pre-flight checks
echo "Running pre-flight checks..."
./arka-migrate preflight --source-version 1.3.0 --target-version 2.0.0

if [ $? -ne 0 ]; then
    echo "Pre-flight checks failed. Aborting."
    exit 1
fi

# Backup database
echo "Creating database backup..."
pg_dump -Fc pact_db > /backups/pact_pre_v2_$(date +%Y%m%d_%H%M%S).dump

# Stop services
echo "Stopping ARKA services..."
kubectl scale deployment arka-api --replicas=0
kubectl scale deployment arka-rtvm --replicas=0
kubectl scale deployment arka-ai-governance --replicas=0

# Wait for graceful shutdown
sleep 30

# Run database migration
echo "Running database migration..."
./arka-migrate database \
    --source-version 1.3.0 \
    --target-version 2.0.0 \
    --connection-string "$DATABASE_URL"

# Migrate configuration
echo "Migrating configuration..."
./arka-migrate config \
    --input /config/v1/pact.yaml \
    --output /config/v2/pact.yaml

# Deploy new version
echo "Deploying ARKA v2.0..."
helm upgrade arka ./helm/pact \
    --set image.tag=2.0.0 \
    --values /config/v2/values.yaml

# Wait for rollout
kubectl rollout status deployme../arka-api --timeout=300s
kubectl rollout status deployme../arka-rtvm --timeout=300s
kubectl rollout status deployment/arka-ai-governance --timeout=300s

# Run post-migration validation
echo "Running post-migration validation..."
./arka-migrate validate --target-version 2.0.0

# Health check
echo "Performing health check..."
curl -f http://localhost:8080/health || exit 1

echo "=== Migration Complete ==="
echo "Completed at: $(date)"
```

---

## Minor Version Upgrades

### Minor Upgrade Procedure

```yaml
Overview:
  Risk Level: Medium
  Estimated Duration: 1-2 hours
  Downtime: Minimal (rolling update)

Pre-requisites:
  □ On latest patch of current minor
  □ Release notes reviewed
  □ Staging tested and validated
  □ Maintenance window scheduled

Upgrade Steps:

  Step 1: Preparation (30 minutes before)
    □ Verify current system health
    □ Confirm backup is current
    □ Notify stakeholders
    □ Prepare rollback command

  Step 2: Staging Final Check
    # Verify staging is on target version
    kubectl get deployment -l app=pact -o jsonpath='{.items[*].spec.template.spec.containers[*].image}'

  Step 3: Production Upgrade
    # Option A: Helm upgrade
    helm upgrade arka ./helm/pact \
      --set image.tag=1.2.0 \
      --wait \
      --timeout 10m

    # Option B: Kubectl rolling update
    kubectl set image deployme../arka-api \
      arka-api=arka-engine/api:1.2.0
    kubectl rollout status deployme../arka-api

  Step 4: Verification
    □ Health endpoints responding
    □ Core functionality working
    □ Integrations connected
    □ No error spikes in logs

  Step 5: Post-Upgrade
    □ Monitor for 1 hour
    □ Confirm with stakeholders
    □ Update documentation
    □ Close maintenance window
```

### Zero-Downtime Minor Upgrade

```yaml
Rolling Update Strategy:
  maxUnavailable: 0
  maxSurge: 1

Process:
  1. New pod created with new version
  2. New pod passes health checks
  3. New pod added to load balancer
  4. Old pod removed from load balancer
  5. Old pod terminated
  6. Repeat for all pods

Kubernetes Configuration:
  apiVersion: apps/v1
  kind: Deployment
  spec:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 0
        maxSurge: 1
    template:
      spec:
        containers:
        - name: arka-api
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10

Canary Deployment (Optional):
  1. Deploy new version to 10% of pods
  2. Monitor metrics for 15 minutes
  3. If healthy, continue rollout
  4. If issues, rollback immediately
```

---

## Patch & Hotfix Upgrades

### Patch Upgrade Procedure

```yaml
Overview:
  Risk Level: Low
  Estimated Duration: 15-30 minutes
  Downtime: None (rolling update)

Quick Upgrade Commands:

  # Check current version
  kubectl get deployment arka-api -o jsonpath='{.spec.template.spec.containers[0].image}'

  # Apply patch upgrade
  kubectl set image deployme../arka-api arka-api=arka-engine/api:1.0.5
  kubectl set image deployme../arka-rtvm arka-rtvm=arka-engine/rtvm:1.0.5
  kubectl set image deployment/arka-ai-governance arka-ai=arka-engine/ai-governance:1.0.5

  # Monitor rollout
  kubectl rollout status deployme../arka-api
  kubectl rollout status deployme../arka-rtvm
  kubectl rollout status deployment/arka-ai-governance

  # Verify version
  curl -s http://localhost:8080/version | jq .

Verification Checklist:
  □ Version endpoint shows new version
  □ Health check passes
  □ No error increase in logs
  □ Core API responding
  □ Metrics normal
```

### Hotfix Procedure

```yaml
Overview:
  Risk Level: Low (but urgent)
  Timeline: Apply immediately
  Downtime: None

Emergency Hotfix Process:

  Step 1: Assess (5 minutes)
    □ Review hotfix announcement
    □ Confirm affects your deployment
    □ Understand the fix

  Step 2: Apply (10 minutes)
    # Pull hotfix image
    docker pull arka-engine/api:1.0.2-hotfix.1

    # Apply to production
    kubectl set image deployme../arka-api \
      arka-api=arka-engine/api:1.0.2-hotfix.1

    # Monitor rollout
    kubectl rollout status deployme../arka-api

  Step 3: Verify (5 minutes)
    □ Confirm fix is effective
    □ Verify no regressions
    □ Document application

  Step 4: Follow-up
    □ Update to next patch when available
    □ Report any issues
    □ Update change records

Hotfix Communication:
  - Check: status.arka-engine.io
  - Subscribe: security@arka-engine.io
  - Emergency: +1 (555) 123-4567 x999
```

---

## Upgrade Procedures

### Kubernetes Upgrade Procedure

```yaml
Prerequisites:
  - kubectl configured
  - Helm installed (if using Helm)
  - Cluster admin access
  - Backup completed

Method 1: Helm Upgrade
  # Add/update repo
  helm repo add pact https://helm.arka-engine.io
  helm repo update

  # Review changes
  helm diff upgrade arka pact/arka-engine \
    --version 1.2.0 \
    --values values.yaml

  # Execute upgrade
  helm upgrade arka pact/arka-engine \
    --version 1.2.0 \
    --values values.yaml \
    --wait \
    --timeout 15m

  # Verify
  helm status pact
  kubectl get pods -l app=pact

Method 2: kubectl Apply
  # Update manifests
  sed -i 's/image: arka-engine\/api:.*/image: arka-engine\/api:1.2.0/' deployment.yaml

  # Apply changes
  kubectl apply -f deployment.yaml

  # Monitor rollout
  kubectl rollout status deployme../arka-api

Method 3: ArgoCD (GitOps)
  # Update version in git
  git checkout main
  sed -i 's/tag: .*/tag: 1.2.0/' helm/values.yaml
  git commit -am "Upgrade ARKA to v1.2.0"
  git push

  # ArgoCD syncs automatically or manually
  argocd app sync pact
  argocd app wait pact
```

### Docker Compose Upgrade

```yaml
Prerequisites:
  - Docker Compose installed
  - Backup completed
  - docker-compose.yml updated

Procedure:
  # Navigate to deployment directory
  cd /opt/pact

  # Update docker-compose.yml with new version
  # Or use environment variable
  export ARKA_VERSION=1.2.0

  # Pull new images
  docker-compose pull

  # Stop and start with new version
  docker-compose down
  docker-compose up -d

  # Verify
  docker-compose ps
  docker-compose logs -f --tail=100

  # Health check
  curl http://localhost:8080/health

Rolling Update (if supported):
  # Scale up new version
  docker-compose up -d --scale arka-api=2 --no-recreate

  # Verify new container healthy
  docker-compose ps

  # Remove old container
  docker stop arka-api-1
  docker rm arka-api-1

  # Scale back
  docker-compose up -d --scale arka-api=1 --no-recreate
```

### Database Migration

```yaml
Migration Types:
  Schema Migration:
    - Table structure changes
    - Index modifications
    - Constraint updates

  Data Migration:
    - Data transformation
    - Data backfill
    - Reference updates

Migration Process:
  # Check pending migrations
 ../arka-cli db migrate status

  # Backup database
  pg_dump -Fc pact_db > pre_migration_backup.dump

  # Run migrations
 ../arka-cli db migrate up

  # Verify migration
 ../arka-cli db migrate verify

  # If issues, rollback
 ../arka-cli db migrate down --steps 1

Migration Best Practices:
  - Always backup before migration
  - Test migrations in staging first
  - Run during low-traffic periods
  - Monitor database performance
  - Have rollback scripts ready
```

---

## Rollback Procedures

### Rollback Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Rollback Decision Matrix                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Issue Severity      │ Resolution Time   │ Action                          │
│   ────────────────────┼───────────────────┼──────────────────────────────── │
│   Critical (P1)       │ > 15 minutes      │ Immediate rollback              │
│   - Service down      │                   │                                 │
│   - Data corruption   │                   │                                 │
│   ────────────────────┼───────────────────┼──────────────────────────────── │
│   High (P2)           │ > 30 minutes      │ Rollback recommended            │
│   - Major feature     │                   │                                 │
│     broken            │                   │                                 │
│   ────────────────────┼───────────────────┼──────────────────────────────── │
│   Medium (P3)         │ > 2 hours         │ Consider rollback               │
│   - Minor feature     │                   │                                 │
│     issues            │                   │                                 │
│   ────────────────────┼───────────────────┼──────────────────────────────── │
│   Low (P4)            │ N/A               │ Fix forward                     │
│   - Cosmetic issues   │                   │                                 │
│                                                                              │
│   Automatic Rollback Triggers:                                               │
│   • Health check failures > 3 consecutive                                   │
│   • Error rate > 5% for 5 minutes                                          │
│   • Latency p99 > 2x baseline for 5 minutes                                │
│   • Memory/CPU > 90% for 5 minutes                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Application Rollback

```yaml
Kubernetes Rollback:
  # Check rollout history
  kubectl rollout history deployme../arka-api

  # Rollback to previous version
  kubectl rollout undo deployme../arka-api

  # Rollback to specific revision
  kubectl rollout undo deployme../arka-api --to-revision=2

  # Monitor rollback
  kubectl rollout status deployme../arka-api

  # Verify
  kubectl get pods -l app=arka-api

Helm Rollback:
  # Check release history
  helm history pact

  # Rollback to previous release
  helm rollback pact

  # Rollback to specific revision
  helm rollback pact 3

  # Verify
  helm status pact

Docker Compose Rollback:
  # Update docker-compose.yml to previous version
  export ARKA_VERSION=1.1.0

  # Restart with old version
  docker-compose down
  docker-compose up -d

  # Verify
  docker-compose ps
```

### Database Rollback

```yaml
Schema Rollback:
  # Check migration status
 ../arka-cli db migrate status

  # Rollback last migration
 ../arka-cli db migrate down --steps 1

  # Rollback to specific version
 ../arka-cli db migrate down --to-version 20240101120000

  # Verify schema
 ../arka-cli db migrate verify

Full Database Restore:
  # Stop application
  kubectl scale deployment arka-api --replicas=0

  # Restore from backup
  pg_restore -d pact_db -c /backups/pre_upgrade_backup.dump

  # Verify restore
  psql pact_db -c "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 1;"

  # Restart application with old version
  kubectl set image deployme../arka-api arka-api=arka-engine/api:1.1.0
  kubectl scale deployment arka-api --replicas=3

Point-in-Time Recovery:
  # Identify target time (before corruption)
  TARGET_TIME="2024-01-15 14:30:00"

  # Stop application
  kubectl scale deployment arka-api --replicas=0

  # Restore to point in time
  pg_restore --target-time="$TARGET_TIME" \
    -d pact_db_restored \
    /backups/base_backup.tar

  # Verify data integrity
  # Switch application to restored database
```

---

## Troubleshooting

### Common Upgrade Issues

```yaml
Issue: Pods stuck in CrashLoopBackOff
  Symptoms:
    - Pods repeatedly restarting
    - Container failing health checks

  Diagnosis:
    kubectl describe pod <pod-name>
    kubectl logs <pod-name> --previous

  Common Causes:
    - Configuration error
    - Missing environment variable
    - Database connection failure
    - Resource limits too low

  Resolution:
    - Check configuration for typos
    - Verify secrets are present
    - Check database connectivity
    - Increase resource limits

Issue: Database migration failed
  Symptoms:
    - Migration command errors
    - Application won't start

  Diagnosis:
   ../arka-cli db migrate status
    psql pact_db -c "SELECT * FROM schema_migrations;"

  Common Causes:
    - Insufficient permissions
    - Disk space full
    - Lock contention
    - Schema conflicts

  Resolution:
    - Grant necessary permissions
    - Free disk space
    - Retry during low activity
    - Manual schema correction

Issue: Integration failures after upgrade
  Symptoms:
    - Webhook delivery failures
    - SSO authentication errors
    - API calls returning errors

  Diagnosis:
    - Check integration logs
    - Verify API compatibility
    - Test authentication flow

  Common Causes:
    - API version mismatch
    - Authentication changes
    - Configuration drift

  Resolution:
    - Update integration to new API version
    - Reconfigure authentication
    - Sync configuration

Issue: Performance degradation
  Symptoms:
    - Increased latency
    - Higher resource usage
    - Timeout errors

  Diagnosis:
    - Check metrics dashboards
    - Review slow query logs
    - Profile application

  Common Causes:
    - New feature overhead
    - Missing indexes
    - Configuration not optimized

  Resolution:
    - Tune configuration
    - Add missing indexes
    - Scale resources
    - Contact support if persistent
```

### Upgrade Support

```yaml
Self-Service Resources:
  Documentation:
    - docs.arka-engine.io/upgrade
    - docs.arka-engine.io/troubleshooting

  Knowledge Base:
    - kb.arka-engine.io
    - Search for specific error codes

  Community:
    - community.arka-engine.io
    - Search similar issues

Support Channels:
  Standard Support:
    - Email: support@arka-engine.io
    - Portal: support.arka-engine.io
    - Hours: Business hours

  Premium Support:
    - Phone: +1 (555) 123-4567
    - Chat: In-app support
    - Hours: 24/7

  Emergency Support:
    - Hotline: +1 (555) 123-4567 x999
    - For P1 incidents only
    - Available 24/7

Information to Provide:
  - Current version
  - Target version
  - Error messages (full)
  - Steps to reproduce
  - Environment details
  - Recent changes
```

---

**Document Revision History**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2024-01-15 | Engineering Team | Initial release |
