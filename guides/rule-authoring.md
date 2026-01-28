# Rule Authoring Guide

This comprehensive guide covers how to write effective compliance rules for the ARKA Engine.

## Table of Contents

- [Rule DSL Overview](#rule-dsl-overview)
- [Rule Structure](#rule-structure)
- [Conditions](#conditions)
- [Actions](#actions)
- [Expressions](#expressions)
- [Data Types](#data-types)
- [Advanced Patterns](#advanced-patterns)
- [Testing Rules](#testing-rules)
- [Best Practices](#best-practices)

---

## Rule DSL Overview

ARKA rules are written in a declarative Domain-Specific Language (DSL) that compiles to RTVM bytecode. The DSL is designed to be:

- **Readable**: Clear syntax that compliance officers can understand
- **Verifiable**: Formal semantics for deterministic execution
- **Expressive**: Rich operators for complex compliance logic
- **Safe**: Type-checked and validated before deployment

### Rule File Format

Rules can be written in YAML or JSON:

```yaml
# rule.yaml
rule:
  metadata:
    name: "rule-name"
    # ... metadata
  inputs:
    # ... input definitions
  conditions:
    # ... condition definitions
  actions:
    # ... action definitions
```

---

## Rule Structure

### Complete Rule Example

```yaml
rule:
  # ═══════════════════════════════════════════════════════════════
  # METADATA
  # ═══════════════════════════════════════════════════════════════
  metadata:
    name: "enhanced-due-diligence-trigger"
    description: |
      Triggers enhanced due diligence requirements for high-risk
      customers based on transaction patterns and risk indicators.
    jurisdiction: "US"
    domain: "AML"
    version: "1.0.0"
    effective_date: "2024-01-01"
    expiry_date: null  # No expiration
    tags:
      - "edd"
      - "high-risk"
      - "kyc"
    author: "compliance-team"
    reviewers:
      - "legal-team"
      - "risk-team"

  # ═══════════════════════════════════════════════════════════════
  # INPUT DEFINITIONS
  # ═══════════════════════════════════════════════════════════════
  inputs:
    - name: "customer"
      type: "Customer"
      required: true
      schema:
        properties:
          id: { type: "string" }
          risk_score: { type: "decimal" }
          country: { type: "string" }
          customer_type: { type: "string" }
          onboarding_date: { type: "datetime" }
          pep_status: { type: "boolean" }

    - name: "transaction_history"
      type: "TransactionHistory"
      required: true
      schema:
        type: "array"
        items:
          properties:
            amount: { type: "decimal" }
            timestamp: { type: "datetime" }
            type: { type: "string" }

  # ═══════════════════════════════════════════════════════════════
  # COMPUTED VALUES (let bindings)
  # ═══════════════════════════════════════════════════════════════
  let:
    # Calculate 90-day transaction volume
    ninety_day_volume: |
      sum(
        transaction_history
          .filter(t => days_between(t.timestamp, now()) <= 90)
          .map(t => t.amount)
      )

    # Count high-value transactions
    high_value_count: |
      count(
        transaction_history
          .filter(t => t.amount >= 50000)
      )

    # Check if customer is from high-risk country
    is_high_risk_country: |
      customer.country in ["Country_A", "Country_B", "Country_C"]

    # Calculate customer tenure in months
    customer_tenure_months: |
      months_between(customer.onboarding_date, now())

  # ═══════════════════════════════════════════════════════════════
  # CONDITIONS
  # ═══════════════════════════════════════════════════════════════
  conditions:
    # Risk score condition
    - id: "high_risk_score"
      description: "Customer risk score exceeds threshold"
      expression: |
        customer.risk_score >= 75

    # PEP condition
    - id: "is_pep"
      description: "Customer is a Politically Exposed Person"
      expression: |
        customer.pep_status == true

    # Volume condition
    - id: "high_volume"
      description: "90-day transaction volume exceeds threshold"
      expression: |
        ninety_day_volume >= 500000

    # Pattern condition
    - id: "frequent_high_value"
      description: "Multiple high-value transactions"
      expression: |
        high_value_count >= 5

    # Geographic condition
    - id: "geographic_risk"
      description: "Customer in high-risk jurisdiction"
      expression: |
        is_high_risk_country

    # Composite: Requires EDD
    - id: "requires_edd"
      description: "Enhanced Due Diligence required"
      expression: |
        is_pep OR
        (high_risk_score AND high_volume) OR
        (geographic_risk AND frequent_high_value)

    # Composite: Critical review
    - id: "critical_review"
      description: "Requires immediate compliance review"
      expression: |
        is_pep AND high_risk_score AND geographic_risk

  # ═══════════════════════════════════════════════════════════════
  # ACTIONS
  # ═══════════════════════════════════════════════════════════════
  actions:
    # Flag for EDD
    - trigger: "requires_edd"
      type: "flag"
      config:
        severity: "high"
        category: "EDD_REQUIRED"
        message: |
          Enhanced Due Diligence required for customer ${customer.id}.
          Risk factors: ${describe_risk_factors()}

    # Critical review escalation
    - trigger: "critical_review"
      type: "flag"
      config:
        severity: "critical"
        category: "IMMEDIATE_REVIEW"
        message: "Critical compliance review required"

    - trigger: "critical_review"
      type: "escalate"
      config:
        queue: "compliance_urgent"
        priority: 1
        sla_hours: 4

    # Annotations
    - trigger: "requires_edd"
      type: "annotate"
      config:
        annotations:
          edd_required: true
          edd_trigger_reason: |
            ${get_trigger_reason()}
          review_deadline: |
            ${add_business_days(now(), 5)}

    # Score adjustment
    - trigger: "requires_edd"
      type: "score_adjustment"
      config:
        field: "risk_score"
        adjustment: 10
        reason: "EDD trigger detected"
```

---

## Conditions

### Simple Conditions

```yaml
conditions:
  # Direct field comparison
  - id: "amount_over_threshold"
    expression: |
      transaction.amount > 10000

  # Equality check
  - id: "is_cash"
    expression: |
      transaction.type == "cash"

  # String matching
  - id: "sender_prefix"
    expression: |
      starts_with(transaction.sender_id, "HIGH_RISK_")

  # Null check
  - id: "has_recipient"
    expression: |
      transaction.recipient != null

  # Boolean field
  - id: "is_flagged"
    expression: |
      customer.flagged == true
```

### Comparison Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `==` | Equal | `a == b` |
| `!=` | Not equal | `a != b` |
| `>` | Greater than | `a > b` |
| `>=` | Greater than or equal | `a >= b` |
| `<` | Less than | `a < b` |
| `<=` | Less than or equal | `a <= b` |
| `in` | In collection | `a in [1, 2, 3]` |
| `not in` | Not in collection | `a not in [1, 2, 3]` |

### Logical Operators

```yaml
conditions:
  # AND
  - id: "high_risk_large_amount"
    expression: |
      customer.risk_score > 75 AND transaction.amount > 50000

  # OR
  - id: "pep_or_sanctioned"
    expression: |
      customer.is_pep OR customer.is_sanctioned

  # NOT
  - id: "not_verified"
    expression: |
      NOT customer.verified

  # Complex combination
  - id: "complex_condition"
    expression: |
      (condition_a AND condition_b) OR
      (condition_c AND NOT condition_d)
```

### Collection Conditions

```yaml
conditions:
  # Any match
  - id: "has_large_transaction"
    expression: |
      any(transactions, t => t.amount > 100000)

  # All match
  - id: "all_domestic"
    expression: |
      all(transactions, t => t.country == "US")

  # None match
  - id: "no_suspicious"
    expression: |
      none(transactions, t => t.flagged == true)

  # Count threshold
  - id: "multiple_cash"
    expression: |
      count(transactions.filter(t => t.type == "cash")) >= 3

  # Sum threshold
  - id: "daily_volume"
    expression: |
      sum(todays_transactions.map(t => t.amount)) > 50000
```

### Date/Time Conditions

```yaml
conditions:
  # Within time range
  - id: "recent_transaction"
    expression: |
      hours_between(transaction.timestamp, now()) < 24

  # Business days
  - id: "within_reporting_window"
    expression: |
      business_days_between(transaction.timestamp, now()) <= 15

  # Specific date comparison
  - id: "after_effective_date"
    expression: |
      transaction.timestamp >= date("2024-01-01")

  # Day of week
  - id: "weekend_transaction"
    expression: |
      day_of_week(transaction.timestamp) in ["Saturday", "Sunday"]

  # Time of day
  - id: "after_hours"
    expression: |
      hour_of_day(transaction.timestamp) < 6 OR
      hour_of_day(transaction.timestamp) > 22
```

---

## Actions

### Flag Action

Creates a compliance flag:

```yaml
actions:
  - trigger: "condition_id"
    type: "flag"
    config:
      severity: "high"          # critical, high, medium, low
      category: "CTR_REQUIRED"  # Classification category
      message: "Description of the flag"
      code: "FL-001"            # Optional code
      metadata:                 # Additional metadata
        field: "transaction.amount"
        value: "${transaction.amount}"
```

### Annotate Action

Adds metadata to the evaluation result:

```yaml
actions:
  - trigger: "condition_id"
    type: "annotate"
    config:
      annotations:
        requires_ctr: true
        reporting_deadline: "${add_days(now(), 15)}"
        risk_factors:
          - "high_amount"
          - "cash_transaction"
```

### Escalate Action

Routes to review queue:

```yaml
actions:
  - trigger: "condition_id"
    type: "escalate"
    config:
      queue: "compliance_review"
      priority: 1                    # 1 = highest
      sla_hours: 24
      assignee_group: "aml_analysts"
      notes: "Requires senior analyst review"
```

### Block Action

Prevents transaction/action:

```yaml
actions:
  - trigger: "condition_id"
    type: "block"
    config:
      reason: "Sanctions match detected"
      code: "BLK-SANCTIONS"
      override_allowed: false
```

### Notify Action

Sends notification:

```yaml
actions:
  - trigger: "condition_id"
    type: "notify"
    config:
      channels:
        - type: "email"
          recipients: ["compliance@company.com"]
        - type: "webhook"
          url: "https://alerts.company.com/webhook"
      template: "high_risk_alert"
      data:
        customer_id: "${customer.id}"
        amount: "${transaction.amount}"
```

### Score Adjustment Action

Modifies risk scores:

```yaml
actions:
  - trigger: "condition_id"
    type: "score_adjustment"
    config:
      field: "risk_score"
      adjustment: 15            # Add 15 points
      # OR
      multiplier: 1.5           # Multiply by 1.5
      # OR
      set_value: 100            # Set to specific value
      max_value: 100
      min_value: 0
      reason: "High-risk pattern detected"
```

---

## Expressions

### Arithmetic Expressions

```yaml
let:
  # Basic math
  total: |
    transaction.amount + transaction.fee

  percentage: |
    (flagged_count / total_count) * 100

  # Decimal precision
  precise_calculation: |
    round(amount * rate, 2)
```

### String Expressions

```yaml
let:
  # Concatenation
  full_name: |
    customer.first_name + " " + customer.last_name

  # Formatting
  formatted_amount: |
    format_currency(transaction.amount, transaction.currency)

  # Pattern matching
  matches_pattern: |
    regex_match(transaction.description, "^WIRE.*INTL$")

  # Case conversion
  normalized_name: |
    upper(trim(customer.name))
```

### Collection Expressions

```yaml
let:
  # Filter
  large_transactions: |
    transactions.filter(t => t.amount > 10000)

  # Map
  amounts: |
    transactions.map(t => t.amount)

  # Reduce
  total_amount: |
    transactions.reduce(0, (acc, t) => acc + t.amount)

  # Sort
  sorted_by_amount: |
    transactions.sort_by(t => t.amount, "desc")

  # Group
  by_type: |
    transactions.group_by(t => t.type)

  # First/Last
  latest_transaction: |
    transactions.sort_by(t => t.timestamp, "desc").first()

  # Distinct
  unique_countries: |
    transactions.map(t => t.country).distinct()
```

### Conditional Expressions

```yaml
let:
  # If-then-else
  risk_category: |
    if risk_score >= 75 then "HIGH"
    else if risk_score >= 50 then "MEDIUM"
    else "LOW"

  # Coalesce (null handling)
  display_name: |
    coalesce(customer.preferred_name, customer.legal_name, "Unknown")

  # Case expression
  fee_rate: |
    case transaction.type
      when "wire" then 0.01
      when "ach" then 0.005
      when "cash" then 0.02
      else 0.015
    end
```

### Date/Time Expressions

```yaml
let:
  # Current time
  current_time: |
    now()

  # Date arithmetic
  due_date: |
    add_business_days(now(), 15)

  deadline: |
    add_hours(transaction.timestamp, 24)

  # Date extraction
  transaction_month: |
    month(transaction.timestamp)

  transaction_year: |
    year(transaction.timestamp)

  # Date formatting
  formatted_date: |
    format_date(transaction.timestamp, "YYYY-MM-DD")

  # Period calculations
  days_since_onboarding: |
    days_between(customer.onboarding_date, now())

  # Fiscal periods
  fiscal_quarter: |
    fiscal_quarter(transaction.timestamp, "US")
```

---

## Data Types

### Primitive Types

| Type | Description | Example |
|------|-------------|---------|
| `boolean` | True/false | `true`, `false` |
| `integer` | Whole number | `42`, `-100` |
| `decimal` | Precise decimal | `123.45` |
| `string` | Text | `"hello"` |
| `datetime` | Date and time | `"2024-01-15T10:30:00Z"` |
| `date` | Date only | `"2024-01-15"` |
| `duration` | Time duration | `"P30D"` (30 days) |

### Collection Types

```yaml
# Array
amounts: [100, 200, 300]

# Object/Map
customer:
  name: "John"
  age: 30

# Array of objects
transactions:
  - id: "t1"
    amount: 100
  - id: "t2"
    amount: 200
```

### Type Conversions

```yaml
let:
  # String to number
  amount_num: |
    to_decimal(transaction.amount_string)

  # Number to string
  amount_str: |
    to_string(transaction.amount)

  # String to date
  parsed_date: |
    parse_date(date_string, "YYYY-MM-DD")

  # Date to string
  date_str: |
    format_date(transaction.timestamp, "MM/DD/YYYY")
```

---

## Advanced Patterns

### Threshold Patterns

```yaml
# Single threshold
conditions:
  - id: "over_threshold"
    expression: |
      transaction.amount > threshold(
        jurisdiction: context.jurisdiction,
        type: "CTR",
        default: 10000
      )
```

### Velocity Patterns

```yaml
let:
  # Count in time window
  transactions_last_hour: |
    count(
      transaction_history.filter(t =>
        minutes_between(t.timestamp, now()) <= 60
      )
    )

  # Sum in time window
  volume_last_24h: |
    sum(
      transaction_history
        .filter(t => hours_between(t.timestamp, now()) <= 24)
        .map(t => t.amount)
    )

conditions:
  - id: "velocity_exceeded"
    expression: |
      transactions_last_hour > 10 OR
      volume_last_24h > 100000
```

### Aggregation Patterns

```yaml
let:
  # Daily aggregate
  daily_totals: |
    transaction_history
      .group_by(t => date(t.timestamp))
      .map((date, txns) => {
        date: date,
        total: sum(txns.map(t => t.amount)),
        count: count(txns)
      })

  # Rolling average
  rolling_avg_7d: |
    avg(
      daily_totals
        .filter(d => days_between(d.date, now()) <= 7)
        .map(d => d.total)
    )

conditions:
  - id: "above_rolling_average"
    expression: |
      transaction.amount > rolling_avg_7d * 3
```

### Relationship Patterns

```yaml
let:
  # Related parties
  related_parties: |
    get_related_parties(customer.id)

  # Beneficial owners
  beneficial_owners: |
    get_beneficial_owners(customer.id, min_ownership: 0.25)

  # Aggregate across related
  related_volume: |
    sum(
      related_parties
        .flat_map(p => get_transactions(p.id, last_days: 30))
        .map(t => t.amount)
    )

conditions:
  - id: "related_party_volume"
    expression: |
      related_volume > 1000000
```

### Cross-Rule References

```yaml
rule:
  metadata:
    name: "secondary-rule"
    depends_on:
      - "primary-rule"

  conditions:
    - id: "build_on_primary"
      expression: |
        rule_result("primary-rule").flagged AND
        additional_condition
```

---

## Testing Rules

### Test File Structure

```yaml
# rule.test.yaml
tests:
  - name: "Test case name"
    description: "What this test verifies"
    input:
      customer:
        id: "cust_001"
        risk_score: 80
      transaction:
        amount: 15000
    expected:
      decision: "non_compliant"
      flags:
        - category: "HIGH_RISK"
          severity: "high"
```

### Test Categories

```yaml
tests:
  # Positive test - should trigger
  - name: "High risk customer triggers flag"
    input:
      customer: { risk_score: 90 }
    expected:
      decision: "non_compliant"
      flags:
        - category: "HIGH_RISK"

  # Negative test - should not trigger
  - name: "Low risk customer passes"
    input:
      customer: { risk_score: 20 }
    expected:
      decision: "compliant"
      flags: []

  # Boundary test
  - name: "Exactly at threshold"
    input:
      transaction: { amount: 10000 }
    expected:
      decision: "compliant"  # >= 10000, not > 10000

  # Edge case
  - name: "Null risk score handled"
    input:
      customer: { risk_score: null }
    expected:
      decision: "compliant"
      annotations:
        risk_score_missing: true
```

### Running Tests

```bash
# Run all tests
arka-cli rule test my-rule.yaml

# Run specific tests
arka-cli rule test my-rule.yaml --filter "high risk"

# Verbose output
arka-cli rule test my-rule.yaml --verbose

# Generate coverage report
arka-cli rule test my-rule.yaml --coverage
```

---

## Best Practices

### 1. Use Descriptive Names

```yaml
# Good
- id: "customer_risk_score_exceeds_high_threshold"
  description: "Customer's risk score is above 75"

# Avoid
- id: "cond1"
  description: "Check score"
```

### 2. Keep Conditions Atomic

```yaml
# Good - separate conditions
conditions:
  - id: "high_amount"
    expression: "amount > 10000"
  - id: "is_cash"
    expression: "type == 'cash'"
  - id: "requires_ctr"
    expression: "high_amount AND is_cash"

# Avoid - monolithic condition
conditions:
  - id: "check_everything"
    expression: |
      amount > 10000 AND type == 'cash' AND
      country == 'US' AND ...
```

### 3. Use Let Bindings for Complex Calculations

```yaml
# Good
let:
  daily_volume: |
    sum(transactions.filter(t => same_day(t)).map(t => t.amount))

conditions:
  - id: "volume_check"
    expression: "daily_volume > 50000"

# Avoid - inline complex expressions
conditions:
  - id: "volume_check"
    expression: |
      sum(transactions.filter(t => same_day(t)).map(t => t.amount)) > 50000
```

### 4. Document Your Rules

```yaml
rule:
  metadata:
    description: |
      This rule implements BSA/AML requirements for Currency
      Transaction Reports (CTRs) per FinCEN guidelines.

      References:
      - 31 CFR 1010.311
      - FinCEN CTR Guidance (2020)

    tags: ["ctr", "bsa", "fincen", "cash-reporting"]
```

### 5. Handle Edge Cases

```yaml
let:
  # Handle null safely
  risk_score: |
    coalesce(customer.risk_score, 0)

  # Handle empty collections
  transaction_count: |
    if transactions == null then 0
    else count(transactions)
```

### 6. Version Your Rules

```yaml
metadata:
  version: "1.2.3"  # major.minor.patch
  changelog:
    - version: "1.2.3"
      date: "2024-01-15"
      changes:
        - "Increased threshold from 10000 to 15000"
    - version: "1.2.2"
      date: "2024-01-01"
      changes:
        - "Added cash transaction condition"
```

---

## Next Steps

- [Testing Strategies](./testing.md) - Comprehensive testing guide
- [RTVM Reference](../components/rtvm.md) - Execution engine details
- [API Reference](../api-reference/rest-api.md) - Deploy and manage rules
