# System Overview

This document provides a comprehensive overview of the ARKA Engine architecture, design principles, and component relationships.

## Table of Contents

- [Design Principles](#design-principles)
- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Data Stores](#data-stores)
- [External Integrations](#external-integrations)
- [Deployment Topologies](#deployment-topologies)

---

## Design Principles

ARKA Engine is built on six foundational principles:

### 1. Determinism First

Every compliance evaluation must be **deterministic**—identical inputs always produce identical outputs, regardless of when or where the evaluation occurs.

```
Input A + Rules V1 + Context C = Output X (always)
```

This enables:
- Reproducible audit trails
- Dispute resolution through re-execution
- Distributed evaluation with consistent results

### 2. Cryptographic Verifiability

Every compliance decision is accompanied by a **cryptographic proof** that can be verified without re-executing the evaluation.

```
Trust Proof = Hash(Input) + Hash(Rules) + Hash(Execution) + Hash(Output) + Signature
```

This enables:
- Third-party verification
- Tamper detection
- Efficient auditing at scale

### 3. Temporal Consistency

The system maintains complete history, enabling evaluation against **any historical rule state**.

```
evaluate(input, rules_at_timestamp=T) → result_as_of_T
```

This enables:
- Retroactive compliance analysis
- Historical audit reconstruction
- Rule change impact assessment

### 4. Jurisdictional Awareness

Rules understand and respect **jurisdictional hierarchies** and conflicts.

```
International → Regional → National → State/Province → Local
```

This enables:
- Multi-jurisdiction compliance
- Conflict resolution
- Regulatory harmonization

### 5. Extensibility Through Plugins

Core functionality is extended through a **secure plugin framework** without modifying the core engine.

```
Core Engine + Plugin A + Plugin B + Plugin N = Extended Capability
```

This enables:
- Domain-specific compliance logic
- Custom integrations
- Third-party extensions

### 6. Edge-Native Distribution

Compliance evaluation can occur at the **network edge** with cryptographic guarantees equivalent to centralized evaluation.

```
Central Control Plane ←→ Edge Agents ←→ Local Systems
```

This enables:
- Low-latency decisions
- Offline operation
- Data locality compliance

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ARKA ENGINE ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         CLIENT LAYER                                    │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │ │
│  │  │Dashboard │  │   CLI    │  │   SDK    │  │   API    │               │ │
│  │  │  (Web)   │  │          │  │(Multiple)│  │ Clients  │               │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                          API GATEWAY                                    │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │ │
│  │  │  Auth    │  │   Rate   │  │  Route   │  │  Load    │               │ │
│  │  │          │  │ Limiting │  │          │  │ Balance  │               │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         CORE SERVICES                                   │ │
│  │                                                                         │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │ │
│  │  │   Rule Service  │  │ Evaluation Svc  │  │  Audit Service  │        │ │
│  │  │                 │  │                 │  │                 │        │ │
│  │  │ • CRUD Rules    │  │ • Route Evals   │  │ • Log Events    │        │ │
│  │  │ • Compile       │  │ • Aggregate     │  │ • Query Logs    │        │ │
│  │  │ • Version       │  │ • Cache Results │  │ • Export        │        │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │ │
│  │                                                                         │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │ │
│  │  │ Governance Svc  │  │ Anchoring Svc   │  │  Agent Service  │        │ │
│  │  │                 │  │                 │  │                 │        │ │
│  │  │ • Proposals     │  │ • Anchor Proofs │  │ • Register      │        │ │
│  │  │ • Workflows     │  │ • Verify        │  │ • Sync Rules    │        │ │
│  │  │ • Simulations   │  │ • Multi-Chain   │  │ • Collect Proofs│        │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                       EXECUTION LAYER                                   │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    RTVM CLUSTER                                  │   │ │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │   │ │
│  │  │  │  RTVM 1   │  │  RTVM 2   │  │  RTVM 3   │  │  RTVM N   │    │   │ │
│  │  │  │           │  │           │  │           │  │           │    │   │ │
│  │  │  │ Bytecode  │  │ Bytecode  │  │ Bytecode  │  │ Bytecode  │    │   │ │
│  │  │  │ Executor  │  │ Executor  │  │ Executor  │  │ Executor  │    │   │ │
│  │  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │                    PLUGIN HOST                                   │   │ │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │   │ │
│  │  │  │ AML       │  │ KYC       │  │ GDPR      │  │ Custom    │    │   │ │
│  │  │  │ Plugin    │  │ Plugin    │  │ Plugin    │  │ Plugin    │    │   │ │
│  │  │  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        DATA LAYER                                       │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │ │
│  │  │PostgreSQL│  │  Redis   │  │   S3/    │  │  Time-   │               │ │
│  │  │          │  │          │  │  Minio   │  │  series  │               │ │
│  │  │ • Rules  │  │ • Cache  │  │ • Proofs │  │ • Metrics│               │ │
│  │  │ • State  │  │ • Queue  │  │ • Blobs  │  │ • Events │               │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                       │
│                                      ▼                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                     EXTERNAL INTEGRATIONS                               │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │ │
│  │  │Blockchain│  │ Identity │  │ External │  │Monitoring│               │ │
│  │  │ Networks │  │Providers │  │   APIs   │  │ Systems  │               │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. API Gateway

The entry point for all client interactions.

**Responsibilities:**
- Authentication and authorization
- Rate limiting and throttling
- Request routing
- Load balancing
- TLS termination
- Request/response logging

**Technologies:** Kong, Envoy, or NGINX

### 2. Rule Service

Manages the lifecycle of compliance rules.

**Responsibilities:**
- Rule CRUD operations
- Rule compilation to bytecode
- Version management
- Rule graph maintenance
- Dependency resolution
- Rule validation

**Key APIs:**
```
POST   /rules              Create rule
GET    /rules/:id          Get rule
PUT    /rules/:id          Update rule
DELETE /rules/:id          Delete rule
GET    /rules/:id/versions Version history
POST   /rules/:id/compile  Compile rule
```

### 3. Evaluation Service

Orchestrates compliance evaluations.

**Responsibilities:**
- Route evaluations to RTVM instances
- Aggregate results from multiple rules
- Cache evaluation results
- Generate trust proofs
- Handle batch evaluations

**Key APIs:**
```
POST   /evaluate           Single evaluation
POST   /evaluate/batch     Batch evaluation
GET    /evaluations/:id    Get evaluation result
POST   /evaluate/replay    Replay historical evaluation
```

### 4. Audit Service

Maintains compliance audit trails.

**Responsibilities:**
- Log all compliance events
- Store trust proofs
- Query and search logs
- Export audit data
- Retention management

**Key APIs:**
```
GET    /audit/events       Query events
GET    /audit/proofs/:id   Get proof
POST   /audit/export       Export audit data
GET    /audit/reports      Generate reports
```

### 5. Governance Service

Manages rule change governance.

**Responsibilities:**
- Create and manage proposals
- Execute governance workflows
- Run simulations
- Generate semantic diffs
- Track approvals

**Key APIs:**
```
POST   /governance/proposals       Create proposal
GET    /governance/proposals/:id   Get proposal
POST   /governance/proposals/:id/approve   Approve
POST   /governance/proposals/:id/reject    Reject
POST   /governance/simulate        Run simulation
```

### 6. Anchoring Service

Handles blockchain anchoring.

**Responsibilities:**
- Anchor proofs to blockchains
- Verify anchored proofs
- Manage multi-chain anchoring
- Generate Merkle proofs
- Track anchoring status

**Key APIs:**
```
POST   /anchoring/anchor    Anchor proof
GET    /anchoring/:id       Get anchor status
POST   /anchoring/verify    Verify anchor
GET    /anchoring/chains    List supported chains
```

### 7. Agent Service

Manages distributed edge agents.

**Responsibilities:**
- Register agents
- Distribute rules to agents
- Collect proofs from agents
- Monitor agent health
- Handle agent attestation

**Key APIs:**
```
POST   /agents/register     Register agent
GET    /agents/:id          Get agent info
POST   /agents/:id/sync     Trigger sync
GET    /agents/:id/proofs   Get agent proofs
POST   /agents/:id/attest   Submit attestation
```

### 8. RTVM (Regulatory Technology Virtual Machine)

The deterministic rule execution engine.

**Responsibilities:**
- Execute compiled bytecode
- Maintain execution isolation
- Generate execution traces
- Enforce resource limits
- Provide deterministic results

**Characteristics:**
- Stack-based architecture
- Fixed-precision arithmetic
- Deterministic random (seeded)
- No external I/O during execution
- Complete execution tracing

### 9. Plugin Host

Manages and executes plugins.

**Responsibilities:**
- Load and initialize plugins
- Provide sandbox isolation
- Route requests to plugins
- Monitor plugin health
- Handle hot-reload

**Plugin Types:**
- Rule Executors
- Data Connectors
- Reporters
- Validators
- Transformers

---

## Data Stores

### PostgreSQL (Primary Database)

Stores structured, relational data:

| Table | Purpose |
|-------|---------|
| `rules` | Rule definitions and metadata |
| `rule_versions` | Immutable rule version history |
| `rule_graph` | Rule relationships (DAG) |
| `evaluations` | Evaluation results |
| `proposals` | Governance proposals |
| `agents` | Registered edge agents |
| `users` | User accounts |
| `audit_events` | Audit log events |

### Redis (Cache & Message Broker)

Handles ephemeral and real-time data:

| Usage | Purpose |
|-------|---------|
| Rule cache | Compiled bytecode cache |
| Result cache | Recent evaluation results |
| Session store | User session data |
| Rate limiting | Request counters |
| Message queue | Async task distribution |
| Pub/Sub | Real-time notifications |

### Object Storage (S3/MinIO)

Stores large binary objects:

| Bucket | Contents |
|--------|----------|
| `proofs` | Trust proofs |
| `bytecode` | Compiled rule bytecode |
| `exports` | Audit exports |
| `backups` | Database backups |
| `plugins` | Plugin artifacts |

### Time-Series Database (InfluxDB/TimescaleDB)

Stores metrics and events:

| Measurement | Data |
|-------------|------|
| `evaluations` | Evaluation metrics |
| `rule_performance` | Rule execution times |
| `system_health` | System metrics |
| `agent_metrics` | Edge agent data |

---

## External Integrations

### Blockchain Networks

ARKA integrates with multiple blockchains for anchoring:

| Network | Purpose | Method |
|---------|---------|--------|
| Ethereum | Primary anchoring | Smart contract |
| Bitcoin | Secondary anchoring | OP_RETURN |
| Polygon | Cost-effective anchoring | Smart contract |
| Private chains | Enterprise anchoring | Configurable |

### Identity Providers

Supports multiple identity systems:

| Provider | Protocol | Use Case |
|----------|----------|----------|
| OIDC providers | OIDC | User authentication |
| SAML providers | SAML 2.0 | Enterprise SSO |
| DID resolvers | DID | Agent identity |
| X.509 CAs | PKI | Certificate validation |

### Monitoring Systems

Integrates with observability platforms:

| System | Integration |
|--------|-------------|
| Prometheus | Metrics export |
| Grafana | Dashboards |
| Jaeger/Zipkin | Distributed tracing |
| ELK Stack | Log aggregation |
| PagerDuty | Alerting |

---

## Deployment Topologies

### Single-Node (Development)

```
┌─────────────────────────────────┐
│          Single Server          │
│  ┌───────────────────────────┐  │
│  │      All Services         │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ │  │
│  │  │Core │ │RTVM │ │ DB  │ │  │
│  │  └─────┘ └─────┘ └─────┘ │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### Multi-Node (Production)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Load Balancer                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│   Core Node   │   │   Core Node   │   │   Core Node   │
│   (Primary)   │   │  (Secondary)  │   │  (Secondary)  │
└───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                            ▼
                ┌───────────────────┐
                │   RTVM Cluster    │
                │  ┌─────┐ ┌─────┐  │
                │  │RTVM1│ │RTVM2│  │
                │  └─────┘ └─────┘  │
                │  ┌─────┐ ┌─────┐  │
                │  │RTVM3│ │RTVM4│  │
                │  └─────┘ └─────┘  │
                └───────────────────┘
                            │
                            ▼
        ┌───────────────────┴───────────────────┐
        │                                       │
        ▼                                       ▼
┌───────────────┐                       ┌───────────────┐
│   PostgreSQL  │                       │     Redis     │
│   (Primary +  │                       │   (Cluster)   │
│   Replicas)   │                       │               │
└───────────────┘                       └───────────────┘
```

### Distributed (Global)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Global Load Balancer                           │
└─────────────────────────────────┬───────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│  US Region    │         │  EU Region    │         │  APAC Region  │
│               │         │               │         │               │
│ ┌───────────┐ │         │ ┌───────────┐ │         │ ┌───────────┐ │
│ │Control    │ │◀───────▶│ │Control    │ │◀───────▶│ │Control    │ │
│ │Plane      │ │  Sync   │ │Plane      │ │  Sync   │ │Plane      │ │
│ └───────────┘ │         │ └───────────┘ │         │ └───────────┘ │
│       │       │         │       │       │         │       │       │
│       ▼       │         │       ▼       │         │       ▼       │
│ ┌───────────┐ │         │ ┌───────────┐ │         │ ┌───────────┐ │
│ │Edge Agents│ │         │ │Edge Agents│ │         │ │Edge Agents│ │
│ └───────────┘ │         │ └───────────┘ │         │ └───────────┘ │
└───────────────┘         └───────────────┘         └───────────────┘
```

---

## Next Steps

- [Data Flow](./data-flow.md) - Understand how data moves through the system
- [Component Interactions](./component-interactions.md) - Deep dive into component communication
- [RTVM Architecture](../components/rtvm.md) - Details of the execution engine
