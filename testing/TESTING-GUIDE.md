# ARKA Engine Testing Guide

## Overview

This guide covers the comprehensive testing infrastructure for ARKA Engine. The testing framework provides tools for unit testing, integration testing, and simulation of compliance scenarios.

## Quick Start

```bash
# Run all tests
pnpm test

# Run tests with coverage
pnpm test:coverage

# Run tests in watch mode
pnpm test:watch

# Run tests with UI
pnpm test:ui

# Run full test suite with verbose output
pnpm test:all
```

## Test Infrastructure

### Directory Structure

```
arka-engine/
├── vitest.config.ts           # Root configuration
├── vitest.setup.ts            # Global setup
├── packages/
│   ├── arka-testing/          # Test utilities package
│   │   ├── src/
│   │   │   ├── mocks/         # Mock factories
│   │   │   ├── factories/     # Data factories
│   │   │   ├── fixtures/      # Sample test data
│   │   │   └── helpers/       # Test helper utilities
│   │   └── vitest.config.ts
│   ├── arka-utils/
│   │   └── src/*.test.ts      # Unit tests
│   ├── arka-crypto/
│   │   └── src/*.test.ts      # Unit tests
│   └── ...
└── plugins/
    └── plugin-*/
        └── src/*.test.ts      # Plugin tests
```

### Configuration

The root `vitest.config.ts` provides:
- Global test configuration
- Coverage settings (v8 provider)
- Path aliases for workspace packages
- Reporter configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: [
      'packages/*/src/**/*.test.ts',
      'plugins/*/src/**/*.test.ts',
    ],
    coverage: {
      provider: 'v8',
      thresholds: {
        statements: 60,
        branches: 50,
        functions: 60,
        lines: 60,
      },
    },
  },
});
```

## Using @arka/testing

The `@arka/testing` package provides comprehensive utilities for testing ARKA components.

### Installation

```bash
pnpm add -D @arka/testing
```

### Mock Factories

```typescript
import {
  mockEvent,
  mockEntity,
  mockRule,
  mockDecision,
  mockRuleEvaluation,
} from '@arka/testing';

// Create mock event
const event = mockEvent({
  type: 'WIRE_TRANSFER',
  payload: { amount: 15000 },
});

// Create mock rule
const rule = mockRule({
  name: 'High Value Alert',
  condition: {
    type: 'compare',
    field: 'payload.amount',
    operator: 'gt',
    value: 10000,
  },
  consequence: {
    decision: 'FLAG',
    code: 'HIGH_VALUE',
    message: 'Amount exceeds threshold',
  },
});
```

### Condition & Consequence Builders

```typescript
import { conditions, consequences } from '@arka/testing';

// Build conditions
const amountCheck = conditions.gt('payload.amount', 10000);
const countryCheck = conditions.compare('payload.country', 'in', ['US', 'CA']);
const combined = conditions.and(amountCheck, countryCheck);

// Build consequences
const denyConsequence = consequences.deny('BLOCKED', 'Transaction blocked');
const flagConsequence = consequences.flag('REVIEW', 'Requires review');
```

### Test Scenarios

```typescript
import { createTestScenario, amlScenarios } from '@arka/testing';

// Use pre-built scenarios
const highValueTest = amlScenarios.highValueTransaction();
const sanctionedTest = amlScenarios.sanctionedCountryTransaction();

// Create custom scenarios
const customScenario = createTestScenario({
  name: 'Cross-border high value',
  event: createTransactionEvent({
    amount: 50000,
    country: 'US',
    destinationCountry: 'MX',
  }),
  rules: [amountThresholdRule, crossBorderRule],
  expectedDecision: 'ALLOW_WITH_FLAGS',
  expectedFlags: ['HIGH_VALUE', 'CROSS_BORDER'],
});
```

### Test Fixtures

```typescript
import {
  sampleRules,
  sampleEvents,
  sampleEntities,
  integrationTestData,
  expectedResults,
} from '@arka/testing';

// Use sample rules
const amountRule = sampleRules.amountThreshold;
const sanctionsRule = sampleRules.sanctionedCountries;

// Use sample events
const domesticEvent = sampleEvents.domesticTransfer;
const pepEvent = sampleEvents.pepTransaction;

// Get all test data for integration tests
const { rules, events, entities } = integrationTestData;
```

### Assertion Helpers

```typescript
import {
  expectAllow,
  expectDeny,
  expectFlag,
  expectRulesEvaluated,
  expectRuleTriggered,
} from '@arka/testing';

// Assert decision outcomes
expectAllow(decision);
expectDeny(decision, 'SANCTIONED_COUNTRY');
expectFlag(decision, ['HIGH_VALUE_TXN', 'PEP_INVOLVED']);

// Assert rule evaluations
expectRulesEvaluated(decision, ['rule_001', 'rule_002']);
expectRuleTriggered(decision, 'rule_high_value');
```

### Performance Testing

```typescript
import { measureTime, expectWithinTime, benchmark } from '@arka/testing';

