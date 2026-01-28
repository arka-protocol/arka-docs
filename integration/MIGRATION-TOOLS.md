# ARKA Engine Migration & Import Tools

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Integration

---

## Overview

This guide covers migrating from legacy compliance systems to ARKA Engine, including data import tools, rule conversion utilities, and integration migration strategies.

---

## Table of Contents

1. [Migration Overview](#migration-overview)
2. [Supported Source Systems](#supported-source-systems)
3. [Migration CLI Tool](#migration-cli-tool)
4. [Data Import](#data-import)
5. [Rule Migration](#rule-migration)
6. [Configuration Migration](#configuration-migration)
7. [Audit History Import](#audit-history-import)
8. [Integration Migration](#integration-migration)
9. [Validation & Testing](#validation--testing)
10. [Rollback Procedures](#rollback-procedures)

---

## Migration Overview

### Migration Phases

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        MIGRATION LIFECYCLE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Phase 1         Phase 2         Phase 3         Phase 4         Phase 5   │
│   ────────        ────────        ────────        ────────        ────────  │
│   ASSESS          PLAN            MIGRATE         VALIDATE        CUTOVER   │
│                                                                              │
│   • Inventory     • Map rules     • Export        • Parallel      • DNS     │
│   • Analyze       • Map data      • Transform     • Compare       • Switch  │
│   • Estimate      • Schedule      • Import        • Test          • Monitor │
│   • Risks         • Resources     • Configure     • Fix issues    • Support │
│                                                                              │
│   1-2 weeks       1-2 weeks       2-4 weeks       1-2 weeks       1 day     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pre-Migration Checklist

```yaml
pre_migration_checklist:
  assessment:
    - [ ] Inventory all existing rules
    - [ ] Document current integrations
    - [ ] List all data sources
    - [ ] Identify customizations
    - [ ] Review audit requirements
    - [ ] Assess downtime tolerance

  planning:
    - [ ] Create migration timeline
    - [ ] Assign migration team
    - [ ] Prepare test environment
    - [ ] Define success criteria
    - [ ] Create rollback plan
    - [ ] Schedule stakeholder reviews

  technical:
    - [ ] Provision ARKA Engine environment
    - [ ] Configure network access
    - [ ] Set up authentication
    - [ ] Prepare data export from source
    - [ ] Test migration tools
    - [ ] Validate connectivity
```

---

## Supported Source Systems

### Compliance Platforms

| Source System | Migration Support | Data Types |
|---------------|-------------------|------------|
| Actimize | Full | Rules, Cases, Alerts |
| NICE Actimize | Full | Rules, Watchlists, SAR |
| Oracle FCCM | Full | Rules, Alerts, Cases |
| SAS AML | Full | Rules, Alerts, Models |
| Quantexa | Partial | Entities, Networks |
| Refinitiv World-Check | Full | Watchlists, Screening |
| Accuity (LexisNexis) | Full | Watchlists, PEP data |
| Dow Jones Risk | Full | Watchlists, Adverse Media |
| Custom/Legacy | Configurable | Via adapters |

### Data Formats

| Format | Import | Export | Notes |
|--------|--------|--------|-------|
| JSON | ✅ | ✅ | Primary format |
| XML | ✅ | ✅ | With schema validation |
| CSV | ✅ | ✅ | For tabular data |
| Parquet | ✅ | ✅ | Large datasets |
| Avro | ✅ | ✅ | Schema evolution |
| SQL Dump | ✅ | - | PostgreSQL, MySQL |
| Excel | ✅ | ✅ | .xlsx format |

---

## Migration CLI Tool

### Installation

```bash
# npm
npm install -g @arka-engine/migration-cli

# Download binary
curl -sSL https://releases.arka-engine.io/migration-cli/latest/install.sh | sh

# Docker
docker pull arka-engine/migration-cli:latest
```

### Configuration

```yaml
# migration-config.yaml
source:
  type: "actimize"
  connection:
    host: "legacy-system.example.com"
    port: 5432
    database: "actimize_db"
    username: "${SOURCE_DB_USER}"
    password: "${SOURCE_DB_PASSWORD}"
    ssl: true

target:
  type: "arka-engine"
  connection:
    apiUrl: "https://api.arka-engine.io"
    apiKey: "${ARKA_API_KEY}"
    tenantId: "your-tenant-id"

migration:
  batchSize: 1000
  parallelism: 4
  dryRun: false
  validateOnly: false
  continueOnError: false

  rules:
    enabled: true
    mapping: "./mappings/rules.yaml"

  data:
    enabled: true
    types:
      - customers
      - transactions
      - watchlists
      - alerts

  audit:
    enabled: true
    startDate: "2020-01-01"
    endDate: "2024-12-01"

logging:
  level: "info"
  format: "json"
  file: "./logs/migration.log"
```

### CLI Commands

```bash
# Assess source system
arka-migrate assess --config migration-config.yaml

# Generate migration plan
arka-migrate plan --config migration-config.yaml --output migration-plan.json

# Run migration
arka-migrate run --config migration-config.yaml

# Validate migration
arka-migrate validate --config migration-config.yaml

# Generate report
arka-migrate report --config migration-config.yaml --format pdf

# Rollback
arka-migrate rollback --config migration-config.yaml --to-checkpoint cp_abc123
```

### Assessment Output

```bash
$ arka-migrate assess --config migration-config.yaml

╔══════════════════════════════════════════════════════════════════╗
║                    MIGRATION ASSESSMENT REPORT                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  Source System: Actimize AML v8.2                                ║
║  Assessment Date: 2024-12-01T10:00:00Z                           ║
║                                                                   ║
╠══════════════════════════════════════════════════════════════════╣
║  RULES INVENTORY                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  Total Rules: 247                                                 ║
║  ├─ Active: 198                                                  ║
║  ├─ Inactive: 34                                                 ║
║  └─ Draft: 15                                                    ║
║                                                                   ║
║  By Category:                                                     ║
║  ├─ AML/Transaction Monitoring: 89                               ║
║  ├─ Sanctions Screening: 45                                      ║
║  ├─ KYC/Due Diligence: 38                                        ║
║  ├─ Fraud Detection: 52                                          ║
║  └─ Custom: 23                                                   ║
║                                                                   ║
║  Migration Complexity:                                            ║
║  ├─ Direct Mapping: 187 (76%)                                    ║
║  ├─ Requires Transformation: 45 (18%)                            ║
║  └─ Manual Review: 15 (6%)                                       ║
║                                                                   ║
╠══════════════════════════════════════════════════════════════════╣
║  DATA INVENTORY                                                   ║
╠══════════════════════════════════════════════════════════════════╣
║  Customers: 1,245,678                                             ║
║  Transactions: 45,678,901                                         ║
║  Alerts: 234,567                                                  ║
║  Cases: 12,345                                                    ║
║  Watchlist Entries: 2,345,678                                     ║
║                                                                   ║
║  Estimated Data Size: 2.3 TB                                      ║
║  Estimated Migration Time: 8-12 hours                             ║
║                                                                   ║
╠══════════════════════════════════════════════════════════════════╣
║  RECOMMENDATIONS                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  1. Review 15 rules flagged for manual conversion                ║
║  2. Schedule migration during low-activity period                 ║
║  3. Plan parallel run for 2 weeks                                 ║
║  4. Allocate 4 team members for validation                        ║
║                                                                   ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Data Import

### Customer Data Import

```bash
# Import customers from CSV
arka-migrate import customers \
  --source customers.csv \
  --mapping ./mappings/customers.yaml \
  --batch-size 5000

# Import from database
arka-migrate import customers \
  --source "postgresql://host/db" \
  --query "SELECT * FROM customers WHERE status = 'active'" \
  --mapping ./mappings/customers.yaml
```

**Mapping Configuration:**

```yaml
# mappings/customers.yaml
source:
  type: csv
  encoding: utf-8
  delimiter: ","
  hasHeader: true

mapping:
  target: pact.customer
  fields:
    - source: customer_id
      target: externalId
      required: true

    - source: full_name
      target: name
      transform: trim

    - source: customer_type
      target: type
      transform:
        mapping:
          "IND": "individual"
          "BUS": "business"
          "FIN": "financial_institution"

    - source: country_code
      target: country
      transform: uppercase

    - source: risk_rating
      target: riskScore
      transform:
        type: number
        scale: 100

    - source: onboarding_date
      target: createdAt
      transform:
        type: date
        format: "YYYY-MM-DD"

    - source: [street, city, state, zip, country]
      target: address
      transform:
        type: object
        template: |
          {
            "street": "${street}",
            "city": "${city}",
            "state": "${state}",
            "postalCode": "${zip}",
            "country": "${country}"
          }

validation:
  rules:
    - field: externalId
      type: required
    - field: country
      type: iso3166
    - field: riskScore
      type: range
      min: 0
      max: 100

  onError: skip_and_log
```

### Transaction Data Import

```yaml
# mappings/transactions.yaml
source:
  type: parquet
  partitions:
    - date
    - transaction_type

mapping:
  target: pact.transaction
  fields:
    - source: txn_id
      target: id
      required: true

    - source: txn_type
      target: type
      transform:
        mapping:
          "WT": "wire_transfer"
          "ACH": "ach_transfer"
          "CC": "card_payment"
          "CASH": "cash_deposit"

    - source: amount
      target: amount
      transform:
        type: decimal
        precision: 2

    - source: currency_code
      target: currency
      transform: uppercase

    - source: sender_id
      target: sender.id
      lookup:
        table: customers
        sourceField: customer_id
        targetField: pactCustomerId

    - source: recipient_id
      target: recipient.id
      lookup:
        table: customers
        sourceField: customer_id
        targetField: pactCustomerId

    - source: txn_timestamp
      target: timestamp
      transform:
        type: datetime
        sourceTimezone: "America/New_York"
        targetTimezone: "UTC"

deduplication:
  enabled: true
  key: [id, timestamp]
  strategy: keep_latest
```

### Watchlist Import

```bash
# Import OFAC SDN list
arka-migrate import watchlist \
  --source ofac_sdn.xml \
  --type sanctions \
  --provider OFAC \
  --format ofac-xml

# Import custom watchlist
arka-migrate import watchlist \
  --source internal_watchlist.csv \
  --type internal \
  --mapping ./mappings/watchlist.yaml
```

```yaml
# mappings/watchlist.yaml
source:
  type: csv

mapping:
  target: pact.watchlist.entry
  fields:
    - source: entity_id
      target: externalId

    - source: entity_name
      target: name

    - source: entity_type
      target: type
      transform:
        mapping:
          "P": "individual"
          "E": "entity"
          "V": "vessel"
          "A": "aircraft"

    - source: aliases
      target: aliases
      transform:
        type: array
        delimiter: "|"

    - source: dob
      target: dateOfBirth
      transform:
        type: date
        format: "MM/DD/YYYY"

    - source: countries
      target: countries
      transform:
        type: array
        delimiter: ","

    - source: programs
      target: programs
      transform:
        type: array
        delimiter: ";"

    - source: identifiers
      target: identifiers
      transform:
        type: json

metadata:
  listType: "sanctions"
  source: "internal"
  effectiveDate: "2024-12-01"
```

---

## Rule Migration

### Rule Conversion

```bash
# Convert rules from source format
arka-migrate rules convert \
  --source ./legacy-rules/ \
  --format actimize \
  --output../arka-rules/ \
  --mapping ./mappings/rules.yaml

# Validate converted rules
arka-migrate rules validate \
  --rules../arka-rules/ \
  --strict

# Import rules to ARKA Engine
arka-migrate rules import \
  --rules../arka-rules/ \
  --activate false
```

### Rule Mapping Configuration

```yaml
# mappings/rules.yaml
source:
  format: actimize
  version: "8.2"

conversion:
  # Operator mapping
  operators:
    "EQUALS": "eq"
    "NOT_EQUALS": "neq"
    "GREATER_THAN": "gt"
    "LESS_THAN": "lt"
    "CONTAINS": "contains"
    "IN_LIST": "in"
    "REGEX_MATCH": "matches"

  # Field mapping
  fields:
    "TXN_AMOUNT": "transaction.amount"
    "TXN_TYPE": "transaction.type"
    "SENDER_COUNTRY": "transaction.sender.country"
    "RECIPIENT_COUNTRY": "transaction.recipient.country"
    "CUSTOMER_RISK": "customer.riskScore"
    "ACCOUNT_AGE_DAYS": "customer.accountAgeDays"

  # Action mapping
  actions:
    "ALERT": { action: "flag", severity: "medium" }
    "BLOCK": { action: "block", severity: "high" }
    "REVIEW": { action: "review", severity: "medium" }
    "REPORT": { action: "report", reportType: "sar" }

  # Category mapping
  categories:
    "AML_TM": "aml.transaction_monitoring"
    "SANCTIONS": "sanctions.screening"
    "KYC": "kyc.due_diligence"
    "FRAUD": "fraud.detection"

templates:
  # Template for threshold rules
  threshold:
    input: |
      IF ${FIELD} ${OPERATOR} ${VALUE} THEN ${ACTION}
    output: |
      {
        "type": "condition",
        "condition": {
          "field": "${field}",
          "operator": "${operator}",
          "value": ${value}
        },
        "actions": ${actions}
      }

  # Template for velocity rules
  velocity:
    input: |
      IF COUNT(${FIELD}) IN LAST ${PERIOD} ${OPERATOR} ${VALUE}
    output: |
      {
        "type": "velocity",
        "metric": {
          "field": "${field}",
          "aggregation": "count",
          "window": "${period}"
        },
        "condition": {
          "operator": "${operator}",
          "value": ${value}
        }
      }
```

### Example Rule Conversion

**Source (Actimize format):**
```xml
<Rule id="AML_HVT_001" name="High Value Transaction Alert">
  <Category>AML_TM</Category>
  <Severity>HIGH</Severity>
  <Condition>
    <Field>TXN_AMOUNT</Field>
    <Operator>GREATER_THAN</Operator>
    <Value>10000</Value>
    <Currency>USD</Currency>
  </Condition>
  <Action type="ALERT">
    <Message>High value transaction detected</Message>
  </Action>
</Rule>
```

**Output (ARKA format):**
```json
{
  "id": "rule_aml_hvt_001",
  "name": "High Value Transaction Alert",
  "category": "aml.transaction_monitoring",
  "severity": "high",
  "version": "1.0.0",
  "status": "draft",
  "definition": {
    "type": "condition",
    "condition": {
      "field": "transaction.amount",
      "operator": "gt",
      "value": 10000
    },
    "actions": [
      {
        "type": "flag",
        "severity": "high",
        "message": "High value transaction detected"
      }
    ]
  },
  "metadata": {
    "sourceSystem": "actimize",
    "sourceRuleId": "AML_HVT_001",
    "migratedAt": "2024-12-01T10:00:00Z"
  }
}
```

---

## Configuration Migration

### System Configuration

```yaml
# Export from legacy system
arka-migrate config export \
  --source legacy \
  --output ./legacy-config/

# Transform configuration
arka-migrate config transform \
  --input ./legacy-config/ \
  --mapping ./mappings/config.yaml \
  --output../arka-config/

# Import to ARKA Engine
arka-migrate config import \
  --config../arka-config/
```

### Configuration Mapping

```yaml
# mappings/config.yaml
thresholds:
  source: threshold_config.xml
  mapping:
    "AML_DAILY_LIMIT": "compliance.aml.dailyLimit"
    "CTR_THRESHOLD": "compliance.aml.ctrThreshold"
    "PEP_ENHANCED_DD": "compliance.kyc.pepEnhancedDueDiligence"

alerts:
  source: alert_config.xml
  mapping:
    escalation_levels:
      "L1": { severity: "low", sla: 48 }
      "L2": { severity: "medium", sla: 24 }
      "L3": { severity: "high", sla: 8 }
      "L4": { severity: "critical", sla: 2 }

    notification_channels:
      "EMAIL": { type: "email", enabled: true }
      "SMS": { type: "sms", enabled: true }
      "SLACK": { type: "webhook", provider: "slack" }

users:
  source: user_export.csv
  mapping:
    roles:
      "ADMIN": "admin"
      "COMPLIANCE_MGR": "compliance_manager"
      "ANALYST": "compliance_analyst"
      "AUDITOR": "auditor"
      "VIEWER": "viewer"
```

---

## Audit History Import

### Audit Data Import

```bash
# Import historical audit logs
arka-migrate audit import \
  --source ./audit-export/ \
  --format json \
  --start-date 2020-01-01 \
  --end-date 2024-12-01 \
  --batch-size 10000
```

### Audit Mapping

```yaml
# mappings/audit.yaml
source:
  type: json
  directory: ./audit-export/

mapping:
  target: pact.audit.entry
  fields:
    - source: audit_id
      target: externalId

    - source: event_type
      target: action
      transform:
        mapping:
          "LOGIN": "user.login"
          "RULE_CREATE": "rule.created"
          "RULE_UPDATE": "rule.updated"
          "ALERT_CREATE": "alert.created"
          "CASE_CLOSE": "case.closed"

    - source: timestamp
      target: timestamp
      transform:
        type: datetime
        format: "YYYY-MM-DD HH:mm:ss"

    - source: user_id
      target: actor.id
      lookup:
        table: users
        sourceField: legacy_user_id
        targetField: pactUserId

    - source: entity_type
      target: resource.type

    - source: entity_id
      target: resource.id

    - source: details
      target: changes
      transform:
        type: json

    - source: ip_address
      target: context.ipAddress

    - source: user_agent
      target: context.userAgent

integrity:
  preserveHashes: true
  verifyChain: true
  anchorToBlockchain: false  # Historical data not anchored
```

---

## Integration Migration

### API Integration Migration

```yaml
# integration-migration.yaml
integrations:
  - name: "core-banking"
    type: api
    source:
      baseUrl: "https://legacy.bank.com/api"
      auth:
        type: basic
        credentials: "${LEGACY_BANKING_CREDS}"
    target:
      type: webhook
      events:
        - transaction.created
        - customer.updated
      endpoint: "https://api.arka-engine.io/webhooks/banking"

  - name: "sanctions-screening"
    type: batch
    source:
      sftp:
        host: "sftp.sanctions-provider.com"
        path: "/daily-updates/"
    target:
      type: scheduled_import
      schedule: "0 6 * * *"  # Daily at 6 AM
      watchlistType: "sanctions"

  - name: "reporting-system"
    type: database
    source:
      connection: "oracle://legacy-reporting:1521/reports"
    target:
      type: api
      endpoints:
        reports: "GET /api/v1/reports"
        export: "POST /api/v1/reports/export"
```

### Webhook Migration

```bash
# Export webhook configurations
arka-migrate webhooks export \
  --source legacy \
  --output ./webhooks-config.json

# Transform and import
arka-migrate webhooks import \
  --config ./webhooks-config.json \
  --test-delivery
```

---

## Validation & Testing

### Migration Validation

```bash
# Run comprehensive validation
arka-migrate validate \
  --config migration-config.yaml \
  --report validation-report.json

# Compare source and target
arka-migrate compare \
  --source legacy \
  --target arka-engine \
  --entities rules,customers,transactions \
  --sample-size 10000
```

### Validation Report

```json
{
  "validationId": "val_abc123",
  "timestamp": "2024-12-01T15:00:00Z",
  "status": "passed_with_warnings",
  "summary": {
    "rulesValidated": 247,
    "rulesPassed": 245,
    "rulesFailed": 2,
    "customersValidated": 1245678,
    "customersPassed": 1245650,
    "customersFailed": 28,
    "transactionsValidated": 45678901,
    "transactionsPassed": 45678890,
    "transactionsFailed": 11
  },
  "issues": [
    {
      "type": "rule_conversion",
      "severity": "error",
      "ruleId": "legacy_rule_123",
      "message": "Unable to convert complex condition expression",
      "recommendation": "Manual conversion required"
    },
    {
      "type": "data_validation",
      "severity": "warning",
      "entity": "customer",
      "count": 28,
      "message": "Missing required field: country",
      "recommendation": "Set default or review source data"
    }
  ],
  "dataIntegrity": {
    "checksumMatch": true,
    "recordCountMatch": true,
    "sampleComparison": {
      "tested": 10000,
      "matched": 9998,
      "mismatched": 2
    }
  }
}
```

### Parallel Running

```yaml
# parallel-run-config.yaml
parallelRun:
  enabled: true
  duration: "14d"

  routing:
    percentage:
      legacy: 100
      pact: 100  # Shadow mode - both process

  comparison:
    enabled: true
    sampleRate: 1.0  # Compare all
    tolerance:
      score: 5        # Allow 5-point difference
      timing: 2000    # Allow 2s difference

  alerts:
    divergence:
      threshold: 5%
      notify: ["migration-team@example.com"]

  reporting:
    schedule: "0 8 * * *"  # Daily at 8 AM
    format: pdf
    recipients: ["stakeholders@example.com"]
```

---

## Rollback Procedures

### Checkpoint Creation

```bash
# Create checkpoint before migration
arka-migrate checkpoint create \
  --name "pre-migration-2024-12-01" \
  --description "Checkpoint before Q4 migration"

# List checkpoints
arka-migrate checkpoint list

# Restore from checkpoint
arka-migrate checkpoint restore \
  --id cp_abc123 \
  --confirm
```

### Rollback Configuration

```yaml
# rollback-config.yaml
rollback:
  automatic:
    enabled: true
    triggers:
      - type: error_rate
        threshold: 5%
        window: 300  # 5 minutes

      - type: latency
        threshold: 5000  # 5 seconds
        percentile: 95

      - type: validation_failure
        threshold: 10

  manual:
    requireApproval: true
    approvers:
      - migration-lead@example.com
      - cto@example.com

  procedure:
    steps:
      - action: pause_migration
        timeout: 60

      - action: restore_checkpoint
        checkpoint: latest

      - action: verify_restore
        tests:
          - rule_count
          - customer_sample
          - transaction_sample

      - action: switch_traffic
        target: legacy
        gradual: true

      - action: notify
        channels:
          - slack
          - email
```

---

## Migration Checklist

```yaml
# Final migration checklist
final_checklist:
  pre_cutover:
    - [ ] All rules migrated and validated
    - [ ] All data imported and verified
    - [ ] Parallel run completed successfully
    - [ ] Stakeholder sign-off obtained
    - [ ] Rollback tested and documented
    - [ ] Support team trained

  cutover:
    - [ ] Create final checkpoint
    - [ ] Stop legacy processing
    - [ ] Final data sync
    - [ ] Switch DNS/routing
    - [ ] Verify ARKA Engine processing
    - [ ] Monitor for 2 hours

  post_cutover:
    - [ ] Verify all integrations
    - [ ] Check audit logging
    - [ ] Confirm alerting works
    - [ ] Generate comparison report
    - [ ] Update documentation
    - [ ] Schedule legacy decommission
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Engineering Team | Initial release |

**Review Schedule:** As needed during migrations
**Next Review:** Post-migration
