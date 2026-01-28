# Testing Guide

This guide covers testing strategies for ARKA Engine rules, plugins, and integrations.

## Overview

Comprehensive testing ensures compliance rules behave correctly across all scenarios:

```
┌─────────────────────────────────────────────────────────────┐
│                    Testing Pyramid                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                        ┌─────┐                               │
│                       /       \          E2E Tests           │
│                      /   E2E   \         (Few, Slow)         │
│                     /───────────\                            │
│                    /             \                           │
│                   /  Integration  \      Integration Tests   │
│                  /─────────────────\     (Some, Medium)      │
│                 /                   \                        │
│                /     Unit Tests      \   Unit Tests          │
│               /───────────────────────\  (Many, Fast)        │
│              /                         \                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Unit Testing Rules

### Rule DSL Testing

```python
import pytest
from pact.testing import RuleTestCase, MockTransaction

class TestAMLRules(RuleTestCase):
    """Unit tests for AML rules."""

    def test_high_value_transaction_flagged(self):
        """Test that high value transactions are flagged."""
        rule = self.load_rule("rules/aml/high-value-transaction.pact")

        transaction = MockTransaction(
            id="txn-001",
            type="wire_transfer",
            amount=75000,
            currency="USD"
        )

        result = self.evaluate(rule, transaction)

        assert not result.compliant
        assert "high_value_transfer" in result.actions
        assert result.get_action("flag_for_review") is not None

    def test_normal_transaction_passes(self):
        """Test that normal transactions pass."""
        rule = self.load_rule("rules/aml/high-value-transaction.pact")

        transaction = MockTransaction(
            id="txn-002",
            type="wire_transfer",
            amount=5000,
            currency="USD"
        )

        result = self.evaluate(rule, transaction)

        assert result.compliant
        assert len(result.violations) == 0

    def test_currency_conversion(self):
        """Test currency conversion in threshold check."""
        rule = self.load_rule("rules/aml/high-value-transaction.pact")

        # EUR transaction that exceeds USD threshold after conversion
        transaction = MockTransaction(
            id="txn-003",
            type="wire_transfer",
            amount=60000,
            currency="EUR"  # ~66000 USD
        )

        result = self.evaluate(rule, transaction)

        assert not result.compliant

    @pytest.mark.parametrize("amount,expected_compliant", [
        (9999, True),
        (10000, True),  # At threshold
        (10001, False),
        (50000, False),
    ])
    def test_threshold_boundary(self, amount, expected_compliant):
        """Test threshold boundary conditions."""
        rule = self.load_rule("rules/aml/reporting-threshold.pact")

        transaction = MockTransaction(
            type="cash_deposit",
            amount=amount,
            currency="USD"
        )

        result = self.evaluate(rule, transaction)
        assert result.compliant == expected_compliant
```

### Testing Rule Conditions

```python
from pact.testing import ConditionTester

class TestRuleConditions:
    """Test individual rule conditions."""

    def setup_method(self):
        self.tester = ConditionTester()

    def test_nested_condition(self):
        """Test nested AND/OR conditions."""
        condition = """
            (transaction.amount > 10000 AND transaction.type == "wire")
            OR
            (transaction.amount > 5000 AND transaction.destination.country IN high_risk_countries)
        """

        # Test first branch
        result = self.tester.evaluate(condition, {
            "transaction": {
                "amount": 15000,
                "type": "wire",
                "destination": {"country": "US"}
            }
        })
        assert result.matched

        # Test second branch
        result = self.tester.evaluate(condition, {
            "transaction": {
                "amount": 7000,
                "type": "ach",
                "destination": {"country": "IR"}
            },
            "high_risk_countries": ["IR", "KP", "SY"]
        })
        assert result.matched

        # Test no match
        result = self.tester.evaluate(condition, {
            "transaction": {
                "amount": 3000,
                "type": "ach",
                "destination": {"country": "CA"}
            }
        })
        assert not result.matched

    def test_aggregate_functions(self):
        """Test aggregate function conditions."""
        condition = """
            aggregate(
                transactions,
                filter: sender_id == transaction.sender_id,
                window: 24h,
                sum: amount
            ) > 20000
        """

        context = {
            "transaction": {
                "sender_id": "user-123",
                "amount": 5000
            },
            "transactions": [
                {"sender_id": "user-123", "amount": 8000, "timestamp": "2024-03-15T10:00:00Z"},
                {"sender_id": "user-123", "amount": 9000, "timestamp": "2024-03-15T12:00:00Z"},
                {"sender_id": "user-456", "amount": 15000, "timestamp": "2024-03-15T11:00:00Z"},
            ]
        }

        result = self.tester.evaluate(condition, context)
        assert result.matched  # 8000 + 9000 + 5000 = 22000 > 20000
