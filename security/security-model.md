# Security Model

This document describes the comprehensive security model of the ARKA Engine, covering authentication, authorization, cryptography, and security best practices.

## Overview

The ARKA Engine implements defense-in-depth security:

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Perimeter Security                          ││
│  │  - WAF, DDoS Protection, Rate Limiting                  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Network Security                            ││
│  │  - mTLS, Network Policies, Segmentation                 ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Identity & Access                           ││
│  │  - Authentication, Authorization, RBAC                  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Application Security                        ││
│  │  - Input Validation, Sandbox Isolation                  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Data Security                               ││
│  │  - Encryption, Key Management, Data Classification      ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Audit & Compliance                          ││
│  │  - Logging, Monitoring, Compliance Controls             ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Authentication

### API Key Authentication

```yaml
# API key configuration
authentication:
  apiKeys:
    enabled: true
    header: "X-API-Key"
    hashAlgorithm: sha256
    rateLimitPerKey: 1000/minute

    # Key rotation settings
    rotation:
      enabled: true
      gracePeriod: 7d
      notifyDaysBefore: 14
```

```python
from pact.security import APIKeyManager

key_manager = APIKeyManager()

# Generate API key
key_info = key_manager.generate(
    name="payment-service",
    scopes=["evaluate", "rules:read"],
    expires_in="365d",
    rate_limit=1000
)

print(f"API Key: {key_info.key}")
print(f"Key ID: {key_info.key_id}")
```

### OAuth 2.0 / OIDC

```yaml
# OAuth configuration
authentication:
  oauth:
    enabled: true
    providers:
      - name: okta
        issuer: https://company.okta.com
        clientId: ${OKTA_CLIENT_ID}
        clientSecret: ${OKTA_CLIENT_SECRET}
        scopes:
          - openid
          - profile
          - pact:evaluate
          - pact:rules

      - name: azure-ad
        issuer: https://login.microsoftonline.com/${TENANT_ID}/v2.0
        clientId: ${AZURE_CLIENT_ID}
        audience: api://arka-engine

    # Token validation
    tokenValidation:
      algorithms: [RS256, ES256]
      clockSkewSeconds: 60
      requireExp: true
      requireIat: true
```

### mTLS Client Authentication

```yaml
# mTLS configuration
authentication:
  mtls:
    enabled: true
    clientCertRequired: true

    # CA certificates
    trustedCAs:
      - /certs/ca.crt
      - /certs/intermediate-ca.crt

    # Certificate validation
    validation:
      checkRevocation: true
      ocspEnabled: true
      crlEndpoints:
        - https://crl.company.com/pact.crl

    # Client certificate mapping
    clientMapping:
      extractPrincipal: subject.CN
      extractRoles: subject.OU
```

### Service Mesh Authentication

```yaml
# Istio mTLS configuration
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: arka-mtls
  namespace: compliance
spec:
  selector:
    matchLabels:
      app: arka-engine
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: arka-authz
  namespace: compliance
spec:
  selector:
    matchLabels:
      app: arka-engine
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/payments/sa/payment-service
              - cluster.local/ns/compliance/sa/compliance-worker
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/evaluate"]
```

## Authorization

### Role-Based Access Control (RBAC)

```yaml
# RBAC configuration
authorization:
  rbac:
    enabled: true

    # Role definitions
    roles:
      # Admin role - full access
      admin:
        permissions:
          - "*"

      # Compliance officer - manage rules and governance
      compliance_officer:
        permissions:
          - rules:read
          - rules:create
          - rules:update
          - rules:delete
          - governance:*
          - audit:read
          - evaluate:*

      # Developer - limited rule management
      developer:
        permissions:
          - rules:read
          - rules:validate
          - evaluate:*
          - audit:read

      # Service - API access only
      service:
        permissions:
          - evaluate:*
          - rules:read
          - proofs:read

      # Auditor - read-only access
      auditor:
        permissions:
          - rules:read
          - audit:read
          - proofs:read
          - governance:read

    # Role assignments
    assignments:
      users:
        admin@company.com: [admin]
        compliance@company.com: [compliance_officer]
        dev@company.com: [developer]

      groups:
        compliance-team: [compliance_officer]
        engineering: [developer]
        audit-team: [auditor]

      services:
        payment-service: [service]
        reporting-service: [auditor]
```

