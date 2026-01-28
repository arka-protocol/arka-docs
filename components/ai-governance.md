# AI Governance Module

The AI Governance module provides intelligent rule proposal, simulation, and approval workflows for managing regulatory rule changes in the ARKA Engine.

## Overview

The AI Governance module addresses a critical challenge in regulatory compliance: how to safely and efficiently update rules in response to changing regulations while maintaining system integrity and auditability.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AI Governance Module                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐               │
│  │   Proposal   │───▶│  Simulation  │───▶│   Approval   │               │
│  │    Engine    │    │   Sandbox    │    │   Workflow   │               │
│  └──────────────┘    └──────────────┘    └──────────────┘               │
│         │                   │                   │                        │
│         ▼                   ▼                   ▼                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐               │
│  │   Semantic   │    │   Impact     │    │   Version    │               │
│  │    Diff      │    │   Analysis   │    │   Control    │               │
│  └──────────────┘    └──────────────┘    └──────────────┘               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### Proposal Engine

The Proposal Engine analyzes regulatory changes and generates rule proposals.

#### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Proposal Engine                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐     ┌─────────────┐     ┌────────────┐ │
│  │  Regulatory │────▶│   NLP       │────▶│  Rule      │ │
│  │  Document   │     │   Parser    │     │  Generator │ │
│  │  Ingestion  │     │             │     │            │ │
│  └─────────────┘     └─────────────┘     └────────────┘ │
│         │                  │                   │         │
│         ▼                  ▼                   ▼         │
│  ┌─────────────┐     ┌─────────────┐     ┌────────────┐ │
│  │  Document   │     │  Semantic   │     │  Proposal  │ │
│  │  Store      │     │  Model      │     │  Queue     │ │
│  └─────────────┘     └─────────────┘     └────────────┘ │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### Proposal Types

| Type | Description | Trigger |
|------|-------------|---------|
| `REGULATORY_UPDATE` | New regulation or amendment | Document ingestion |
| `CONFLICT_RESOLUTION` | Address rule conflicts | Conflict detection |
| `OPTIMIZATION` | Performance improvement | Analytics |
| `GAP_ANALYSIS` | Missing coverage | Compliance audit |
| `DEPRECATION` | Remove obsolete rules | Version review |

#### Creating Proposals

```yaml
# proposal-example.yaml
apiVersion: arka.io/v1
kind: RuleProposal
metadata:
  name: aml-threshold-update-2024
  namespace: compliance
  labels:
    domain: aml
    priority: high
    source: regulatory-update
spec:
  title: "Update AML Transaction Threshold"
  description: |
    Update transaction monitoring thresholds per
    FinCEN guidance 2024-01

  source:
    type: regulatory_document
    reference: "FinCEN-2024-01"
    url: "https://fincen.gov/guidance/2024-01"
    effectiveDate: "2024-07-01"

  changes:
    - type: modify
      target: rule/aml/transaction-threshold
      field: conditions.amount.threshold
      oldValue: 10000
      newValue: 3000
      rationale: "Lower threshold for enhanced monitoring"

    - type: add
      target: rule/aml/aggregation-window
      content: |
        rule aggregation_window {
          condition: aggregate_transactions(
            entity_id,
            window: 24h
          ) > 10000
          action: flag_for_review
        }

  impact:
    estimatedAffectedTransactions: 150000
    riskLevel: medium
    backwardsCompatible: true

  reviewers:
    - compliance-team
    - legal-team

  deadline: "2024-06-15"
```

#### Proposal API

```python
from pact.governance import ProposalEngine, Proposal

# Initialize engine
engine = ProposalEngine(
    model="gpt-4-turbo",
    rule_repository=rule_repo,
    semantic_model=semantic_model
)

# Create proposal from regulatory document
proposal = await engine.analyze_document(
    document_path="/regulations/fincen-2024-01.pdf",
    document_type="regulatory_guidance",
    jurisdictions=["US"],
    domains=["AML", "KYC"]
)

# Review generated changes
for change in proposal.changes:
    print(f"Change: {change.type} -> {change.target}")
    print(f"Rationale: {change.rationale}")
    print(f"Confidence: {change.confidence}")

# Submit for review
proposal_id = await engine.submit(
    proposal,
    priority="high",
    reviewers=["compliance-team"]
)
```

