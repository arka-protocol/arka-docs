# ARKA Engine IAM & SSO Integration Guide

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Integration

---

## Overview

ARKA Engine provides enterprise-grade identity and access management (IAM) integration, supporting industry-standard protocols for Single Sign-On (SSO), user provisioning, and access control. This guide covers integration with SAML 2.0, OAuth 2.0/OIDC, and SCIM 2.0.

---

## Table of Contents

1. [Supported Protocols](#supported-protocols)
2. [SAML 2.0 Integration](#saml-20-integration)
3. [OAuth 2.0 / OIDC Integration](#oauth-20--oidc-integration)
4. [SCIM 2.0 User Provisioning](#scim-20-user-provisioning)
5. [Identity Provider Configuration](#identity-provider-configuration)
6. [Role Mapping](#role-mapping)
7. [Multi-Factor Authentication](#multi-factor-authentication)
8. [Session Management](#session-management)
9. [Troubleshooting](#troubleshooting)

---

## Supported Protocols

| Protocol | Version | Use Case |
|----------|---------|----------|
| SAML 2.0 | 2.0 | Enterprise SSO, Federation |
| OAuth 2.0 | 2.0 | API Authorization, Mobile Apps |
| OpenID Connect | 1.0 | Authentication, User Info |
| SCIM | 2.0 | User Provisioning, Sync |
| LDAP | v3 | Directory Integration |

### Supported Identity Providers

| Provider | SAML | OIDC | SCIM | Status |
|----------|------|------|------|--------|
| Okta | ✅ | ✅ | ✅ | Certified |
| Azure AD | ✅ | ✅ | ✅ | Certified |
| Auth0 | ✅ | ✅ | ✅ | Certified |
| Google Workspace | ✅ | ✅ | ✅ | Certified |
| OneLogin | ✅ | ✅ | ✅ | Certified |
| Ping Identity | ✅ | ✅ | ✅ | Certified |
| AWS IAM Identity Center | ✅ | ✅ | ✅ | Certified |
| Keycloak | ✅ | ✅ | ✅ | Supported |
| Active Directory | ADFS | - | - | Supported |

---

## SAML 2.0 Integration

### ARKA Engine SAML Configuration

#### Service Provider Metadata

```xml
<?xml version="1.0" encoding="UTF-8"?>
<md:EntityDescriptor
    xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
    entityID="https://api.arka-engine.io/saml/metadata">

    <md:SPSSODescriptor
        AuthnRequestsSigned="true"
        WantAssertionsSigned="true"
        protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">

        <md:KeyDescriptor use="signing">
            <ds:KeyInfo xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
                <ds:X509Data>
                    <ds:X509Certificate><!-- Your Certificate --></ds:X509Certificate>
                </ds:X509Data>
            </ds:KeyInfo>
        </md:KeyDescriptor>

        <md:KeyDescriptor use="encryption">
            <ds:KeyInfo xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
                <ds:X509Data>
                    <ds:X509Certificate><!-- Your Certificate --></ds:X509Certificate>
                </ds:X509Data>
            </ds:KeyInfo>
        </md:KeyDescriptor>

        <md:SingleLogoutService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            Location="https://api.arka-engine.io/saml/slo"/>

        <md:NameIDFormat>
            urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress
        </md:NameIDFormat>

        <md:AssertionConsumerService
            Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
            Location="https://api.arka-engine.io/saml/acs"
            index="0"
            isDefault="true"/>

    </md:SPSSODescriptor>
</md:EntityDescriptor>
```

#### SAML Endpoints

| Endpoint | URL | Method |
|----------|-----|--------|
| Metadata | `https://api.arka-engine.io/saml/metadata` | GET |
| ACS (Assertion Consumer Service) | `https://api.arka-engine.io/saml/acs` | POST |
| SLO (Single Logout) | `https://api.arka-engine.io/saml/slo` | POST |
| Login Initiation | `https://api.arka-engine.io/saml/login` | GET |

### Environment Configuration

```yaml
# config/saml.yaml
saml:
  enabled: true
  strict: true
  debug: false

  sp:
    entityId: "https://api.arka-engine.io/saml/metadata"
    assertionConsumerService:
      url: "https://api.arka-engine.io/saml/acs"
      binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
    singleLogoutService:
      url: "https://api.arka-engine.io/saml/slo"
      binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
    nameIdFormat: "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
    x509cert: "${ARKA_SAML_SP_CERT}"
    privateKey: "${ARKA_SAML_SP_KEY}"

  idp:
    entityId: "${ARKA_SAML_IDP_ENTITY_ID}"
    singleSignOnService:
      url: "${ARKA_SAML_IDP_SSO_URL}"
      binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    singleLogoutService:
      url: "${ARKA_SAML_IDP_SLO_URL}"
      binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
    x509cert: "${ARKA_SAML_IDP_CERT}"

  security:
    authnRequestsSigned: true
    logoutRequestSigned: true
    logoutResponseSigned: true
    wantAssertionsSigned: true
    wantAssertionsEncrypted: true
    wantNameIdEncrypted: false
    signatureAlgorithm: "http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"
    digestAlgorithm: "http://www.w3.org/2001/04/xmlenc#sha256"

  attributeMapping:
    email: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
    firstName: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname"
    lastName: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname"
    groups: "http://schemas.xmlsoap.org/claims/Group"
    role: "http://schemas.arka-engine.io/claims/role"
```

### Required SAML Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `email` | Yes | User email address (NameID) |
| `firstName` | Yes | User first name |
| `lastName` | Yes | User last name |
| `groups` | No | Group memberships for role mapping |
| `role` | No | Direct role assignment |
| `department` | No | Department for access policies |
| `employeeId` | No | Employee identifier |

---

## OAuth 2.0 / OIDC Integration

### Supported Grant Types

| Grant Type | Use Case | Status |
|------------|----------|--------|
| Authorization Code | Web Applications | ✅ Recommended |
| Authorization Code + PKCE | SPAs, Mobile Apps | ✅ Recommended |
| Client Credentials | Service-to-Service | ✅ Supported |
| Refresh Token | Token Renewal | ✅ Supported |
| Device Code | CLI, IoT | ✅ Supported |

### OIDC Configuration

#### Well-Known Configuration

```
GET https://api.arka-engine.io/.well-known/openid-configuration
```

```json
{
  "issuer": "https://api.arka-engine.io",
  "authorization_endpoint": "https://api.arka-engine.io/oauth/authorize",
  "token_endpoint": "https://api.arka-engine.io/oauth/token",
  "userinfo_endpoint": "https://api.arka-engine.io/oauth/userinfo",
  "jwks_uri": "https://api.arka-engine.io/.well-known/jwks.json",
  "revocation_endpoint": "https://api.arka-engine.io/oauth/revoke",
  "introspection_endpoint": "https://api.arka-engine.io/oauth/introspect",
  "end_session_endpoint": "https://api.arka-engine.io/oauth/logout",
  "registration_endpoint": "https://api.arka-engine.io/oauth/register",
  "scopes_supported": [
    "openid",
    "profile",
    "email",
    "offline_access",
    "pact:read",
    "pact:write",
    "pact:admin",
    "rules:read",
    "rules:write",
    "audit:read"
  ],
  "response_types_supported": [
    "code",
    "token",
    "id_token",
    "code token",
    "code id_token",
    "token id_token",
    "code token id_token"
  ],
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "client_credentials",
    "urn:ietf:params:oauth:grant-type:device_code"
  ],
  "subject_types_supported": ["public", "pairwise"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post",
    "private_key_jwt"
  ],
  "claims_supported": [
    "sub",
    "iss",
    "aud",
    "exp",
    "iat",
    "auth_time",
    "nonce",
    "email",
    "email_verified",
    "name",
    "given_name",
    "family_name",
    "picture",
    "locale",
    "roles",
    "groups",
    "tenant_id"
  ],
  "code_challenge_methods_supported": ["S256"]
}
```

### OAuth Client Registration

```bash
# Register a new OAuth client
curl -X POST https://api.arka-engine.io/oauth/register \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "My Application",
    "redirect_uris": [
      "https://myapp.example.com/callback",
      "https://myapp.example.com/silent-callback"
    ],
    "grant_types": ["authorization_code", "refresh_token"],
    "response_types": ["code"],
    "token_endpoint_auth_method": "client_secret_basic",
    "scope": "openid profile email pact:read pact:write",
    "contacts": ["admin@example.com"],
    "logo_uri": "https://myapp.example.com/logo.png",
    "tos_uri": "https://myapp.example.com/tos",
    "policy_uri": "https://myapp.example.com/privacy"
  }'
```

Response:
```json
{
  "client_id": "pact_client_abc123",
  "client_secret": "pact_secret_xyz789",
  "client_id_issued_at": 1701388800,
  "client_secret_expires_at": 0,
  "registration_access_token": "reg_token_...",
  "registration_client_uri": "https://api.arka-engine.io/oauth/register/pact_client_abc123"
}
```

### Authorization Code Flow with PKCE

```typescript
// TypeScript example for SPA integration
import { createPKCEChallenge, generateRandomState } from '@arka/auth-utils';

async function initiateLogin() {
  const state = generateRandomState();
  const { codeVerifier, codeChallenge } = await createPKCEChallenge();

  // Store for later verification
  sessionStorage.setItem('oauth_state', state);
  sessionStorage.setItem('code_verifier', codeVerifier);

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: 'your_client_id',
    redirect_uri: 'https://yourapp.com/callback',
    scope: 'openid profile email pact:read pact:write',
    state: state,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    prompt: 'consent'
  });

  window.location.href = `https://api.arka-engine.io/oauth/authorize?${params}`;
}

async function handleCallback(code: string, state: string) {
  // Verify state
  const savedState = sessionStorage.getItem('oauth_state');
  if (state !== savedState) {
    throw new Error('State mismatch - possible CSRF attack');
  }

  const codeVerifier = sessionStorage.getItem('code_verifier');

  // Exchange code for tokens
  const response = await fetch('https://api.arka-engine.io/oauth/token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      client_id: 'your_client_id',
      code: code,
      redirect_uri: 'https://yourapp.com/callback',
      code_verifier: codeVerifier!
    })
  });

  const tokens = await response.json();
  return tokens;
}
```

### Environment Configuration for OAuth/OIDC

```yaml
# config/oauth.yaml
oauth:
  enabled: true

  providers:
    okta:
      enabled: true
      issuer: "https://your-org.okta.com/oauth2/default"
      clientId: "${OKTA_CLIENT_ID}"
      clientSecret: "${OKTA_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email", "groups"]
      attributeMapping:
        id: "sub"
        email: "email"
        name: "name"
        groups: "groups"

    azure:
      enabled: true
      issuer: "https://login.microsoftonline.com/${AZURE_TENANT_ID}/v2.0"
      clientId: "${AZURE_CLIENT_ID}"
      clientSecret: "${AZURE_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email", "User.Read"]
      attributeMapping:
        id: "oid"
        email: "email"
        name: "name"
        groups: "groups"

    google:
      enabled: true
      issuer: "https://accounts.google.com"
      clientId: "${GOOGLE_CLIENT_ID}"
      clientSecret: "${GOOGLE_CLIENT_SECRET}"
      scopes: ["openid", "profile", "email"]
      hostedDomain: "your-company.com"  # Restrict to domain

  jwt:
    algorithm: "RS256"
    publicKeyUrl: "${ARKA_JWT_JWKS_URL}"
    issuer: "https://api.arka-engine.io"
    audience: "arka-engine"
    accessTokenTTL: 3600        # 1 hour
    refreshTokenTTL: 2592000    # 30 days
    idTokenTTL: 3600            # 1 hour
```

---

## SCIM 2.0 User Provisioning

### SCIM Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/scim/v2/Users` | GET | List users |
| `/scim/v2/Users` | POST | Create user |
| `/scim/v2/Users/{id}` | GET | Get user |
| `/scim/v2/Users/{id}` | PUT | Replace user |
| `/scim/v2/Users/{id}` | PATCH | Update user |
| `/scim/v2/Users/{id}` | DELETE | Delete user |
| `/scim/v2/Groups` | GET | List groups |
| `/scim/v2/Groups` | POST | Create group |
| `/scim/v2/Groups/{id}` | GET | Get group |
| `/scim/v2/Groups/{id}` | PUT | Replace group |
| `/scim/v2/Groups/{id}` | PATCH | Update group |
| `/scim/v2/Groups/{id}` | DELETE | Delete group |
| `/scim/v2/ServiceProviderConfig` | GET | Get configuration |
| `/scim/v2/Schemas` | GET | List schemas |
| `/scim/v2/ResourceTypes` | GET | List resource types |

### SCIM Authentication

```bash
# Bearer token authentication
curl -X GET https://api.arka-engine.io/scim/v2/Users \
  -H "Authorization: Bearer ${SCIM_TOKEN}" \
  -H "Content-Type: application/scim+json"
```

### SCIM User Schema

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"],
  "id": "user_abc123",
  "externalId": "emp-12345",
  "userName": "john.doe@example.com",
  "name": {
    "formatted": "John Doe",
    "familyName": "Doe",
    "givenName": "John"
  },
  "displayName": "John Doe",
  "emails": [
    {
      "value": "john.doe@example.com",
      "type": "work",
      "primary": true
    }
  ],
  "active": true,
  "groups": [
    {
      "value": "group_compliance_admin",
      "display": "Compliance Administrators",
      "type": "direct"
    }
  ],
  "roles": [
    {
      "value": "compliance_officer",
      "display": "Compliance Officer",
      "type": "pact_role",
      "primary": true
    }
  ],
  "urn:pact:scim:schemas:1.0:User": {
    "tenantId": "tenant_xyz",
    "department": "Legal & Compliance",
    "permissions": ["rules:read", "rules:write", "audit:read"]
  },
  "meta": {
    "resourceType": "User",
    "created": "2024-01-15T09:30:00Z",
    "lastModified": "2024-12-01T14:22:00Z",
    "location": "https://api.arka-engine.io/scim/v2/Users/user_abc123",
    "version": "W/\"a1b2c3d4\""
  }
}
```

### SCIM Group Schema

```json
{
  "schemas": ["urn:ietf:params:scim:schemas:core:2.0:Group"],
  "id": "group_compliance_admin",
  "externalId": "compliance-admins",
  "displayName": "Compliance Administrators",
  "members": [
    {
      "value": "user_abc123",
      "display": "John Doe",
      "type": "User",
      "$ref": "https://api.arka-engine.io/scim/v2/Users/user_abc123"
    }
  ],
  "urn:pact:scim:schemas:1.0:Group": {
    "roles": ["compliance_admin"],
    "permissions": ["rules:*", "audit:*", "users:read"]
  },
  "meta": {
    "resourceType": "Group",
    "created": "2024-01-01T00:00:00Z",
    "lastModified": "2024-12-01T12:00:00Z",
    "location": "https://api.arka-engine.io/scim/v2/Groups/group_compliance_admin"
  }
}
```

### SCIM Configuration

```yaml
# config/scim.yaml
scim:
  enabled: true
  version: "2.0"

  authentication:
    type: "bearer"
    tokenValidation: "jwt"
    requiredScopes: ["scim:read", "scim:write"]

  features:
    patch: true
    bulk: true
    bulkMaxOperations: 1000
    bulkMaxPayloadSize: 1048576  # 1MB
    filter: true
    filterMaxResults: 200
    changePassword: true
    sort: true
    etag: true

  schemas:
    user:
      - "urn:ietf:params:scim:schemas:core:2.0:User"
      - "urn:pact:scim:schemas:1.0:User"
    group:
      - "urn:ietf:params:scim:schemas:core:2.0:Group"
      - "urn:pact:scim:schemas:1.0:Group"

  provisioning:
    autoCreateTenant: false
    defaultRole: "viewer"
    syncInterval: 300  # 5 minutes

  attributeMapping:
    user:
      externalId: "employeeId"
      email: "emails[type eq 'work'].value"
      firstName: "name.givenName"
      lastName: "name.familyName"
      department: "urn:pact:scim:schemas:1.0:User.department"
```

---

## Identity Provider Configuration

### Okta Configuration

1. **Create SAML Application**
   - Go to Applications → Create App Integration
   - Select SAML 2.0
   - Configure:
     ```
     Single Sign-On URL: https://api.arka-engine.io/saml/acs
     Audience URI: https://api.arka-engine.io/saml/metadata
     Name ID Format: EmailAddress
     ```

2. **Configure Attribute Statements**
   ```
   firstName → user.firstName
   lastName → user.lastName
   email → user.email
   groups → getFilteredGroups({"arka-"}, ...)
   ```

3. **Enable SCIM Provisioning**
   - Go to Provisioning → Integration
   - Enable "Enable API Integration"
   - Configure:
     ```
     SCIM connector base URL: https://api.arka-engine.io/scim/v2
     Authentication mode: HTTP Header
     Header: Authorization: Bearer ${SCIM_TOKEN}
     ```

### Azure AD Configuration

1. **Register Enterprise Application**
   ```powershell
   # Azure CLI
   az ad app create \
     --display-name "ARKA Engine" \
     --identifier-uris "https://api.arka-engine.io" \
     --reply-urls "https://api.arka-engine.io/saml/acs"
   ```

2. **Configure SAML SSO**
   - Entity ID: `https://api.arka-engine.io/saml/metadata`
   - Reply URL: `https://api.arka-engine.io/saml/acs`
   - Sign-on URL: `https://api.arka-engine.io/saml/login`

3. **Configure Claims**
   ```
   email → user.mail
   given_name → user.givenname
   family_name → user.surname
   groups → user.groups
   ```

### Google Workspace Configuration

1. **Create SAML App**
   - Admin Console → Apps → Web and mobile apps
   - Add App → Add custom SAML app

2. **Configure**
   ```
   ACS URL: https://api.arka-engine.io/saml/acs
   Entity ID: https://api.arka-engine.io/saml/metadata
   Start URL: https://api.arka-engine.io
   Name ID: Email
   ```

---

## Role Mapping

### Group-to-Role Mapping

```yaml
# config/role-mapping.yaml
roleMapping:
  enabled: true

  rules:
    # Direct group mappings
    - groupPattern: "arka-admins"
      role: "admin"
      permissions: ["*"]

    - groupPattern: "arka-compliance-officers"
      role: "compliance_officer"
      permissions:
        - "rules:read"
        - "rules:write"
        - "audit:read"
        - "reports:*"

    - groupPattern: "arka-developers"
      role: "developer"
      permissions:
        - "rules:read"
        - "api:read"
        - "api:write"

    - groupPattern: "arka-auditors"
      role: "auditor"
      permissions:
        - "audit:read"
        - "reports:read"

    # Wildcard patterns
    - groupPattern: "arka-tenant-*-admins"
      role: "tenant_admin"
      tenantFromGroup: true
      permissions:
        - "tenant:*"

    # Department-based
    - attributeMatch:
        department: "Legal"
      role: "legal_reviewer"
      permissions:
        - "rules:read"
        - "audit:read"

  defaults:
    role: "viewer"
    permissions:
      - "rules:read"
```

### Custom Role Definition

```typescript
// Define custom roles via API
const customRole = {
  name: "regional_compliance_manager",
  displayName: "Regional Compliance Manager",
  description: "Manages compliance for a specific region",
  permissions: [
    "rules:read",
    "rules:write:region",
    "audit:read:region",
    "reports:generate",
    "users:read:region"
  ],
  constraints: {
    regionRequired: true,
    maxSessions: 3
  }
};

await pactClient.roles.create(customRole);
```

---

## Multi-Factor Authentication

### Supported MFA Methods

| Method | Status | Notes |
|--------|--------|-------|
| TOTP (Authenticator Apps) | ✅ Supported | Google Authenticator, Authy |
| SMS OTP | ✅ Supported | Twilio integration |
| Email OTP | ✅ Supported | Built-in |
| Push Notifications | ✅ Supported | Mobile app required |
| WebAuthn/FIDO2 | ✅ Supported | Hardware keys, biometrics |
| Backup Codes | ✅ Supported | One-time recovery codes |

### MFA Configuration

```yaml
# config/mfa.yaml
mfa:
  enabled: true
  enforcement: "required"  # "optional", "required", "risk-based"

  methods:
    totp:
      enabled: true
      issuer: "ARKA Engine"
      digits: 6
      period: 30
      algorithm: "SHA1"

    sms:
      enabled: true
      provider: "twilio"
      twilioAccountSid: "${TWILIO_ACCOUNT_SID}"
      twilioAuthToken: "${TWILIO_AUTH_TOKEN}"
      twilioFromNumber: "${TWILIO_FROM_NUMBER}"

    webauthn:
      enabled: true
      rpName: "ARKA Engine"
      rpId: "arka-engine.io"
      attestation: "none"
      userVerification: "preferred"

  policies:
    admin:
      required: true
      methods: ["totp", "webauthn"]
      rememberDevice: false

    default:
      required: true
      methods: ["totp", "sms", "webauthn"]
      rememberDevice: true
      rememberDuration: 2592000  # 30 days

  riskBased:
    enabled: true
    triggers:
      - newDevice: true
      - newLocation: true
      - unusualTime: true
      - suspiciousIP: true
```

---

## Session Management

### Session Configuration

```yaml
# config/session.yaml
session:
  store: "redis"

  cookie:
    name: "pact_session"
    secure: true
    httpOnly: true
    sameSite: "strict"
    domain: ".arka-engine.io"
    path: "/"

  timeout:
    idle: 1800         # 30 minutes
    absolute: 28800    # 8 hours
    renewal: 900       # 15 minutes

  concurrency:
    maxSessions: 5
    strategy: "kick-oldest"  # or "deny-new"

  security:
    rotateOnAuth: true
    bindToIP: false
    bindToUserAgent: true

  events:
    onLogin: true
    onLogout: true
    onExpiry: true
    onConcurrencyViolation: true
```

### Session Management API

```bash
# List active sessions
curl -X GET https://api.arka-engine.io/api/v1/users/me/sessions \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# Revoke a specific session
curl -X DELETE https://api.arka-engine.io/api/v1/users/me/sessions/{session_id} \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# Revoke all sessions (logout everywhere)
curl -X DELETE https://api.arka-engine.io/api/v1/users/me/sessions \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

---

## Troubleshooting

### Common SAML Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Invalid Signature | Certificate mismatch | Update IdP certificate |
| Clock Skew | Time difference > 5 min | Sync server time with NTP |
| Destination Mismatch | Wrong ACS URL | Verify ACS URL in IdP |
| Audience Mismatch | Wrong Entity ID | Verify SP Entity ID |
| Missing Attributes | Attribute not mapped | Configure attribute statements |

### Debug Mode

```yaml
# Enable debug logging
logging:
  level:
    auth: DEBUG
    saml: DEBUG
    oauth: DEBUG
    scim: DEBUG

# Enable SAML debug
saml:
  debug: true
  logSamlMessages: true
```

### SAML Trace

```bash
# Decode SAML Response
echo "${SAML_RESPONSE}" | base64 -d | xmllint --format -

# Verify signature
xmlsec1 --verify --id-attr:ID urn:oasis:names:tc:SAML:2.0:assertion:Assertion \
  --pubkey-cert-pem idp-cert.pem saml-response.xml
```

### OAuth/OIDC Debugging

```bash
# Decode JWT
echo "${ACCESS_TOKEN}" | cut -d'.' -f2 | base64 -d | jq .

# Introspect token
curl -X POST https://api.arka-engine.io/oauth/introspect \
  -u "${CLIENT_ID}:${CLIENT_SECRET}" \
  -d "token=${ACCESS_TOKEN}"
```

---

## API Reference

### Authentication Endpoints

```yaml
endpoints:
  saml:
    metadata: GET /saml/metadata
    login: GET /saml/login
    acs: POST /saml/acs
    slo: POST /saml/slo

  oauth:
    authorize: GET /oauth/authorize
    token: POST /oauth/token
    userinfo: GET /oauth/userinfo
    revoke: POST /oauth/revoke
    introspect: POST /oauth/introspect
    jwks: GET /.well-known/jwks.json
    openidConfig: GET /.well-known/openid-configuration

  scim:
    users: /scim/v2/Users
    groups: /scim/v2/Groups
    schemas: GET /scim/v2/Schemas
    config: GET /scim/v2/ServiceProviderConfig
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Engineering Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
