# ARKA Protocol Architecture v2

## Document Status
- **Version**: 2.0.0
- **Last Updated**: 2024
- **Status**: Implementation In Progress

---

## Part 1: Current Architecture Summary

### Repository Structure

```
arka-engine/
├── apps/
│   ├── arka-core-service/       # HTTP API for rules engine
│   ├── arka-ai-gateway/         # AI abstraction layer (OpenRouter)
│   └── arka-cortex-service/     # AI meta-engine
├── packages/
│   ├── arka-types/              # Shared TypeScript types
│   ├── arka-core/               # Deterministic rules engine library
│   ├── arka-utils/              # Shared utilities
│   ├── arka-test-data/          # Test fixtures
│   ├── arka-crypto/             # Merkle trees, hashing, serialization
│   ├── arka-blockchain/         # Multi-chain blockchain adapters
│   └── arka-plugin-sdk/         # Plugin development SDK
├── plugins/
│   └── arka-loans/              # Loan compliance domain plugin
└── tools/
    ├── pactctl/                 # CLI management tool
    └── arka-installer/          # Deployment wizard
```

### Core Components

#### ARKA Core (`packages/arka-core`)
- Deterministic, pure rules engine
- Rule DSL with logical operators (AND, OR, NOT)
- Comparison operators (==, !=, <, >, in, contains, regex, etc.)
- Entity validation via JSON Schema
- Decision auditing with integrity hashes

#### ARKA Core Service (`apps/arka-core-service`)
- REST API for event processing
- Rule CRUD operations
- Entity management
- Audit trail storage (PostgreSQL)

#### ARKA AI Gateway (`apps/arka-ai-gateway`)
- OpenRouter integration
- Natural language → Rule DSL conversion
- Risk analysis, legal diff, explanations

#### ARKA Cortex Service (`apps/arka-cortex-service`)
- AI meta-engine for rule proposals
- Simulations and backtesting
- Legal document ingestion
- Anomaly detection

### Data Models

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   events    │────▶│  decisions  │────▶│   audits    │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │
       ▼                   ▼
