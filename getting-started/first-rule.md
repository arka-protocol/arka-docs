# Your First Rule

This tutorial walks you through creating, testing, and deploying your first compliance rule in ARKA Engine.

## Prerequisites

- ARKA Engine running (see [Quick Start](./quickstart.md))
- Basic understanding of [ARKA concepts](./concepts.md)

## What We'll Build

We'll create an AML (Anti-Money Laundering) rule that:
1. Flags transactions over $10,000 (CTR threshold)
2. Flags rapid successive transactions from the same sender
3. Produces a risk score based on multiple factors

---

## Step 1: Understanding the Rule DSL

ARKA rules are written in a declarative DSL. Here's the basic structure:

```yaml
rule:
  metadata:
    name: "rule-name"
    description: "What this rule does"
    jurisdiction: "US"
    domain: "AML"
    version: "1.0.0"

  inputs:
    - name: "transaction"
      type: "Transaction"

  conditions:
    - # condition definitions

  actions:
    - # action definitions
```

---

## Step 2: Create the Rule File

Create a file named `ctr-threshold-rule.yaml`:

```yaml
# ctr-threshold-rule.yaml
rule:
  metadata:
    name: "ctr-threshold-flag"
    description: "Flag transactions meeting CTR reporting threshold"
    jurisdiction: "US"
    domain: "AML"
    version: "1.0.0"
    effective_date: "2024-01-01"
    tags:
      - "ctr"
      - "bsa"
      - "reporting"

  inputs:
    - name: "transaction"
      type: "Transaction"
      schema:
        properties:
          id:
            type: "string"
          amount:
            type: "decimal"
          currency:
            type: "string"
          sender_id:
            type: "string"
          recipient_id:
            type: "string"
          timestamp:
            type: "datetime"
          type:
            type: "string"
            enum: ["wire", "ach", "cash", "check"]

  # Define reusable expressions
  let:
    amount_usd: |
      if transaction.currency == "USD" then
        transaction.amount
      else
        convert_currency(transaction.amount, transaction.currency, "USD")

    is_cash_transaction: |
      transaction.type == "cash"

  conditions:
    # Primary condition: amount threshold
    - id: "amount_threshold"
      description: "Transaction amount meets CTR threshold"
      expression: |
        amount_usd >= 10000

    # Secondary condition: cash transaction
    - id: "cash_transaction"
      description: "Transaction is cash-based"
      expression: |
        is_cash_transaction

    # Combined condition
    - id: "ctr_reportable"
      description: "Transaction requires CTR filing"
      expression: |
        amount_threshold AND cash_transaction

  actions:
    - trigger: "ctr_reportable"
      type: "flag"
      config:
        severity: "high"
        category: "CTR_REQUIRED"
        message: "Cash transaction of ${amount_usd} USD requires CTR filing"

    - trigger: "ctr_reportable"
      type: "annotate"
      config:
        annotations:
          ctr_required: true
          reporting_deadline: "15_business_days"

    - trigger: "amount_threshold AND NOT cash_transaction"
      type: "flag"
      config:
        severity: "medium"
        category: "LARGE_TRANSACTION"
        message: "Non-cash transaction exceeds $10,000 threshold"
```

---

## Step 3: Validate the Rule

Before deploying, validate the rule syntax and semantics:

```bash
# Using the CLI
arka-cli rule validate ctr-threshold-rule.yaml

# Expected output:
# ✓ Syntax valid
# ✓ Schema valid
# ✓ Expressions valid
# ✓ Actions valid
# Rule validation passed!
```

You can also use the API:

```bash
curl -X POST http://localhost:8080/api/v1/rules/validate \
  -H "Content-Type: application/x-yaml" \
  --data-binary @ctr-threshold-rule.yaml
```

---

## Step 4: Test the Rule Locally

Create test cases in `ctr-threshold-rule.test.yaml`:

```yaml
# ctr-threshold-rule.test.yaml
tests:
  - name: "Cash transaction over threshold triggers CTR flag"
    input:
      transaction:
        id: "txn_001"
        amount: 15000
        currency: "USD"
        sender_id: "cust_123"
        recipient_id: "merchant_456"
        timestamp: "2024-01-15T10:30:00Z"
        type: "cash"
    expected:
      decision: "non_compliant"
      flags:
        - category: "CTR_REQUIRED"
          severity: "high"
      annotations:
        ctr_required: true

  - name: "Wire transfer over threshold triggers medium flag"
    input:
      transaction:
        id: "txn_002"
        amount: 25000
        currency: "USD"
        sender_id: "cust_123"
        recipient_id: "business_789"
        timestamp: "2024-01-15T11:00:00Z"
        type: "wire"
    expected:
      decision: "non_compliant"
      flags:
        - category: "LARGE_TRANSACTION"
          severity: "medium"
      annotations:
        ctr_required: false

  - name: "Small cash transaction passes"
    input:
      transaction:
        id: "txn_003"
        amount: 5000
        currency: "USD"
        sender_id: "cust_456"
        recipient_id: "merchant_123"
        timestamp: "2024-01-15T12:00:00Z"
        type: "cash"
    expected:
      decision: "compliant"
      flags: []

  - name: "Foreign currency conversion works"
    input:
      transaction:
        id: "txn_004"
        amount: 12000
        currency: "EUR"
        sender_id: "cust_789"
        recipient_id: "merchant_321"
        timestamp: "2024-01-15T13:00:00Z"
        type: "cash"
    expected:
      decision: "non_compliant"
      flags:
        - category: "CTR_REQUIRED"
```