### Attribute-Based Access Control (ABAC)

```yaml
# ABAC policies
authorization:
  abac:
    enabled: true
    policies:
      # Jurisdiction-based access
      - name: jurisdiction-access
        description: "Users can only access rules for their jurisdiction"
        condition: |
          resource.jurisdiction in user.allowed_jurisdictions

      # Domain-based access
      - name: domain-access
        description: "Users can only access rules for their domain"
        condition: |
          resource.domain in user.allowed_domains

      # Time-based access
      - name: business-hours
        description: "Certain operations only during business hours"
        condition: |
          action in ['rules:delete', 'governance:approve']
          implies
          time.hour >= 9 and time.hour <= 17 and time.weekday < 5

      # Classification-based access
      - name: classification-access
        description: "Access based on data classification"
        condition: |
          resource.classification == 'public'
          or
          resource.classification in user.clearance_levels
```

### Permission Enforcement

```python
from pact.security import PermissionChecker, require_permission

checker = PermissionChecker()

# Decorator-based enforcement
@require_permission("rules:create")
async def create_rule(request: CreateRuleRequest):
    # Implementation
    pass

# Programmatic enforcement
async def update_rule(rule_id: str, request: UpdateRuleRequest, user: User):
    # Check base permission
    if not await checker.has_permission(user, "rules:update"):
        raise PermissionDenied("Cannot update rules")

    # Check resource-specific permission
    rule = await get_rule(rule_id)
    if not await checker.can_access_resource(user, rule):
        raise PermissionDenied("Cannot access this rule")

    # Check ABAC policy
    if not await checker.evaluate_policy(user, "jurisdiction-access", rule):
        raise PermissionDenied("Not authorized for this jurisdiction")

    # Proceed with update
    return await perform_update(rule_id, request)
```

## Cryptography

### Key Management

```yaml
# Key management configuration
cryptography:
  keyManagement:
    provider: vault
    config:
      address: https://vault.company.com:8200
      authMethod: kubernetes
      role: arka-engine
      namespace: pact

    # Key types
    keys:
      signing:
        algorithm: Ed25519
        purpose: proof_signing
        rotation: 90d

      encryption:
        algorithm: AES-256-GCM
        purpose: data_encryption
        rotation: 180d

      hmac:
        algorithm: HMAC-SHA256
        purpose: api_signatures
        rotation: 30d

    # HSM configuration (optional)
    hsm:
      enabled: true
      provider: aws-cloudhsm
      clusterId: ${HSM_CLUSTER_ID}
      keyLabel: arka-signing-key
```

### Encryption at Rest

```yaml
# Data encryption configuration
cryptography:
  encryption:
    atRest:
      enabled: true
      algorithm: AES-256-GCM
      keyDerivation: HKDF-SHA256

      # Encrypt specific fields
      sensitiveFields:
        - transaction.account_number
        - transaction.routing_number
        - customer.ssn
        - customer.tax_id

      # Database encryption
      database:
        provider: transparent
        keySource: vault
        keyRotation: enabled

      # File encryption
      files:
        algorithm: AES-256-GCM
        keyWrapping: RSA-OAEP
```

### Encryption in Transit

```yaml
# TLS configuration
cryptography:
  tls:
    minVersion: "1.3"
    cipherSuites:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_AES_128_GCM_SHA256

    # Certificate configuration
    certificates:
      certFile: /certs/server.crt
      keyFile: /certs/server.key
      caFile: /certs/ca.crt

    # HSTS
    hsts:
      enabled: true
      maxAge: 31536000
      includeSubDomains: true
      preload: true
```

### Digital Signatures