┌─────────────┐     ┌─────────────┐
│  entities   │     │    rules    │
└─────────────┘     └─────────────┘
```

### Existing APIs

**Core Service (port 3001)**:
- `POST /v1/events/process` - Process event against rules
- `GET /v1/decisions/:id` - Retrieve decision details
- `POST/GET/PUT/DELETE /v1/rules` - Rule management
- `POST/GET /v1/entities` - Entity management

**Cortex Service (port 3003)**:
- `POST /v1/rules/propose-from-text` - AI rule proposal
- `POST /v1/simulations/run` - Run simulations
- `POST /v1/legal/ingest` - Ingest legal documents

---

## Part 2: ARKA v2 Enhancements Overview

The following 20 enhancements extend ARKA Protocol with advanced capabilities for enterprise compliance.

### 1) Verifiable AI Governance Layer
Track and verify all AI-generated proposals with versioning, hashing, and consistency scoring by a critic model. Every AI output becomes an auditable artifact.

### 2) Multi-Model OpenRouter Ensemble
For critical tasks, query multiple AI models in parallel and use a judge model to select/merge the best outputs. Improves reliability for NL→Rule and legal ingestion.

### 3) Rule-Time Virtual Machine (RTVM)
A deterministic bytecode VM for rule execution. Compiles Rule DSL to bytecode for faster, verifiable execution with bounded steps and no side effects.

### 4) Autonomous Rule Graph Engine
Represent rules as a directed graph with dependencies, precedence, and overrides. Detect conflicts and compute optimal execution order automatically.

### 5) Time-Travel Simulations (Temporal Compliance Engine)
Simulate decisions at any historical point in time. Reconstruct rule sets and entity states as they existed at a specific moment for audit and analysis.

### 6) Multi-Jurisdiction Harmonization Engine
Detect and resolve rule conflicts across jurisdictions. AI-assisted suggestions for creating harmonized rules or override hierarchies.

### 7) Real-Time Event Ingestion Mesh
Support multiple ingestion patterns beyond HTTP: Kafka, MQTT, NATS. Modular adapter architecture for enterprise event streaming.

### 8) Zero-Trust Rule Verification (ZK-Compliance)
Design hooks for zero-knowledge proofs of correct rule execution. Prove compliance without revealing sensitive data. (Stub implementation ready for ZK integration)

### 9) Autonomous Legal Change Detection
Monitor legal/regulatory text changes and automatically trigger rule proposals when documents are updated. Built-in diff analysis for legal text.

### 10) Distributed Compliance Agents
Edge deployment pattern for running lightweight ARKA nodes that sync rules and report decisions back to a central coordinator.

### 11) Compliance Policy Diff Engine
Git-style structured diff for rules. Compare any two rule versions and see exactly what changed in conditions, consequences, and metadata.

### 12) Multi-Ledger Integration Abstraction
Clean abstraction layer for anchoring to multiple blockchains. Pluggable chain adapters with unified interface.

### 13) Adaptive Compliance Scoring Engine
Calculate risk scores per entity based on violation history, decision patterns, and AI-enhanced risk models. Continuous risk monitoring.

### 14) Global Rule Language (GRL) Specification
Formal specification of the Rule DSL as a branded "Global Rule Language" with versioning, syntax documentation, and validation tools.

### 15) Role-Based Rule Execution Governance
Multi-signature approval workflow for rules. Rules require sign-off from designated roles before activation. Audit trail of approvals.

### 16) Temporal Entity Snapshots & Multi-Entity Joins
Cross-entity rule conditions (e.g., loan.borrower.creditScore) and point-in-time entity state reconstruction.

### 17) Neural Compliance Linking (NCL)
AI-powered pattern recognition over decision history. Cluster similar decisions, identify anomalies, and surface emerging compliance risks.

### 18) Global Compliance Graph (GCG)
Graph representation of all compliance artifacts: rules, entities, jurisdictions, patterns. Query relationships and visualize compliance landscape.

### 19) Real-Time Compliance Dashboard APIs
Backend APIs for dashboard UI: summary statistics, top violated rules, jurisdiction breakdowns, real-time metrics.

### 20) ARKA Developer Cloud (Dev Tools)
Enhanced developer experience: GRL builder, sample rule packs, local simulation tools, rule validation CLI.

---

## Part 3: Implementation Status

| # | Enhancement | Package/Location | Status |
|---|-------------|------------------|--------|
| 1 | AI Governance Layer | `arka-cortex-service`, `arka-ai-gateway` | 🔄 In Progress |
| 2 | Multi-Model Ensemble | `arka-ai-gateway` | ⏳ Pending |
| 3 | RTVM | `packages/arka-core/src/rtvm` | ⏳ Pending |
| 4 | Rule Graph Engine | `packages/arka-core/src/rule-graph` | ⏳ Pending |
| 5 | Time-Travel Simulations | `arka-core-service` | ⏳ Pending |
| 6 | Harmonization Engine | `packages/arka-core`, `arka-cortex-service` | ⏳ Pending |
| 7 | Event Ingestion Mesh | `arka-core-service/src/ingestion` | ⏳ Pending |
| 8 | ZK-Compliance | `packages/arka-core/src/zk` | ⏳ Pending |
| 9 | Legal Change Detection | `arka-cortex-service` | ⏳ Pending |
| 10 | Distributed Agents | `packages/arka-types`, `arka-core-service` | ⏳ Pending |
| 11 | Rule Diff Engine | `packages/arka-core/src/rule-diff` | ⏳ Pending |
| 12 | Multi-Ledger Adapter | `packages/arka-chain-adapter` | ⏳ Pending |
| 13 | Risk Scoring Engine | `packages/arka-core/src/scoring` | ⏳ Pending |
| 14 | GRL Specification | `docs/grl-spec-v1.md` | ⏳ Pending |
| 15 | Rule Governance | `arka-core-service` | ⏳ Pending |
| 16 | Temporal Entities | `packages/arka-core` | ⏳ Pending |
| 17 | Neural Compliance | `arka-cortex-service` | ⏳ Pending |
| 18 | Compliance Graph | `arka-cortex-service` | ⏳ Pending |
| 19 | Dashboard APIs | `arka-core-service` | ⏳ Pending |
| 20 | Developer Tools | `packages/arka-dev-tools` | ⏳ Pending |

---

## Part 4: New Packages & Modules

### New Packages
- `packages/arka-chain-adapter/` - Multi-ledger blockchain abstraction
- `packages/arka-dev-tools/` - Developer experience tools

### New Modules in arka-core
- `src/rtvm/` - Rule-Time Virtual Machine
- `src/rule-graph/` - Rule dependency graph
- `src/rule-diff/` - Rule comparison engine
- `src/scoring/` - Risk scoring engine
- `src/zk/` - Zero-knowledge proof interfaces
- `src/harmonization/` - Jurisdiction harmonization
- `src/temporal/` - Time-travel evaluation

### New Modules in arka-core-service
- `src/ingestion/` - Event ingestion adapters
- `src/dashboard/` - Dashboard API routes
- `src/governance/` - Rule approval workflows

### New Modules in arka-cortex-service
- `src/ai-governance/` - AI proposal tracking
- `src/legal-detection/` - Legal change monitoring
- `src/patterns/` - Neural compliance linking
- `src/graph/` - Global compliance graph

### New Modules in arka-ai-gateway
- `src/ensemble/` - Multi-model orchestration
- `src/consistency/` - AI output verification

---

## Part 5: Database Schema Extensions

### New Tables

```sql
-- AI Governance
CREATE TABLE ai_proposals (
  id UUID PRIMARY KEY,
  proposal_type VARCHAR(50) NOT NULL,
  payload JSONB NOT NULL,
  source_model VARCHAR(100),
  input_hash VARCHAR(64),
  output_hash VARCHAR(64),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE ai_evaluations (
  id UUID PRIMARY KEY,
  proposal_id UUID REFERENCES ai_proposals(id),
  critic_model VARCHAR(100),
  consistency_score DECIMAL(3,2),
  comments TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Rule Governance
CREATE TABLE rule_approvals (
  id UUID PRIMARY KEY,
  rule_id UUID REFERENCES rules(id),
  rule_version INT,
  approver_role VARCHAR(50),
  approver_id VARCHAR(100),
  signature TEXT,
  approved_at TIMESTAMP DEFAULT NOW()
);

-- Legal Sources
CREATE TABLE legal_sources (
  id UUID PRIMARY KEY,
  source_name VARCHAR(200),
  jurisdiction VARCHAR(50),
  text TEXT,
  version INT,
  text_hash VARCHAR(64),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Risk Scoring
CREATE TABLE entity_risk_scores (
  id UUID PRIMARY KEY,
  entity_id UUID REFERENCES entities(id),
  risk_score DECIMAL(5,2),
  risk_band VARCHAR(20),
  factors JSONB,
  calculated_at TIMESTAMP DEFAULT NOW()
);

-- Compliance Patterns
CREATE TABLE compliance_patterns (
  id UUID PRIMARY KEY,
  pattern_type VARCHAR(50),
  cluster_id VARCHAR(100),
  description TEXT,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Global Compliance Graph
CREATE TABLE graph_nodes (
  id UUID PRIMARY KEY,
  node_type VARCHAR(50),
  ref_id UUID,
  metadata JSONB
);

CREATE TABLE graph_edges (
  id UUID PRIMARY KEY,
  from_node_id UUID REFERENCES graph_nodes(id),
  to_node_id UUID REFERENCES graph_nodes(id),
  edge_type VARCHAR(50),
  metadata JSONB
);

-- Simulations
CREATE TABLE simulations (
  id UUID PRIMARY KEY,
  simulation_type VARCHAR(50),
  parameters JSONB,
  results JSONB,
  metrics JSONB,
  ai_proposal_id UUID REFERENCES ai_proposals(id),
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## Part 6: API Extensions

### Core Service New Endpoints

```
POST /v1/simulations/time-travel     # Time-travel simulation
POST /v1/rules/diff                  # Compare rule versions
POST /v1/rules/:id/approve           # Submit rule approval
GET  /v1/rules/:id/approvals         # Get rule approvals
GET  /v1/dashboard/summary           # Dashboard summary
GET  /v1/dashboard/rules/top-violated
GET  /v1/dashboard/jurisdictions/summary
```

### Cortex Service New Endpoints

```
POST /v1/harmonization/analyze       # Jurisdiction harmonization
POST /v1/legal/ingest-or-update      # Legal change detection
GET  /v1/patterns                    # Compliance patterns
GET  /v1/graph/query                 # Graph queries
```

---

## Part 7: Configuration Extensions

### AI Gateway Model Config

```typescript
// apps/arka-ai-gateway/config/models.config.ts
export const modelConfig = {
  ensemble: {
    'nl-to-rule': {
      models: ['anthropic/claude-3-opus', 'openai/gpt-4-turbo', 'google/gemini-pro'],
      judge: 'anthropic/claude-3-opus',
      strategy: 'best-of-n'
    },
    'legal-to-rule': {
      models: ['anthropic/claude-3-opus', 'openai/gpt-4-turbo'],
      judge: 'anthropic/claude-3-opus',
      strategy: 'merge'
    }
  },
  consistency: {
    criticModel: 'anthropic/claude-3-sonnet'
  }
};
```

### Ingestion Config

```typescript
// apps/arka-core-service/config/ingestion.config.ts
export const ingestionConfig = {
  http: { enabled: true },
  kafka: {
    enabled: false,
    brokers: ['localhost:9092'],
    topic: 'arka-events',
    groupId: 'arka-core'
  },
  mqtt: { enabled: false },
  nats: { enabled: false }
};
```

---

## Appendix: Design Principles

1. **Backwards Compatibility** - All existing APIs remain functional
2. **Modularity** - Each enhancement is a separate module
3. **Configurability** - Features can be enabled/disabled
4. **Testability** - Every component has unit tests
5. **Documentation** - All features documented inline and in docs/
6. **Type Safety** - Full TypeScript coverage
7. **Determinism** - Core evaluation remains pure and deterministic