```

### Testing Rule Actions

```python
from pact.testing import ActionTester, MockActionHandler

class TestRuleActions:
    """Test rule action execution."""

    def setup_method(self):
        self.action_handler = MockActionHandler()
        self.tester = ActionTester(action_handler=self.action_handler)

    def test_flag_for_review_action(self):
        """Test flag_for_review action execution."""
        rule_dsl = """
            rule test_rule {
                when {
                    transaction.amount > 10000
                }
                then {
                    flag_for_review(
                        reason: "High value transaction",
                        priority: "high",
                        assigned_to: "compliance-team"
                    )
                }
            }
        """

        self.tester.execute(rule_dsl, {
            "transaction": {"amount": 15000}
        })

        # Verify action was called
        assert self.action_handler.was_called("flag_for_review")
        action = self.action_handler.get_action("flag_for_review")
        assert action.params["reason"] == "High value transaction"
        assert action.params["priority"] == "high"

    def test_multiple_actions(self):
        """Test multiple actions are executed."""
        rule_dsl = """
            rule test_rule {
                when {
                    transaction.risk_score > 80
                }
                then {
                    flag_for_review(reason: "High risk")
                    notify(channel: "alerts", severity: "high")
                    log_event(type: "high_risk_detected")
                }
            }
        """

        self.tester.execute(rule_dsl, {
            "transaction": {"risk_score": 90}
        })

        assert self.action_handler.action_count == 3
        assert self.action_handler.was_called("flag_for_review")
        assert self.action_handler.was_called("notify")
        assert self.action_handler.was_called("log_event")

    def test_conditional_actions(self):
        """Test conditional action execution."""
        rule_dsl = """
            rule test_rule {
                when {
                    transaction.amount > 10000
                }
                then {
                    if transaction.amount > 50000 {
                        escalate(to: "senior-compliance")
                    } else {
                        flag_for_review(reason: "Review needed")
                    }
                }
            }
        """

        # Test escalation path
        self.tester.execute(rule_dsl, {
            "transaction": {"amount": 75000}
        })
        assert self.action_handler.was_called("escalate")
        assert not self.action_handler.was_called("flag_for_review")

        # Reset and test review path
        self.action_handler.reset()
        self.tester.execute(rule_dsl, {
            "transaction": {"amount": 25000}
        })
        assert not self.action_handler.was_called("escalate")
        assert self.action_handler.was_called("flag_for_review")
```

## Integration Testing

### Testing Rule Sets

```python
import pytest
from pact.testing import IntegrationTestSuite, TestScenario