Run the tests:

```bash
arka-cli rule test ctr-threshold-rule.yaml --tests ctr-threshold-rule.test.yaml

# Expected output:
# Running 4 test cases...
# ✓ Cash transaction over threshold triggers CTR flag
# ✓ Wire transfer over threshold triggers medium flag
# ✓ Small cash transaction passes
# ✓ Foreign currency conversion works
#
# 4/4 tests passed
```

---

## Step 5: Deploy the Rule

Deploy the rule to ARKA Engine:

```bash
# Using the CLI
arka-cli rule deploy ctr-threshold-rule.yaml

# Expected output:
# Compiling rule...
# Rule compiled successfully
# Bytecode hash: sha256:abc123...
#
# Deploying to ARKA Engine...
# Rule deployed!
#
# Rule ID: rule_ctr_threshold_flag_v1
# Version: 1.0.0
# Status: active
```

Or use the API:

```bash
curl -X POST http://localhost:8080/api/v1/rules \
  -H "Content-Type: application/x-yaml" \
  --data-binary @ctr-threshold-rule.yaml
```

---

## Step 6: Evaluate Transactions

Now let's evaluate some transactions against our rule:

```bash
# Evaluate a single transaction
curl -X POST http://localhost:8080/api/v1/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "jurisdiction": "US",
      "domain": "AML",
      "timestamp": "2024-01-15T14:00:00Z"
    },
    "input": {
      "transaction": {
        "id": "txn_live_001",
        "amount": 12500,
        "currency": "USD",
        "sender_id": "customer_abc",
        "recipient_id": "merchant_xyz",
        "timestamp": "2024-01-15T14:00:00Z",
        "type": "cash"
      }
    }
  }'
```

Response:

```json
{
  "evaluation_id": "eval_abc123",
  "timestamp": "2024-01-15T14:00:01Z",
  "result": {
    "decision": "non_compliant",
    "flags": [
      {
        "rule_id": "rule_ctr_threshold_flag_v1",
        "condition_id": "ctr_reportable",
        "category": "CTR_REQUIRED",
        "severity": "high",
        "message": "Cash transaction of 12500 USD requires CTR filing"
      }
    ],
    "annotations": {
      "ctr_required": true,
      "reporting_deadline": "15_business_days"
    }
  },
  "metadata": {
    "rules_evaluated": ["rule_ctr_threshold_flag_v1"],
    "rule_version": "1.0.0",
    "evaluation_duration_ms": 2
  },
  "trust_proof": {
    "proof_id": "proof_xyz789",
    "input_hash": "sha256:input123...",
    "rule_hash": "sha256:rule456...",
    "output_hash": "sha256:output789...",
    "signature": "sig_abc..."
  }
}
```

---

## Step 7: View in Dashboard

Open the ARKA Dashboard at `http://localhost:8080/dashboard` to:

1. **View Rule Details**: See the compiled rule, conditions, and actions
2. **Monitor Evaluations**: Watch real-time evaluation metrics
3. **Analyze Flags**: Review triggered flags and their distribution
4. **Verify Proofs**: Inspect trust proofs for any evaluation

---

## Step 8: Create a More Complex Rule

Let's add a structuring detection rule that looks for patterns:

