# ARKA Engine Architecture

## System Overview

ARKA (Protocol for Automated Compliance Technology) is a comprehensive compliance operating system that combines rule-based decision making, AI-powered analysis, and blockchain-backed audit trails.

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                                    ARKA ENGINE                                        │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐ │
│  │                              Applications                                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  │ │
│  │  │ arka-ai-api │  │ chain-node  │  │  explorer   │  │   Other ARKA Apps     │  │ │
│  │  │   :3010     │  │    :4040    │  │    :4041    │  │                       │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────────────────┘  │ │
│  └─────────────────────────────────────────────────────────────────────────────────┘ │
│                                          │                                            │
│  ┌───────────────────────────────────────┼───────────────────────────────────────┐   │
│  │                               Core Packages                                    │   │
│  │                                       │                                        │   │
│  │  ┌────────────────────────────────────┴───────────────────────────────────┐   │   │
│  │  │                                                                         │   │   │
│  │  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                  │   │   │
│  │  │   │ @arka/types │   │ @arka/utils │   │ @arka/crypto│                  │   │   │
│  │  │   │  (Shared)   │   │  (Helpers)  │   │  (Security) │                  │   │   │
│  │  │   └─────────────┘   └─────────────┘   └─────────────┘                  │   │   │
│  │  │                                                                         │   │   │
│  │  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                  │   │   │
│  │  │   │ @arka/core  │   │@arka/plugin │   │@arka/api-gw │                  │   │   │
│  │  │   │  (Rules)    │   │    -sdk     │   │  (Gateway)  │                  │   │   │
│  │  │   └─────────────┘   └─────────────┘   └─────────────┘                  │   │   │
│  │  │                                                                         │   │   │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                                │   │
│  │  ┌─────────────────────────────────┐  ┌────────────────────────────────────┐  │   │
│  │  │         AI Layer                │  │        Blockchain Layer            │  │   │
│  │  │                                 │  │                                    │  │   │
│  │  │  ┌──────────┐  ┌──────────┐    │  │  ┌───────────┐  ┌───────────┐     │  │   │
│  │  │  │ ai-core  │  │ ai-risk  │    │  │  │chain-core │  │chain-node │     │  │   │
│  │  │  └──────────┘  └──────────┘    │  │  └───────────┘  └───────────┘     │  │   │
│  │  │  ┌──────────┐  ┌──────────┐    │  │  ┌───────────────────────────┐     │  │   │
│  │  │  │ai-monitor│  │ai-copilot│    │  │  │      chain-client         │     │  │   │
│  │  │  └──────────┘  └──────────┘    │  │  └───────────────────────────┘     │  │   │
│  │  │  ┌──────────────────────────┐  │  │                                    │  │   │
│  │  │  │    ai-remediation        │  │  │                                    │  │   │
│  │  │  └──────────────────────────┘  │  │                                    │  │   │
│  │  └─────────────────────────────────┘  └────────────────────────────────────┘  │   │
│  │                                                                                │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐ │
│  │                                Plugins                                           │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │ │
│  │  │   AML    │ │   KYC    │ │ Sanctions│ │  RegDoc  │ │ TaxVault │ │  Custom  │ │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Package Structure

### Foundation Packages

| Package | Purpose | Dependencies |
|---------|---------|--------------|
| `@arka/types` | Shared TypeScript type definitions | None |
| `@arka/utils` | Common utilities and helpers | `@arka/types` |
| `@arka/crypto` | Cryptographic operations | `@arka/types` |
| `@arka/core` | Rules engine and decision logic | `@arka/types`, `@arka/utils` |
| `@arka/plugin-sdk` | Plugin development kit | `@arka/types`, `@arka/core` |
| `@arka/api-gateway` | API gateway and routing | `@arka/types`, `@arka/core` |

### AI Layer Packages

