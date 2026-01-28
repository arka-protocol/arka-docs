# ARKA Engine Zero-Trust & IAM Integration

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Internal / Customer-Facing

---

## Overview

This document describes ARKA Engine's zero-trust security architecture and Identity and Access Management (IAM) integration capabilities. ARKA Engine is designed from the ground up with zero-trust principles, ensuring that no user, device, or network is inherently trusted.

---

## Table of Contents

1. [Zero-Trust Architecture](#zero-trust-architecture)
2. [Identity Management](#identity-management)
3. [Authentication Methods](#authentication-methods)
4. [Authorization Framework](#authorization-framework)
5. [SSO & Federation](#sso--federation)
6. [API Security](#api-security)
7. [Service-to-Service Authentication](#service-to-service-authentication)
8. [Session Management](#session-management)
9. [Audit & Monitoring](#audit--monitoring)
10. [Implementation Guide](#implementation-guide)

---

## Zero-Trust Architecture

### Core Principles

ARKA Engine implements the NIST SP 800-207 Zero Trust Architecture framework:

| Principle | Implementation |
|-----------|----------------|
| **Never trust, always verify** | Every request authenticated and authorized |
| **Assume breach** | Defense in depth, micro-segmentation |
| **Least privilege access** | Minimal permissions, just-in-time access |
| **Explicit verification** | Context-aware access decisions |
| **Continuous validation** | Real-time policy enforcement |

### Architecture Overview

```
                                    ┌─────────────────────────────────────┐
                                    │         Policy Decision Point       │
                                    │              (PDP)                  │
                                    │  ┌─────────┐  ┌─────────┐          │
                                    │  │ Policy  │  │ Context │          │
                                    │  │ Engine  │  │ Engine  │          │
                                    │  └─────────┘  └─────────┘          │
                                    └─────────────────┬───────────────────┘
                                                      │
                                                      │ Policy Decision
                                                      ▼
┌──────────────┐    ┌─────────────────────────────────────────────────────────┐
│              │    │                Policy Enforcement Point (PEP)            │
│    Users     │───▶│  ┌───────────┐  ┌───────────┐  ┌───────────┐           │
│   Devices    │    │  │   API     │  │  Service  │  │   Data    │           │
│   Services   │    │  │  Gateway  │  │   Mesh    │  │   Layer   │           │
│              │    │  └───────────┘  └───────────┘  └───────────┘           │
└──────────────┘    └─────────────────────────────────────────────────────────┘
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────┐
                    │                                 ▼                         │
                    │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │
                    │  │   ARKA    │  │   ARKA    │  │   ARKA    │            │
                    │  │   Core    │  │   RTVM    │  │  Plugins  │            │
                    │  └───────────┘  └───────────┘  └───────────┘            │
                    │                   Protected Resources                    │
                    └─────────────────────────────────────────────────────────┘
```

### Trust Boundaries

| Boundary | Protection Mechanism |
|----------|---------------------|
| External → API Gateway | TLS 1.3, Authentication, Rate Limiting |
| API Gateway → Services | mTLS, Service Mesh, JWT Validation |
| Service → Service | mTLS, SPIFFE/SPIRE, Network Policies |
| Service → Database | Encrypted connections, IAM authentication |
| Service → External | Outbound TLS, Egress filtering |

---

## Identity Management

### Identity Types

| Identity Type | Description | Authentication |
|---------------|-------------|----------------|
| **Human Users** | Employees, customers, partners | SSO, MFA, Password |
| **Service Accounts** | Backend services, jobs | Certificates, Tokens |
| **Machine Identities** | Infrastructure, IoT | Certificates, TPM |
| **API Clients** | External integrations | API Keys, OAuth |

### Identity Lifecycle

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Provision  │───▶│   Manage     │───▶│   Review     │───▶│  Deprovision │
│              │    │              │    │              │    │              │
│ • Create     │    │ • Update     │    │ • Audit      │    │ • Disable    │
│ • Enroll MFA │    │ • Role change│    │ • Recertify  │    │ • Remove     │
│ • Assign role│    │ • Reset creds│    │ • Investigate│    │ • Archive    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Identity Attributes

```yaml
# User identity structure
user:
  id: "usr_abc123"
  email: "user@company.com"
  type: "human"
  status: "active"

  authentication:
    methods:
      - type: "password"
        strength: "strong"
        last_changed: "2024-11-15"
      - type: "totp"
        enrolled: "2024-11-01"
      - type: "webauthn"
        keys: 2

  attributes:
    department: "compliance"
    job_title: "Compliance Analyst"
    location: "US-NY"
    risk_score: 25

  roles:
    - "compliance_analyst"
    - "report_viewer"

  permissions:
    - "rules:read"
    - "transactions:read"
    - "alerts:manage"

  restrictions:
    ip_allowlist: ["10.0.0.0/8"]
    time_restrictions: "08:00-20:00 EST"
    mfa_required: true
```

---

## Authentication Methods

### Supported Methods

| Method | Use Case | Security Level |
|--------|----------|---------------|
| **Password + MFA** | Interactive users | High |
| **SAML 2.0** | Enterprise SSO | High |
| **OAuth 2.0 / OIDC** | Modern SSO, API | High |
| **mTLS** | Service-to-service | Very High |
| **API Keys** | Programmatic access | Medium |
| **JWT Tokens** | Stateless auth | High |
| **FIDO2/WebAuthn** | Passwordless | Very High |

### Multi-Factor Authentication (MFA)

```yaml
mfa_configuration:
  # Enforcement levels
  enforcement:
    all_users: true
    admin_users: required
    api_access: optional

  # Supported factors
  factors:
    totp:
      enabled: true
      apps: ["Google Authenticator", "Authy", "1Password"]

    webauthn:
      enabled: true
      types: ["security_key", "platform"]

    sms:
      enabled: false  # Disabled due to SIM swap risks

    email:
      enabled: true
      use_case: "recovery_only"

    push:
      enabled: true
      providers: ["Duo", "Okta Verify"]

  # Risk-based step-up
  step_up_triggers:
    - sensitive_action
    - new_device
    - unusual_location
    - high_risk_score
```

### Passwordless Authentication

```yaml
passwordless:
  enabled: true

  methods:
    # FIDO2/WebAuthn
    webauthn:
      enabled: true
      resident_keys: preferred
      user_verification: required
      attestation: direct

    # Magic links
    magic_link:
      enabled: true
      expiry: 300  # 5 minutes
      single_use: true

    # Passkeys
    passkeys:
      enabled: true
      sync_allowed: true
      platforms: ["Apple", "Google", "Microsoft"]
```

---

## Authorization Framework

### Role-Based Access Control (RBAC)

```yaml
# Role hierarchy
roles:
  super_admin:
    description: "Full system access"
    inherits: ["admin"]
    permissions:
      - "*"

  admin:
    description: "Administrative access"
    inherits: ["analyst"]
    permissions:
      - "users:*"
      - "roles:*"
      - "settings:*"
      - "audit:*"

  compliance_manager:
    description: "Compliance team lead"
    inherits: ["compliance_analyst"]
    permissions:
      - "rules:create"
      - "rules:approve"
      - "reports:generate"
      - "team:manage"

  compliance_analyst:
    description: "Compliance analyst"
    permissions:
      - "rules:read"
      - "rules:propose"
      - "transactions:read"
      - "alerts:manage"
      - "entities:read"

  auditor:
    description: "Read-only audit access"
    permissions:
      - "audit:read"
      - "rules:read"
      - "transactions:read"
      - "reports:read"

  api_integration:
    description: "API integration access"
    permissions:
      - "events:submit"
      - "decisions:read"
      - "webhooks:configure"
```

### Attribute-Based Access Control (ABAC)

```yaml
# ABAC policy example
policies:
  - name: "tenant_isolation"
    description: "Users can only access their tenant data"
    condition: |
      user.tenant_id == resource.tenant_id
    effect: "allow"

  - name: "jurisdiction_access"
    description: "Users can only access jurisdictions they're assigned to"
    condition: |
      resource.jurisdiction in user.attributes.jurisdictions
    effect: "allow"

  - name: "sensitive_data_access"
    description: "PII access requires additional clearance"
    condition: |
      resource.classification != "PII" or
      "pii_access" in user.permissions
    effect: "allow"

  - name: "time_based_access"
    description: "Restrict access outside business hours"
    condition: |
      action in ["read"] or
      current_time.hour >= 8 and current_time.hour <= 20
    effect: "allow"
```

### Permission Model

```yaml
# Permission structure: resource:action or resource:action:scope
permissions:
  # Rule management
  rules:read: "View compliance rules"
  rules:create: "Create new rules"
  rules:update: "Modify existing rules"
  rules:delete: "Delete rules"
  rules:approve: "Approve rule changes"

  # Transaction monitoring
  transactions:read: "View transactions"
  transactions:export: "Export transaction data"

  # Alert management
  alerts:read: "View alerts"
  alerts:manage: "Work on alerts"
  alerts:escalate: "Escalate alerts"
  alerts:close: "Close alerts"

  # Entity management
  entities:read: "View entities"
  entities:create: "Create entities"
  entities:update: "Update entities"
  entities:risk_override: "Override risk scores"

  # Reports
  reports:read: "View reports"
  reports:generate: "Generate reports"
  reports:file: "File regulatory reports"

  # System administration
  users:read: "View users"
  users:create: "Create users"
  users:update: "Update users"
  users:delete: "Delete users"
  roles:manage: "Manage roles"
  settings:read: "View settings"
  settings:update: "Update settings"
  audit:read: "View audit logs"
```

---

## SSO & Federation

### SAML 2.0 Integration

```yaml
saml:
  enabled: true

  identity_providers:
    - name: "Corporate IdP"
      entity_id: "https://idp.company.com"
      sso_url: "https://idp.company.com/saml/sso"
      slo_url: "https://idp.company.com/saml/slo"
      certificate: "/etc/pact/saml/idp-cert.pem"

      # Attribute mapping
      attribute_mapping:
        email: "urn:oid:0.9.2342.19200300.100.1.3"
        firstName: "urn:oid:2.5.4.42"
        lastName: "urn:oid:2.5.4.4"
        groups: "urn:oid:1.3.6.1.4.1.5923.1.5.1.1"

      # Group to role mapping
      role_mapping:
        "CN=Compliance Admins,OU=Groups": "admin"
        "CN=Compliance Analysts,OU=Groups": "compliance_analyst"
        "CN=Auditors,OU=Groups": "auditor"

  service_provider:
    entity_id: "https://arka.company.com"
    acs_url: "https://arka.company.com/auth/saml/callback"
    slo_url: "https://arka.company.com/auth/saml/logout"
    certificate: "/etc/pact/saml/sp-cert.pem"
    private_key: "/etc/pact/saml/sp-key.pem"

    # Security settings
    sign_requests: true
    require_signed_assertions: true
    require_encrypted_assertions: true
```

### OAuth 2.0 / OIDC Integration

```yaml
oidc:
  enabled: true

  providers:
    # Okta
    okta:
      issuer: "https://company.okta.com"
      client_id: "${OKTA_CLIENT_ID}"
      client_secret: "${OKTA_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email", "groups"]

      # Claims mapping
      claims_mapping:
        email: "email"
        name: "name"
        groups: "groups"

    # Azure AD
    azure_ad:
      issuer: "https://login.microsoftonline.com/{tenant}/v2.0"
      client_id: "${AZURE_CLIENT_ID}"
      client_secret: "${AZURE_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email"]

      # Claims mapping
      claims_mapping:
        email: "preferred_username"
        name: "name"
        groups: "groups"

    # Google Workspace
    google:
      issuer: "https://accounts.google.com"
      client_id: "${GOOGLE_CLIENT_ID}"
      client_secret: "${GOOGLE_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email"]
      hd: "company.com"  # Restrict to domain

  # Token validation
  token_validation:
    issuer_validation: strict
    audience_validation: required
    expiry_tolerance: 60  # seconds
    jwks_cache_ttl: 3600  # seconds
```

### Directory Integration

```yaml
# LDAP/Active Directory
directory:
  type: "active_directory"

  connection:
    servers:
      - "ldaps://dc1.company.com:636"
      - "ldaps://dc2.company.com:636"
    base_dn: "DC=company,DC=com"
    bind_dn: "CN=ARKA Service,OU=Service Accounts,DC=company,DC=com"
    bind_password: "${LDAP_BIND_PASSWORD}"
    tls_verify: true

  search:
    user_base: "OU=Users,DC=company,DC=com"
    user_filter: "(&(objectClass=user)(sAMAccountName={username}))"
    group_base: "OU=Groups,DC=company,DC=com"
    group_filter: "(member={user_dn})"

  attribute_mapping:
    username: "sAMAccountName"
    email: "mail"
    first_name: "givenName"
    last_name: "sn"
    display_name: "displayName"
    groups: "memberOf"

  sync:
    enabled: true
    interval: 3600  # seconds
    full_sync_interval: 86400  # daily
```

---

## API Security

### API Key Management

```yaml
api_keys:
  # Key generation
  generation:
    algorithm: "secure_random"
    length: 32
    prefix: "pact_"
    encoding: "base62"

  # Key structure
  format: "pact_{environment}_{random}"
  # Example: pact_prod_A1b2C3d4E5f6G7h8I9j0K1l2M3n4O5p6

  # Security policies
  policies:
    max_keys_per_user: 5
    expiry_required: true
    max_expiry_days: 365
    rotation_reminder_days: 30

  # Rate limiting by key
  rate_limits:
    default:
      requests_per_second: 100
      requests_per_day: 100000

    tiers:
      starter:
        requests_per_second: 10
        requests_per_day: 10000
      professional:
        requests_per_second: 100
        requests_per_day: 100000
      enterprise:
        requests_per_second: 1000
        requests_per_day: 10000000

  # IP restrictions
  ip_restrictions:
    enabled: true
    max_ips_per_key: 10
```

### JWT Token Configuration

```yaml
jwt:
  # Access tokens
  access_token:
    algorithm: "RS256"
    issuer: "https://arka.company.com"
    audience: ["arka-api"]
    expiry: 900  # 15 minutes

    claims:
      - sub       # Subject (user ID)
      - iat       # Issued at
      - exp       # Expiry
      - iss       # Issuer
      - aud       # Audience
      - jti       # JWT ID (for revocation)
      - tenant_id # Tenant context
      - roles     # User roles
      - permissions # Specific permissions

  # Refresh tokens
  refresh_token:
    algorithm: "RS256"
    expiry: 604800  # 7 days
    rotation: true
    reuse_detection: true

    # Refresh token families for revocation
    family_tracking: true

  # Token signing keys
  keys:
    rotation_interval: 86400  # 24 hours
    overlap_period: 3600      # 1 hour
    storage: "vault"          # HSM/Vault for production
```

---

## Service-to-Service Authentication

### mTLS Configuration

```yaml
mtls:
  enabled: true

  # Certificate configuration
  certificates:
    ca_cert: "/etc/pact/certs/ca.crt"
    server_cert: "/etc/pact/certs/server.crt"
    server_key: "/etc/pact/certs/server.key"

    # Client verification
    client_verification: required
    allowed_cn_patterns:
      - "*.arka-services.internal"
      - "arka-*"

  # Certificate rotation
  rotation:
    enabled: true
    method: "spiffe"  # or "cert-manager"
    ttl: 86400        # 24 hours

  # SPIFFE/SPIRE integration
  spiffe:
    enabled: true
    trust_domain: "arka.company.com"
    socket_path: "/run/spire/sockets/agent.sock"
```

### Service Mesh Integration

```yaml
# Istio service mesh configuration
service_mesh:
  type: "istio"

  # mTLS mode
  mtls_mode: STRICT

  # Authorization policies
  authorization_policies:
    - name: "arka-core-access"
      rules:
        - from:
            source:
              principals:
                - "cluster.local/ns/pact/sa/api-gateway"
                - "cluster.local/ns/pact/sa/worker"
          to:
            operation:
              methods: ["GET", "POST"]
              paths: ["/api/*"]

    - name: "database-access"
      rules:
        - from:
            source:
              principals:
                - "cluster.local/ns/pact/sa/arka-core"
          to:
            operation:
              ports: ["5432"]
```

---

## Session Management

### Session Configuration

```yaml
sessions:
  # Session store
  store:
    type: "redis"
    prefix: "pact:session:"
    encryption: true

  # Session settings
  settings:
    idle_timeout: 1800       # 30 minutes
    absolute_timeout: 28800  # 8 hours
    renewal_threshold: 300   # 5 minutes before expiry

  # Cookie settings
  cookie:
    name: "pact_session"
    secure: true
    http_only: true
    same_site: "strict"
    path: "/"
    domain: ".company.com"

  # Concurrent sessions
  concurrent:
    max_sessions: 3
    enforcement: "oldest_logout"  # or "newest_denied"

  # Session binding
  binding:
    ip_binding: "subnet"  # none, exact, subnet
    device_binding: true
    user_agent_binding: false
```

### Session Security

```yaml
session_security:
  # Session fixation prevention
  regeneration:
    on_login: true
    on_privilege_escalation: true
    on_role_change: true

  # Session invalidation
  invalidation:
    on_password_change: true
    on_role_change: optional
    on_security_event: true

  # Suspicious activity
  anomaly_detection:
    enabled: true
    triggers:
      - ip_change
      - device_change
      - unusual_time
      - concurrent_login_attempt
    action: "step_up_auth"
```

---

## Audit & Monitoring

### Authentication Events

```yaml
audit:
  authentication:
    events:
      - login_success
      - login_failure
      - logout
      - mfa_challenge
      - mfa_success
      - mfa_failure
      - password_change
      - password_reset
      - session_created
      - session_expired
      - session_revoked
      - token_issued
      - token_refreshed
      - token_revoked

    # Event details
    capture:
      - timestamp
      - user_id
      - ip_address
      - user_agent
      - geolocation
      - auth_method
      - result
      - failure_reason
      - session_id
```

### Authorization Events

```yaml
audit:
  authorization:
    events:
      - permission_granted
      - permission_denied
      - role_assigned
      - role_revoked
      - privilege_escalation
      - sensitive_access

    # Event details
    capture:
      - timestamp
      - user_id
      - resource
      - action
      - decision
      - policy_applied
      - context
```

### Monitoring & Alerting

```yaml
monitoring:
  # Metrics
  metrics:
    - authentication_attempts_total
    - authentication_failures_total
    - active_sessions_count
    - token_issuance_rate
    - mfa_usage_rate

  # Alerts
  alerts:
    - name: "brute_force_detection"
      condition: "authentication_failures > 10 in 5m"
      severity: "high"
      action: "block_ip"

    - name: "anomalous_login"
      condition: "impossible_travel_detected"
      severity: "medium"
      action: "require_step_up"

    - name: "privilege_escalation"
      condition: "admin_role_assigned"
      severity: "medium"
      action: "notify_security"
```

---

## Implementation Guide

### Prerequisites

1. **Identity Provider** - SAML/OIDC IdP configured
2. **Certificate Authority** - For mTLS certificates
3. **Secret Management** - Vault or cloud secrets manager
4. **Redis** - For session storage

### Step 1: Configure IdP Integration

```typescript
// config/auth.ts
import { AuthConfig } from '@arka/core';

export const authConfig: AuthConfig = {
  providers: {
    oidc: {
      issuer: process.env.OIDC_ISSUER,
      clientId: process.env.OIDC_CLIENT_ID,
      clientSecret: process.env.OIDC_CLIENT_SECRET,
      scopes: ['openid', 'profile', 'email', 'groups'],
    },
  },

  session: {
    store: 'redis',
    redisUrl: process.env.REDIS_URL,
    cookieSecure: true,
    cookieSameSite: 'strict',
  },

  mfa: {
    required: true,
    methods: ['totp', 'webauthn'],
  },
};
```

### Step 2: Define Roles and Permissions

```typescript
// config/rbac.ts
import { RBACConfig } from '@arka/core';

export const rbacConfig: RBACConfig = {
  roles: {
    admin: {
      permissions: ['*'],
    },
    compliance_analyst: {
      permissions: [
        'rules:read',
        'transactions:read',
        'alerts:manage',
      ],
    },
  },

  defaultRole: 'viewer',
};
```

### Step 3: Enable mTLS for Services

```yaml
# kubernetes/mtls-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: arka-mtls
  namespace: pact
spec:
  mtls:
    mode: STRICT
```

### Step 4: Configure API Security

```typescript
// middleware/auth.ts
import { authMiddleware, requirePermission } from '@arka/auth';

app.use('/api', authMiddleware({
  jwtSecret: process.env.JWT_SECRET,
  issuer: 'https://arka.company.com',
}));

app.get('/api/rules',
  requirePermission('rules:read'),
  rulesController.list
);
```

---

## Compliance Mapping

| Requirement | Implementation |
|-------------|----------------|
| NIST 800-63B | MFA, password policy |
| NIST 800-207 | Zero-trust architecture |
| SOC 2 CC6.1 | Access controls |
| ISO 27001 A.9 | Access management |
| PCI DSS 8 | Identity and authentication |
| GDPR Art. 32 | Access security |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Security Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
