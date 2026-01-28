# ARKA Engine Webhooks & API Reference

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Integration

---

## Overview

ARKA Engine provides comprehensive webhook and API capabilities for real-time event notifications and system integration. This document covers webhook configuration, event types, API endpoints, and integration patterns.

---

## Table of Contents

1. [Webhooks](#webhooks)
2. [REST API Reference](#rest-api-reference)
3. [gRPC API Reference](#grpc-api-reference)
4. [GraphQL API](#graphql-api)
5. [Event Types](#event-types)
6. [Authentication](#authentication)
7. [Rate Limiting](#rate-limiting)
8. [Error Handling](#error-handling)

---

## Webhooks

### Webhook Configuration

#### Create Webhook Endpoint

```bash
curl -X POST https://api.arka-engine.io/api/v1/webhooks \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Compliance Alerts",
    "url": "https://your-app.com/webhooks/pact",
    "events": [
      "compliance.checked",
      "compliance.violated",
      "rule.executed",
      "audit.entry.created"
    ],
    "secret": "whsec_your-webhook-secret-key",
    "enabled": true,
    "config": {
      "retryPolicy": {
        "maxRetries": 5,
        "initialDelay": 1000,
        "maxDelay": 60000,
        "exponentialBase": 2
      },
      "timeout": 30000,
      "headers": {
        "X-Custom-Header": "custom-value"
      }
    }
  }'
```

Response:
```json
{
  "id": "wh_abc123xyz",
  "name": "Compliance Alerts",
  "url": "https://your-app.com/webhooks/pact",
  "events": [
    "compliance.checked",
    "compliance.violated",
    "rule.executed",
    "audit.entry.created"
  ],
  "enabled": true,
  "status": "active",
  "secret": "whsec_your-webhook-secret-key",
  "createdAt": "2024-12-01T10:00:00Z",
  "updatedAt": "2024-12-01T10:00:00Z"
}
```

### Webhook Payload Structure

```json
{
  "id": "evt_abc123",
  "type": "compliance.violated",
  "version": "1.0",
  "timestamp": "2024-12-01T14:30:00.000Z",
  "tenantId": "tenant_xyz",
  "data": {
    "complianceCheckId": "cc_789",
    "transactionId": "txn_456",
    "status": "NON_COMPLIANT",
    "score": 35,
    "violations": [
      {
        "ruleId": "rule_aml_001",
        "ruleName": "High-Value Transaction Screening",
        "severity": "critical",
        "message": "Transaction exceeds threshold without enhanced due diligence",
        "details": {
          "threshold": 10000,
          "amount": 50000,
          "currency": "USD"
        }
      }
    ],
    "executionTime": 145
  },
  "metadata": {
    "source": "api",
    "correlationId": "corr_xyz789",
    "userAgent": "PACT-SDK/1.0.0"
  }
}
```

### Webhook Signature Verification

ARKA Engine signs all webhook payloads using HMAC-SHA256. Verify signatures to ensure authenticity.

**Header:** `X-PACT-Signature`

```typescript
// TypeScript verification example
import crypto from 'crypto';

function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(`sha256=${expectedSignature}`)
  );
}

// Express.js middleware
app.post('/webhooks/pact', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-arka-signature'] as string;
  const payload = req.body.toString();

  if (!verifyWebhookSignature(payload, signature, WEBHOOK_SECRET)) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const event = JSON.parse(payload);
  // Process event...

  res.status(200).json({ received: true });
});
```

```python
# Python verification example
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode('utf-8'),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# Flask example
@app.route('/webhooks/pact', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-PACT-Signature')
    payload = request.get_data()

    if not verify_webhook_signature(payload, signature, WEBHOOK_SECRET):
        return jsonify({'error': 'Invalid signature'}), 401

    event = json.loads(payload)
    # Process event...

    return jsonify({'received': True}), 200
```

### Webhook Management Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/webhooks` | GET | List all webhooks |
| `/api/v1/webhooks` | POST | Create webhook |
| `/api/v1/webhooks/{id}` | GET | Get webhook |
| `/api/v1/webhooks/{id}` | PUT | Update webhook |
| `/api/v1/webhooks/{id}` | DELETE | Delete webhook |
| `/api/v1/webhooks/{id}/test` | POST | Send test event |
| `/api/v1/webhooks/{id}/deliveries` | GET | List delivery attempts |
| `/api/v1/webhooks/{id}/deliveries/{deliveryId}/retry` | POST | Retry delivery |

### Webhook Retry Policy

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 1 second |
| 3 | 5 seconds |
| 4 | 30 seconds |
| 5 | 2 minutes |
| 6 | 10 minutes |
| 7 | 30 minutes |
| 8 | 1 hour |

After 8 failed attempts, the webhook is marked as failing and an alert is generated.

---

## REST API Reference

### Base URL

```
Production: https://api.arka-engine.io/api/v1
Staging:    https://staging.arka-engine.io/api/v1
```

### API Versioning

ARKA Engine uses URL-based versioning:

```
/api/v1/...  # Current stable version
/api/v2/...  # Next version (when available)
```

### Compliance Endpoints

#### Check Compliance

```http
POST /api/v1/compliance/check
Content-Type: application/json
Authorization: Bearer {token}

{
  "transaction": {
    "id": "txn_123",
    "type": "wire_transfer",
    "amount": 50000.00,
    "currency": "USD",
    "timestamp": "2024-12-01T14:30:00Z",
    "sender": {
      "id": "party_001",
      "name": "John Doe",
      "type": "individual",
      "country": "US",
      "identifiers": {
        "taxId": "XXX-XX-1234"
      }
    },
    "recipient": {
      "id": "party_002",
      "name": "Acme Corporation",
      "type": "business",
      "country": "GB",
      "identifiers": {
        "lei": "5493001KJTIIGC8Y1R12"
      }
    },
    "metadata": {
      "purpose": "invoice_payment",
      "reference": "INV-2024-001"
    }
  },
  "rulesets": ["aml-kyc", "sanctions", "fatf"],
  "options": {
    "includeExplanation": true,
    "includeAuditTrail": true,
    "timeout": 5000,
    "failFast": false
  }
}
```

Response:
```json
{
  "id": "cc_abc123",
  "status": "NON_COMPLIANT",
  "score": 45,
  "violations": [
    {
      "id": "viol_001",
      "ruleId": "rule_aml_htv_001",
      "ruleName": "High-Value Transaction Monitoring",
      "ruleVersion": "2.1.0",
      "severity": "high",
      "category": "aml",
      "message": "Transaction amount exceeds $10,000 threshold - requires CTR filing",
      "details": {
        "threshold": 10000,
        "amount": 50000,
        "requiredAction": "ctr_filing"
      },
      "remediation": "File Currency Transaction Report (CTR) within 15 days"
    }
  ],
  "warnings": [
    {
      "id": "warn_001",
      "ruleId": "rule_kyc_refresh",
      "message": "Customer KYC data is older than 12 months",
      "severity": "medium"
    }
  ],
  "rulesExecuted": 47,
  "rulesPassed": 45,
  "rulesFailed": 2,
  "executionTime": 234,
  "explanation": {
    "summary": "Transaction flagged for high-value monitoring requirements",
    "factors": [
      {
        "factor": "Transaction Amount",
        "impact": "high",
        "description": "Amount of $50,000 exceeds CTR threshold of $10,000"
      }
    ],
    "recommendations": [
      "File CTR within 15 days",
      "Update customer KYC documentation"
    ]
  },
  "auditTrail": {
    "id": "audit_xyz789",
    "timestamp": "2024-12-01T14:30:00.234Z",
    "hash": "sha256:abc123...",
    "blockchainAnchor": {
      "network": "polygon",
      "transactionHash": "0x123...",
      "blockNumber": 12345678
    }
  }
}
```

#### Batch Compliance Check

```http
POST /api/v1/compliance/batch
Content-Type: application/json
Authorization: Bearer {token}

{
  "transactions": [
    { "id": "txn_001", ... },
    { "id": "txn_002", ... },
    { "id": "txn_003", ... }
  ],
  "rulesets": ["aml-kyc"],
  "options": {
    "parallelism": 10,
    "continueOnError": true
  }
}
```

### Rules Endpoints

#### List Rules

```http
GET /api/v1/rules?status=active&category=aml&limit=50&offset=0
Authorization: Bearer {token}
```

Response:
```json
{
  "data": [
    {
      "id": "rule_aml_001",
      "name": "High-Value Transaction Monitoring",
      "version": "2.1.0",
      "status": "active",
      "category": "aml",
      "subcategory": "transaction_monitoring",
      "jurisdiction": ["US", "EU"],
      "severity": "high",
      "description": "Monitors transactions exceeding regulatory thresholds",
      "createdAt": "2024-01-15T09:00:00Z",
      "updatedAt": "2024-11-20T14:30:00Z"
    }
  ],
  "pagination": {
    "total": 156,
    "limit": 50,
    "offset": 0,
    "hasMore": true
  }
}
```

#### Create Rule

```http
POST /api/v1/rules
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "Custom Screening Rule",
  "category": "sanctions",
  "jurisdiction": ["US"],
  "severity": "critical",
  "definition": {
    "type": "condition",
    "condition": {
      "field": "recipient.country",
      "operator": "in",
      "value": ["CU", "IR", "KP", "SY"]
    },
    "onMatch": {
      "action": "block",
      "message": "Transaction blocked - sanctioned country"
    }
  },
  "metadata": {
    "regulatoryReference": "OFAC SDN List",
    "effectiveDate": "2024-01-01"
  }
}
```

#### Execute Rule

```http
POST /api/v1/rules/{ruleId}/execute
Content-Type: application/json
Authorization: Bearer {token}

{
  "input": {
    "transaction": { ... },
    "context": {
      "customerRiskScore": 75,
      "previousTransactions": 12
    }
  },
  "options": {
    "dryRun": false,
    "includeTrace": true
  }
}
```

### Audit Endpoints

#### Query Audit Logs

```http
GET /api/v1/audit/logs?startTime=2024-12-01T00:00:00Z&endTime=2024-12-01T23:59:59Z&action=compliance.check&limit=100
Authorization: Bearer {token}
```

#### Get Audit Entry

```http
GET /api/v1/audit/logs/{entryId}
Authorization: Bearer {token}
```

#### Verify Audit Entry Integrity

```http
POST /api/v1/audit/logs/{entryId}/verify
Authorization: Bearer {token}
```

Response:
```json
{
  "entryId": "audit_xyz789",
  "verified": true,
  "hashChainValid": true,
  "blockchainAnchor": {
    "verified": true,
    "network": "polygon",
    "transactionHash": "0x123...",
    "blockNumber": 12345678,
    "blockTimestamp": "2024-12-01T14:30:15Z"
  },
  "signatureValid": true,
  "verifiedAt": "2024-12-01T15:00:00Z"
}
```

### Users & Tenants Endpoints

#### List Users

```http
GET /api/v1/users?role=compliance_officer&status=active
Authorization: Bearer {token}
```

#### Get User

```http
GET /api/v1/users/{userId}
Authorization: Bearer {token}
```

#### Get Current Tenant

```http
GET /api/v1/tenants/current
Authorization: Bearer {token}
```

### Reports Endpoints

#### Generate Compliance Report

```http
POST /api/v1/reports/compliance
Content-Type: application/json
Authorization: Bearer {token}

{
  "type": "monthly_summary",
  "period": {
    "start": "2024-11-01T00:00:00Z",
    "end": "2024-11-30T23:59:59Z"
  },
  "filters": {
    "rulesets": ["aml-kyc", "sanctions"],
    "status": ["NON_COMPLIANT", "PENDING_REVIEW"]
  },
  "format": "pdf",
  "delivery": {
    "method": "email",
    "recipients": ["compliance@example.com"]
  }
}
```

#### Get Report Status

```http
GET /api/v1/reports/{reportId}
Authorization: Bearer {token}
```

#### Download Report

```http
GET /api/v1/reports/{reportId}/download
Authorization: Bearer {token}
Accept: application/pdf
```

---

## gRPC API Reference

### Service Definition

```protobuf
syntax = "proto3";

package arka.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

// Compliance Service
service ComplianceService {
  // Check compliance for a single transaction
  rpc Check(CheckRequest) returns (CheckResponse);

  // Check compliance for multiple transactions
  rpc BatchCheck(BatchCheckRequest) returns (stream CheckResponse);

  // Stream compliance checks
  rpc StreamCheck(stream CheckRequest) returns (stream CheckResponse);
}

// Rules Service
service RulesService {
  // List rules
  rpc List(ListRulesRequest) returns (ListRulesResponse);

  // Get rule by ID
  rpc Get(GetRuleRequest) returns (Rule);

  // Create rule
  rpc Create(CreateRuleRequest) returns (Rule);

  // Update rule
  rpc Update(UpdateRuleRequest) returns (Rule);

  // Delete rule
  rpc Delete(DeleteRuleRequest) returns (DeleteRuleResponse);

  // Execute rule
  rpc Execute(ExecuteRuleRequest) returns (ExecuteRuleResponse);
}

// Audit Service
service AuditService {
  // Query audit logs
  rpc Query(QueryAuditRequest) returns (QueryAuditResponse);

  // Get audit entry
  rpc Get(GetAuditRequest) returns (AuditEntry);

  // Verify audit entry
  rpc Verify(VerifyAuditRequest) returns (VerifyAuditResponse);

  // Stream audit events
  rpc StreamEvents(StreamEventsRequest) returns (stream AuditEvent);
}

// Events Service
service EventsService {
  // Subscribe to events
  rpc Subscribe(SubscribeRequest) returns (stream Event);

  // Publish event
  rpc Publish(PublishRequest) returns (PublishResponse);
}

// Messages
message CheckRequest {
  Transaction transaction = 1;
  repeated string rulesets = 2;
  CheckOptions options = 3;
}

message CheckResponse {
  string id = 1;
  ComplianceStatus status = 2;
  float score = 3;
  repeated Violation violations = 4;
  repeated Warning warnings = 5;
  int32 rules_executed = 6;
  int64 execution_time_ms = 7;
  Explanation explanation = 8;
  AuditEntry audit_trail = 9;
}

message Transaction {
  string id = 1;
  string type = 2;
  double amount = 3;
  string currency = 4;
  google.protobuf.Timestamp timestamp = 5;
  Party sender = 6;
  Party recipient = 7;
  google.protobuf.Struct metadata = 8;
}

message Party {
  string id = 1;
  string name = 2;
  string type = 3;
  string country = 4;
  map<string, string> identifiers = 5;
}

enum ComplianceStatus {
  COMPLIANCE_STATUS_UNSPECIFIED = 0;
  COMPLIANT = 1;
  NON_COMPLIANT = 2;
  PENDING_REVIEW = 3;
  ERROR = 4;
}

message Violation {
  string id = 1;
  string rule_id = 2;
  string rule_name = 3;
  Severity severity = 4;
  string category = 5;
  string message = 6;
  google.protobuf.Struct details = 7;
  string remediation = 8;
}

enum Severity {
  SEVERITY_UNSPECIFIED = 0;
  LOW = 1;
  MEDIUM = 2;
  HIGH = 3;
  CRITICAL = 4;
}
```

### gRPC Client Examples

```go
// Go gRPC client
package main

import (
    "context"
    "log"

    pb "github.com/arka-engine/arka-sdk-go/proto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

func main() {
    // Create connection with TLS
    creds, err := credentials.NewClientTLSFromFile("ca.crt", "")
    if err != nil {
        log.Fatalf("Failed to load TLS credentials: %v", err)
    }

    conn, err := grpc.Dial(
        "grpc.arka-engine.io:443",
        grpc.WithTransportCredentials(creds),
        grpc.WithPerRPCCredentials(&tokenAuth{token: "your-api-token"}),
    )
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewComplianceServiceClient(conn)

    // Check compliance
    resp, err := client.Check(context.Background(), &pb.CheckRequest{
        Transaction: &pb.Transaction{
            Id:       "txn-123",
            Type:     "wire_transfer",
            Amount:   50000.00,
            Currency: "USD",
        },
        Rulesets: []string{"aml-kyc", "sanctions"},
    })
    if err != nil {
        log.Fatalf("Check failed: %v", err)
    }

    log.Printf("Status: %v, Score: %v", resp.Status, resp.Score)
}
```

---

## GraphQL API

### Endpoint

```
https://api.arka-engine.io/graphql
```

### Schema

```graphql
type Query {
  # Compliance
  complianceCheck(id: ID!): ComplianceResult
  complianceHistory(
    transactionId: ID
    startTime: DateTime
    endTime: DateTime
    status: ComplianceStatus
    limit: Int = 50
    offset: Int = 0
  ): ComplianceResultConnection!

  # Rules
  rule(id: ID!): Rule
  rules(
    status: RuleStatus
    category: String
    jurisdiction: [String!]
    limit: Int = 50
    offset: Int = 0
  ): RuleConnection!

  # Audit
  auditEntry(id: ID!): AuditEntry
  auditLogs(
    startTime: DateTime!
    endTime: DateTime!
    action: String
    entityType: String
    limit: Int = 100
  ): AuditEntryConnection!

  # Users
  me: User
  user(id: ID!): User
  users(role: String, status: UserStatus, limit: Int = 50): UserConnection!
}

type Mutation {
  # Compliance
  checkCompliance(input: ComplianceCheckInput!): ComplianceResult!
  batchCheckCompliance(input: BatchComplianceCheckInput!): BatchComplianceResult!

  # Rules
  createRule(input: CreateRuleInput!): Rule!
  updateRule(id: ID!, input: UpdateRuleInput!): Rule!
  deleteRule(id: ID!): DeleteRuleResult!
  executeRule(id: ID!, input: ExecuteRuleInput!): RuleExecutionResult!

  # Webhooks
  createWebhook(input: CreateWebhookInput!): Webhook!
  updateWebhook(id: ID!, input: UpdateWebhookInput!): Webhook!
  deleteWebhook(id: ID!): DeleteWebhookResult!
  testWebhook(id: ID!): WebhookTestResult!
}

type Subscription {
  # Real-time compliance events
  complianceEvents(
    transactionTypes: [String!]
    severities: [Severity!]
  ): ComplianceEvent!

  # Rule execution events
  ruleEvents(ruleIds: [ID!]): RuleEvent!

  # Audit events
  auditEvents(actions: [String!]): AuditEvent!
}

# Types
type ComplianceResult {
  id: ID!
  status: ComplianceStatus!
  score: Float!
  violations: [Violation!]!
  warnings: [Warning!]!
  rulesExecuted: Int!
  executionTime: Int!
  explanation: Explanation
  auditTrail: AuditEntry
  createdAt: DateTime!
}

type Rule {
  id: ID!
  name: String!
  version: String!
  status: RuleStatus!
  category: String!
  jurisdiction: [String!]!
  severity: Severity!
  description: String
  definition: JSON!
  metadata: JSON
  versions: [RuleVersion!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum ComplianceStatus {
  COMPLIANT
  NON_COMPLIANT
  PENDING_REVIEW
  ERROR
}

enum Severity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

input ComplianceCheckInput {
  transaction: TransactionInput!
  rulesets: [String!]!
  options: ComplianceOptionsInput
}

input TransactionInput {
  id: String!
  type: String!
  amount: Float!
  currency: String!
  timestamp: DateTime
  sender: PartyInput!
  recipient: PartyInput!
  metadata: JSON
}
```

### GraphQL Queries

```graphql
# Check compliance
mutation CheckCompliance($input: ComplianceCheckInput!) {
  checkCompliance(input: $input) {
    id
    status
    score
    violations {
      id
      ruleId
      ruleName
      severity
      message
      remediation
    }
    explanation {
      summary
      factors {
        factor
        impact
        description
      }
    }
    executionTime
  }
}

# Query rules with filters
query GetRules($category: String, $status: RuleStatus) {
  rules(category: $category, status: $status, limit: 20) {
    edges {
      node {
        id
        name
        version
        category
        severity
        jurisdiction
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Subscribe to compliance events
subscription OnComplianceEvent {
  complianceEvents(severities: [HIGH, CRITICAL]) {
    id
    type
    timestamp
    transaction {
      id
      type
      amount
    }
    violations {
      ruleId
      severity
      message
    }
  }
}
```

---

## Event Types

### Compliance Events

| Event Type | Description | Payload |
|------------|-------------|---------|
| `compliance.checked` | Compliance check completed | ComplianceResult |
| `compliance.violated` | Violation detected | ComplianceResult with violations |
| `compliance.approved` | Transaction approved | ComplianceResult |
| `compliance.pending_review` | Requires manual review | ComplianceResult |

### Rule Events

| Event Type | Description | Payload |
|------------|-------------|---------|
| `rule.created` | New rule created | Rule |
| `rule.updated` | Rule updated | Rule |
| `rule.deleted` | Rule deleted | { ruleId } |
| `rule.executed` | Rule executed | RuleExecutionResult |
| `rule.activated` | Rule activated | Rule |
| `rule.deactivated` | Rule deactivated | Rule |

### Audit Events

| Event Type | Description | Payload |
|------------|-------------|---------|
| `audit.entry.created` | Audit entry created | AuditEntry |
| `audit.anchored` | Entry anchored to blockchain | AnchorResult |
| `audit.verified` | Entry verification completed | VerificationResult |

### System Events

| Event Type | Description | Payload |
|------------|-------------|---------|
| `system.health.changed` | Health status changed | HealthStatus |
| `system.rate_limit.exceeded` | Rate limit exceeded | RateLimitInfo |
| `webhook.delivery.failed` | Webhook delivery failed | DeliveryInfo |

---

## Authentication

### API Key Authentication

```http
Authorization: Bearer pact_key_xxxxxxxxxxxxxxxxxxxx
```

### JWT Authentication

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

### OAuth 2.0

```bash
# Get access token
curl -X POST https://api.arka-engine.io/oauth/token \
  -d "grant_type=client_credentials" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "scope=pact:read pact:write"
```

---

## Rate Limiting

### Limits

| Plan | Requests/Second | Requests/Day | Burst |
|------|-----------------|--------------|-------|
| Starter | 10 | 10,000 | 20 |
| Professional | 100 | 100,000 | 200 |
| Enterprise | 1,000 | Unlimited | 2,000 |

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1701388800
X-RateLimit-RetryAfter: 5
```

### Handling Rate Limits

```typescript
async function makeRequest(url: string, options: RequestOptions) {
  const response = await fetch(url, options);

  if (response.status === 429) {
    const retryAfter = parseInt(response.headers.get('X-RateLimit-RetryAfter') || '5');
    await delay(retryAfter * 1000);
    return makeRequest(url, options);
  }

  return response;
}
```

---

## Error Handling

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid transaction data",
    "details": [
      {
        "field": "transaction.amount",
        "message": "Amount must be greater than 0"
      }
    ],
    "requestId": "req_abc123",
    "documentation": "https://docs.arka-engine.io/errors/validation-error"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Invalid or missing authentication |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `RATE_LIMITED` | 429 | Rate limit exceeded |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |
| `RULE_EXECUTION_ERROR` | 422 | Rule execution failed |
| `TIMEOUT` | 408 | Request timeout |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Engineering Team | Initial release |

**Review Schedule:** Monthly
**Next Review:** January 2025