| Package | Purpose | Key Features |
|---------|---------|--------------|
| `@arka/ai-core` | AI provider abstraction | Multi-provider support, prompt management |
| `@arka/ai-risk` | Risk assessment engine | Scoring, analysis, profiling |
| `@arka/ai-monitor` | Compliance monitoring | Anomaly detection, alerts |
| `@arka/ai-copilot` | Conversational assistant | NLU, multi-turn conversations |
| `@arka/ai-remediation` | Issue remediation | Analysis, recommendations, workflows |

### Blockchain Layer Packages

| Package | Purpose | Key Features |
|---------|---------|--------------|
| `@arka/chain-core` | Blockchain engine | Hashing, Merkle trees, blocks, storage |
| `@arka/chain-node` | Node service | Mempool, mining, transaction processing |
| `@arka/chain-client` | Client library | Local, HTTP, cloud integration |

### Applications

| App | Port | Purpose |
|-----|------|---------|
| `arka-ai-api` | 3010 | AI services HTTP API |
| `arka-chain-node-api` | 4040 | Blockchain node HTTP API |
| `arka-chain-explorer` | 4041 | Blockchain web explorer |

---

## Data Flow

### Compliance Event Processing

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   External  │    │    ARKA     │    │    ARKA     │    │    ARKA     │
│    Event    │───▶│    Core     │───▶│     AI      │───▶│ Blockchain  │
│   (Input)   │    │  (Rules)    │    │ (Analysis)  │    │  (Record)   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │                  │
       │                  ▼                  ▼                  ▼
       │          ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
       │          │  Decision   │    │    Risk     │    │  Immutable  │
       │          │   Output    │    │   Score     │    │   Record    │
       │          └─────────────┘    └─────────────┘    └─────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Event Types                                   │
│  • Transaction screening    • KYC verification                       │
│  • Sanctions check          • Regulatory filing                      │
│  • Risk assessment          • Audit action                           │
└─────────────────────────────────────────────────────────────────────┘
```

### AI-Assisted Decision Flow

```
┌─────────────┐
│   Query/    │
│   Event     │
└──────┬──────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                        AI Layer                                   │
│                                                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  AI Core    │───▶│   Model     │───▶│  Response   │          │
│  │  (Prompt)   │    │  Provider   │    │  Parsing    │          │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
│                                               │                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Specialized Modules                       │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │ │
│  │  │   Risk   │  │ Monitor  │  │ Copilot  │  │Remediate │    │ │
│  │  │ Scoring  │  │ Anomaly  │  │  Query   │  │ Suggest  │    │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────┬──────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   AI-Enhanced       │
                    │   Decision/Output   │
                    └─────────────────────┘
```

### Blockchain Recording Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Compliance │    │   Create    │    │   Submit    │    │    Mine     │
│    Event    │───▶│ Transaction │───▶│  to Node    │───▶│   Block     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                          │                  │                  │
                          ▼                  ▼                  ▼
                   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
                   │   Hash TX   │    │   Mempool   │    │  Merkle     │
                   │  (SHA-256)  │    │  (Pending)  │    │   Root      │
                   └─────────────┘    └─────────────┘    └─────────────┘
                                                               │
                                                               ▼
                                                        ┌─────────────┐
                                                        │  Immutable  │
                                                        │    Chain    │
                                                        └─────────────┘
```

---

## Integration Patterns

### Pattern 1: Rule + AI Hybrid Decision