```python
from pact.security.crypto import SignatureService

signer = SignatureService(
    algorithm="Ed25519",
    key_provider=vault_provider
)

# Sign proof
signature = await signer.sign(
    data=proof.canonical_bytes(),
    key_id="signing-key-001"
)

# Verify signature
is_valid = await signer.verify(
    data=proof.canonical_bytes(),
    signature=signature,
    public_key=agent_public_key
)
```

## Data Protection

### Data Classification

```yaml
# Data classification policy
dataProtection:
  classification:
    levels:
      - name: public
        description: "Publicly available information"
        controls: []

      - name: internal
        description: "Internal use only"
        controls:
          - authentication_required

      - name: confidential
        description: "Sensitive business data"
        controls:
          - authentication_required
          - encryption_required
          - audit_logging

      - name: restricted
        description: "Highly sensitive/regulated data"
        controls:
          - authentication_required
          - encryption_required
          - audit_logging
          - access_logging
          - mfa_required
          - data_masking

    # Field classifications
    fields:
      transaction.id: internal
      transaction.amount: confidential
      transaction.account_number: restricted
      customer.name: confidential
      customer.ssn: restricted
      customer.email: confidential
```

### Data Masking

```python
from pact.security import DataMasker, MaskingRule

masker = DataMasker(rules=[
    MaskingRule(
        field="account_number",
        pattern=r"\d{10,16}",
        mask_type="partial",
        visible_chars=4,
        mask_char="*"
    ),
    MaskingRule(
        field="ssn",
        pattern=r"\d{3}-\d{2}-\d{4}",
        mask_type="full",
        replacement="***-**-****"
    ),
    MaskingRule(
        field="email",
        pattern=r"(.+)@(.+)",
        mask_type="partial_email"
    )
])

# Mask sensitive data
masked_data = masker.mask({
    "account_number": "1234567890123456",
    "ssn": "123-45-6789",
    "email": "user@company.com"
})

# Result:
# {
#   "account_number": "************3456",
#   "ssn": "***-**-****",
#   "email": "u***@company.com"
# }
```

### Data Retention

```yaml
# Data retention policy
dataProtection:
  retention:
    policies:
      # Evaluation results
      evaluations:
        hot: 30d
        warm: 90d
        cold: 365d
        delete: 7y

      # Trust proofs
      proofs:
        hot: 90d
        warm: 365d
        cold: never
        delete: never  # Regulatory requirement

      # Audit logs
      auditLogs:
        hot: 90d
        warm: 365d
        cold: 7y
        delete: 10y

      # Transaction data
      transactions:
        hot: 30d
        warm: 90d
        cold: 365d
        delete: 7y

    # Automated cleanup
    cleanup:
      enabled: true
      schedule: "0 2 * * *"  # Daily at 2 AM
      batchSize: 10000
```

## Network Security

### Network Policies

```yaml
# Kubernetes network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: arka-engine-policy
  namespace: compliance
spec:
  podSelector:
    matchLabels:
      app: arka-engine
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow from API gateway
    - from:
        - namespaceSelector:
            matchLabels:
              name: api-gateway
          podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

    # Allow from internal services
    - from:
        - namespaceSelector:
            matchLabels:
              name: payments
        - namespaceSelector:
            matchLabels:
              name: compliance
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 50051

  egress:
    # Allow to database
    - to:
        - namespaceSelector:
            matchLabels:
              name: data
          podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow to Redis
    - to:
        - namespaceSelector:
            matchLabels:
              name: data
          podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379

    # Allow to control plane
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 443

    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### Web Application Firewall

```yaml
# WAF rules
waf:
  enabled: true
  provider: cloudflare

  rules:
    # Block SQL injection
    - id: sql-injection
      action: block
      conditions:
        - field: body
          operator: contains
          patterns:
            - "SELECT.*FROM"
            - "INSERT.*INTO"
            - "UPDATE.*SET"
            - "DELETE.*FROM"
            - "DROP.*TABLE"

    # Block XSS
    - id: xss-prevention
      action: block
      conditions:
        - field: body
          operator: contains
          patterns:
            - "<script"
            - "javascript:"
            - "onerror="
            - "onload="

    # Rate limiting
    - id: rate-limit
      action: challenge
      conditions:
        - field: ip
          operator: rate_exceeds
          threshold: 100
          period: 60

    # Geo blocking (if required)
    - id: geo-block
      action: block
      conditions:
        - field: ip.geoip.country
          operator: in
          values: [KP, IR, SY, CU]
