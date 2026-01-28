# REST API Reference

Complete reference for the ARKA Engine REST API.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Common Patterns](#common-patterns)
- [Endpoints](#endpoints)
  - [Rules](#rules)
  - [Evaluations](#evaluations)
  - [Audit](#audit)
  - [Governance](#governance)
  - [Agents](#agents)
  - [System](#system)

---

## Overview

### Base URL

```
Production: https://api.arka-engine.example.com/v1
Development: http://localhost:8080/api/v1
```

### API Versioning

The API version is included in the URL path (`/v1/`). Breaking changes result in a new version.

### Request Format

```http
POST /api/v1/evaluate HTTP/1.1
Host: api.arka-engine.example.com
Authorization: Bearer <token>
Content-Type: application/json
X-Request-ID: uuid-here

{
  "context": {...},
  "input": {...}
}
```

### Response Format

```json
{
  "data": { },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z",
    "duration": 45
  }
}
```

### Error Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "input.amount",
        "message": "Must be a positive number"
      }
    ]
  },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

---

## Authentication

### API Key Authentication

```http
Authorization: Bearer <api_key>
```

### JWT Authentication

```http
Authorization: Bearer <jwt_token>
```

### Obtaining Tokens

```http
POST /api/v1/auth/token
Content-Type: application/json

{
  "client_id": "your-client-id",
  "client_secret": "your-client-secret",
  "grant_type": "client_credentials"
}
```

Response:
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

## Common Patterns

### Pagination

```http
GET /api/v1/rules?page=1&per_page=20
```

Response includes pagination metadata:
```json
{
  "data": [...],
  "meta": {
    "pagination": {
      "page": 1,
      "perPage": 20,
      "total": 150,
      "totalPages": 8
    }
  }
}
```

### Filtering

```http
GET /api/v1/rules?jurisdiction=US&domain=AML&status=active
```

### Sorting

```http
GET /api/v1/rules?sort=-created_at,name
```
- Prefix with `-` for descending order
- Multiple fields separated by comma

### Field Selection

```http
GET /api/v1/rules?fields=id,name,version,status
```

### Including Related Resources

```http
GET /api/v1/rules/rule_123?include=versions,dependencies
```

---

## Endpoints

### Rules

#### List Rules

```http
GET /api/v1/rules
```

Query Parameters:
| Parameter | Type | Description |
|-----------|------|-------------|
| `jurisdiction` | string | Filter by jurisdiction |
| `domain` | string | Filter by domain |
| `status` | string | Filter by status (active, inactive, deprecated) |
| `search` | string | Search in name and description |
| `page` | integer | Page number (default: 1) |
| `per_page` | integer | Items per page (default: 20, max: 100) |

Response:
```json
{
  "data": [
    {
      "id": "rule_abc123",
      "name": "ctr-threshold-flag",
      "description": "Flag transactions meeting CTR threshold",
      "jurisdiction": "US",
      "domain": "AML",
      "version": "1.2.0",
      "status": "active",
      "createdAt": "2024-01-10T08:00:00Z",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "perPage": 20,
      "total": 45
    }
  }
}
```

#### Get Rule

```http
GET /api/v1/rules/{rule_id}
```

Response:
```json
{
  "data": {
    "id": "rule_abc123",
    "name": "ctr-threshold-flag",
    "description": "Flag transactions meeting CTR threshold",
    "jurisdiction": "US",
    "domain": "AML",
    "version": "1.2.0",
    "status": "active",
    "ruleDsl": {
      "conditions": [...],
      "actions": [...]
    },
    "metadata": {
      "effectiveDate": "2024-01-01",
      "tags": ["ctr", "bsa"],
      "author": "compliance-team"
    },
    "bytecodeHash": "sha256:abc123...",
    "createdAt": "2024-01-10T08:00:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

#### Create Rule

```http
POST /api/v1/rules
Content-Type: application/json

{
  "name": "new-rule",
  "description": "Description of the rule",
  "jurisdiction": "US",
  "domain": "AML",
  "ruleDsl": {
    "conditions": [
      {
        "field": "transaction.amount",
        "operator": "greater_than",
        "value": 10000
      }
    ],
    "actions": [
      {
        "type": "flag",
        "severity": "high",
        "message": "Amount exceeds threshold"
      }
    ]
  },
  "metadata": {
    "effectiveDate": "2024-02-01",
    "tags": ["new", "test"]
  }
}
```

Response: `201 Created`
```json
{
  "data": {
    "id": "rule_new123",
    "name": "new-rule",
    "version": "1.0.0",
    "status": "active",
    "bytecodeHash": "sha256:def456..."
  }
}
```

#### Update Rule

```http
PUT /api/v1/rules/{rule_id}
Content-Type: application/json

{
  "description": "Updated description",
  "ruleDsl": {
    "conditions": [...],
    "actions": [...]
  }
}
```

Response: `200 OK`
```json
{
  "data": {
    "id": "rule_abc123",
    "version": "1.3.0",
    "previousVersion": "1.2.0"
  }
}
```

#### Delete Rule

```http
DELETE /api/v1/rules/{rule_id}
```

Response: `204 No Content`

#### Get Rule Versions

```http
GET /api/v1/rules/{rule_id}/versions
```

Response:
```json
{
  "data": [
    {
      "version": "1.2.0",
      "createdAt": "2024-01-15T10:30:00Z",
      "bytecodeHash": "sha256:abc123...",
      "changesSummary": "Increased threshold from 5000 to 10000"
    },
    {
      "version": "1.1.0",
      "createdAt": "2024-01-10T08:00:00Z",
      "bytecodeHash": "sha256:def456...",
      "changesSummary": "Added new condition"
    }
  ]
}
```

#### Validate Rule

```http
POST /api/v1/rules/validate
Content-Type: application/json

{
  "ruleDsl": {
    "conditions": [...],
    "actions": [...]
  }
}
```

Response:
```json
{
  "data": {
    "valid": true,
    "warnings": [
      {
        "type": "COMPLEXITY",
        "message": "Rule has high complexity score"
      }
    ]
  }
}
```

---

### Evaluations

#### Evaluate Compliance

```http
POST /api/v1/evaluate
Content-Type: application/json

{
  "context": {
    "jurisdiction": "US",
    "domain": "AML",
    "timestamp": "2024-01-15T10:30:00Z"
  },
  "input": {
    "transaction": {
      "id": "txn_001",
      "amount": 15000,
      "currency": "USD",
      "sender": "customer_123",
      "recipient": "merchant_456",
      "type": "wire"
    }
  },
  "options": {
    "includeTrace": false,
    "timeout": 5000
  }
}
```

Response:
```json
{
  "data": {
    "evaluationId": "eval_xyz789",
    "timestamp": "2024-01-15T10:30:01Z",
    "result": {
      "decision": "non_compliant",
      "flags": [
        {
          "ruleId": "rule_abc123",
          "ruleVersion": "1.2.0",
          "conditionId": "amount_threshold",
          "severity": "high",
          "category": "CTR_REQUIRED",
          "message": "Transaction amount $15,000 exceeds CTR threshold",
          "field": "transaction.amount",
          "value": 15000
        }
      ],
      "annotations": {
        "ctrRequired": true,
        "reportingDeadline": "2024-01-30"
      }
    },
    "metadata": {
      "rulesEvaluated": ["rule_abc123", "rule_def456"],
      "ruleVersions": {
        "rule_abc123": "1.2.0",
        "rule_def456": "2.0.0"
      },
      "evaluationDurationMs": 12,
      "cached": false
    },
    "trustProof": {
      "proofId": "proof_qrs123",
      "inputHash": "sha256:abc...",
      "rulesHash": "sha256:def...",
      "outputHash": "sha256:ghi...",
      "signature": "sig_xyz..."
    }
  }
}
```

#### Batch Evaluation

```http
POST /api/v1/evaluate/batch
Content-Type: application/json

{
  "context": {
    "jurisdiction": "US",
    "domain": "AML"
  },
  "inputs": [
    {
      "id": "batch_item_1",
      "transaction": {...}
    },
    {
      "id": "batch_item_2",
      "transaction": {...}
    }
  ],
  "options": {
    "parallelism": 10,
    "stopOnFirstFailure": false
  }
}
```

Response:
```json
{
  "data": {
    "batchId": "batch_abc123",
    "results": [
      {
        "id": "batch_item_1",
        "evaluationId": "eval_001",
        "result": {...}
      },
      {
        "id": "batch_item_2",
        "evaluationId": "eval_002",
        "result": {...}
      }
    ],
    "summary": {
      "total": 2,
      "compliant": 1,
      "nonCompliant": 1,
      "errors": 0
    }
  }
}
```

#### Get Evaluation

```http
GET /api/v1/evaluations/{evaluation_id}
```

Response:
```json
{
  "data": {
    "evaluationId": "eval_xyz789",
    "timestamp": "2024-01-15T10:30:01Z",
    "context": {...},
    "input": {...},
    "result": {...},
    "trustProof": {...}
  }
}
```

#### Replay Evaluation

```http
POST /api/v1/evaluate/replay
Content-Type: application/json

{
  "evaluationId": "eval_xyz789",
  "options": {
    "ruleVersion": "1.1.0",
    "timestamp": "2024-01-01T00:00:00Z"
  }
}
```

---

### Audit

#### Query Audit Events

```http
GET /api/v1/audit/events
```

Query Parameters:
| Parameter | Type | Description |
|-----------|------|-------------|
| `startTime` | datetime | Start of time range |
| `endTime` | datetime | End of time range |
| `eventType` | string | Filter by event type |
| `entityId` | string | Filter by entity ID |
| `userId` | string | Filter by user ID |

Response:
```json
{
  "data": [
    {
      "eventId": "evt_abc123",
      "timestamp": "2024-01-15T10:30:00Z",
      "eventType": "RULE_CREATED",
      "entityType": "rule",
      "entityId": "rule_abc123",
      "userId": "user_xyz",
      "changes": {
        "before": null,
        "after": {...}
      },
      "metadata": {
        "ipAddress": "192.168.1.1",
        "userAgent": "PACT-CLI/1.0.0"
      }
    }
  ]
}
```

#### Get Trust Proof

```http
GET /api/v1/audit/proofs/{proof_id}
```

Response:
```json
{
  "data": {
    "proofId": "proof_qrs123",
    "evaluationId": "eval_xyz789",
    "timestamp": "2024-01-15T10:30:01Z",
    "content": {
      "inputHash": "sha256:abc...",
      "rulesRoot": "sha256:def...",
      "ruleProofs": [...],
      "traceHash": "sha256:ghi...",
      "outputHash": "sha256:jkl..."
    },
    "attestation": {
      "signature": "sig_xyz...",
      "certificateChain": [...]
    },
    "anchors": [
      {
        "network": "ethereum",
        "transactionHash": "0xabc...",
        "blockNumber": 18234567,
        "timestamp": "2024-01-15T11:00:00Z"
      }
    ]
  }
}
```

#### Verify Proof

```http
POST /api/v1/audit/proofs/{proof_id}/verify
Content-Type: application/json

{
  "includeReplay": true,
  "originalInput": {...}
}
```

Response:
```json
{
  "data": {
    "valid": true,
    "verificationSteps": [
      {"step": "signature", "status": "passed"},
      {"step": "ruleCommitment", "status": "passed"},
      {"step": "blockchainAnchor", "status": "passed"},
      {"step": "replay", "status": "passed"}
    ]
  }
}
```

#### Export Audit Data

```http
POST /api/v1/audit/export
Content-Type: application/json

{
  "startTime": "2024-01-01T00:00:00Z",
  "endTime": "2024-01-31T23:59:59Z",
  "format": "json",
  "filters": {
    "eventTypes": ["EVALUATION_COMPLETED"]
  }
}
```

Response: `202 Accepted`
```json
{
  "data": {
    "exportId": "export_abc123",
    "status": "processing",
    "estimatedCompletion": "2024-01-15T11:00:00Z"
  }
}
```

---

### Governance

#### Create Proposal

```http
POST /api/v1/governance/proposals
Content-Type: application/json

{
  "type": "MODIFY",
  "targetRuleId": "rule_abc123",
  "changes": {
    "ruleDsl": {
      "conditions": [...]
    }
  },
  "rationale": "Updating threshold based on regulatory guidance",
  "evidence": [
    {
      "type": "REGULATORY_GUIDANCE",
      "reference": "FinCEN-2024-001",
      "url": "https://..."
    }
  ]
}
```

Response: `201 Created`
```json
{
  "data": {
    "proposalId": "prop_xyz789",
    "status": "draft",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

#### Get Proposal

```http
GET /api/v1/governance/proposals/{proposal_id}
```

Response:
```json
{
  "data": {
    "proposalId": "prop_xyz789",
    "type": "MODIFY",
    "status": "pending_review",
    "targetRuleId": "rule_abc123",
    "changes": {...},
    "rationale": "...",
    "proposer": {
      "userId": "user_123",
      "name": "John Doe"
    },
    "workflow": {
      "currentStage": "technical_review",
      "stages": [
        {
          "name": "technical_review",
          "status": "in_progress",
          "reviewers": ["user_456"],
          "votes": []
        },
        {
          "name": "compliance_review",
          "status": "pending"
        },
        {
          "name": "business_approval",
          "status": "pending"
        }
      ]
    },
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

#### Vote on Proposal

```http
POST /api/v1/governance/proposals/{proposal_id}/vote
Content-Type: application/json

{
  "decision": "approve",
  "comments": "Technical review passed, all tests successful",
  "conditions": []
}
```

Response:
```json
{
  "data": {
    "voteId": "vote_abc123",
    "recorded": true,
    "proposalStatus": "pending_review",
    "nextStage": "compliance_review"
  }
}
```

#### Run Simulation

```http
POST /api/v1/governance/proposals/{proposal_id}/simulate
Content-Type: application/json

{
  "simulationType": "historical_replay",
  "options": {
    "dateRange": {
      "start": "2024-01-01",
      "end": "2024-01-14"
    },
    "sampleSize": 10000
  }
}
```

Response: `202 Accepted`
```json
{
  "data": {
    "simulationId": "sim_xyz123",
    "status": "running",
    "estimatedCompletion": "2024-01-15T11:00:00Z"
  }
}
```

#### Get Simulation Results

```http
GET /api/v1/governance/simulations/{simulation_id}
```

Response:
```json
{
  "data": {
    "simulationId": "sim_xyz123",
    "status": "completed",
    "results": {
      "totalCases": 10000,
      "outcomeChanges": 234,
      "changeRate": 0.0234,
      "breakdown": {
        "newFlags": 150,
        "removedFlags": 84,
        "changedSeverity": 0
      },
      "performance": {
        "baselineAvgMs": 12,
        "proposedAvgMs": 14,
        "delta": 2
      }
    },
    "completedAt": "2024-01-15T10:45:00Z"
  }
}
```

---

### Agents

#### Register Agent

```http
POST /api/v1/agents/register
Content-Type: application/json

{
  "name": "edge-agent-us-west-1",
  "type": "edge",
  "capabilities": {
    "domains": ["AML", "KYC"],
    "jurisdictions": ["US"],
    "maxThroughput": 10000
  },
  "publicKey": "-----BEGIN PUBLIC KEY-----\n...",
  "attestation": {
    "type": "sgx",
    "report": "..."
  }
}
```

Response: `201 Created`
```json
{
  "data": {
    "agentId": "agent_xyz789",
    "status": "registered",
    "credentials": {
      "certificate": "-----BEGIN CERTIFICATE-----\n...",
      "syncToken": "sync_abc..."
    }
  }
}
```

#### List Agents

```http
GET /api/v1/agents
```

Response:
```json
{
  "data": [
    {
      "agentId": "agent_xyz789",
      "name": "edge-agent-us-west-1",
      "status": "healthy",
      "lastHeartbeat": "2024-01-15T10:29:55Z",
      "ruleVersion": "1.2.0",
      "metrics": {
        "evaluationsTotal": 1500000,
        "evaluationsPerSecond": 250,
        "errorRate": 0.0001
      }
    }
  ]
}
```

#### Sync Agent

```http
POST /api/v1/agents/{agent_id}/sync
Content-Type: application/json

{
  "currentVersion": "1.1.0"
}
```

Response:
```json
{
  "data": {
    "updateAvailable": true,
    "targetVersion": "1.2.0",
    "deltaPackageUrl": "https://...",
    "deltaPackageHash": "sha256:abc..."
  }
}
```

#### Get Agent Proofs

```http
GET /api/v1/agents/{agent_id}/proofs
```

Query Parameters:
| Parameter | Type | Description |
|-----------|------|-------------|
| `startTime` | datetime | Start time |
| `endTime` | datetime | End time |
| `status` | string | Proof status |

---

### System

#### Health Check

```http
GET /api/v1/health
```

Response:
```json
{
  "data": {
    "status": "healthy",
    "version": "1.0.0",
    "components": {
      "database": "healthy",
      "cache": "healthy",
      "rtvm": "healthy",
      "queue": "healthy"
    }
  }
}
```

#### Detailed Health

```http
GET /api/v1/health/detailed
```

Response:
```json
{
  "data": {
    "status": "healthy",
    "components": {
      "database": {
        "status": "healthy",
        "latencyMs": 5,
        "connections": 20
      },
      "cache": {
        "status": "healthy",
        "hitRate": 0.95,
        "memoryUsed": "512MB"
      },
      "rtvm": {
        "status": "healthy",
        "workers": 8,
        "queueDepth": 12
      }
    }
  }
}
```

#### System Metrics

```http
GET /api/v1/metrics
```

Response: Prometheus format metrics

```
# HELP pact_evaluations_total Total evaluations
# TYPE pact_evaluations_total counter
pact_evaluations_total{jurisdiction="US",domain="AML"} 1500000

# HELP pact_evaluation_duration_seconds Evaluation duration
# TYPE pact_evaluation_duration_seconds histogram
pact_evaluation_duration_seconds_bucket{le="0.01"} 1400000
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid request data |
| `AUTHENTICATION_ERROR` | 401 | Invalid or missing credentials |
| `AUTHORIZATION_ERROR` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Resource conflict |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| `/evaluate` | 1000 req/min |
| `/evaluate/batch` | 100 req/min |
| `/rules` (read) | 500 req/min |
| `/rules` (write) | 100 req/min |
| `/audit/*` | 200 req/min |

Rate limit headers:
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1705312800
```

---

## SDKs

Official SDKs available:
- [JavaScript/TypeScript](https://github.com/arka-engi../arka-js)
- [Python](https://github.com/arka-engi../arka-python)
- [Go](https://github.com/arka-engi../arka-go)
- [Java](https://github.com/arka-engi../arka-java)

---

## Next Steps

- [gRPC API Reference](./grpc-api.md) - gRPC service definitions
- [SDK Reference](./sdk-reference.md) - Client SDK documentation
- [Integration Patterns](../guides/integration-patterns.md) - Common integrations