class TestAMLRuleSet(IntegrationTestSuite):
    """Integration tests for AML rule set."""

    @pytest.fixture(autouse=True)
    def setup_rules(self):
        """Load all AML rules for testing."""
        self.load_rule_set("rules/aml/")

    def test_structuring_detection(self):
        """Test detection of structuring attempts."""
        # Multiple small transactions to avoid reporting threshold
        transactions = [
            {"id": f"txn-{i}", "amount": 9500, "sender_id": "user-123"}
            for i in range(5)
        ]

        for txn in transactions:
            self.submit_transaction(txn)

        # The 5th transaction should trigger structuring detection
        results = self.get_evaluation_results()

        assert any(
            r.rule_id == "structuring_detection"
            for r in results
            if not r.compliant
        )

    def test_end_to_end_wire_transfer(self):
        """End-to-end test for wire transfer compliance."""
        scenario = TestScenario(
            name="International Wire Transfer",
            transaction={
                "id": "wire-001",
                "type": "wire_transfer",
                "amount": 25000,
                "currency": "USD",
                "source": {
                    "account": "acct-001",
                    "bank": "US_BANK",
                    "country": "US"
                },
                "destination": {
                    "account": "acct-002",
                    "bank": "UK_BANK",
                    "country": "UK"
                },
                "purpose": "business_payment"
            },
            expected_rules=[
                "aml/wire-transfer-monitoring",
                "aml/cross-border-transfer",
                "sanctions/ofac-check"
            ],
            expected_compliant=True
        )

        result = self.run_scenario(scenario)
        self.assert_scenario_passed(result)

    def test_high_risk_scenario(self):
        """Test high-risk transaction scenario."""
        scenario = TestScenario(
            name="High Risk Wire Transfer",
            transaction={
                "id": "wire-002",
                "type": "wire_transfer",
                "amount": 100000,
                "currency": "USD",
                "source": {"country": "US"},
                "destination": {"country": "IR"},  # Sanctioned country
                "customer": {
                    "is_pep": True,
                    "risk_score": 85
                }
            },
            expected_compliant=False,
            expected_violations=[
                {"rule": "sanctions/country-check", "severity": "critical"},
                {"rule": "pep/enhanced-due-diligence", "severity": "high"}
            ]
        )

        result = self.run_scenario(scenario)
        self.assert_scenario_passed(result)
```

### Testing with Mock Services

```python
from pact.testing import MockServer, ServiceMock

class TestExternalIntegrations:
    """Test integrations with external services."""

    @pytest.fixture
    def sanctions_mock(self):
        """Mock sanctions checking service."""
        mock = ServiceMock("sanctions-api")
        mock.when(
            method="POST",
            path="/check",
            body_contains={"entity_id": "sanctioned-entity"}
        ).respond(
            status=200,
            body={"match": True, "lists": ["OFAC"]}
        )
        mock.when(
            method="POST",
            path="/check"
        ).respond(
            status=200,
            body={"match": False, "lists": []}
        )
        return mock

    def test_sanctions_integration(self, sanctions_mock):
        """Test sanctions check integration."""
        with sanctions_mock:
            engine = create_test_engine(
                plugins={
                    "sanctions": {"api_url": sanctions_mock.url}
                }
            )

            # Test sanctioned entity
            result = engine.evaluate({
                "transaction": {
                    "sender": {"entity_id": "sanctioned-entity"}
                }
            })
            assert not result.compliant
            assert "sanctions" in [v.rule_id for v in result.violations]

            # Test clean entity
            result = engine.evaluate({
                "transaction": {
                    "sender": {"entity_id": "clean-entity"}
                }
            })
            # Sanctions check should pass
            assert not any(
                "sanctions" in v.rule_id
                for v in result.violations
            )
```

### Contract Testing

```python
from pact.testing import ContractTest, Provider, Consumer

class TestAPIContract(ContractTest):
    """Contract tests for ARKA API."""

    @pytest.fixture
    def provider(self):
        return Provider(
            name="arka-engine",
            url="http://localhost:8080"
        )

    @pytest.fixture
    def consumer(self):
        return Consumer(
            name="payment-service"
        )

    def test_evaluation_contract(self, provider, consumer):
        """Test evaluation API contract."""
        # Define expected interaction
        consumer.upon_receiving("an evaluation request") \
            .with_request(
                method="POST",
                path="/api/v1/evaluate",
                headers={"Content-Type": "application/json"},
                body={
                    "transaction": {
                        "id": "txn-123",
                        "amount": 10000,
                        "type": "payment"
                    },
                    "jurisdictions": ["US"],
                    "domains": ["AML"]
                }
            ) \
            .will_respond_with(
                status=200,
                headers={"Content-Type": "application/json"},
                body={
                    "evaluation_id": like("eval-123"),
                    "compliant": boolean(),
                    "violations": each_like({
                        "rule_id": like("rule/aml/test"),
                        "severity": like("medium"),
                        "description": like("Test violation")
                    }),
                    "proof": {
                        "proof_id": like("proof-123"),
                        "hash": regex(r"[a-f0-9]{64}")
                    }
                }
            )

        # Verify contract
        with provider:
            result = consumer.verify()
            assert result.success