### Simulation Sandbox

The Simulation Sandbox enables safe testing of rule changes before deployment.

#### Sandbox Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Simulation Sandbox                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Isolation Boundary                       │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │ │
│  │  │  Sandboxed   │  │  Synthetic   │  │    Historical    │  │ │
│  │  │    RTVM      │  │  Data Gen    │  │    Replay        │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘  │ │
│  │         │                 │                   │             │ │
│  │         └─────────────────┼───────────────────┘             │ │
│  │                           ▼                                  │ │
│  │                  ┌──────────────┐                           │ │
│  │                  │   Results    │                           │ │
│  │                  │   Collector  │                           │ │
│  │                  └──────────────┘                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Impact     │  │  Regression  │  │    Compliance        │   │
│  │   Analysis   │  │   Detection  │  │    Coverage          │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Simulation Modes

**1. Historical Replay**

Re-evaluate past transactions with proposed rules:

```python
from pact.governance import SimulationSandbox

sandbox = SimulationSandbox(
    rtvm_config={
        "deterministic": True,
        "time_travel_enabled": True
    }
)

# Run historical replay simulation
results = await sandbox.simulate(
    proposal_id="aml-threshold-update-2024",
    mode="historical_replay",
    config={
        "start_date": "2024-01-01",
        "end_date": "2024-03-31",
        "sample_size": 100000,
        "sampling_strategy": "stratified"
    }
)

print(f"Transactions evaluated: {results.total_evaluated}")
print(f"New flags raised: {results.new_flags}")
print(f"Flags removed: {results.removed_flags}")
print(f"False positive estimate: {results.false_positive_rate}")
```

**2. Synthetic Data Generation**

Test edge cases with generated data:

```python
# Configure synthetic data generation
synthetic_config = {
    "generator": "adversarial",
    "scenarios": [
        {
            "name": "structuring_attempt",
            "pattern": "multiple_small_transactions",
            "parameters": {
                "count": 10,
                "amount_range": [2500, 2999],
                "time_window": "4h"
            }
        },
        {
            "name": "high_risk_jurisdiction",
            "pattern": "cross_border_transfer",
            "parameters": {
                "source_country": "US",
                "dest_countries": ["IR", "KP", "SY"],
                "amount_range": [1000, 50000]
            }
        }
    ],
    "volume": 10000
}

results = await sandbox.simulate(
    proposal_id="aml-threshold-update-2024",
    mode="synthetic",
    config=synthetic_config
)
```

**3. A/B Comparison**

Compare current vs proposed rules:

```python
ab_results = await sandbox.compare(
    baseline_rules="production",
    candidate_rules="proposal:aml-threshold-update-2024",
    dataset="historical:2024-Q1",
    metrics=[
        "flag_rate",
        "false_positive_rate",
        "processing_latency",
        "rule_conflicts"
    ]
)

# Generate comparison report
report = ab_results.generate_report()
print(report.summary)

# Detailed metrics
for metric in ab_results.metrics:
    print(f"{metric.name}:")
    print(f"  Baseline: {metric.baseline_value}")
    print(f"  Candidate: {metric.candidate_value}")
    print(f"  Delta: {metric.delta} ({metric.delta_pct}%)")
```

#### Impact Analysis

```python
from pact.governance import ImpactAnalyzer

analyzer = ImpactAnalyzer()

impact = await analyzer.analyze(
    proposal_id="aml-threshold-update-2024",
    simulation_results=results
)

# Review impact assessment
print(f"Risk Score: {impact.risk_score}/100")
print(f"Affected Rules: {len(impact.affected_rules)}")
print(f"Downstream Dependencies: {len(impact.dependencies)}")

# Detailed breakdown
for category in impact.categories:
    print(f"\n{category.name}:")
    print(f"  Severity: {category.severity}")
    print(f"  Affected Items: {category.count}")
    for item in category.items[:5]:
        print(f"    - {item.description}")
```

