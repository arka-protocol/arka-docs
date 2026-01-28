# ARKA License Management System

The ARKA License Management System provides cloud-based API key issuance and validation for the ARKA compliance platform. It ensures that only licensed customers can access plugins, jurisdictions, and features.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ARKA License System                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌──────────────────────────────────────┐   │
│  │  Admin CLI      │───▶│     License Authority Service        │   │
│  │  (Python)       │    │     (cloud/services/license-authority)│   │
│  │                 │    │                                      │   │
│  │  arka-admin     │    │  • JWT-based API key generation      │   │
│  │                 │    │  • Cryptographic signing (KMS)       │   │
│  └─────────────────┘    │  • Redis caching                     │   │
│                         │  • PostgreSQL storage                │   │
│  ┌─────────────────┐    │                                      │   │
│  │  Client CLI     │───▶│                                      │   │
│  │  (Python)       │    └──────────────────────────────────────┘   │
│  │                 │                    │                          │
│  │  arka-client    │                    ▼                          │
│  └─────────────────┘    ┌──────────────────────────────────────┐   │
│                         │         Plugin SDK                    │   │
│                         │   (packages/arka-plugin-sdk)         │   │
│                         │                                      │   │
│                         │  • LicenseEnforcer middleware        │   │
│                         │  • Offline validation (JWT)          │   │
│                         │  • 2-day grace period                │   │
│                         └──────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Key Features

### Programmable JWT-Based API Keys

API keys are self-contained JWTs that include:
- Client information (name, email, company, tier)
- Subscriptions (plugins, jurisdictions, features)
- Usage limits (decisions/day, nodes, API calls/min)
- Validity period with configurable grace period

### Offline Validation with Grace Period

When the License Authority is unreachable, the system:
1. Checks local cache for recent validation
2. Decodes JWT payload for embedded license data
3. Allows operation within the 2-day grace period
4. Blocks access only after grace period expires

### Admin API Key Authentication

All administrative operations require authentication with an admin API key, ensuring secure management of customer licenses.

## Quick Start

### 1. Start the License Authority Service

```bash
cd cloud
docker compose up license-authority -d
```

### 2. Bootstrap Admin Key

```bash
# Using CLI
pip install -e too../arka-license-admin
arka-admin config set-url http://localhost:3003
arka-admin config bootstrap

# Or via API
curl -X POST http://localhost:3003/admin/bootstrap \
  -H "Content-Type: application/json" \
  -d '{"name": "Primary Admin"}'
```

### 3. Issue an API Key

```bash
arka-admin key issue \
  --tenant "550e8400-e29b-41d4-a716-446655440000" \
  --name "Acme Corporation" \
  --email "admin@acme.com" \
  --tier enterprise \
  --plugins kyc-verification,sanctions-screening,tax-vault \
  --jurisdictions US,EU,UK \
  --features decision-engine,audit-trail \
  --validity-days 365
```

### 4. Validate an API Key

```bash
# Using Client CLI
pip install -e too../arka-license-client
arka-client configure --key "pact_live_eyJhbGc..."
arka-client validate
arka-client status
```

## API Endpoints

### Public Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Service health check |
| `/api-key/validate` | POST | Validate an API key |

### Admin Endpoints (Require Admin API Key)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api-key/generate` | POST | Generate new API key |
| `/api-key/:keyId` | GET | Get API key details |
| `/api-key/:keyId` | PATCH | Update API key |
| `/api-key/:keyId/revoke` | POST | Revoke API key |
| `/api-key` | GET | List all API keys |
| `/api-key/tenant/:tenantId` | GET | List tenant's API keys |
| `/api-key/expiring/:days` | GET | Get expiring keys |
| `/admin/bootstrap` | POST | Create admin key |

## API Key Structure

```json
{
  "keyId": "550e8400-e29b-41d4-a716-446655440000",
  "tenantId": "tenant-uuid",
  "client": {
    "name": "Acme Corporation",
    "email": "admin@acme.com",
    "company": "Acme Corp",
    "tier": "enterprise"
  },
  "subscriptions": {
    "plugins": ["kyc-verification", "sanctions-screening"],
    "jurisdictions": ["US", "EU", "UK"],
    "features": ["decision-engine", "audit-trail"]
  },
  "limits": {
    "maxDecisionsPerDay": 100000,
    "maxNodes": 10,
    "maxApiCallsPerMinute": 1000
  },
  "validity": {
    "issuedAt": "2025-01-01T00:00:00Z",
    "expiresAt": "2026-01-01T00:00:00Z",
    "gracePeriodDays": 2
  },
  "status": "active"
}
```

## Plugin Integration

### Using the License Enforcer

