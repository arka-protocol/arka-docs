# ARKA Engine User Guide

**Version:** 1.0.0
**Last Updated:** December 2024

## Overview

This guide is designed for compliance officers, analysts, and business users who use ARKA Engine to manage compliance rules, monitor transactions, and generate reports.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Dashboard Overview](#dashboard-overview)
3. [Managing Compliance Rules](#managing-compliance-rules)
4. [Monitoring Transactions](#monitoring-transactions)
5. [Reviewing Decisions](#reviewing-decisions)
6. [Managing Alerts](#managing-alerts)
7. [Entity Management](#entity-management)
8. [Generating Reports](#generating-reports)
9. [Using the Testing Sandbox](#using-the-testing-sandbox)
10. [API Access](#api-access)
11. [Best Practices](#best-practices)
12. [Getting Help](#getting-help)

---

## Getting Started

### Logging In

1. Navigate to your ARKA Engine URL (e.g., `https://pact.yourcompany.com`)
2. Enter your email and password
3. Complete two-factor authentication if enabled
4. You'll be directed to the main dashboard

### First-Time Setup

After your first login:

1. **Update your password** - Navigate to Settings → Security
2. **Set your timezone** - Settings → Preferences → Timezone
3. **Configure notifications** - Settings → Notifications
4. **Review your permissions** - Settings → My Account

### User Interface Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Logo    Dashboard  Rules  Events  Alerts  Reports  ⚙️ 👤  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Pending    │  │   Today's    │  │    Active    │      │
│  │   Alerts     │  │   Events     │  │    Rules     │      │
│  │     42       │  │   12,456     │  │     156      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  Recent Activity                                            │
│  ─────────────────────────────────────────────────────     │
│  • Alert #1234 - High risk transaction flagged              │
│  • Rule "AML-001" updated by John                           │
│  • Report "Monthly SAR" generated                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Dashboard Overview

### Key Metrics Cards

| Card | Description |
|------|-------------|
| **Pending Alerts** | Alerts requiring your review |
| **Today's Events** | Transactions processed today |
| **Active Rules** | Currently enabled compliance rules |
| **Decision Rate** | Allow/Deny/Flag distribution |
| **Risk Score** | Average transaction risk |

### Activity Feed

The activity feed shows real-time updates:
- New alerts generated
- Rule changes
- Report completions
- System notifications

### Quick Actions

- **+ New Rule** - Create a compliance rule
- **📊 Generate Report** - Create a new report
- **🔍 Search** - Search transactions/entities
- **⚡ Quick Check** - Manual compliance check

---

## Managing Compliance Rules

### Viewing Rules

1. Navigate to **Rules** in the main menu
2. Use filters to find specific rules:
   - Status (Active, Inactive, Draft)
   - Jurisdiction (US, EU, UK, etc.)
   - Category (AML, KYC, Sanctions, etc.)
   - Severity (Low, Medium, High, Critical)

### Creating a New Rule

#### Using the Rule Builder

1. Click **+ New Rule**
2. Choose a creation method:
   - **Visual Builder** - Drag-and-drop interface
   - **Natural Language** - Write in plain English
   - **Template** - Start from a template
   - **Advanced** - Write rule code directly

#### Visual Builder Example

```
IF
  └── Transaction Amount > $10,000
  AND
  └── Destination Country IN [High Risk Countries]
THEN
  └── FLAG for Review
  └── Generate Alert (Severity: High)
```

#### Natural Language Example

```
"Flag any wire transfer over $10,000 going to a high-risk country"
```

This generates:

```yaml
name: High Value Transfer to High Risk Country
conditions:
  - field: transaction.type
    operator: equals
    value: wire_transfer
  - field: transaction.amount
    operator: greater_than
    value: 10000
  - field: transaction.destination_country
    operator: in
    value: ${high_risk_countries}
actions:
  - type: flag
    reason: High value transfer to high-risk jurisdiction
  - type: alert
    severity: high
```

### Rule Lifecycle

```
Draft → Pending Review → Approved → Active → Deprecated
                ↓
            Rejected
```

### Testing Rules Before Activation

1. Open the rule
2. Click **Test Rule**
3. Enter test transaction data or select from samples
4. Review the simulated outcome
5. Adjust conditions if needed

### Rule Versioning

- All changes create new versions
- View version history in the **Versions** tab
- Compare versions side-by-side
- Rollback to previous versions if needed

---

## Monitoring Transactions

### Transaction List

Navigate to **Events** → **Transactions** to view:

| Column | Description |
|--------|-------------|
| ID | Unique transaction identifier |
| Type | Wire, ACH, Card, Crypto, etc. |
| Amount | Transaction value |
| From/To | Sender and recipient |
| Status | Processed, Flagged, Blocked |
| Risk Score | 0-100 risk assessment |
| Timestamp | When it occurred |

### Filtering Transactions

Use the filter panel to narrow results:

```
Date Range:     [Last 24 Hours ▼]
Status:         [All ▼] [Flagged ▼] [Blocked ▼]
Amount:         Min: [$____] Max: [$____]
Risk Score:     [0] ──●────── [100]
Entity:         [Search entity...]
Jurisdiction:   [All ▼]
```

### Transaction Details

Click any transaction to view:

- **Summary** - Key transaction information
- **Entities** - Sender and recipient details
- **Risk Analysis** - Why this risk score
- **Rules Triggered** - Which rules matched
- **Decision** - Final compliance decision
- **Timeline** - Processing history
- **Related** - Linked transactions

### Real-Time Monitoring

Enable **Live Mode** to see transactions as they're processed:

1. Click the **Live** toggle in the top right
2. Watch transactions appear in real-time
3. Filter for specific criteria
4. Click any transaction for immediate details

---

## Reviewing Decisions

### Decision Queue

Navigate to **Decisions** → **Pending Review**:

1. Review flagged transactions requiring human decision
2. Click a decision to view full context
3. Make your determination:
   - **Approve** - Allow the transaction
   - **Deny** - Block the transaction
   - **Escalate** - Send to supervisor
   - **Request Info** - Need more information

### Making a Decision

When reviewing a flagged transaction:

1. **Review the Summary**
   - Transaction details
   - Triggered rules
   - Risk score breakdown

2. **Check Entity History**
   - Previous transactions
   - Risk profile
   - KYC status

3. **Review Supporting Evidence**
   - Documents
   - Screening results
   - Related alerts

4. **Make Your Decision**
   - Select decision type
   - Add justification (required)
   - Submit

### Decision Audit Trail

All decisions are logged with:
- Who made the decision
- When it was made
- What information was reviewed
- The justification provided

---

## Managing Alerts

### Alert Dashboard

Navigate to **Alerts** to see:

- **Open Alerts** - Requiring action
- **In Progress** - Being investigated
- **Resolved** - Completed
- **Escalated** - Sent to supervisors

### Alert Types

| Type | Description | Priority |
|------|-------------|----------|
| 🔴 Critical | Immediate action required | P1 |
| 🟠 High | Action within 4 hours | P2 |
| 🟡 Medium | Action within 24 hours | P3 |
| 🟢 Low | Review when available | P4 |

### Working an Alert

1. **Assign to yourself** - Click "Take"
2. **Review details** - Entity, transactions, rules
3. **Investigate** - Check additional sources
4. **Document findings** - Add notes
5. **Resolve** - Close with resolution type:
   - True Positive - Confirmed suspicious
   - False Positive - Not suspicious
   - Escalated - Needs higher review
   - SAR Filed - Reported to authorities

### Alert Notes and Collaboration

- Add notes to document your investigation
- Tag colleagues for input
- Upload supporting documents
- View full investigation timeline

---

## Entity Management

### Viewing Entities

Navigate to **Entities** to manage:

- **Individuals** - Personal accounts
- **Organizations** - Business entities
- **Related Parties** - Beneficial owners, directors

### Entity Profile

Each entity profile shows:

- **Overview** - Basic information
- **Risk Score** - Current risk assessment
- **KYC Status** - Verification status
- **Transactions** - Transaction history
- **Alerts** - Associated alerts
- **Relationships** - Connected entities
- **Documents** - ID, registration docs
- **Notes** - Investigation notes

### Updating Entity Risk

To manually adjust entity risk:

1. Open entity profile
2. Click **Risk Assessment**
3. Select new risk level
4. Provide justification
5. Submit for approval (if required)

### Entity Watchlists

Add entities to watchlists for enhanced monitoring:

1. Open entity profile
2. Click **Add to Watchlist**
3. Select watchlist type:
   - Internal Watch
   - Enhanced Monitoring
   - Restricted
   - Blocked

---

## Generating Reports

### Available Reports

| Report | Description | Schedule |
|--------|-------------|----------|
| Daily Summary | Transaction overview | Daily |
| Weekly Alerts | Alert statistics | Weekly |
| SAR Filing | Suspicious Activity Report | As needed |
| CTR Filing | Currency Transaction Report | As needed |
| Compliance Dashboard | Executive summary | Monthly |
| Audit Report | Detailed audit trail | Quarterly |

### Creating a Report

1. Navigate to **Reports** → **New Report**
2. Select report type
3. Configure parameters:
   - Date range
   - Jurisdictions
   - Entity filters
   - Format (PDF, Excel, CSV)
4. Click **Generate**
5. Download when ready or schedule delivery

### Scheduling Reports

1. Open report configuration
2. Enable **Schedule**
3. Set frequency (Daily, Weekly, Monthly)
4. Add recipients
5. Save schedule

### Report Templates

Create custom report templates:

1. Navigate to **Reports** → **Templates**
2. Click **New Template**
3. Configure sections and data
4. Save template
5. Use for future reports

---

## Using the Testing Sandbox

### Purpose

The sandbox allows you to:
- Test rules before production
- Simulate transactions
- Validate rule changes
- Train on the system

### Creating a Simulation

1. Navigate to **Sandbox** → **New Simulation**
2. Upload test transactions or generate them
3. Select rules to test
4. Run simulation
5. Review results

### Interpreting Results

```
Simulation Results: AML Rule Test
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Transactions Tested:    1,000
Allowed:                  850  (85%)
Flagged:                  120  (12%)
Blocked:                   30   (3%)

Rules Triggered:
├── AML-001 (High Value):     45 times
├── AML-002 (Velocity):       38 times
├── AML-003 (Jurisdiction):   67 times
└── AML-004 (Pattern):        22 times
```

### Regression Testing

Before rule changes:

1. Create baseline snapshot
2. Make rule changes
3. Run regression test
4. Compare results
5. Verify no unexpected impacts

---

## API Access

### Getting API Credentials

1. Navigate to **Settings** → **API Keys**
2. Click **Generate New Key**
3. Name your key
4. Select permissions
5. Copy and secure your key

### Basic API Usage

```bash
# Check health
curl -H "X-API-Key: your_key" \
  https://api.arka.example.com/v1/health

# Get decisions
curl -H "X-API-Key: your_key" \
  https://api.arka.example.com/v1/decisions?status=pending

# Submit event
curl -X POST \
  -H "X-API-Key: your_key" \
  -H "Content-Type: application/json" \
  -d '{"type":"transaction","data":{...}}' \
  https://api.arka.example.com/v1/events
```

### API Documentation

Full API documentation available at:
`https://api.arka.example.com/v1/docs`

---

## Best Practices

### Rule Management

1. **Start simple** - Begin with basic rules, add complexity as needed
2. **Test thoroughly** - Always test in sandbox before production
3. **Document everything** - Add clear descriptions and justifications
4. **Review regularly** - Audit rules quarterly for relevance
5. **Version control** - Track all changes with meaningful comments

### Alert Handling

1. **Prioritize** - Work critical alerts first
2. **Document** - Record investigation steps
3. **Escalate** - Don't sit on difficult cases
4. **Learn** - Use false positives to improve rules
5. **Collaborate** - Tag colleagues for second opinions

### Reporting

1. **Schedule** - Automate recurring reports
2. **Customize** - Create templates for common needs
3. **Archive** - Keep reports per retention policy
4. **Review** - Regularly check report accuracy
5. **Share** - Distribute to appropriate stakeholders

---

## Getting Help

### In-App Help

- Click **?** icon for contextual help
- Hover over fields for tooltips
- Access guided tours from Help menu

### Documentation

- User Guide (this document)
- [Quick Reference Card](./quick-reference.md)
- [FAQ](./TROUBLESHOOTING-FAQ.md)
- Video tutorials in Help Center

### Support Channels

| Issue | Channel |
|-------|---------|
| How-to questions | Help Center / Knowledge Base |
| Bug reports | support@arka-engine.io |
| Feature requests | feedback@arka-engine.io |
| Urgent issues | Emergency hotline |

### Training Resources

- New user onboarding course
- Advanced rule authoring workshop
- Monthly webinars
- Certification program

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+K` | Quick search |
| `Ctrl+N` | New item |
| `Ctrl+S` | Save |
| `Ctrl+/` | Show shortcuts |
| `Esc` | Close modal |
| `?` | Help |

---

## Glossary

| Term | Definition |
|------|------------|
| **Alert** | Notification of potential compliance issue |
| **Decision** | Compliance determination (allow/deny/flag) |
| **Entity** | Person or organization in the system |
| **Event** | Transaction or activity for compliance check |
| **Rule** | Compliance logic defining when to take action |
| **Sandbox** | Testing environment for rules |
| **SAR** | Suspicious Activity Report |
| **CTR** | Currency Transaction Report |

---

## Next Steps

1. Complete the [Getting Started Tutorial](./getting-started/quickstart.md)
2. Create your [First Rule](./getting-started/first-rule.md)
3. Explore the [Sandbox](./guides/testing.md)
4. Review [Best Practices](#best-practices)