```

## Application Security

### Input Validation

```python
from pact.security import InputValidator, ValidationRule
from pydantic import BaseModel, validator

class TransactionInput(BaseModel):
    """Validated transaction input."""
    id: str
    amount: float
    currency: str
    type: str
    source: dict
    destination: dict

    @validator('id')
    def validate_id(cls, v):
        if not re.match(r'^[a-zA-Z0-9\-]{1,64}$', v):
            raise ValueError('Invalid transaction ID format')
        return v

    @validator('amount')
    def validate_amount(cls, v):
        if v <= 0 or v > 10_000_000_000:
            raise ValueError('Amount out of range')
        return v

    @validator('currency')
    def validate_currency(cls, v):
        allowed = ['USD', 'EUR', 'GBP', 'JPY', 'CHF']
        if v not in allowed:
            raise ValueError(f'Currency must be one of {allowed}')
        return v

    @validator('type')
    def validate_type(cls, v):
        allowed = ['payment', 'wire', 'ach', 'cash', 'check']
        if v not in allowed:
            raise ValueError(f'Type must be one of {allowed}')
        return v

# Custom validator with security checks
validator = InputValidator(
    max_depth=10,
    max_string_length=10000,
    sanitize_html=True,
    check_encoding=True
)

async def process_transaction(raw_input: dict):
    # Validate structure
    validated = validator.validate(raw_input)

    # Parse with Pydantic
    transaction = TransactionInput(**validated)

    return await evaluate(transaction)
```

### Sandbox Isolation

```yaml
# Plugin sandbox configuration
sandbox:
  enabled: true
  runtime: wasm

  # Resource limits
  resources:
    maxCpu: 0.5
    maxMemory: 256Mi
    maxExecutionTime: 30s
    maxFileDescriptors: 100

  # Network restrictions
  network:
    enabled: false
    allowlist:
      - api.trusted-service.com
    denylist:
      - "*.internal"
      - "10.0.0.0/8"
      - "172.16.0.0/12"
      - "192.168.0.0/16"

  # File system restrictions
  filesystem:
    readOnly: true
    allowedPaths:
      - /data/plugins/${PLUGIN_NAME}/read
    tempDir: /tmp/plugins/${PLUGIN_NAME}
    maxTempSize: 10Mi

  # Syscall restrictions
  seccomp:
    profile: strict
    allowed:
      - read
      - write
      - open
      - close
      - mmap
      - munmap
    denied:
      - execve
      - fork
      - clone
      - ptrace
```

## Security Monitoring

### Security Events

```yaml
# Security event logging
security:
  events:
    # Authentication events
    - type: auth.success
      severity: info
      fields: [user, method, ip]

    - type: auth.failure
      severity: warning
      fields: [user, method, ip, reason]

    - type: auth.mfa_challenge
      severity: info
      fields: [user, method]

    # Authorization events
    - type: authz.denied
      severity: warning
      fields: [user, resource, action, reason]

    - type: authz.elevated
      severity: info
      fields: [user, role, reason]

    # Data access events
    - type: data.access
      severity: info
      fields: [user, resource, classification]

    - type: data.export
      severity: warning
      fields: [user, resource, destination]

    # Security incidents
    - type: security.injection_attempt
      severity: critical
      fields: [ip, payload, target]

    - type: security.rate_limit_exceeded
      severity: warning
      fields: [ip, endpoint, rate]
```

### Intrusion Detection

```python
from pact.security import IntrusionDetector, ThreatIndicator