// Measure execution time
const { result, duration } = await measureTime(async () => {
  return await evaluateRules(event, rules);
});
console.log(`Took ${duration}ms`);

// Assert time limit
const result = await expectWithinTime(
  () => evaluateRules(event, rules),
  100 // max 100ms
);

// Run benchmark
const stats = await benchmark(() => evaluateRules(event, rules), 1000);
console.log(`P95: ${stats.p95}ms, P99: ${stats.p99}ms`);
```

## Testing Patterns

### Unit Testing Rules

```typescript
import { describe, it, expect } from 'vitest';
import { mockEvent, expectDeny, expectAllow } from '@arka/testing';

describe('Amount Threshold Rule', () => {
  const rule = createAmountThresholdRule(10000, 'FLAG');

  it('should flag amounts over threshold', async () => {
    const event = mockEvent({ payload: { amount: 15000 } });
    const decision = await evaluateRules(event, [rule]);
    expectFlag(decision, ['AMOUNT_THRESHOLD_EXCEEDED']);
  });

  it('should allow amounts under threshold', async () => {
    const event = mockEvent({ payload: { amount: 5000 } });
    const decision = await evaluateRules(event, [rule]);
    expectAllow(decision);
  });
});
```

### Testing Plugins

```typescript
import { PluginTestHarness } from '@arka/testing';
import { MyPlugin } from './my-plugin';

describe('MyPlugin', () => {
  const plugin = new MyPlugin();
  const harness = new PluginTestHarness(plugin);

  it('should define entity types', () => {
    const result = harness.testEntityTypes();
    expect(result.passed).toBe(true);
  });

  it('should define default rules', () => {
    const result = harness.testDefaultRules();
    expect(result.passed).toBe(true);
  });

  it('should map events correctly', () => {
    const domainEvent = {
      type: 'TRANSACTION',
      payload: { amount: 1000 },
    };
    const result = harness.testEventMapping(domainEvent);
    expect(result.passed).toBe(true);
  });

  it('should pass all tests', () => {
    const results = harness.runAll();
    expect(results.failed).toBe(0);
  });
});
```

### Integration Testing

```typescript
import { integrationTestData, expectedResults } from '@arka/testing';

describe('Rule Engine Integration', () => {
  const { rules, events } = integrationTestData;

  Object.entries(events).forEach(([name, event]) => {
    it(`should produce correct decision for ${name}`, async () => {
      const decision = await evaluateRules(event, rules);
      const expected = expectedResults[name];

      expect(decision.status).toBe(expected.decision);

      if (expected.flags.length > 0) {
        const flagCodes = decision.ruleEvaluations
          .filter(e => e.result === 'FAIL')
          .map(e => e.consequence?.code);
        expect(flagCodes).toEqual(expect.arrayContaining(expected.flags));
      }
    });
  });
});
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 9

      - name: Install dependencies
        run: pnpm install

      - name: Build packages
        run: pnpm build

      - name: Run tests
        run: pnpm test:all

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```

### Pre-commit Hook

```bash
# .husky/pre-commit
pnpm test --changed
```

## Coverage Requirements

### Minimum Thresholds

| Metric | Required | Target |
|--------|----------|--------|
| Statements | 60% | 80% |
| Branches | 50% | 70% |
| Functions | 60% | 80% |
| Lines | 60% | 80% |

### Viewing Coverage

```bash
# Generate coverage report
pnpm test:coverage

# Coverage report available at:
# ./coverage/index.html
```

## Best Practices

### 1. Test Organization

- Group tests by feature/module
- Use descriptive test names
- Keep tests focused and isolated

### 2. Mock External Dependencies

```typescript
import { vi } from 'vitest';

vi.mock('./external-service', () => ({
  fetchData: vi.fn().mockResolvedValue({ data: 'mocked' }),
}));
```

### 3. Clean Up After Tests

```typescript
import { resetMockCounters } from '@arka/testing';

beforeEach(() => {
  resetMockCounters();
});

afterEach(() => {
  vi.clearAllMocks();
});
```

### 4. Test Edge Cases

- Empty inputs
- Boundary values
- Invalid data
- Error conditions

### 5. Use Snapshot Testing Sparingly

```typescript
it('should produce stable output', () => {
  const result = formatDecision(decision);
  expect(result).toMatchSnapshot();
});
```

## Troubleshooting

### Common Issues

**Tests not found:**
```bash
# Check pattern matches
pnpm vitest --reporter=verbose
```

**Type errors in tests:**
```bash
# Ensure @arka/types is built
pnpm --filter @arka/types build
```

**Module resolution errors:**
```bash
# Check vitest.config.ts aliases
```

### Debug Mode

```bash
# Run specific test file
pnpm vitest packages/arka-utils/src/id.test.ts

# Run with debugger
node --inspect-brk ./node_modules/vitest/vitest.mjs run
```

## Resources

- [Vitest Documentation](https://vitest.dev/)
- [@arka/testing API Reference](./packages/arka-testing/README.md)
- [Test Coverage Dashboard](./coverage/index.html)