```

## Performance Testing

### Load Testing Rules

```python
import asyncio
from pact.testing import LoadTestRunner, LoadProfile

class TestRulePerformance:
    """Performance tests for rules."""

    @pytest.fixture
    def load_runner(self):
        return LoadTestRunner(
            endpoint="http://localhost:8080",
            api_key="test-key"
        )

    async def test_throughput(self, load_runner):
        """Test evaluation throughput."""
        profile = LoadProfile(
            duration_seconds=60,
            ramp_up_seconds=10,
            target_rps=1000,
            concurrent_users=50
        )

        transaction_generator = lambda: {
            "id": f"txn-{uuid.uuid4()}",
            "type": "payment",
            "amount": random.randint(100, 100000),
            "currency": "USD"
        }

        results = await load_runner.run(
            profile=profile,
            request_generator=transaction_generator,
            jurisdictions=["US"],
            domains=["AML"]
        )

        # Assert performance requirements
        assert results.avg_latency_ms < 100
        assert results.p99_latency_ms < 500
        assert results.success_rate > 0.99
        assert results.actual_rps >= profile.target_rps * 0.95

    async def test_latency_under_load(self, load_runner):
        """Test latency distribution under load."""
        profile = LoadProfile(
            duration_seconds=120,
            target_rps=500,
            concurrent_users=25
        )

        results = await load_runner.run(
            profile=profile,
            request_generator=lambda: generate_random_transaction(),
            jurisdictions=["US"],
            domains=["AML"]
        )

        # Check latency percentiles
        assert results.p50_latency_ms < 50
        assert results.p90_latency_ms < 100
        assert results.p95_latency_ms < 200
        assert results.p99_latency_ms < 500

    async def test_spike_handling(self, load_runner):
        """Test handling of traffic spikes."""
        profile = LoadProfile(
            phases=[
                {"duration": 30, "rps": 100},   # Normal load
                {"duration": 10, "rps": 1000},  # Spike
                {"duration": 30, "rps": 100},   # Back to normal
            ]
        )

        results = await load_runner.run(
            profile=profile,
            request_generator=lambda: generate_random_transaction(),
            jurisdictions=["US"],
            domains=["AML"]
        )

        # System should handle spike without failures
        assert results.success_rate > 0.98
        assert results.max_latency_ms < 5000
```

### Benchmarking Rules

```python
from pact.testing import RuleBenchmark

class TestRuleBenchmarks:
    """Benchmark tests for rule evaluation."""

    def setup_method(self):
        self.benchmark = RuleBenchmark()

    def test_simple_rule_benchmark(self):
        """Benchmark simple rule evaluation."""
        rule = """
            rule simple_check {
                when {
                    transaction.amount > 10000
                }
                then {
                    flag_for_review()
                }
            }
        """

        result = self.benchmark.run(
            rule=rule,
            iterations=10000,
            transaction={"amount": 15000}
        )

        print(f"Simple rule: {result.avg_us:.2f}µs average")
        assert result.avg_us < 100  # Should be very fast

    def test_complex_rule_benchmark(self):
        """Benchmark complex rule with external lookups."""
        rule = """
            rule complex_check {
                when {
                    let sanctions = plugin.sanctions.check(transaction.sender.id)
                    let risk = plugin.risk.calculate(transaction)

                    sanctions.match == true OR
                    (risk.score > 80 AND transaction.amount > 50000)
                }
                then {
                    block(reason: "High risk")
                    notify(channel: "critical-alerts")
                }
            }
        """

        result = self.benchmark.run(
            rule=rule,
            iterations=1000,
            transaction={
                "amount": 75000,
                "sender": {"id": "user-123"}
            },
            mock_plugins=True
        )

        print(f"Complex rule: {result.avg_ms:.2f}ms average")
        assert result.avg_ms < 50

    def test_rule_set_benchmark(self):
        """Benchmark full rule set evaluation."""
        result = self.benchmark.run_rule_set(
            rule_path="rules/aml/",
            iterations=1000,
            transaction=generate_sample_transaction()
        )

        print(f"Rule set ({result.rule_count} rules): {result.avg_ms:.2f}ms")
        assert result.avg_ms < 200