```typescript
import { RulesEngine } from '@arka/core';
import { RiskAssessor } from '@arka/ai-risk';
import { PactChainClient } from '@arka/chain-client';

async function processTransaction(tx) {
  // Step 1: Rule-based evaluation
  const ruleResult = await rulesEngine.evaluate('aml-threshold', tx);

  // Step 2: AI risk assessment
  const aiRisk = await riskAssessor.assessTransaction(tx);

  // Step 3: Combined decision
  const decision = {
    approved: ruleResult.decision === 'ALLOW' && aiRisk.score < 80,
    ruleDecision: ruleResult.decision,
    aiRiskScore: aiRisk.score,
    requiresReview: aiRisk.score >= 60 && aiRisk.score < 80,
  };

  // Step 4: Record on blockchain
  await chainClient.submitEventTransaction({
    type: 'ARKA_EVENT',
    eventId: `tx-${tx.id}`,
    entityId: tx.accountId,
    eventType: 'TRANSACTION_SCREENED',
    payload: {
      transaction: tx,
      decision: decision,
      ruleResult: ruleResult,
      aiRisk: aiRisk,
    },
  });

  return decision;
}
```

### Pattern 2: AI-Monitored Compliance

```typescript
import { ComplianceMonitor } from '@arka/ai-monitor';
import { PactChainClient } from '@arka/chain-client';

const monitor = new ComplianceMonitor({ provider });

// Subscribe to anomalies
monitor.on('anomaly', async (anomaly) => {
  // Record anomaly on blockchain
  await chainClient.submitEventTransaction({
    type: 'ARKA_EVENT',
    eventId: `anomaly-${anomaly.id}`,
    entityId: anomaly.entityId,
    eventType: 'ANOMALY_DETECTED',
    payload: anomaly,
  });

  // Generate alert
  await alertService.send({
    type: 'compliance_anomaly',
    severity: anomaly.severity,
    details: anomaly,
  });
});

// Process events continuously
eventStream.on('event', (event) => {
  monitor.processEvent(event);
});
```

### Pattern 3: Copilot-Assisted Workflow

```typescript
import { ComplianceCopilot } from '@arka/ai-copilot';
import { RemediationEngine } from '@arka/ai-remediation';

async function handleComplianceIssue(issue) {
  // Get AI explanation
  const explanation = await copilot.query(
    `Explain this compliance issue: ${issue.description}`
  );

  // Get remediation recommendations
  const remediation = await remediationEngine.analyzeIssue(issue);

  // Present to compliance officer
  return {
    issue: issue,
    explanation: explanation.answer,
    recommendations: remediation.recommendations,
    workflow: remediation.workflow,
    regulatoryContext: remediation.regulatoryImpact,
  };
}
```

---

## Security Model

### Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────────┐
│                      Security Layers                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: API Gateway                                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • JWT/OAuth2 authentication                                │ │
│  │  • Rate limiting                                            │ │
│  │  • Request validation                                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Layer 2: Service Authorization                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • Role-based access control (RBAC)                         │ │
│  │  • Service-to-service auth (mTLS)                           │ │
│  │  • Permission scoping                                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Layer 3: Data Protection                                        │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • Encryption at rest (AES-256)                             │ │
│  │  • Encryption in transit (TLS 1.3)                          │ │
│  │  • PII masking in AI prompts                                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Layer 4: Blockchain Integrity                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • SHA-256 transaction hashing                              │ │
│  │  • Merkle tree verification                                 │ │
│  │  • Immutable audit trail                                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Audit Trail

All compliance-relevant actions are recorded on the blockchain:

| Event Type | What's Recorded |
|------------|-----------------|
| `KYC_VERIFICATION` | Verification result, risk score, documents checked |
| `TRANSACTION_SCREENED` | Transaction details, decision, rules triggered |
| `SANCTIONS_CHECK` | Lists checked, match results, confidence |
| `COMPLIANCE_ALERT` | Alert details, severity, assigned owner |
| `AUDIT_ACCESS` | Who accessed what data, when, why |
| `RULE_UPDATED` | Rule changes, approver, effective date |

---

## Deployment Architecture

### Local Development

```
┌─────────────────────────────────────────────────────────────────┐
│                    Local Development                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  AI API      │  │  Chain Node  │  │  Explorer    │          │
│  │  :3010       │  │  :4040       │  │  :4041       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                    ┌──────────────┐                              │
│                    │   In-Memory  │                              │
│                    │   Storage    │                              │
│                    └──────────────┘                              │
└─────────────────────────────────────────────────────────────────┘

Commands:
  pnpm chain:start     # Start blockchain node
  pnpm chain:explorer  # Start explorer
  pnpm chain:demo      # Run demo
```

