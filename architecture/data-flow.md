# Data Flow

This document describes how data flows through the ARKA Engine during key operations.

## Table of Contents

- [Rule Creation Flow](#rule-creation-flow)
- [Compliance Evaluation Flow](#compliance-evaluation-flow)
- [Trust Proof Flow](#trust-proof-flow)
- [Rule Synchronization Flow](#rule-synchronization-flow)
- [Governance Workflow Flow](#governance-workflow-flow)
- [Blockchain Anchoring Flow](#blockchain-anchoring-flow)

---

## Rule Creation Flow

When a new compliance rule is created:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         RULE CREATION FLOW                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Client submits rule                                                     │
│  ┌──────────┐                                                               │
│  │  Client  │                                                               │
│  │          │──────────────────────────────────────────────────────────┐    │
│  └──────────┘                                                          │    │
│       │                                                                │    │
│       │ POST /rules                                                    │    │
│       │ {rule_dsl, metadata}                                           │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │   API    │  2. Validate request                                     │    │
│  │ Gateway  │     - Auth check                                         │    │
│  │          │     - Rate limit                                         │    │
│  └────┬─────┘     - Schema validation                                  │    │
│       │                                                                │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │  Rule    │  3. Process rule                                         │    │
│  │ Service  │     - Parse DSL                                          │    │
│  │          │     - Validate semantics                                 │    │
│  └────┬─────┘     - Check dependencies                                 │    │
│       │                                                                │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │ Compiler │  4. Compile to bytecode                                  │    │
│  │          │     - Generate IR                                        │    │
│  │          │     - Optimize                                           │    │
│  └────┬─────┘     - Emit bytecode                                      │    │
│       │                                                                │    │
│       │           5. Compute hashes                                    │    │
│       │              - bytecode_hash = SHA256(bytecode)                │    │
│       │              - rule_hash = SHA256(canonical(rule))             │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │PostgreSQL│  6. Store rule                                           │    │
│  │          │     - rules table                                        │    │
│  │          │     - rule_versions table                                │    │
│  └────┬─────┘     - rule_graph table                                   │    │
│       │                                                                │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │  Redis   │  7. Update caches                                        │    │
│  │          │     - Bytecode cache                                     │    │
│  │          │     - Rule metadata cache                                │    │
│  └────┬─────┘     - Invalidate affected caches                         │    │
│       │                                                                │    │
│       ▼                                                                │    │
│  ┌──────────┐                                                          │    │
│  │   S3     │  8. Store artifacts                                      │    │
│  │          │     - Bytecode blob                                      │    │
│  │          │     - Source archive                                     │    │
│  └────┬─────┘                                                          │    │
│       │                                                                │    │
│       │◀───────────────────────────────────────────────────────────────┘    │
│       │  9. Return response                                                 │
│       │     {rule_id, version, bytecode_hash}                               │
│       ▼                                                                     │
│  ┌──────────┐                                                               │
│  │  Client  │                                                               │
│  └──────────┘                                                               │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Data Transformations

| Stage | Input | Output |
|-------|-------|--------|
| Parse | Rule DSL (YAML/JSON) | Abstract Syntax Tree |
| Validate | AST | Validated AST |
| Compile | Validated AST | Intermediate Representation |
| Optimize | IR | Optimized IR |
| Emit | Optimized IR | Bytecode |

---

## Compliance Evaluation Flow

When a compliance evaluation is requested:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      COMPLIANCE EVALUATION FLOW                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐  1. Submit evaluation request                                 │
│  │  Client  │     {context, input_data}                                     │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       │ POST /evaluate                                                      │
│       ▼                                                                     │
│  ┌──────────┐  2. Authenticate & route                                      │
│  │   API    │                                                               │
│  │ Gateway  │                                                               │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐  3. Prepare evaluation                                        │
│  │Evaluation│     a. Determine applicable rules                             │
│  │ Service  │     b. Check cache for recent identical eval                  │
│  │          │     c. Resolve rule dependencies                              │
│  └────┬─────┘     d. Load rule bytecode                                     │
│       │                                                                     │
│       │           ┌─────────────────────────────────────┐                   │
│       │           │  Rule Resolution                     │                   │
│       │           │                                      │                   │
│       │           │  context.jurisdiction = "US"         │                   │
│       │           │  context.domain = "AML"              │                   │
│       │           │           │                          │                   │
│       │           │           ▼                          │                   │
│       │           │  Query rule_graph for:               │                   │
│       │           │  - jurisdiction = "US" OR "ALL"      │                   │
│       │           │  - domain = "AML"                    │                   │
│       │           │  - effective_date <= now             │                   │
│       │           │  - (expiry_date IS NULL OR > now)    │                   │
│       │           │           │                          │                   │
│       │           │           ▼                          │                   │
│       │           │  Topological sort by dependencies    │                   │
│       │           │  [rule_1, rule_2, rule_3]            │                   │
│       │           └─────────────────────────────────────┘                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐  4. Execute in RTVM                                           │
│  │   RTVM   │                                                               │
│  │          │     ┌─────────────────────────────────────┐                   │
│  │          │     │  For each rule in evaluation_order: │                   │
│  │          │     │                                      │                   │
│  │          │     │  a. Create execution context         │                   │
│  │          │     │  b. Push input onto stack            │                   │
│  │          │     │  c. Execute bytecode instructions    │                   │
│  │          │     │  d. Record execution trace           │                   │
│  │          │     │  e. Collect result from stack        │                   │
│  │          │     │                                      │                   │
│  │          │     │  Execution is:                       │                   │
│  │          │     │  - Isolated (no external I/O)        │                   │
│  │          │     │  - Deterministic                     │                   │
│  │          │     │  - Resource-limited                  │                   │
│  │          │     └─────────────────────────────────────┘                   │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       │  5. Aggregate results                                               │
│       │     - Merge flags from all rules                                    │
│       │     - Compute overall decision                                      │
│       │     - Collect evidence                                              │
│       ▼                                                                     │
│  ┌──────────┐  6. Generate trust proof                                      │
│  │  Proof   │                                                               │
│  │Generator │     input_hash = SHA256(canonical(input))                     │
│  │          │     rules_hash = MerkleRoot(rule_hashes)                      │
│  │          │     trace_hash = SHA256(execution_trace)                      │
│  │          │     output_hash = SHA256(canonical(result))                   │
│  │          │                                                               │
│  │          │     proof = Sign(input_hash || rules_hash ||                  │
│  │          │                  trace_hash || output_hash)                   │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐  7. Persist evaluation                                        │
│  │PostgreSQL│     - evaluations table                                       │
│  │          │     - evaluation_results table                                │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐  8. Cache result                                              │
│  │  Redis   │     - TTL based on cache policy                               │
│  │          │     - Key = hash(context + input)                             │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐  9. Queue proof for anchoring (async)                         │
│  │  Queue   │                                                               │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       │◀────────────────────────────────────────────────────────────────    │
│       │  10. Return response                                                │
│       │      {evaluation_id, result, trust_proof}                           │
│       ▼                                                                     │
│  ┌──────────┐                                                               │
│  │  Client  │                                                               │
│  └──────────┘                                                               │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Evaluation Data Model

```typescript
// Request
interface EvaluationRequest {
  context: {
    jurisdiction: string;
    domain: string;
    timestamp?: string;      // For historical evaluation
    entityId?: string;       // Entity being evaluated
  };
  input: Record<string, any>; // Domain-specific input
  options?: {
    skipCache?: boolean;
    includeTrace?: boolean;
    timeout?: number;
  };
}

// Response
interface EvaluationResponse {
  evaluationId: string;
  timestamp: string;
  result: {
    decision: 'compliant' | 'non_compliant' | 'needs_review';
    flags: Flag[];
    annotations: Record<string, any>;
    evidence: Evidence[];
  };
  metadata: {
    rulesEvaluated: string[];
    ruleVersions: Record<string, string>;
    evaluationDurationMs: number;
    cached: boolean;
  };
  trustProof: TrustProof;
}
```

---

## Trust Proof Flow

How trust proofs are generated, stored, and verified:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         TRUST PROOF FLOW                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GENERATION (during evaluation)                                             │
│  ──────────────────────────────                                             │
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │   Input     │    │   Rules     │    │  Execution  │                     │
│  │   Data      │    │   Used      │    │   Trace     │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         │                  │                  │                             │
│         ▼                  ▼                  ▼                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │ Canonicalize│    │Build Merkle │    │   Hash      │                     │
│  │  & Hash     │    │   Tree      │    │   Trace     │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         │                  │                  │                             │
│         │ input_hash       │ rules_root       │ trace_hash                  │
│         │                  │                  │                             │
│         └──────────────────┼──────────────────┘                             │
│                            │                                                │
│                            ▼                                                │
│                    ┌─────────────┐                                          │
│                    │   Output    │                                          │
│                    │   Result    │                                          │
│                    └──────┬──────┘                                          │
│                           │                                                 │
│                           ▼                                                 │
│                    ┌─────────────┐                                          │
│                    │   Hash      │                                          │
│                    │   Output    │                                          │
│                    └──────┬──────┘                                          │
│                           │ output_hash                                     │
│                           │                                                 │
│                           ▼                                                 │
│         ┌─────────────────────────────────────┐                            │
│         │         PROOF CONTENT                │                            │
│         │                                      │                            │
│         │  {                                   │                            │
│         │    input_hash,                       │                            │
│         │    rules_root,                       │                            │
│         │    rule_proofs: [...],               │                            │
│         │    trace_hash,                       │                            │
│         │    output_hash,                      │                            │
│         │    timestamp,                        │                            │
│         │    agent_id                          │                            │
│         │  }                                   │                            │
│         └──────────────────┬──────────────────┘                            │
│                            │                                                │
│                            ▼                                                │
│                    ┌─────────────┐                                          │
│                    │    Sign     │                                          │
│                    │  (Agent Key)│                                          │
│                    └──────┬──────┘                                          │
│                           │                                                 │
│                           ▼                                                 │
│         ┌─────────────────────────────────────┐                            │
│         │         TRUST PROOF                  │                            │
│         │                                      │                            │
│         │  {                                   │                            │
│         │    proof_id,                         │                            │
│         │    content: {...},                   │                            │
│         │    signature,                        │                            │
│         │    certificate_chain                 │                            │
│         │  }                                   │                            │
│         └─────────────────────────────────────┘                            │
│                                                                             │
│                                                                             │
│  VERIFICATION                                                               │
│  ────────────                                                               │
│                                                                             │
│  ┌──────────┐                                                               │
│  │  Proof   │                                                               │
│  │  Input   │                                                               │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                    VERIFICATION STEPS                         │          │
│  │                                                               │          │
│  │  1. Verify signature                                          │          │
│  │     - Check signature against public key                      │          │
│  │     - Validate certificate chain                              │          │
│  │                                                               │          │
│  │  2. Verify agent identity                                     │          │
│  │     - Lookup agent in registry                                │          │
│  │     - Check agent is authorized                               │          │
│  │                                                               │          │
│  │  3. Verify rule commitments                                   │          │
│  │     - Check rules_root matches known version                  │          │
│  │     - Verify individual rule Merkle proofs                    │          │
│  │                                                               │          │
│  │  4. (Optional) Verify input/output                            │          │
│  │     - Recompute hashes from original data                     │          │
│  │     - Compare against proof commitments                       │          │
│  │                                                               │          │
│  │  5. (Optional) Replay execution                               │          │
│  │     - Re-execute rules with same input                        │          │
│  │     - Verify output matches                                   │          │
│  │                                                               │          │
│  └──────────────────────────────────────────────────────────────┘          │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────┐                                                               │
│  │  Result  │  VALID | INVALID (with reason)                               │
│  └──────────┘                                                               │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Rule Synchronization Flow

How rules are distributed to edge agents:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      RULE SYNCHRONIZATION FLOW                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONTROL PLANE                              EDGE AGENT                      │
│  ─────────────                              ──────────                      │
│                                                                             │
│  1. New rule version published                                              │
│  ┌──────────────┐                                                           │
│  │ Rule Service │                                                           │
│  │   v1.0 → v1.1│                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐  2. Compute delta                                         │
│  │    Delta     │     - Diff v1.0 and v1.1                                  │
│  │   Builder    │     - Create delta package                                │
│  └──────┬───────┘     - Sign package                                        │
│         │                                                                   │
│         │  delta_package = {                                                │
│         │    from_version: "1.0.0",                                         │
│         │    to_version: "1.1.0",                                           │
│         │    changes: [...],                                                │
│         │    merkle_root: "...",                                            │
│         │    signature: "..."                                               │
│         │  }                                                                │
│         │                                                                   │
│         ▼                                                                   │
│  ┌──────────────┐  3. Notify agents                                         │
│  │ Distribution │                                                           │
│  │   Service    │────────────────────────────────────▶  ┌──────────────┐   │
│  └──────────────┘      VERSION_AVAILABLE                │    Agent     │   │
│                        {version: "1.1.0"}               │  Listener    │   │
│                                                         └──────┬───────┘   │
│                                                                │            │
│                        4. Request delta                        │            │
│  ┌──────────────┐◀─────────────────────────────────────────────┘            │
│  │ Distribution │      GET /sync/delta?from=1.0.0&to=1.1.0                  │
│  │   Service    │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         │          5. Send delta package                                    │
│         │─────────────────────────────────────────────▶  ┌──────────────┐   │
│         │                                                │    Agent     │   │
│                                                          │ Synchronizer │   │
│                                                          └──────┬───────┘   │
│                                                                 │            │
│                                                          6. Verify package  │
│                                                          ┌──────▼───────┐   │
│                                                          │   Verify     │   │
│                                                          │  - Signature │   │
│                                                          │  - Chain     │   │
│                                                          │  - Integrity │   │
│                                                          └──────┬───────┘   │
│                                                                 │            │
│                                                          7. Apply changes   │
│                                                          ┌──────▼───────┐   │
│                                                          │ Local Cache  │   │
│                                                          │   Update     │   │
│                                                          │  (atomic)    │   │
│                                                          └──────┬───────┘   │
│                                                                 │            │
│                        8. Acknowledge                           │            │
│  ┌──────────────┐◀──────────────────────────────────────────────┘            │
│  │    Agent     │      ACK {version: "1.1.0", status: "applied"}            │
│  │   Registry   │                                                           │
│  └──────────────┘                                                           │
│                                                                             │
│                                                                             │
│  FAILURE HANDLING                                                           │
│  ────────────────                                                           │
│                                                                             │
│  If verification fails:                                                     │
│    - Agent requests full sync instead of delta                              │
│    - Control plane notified of sync failure                                 │
│                                                                             │
│  If apply fails:                                                            │
│    - Rollback to previous version                                           │
│    - Report error to control plane                                          │
│    - Retry with exponential backoff                                         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Governance Workflow Flow

How rule changes go through governance:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      GOVERNANCE WORKFLOW FLOW                               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐  1. Create proposal                                           │
│  │ Proposer │     {type: MODIFY, rule_id, changes, rationale}              │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         PROPOSAL CREATED                              │  │
│  │                                                                       │  │
│  │  proposal_id: "prop_123"                                              │  │
│  │  status: DRAFT                                                        │  │
│  │  type: MODIFY                                                         │  │
│  │  target_rule: "rule_abc"                                              │  │
│  │  changes: {threshold: 10000 → 15000}                                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       │  2. Submit for review                                               │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     TECHNICAL REVIEW STAGE                            │  │
│  │                                                                       │  │
│  │  Automated checks:                                                    │  │
│  │  ├── Syntax validation ✓                                              │  │
│  │  ├── Semantic validation ✓                                            │  │
│  │  ├── Dependency check ✓                                               │  │
│  │  └── Security scan ✓                                                  │  │
│  │                                                                       │  │
│  │  Simulation results:                                                  │  │
│  │  ├── Historical replay: 100,000 cases                                 │  │
│  │  ├── Outcome changes: 2,340 (2.3%)                                    │  │
│  │  ├── False positive delta: -1.2%                                      │  │
│  │  └── Performance impact: +0.3ms avg                                   │  │
│  │                                                                       │  │
│  │  Reviewers: [tech_reviewer_1, tech_reviewer_2]                        │  │
│  │  Votes: [APPROVE, APPROVE]                                            │  │
│  │  Status: APPROVED                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     COMPLIANCE REVIEW STAGE                           │  │
│  │                                                                       │  │
│  │  Semantic diff:                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │  THRESHOLD CHANGED                                              │  │  │
│  │  │  Field: transaction.amount threshold                            │  │  │
│  │  │  Before: $10,000                                                │  │  │
│  │  │  After: $15,000                                                 │  │  │
│  │  │                                                                 │  │  │
│  │  │  IMPACT: Fewer transactions will be flagged.                    │  │  │
│  │  │  Estimated 15% reduction in CTR filings.                        │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  │  Regulatory alignment check:                                          │  │
│  │  ├── US BSA/AML: ⚠ Warning - CTR threshold is $10,000                │  │
│  │  └── Recommendation: Verify regulatory basis for change              │  │
│  │                                                                       │  │
│  │  Reviewers: [compliance_officer_1]                                    │  │
│  │  Votes: [REQUEST_CHANGES]                                             │  │
│  │  Comments: "Please provide regulatory justification"                  │  │
│  │  Status: CHANGES_REQUESTED                                            │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       │  3. Proposer updates with justification                             │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     COMPLIANCE REVIEW (Retry)                         │  │
│  │                                                                       │  │
│  │  Additional context provided:                                         │  │
│  │  "This rule is for internal risk scoring, not CTR filing.            │  │
│  │   CTR threshold rule (rule_xyz) remains unchanged at $10,000."       │  │
│  │                                                                       │  │
│  │  Reviewers: [compliance_officer_1]                                    │  │
│  │  Votes: [APPROVE]                                                     │  │
│  │  Status: APPROVED                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      BUSINESS REVIEW STAGE                            │  │
│  │                                                                       │  │
│  │  Impact assessment:                                                   │  │
│  │  ├── Affected business units: [Retail, Commercial]                   │  │
│  │  ├── Operational impact: Low                                         │  │
│  │  └── Training required: No                                           │  │
│  │                                                                       │  │
│  │  Reviewers: [business_owner_1]                                        │  │
│  │  Votes: [APPROVE]                                                     │  │
│  │  Status: APPROVED                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      DEPLOYMENT STAGE                                 │  │
│  │                                                                       │  │
│  │  Deployment plan:                                                     │  │
│  │  ├── Strategy: Rolling (10% → 50% → 100%)                            │  │
│  │  ├── Rollback trigger: Error rate > 1%                               │  │
│  │  └── Monitoring: Enhanced for 24h                                    │  │
│  │                                                                       │  │
│  │  Execution:                                                           │  │
│  │  ├── 10% rollout: ✓ Complete, metrics normal                         │  │
│  │  ├── 50% rollout: ✓ Complete, metrics normal                         │  │
│  │  └── 100% rollout: ✓ Complete                                        │  │
│  │                                                                       │  │
│  │  Status: DEPLOYED                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         COMPLETE                                      │  │
│  │                                                                       │  │
│  │  Rule "rule_abc" updated from v1.0.0 to v1.1.0                        │  │
│  │  Proposal "prop_123" closed                                           │  │
│  │  Audit trail recorded                                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Blockchain Anchoring Flow

How trust proofs are anchored to blockchains:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      BLOCKCHAIN ANCHORING FLOW                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Proofs accumulate in queue                                              │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                     PROOF QUEUE                               │          │
│  │  [proof_1] [proof_2] [proof_3] ... [proof_n]                 │          │
│  └──────────────────────────────────────────────────────────────┘          │
│       │                                                                     │
│       │  2. Batch trigger (time or count threshold)                         │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                   MERKLE TREE BUILDER                         │          │
│  │                                                               │          │
│  │         Build Merkle tree from proof hashes:                  │          │
│  │                                                               │          │
│  │                      Root: H(H12 || H34)                      │          │
│  │                           /          \                        │          │
│  │                    H12: H(H1||H2)   H34: H(H3||H4)           │          │
│  │                     /      \         /      \                 │          │
│  │                   H1       H2       H3       H4               │          │
│  │                   │        │        │        │                │          │
│  │                proof_1  proof_2  proof_3  proof_4             │          │
│  │                                                               │          │
│  │  merkle_root = "0xabc123..."                                  │          │
│  └──────────────────────────────────────────────────────────────┘          │
│       │                                                                     │
│       │  3. Select anchoring strategy                                       │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                  STRATEGY SELECTOR                            │          │
│  │                                                               │          │
│  │  Configured strategy: REDUNDANT                               │          │
│  │  Target networks: [Ethereum, Polygon]                         │          │
│  │                                                               │          │
│  │  Cost estimation:                                             │          │
│  │  ├── Ethereum: ~$15 (50 gwei)                                │          │
│  │  └── Polygon: ~$0.01                                         │          │
│  └──────────────────────────────────────────────────────────────┘          │
│       │                                                                     │
│       │  4. Submit to blockchains (parallel)                                │
│       ▼                                                                     │
│  ┌───────────────────────┐    ┌───────────────────────┐                    │
│  │   ETHEREUM ADAPTER    │    │   POLYGON ADAPTER     │                    │
│  │                       │    │                       │                    │
│  │  Contract call:       │    │  Contract call:       │                    │
│  │  anchor(merkle_root)  │    │  anchor(merkle_root)  │                    │
│  │                       │    │                       │                    │
│  │  tx_hash: 0xdef...    │    │  tx_hash: 0x789...    │                    │
│  └───────────┬───────────┘    └───────────┬───────────┘                    │
│              │                            │                                 │
│              │  5. Wait for confirmations │                                 │
│              ▼                            ▼                                 │
│  ┌───────────────────────┐    ┌───────────────────────┐                    │
│  │   12 confirmations    │    │   128 confirmations   │                    │
│  │   Block: 18,234,567   │    │   Block: 51,234,890   │                    │
│  │   Timestamp: ...      │    │   Timestamp: ...      │                    │
│  └───────────┬───────────┘    └───────────┬───────────┘                    │
│              │                            │                                 │
│              └────────────┬───────────────┘                                 │
│                           │                                                 │
│                           ▼                                                 │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                   ANCHOR RECORD                               │          │
│  │                                                               │          │
│  │  {                                                            │          │
│  │    anchor_id: "anchor_xyz",                                   │          │
│  │    merkle_root: "0xabc123...",                                │          │
│  │    proof_count: 1000,                                         │          │
│  │    timestamp: "2024-01-15T12:00:00Z",                         │          │
│  │    chains: [                                                  │          │
│  │      {                                                        │          │
│  │        network: "ethereum",                                   │          │
│  │        tx_hash: "0xdef...",                                   │          │
│  │        block_number: 18234567,                                │          │
│  │        confirmations: 12                                      │          │
│  │      },                                                       │          │
│  │      {                                                        │          │
│  │        network: "polygon",                                    │          │
│  │        tx_hash: "0x789...",                                   │          │
│  │        block_number: 51234890,                                │          │
│  │        confirmations: 128                                     │          │
│  │      }                                                        │          │
│  │    ]                                                          │          │
│  │  }                                                            │          │
│  └──────────────────────────────────────────────────────────────┘          │
│       │                                                                     │
│       │  6. Update proof records with anchor reference                      │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │                   UPDATE PROOFS                               │          │
│  │                                                               │          │
│  │  For each proof in batch:                                     │          │
│  │    - Store Merkle proof path                                  │          │
│  │    - Link to anchor record                                    │          │
│  │    - Update status: ANCHORED                                  │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                             │
│                                                                             │
│  VERIFICATION (later)                                                       │
│  ────────────────────                                                       │
│                                                                             │
│  ┌──────────┐                                                               │
│  │  Proof   │                                                               │
│  │  Query   │                                                               │
│  └────┬─────┘                                                               │
│       │                                                                     │
│       ▼                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │  1. Get proof's Merkle path from anchor record                │          │
│  │  2. Recompute root from proof hash + path                     │          │
│  │  3. Query blockchain for anchored root                        │          │
│  │  4. Compare computed root with anchored root                  │          │
│  │  5. Return: VERIFIED or INVALID                               │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

- [Component Interactions](./component-interactions.md) - How components communicate
- [System Overview](./system-overview.md) - Architecture overview
- [API Reference](../api-reference/rest-api.md) - Detailed API documentation