### Approval Workflow

The Approval Workflow manages multi-stage review and deployment of rule changes.

#### Workflow Stages

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  DRAFT   │───▶│  REVIEW  │───▶│ APPROVAL │───▶│ STAGING  │───▶│PRODUCTION│
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │               │               │
     │               │               │               │               │
     ▼               ▼               ▼               ▼               ▼
 ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
 │Proposal│     │Technical│    │Business │    │ Canary  │    │  Full   │
 │Created │     │Review   │    │Sign-off │    │ Deploy  │    │ Rollout │
 └────────┘     └────────┘     └────────┘     └────────┘     └────────┘
```

#### Workflow Configuration

```yaml
# governance-workflow.yaml
apiVersion: arka.io/v1
kind: GovernanceWorkflow
metadata:
  name: standard-rule-approval
  namespace: governance
spec:
  stages:
    - name: draft
      description: "Initial proposal creation"
      autoTransition: false
      validations:
        - type: schema
          schema: proposal-schema-v1
        - type: completeness
          requiredFields:
            - title
            - description
            - changes
            - rationale

    - name: technical_review
      description: "Technical team review"
      autoTransition: false
      assignees:
        type: team
        team: engineering
      requirements:
        approvals: 2
        timeout: 5d
      validations:
        - type: simulation
          config:
            modes: [historical_replay, synthetic]
            minCoverage: 80
        - type: regression
          baseline: production
          maxDelta: 5%
      notifications:
        - channel: slack
          webhook: ${SLACK_ENGINEERING_WEBHOOK}

    - name: compliance_review
      description: "Compliance team approval"
      autoTransition: false
      assignees:
        type: team
        team: compliance
      requirements:
        approvals: 1
        roles: [compliance_officer]
        timeout: 3d
      validations:
        - type: regulatory_mapping
          requiredMappings: true
        - type: audit_trail
          complete: true

    - name: legal_review
      description: "Legal team sign-off"
      autoTransition: false
      assignees:
        type: team
        team: legal
      requirements:
        approvals: 1
        roles: [legal_counsel]
      conditions:
        - field: impact.riskLevel
          operator: in
          values: [high, critical]

    - name: staging
      description: "Staging environment deployment"
      autoTransition: true
      transitionDelay: 1h
      actions:
        - type: deploy
          environment: staging
          strategy: canary
          canaryPercent: 10
      validations:
        - type: health_check
          timeout: 30m
        - type: error_rate
          maxRate: 0.1%
      rollback:
        automatic: true
        conditions:
          - errorRate: "> 1%"
          - latencyP99: "> 500ms"

    - name: production
      description: "Production deployment"
      autoTransition: false
      requirements:
        approvals: 1
        roles: [release_manager]
      actions:
        - type: deploy
          environment: production
          strategy: rolling
          batchSize: 25%
          batchDelay: 15m
      monitoring:
        duration: 24h
        alerts:
          - metric: error_rate
            threshold: 0.5%
          - metric: flag_rate_delta
            threshold: 20%

  notifications:
    onStageChange:
      - channel: email
        recipients: ${STAKEHOLDER_LIST}
      - channel: slack
        webhook: ${SLACK_GOVERNANCE_WEBHOOK}

    onApproval:
      - channel: audit_log
        level: info

    onRejection:
      - channel: email
        recipients: [proposer, reviewers]
      - channel: slack
        webhook: ${SLACK_GOVERNANCE_WEBHOOK}

  escalation:
    enabled: true
    rules:
      - condition: "stage.timeout_exceeded"
        action: escalate
        to: manager
      - condition: "proposal.priority == 'critical'"
        action: fast_track
        skipStages: [legal_review]
```

#### Workflow API

```python
from pact.governance import WorkflowEngine, ApprovalRequest

workflow = WorkflowEngine(config_path="governance-workflow.yaml")

# Submit proposal to workflow
workflow_instance = await workflow.start(
    proposal_id="aml-threshold-update-2024",
    workflow="standard-rule-approval",
    metadata={
        "requestor": "compliance-team",
        "priority": "high",
        "deadline": "2024-06-15"
    }
)