```typescript
import {
  getLicenseEnforcer,
  withLicense,
  extractApiKey
} from '@arka/plugin-sdk';

// Option 1: Manual validation
const enforcer = getLicenseEnforcer({
  licenseAuthorityUrl: 'http://localhost:3003',
  gracePeriodDays: 2,
});

const validation = await enforcer.validatePluginAccess({
  pluginId: 'kyc-verification',
  jurisdiction: 'US',
  apiKey: request.headers['x-api-key'],
});

if (!validation.valid) {
  throw new Error(`License validation failed: ${validation.reason}`);
}

// Option 2: Middleware wrapper
const middleware = enforcer.createMiddleware();

const result = await middleware(
  { pluginId: 'kyc-verification', apiKey },
  async () => {
    // Your plugin logic here
    return performKycVerification(data);
  }
);

// Option 3: Wrap entire plugin
const licensedPlugin = withLicense(myPlugin, 'kyc-verification');
```

### Enforcement Modes

```typescript
// Strict mode (default) - blocks unlicensed access
enforcer.setMode('strict');

// Warn mode - logs warning but allows access
enforcer.setMode('warn');

// Disabled mode - no license checking
enforcer.setMode('disabled');
```

## CLI Reference

### Admin CLI (arka-admin)

```bash
# Configuration
arka-admin config show                    # Show current config
arka-admin config set-url <url>           # Set service URL
arka-admin config set-key <admin-key>     # Set admin API key
arka-admin config test                    # Test connection
arka-admin config bootstrap               # Create admin key

# Key Management
arka-admin key issue [options]            # Issue new key
arka-admin key revoke <key-id> [-r reason]  # Revoke key
arka-admin key update <key-id> [options]  # Update key
arka-admin key inspect <key-id>           # View key details
arka-admin key list [--tenant] [--status] # List keys
arka-admin key expiring [--days 30]       # List expiring keys
arka-admin key validate <api-key>         # Validate key

# Health
arka-admin health                         # Check service health
```

### Client CLI (arka-client)

```bash
# Configuration
arka-client configure --key <api-key>     # Set API key
arka-client configure --endpoint <url>    # Set service URL

# Validation
arka-client validate [api-key]            # Validate key
arka-client status                        # Show license status
arka-client info                          # Show detailed info
arka-client refresh                       # Refresh cache

# Maintenance
arka-client clear-cache                   # Clear local cache
arka-client health                        # Check service health
```

## Environment Variables

### License Authority Service

```bash
PORT=3003
HOST=0.0.0.0
DB_HOST=postgres
DB_PORT=5432
DB_NAME=arka_cloud
DB_USER=pact_admin
DB_PASSWORD=pact_local_dev
REDIS_HOST=redis
REDIS_PORT=6379
USE_KMS=false                    # Use AWS KMS for signing
KMS_KEY_ID=                      # KMS key ID (production)
JWT_ISSUER=arka-license-authority
JWT_AUDIENCE=arka-cloud
ADMIN_BOOTSTRAP_SECRET=          # Optional bootstrap secret
```

### Admin CLI

```bash
ARKA_LICENSE_AUTHORITY_URL=http://localhost:3003
ARKA_ADMIN_API_KEY=pact_admin_xxxxx
ARKA_OUTPUT_FORMAT=table
ARKA_TIMEOUT=30
```

### Client CLI

```bash
ARKA_LICENSE_AUTHORITY_URL=http://localhost:3003
ARKA_API_KEY=pact_live_xxxxx
ARKA_GRACE_PERIOD_DAYS=2
ARKA_CACHE_ENABLED=true
ARKA_CACHE_TTL=300
```

## Database Schema

The licensing system uses the following tables:

- `programmable_api_keys` - Stores API key metadata
- `admin_api_keys` - Stores admin key hashes
- `api_key_validation_cache` - Tracks validation history
- `api_key_usage` - Records API key usage

See `cloud/infrastructure/migrations/010_add_programmable_api_keys.sql` for full schema.

## Security Considerations

1. **API Key Storage**: Never store raw API keys in logs or databases. Only store hashes.
2. **Admin Keys**: Protect admin keys carefully - they have full management access.
3. **Grace Period**: The 2-day grace period allows for temporary outages but should be monitored.
4. **KMS Integration**: Use AWS KMS for production signing to protect key integrity.
5. **IP Whitelisting**: Consider enabling IP restrictions for sensitive deployments.

## Monitoring

### Key Metrics to Track

- License validation success/failure rate
- Offline validation usage (indicates connectivity issues)
- Grace period activations
- API key expiration rates
- Usage by plugin/jurisdiction/feature

### Alerts to Configure

- Keys expiring within 30 days
- High rate of validation failures
- Frequent grace period activations
- Admin key usage anomalies