```yaml
# structuring-detection-rule.yaml
rule:
  metadata:
    name: "structuring-detection"
    description: "Detect potential structuring (smurfing) behavior"
    jurisdiction: "US"
    domain: "AML"
    version: "1.0.0"

  inputs:
    - name: "transaction"
      type: "Transaction"
    - name: "transaction_history"
      type: "TransactionHistory"
      description: "Recent transactions for the sender"

  let:
    # Calculate aggregates from history
    same_day_transactions: |
      transaction_history.filter(t =>
        same_day(t.timestamp, transaction.timestamp) AND
        t.sender_id == transaction.sender_id
      )

    same_day_total: |
      sum(same_day_transactions.map(t => t.amount)) + transaction.amount

    same_day_count: |
      count(same_day_transactions) + 1

    # Check for just-under-threshold pattern
    transactions_just_under: |
      same_day_transactions.filter(t =>
        t.amount >= 8000 AND t.amount < 10000
      )

    just_under_count: |
      count(transactions_just_under) +
      (if transaction.amount >= 8000 AND transaction.amount < 10000 then 1 else 0)

  conditions:
    - id: "aggregate_over_threshold"
      description: "Same-day aggregate exceeds CTR threshold"
      expression: |
        same_day_total >= 10000

    - id: "multiple_transactions"
      description: "Multiple transactions in same day"
      expression: |
        same_day_count >= 3

    - id: "just_under_pattern"
      description: "Multiple transactions just under threshold"
      expression: |
        just_under_count >= 2

    - id: "potential_structuring"
      description: "Pattern indicates potential structuring"
      expression: |
        aggregate_over_threshold AND
        multiple_transactions AND
        just_under_pattern

    - id: "high_confidence_structuring"
      description: "High confidence structuring detection"
      expression: |
        potential_structuring AND same_day_count >= 5

  actions:
    - trigger: "high_confidence_structuring"
      type: "flag"
      config:
        severity: "critical"
        category: "STRUCTURING_HIGH"
        message: "High confidence structuring detected: ${same_day_count} transactions totaling ${same_day_total} USD"

    - trigger: "potential_structuring AND NOT high_confidence_structuring"
      type: "flag"
      config:
        severity: "high"
        category: "STRUCTURING_POTENTIAL"
        message: "Potential structuring: ${same_day_count} transactions totaling ${same_day_total} USD"

    - trigger: "potential_structuring"
      type: "escalate"
      config:
        queue: "aml_investigations"
        priority: "high"

    - trigger: "potential_structuring"
      type: "annotate"
      config:
        annotations:
          structuring_risk_score: |
            min(100, (just_under_count * 20) + (same_day_count * 10))
          requires_sar_review: true
```

Deploy and test:

```bash
arka-cli rule validate structuring-detection-rule.yaml
arka-cli rule deploy structuring-detection-rule.yaml
```

---

## Step 9: Link Rules Together

Create a rule set that combines our rules:

```yaml
# aml-rule-set.yaml
ruleset:
  metadata:
    name: "us-aml-basic"
    description: "Basic US AML compliance rule set"
    jurisdiction: "US"
    domain: "AML"
    version: "1.0.0"

  rules:
    - ref: "ctr-threshold-flag"
      version: "^1.0.0"
      required: true

    - ref: "structuring-detection"
      version: "^1.0.0"
      required: true
      depends_on:
        - "ctr-threshold-flag"  # Evaluate CTR first

  evaluation_order: "dependency"  # Respect depends_on

  aggregate_decision:
    strategy: "most_severe"  # Use most severe flag as overall decision
```

Deploy the rule set:

```bash
arka-cli ruleset deploy aml-rule-set.yaml
```

---

## Step 10: Monitor and Iterate

### View Rule Metrics

```bash
# Get rule evaluation statistics
curl http://localhost:8080/api/v1/rules/rule_ctr_threshold_flag_v1/metrics

# Response:
{
  "rule_id": "rule_ctr_threshold_flag_v1",
  "period": "last_24h",
  "metrics": {
    "evaluations": 15420,
    "flags_triggered": 342,
    "flag_rate": 0.0222,
    "avg_evaluation_ms": 1.8,
    "p99_evaluation_ms": 5.2
  }
}
```

### Update the Rule

When regulations change, create a new version:

```yaml
# ctr-threshold-rule-v1.1.yaml
rule:
  metadata:
    name: "ctr-threshold-flag"
    version: "1.1.0"  # Incremented version
    # ... rest of metadata

  conditions:
    - id: "amount_threshold"
      expression: |
        amount_usd >= 10000  # Unchanged

    # New condition for aggregated reporting
    - id: "aggregate_threshold"
      description: "Aggregated transactions in 24h exceed threshold"
      expression: |
        sum_24h_transactions(transaction.sender_id) >= 10000
```

Deploy the update:

```bash
arka-cli rule deploy ctr-threshold-rule-v1.1.yaml

# Output:
# Previous version: 1.0.0
# New version: 1.1.0
# Changes detected:
#   + Added condition: aggregate_threshold
#
# Rule updated successfully
```

---

## Summary

You've learned how to:

1. Write rules using the ARKA DSL
2. Validate rules before deployment
3. Test rules with test cases
4. Deploy rules to ARKA Engine
5. Evaluate transactions against rules
6. Create complex rules with aggregations
7. Link rules into rule sets
8. Monitor and update rules

## Next Steps

- [Rule Authoring Guide](../guides/rule-authoring.md) - Advanced rule patterns
- [RTVM Reference](../components/rtvm.md) - Understand the execution engine
- [Testing Strategies](../guides/testing.md) - Comprehensive testing approaches
- [Integration Patterns](../guides/integration-patterns.md) - Connect to your systems