# Check current status
status = await workflow.get_status(workflow_instance.id)
print(f"Current Stage: {status.current_stage}")
print(f"Pending Approvals: {status.pending_approvals}")

# Submit approval
approval = await workflow.approve(
    workflow_id=workflow_instance.id,
    stage="technical_review",
    approver="engineer@company.com",
    decision="approved",
    comments="Simulation results look good. No regressions detected."
)

# Reject with feedback
rejection = await workflow.reject(
    workflow_id=workflow_instance.id,
    stage="compliance_review",
    approver="compliance@company.com",
    decision="rejected",
    reason="Missing regulatory mapping for EU jurisdictions",
    required_changes=[
        "Add GDPR impact assessment",
        "Include EU AML directive reference"
    ]
)
```

### Semantic Diff Engine

The Semantic Diff Engine analyzes rule changes beyond textual differences.

#### Diff Types

```
┌─────────────────────────────────────────────────────────────┐
│                    Semantic Diff Engine                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Textual    │  │  Structural │  │  Behavioral         │  │
│  │  Diff       │  │  Diff       │  │  Diff               │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │               │                    │               │
│         └───────────────┼────────────────────┘               │
│                         ▼                                    │
│                ┌─────────────────┐                           │
│                │  Unified Diff   │                           │
│                │     Report      │                           │
│                └─────────────────┘                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Using the Diff Engine

```python
from pact.governance import SemanticDiff

differ = SemanticDiff()

# Compare rule versions
diff = await differ.compare(
    source="rule/aml/transaction-threshold@v1.2.0",
    target="rule/aml/transaction-threshold@v1.3.0-proposed"
)

# Textual changes
print("Textual Changes:")
for change in diff.textual:
    print(f"  Line {change.line}: {change.type}")
    print(f"    - {change.old_text}")
    print(f"    + {change.new_text}")

# Structural changes
print("\nStructural Changes:")
for change in diff.structural:
    print(f"  {change.element}: {change.type}")
    print(f"    Path: {change.path}")
    print(f"    Impact: {change.impact}")

# Behavioral changes
print("\nBehavioral Changes:")
for change in diff.behavioral:
    print(f"  Scenario: {change.scenario}")
    print(f"    Old Result: {change.old_result}")
    print(f"    New Result: {change.new_result}")
    print(f"    Significance: {change.significance}")
```

#### Diff Report

```python
# Generate comprehensive diff report
report = await differ.generate_report(
    diff=diff,
    format="html",
    include_examples=True,
    include_impact_analysis=True
)

# Save report
report.save("/reports/aml-threshold-diff-report.html")

# Get summary
summary = report.summary
print(f"Total Changes: {summary.total_changes}")
print(f"Breaking Changes: {summary.breaking_changes}")
print(f"Additions: {summary.additions}")
print(f"Modifications: {summary.modifications}")
print(f"Deletions: {summary.deletions}")
```

## Governance Policies

### Policy Definition

```yaml
# governance-policies.yaml
apiVersion: arka.io/v1
kind: GovernancePolicy
metadata:
  name: rule-change-policy
  namespace: governance
spec:
  rules:
    # Require simulation for all changes
    - name: simulation-required
      description: "All rule changes must pass simulation"
      scope:
        ruleTypes: ["*"]
      enforcement: mandatory
      requirements:
        simulation:
          modes: [historical_replay]
          minCoverage: 70
          maxRegressionRate: 5

    # Additional review for high-risk domains
    - name: high-risk-review
      description: "High-risk domains require additional approval"
      scope:
        domains: [AML, SANCTIONS, PEP]
      enforcement: mandatory
      requirements:
        approvals:
          count: 3
          roles:
            - compliance_officer
            - risk_manager
            - legal_counsel

    # Expedited path for minor changes
    - name: minor-change-fast-track
      description: "Minor changes can skip legal review"
      scope:
        changeTypes: [documentation, typo, threshold_adjustment]
      conditions:
        - field: impact.riskLevel
          operator: eq
          value: low
      modifications:
        skipStages: [legal_review]
        reducedApprovals: 1

    # Emergency change protocol
    - name: emergency-change
      description: "Emergency changes with expedited approval"
      scope:
        priority: emergency
      enforcement: mandatory
      requirements:
        approvals:
          count: 1
          roles: [incident_commander]
        postDeployment:
          reviewRequired: true
          reviewDeadline: 48h

    # Prevent direct production changes
    - name: no-direct-production
      description: "No direct changes to production"
      scope:
        environments: [production]
      enforcement: blocking
      requirements:
        sourceEnvironment: staging
        minStagingDuration: 24h
```

