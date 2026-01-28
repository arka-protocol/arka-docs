# Concepts & Terminology

This document introduces the key concepts and terminology used throughout the ARKA Engine documentation and system.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Rule System](#rule-system)
- [Execution Model](#execution-model)
- [Trust & Verification](#trust--verification)
- [Governance](#governance)
- [Distribution](#distribution)

---

## Core Concepts

### ARKA Engine

**PACT** (Protocol for Auditable Compliance Technology) is a comprehensive regulatory compliance platform that provides:

- **Deterministic Execution**: Identical inputs always produce identical outputs
- **Cryptographic Verification**: Every decision can be mathematically proven
- **Temporal Consistency**: Evaluate compliance at any point in time
- **Jurisdictional Awareness**: Handle multi-jurisdiction regulatory requirements

### Compliance Evaluation

A **Compliance Evaluation** is the process of determining whether a given input (transaction, entity, document) satisfies applicable regulatory rules.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Input     │────▶│   ARKA      │────▶│  Decision   │
│   Data      │     │   Engine    │     │  + Proof    │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Trust Proof

A **Trust Proof** is a cryptographic attestation that binds:
- The input data (what was evaluated)
- The rules used (which version, which rules)
- The execution trace (how it was evaluated)
- The output decision (what was decided)

Trust proofs enable anyone to verify that a compliance decision was made correctly without re-executing the entire evaluation.

---

## Rule System

### Rule

A **Rule** is the fundamental unit of compliance logic. Rules define:

- **Conditions**: What must be true for the rule to apply
- **Actions**: What happens when conditions are met
- **Metadata**: Jurisdiction, domain, effective dates, version

Example rule structure:
```yaml
rule:
  name: "high-value-transaction-flag"
  jurisdiction: "US"
  domain: "AML"
  conditions:
    - field: "transaction.amount"
      operator: "greater_than"
      value: 10000
  actions:
    - type: "flag"
      severity: "high"
```

### Rule DSL (Domain-Specific Language)

ARKA provides a **Rule DSL** for expressing compliance logic. The DSL supports:

- Conditional expressions
- Threshold comparisons
- Date/time operations
- Entity relationship traversal
- Aggregate calculations

### Compiled Rule

Rules written in the DSL are **compiled** into RTVM bytecode for execution. Compilation:
- Validates rule syntax and semantics
- Optimizes rule logic
- Generates deterministic bytecode
- Produces a content hash for verification

### Rule Graph

The **Rule Graph** is a directed acyclic graph (DAG) that represents relationships between rules:

```
          ┌─────────────┐
          │  Base Rule  │
          │  (Generic)  │
          └──────┬──────┘
                 │
        ┌────────┴────────┐
        │                 │
  ┌─────▼─────┐     ┌─────▼─────┐
  │  US Rule  │     │  EU Rule  │
  │(Specific) │     │(Specific) │
  └───────────┘     └───────────┘
```

Edge types in the rule graph:
- **depends_on**: Rule requires another rule's evaluation first
- **specializes**: Rule is a jurisdiction-specific version
- **conflicts**: Rules cannot both apply
- **supersedes**: Rule replaces another rule

### Rule Version

Every rule change creates a new **version**. ARKA uses semantic versioning:
- **Major**: Breaking changes to rule logic
- **Minor**: New conditions or actions
- **Patch**: Bug fixes, clarifications

All versions are immutable and retained for historical evaluation.

---

## Execution Model

### RTVM (Regulatory Technology Virtual Machine)

The **RTVM** is the execution engine that evaluates compiled rules. Key properties:

- **Deterministic**: Same inputs always produce same outputs
- **Isolated**: Each evaluation runs in isolation
- **Auditable**: Complete execution trace captured
- **Efficient**: Optimized bytecode execution

### Bytecode

**Bytecode** is the low-level instruction format executed by the RTVM. Bytecode categories:

| Category | Purpose | Examples |
|----------|---------|----------|
| Stack | Value manipulation | PUSH, POP, DUP |
| Arithmetic | Calculations | ADD_DEC, MUL_DEC |
| Temporal | Date operations | DATE_CMP, DATE_ADD |
| Jurisdictional | Context access | JUR_LOAD, JUR_PRECEDENCE |
| Rule Graph | Rule traversal | RULE_LOAD, RULE_DEPS |
| Control | Flow control | JMP, CALL, RET |
| Evidence | Audit trail | EVIDENCE_ADD, ATTEST |

### Execution Context

The **Execution Context** provides environmental information to rule execution:

```typescript
interface ExecutionContext {
  timestamp: Timestamp;        // When evaluation occurs
  jurisdiction: Jurisdiction;  // Primary jurisdiction
  domain: Domain;              // Regulatory domain
  entityContext: EntityContext; // Entity being evaluated
  ruleVersion: Version;        // Rule version to use
}
```

### Evaluation Result

An **Evaluation Result** contains:

```typescript
interface EvaluationResult {
  decision: Decision;          // COMPLIANT, NON_COMPLIANT, NEEDS_REVIEW
  flags: Flag[];               // Triggered conditions
  evidence: Evidence[];        // Supporting information
  rulesEvaluated: RuleRef[];   // Which rules ran
  executionTrace: Trace;       // Detailed execution log
  trustProof: TrustProof;      // Cryptographic proof
}
```

---

## Trust & Verification

### Merkle Tree

ARKA uses **Merkle Trees** to efficiently verify large sets of data:

```
              Root Hash
              /       \
         Hash AB      Hash CD
         /    \       /    \
      Hash A  Hash B  Hash C  Hash D
        |       |       |       |
      Rule A  Rule B  Rule C  Rule D
```

Properties:
- Any change modifies the root hash
- Individual items verified with O(log n) proof
- Enables efficient synchronization

### Blockchain Anchoring

**Blockchain Anchoring** records Merkle roots on public blockchains:

- Creates immutable timestamp proof
- Enables third-party verification
- Supports multiple chains (Ethereum, Bitcoin, Polygon)

### Attestation

An **Attestation** is a cryptographic signature binding a statement to an identity:

```typescript
interface Attestation {
  statement: Hash;           // What is being attested
  signer: DID;               // Who is attesting
  signature: Signature;      // Cryptographic signature
  timestamp: Timestamp;      // When attested
}
```

### Decentralized Identifier (DID)

A **DID** is a globally unique identifier for entities in ARKA:

```
did:arka:org:acme-corp:compliance-system-1
    │    │   │         │
    │    │   │         └── Specific identifier
    │    │   └── Organization
    │    └── DID method
    └── DID scheme
```

DIDs enable:
- Self-sovereign identity
- Cryptographic authentication
- Verifiable credentials

---

## Governance

### Rule Proposal

A **Rule Proposal** is a suggested change to the rule set:

```typescript
interface RuleProposal {
  proposalId: UUID;
  type: ADD | MODIFY | DELETE;
  targetRule: RuleRef;
  proposedChanges: RuleDiff;
  rationale: string;
  evidence: Evidence[];
  proposer: Identity;
}
```

### Governance Workflow

The **Governance Workflow** is the approval process for rule changes:

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Proposal │──▶│Technical │──▶│Compliance│──▶│ Business │──▶│ Deployed │
│ Submit   │   │ Review   │   │ Review   │   │ Approval │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### Simulation Sandbox

The **Simulation Sandbox** tests rule changes before deployment:

- Historical replay against past data
- Synthetic scenario generation
- Impact analysis
- Performance benchmarking

### Semantic Diff

A **Semantic Diff** explains rule changes in human-readable terms:

```
Rule: high-value-transaction-flag v1.0.0 → v1.1.0

Changes:
- Threshold INCREASED: $10,000 → $15,000
- New condition ADDED: transaction.type != "internal"
- Jurisdiction EXPANDED: US → US, CA

Impact:
- Estimated 15% fewer flags
- Affects 50,000 monthly transactions
```

---

## Distribution

### Edge Agent

An **Edge Agent** is a lightweight ARKA node deployed at network edges:

```
┌─────────────────────────────────────────┐
│              Edge Agent                  │
│  ┌───────────┐  ┌───────────┐           │
│  │Local Rules│  │Local Cache│           │
│  └───────────┘  └───────────┘           │
│  ┌───────────┐  ┌───────────┐           │
│  │ Executor  │  │Proof Gen  │           │
│  └───────────┘  └───────────┘           │
└─────────────────────────────────────────┘
```

Edge agents provide:
- Low-latency local evaluation
- Offline operation capability
- Reduced central system load

### Rule Synchronization

**Rule Synchronization** keeps edge agents updated:

1. Control plane publishes new rule version
2. Agents receive update notification
3. Agents download delta package
4. Agents verify integrity
5. Agents atomically apply update

### Offline Mode

**Offline Mode** allows edge agents to operate during network partitions:

- Cached rules continue operating
- Proofs stored locally
- Reconciliation on reconnection
- Configurable offline policies

### Proof Aggregation

**Proof Aggregation** combines multiple trust proofs for efficiency:

```
Individual Proofs:        Aggregate Proof:
┌───────┐                 ┌─────────────────┐
│Proof 1│──┐              │ Period: 1 hour  │
├───────┤  │              │ Count: 10,000   │
│Proof 2│──┼──────────────│ Merkle Root: x  │
├───────┤  │              │ Signature: y    │
│Proof N│──┘              └─────────────────┘
└───────┘
```

---

## Glossary

| Term | Definition |
|------|------------|
| **Attestation** | Cryptographic signature binding a statement to an identity |
| **Bytecode** | Low-level instructions executed by the RTVM |
| **Compliance Evaluation** | Process of checking input against regulatory rules |
| **DID** | Decentralized Identifier for entities |
| **Edge Agent** | Lightweight ARKA node at network edge |
| **Governance Workflow** | Approval process for rule changes |
| **Merkle Tree** | Hash tree for efficient verification |
| **Rule** | Unit of compliance logic |
| **Rule DSL** | Domain-specific language for rules |
| **Rule Graph** | DAG of rule relationships |
| **RTVM** | Regulatory Technology Virtual Machine |
| **Semantic Diff** | Human-readable rule change explanation |
| **Trust Proof** | Cryptographic proof of correct execution |

---

## Next Steps

- [Your First Rule](./first-rule.md) - Create your first compliance rule
- [System Overview](../architecture/system-overview.md) - Understand the architecture
- [Rule Authoring Guide](../guides/rule-authoring.md) - Deep dive into rule creation
