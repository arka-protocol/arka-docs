# ARKA AI Layer

## Overview

The ARKA AI Layer provides intelligent compliance automation through machine learning and natural language processing. It enables real-time risk assessment, automated monitoring, conversational compliance assistance, and AI-driven remediation recommendations.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ARKA AI Layer                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │  AI Core    │  │  AI Risk    │  │ AI Monitor  │  │ AI Remediation  │ │
│  │             │  │             │  │             │  │                 │ │
│  │ - Provider  │  │ - Scoring   │  │ - Anomaly   │  │ - Analysis      │ │
│  │ - Prompts   │  │ - Analysis  │  │ - Alerts    │  │ - Actions       │ │
│  │ - Tools     │  │ - Factors   │  │ - Patterns  │  │ - Workflows     │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘ │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         AI Copilot                                   │ │
│  │  - Natural language compliance queries                               │ │
│  │  - Context-aware responses                                           │ │
│  │  - Multi-turn conversations                                          │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         arka-ai-api                                  │ │
│  │  HTTP REST API - Port 3010                                           │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Packages

### @arka/ai-core

The foundation package providing AI provider abstraction, prompt management, and common utilities.

**Key Features:**
- Multi-provider support (AWS Bedrock, OpenAI, Anthropic)
- Prompt template management with variable substitution
- Token estimation and cost tracking
- Retry logic with exponential backoff
- Response caching

**Usage:**
```typescript
import { createAIProvider, PromptManager } from '@arka/ai-core';

// Create AI provider
const provider = createAIProvider({
  provider: 'bedrock',
  model: 'anthropic.claude-3-sonnet',
  region: 'us-east-1',
});

// Use prompt manager
const promptManager = new PromptManager();
promptManager.registerPrompt('risk-analysis', {
  template: 'Analyze the following transaction for compliance risks: {{transaction}}',
  variables: ['transaction'],
});

const prompt = promptManager.render('risk-analysis', {
  transaction: JSON.stringify(txData),
});

const response = await provider.complete(prompt);
```

**API Reference:**

| Class/Function | Description |
|----------------|-------------|
| `createAIProvider(config)` | Factory function to create AI provider instance |
| `PromptManager` | Manages prompt templates with variable substitution |
| `estimateTokens(text)` | Estimates token count for text |
| `AIProviderConfig` | Configuration interface for providers |

---

### @arka/ai-risk

Intelligent risk assessment and scoring engine powered by AI.

**Key Features:**
- Multi-factor risk scoring (0-100 scale)
- Transaction pattern analysis
- Entity risk profiling
- Historical risk trend analysis
- Configurable risk thresholds

**Usage:**
```typescript
import { RiskAssessor, createRiskProfile } from '@arka/ai-risk';

const assessor = new RiskAssessor({
  provider: aiProvider,
  thresholds: {
    low: 30,
    medium: 60,
    high: 80,
    critical: 95,
  },
});

// Assess transaction risk
const result = await assessor.assessTransaction({
  amount: 50000,
  currency: 'USD',
  counterparty: 'ACME Corp',
  jurisdiction: 'US',
  transactionType: 'wire_transfer',
});

console.log(result);
// {
//   score: 45,
//   level: 'medium',
//   factors: [
//     { name: 'amount', score: 60, weight: 0.3 },
//     { name: 'jurisdiction', score: 20, weight: 0.2 },
//     { name: 'counterparty', score: 50, weight: 0.3 },
//     { name: 'pattern', score: 40, weight: 0.2 },
//   ],
//   recommendations: ['Enhanced due diligence recommended'],
//   explanation: 'Medium risk due to transaction amount...',
// }

// Create entity risk profile
const profile = await createRiskProfile({
  entityId: 'customer-123',
  entityType: 'individual',
  transactions: historicalTxs,
  kycData: kycInfo,
});
```

**Risk Factors:**

| Factor | Weight | Description |
|--------|--------|-------------|
| Amount | 0.25 | Transaction value relative to thresholds |
| Jurisdiction | 0.20 | Country/region risk rating |
| Counterparty | 0.20 | Known entity risk or new relationship |
| Pattern | 0.15 | Deviation from historical patterns |
| Velocity | 0.10 | Transaction frequency analysis |
| Time | 0.10 | Unusual timing indicators |

---

### @arka/ai-monitor

Real-time compliance monitoring with anomaly detection.

**Key Features:**
- Streaming event monitoring
- Anomaly detection algorithms
- Pattern recognition
- Alert generation and routing
- Compliance threshold monitoring