### Policy Enforcement

```python
from pact.governance import PolicyEngine

policy_engine = PolicyEngine(
    policies_path="governance-policies.yaml"
)

# Evaluate proposal against policies
evaluation = await policy_engine.evaluate(
    proposal_id="aml-threshold-update-2024",
    context={
        "domain": "AML",
        "risk_level": "medium",
        "change_type": "threshold_adjustment"
    }
)

print(f"Policy Compliance: {evaluation.compliant}")
print(f"Required Approvals: {evaluation.required_approvals}")
print(f"Required Stages: {evaluation.required_stages}")

if not evaluation.compliant:
    print("Policy Violations:")
    for violation in evaluation.violations:
        print(f"  - {violation.policy}: {violation.reason}")
```

## AI Model Integration

### Model Configuration

```yaml
# ai-config.yaml
aiModels:
  proposalGeneration:
    provider: openai
    model: gpt-4-turbo
    temperature: 0.3
    maxTokens: 4000
    systemPrompt: |
      You are a regulatory compliance expert. Analyze
      regulatory documents and generate precise rule
      proposals in ARKA DSL format.

  impactAnalysis:
    provider: anthropic
    model: claude-3-opus
    temperature: 0.2
    systemPrompt: |
      Analyze rule changes and assess their impact on
      compliance operations. Consider edge cases and
      potential risks.

  conflictDetection:
    provider: openai
    model: gpt-4
    temperature: 0.1
    systemPrompt: |
      Identify conflicts between regulatory rules across
      jurisdictions and domains. Suggest resolution
      strategies.

  documentParsing:
    provider: azure
    deployment: gpt-4-vision
    capabilities:
      - document_ocr
      - table_extraction
      - legal_citation_parsing
```

### Custom Model Integration

```python
from pact.governance.ai import AIModelProvider, ModelConfig

class CustomComplianceModel(AIModelProvider):
    """Custom model trained on organization's compliance data."""

    def __init__(self, model_path: str):
        self.model = load_model(model_path)

    async def generate_proposal(
        self,
        document: str,
        context: dict
    ) -> ProposalDraft:
        # Custom inference logic
        embeddings = self.model.encode(document)
        rules = self.model.generate_rules(embeddings, context)

        return ProposalDraft(
            changes=rules,
            confidence=self.model.confidence_score,
            rationale=self.model.explain()
        )

    async def analyze_impact(
        self,
        proposal: Proposal,
        historical_data: DataFrame
    ) -> ImpactAssessment:
        # Custom impact analysis
        return self.model.assess_impact(proposal, historical_data)

# Register custom model
governance_engine.register_model(
    name="custom-compliance",
    provider=CustomComplianceModel("/models/compliance-v2"),
    tasks=["proposal_generation", "impact_analysis"]
)
```

## Audit and Compliance

### Audit Trail

All governance actions are recorded in an immutable audit trail:

```python
from pact.governance import AuditLog

audit = AuditLog()

# Query audit trail
events = await audit.query(
    proposal_id="aml-threshold-update-2024",
    event_types=["approval", "rejection", "deployment"],
    start_date="2024-01-01"
)

for event in events:
    print(f"[{event.timestamp}] {event.type}")
    print(f"  Actor: {event.actor}")
    print(f"  Action: {event.action}")
    print(f"  Details: {event.details}")
    print(f"  Hash: {event.hash}")
```

### Compliance Reporting