detector = IntrusionDetector(
    indicators=[
        ThreatIndicator(
            name="brute_force",
            pattern="auth.failure",
            threshold=5,
            window="5m",
            action="block_ip"
        ),
        ThreatIndicator(
            name="sql_injection",
            pattern="request.body matches '(?i)(select|insert|update|delete|drop)'",
            threshold=1,
            action="block_and_alert"
        ),
        ThreatIndicator(
            name="data_exfiltration",
            pattern="data.export",
            threshold=100,
            window="1h",
            action="alert"
        ),
    ]
)

# Process security event
async def handle_security_event(event: SecurityEvent):
    threat = await detector.analyze(event)

    if threat:
        await threat.execute_action()
        await alert_security_team(threat)
```

## Compliance Controls

### SOC 2 Controls

```yaml
# SOC 2 compliance controls
compliance:
  soc2:
    # CC6.1 - Logical and Physical Access Controls
    accessControls:
      enabled: true
      mfaRequired: true
      passwordPolicy:
        minLength: 14
        requireUppercase: true
        requireLowercase: true
        requireNumbers: true
        requireSymbols: true
        maxAge: 90d
        historyCount: 12

    # CC6.2 - System Account Management
    accountManagement:
      approvalRequired: true
      reviewFrequency: 90d
      inactiveDisable: 30d
      terminationProcess: automated

    # CC6.3 - Security Monitoring
    monitoring:
      enabled: true
      alerting: true
      auditLogging: true
      intrusionDetection: true

    # CC7.1 - System Operations
    operations:
      changeManagement: enabled
      incidentResponse: enabled
      businessContinuity: enabled
```

### PCI DSS Controls

```yaml
# PCI DSS compliance controls
compliance:
  pciDss:
    # Requirement 3 - Protect Stored Data
    dataProtection:
      encryptionRequired: true
      algorithm: AES-256
      keyManagement: vault
      maskPAN: true

    # Requirement 4 - Encrypt Transmission
    transmission:
      tlsRequired: true
      minVersion: "1.2"
      strongCiphersOnly: true

    # Requirement 7 - Restrict Access
    accessControl:
      needToKnow: true
      roleBasedAccess: true
      uniqueIds: true

    # Requirement 8 - Identify Users
    authentication:
      mfaRequired: true
      passwordComplexity: true
      sessionTimeout: 15m
      lockoutThreshold: 6

    # Requirement 10 - Track Access
    auditTrail:
      enabled: true
      tamperProof: true
      retention: 1y
```

## Security Best Practices

### Development

1. **Secure coding**: Follow OWASP guidelines
2. **Dependency scanning**: Regularly scan dependencies
3. **Code review**: Security-focused code reviews
4. **Static analysis**: Use SAST tools

### Deployment

1. **Least privilege**: Minimal permissions
2. **Secrets management**: Use Vault or similar
3. **Image scanning**: Scan container images
4. **Network segmentation**: Isolate components

### Operations

1. **Patch management**: Regular security updates
2. **Monitoring**: Continuous security monitoring
3. **Incident response**: Documented procedures
4. **Penetration testing**: Regular security assessments

## Security Incident Response

### Incident Classification

| Severity | Description | Response Time |
|----------|-------------|---------------|
| Critical | Active breach, data exfiltration | Immediate |
| High | Vulnerability being exploited | < 1 hour |
| Medium | Potential security issue | < 4 hours |
| Low | Security improvement needed | < 24 hours |

### Response Procedures

```yaml
# Incident response procedures
incidentResponse:
  procedures:
    - severity: critical
      steps:
        - isolate_affected_systems
        - notify_security_team
        - notify_management
        - preserve_evidence
        - begin_investigation
        - notify_affected_parties

    - severity: high
      steps:
        - assess_impact
        - notify_security_team
        - implement_mitigation
        - document_findings

  contacts:
    security_team: security@company.com
    management: ciso@company.com
    legal: legal@company.com
```

## Related Documentation

- [Authentication Guide](./authentication.md)
- [Deployment Guide](../deployment/deployment-guide.md)
- [Monitoring Guide](../deployment/monitoring.md)
- [Compliance Documentation](./compliance.md)