```

## Regression Testing

### Rule Regression Suite

```python
from pact.testing import RegressionSuite, GoldenFile

class TestRuleRegression(RegressionSuite):
    """Regression tests for rule behavior."""

    @pytest.fixture(autouse=True)
    def setup_golden_files(self):
        """Load golden files for regression testing."""
        self.golden = GoldenFile("tests/golden/")

    def test_evaluation_consistency(self):
        """Test that evaluations match golden file results."""
        test_cases = self.golden.load("evaluation_cases.json")

        for case in test_cases:
            result = self.evaluate(
                transaction=case["transaction"],
                jurisdictions=case["jurisdictions"],
                domains=case["domains"]
            )

            # Compare with expected result
            self.assert_matches_golden(
                result,
                case["expected"],
                case_id=case["id"]
            )

    def test_proof_determinism(self):
        """Test that proofs are deterministic."""
        transaction = {
            "id": "txn-determinism-test",
            "amount": 15000,
            "type": "wire"
        }

        # Evaluate same transaction multiple times
        results = [
            self.evaluate(transaction, ["US"], ["AML"])
            for _ in range(10)
        ]

        # All proof hashes should be identical
        proof_hashes = [r.proof.hash for r in results]
        assert len(set(proof_hashes)) == 1

    def test_backward_compatibility(self):
        """Test backward compatibility with previous rule versions."""
        # Load transactions evaluated with previous version
        historical_data = self.golden.load("v1.2.0_evaluations.json")

        for case in historical_data:
            result = self.evaluate(
                transaction=case["transaction"],
                jurisdictions=case["jurisdictions"],
                domains=case["domains"]
            )

            # Compliance decision should be consistent
            assert result.compliant == case["expected_compliant"], \
                f"Regression in case {case['id']}: " \
                f"expected {case['expected_compliant']}, got {result.compliant}"
```

### Change Impact Testing

```python
from pact.testing import ChangeImpactTest, RuleDiff

class TestRuleChanges:
    """Test impact of rule changes."""

    def test_rule_change_impact(self):
        """Test impact of proposed rule change."""
        diff = RuleDiff(
            old_rule="rules/aml/threshold.pact",
            new_rule="rules/aml/threshold_v2.pact"
        )

        # Load historical transactions
        historical = self.load_historical_transactions(
            start_date="2024-01-01",
            end_date="2024-03-31",
            sample_size=10000
        )

        impact = ChangeImpactTest.analyze(
            diff=diff,
            transactions=historical
        )

        print(f"Impact Analysis:")
        print(f"  Transactions affected: {impact.affected_count}")
        print(f"  New violations: {impact.new_violations}")
        print(f"  Removed violations: {impact.removed_violations}")
        print(f"  Changed severity: {impact.severity_changes}")

        # Assert acceptable impact
        assert impact.false_positive_delta < 0.05  # <5% change in false positives
        assert impact.false_negative_delta < 0.01  # <1% change in false negatives
```

## Test Fixtures and Utilities

### Transaction Generators

```python
from pact.testing.generators import TransactionGenerator, CustomerGenerator

