# ARKA Engine Enterprise Licensing Model

**Version:** 1.0.0
**Effective Date:** December 1, 2024
**Last Updated:** December 2024

---

## Overview

ARKA Engine offers flexible enterprise licensing to meet diverse deployment requirements. This document outlines available licensing models, deployment options, and pricing structures.

---

## Table of Contents

1. [Licensing Models](#licensing-models)
2. [Deployment Options](#deployment-options)
3. [Pricing Tiers](#pricing-tiers)
4. [License Metrics](#license-metrics)
5. [Enterprise Features by Tier](#enterprise-features-by-tier)
6. [Add-On Modules](#add-on-modules)
7. [Volume Discounts](#volume-discounts)
8. [License Management](#license-management)
9. [Procurement Process](#procurement-process)

---

## Licensing Models

### Available Models

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LICENSING MODELS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SUBSCRIPTION (SaaS)                               │   │
│  │  • Hosted by ARKA Engine                                            │   │
│  │  • Monthly or annual billing                                         │   │
│  │  • Automatic updates                                                 │   │
│  │  • No infrastructure management                                      │   │
│  │  • Pay-as-you-grow model                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    ON-PREMISES LICENSE                               │   │
│  │  • Deploy on your infrastructure                                     │   │
│  │  • Perpetual or term license                                        │   │
│  │  • Full data control                                                │   │
│  │  • Annual maintenance & support                                      │   │
│  │  • Air-gapped deployment available                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    HYBRID DEPLOYMENT                                 │   │
│  │  • Mix of SaaS and on-premises                                      │   │
│  │  • Flexible data residency                                          │   │
│  │  • Unified management                                               │   │
│  │  • Best of both models                                              │   │
│  │  • Geographic flexibility                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    OEM / EMBEDDED LICENSE                            │   │
│  │  • Embed ARKA in your product                                       │   │
│  │  • White-label capabilities                                         │   │
│  │  • Revenue share or flat fee                                        │   │
│  │  • Co-marketing opportunities                                       │   │
│  │  • Technical integration support                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Model Comparison

| Aspect | SaaS | On-Premises | Hybrid | OEM |
|--------|------|-------------|--------|-----|
| **Deployment** | Cloud-hosted | Customer infrastructure | Mixed | Embedded |
| **Data Location** | ARKA cloud | Customer controlled | Flexible | Per agreement |
| **Updates** | Automatic | Customer controlled | Configurable | Per release |
| **Maintenance** | Included | Separate contract | Included | Per agreement |
| **Customization** | Configuration | Full access | Both | White-label |
| **Time to Deploy** | Hours | Days-Weeks | Variable | Integration time |
| **Scaling** | Automatic | Customer managed | Both | Per product |
| **Compliance** | Shared responsibility | Customer responsibility | Split | Per agreement |

---

## Deployment Options

### SaaS Deployment

```yaml
saas_deployment:
  regions:
    - name: "US East"
      provider: AWS
      location: us-east-1
      data_residency: US

    - name: "EU West"
      provider: AWS
      location: eu-west-1
      data_residency: EU/GDPR

    - name: "Asia Pacific"
      provider: AWS
      location: ap-southeast-1
      data_residency: APAC

  features:
    - automatic_scaling
    - managed_updates
    - built_in_dr
    - 99.99_sla
    - soc2_compliant

  included_services:
    - infrastructure_management
    - security_monitoring
    - backup_and_recovery
    - 24x7_support
    - compliance_maintenance
```

### On-Premises Deployment

```yaml
on_premises_deployment:
  supported_platforms:
    kubernetes:
      - EKS
      - AKS
      - GKE
      - OpenShift
      - Rancher
      - Vanilla K8s 1.28+

    virtual_machines:
      - VMware vSphere
      - Hyper-V
      - KVM
      - Proxmox

    bare_metal:
      - Linux (Ubuntu, RHEL, Debian)
      - Custom configurations

  deployment_options:
    - name: "Standard"
      nodes: 3-6
      ha: true
      use_case: "Small to medium deployments"

    - name: "Enterprise"
      nodes: 6-12
      ha: true
      multi_region: true
      use_case: "Large deployments"

    - name: "Air-Gapped"
      nodes: 3+
      ha: true
      internet: false
      use_case: "Highly secure environments"

  requirements:
    minimum:
      cpu: "16 cores"
      memory: "64 GB"
      storage: "500 GB SSD"
      network: "1 Gbps"

    recommended:
      cpu: "32+ cores"
      memory: "128+ GB"
      storage: "1+ TB NVMe"
      network: "10 Gbps"
```

### Hybrid Deployment

```yaml
hybrid_deployment:
  configurations:
    - name: "Control Plane SaaS + Data On-Prem"
      description: "Management in cloud, data processing on-premises"
      use_case: "Data sovereignty with cloud management"

    - name: "Primary SaaS + DR On-Prem"
      description: "Primary in cloud with on-prem disaster recovery"
      use_case: "Business continuity requirements"

    - name: "Geographic Split"
      description: "Different regions with different deployment models"
      use_case: "Multi-jurisdictional compliance"

    - name: "Edge + Cloud"
      description: "Edge processing with cloud aggregation"
      use_case: "Low-latency requirements"

  connectivity:
    - vpn_tunnel
    - direct_connect
    - private_link
    - api_gateway

  data_sync:
    - real_time_replication
    - scheduled_sync
    - event_driven_sync
    - manual_sync
```

---

## Pricing Tiers

### Tier Overview

| Tier | Target | Starting Price | Key Features |
|------|--------|----------------|--------------|
| **Starter** | Small teams | $2,500/month | Basic compliance, 5 users |
| **Professional** | Growing companies | $7,500/month | Advanced features, 25 users |
| **Enterprise** | Large organizations | $25,000/month | Full platform, unlimited |
| **Custom** | Special requirements | Contact sales | Tailored solution |

### Detailed Pricing

#### Starter Tier

```
STARTER - $2,500/month (annual) or $3,000/month (monthly)

Included:
├── Users: 5 named users
├── Transactions: 100,000/month
├── Rules: 100 active rules
├── Integrations: 3
├── Storage: 50 GB
├── Support: Business hours email
├── SLA: 99.5% uptime
└── Features:
    ├── Core compliance engine
    ├── Basic rule builder
    ├── Standard reports
    ├── REST API access
    └── Community plugins

Overage Rates:
├── Additional users: $100/user/month
├── Additional transactions: $0.01/transaction
└── Additional storage: $1/GB/month
```

#### Professional Tier

```
PROFESSIONAL - $7,500/month (annual) or $9,000/month (monthly)

Included:
├── Users: 25 named users
├── Transactions: 1,000,000/month
├── Rules: 500 active rules
├── Integrations: 10
├── Storage: 250 GB
├── Support: 24/7 (P1/P2), business hours (P3/P4)
├── SLA: 99.9% uptime
└── Features:
    ├── Everything in Starter
    ├── Advanced rule builder
    ├── AI-assisted rules
    ├── Custom reports
    ├── Webhook integrations
    ├── SSO (SAML/OIDC)
    ├── Audit logging
    ├── API rate limit: 1000/min
    └── 5 premium plugins

Overage Rates:
├── Additional users: $75/user/month
├── Additional transactions: $0.005/transaction
└── Additional storage: $0.75/GB/month
```

#### Enterprise Tier

```
ENTERPRISE - $25,000/month (annual) or $30,000/month (monthly)

Included:
├── Users: Unlimited
├── Transactions: 10,000,000/month
├── Rules: Unlimited
├── Integrations: Unlimited
├── Storage: 1 TB
├── Support: 24/7/365 with dedicated CSM
├── SLA: 99.99% uptime
└── Features:
    ├── Everything in Professional
    ├── Multi-tenant platform
    ├── White-label options
    ├── Custom branding
    ├── Advanced analytics
    ├── Blockchain anchoring
    ├── Dedicated infrastructure option
    ├── Custom SLA terms
    ├── Quarterly business reviews
    ├── Named support engineer
    ├── Early access program
    ├── All premium plugins
    └── API rate limit: 10,000/min

Overage Rates:
├── Additional transactions: $0.002/transaction
└── Additional storage: $0.50/GB/month
```

#### Custom / Enterprise Plus

```
CUSTOM - Contact Sales

For organizations requiring:
├── Unlimited transactions
├── On-premises deployment
├── Air-gapped environments
├── Custom compliance frameworks
├── White-label / OEM licensing
├── Multi-region deployment
├── Custom SLA (99.999%+)
├── Dedicated support team
├── Custom development
├── Professional services
└── Special legal terms

Typical starting point: $100,000+/year
```

---

## License Metrics

### Metric Types

| Metric | Description | Best For |
|--------|-------------|----------|
| **Per User** | Named users with system access | Small teams, defined users |
| **Per Core** | CPU cores in deployment | On-premises, predictable load |
| **Per Transaction** | Events/decisions processed | Variable workloads |
| **Per Tenant** | Sub-organizations hosted | MSP/CaaS providers |
| **Per Revenue** | Based on customer revenue | OEM/embedded |

### Per User Licensing

```yaml
per_user_licensing:
  user_types:
    - name: "Named User"
      description: "Specific individual with login credentials"
      transferable: false

    - name: "Concurrent User"
      description: "Shared pool of simultaneous users"
      ratio: "3:1 named to concurrent"

    - name: "API User"
      description: "Service account for integrations"
      included: "2 per 10 named users"

  tiers:
    starter: 5
    professional: 25
    enterprise: unlimited

  overage:
    starter: "$100/user/month"
    professional: "$75/user/month"
    enterprise: "included"
```

### Per Transaction Licensing

```yaml
per_transaction_licensing:
  definition: "A single event submitted for compliance evaluation"

  counting:
    - event_submission: 1 transaction
    - batch_submission: "1 transaction per event"
    - retry: "not counted"
    - webhook_delivery: "not counted"

  tiers:
    starter:
      included: 100000
      overage: "$0.01/transaction"

    professional:
      included: 1000000
      overage: "$0.005/transaction"

    enterprise:
      included: 10000000
      overage: "$0.002/transaction"

  volume_discounts:
    - range: "10M-50M"
      discount: "20%"
    - range: "50M-100M"
      discount: "35%"
    - range: "100M+"
      discount: "50%"
```

---

## Enterprise Features by Tier

### Feature Matrix

| Feature | Starter | Professional | Enterprise |
|---------|---------|--------------|------------|
| **Core Engine** | | | |
| Rule evaluation | ✅ | ✅ | ✅ |
| Basic rule builder | ✅ | ✅ | ✅ |
| Advanced rule builder | - | ✅ | ✅ |
| Natural language rules | - | ✅ | ✅ |
| Rule versioning | Basic | Advanced | Full |
| **Compliance** | | | |
| KYC verification | ✅ | ✅ | ✅ |
| AML monitoring | - | ✅ | ✅ |
| Sanctions screening | - | ✅ | ✅ |
| Cross-border compliance | - | - | ✅ |
| Regulatory reporting | - | ✅ | ✅ |
| **Integration** | | | |
| REST API | ✅ | ✅ | ✅ |
| gRPC API | - | ✅ | ✅ |
| Webhooks | ✅ (3) | ✅ (10) | ✅ (Unlimited) |
| SDK access | ✅ | ✅ | ✅ |
| Custom integrations | - | - | ✅ |
| **Security** | | | |
| SSO (SAML/OIDC) | - | ✅ | ✅ |
| MFA | ✅ | ✅ | ✅ |
| RBAC | Basic | Advanced | Full |
| IP allowlisting | - | ✅ | ✅ |
| Audit logging | Basic | Advanced | Full + Export |
| **Platform** | | | |
| Multi-tenant | - | - | ✅ |
| White-label | - | - | ✅ |
| Custom branding | - | - | ✅ |
| Blockchain anchoring | - | - | ✅ |
| **Analytics** | | | |
| Standard reports | ✅ | ✅ | ✅ |
| Custom reports | - | ✅ | ✅ |
| Advanced analytics | - | - | ✅ |
| Predictive compliance | - | - | ✅ |
| **Support** | | | |
| Email support | ✅ | ✅ | ✅ |
| Chat support | - | ✅ | ✅ |
| Phone support | - | P1/P2 | ✅ |
| Dedicated CSM | - | Shared | Named |
| Named engineer | - | - | ✅ |
| SLA | 99.5% | 99.9% | 99.99% |

---

## Add-On Modules

### Available Add-Ons

| Add-On | Description | Price |
|--------|-------------|-------|
| **AI/ML Pack** | Predictive compliance, anomaly detection | $5,000/month |
| **Blockchain Pack** | Multi-chain anchoring, smart contracts | $3,000/month |
| **Advanced Reporting** | Custom dashboards, data warehouse | $2,500/month |
| **Multi-Region** | Geographic redundancy, data residency | $7,500/month |
| **Premium Support** | Upgrade support tier | Variable |
| **Professional Services** | Implementation, customization | $250/hour |
| **Training Credits** | Instructor-led training | $5,000/pack |
| **Additional Storage** | Beyond included allocation | $0.50-1.00/GB |

### Add-On Compatibility

| Add-On | Starter | Professional | Enterprise |
|--------|---------|--------------|------------|
| AI/ML Pack | - | ✅ | Included |
| Blockchain Pack | - | ✅ | Included |
| Advanced Reporting | - | ✅ | Included |
| Multi-Region | - | - | ✅ |
| Premium Support | ✅ | ✅ | Included |
| Professional Services | ✅ | ✅ | ✅ |
| Training Credits | ✅ | ✅ | ✅ |

---

## Volume Discounts

### Discount Structure

| Annual Commitment | Discount |
|-------------------|----------|
| $50,000 - $99,999 | 5% |
| $100,000 - $249,999 | 10% |
| $250,000 - $499,999 | 15% |
| $500,000 - $999,999 | 20% |
| $1,000,000+ | Custom |

### Multi-Year Discounts

| Term | Discount |
|------|----------|
| 1 year | Standard |
| 2 years | +5% |
| 3 years | +10% |
| 5 years | +15% |

### Non-Profit / Education

| Organization Type | Discount |
|-------------------|----------|
| Registered Non-Profit | 25% |
| Educational Institution | 50% |
| Government | Contact sales |
| Startup (< $5M funding) | 30% |

---

## License Management

### License Portal

Access the license management portal at: https://license.arka-engine.io

Features:
- View active licenses
- Track usage metrics
- Download license keys
- Manage users
- View billing history
- Request changes

### License Key Management

```yaml
license_management:
  key_types:
    - name: "Production Key"
      environment: production
      validity: subscription_term
      features: full

    - name: "Non-Production Key"
      environment: dev/staging/test
      validity: subscription_term
      features: full
      limitations: "Not for production data"

    - name: "Trial Key"
      environment: any
      validity: 30 days
      features: full
      limitations: "Evaluation only"

  activation:
    online: automatic
    offline: manual_process
    air_gapped: usb_key_file

  compliance:
    usage_reporting: automatic
    audit_support: on_request
    violation_handling: notification_first
```

### Usage Monitoring

```bash
# CLI commands for license management

# Check license status
arka-engine license status

# View usage metrics
arka-engine license usage

# Export usage report
arka-engine license report --format csv --period monthly

# Verify license
arka-engine license verify
```

---

## Procurement Process

### Sales Process

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PROCUREMENT PROCESS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. DISCOVERY                                                               │
│     ├── Initial contact (sales@arka-engine.io)                             │
│     ├── Requirements gathering call                                         │
│     └── Technical discovery session                                         │
│                                                                              │
│  2. EVALUATION                                                              │
│     ├── Trial license (30 days)                                            │
│     ├── Proof of concept support                                            │
│     └── Technical validation                                                │
│                                                                              │
│  3. PROPOSAL                                                                │
│     ├── Solution architecture                                               │
│     ├── Pricing proposal                                                    │
│     └── Contract review                                                     │
│                                                                              │
│  4. NEGOTIATION                                                             │
│     ├── Terms discussion                                                    │
│     ├── Legal review                                                        │
│     └── Final adjustments                                                   │
│                                                                              │
│  5. CLOSE                                                                   │
│     ├── Contract signature                                                  │
│     ├── Payment processing                                                  │
│     └── License provisioning                                                │
│                                                                              │
│  6. ONBOARDING                                                              │
│     ├── Kickoff call                                                        │
│     ├── Implementation support                                              │
│     └── Go-live assistance                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Required Documents

| Document | Purpose | Timeline |
|----------|---------|----------|
| **NDA** | Confidentiality for discussions | Before demo |
| **Master Service Agreement** | Main contract terms | Before purchase |
| **Order Form** | Specific license details | At purchase |
| **DPA** | Data processing terms (if applicable) | Before go-live |
| **BAA** | Business associate agreement (if HIPAA) | Before go-live |

### Payment Terms

| Method | Availability | Terms |
|--------|--------------|-------|
| Credit Card | Starter only | Due on order |
| Invoice | All tiers | Net 30 |
| Wire Transfer | Enterprise | Net 30 |
| PO/Procurement | Enterprise | As agreed |

---

## Contact Sales

**Email:** sales@arka-engine.io
**Phone:** +1-800-ARKA-ENG
**Web:** https://arka-engine.io/pricing

### Request a Quote

1. Visit https://arka-engine.io/contact
2. Fill out requirements form
3. Receive proposal within 2 business days
4. Schedule demo/discussion

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Sales Operations | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
