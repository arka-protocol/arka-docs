# ARKA Control Center Security & Threat Model

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** INTERNAL - CONFIDENTIAL
**Author:** ARKA Systems LLC Security Team

---

## Executive Summary

This document defines the security architecture, threat model, and defensive controls for the ARKA Control Center - a multi-tenant SaaS platform for managing blockchain-based compliance infrastructure. The system handles sensitive regulatory data and requires enterprise-grade security controls.

---

## 1. Security Architecture Overview

### 1.1 Trust Zones

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           TRUST ZONE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   PUBLIC    │    │AUTHENTICATED│    │ PRIVILEGED  │    │  REGULATOR  │  │
│  │    ZONE     │    │    ZONE     │    │ ADMIN ZONE  │    │    ZONE     │  │
│  │             │    │             │    │             │    │             │  │
│  │ - Login     │    │ - Dashboard │    │ - Tenant    │    │ - Read-only │  │
│  │ - Health    │    │ - Viewer    │    │   Mgmt      │    │   Access    │  │
│  │ - Public    │    │ - Basic     │    │ - Chain Ops │    │ - Audit     │  │
│  │   Docs      │    │   APIs      │    │ - Rule Ops  │    │   Trails    │  │
│  │             │    │             │    │ - AI Ops    │    │ - Reports   │  │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│        │                  │                  │                  │          │
│        ▼                  ▼                  ▼                  ▼          │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        AWS WAF + CloudFront                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Security Boundaries

| Boundary | Controls | Monitoring |
|----------|----------|------------|
| Internet → CloudFront | WAF, Rate Limiting, Geo-blocking | CloudWatch, GuardDuty |
| CloudFront → ALB | TLS 1.3, Origin Auth | VPC Flow Logs |
| ALB → ECS | Security Groups, mTLS | X-Ray, CloudWatch |
| ECS → Aurora | VPC Isolation, TLS, IAM Auth | RDS Audit Logs |
| ECS → ElastiCache | Encryption in Transit | Redis Logs |
| ECS → Bedrock | VPC Endpoints, IAM | CloudTrail |

---

## 2. Authentication & Authorization

### 2.1 Amazon Cognito Configuration

**User Pool Settings:**
- MFA: Required for admin roles, optional for others
- Password Policy: Min 12 chars, mixed case, digits, symbols
- Account Lockout: 5 failed attempts → 30 min lockout
- Session Duration: Access token 1hr, Refresh token 30 days

**Identity Federation:**
- SAML 2.0 for enterprise SSO
- OIDC for modern identity providers
- External IdP attribute mapping to ARKA roles