```python
from pact.governance import ComplianceReporter

reporter = ComplianceReporter()

# Generate compliance report
report = await reporter.generate(
    period="2024-Q1",
    include=[
        "proposal_statistics",
        "approval_metrics",
        "simulation_results",
        "deployment_history",
        "policy_violations"
    ]
)

# Export for regulators
report.export(
    format="pdf",
    template="regulatory-report",
    output="/reports/governance-Q1-2024.pdf"
)
```

## Configuration Reference

### Complete Configuration

```yaml
# governance-config.yaml
governance:
  # Proposal settings
  proposals:
    defaultPriority: medium
    maxPendingPerUser: 10
    autoExpireDays: 90
    requireRationale: true
    minRationaleLength: 100

  # Simulation settings
  simulation:
    defaultMode: historical_replay
    maxDuration: 4h
    resourceLimits:
      cpu: 4
      memory: 16Gi
    dataRetention: 30d

  # Workflow settings
  workflow:
    defaultWorkflow: standard-rule-approval
    allowCustomWorkflows: true
    maxStages: 10
    defaultTimeout: 7d
    escalationEnabled: true

  # Diff settings
  diff:
    includeExamples: true
    maxExamplesPerChange: 5
    behavioralTestCount: 100

  # AI settings
  ai:
    enabled: true
    defaultProvider: openai
    fallbackProvider: anthropic
    rateLimits:
      requestsPerMinute: 60
      tokensPerDay: 1000000
    caching:
      enabled: true
      ttl: 1h

  # Notifications
  notifications:
    channels:
      - type: email
        smtp:
          host: smtp.company.com
          port: 587
      - type: slack
        webhook: ${SLACK_WEBHOOK}
      - type: teams
        webhook: ${TEAMS_WEBHOOK}

    templates:
      proposalCreated: /templates/proposal-created.html
      approvalRequired: /templates/approval-required.html
      deployed: /templates/deployed.html

  # Security
  security:
    requireMfa: true
    sessionTimeout: 1h
    auditRetention: 7y
    encryption:
      atRest: AES-256
      inTransit: TLS1.3
```

## Best Practices

### Proposal Quality

1. **Clear Rationale**: Always include detailed rationale for changes
2. **Regulatory Mapping**: Link proposals to specific regulatory requirements
3. **Impact Assessment**: Provide estimated impact before simulation
4. **Incremental Changes**: Prefer small, focused changes over large batches

### Simulation Strategy

1. **Representative Data**: Use stratified sampling for historical replay
2. **Edge Cases**: Include adversarial synthetic scenarios
3. **Performance Baseline**: Always compare against current production
4. **Coverage Metrics**: Ensure adequate coverage across jurisdictions

### Workflow Design

1. **Appropriate Stages**: Match workflow complexity to change risk
2. **Clear Ownership**: Define explicit approvers for each stage
3. **Timeout Handling**: Configure reasonable timeouts with escalation
4. **Rollback Planning**: Always have automated rollback criteria

### Security

1. **Least Privilege**: Grant minimum necessary permissions
2. **Audit Everything**: Enable comprehensive audit logging
3. **Separation of Duties**: Different approvers for each stage
4. **MFA Required**: Enforce multi-factor authentication for approvals

## Troubleshooting

### Common Issues

**Simulation Timeout**
```yaml
# Increase simulation timeout
simulation:
  timeout: 8h
  checkpointInterval: 30m
```

**Workflow Stuck**
```python
# Force workflow transition (admin only)
await workflow.force_transition(
    workflow_id="workflow-123",
    to_stage="production",
    reason="Emergency deployment required",
    approver="admin@company.com"
)
```

**AI Model Errors**
```python
# Check model health
health = await governance_engine.check_ai_health()
for model, status in health.items():
    print(f"{model}: {status.state}")
    if status.errors:
        print(f"  Errors: {status.errors}")
```

## Related Documentation

- [Rule Authoring Guide](../guides/rule-authoring.md)
- [RTVM Reference](./rtvm.md)
- [REST API Reference](../api-reference/rest-api.md)
- [Security Model](../security/security-model.md)