### Production (AWS)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              AWS Region                                       │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                           VPC                                            │ │
│  │                                                                          │ │
│  │  ┌─────────────────────┐         ┌─────────────────────┐                │ │
│  │  │    Public Subnet    │         │    Public Subnet    │                │ │
│  │  │       (AZ-a)        │         │       (AZ-b)        │                │ │
│  │  │                     │         │                     │                │ │
│  │  │  ┌───────────────┐  │         │  ┌───────────────┐  │                │ │
│  │  │  │     ALB       │  │         │  │     ALB       │  │                │ │
│  │  │  └───────────────┘  │         │  └───────────────┘  │                │ │
│  │  │         │           │         │         │           │                │ │
│  │  │  ┌───────────────┐  │         │  ┌───────────────┐  │                │ │
│  │  │  │ ECS Fargate   │  │         │  │ ECS Fargate   │  │                │ │
│  │  │  │ (Chain Node)  │  │         │  │ (Chain Node)  │  │                │ │
│  │  │  └───────────────┘  │         │  └───────────────┘  │                │ │
│  │  │                     │         │                     │                │ │
│  │  └─────────────────────┘         └─────────────────────┘                │ │
│  │                                                                          │ │
│  │  ┌─────────────────────┐         ┌─────────────────────┐                │ │
│  │  │   Private Subnet    │         │   Private Subnet    │                │ │
│  │  │                     │         │                     │                │ │
│  │  │  ┌───────────────┐  │         │  ┌───────────────┐  │                │ │
│  │  │  │   RDS/Aurora  │◀─┼─────────┼─▶│   RDS/Aurora  │  │                │ │
│  │  │  │  (Indexing)   │  │         │  │   (Replica)   │  │                │ │
│  │  │  └───────────────┘  │         │  └───────────────┘  │                │ │
│  │  │                     │         │                     │                │ │
│  │  │  ┌───────────────┐  │         │  ┌───────────────┐  │                │ │
│  │  │  │     EFS       │◀─┼─────────┼─▶│     EFS       │  │                │ │
│  │  │  │ (Chain Data)  │  │         │  │   (Replica)   │  │                │ │
│  │  │  └───────────────┘  │         │  └───────────────┘  │                │ │
│  │  └─────────────────────┘         └─────────────────────┘                │ │
│  │                                                                          │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                               │
│  External Services:                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                        │
│  │  CloudWatch  │  │   Bedrock    │  │    Secrets   │                        │
│  │   (Logs)     │  │    (AI)      │  │   Manager    │                        │
│  └──────────────┘  └──────────────┘  └──────────────┘                        │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Commands

| Command | Description |
|---------|-------------|
| `pnpm build` | Build all packages |
| `pnpm test` | Run all tests |
| `pnpm chain:start` | Start blockchain node |
| `pnpm chain:explorer` | Start block explorer |
| `pnpm chain:demo` | Run blockchain demo |

### Ports

| Service | Port | Description |
|---------|------|-------------|
| AI API | 3010 | ARKA AI HTTP API |
| Chain Node | 4040 | Blockchain node API |
| Explorer | 4041 | Block explorer UI |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `AI_PROVIDER` | AI provider (mock, bedrock, openai) |
| `ARKA_CHAIN_ID` | Blockchain chain ID |
| `ARKA_NODE_ID` | Blockchain node ID |

---

## Further Reading

- [ARKA AI Documentation](./ARKA-AI.md)
- [ARKA Blockchain Documentation](./PACT-BLOCKCHAIN.md)
- [Plugin Development Guide](./PLUGIN-DEVELOPMENT.md)
- [API Reference](./API-REFERENCE.md)