**Usage:**
```typescript
import { ComplianceMonitor, AnomalyDetector } from '@arka/ai-monitor';

const monitor = new ComplianceMonitor({
  provider: aiProvider,
  alertThreshold: 0.8,
  windowSize: 100,
});

// Start monitoring
monitor.on('anomaly', (event) => {
  console.log('Anomaly detected:', event);
  // {
  //   type: 'velocity_spike',
  //   severity: 'high',
  //   entityId: 'account-456',
  //   score: 0.92,
  //   details: 'Transaction velocity 3x normal rate',
  // }
});

monitor.on('alert', (alert) => {
  // Route to compliance team
  notificationService.send(alert);
});

// Process events
await monitor.processEvent({
  type: 'TRANSACTION',
  entityId: 'account-456',
  amount: 100000,
  timestamp: new Date(),
});

// Anomaly detection
const detector = new AnomalyDetector({
  algorithm: 'isolation_forest',
  contamination: 0.1,
});

const anomalies = await detector.detect(transactionBatch);
```

**Alert Types:**

| Type | Description | Default Threshold |
|------|-------------|-------------------|
| `velocity_spike` | Unusual transaction frequency | 3x baseline |
| `amount_outlier` | Amount significantly above normal | 5x average |
| `new_counterparty` | First transaction with entity | Always alert |
| `jurisdiction_change` | New high-risk jurisdiction | Risk score > 70 |
| `pattern_deviation` | Behavior doesn't match profile | 2 std deviations |

---

### @arka/ai-copilot

Conversational AI assistant for compliance queries.

**Key Features:**
- Natural language understanding
- Context-aware responses
- Multi-turn conversations
- Compliance knowledge base integration
- Citation and source tracking

**Usage:**
```typescript
import { ComplianceCopilot, ConversationManager } from '@arka/ai-copilot';

const copilot = new ComplianceCopilot({
  provider: aiProvider,
  knowledgeBase: complianceKB,
});

// Single query
const response = await copilot.query(
  'What are the KYC requirements for a high-risk customer?'
);

console.log(response);
// {
//   answer: 'For high-risk customers, enhanced due diligence (EDD) is required...',
//   sources: [
//     { document: 'KYC Policy v2.3', section: '4.2.1' },
//     { regulation: 'BSA/AML', citation: '31 CFR 1020.220' },
//   ],
//   confidence: 0.95,
// }

// Multi-turn conversation
const conversation = new ConversationManager();

const msg1 = await copilot.chat(conversation, 'What is a PEP?');
const msg2 = await copilot.chat(conversation, 'How do I screen for them?');
const msg3 = await copilot.chat(conversation, 'What if the match is uncertain?');

// Get conversation history
const history = conversation.getHistory();
```

**Supported Query Types:**

| Category | Example Queries |
|----------|-----------------|
| Regulatory | "What does FATF Recommendation 16 require?" |
| Procedural | "How do I file a SAR?" |
| Risk | "Is this transaction suspicious?" |
| Entity | "What is the risk profile for this customer?" |
| Policy | "What is our policy on PEP relationships?" |

---

### @arka/ai-remediation

AI-powered compliance issue remediation recommendations.

**Key Features:**
- Issue classification and prioritization
- Automated remediation suggestions
- Workflow generation
- Impact analysis
- Regulatory mapping

**Usage:**
```typescript
import { RemediationEngine, IssueAnalyzer } from '@arka/ai-remediation';

const engine = new RemediationEngine({
  provider: aiProvider,
  policyRepository: policies,
});

// Analyze compliance issue
const analysis = await engine.analyzeIssue({
  type: 'missing_documentation',
  entityId: 'customer-789',
  description: 'KYC documentation expired 30 days ago',
  severity: 'medium',
});

console.log(analysis);
// {
//   classification: 'documentation_gap',
//   priority: 2,
//   rootCause: 'Automated renewal process failed',
//   recommendations: [
//     {
//       action: 'request_updated_documents',
//       description: 'Send document renewal request to customer',
//       deadline: '2024-01-15',
//       owner: 'kyc_team',
//     },
//     {
//       action: 'restrict_activity',
//       description: 'Apply transaction limits until resolved',
//       deadline: 'immediate',
//       owner: 'operations',
//     },
//   ],
//   regulatoryImpact: ['BSA/AML 31 CFR 1020.220'],
//   estimatedResolutionTime: '5 business days',
// }

// Generate remediation workflow
const workflow = await engine.generateWorkflow(analysis);
```

---

## arka-ai-api

HTTP REST API for accessing ARKA AI services.

**Base URL:** `http://localhost:3010`

### Endpoints

#### Health Check
```
GET /health
```
Response:
```json
{
  "status": "ok",
  "version": "0.1.0",
  "timestamp": "2024-01-10T10:00:00.000Z"
}
```

#### Risk Assessment
```
POST /risk/assess
Content-Type: application/json

{
  "entityId": "customer-123",
  "entityType": "individual",
  "transaction": {
    "amount": 50000,
    "currency": "USD",
    "counterparty": "ACME Corp",
    "type": "wire_transfer"
  }
}
```
Response:
```json
{
  "score": 45,
  "level": "medium",
  "factors": [...],
  "recommendations": [...]
}
```