# Create transaction generator
txn_gen = TransactionGenerator(
    seed=42,  # For reproducibility
    profiles={
        "normal": {
            "amount_range": (100, 10000),
            "types": ["payment", "ach"],
            "weight": 0.8
        },
        "high_value": {
            "amount_range": (50000, 500000),
            "types": ["wire"],
            "weight": 0.15
        },
        "suspicious": {
            "amount_range": (9000, 9999),  # Structuring
            "types": ["cash"],
            "weight": 0.05
        }
    }
)

# Generate test transactions
transactions = txn_gen.generate(count=1000)

# Generate specific scenarios
structuring_scenario = txn_gen.generate_scenario(
    scenario="structuring",
    customer_id="user-123",
    transaction_count=10,
    time_window_hours=24
)
```

### Mock Data Providers

```python
from pact.testing.mocks import MockDataProvider

class TestWithMockData:
    """Tests using mock data providers."""

    @pytest.fixture
    def mock_data(self):
        provider = MockDataProvider()

        # Configure mock sanctions data
        provider.register_sanctions([
            {"entity_id": "sanctioned-001", "list": "OFAC", "type": "individual"},
            {"entity_id": "sanctioned-002", "list": "UN", "type": "organization"},
        ])

        # Configure mock PEP data
        provider.register_peps([
            {"entity_id": "pep-001", "role": "Government Official", "country": "XX"},
        ])

        # Configure mock customer data
        provider.register_customers([
            {"id": "cust-001", "risk_score": 25, "kyc_status": "verified"},
            {"id": "cust-002", "risk_score": 75, "kyc_status": "pending"},
        ])

        return provider

    def test_with_mock_data(self, mock_data):
        engine = create_test_engine(data_provider=mock_data)

        result = engine.evaluate({
            "transaction": {
                "sender": {"entity_id": "sanctioned-001"}
            }
        })

        assert not result.compliant
```

## Test Configuration

### pytest Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

markers =
    unit: Unit tests (fast, no external dependencies)
    integration: Integration tests (may require services)
    e2e: End-to-end tests (require full system)
    performance: Performance tests
    regression: Regression tests

addopts =
    --strict-markers
    -v
    --tb=short

filterwarnings =
    ignore::DeprecationWarning
```

### Test Environment Setup

```yaml
# docker-compose.test.yaml
version: '3.8'

services:
  arka-engine:
    image: pact/engine:test
    ports:
      - "8080:8080"
    environment:
      - ARKA_MODE=test
      - LOG_LEVEL=debug
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 5

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: pact_test
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact

  redis:
    image: redis:7-alpine

  mock-services:
    build:
      context: ./tests/mocks
    ports:
      - "9090:9090"
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -e ".[test]"

      - name: Run unit tests
        run: pytest tests/unit -m unit --cov=pact --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: pact_test
          POSTGRES_PASSWORD: pact
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3

      - name: Run integration tests
        run: |
          docker-compose -f docker-compose.test.yaml up -d
          pytest tests/integration -m integration
          docker-compose down

  performance-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3

      - name: Run performance tests
        run: pytest tests/performance -m performance --benchmark-json=benchmark.json

      - name: Store benchmark result
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'pytest'
          output-file-path: benchmark.json
```

## Best Practices

### Test Organization

1. **Mirror source structure**: Test files should mirror source layout
2. **Clear naming**: Test names should describe the scenario
3. **Isolation**: Tests should be independent and isolated
4. **Determinism**: Tests should produce consistent results

### Test Coverage

1. **Happy path**: Test normal successful scenarios
2. **Edge cases**: Test boundary conditions
3. **Error cases**: Test error handling
4. **Security cases**: Test for security vulnerabilities

### Test Performance

1. **Fast unit tests**: Keep unit tests under 1 second
2. **Parallel execution**: Run tests in parallel where possible
3. **Mock external services**: Don't call real external services in tests
4. **Selective test runs**: Use markers to run specific test subsets

## Related Documentation

- [Rule Authoring Guide](./rule-authoring.md)
- [Plugin Development Guide](./plugin-development.md)
- [Integration Patterns](./integration-patterns.md)
- [CI/CD Integration](../deployment/deployment-guide.md)