### 2.2 Role-Based Access Control (RBAC)

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            RBAC HIERARCHY                                  │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ARKA_GLOBAL_ADMIN ─────────────────────────────────────────────────────┐ │
│  │ • Full system access                                                 │ │
│  │ • Cross-tenant operations                                            │ │
│  │ • Platform configuration                                             │ │
│  │                                                                      │ │
│  ├── TENANT_ADMIN ────────────────────────────────────────────────────┐ │ │
│  │   │ • Tenant-scoped full access                                    │ │ │
│  │   │ • User management                                              │ │ │
│  │   │ • Chain deployment                                             │ │ │
│  │   │                                                                │ │ │
│  │   ├── COMPLIANCE_OFFICER ──────────────────────────────────────┐   │ │ │
│  │   │   │ • Rule management                                      │   │ │ │
│  │   │   │ • Incident resolution                                  │   │ │ │
│  │   │   │ • Attestation review                                   │   │ │ │
│  │   │   │                                                        │   │ │ │
│  │   │   ├── AUDITOR ─────────────────────────────────────────┐   │   │ │ │
│  │   │   │   │ • Read-only access                             │   │   │ │ │
│  │   │   │   │ • Audit log access                             │   │   │ │ │
│  │   │   │   │ • Report generation                            │   │   │ │ │
│  │   │   │   │                                                │   │   │ │ │
│  │   │   └───┴────────────────────────────────────────────────┘   │   │ │ │
│  │   │                                                            │   │ │ │
│  │   ├── DEVELOPER ───────────────────────────────────────────────┤   │ │ │
│  │   │   • API access                                             │   │ │ │
│  │   │   • SDK usage                                              │   │ │ │
│  │   │   • Test environment                                       │   │ │ │
│  │   │                                                            │   │ │ │
│  │   └── BILLING_USER ────────────────────────────────────────────┤   │ │ │
│  │       • Invoice access                                         │   │ │ │
│  │       • Payment methods                                        │   │ │ │
│  │       • Usage reports                                          │   │ │ │
│  │                                                                │   │ │ │
│  └───┴────────────────────────────────────────────────────────────┘   │ │ │
│                                                                       │ │ │
│  REGULATOR (External) ────────────────────────────────────────────────┘ │ │
│  │ • Cross-tenant read-only                                             │ │
│  │ • Regulatory report access                                           │ │
│  │ • Attestation verification                                           │ │
│  │                                                                      │ │
└──┴──────────────────────────────────────────────────────────────────────┘ │
```

### 2.3 Permission Matrix

| Resource | GLOBAL_ADMIN | TENANT_ADMIN | COMPLIANCE | AUDITOR | DEVELOPER | REGULATOR |
|----------|:------------:|:------------:|:----------:|:-------:|:---------:|:---------:|
| Tenants (CRUD) | ✓ | ✓* | - | R | - | R |
| Chains (Deploy) | ✓ | ✓ | - | - | R | R |
| Rules (CRUD) | ✓ | ✓ | ✓ | R | R | R |
| Rules (Approve) | ✓ | ✓ | ✓ | - | - | - |
| Rules (Deploy) | ✓ | ✓ | ✓ | - | - | - |
| AI Config | ✓ | ✓ | - | - | - | - |
| Attestations | ✓ | ✓ | ✓ | R | R | R |
| Incidents | ✓ | ✓ | ✓ | R | R | R |
| Incidents (Resolve) | ✓ | ✓ | ✓ | - | - | - |
| Plugins | ✓ | ✓ | - | - | R | - |
| Billing | ✓ | ✓ | - | - | - | - |
| Audit Logs | ✓ | ✓ | R | ✓ | - | ✓ |

`✓` = Full Access, `R` = Read Only, `✓*` = Own tenant only, `-` = No Access

---

## 3. Threat Model (STRIDE)

### 3.1 Spoofing Identity

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| Credential theft via phishing | HIGH | MEDIUM | MFA required, security awareness training |
| Session hijacking | HIGH | LOW | Short-lived tokens, secure cookies, TLS |
| Token replay attack | HIGH | LOW | Token binding, JTI claim validation |
| Impersonation via SAML assertion | HIGH | LOW | Signature validation, time bounds |

### 3.2 Tampering

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| API request modification | HIGH | LOW | TLS, request signing for critical ops |
| Database record tampering | CRITICAL | LOW | RLS policies, audit logging, encryption |
| Attestation data corruption | CRITICAL | VERY LOW | Blockchain immutability, Merkle proofs |
| Configuration tampering | HIGH | LOW | Version control, approval workflows |

### 3.3 Repudiation

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| Denial of admin actions | HIGH | MEDIUM | Comprehensive audit logging |
| Denial of rule changes | HIGH | MEDIUM | Blockchain-anchored change history |
| Denial of data access | MEDIUM | MEDIUM | Access logs with user identity |

### 3.4 Information Disclosure

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| Cross-tenant data leakage | CRITICAL | LOW | Row-level security, tenant isolation |
| API response data exposure | HIGH | MEDIUM | Response filtering, field-level permissions |
| Log data exposure | MEDIUM | LOW | Log sanitization, access controls |
| Backup data exposure | HIGH | LOW | Encryption at rest, access logging |

### 3.5 Denial of Service

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| API rate exhaustion | MEDIUM | HIGH | Rate limiting, WAF rules |
| Resource exhaustion | HIGH | MEDIUM | Auto-scaling, resource quotas |
| Chain network flooding | HIGH | LOW | Validator controls, transaction limits |
| AI service abuse | MEDIUM | MEDIUM | Token limits, cost monitoring |

### 3.6 Elevation of Privilege

| Threat | Impact | Likelihood | Controls |
|--------|--------|------------|----------|
| Role escalation | CRITICAL | LOW | Role hierarchy enforcement, audit |
| Cross-tenant access | CRITICAL | LOW | Tenant context validation |
| Admin impersonation | CRITICAL | VERY LOW | MFA, admin IP allowlisting |
| API bypass | HIGH | LOW | Middleware chain enforcement |

---

## 4. Data Protection

### 4.1 Encryption Standards

| Data Type | At Rest | In Transit | Key Management |
|-----------|---------|------------|----------------|
| Database | AES-256 (Aurora) | TLS 1.3 | AWS KMS |
| Cache | AES-256 (Redis) | TLS 1.2+ | AWS KMS |
| Object Storage | AES-256 (S3) | TLS 1.2+ | S3-managed/KMS |
| Attestation Data | Blockchain | TLS 1.3 | Tenant HSM |
| Secrets | Secrets Manager | TLS 1.3 | AWS KMS |

### 4.2 Sensitive Data Classification

| Classification | Examples | Handling Requirements |
|----------------|----------|----------------------|
| PUBLIC | API documentation, health status | No restrictions |
| INTERNAL | Tenant names, user emails | Authentication required |
| CONFIDENTIAL | Attestation data, rule definitions | Role-based access, encryption |
| RESTRICTED | Private keys, audit logs | MFA, audit logging, encryption |

### 4.3 Data Retention

| Data Type | Retention | Deletion Method |
|-----------|-----------|-----------------|
| Audit Logs | 7 years | Archived to Glacier, then purged |
| Attestations | Indefinite | Blockchain-anchored, immutable |
| Session Data | 30 days | Automatic expiration |
| Backups | 35 days (prod) | Automated lifecycle policy |

---

## 5. Network Security

### 5.1 VPC Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VPC (10.0.0.0/16)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     PUBLIC SUBNETS (10.0.0.0/24)                    │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │    │
│  │  │   NAT GW    │  │   NAT GW    │  │   NAT GW    │  (Multi-AZ)     │    │
│  │  │   AZ-a      │  │   AZ-b      │  │   AZ-c      │                 │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                 │    │
│  │  ┌───────────────────────────────────────────────┐                 │    │
│  │  │         Application Load Balancer              │                 │    │
│  │  └───────────────────────────────────────────────┘                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    PRIVATE SUBNETS (10.0.16.0/24)                   │    │
│  │  ┌─────────────────────────────────────────────────────────────┐   │    │
│  │  │                    ECS Fargate Tasks                        │   │    │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │   │    │
│  │  │  │ Task 1  │  │ Task 2  │  │ Task 3  │  │ Task N  │        │   │    │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │   │    │
│  │  └─────────────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                   ISOLATED SUBNETS (10.0.32.0/24)                   │    │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐         │    │
│  │  │    Aurora PostgreSQL    │  │    ElastiCache Redis    │         │    │
│  │  │    (Multi-AZ Cluster)   │  │    (Cluster Mode)       │         │    │
│  │  └─────────────────────────┘  └─────────────────────────┘         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  VPC Endpoints: S3, Secrets Manager, Bedrock, CloudWatch, ECR               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Security Groups

| Security Group | Inbound | Outbound |
|----------------|---------|----------|
| ALB-SG | 443 (0.0.0.0/0) | ECS-SG:3200 |
| ECS-SG | 3200 (ALB-SG) | DB-SG:5432, Cache-SG:6379, VPC Endpoints |
| DB-SG | 5432 (ECS-SG) | None |
| Cache-SG | 6379 (ECS-SG) | None |

---

## 6. Incident Response

### 6.1 Security Incident Classifications

| Severity | Description | Response Time | Escalation |
|----------|-------------|---------------|------------|
| P1 - Critical | Active breach, data exfiltration | 15 min | CISO, Legal, CEO |
| P2 - High | Attempted breach, vulnerability exploit | 1 hour | Security Lead, CTO |
| P3 - Medium | Suspicious activity, policy violation | 4 hours | Security Team |
| P4 - Low | Informational, minor anomaly | 24 hours | On-call engineer |

### 6.2 Response Procedures

1. **Detection**: GuardDuty, CloudWatch Alarms, WAF logs
2. **Containment**: Isolate affected resources, revoke sessions
3. **Eradication**: Remove threat, patch vulnerabilities
4. **Recovery**: Restore from backups, verify integrity
5. **Lessons Learned**: Post-mortem, update controls

---

## 7. Compliance Mapping

### 7.1 SOC 2 Type II Controls

| Control | Implementation |
|---------|----------------|
| CC1.1 - Board Oversight | Quarterly security reviews, risk assessment |
| CC6.1 - Logical Access | Cognito RBAC, MFA, session management |
| CC6.6 - Encryption | TLS 1.3, AES-256, KMS key management |
| CC7.2 - Monitoring | CloudWatch, GuardDuty, CloudTrail |

### 7.2 GDPR Requirements

| Requirement | Implementation |
|-------------|----------------|
| Data Minimization | Tenant-scoped data, retention policies |
| Right to Erasure | Soft delete with scheduled purge |
| Data Portability | Export API, standard formats |
| Breach Notification | Automated alerting, 72-hour SLA |

---

## 8. Security Monitoring

### 8.1 Logging Strategy

| Log Source | Retention | Analysis |
|------------|-----------|----------|
| CloudTrail | 90 days (hot), 7 years (archive) | Athena, SIEM |
| VPC Flow Logs | 30 days | CloudWatch Insights |
| Application Logs | 30 days | CloudWatch, X-Ray |
| WAF Logs | 30 days | Athena, GuardDuty |
| RDS Audit Logs | 90 days | CloudWatch, SIEM |

### 8.2 Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Failed Auth Attempts | >10/min | >50/min | Block IP, notify |
| 4xx Errors | >5% | >10% | Investigate, scale |
| 5xx Errors | >1% | >5% | Page on-call |
| DB Connections | >70% | >90% | Scale, investigate |

---

## 9. Security Checklist

### 9.1 Pre-Deployment

- [ ] Security review completed
- [ ] Penetration test passed
- [ ] Dependency vulnerabilities patched
- [ ] Secrets rotated
- [ ] IAM policies reviewed
- [ ] WAF rules configured
- [ ] Monitoring alerts configured

### 9.2 Ongoing

- [ ] Weekly vulnerability scans
- [ ] Monthly access reviews
- [ ] Quarterly penetration tests
- [ ] Annual security audit
- [ ] Continuous dependency updates

---

## 10. Contact Information

| Role | Contact | Escalation |
|------|---------|------------|
| Security Team | security@arka.systems | PagerDuty |
| CISO | ciso@arka.systems | Direct |
| Incident Response | incident@arka.systems | PagerDuty |

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | Dec 2024 | ARKA Security | Initial release |