#### Copilot Query
```
POST /copilot/query
Content-Type: application/json

{
  "query": "What are the SAR filing requirements?",
  "context": {
    "jurisdiction": "US",
    "entityType": "bank"
  }
}
```
Response:
```json
{
  "answer": "SAR filing is required when...",
  "sources": [...],
  "confidence": 0.92
}
```

#### Monitor Event
```
POST /monitor/event
Content-Type: application/json

{
  "type": "TRANSACTION",
  "entityId": "account-456",
  "data": {
    "amount": 100000,
    "currency": "USD"
  }
}
```
Response:
```json
{
  "processed": true,
  "alerts": [],
  "anomalyScore": 0.12
}
```

#### Remediation Analysis
```
POST /remediation/analyze
Content-Type: application/json

{
  "issueType": "missing_documentation",
  "entityId": "customer-789",
  "description": "KYC documentation expired",
  "severity": "medium"
}
```
Response:
```json
{
  "classification": "documentation_gap",
  "priority": 2,
  "recommendations": [...],
  "workflow": {...}
}
```

---

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AI_PROVIDER` | AI provider (bedrock, openai, anthropic) | `mock` |
| `AI_MODEL` | Model identifier | `anthropic.claude-3-sonnet` |
| `AWS_REGION` | AWS region for Bedrock | `us-east-1` |
| `OPENAI_API_KEY` | OpenAI API key | - |
| `ANTHROPIC_API_KEY` | Anthropic API key | - |
| `AI_MAX_TOKENS` | Maximum response tokens | `4096` |
| `AI_TEMPERATURE` | Response temperature | `0.7` |
| `PORT` | API server port | `3010` |

### Provider Configuration

**AWS Bedrock:**
```typescript
const config = {
  provider: 'bedrock',
  model: 'anthropic.claude-3-sonnet-20240229-v1:0',
  region: 'us-east-1',
  maxTokens: 4096,
};
```

**OpenAI:**
```typescript
const config = {
  provider: 'openai',
  model: 'gpt-4-turbo',
  apiKey: process.env.OPENAI_API_KEY,
  maxTokens: 4096,
};
```

**Mock (for testing):**
```typescript
const config = {
  provider: 'mock',
  responses: {
    'risk-assessment': { score: 50, level: 'medium' },
  },
};
```

---

## Testing

Run all AI tests:
```bash
pnpm --filter "@arka/ai-*" test
```

Run specific package tests:
```bash
pnpm --filter @arka/ai-core test
pnpm --filter @arka/ai-risk test
pnpm --filter @arka/ai-monitor test
pnpm --filter @arka/ai-copilot test
pnpm --filter @arka/ai-remediation test
```

---

## Best Practices

### 1. Prompt Engineering
- Use specific, structured prompts
- Include relevant context
- Define expected output format
- Test prompts with edge cases

### 2. Rate Limiting
- Implement request throttling
- Use caching for repeated queries
- Batch similar requests

### 3. Error Handling
- Graceful degradation when AI unavailable
- Fallback to rule-based logic
- Log all AI interactions for audit

### 4. Cost Management
- Monitor token usage
- Use appropriate model sizes
- Cache frequent responses

### 5. Security
- Never include PII in prompts without masking
- Audit all AI interactions
- Review AI outputs before automated actions

---

## Integration Examples

### With ARKA Core Rules Engine
```typescript
import { RulesEngine } from '@arka/core';
import { RiskAssessor } from '@arka/ai-risk';

const rulesEngine = new RulesEngine();
const riskAssessor = new RiskAssessor({ provider });

// Combine rule-based and AI assessment
async function assessTransaction(tx) {
  // Rule-based check
  const ruleResult = await rulesEngine.evaluate('aml-threshold', tx);

  // AI risk assessment
  const aiResult = await riskAssessor.assessTransaction(tx);

  // Combine results
  return {
    ruleDecision: ruleResult.decision,
    aiRiskScore: aiResult.score,
    finalDecision: combineDecisions(ruleResult, aiResult),
  };
}
```

### With ARKA Blockchain
```typescript
import { PactChainClient } from '@arka/chain-client';
import { ComplianceMonitor } from '@arka/ai-monitor';

const chainClient = new PactChainClient({ baseUrl: 'http://localhost:4040' });
const monitor = new ComplianceMonitor({ provider });

// Monitor and record compliance events
monitor.on('alert', async (alert) => {
  // Record alert on blockchain
  await chainClient.submitEventTransaction({
    type: 'ARKA_EVENT',
    eventId: `alert-${alert.id}`,
    entityId: alert.entityId,
    eventType: 'COMPLIANCE_ALERT',
    payload: alert,
  });
});
```

---

## Roadmap

### v0.2.0
- [ ] Streaming responses
- [ ] Multi-modal document analysis
- [ ] Fine-tuned compliance models

### v0.3.0
- [ ] Automated SAR narrative generation
- [ ] Cross-entity relationship analysis
- [ ] Regulatory change impact assessment

### v1.0.0
- [ ] Production-ready Bedrock integration
- [ ] SOC 2 compliant logging
- [ ] Enterprise SLA support
