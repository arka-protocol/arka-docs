# ARKA Engine Automated Test Suites

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Quality Assurance

---

## Overview

ARKA Engine employs a comprehensive testing strategy with multiple layers of automated tests to ensure reliability, correctness, and performance. This document covers unit tests, integration tests, end-to-end tests, and testing best practices.

---

## Table of Contents

1. [Testing Strategy](#testing-strategy)
2. [Unit Tests](#unit-tests)
3. [Integration Tests](#integration-tests)
4. [End-to-End Tests](#end-to-end-tests)
5. [Contract Tests](#contract-tests)
6. [Performance Tests](#performance-tests)
7. [Security Tests](#security-tests)
8. [Test Infrastructure](#test-infrastructure)
9. [CI/CD Integration](#cicd-integration)
10. [Best Practices](#best-practices)

---

## Testing Strategy

### Testing Pyramid

```
                    ┌─────────────┐
                    │    E2E      │  ← Fewer, slower, high confidence
                    │   Tests     │
                    ├─────────────┤
                   ╱               ╲
                  ╱   Integration   ╲  ← Moderate count, test boundaries
                 ╱      Tests        ╲
                ├─────────────────────┤
               ╱                       ╲
              ╱       Unit Tests        ╲  ← Many, fast, isolated
             ╱                           ╲
            └─────────────────────────────┘
```

### Coverage Requirements

| Test Level | Coverage Target | Execution Time | Run Frequency |
|------------|-----------------|----------------|---------------|
| Unit | ≥ 80% | < 5 minutes | Every commit |
| Integration | ≥ 70% | < 15 minutes | Every PR |
| E2E | Critical paths | < 30 minutes | Pre-deploy |
| Performance | Benchmarks | < 1 hour | Weekly |
| Security | OWASP Top 10 | < 2 hours | Weekly |

### Test Categories

```yaml
test_categories:
  smoke:
    description: "Quick sanity checks"
    duration: "< 2 minutes"
    scope: "Critical paths only"
    when: "Every deployment"

  regression:
    description: "Full regression suite"
    duration: "< 30 minutes"
    scope: "All functionality"
    when: "Every PR merge"

  nightly:
    description: "Comprehensive test suite"
    duration: "< 2 hours"
    scope: "All tests + performance"
    when: "Daily at midnight"

  release:
    description: "Full validation suite"
    duration: "< 4 hours"
    scope: "All tests + security + compliance"
    when: "Before each release"
```

---

## Unit Tests

### Framework & Tools

| Package | Testing Framework | Coverage Tool | Mocking |
|---------|-------------------|---------------|---------|
| arka-core | Vitest | c8/istanbul | vitest-mock |
| arka-rtvm | Vitest | c8/istanbul | vitest-mock |
| arka-ai-governance | Vitest | c8/istanbul | vitest-mock |
| arka-blockchain | Vitest | c8/istanbul | vitest-mock |
| arka-plugin-sdk | Vitest | c8/istanbul | vitest-mock |

### Test Structure

```
packages/arka-core/
├── src/
│   ├── rules/
│   │   ├── engine.ts
│   │   └── parser.ts
│   └── utils/
│       └── validation.ts
└── tests/
    ├── unit/
    │   ├── rules/
    │   │   ├── engine.test.ts
    │   │   └── parser.test.ts
    │   └── utils/
    │       └── validation.test.ts
    ├── fixtures/
    │   └── sample-rules.json
    └── helpers/
        └── test-utils.ts
```

### Unit Test Example

```typescript
// tests/unit/rules/engine.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { RuleEngine } from '../../../src/rules/engine';
import { Rule, RuleContext, RuleResult } from '../../../src/types';

describe('RuleEngine', () => {
  let engine: RuleEngine;

  beforeEach(() => {
    engine = new RuleEngine({
      maxExecutionTime: 5000,
      enableCaching: false,
    });
  });

  describe('execute()', () => {
    it('should execute a simple threshold rule', async () => {
      const rule: Rule = {
        id: 'rule_001',
        name: 'High Value Transaction',
        version: '1.0.0',
        definition: {
          type: 'condition',
          condition: {
            field: 'transaction.amount',
            operator: 'gt',
            value: 10000,
          },
          actions: [
            { type: 'flag', severity: 'high' },
          ],
        },
      };

      const context: RuleContext = {
        transaction: {
          id: 'txn_123',
          amount: 50000,
          currency: 'USD',
        },
      };

      const result = await engine.execute(rule, context);

      expect(result.status).toBe('triggered');
      expect(result.actions).toHaveLength(1);
      expect(result.actions[0].type).toBe('flag');
      expect(result.actions[0].severity).toBe('high');
    });

    it('should not trigger when condition is not met', async () => {
      const rule: Rule = {
        id: 'rule_002',
        name: 'High Value Transaction',
        version: '1.0.0',
        definition: {
          type: 'condition',
          condition: {
            field: 'transaction.amount',
            operator: 'gt',
            value: 10000,
          },
          actions: [],
        },
      };

      const context: RuleContext = {
        transaction: {
          id: 'txn_124',
          amount: 5000,
          currency: 'USD',
        },
      };

      const result = await engine.execute(rule, context);

      expect(result.status).toBe('passed');
      expect(result.actions).toHaveLength(0);
    });

    it('should handle nested conditions with AND logic', async () => {
      const rule: Rule = {
        id: 'rule_003',
        name: 'High Value International Transfer',
        version: '1.0.0',
        definition: {
          type: 'compound',
          operator: 'and',
          conditions: [
            {
              field: 'transaction.amount',
              operator: 'gt',
              value: 10000,
            },
            {
              field: 'transaction.sender.country',
              operator: 'neq',
              value: 'transaction.recipient.country',
            },
          ],
          actions: [
            { type: 'flag', severity: 'high' },
          ],
        },
      };

      const context: RuleContext = {
        transaction: {
          id: 'txn_125',
          amount: 50000,
          currency: 'USD',
          sender: { country: 'US' },
          recipient: { country: 'GB' },
        },
      };

      const result = await engine.execute(rule, context);

      expect(result.status).toBe('triggered');
    });

    it('should timeout for long-running rules', async () => {
      const slowEngine = new RuleEngine({
        maxExecutionTime: 100,
        enableCaching: false,
      });

      const rule: Rule = {
        id: 'rule_slow',
        name: 'Slow Rule',
        version: '1.0.0',
        definition: {
          type: 'custom',
          handler: async () => {
            await new Promise(resolve => setTimeout(resolve, 500));
            return { triggered: true };
          },
        },
      };

      await expect(slowEngine.execute(rule, {}))
        .rejects.toThrow('Rule execution timeout');
    });

    it('should cache rule compilation results', async () => {
      const cachingEngine = new RuleEngine({
        maxExecutionTime: 5000,
        enableCaching: true,
      });

      const rule: Rule = {
        id: 'rule_cached',
        name: 'Cached Rule',
        version: '1.0.0',
        definition: {
          type: 'condition',
          condition: {
            field: 'value',
            operator: 'eq',
            value: true,
          },
          actions: [],
        },
      };

      // First execution - compiles rule
      await cachingEngine.execute(rule, { value: true });

      // Second execution - uses cache
      const compileSpy = vi.spyOn(cachingEngine as any, 'compileRule');
      await cachingEngine.execute(rule, { value: true });

      expect(compileSpy).not.toHaveBeenCalled();
    });
  });

  describe('validate()', () => {
    it('should validate correct rule syntax', () => {
      const rule: Rule = {
        id: 'rule_valid',
        name: 'Valid Rule',
        version: '1.0.0',
        definition: {
          type: 'condition',
          condition: {
            field: 'amount',
            operator: 'gt',
            value: 0,
          },
          actions: [],
        },
      };

      const validation = engine.validate(rule);

      expect(validation.valid).toBe(true);
      expect(validation.errors).toHaveLength(0);
    });

    it('should reject invalid operator', () => {
      const rule: Rule = {
        id: 'rule_invalid',
        name: 'Invalid Rule',
        version: '1.0.0',
        definition: {
          type: 'condition',
          condition: {
            field: 'amount',
            operator: 'invalid_op' as any,
            value: 0,
          },
          actions: [],
        },
      };

      const validation = engine.validate(rule);

      expect(validation.valid).toBe(false);
      expect(validation.errors).toContainEqual(
        expect.objectContaining({
          path: 'definition.condition.operator',
          message: expect.stringContaining('Invalid operator'),
        })
      );
    });
  });
});
```

### Running Unit Tests

```bash
# Run all unit tests
pnpm run test:unit

# Run with coverage
pnpm run test:unit --coverage

# Run specific test file
pnpm run test:unit -- packages/arka-core/tests/unit/rules/engine.test.ts

# Run tests matching pattern
pnpm run test:unit -- --grep "RuleEngine"

# Watch mode for development
pnpm run test:unit --watch

# Generate coverage report
pnpm run test:unit --coverage --coverage.reporter=html
```

### Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    include: ['**/tests/unit/**/*.test.ts'],
    exclude: ['**/node_modules/**', '**/dist/**'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.d.ts',
        '**/*.config.*',
        '**/types/**',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
    testTimeout: 10000,
    hookTimeout: 10000,
    pool: 'threads',
    poolOptions: {
      threads: {
        singleThread: false,
        isolate: true,
      },
    },
  },
});
```

---

## Integration Tests

### Test Scope

Integration tests verify interactions between components:

- Database operations
- Cache interactions
- Message queue publishing/consuming
- External service integrations
- API endpoint behavior

### Test Infrastructure

```yaml
# docker-compose.test.yml
version: '3.9'

services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: pact_test
      POSTGRES_PASSWORD: test_password
      POSTGRES_DB: pact_test
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data

  redis-test:
    image: redis:7-alpine
    ports:
      - "6380:6379"

  kafka-test:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-test:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-test:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9093:9092"
    depends_on:
      - zookeeper-test

  zookeeper-test:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
```

### Integration Test Examples

```typescript
// tests/integration/compliance/compliance-check.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { createTestClient, TestClient } from '../../helpers/test-client';
import { seedTestData, cleanupTestData } from '../../helpers/test-data';
import { Transaction, ComplianceResult } from '../../../src/types';

describe('Compliance Check Integration', () => {
  let client: TestClient;

  beforeAll(async () => {
    client = await createTestClient({
      database: process.env.TEST_DATABASE_URL,
      redis: process.env.TEST_REDIS_URL,
    });
    await client.connect();
  });

  afterAll(async () => {
    await client.disconnect();
  });

  beforeEach(async () => {
    await cleanupTestData(client);
    await seedTestData(client, {
      rules: ['aml-basic', 'sanctions-screening'],
      customers: 10,
      watchlistEntries: 100,
    });
  });

  describe('Single Transaction Check', () => {
    it('should pass compliant transaction through all rules', async () => {
      const transaction: Transaction = {
        id: 'txn_compliant_001',
        type: 'wire_transfer',
        amount: 5000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: {
          id: 'cust_001',
          name: 'John Doe',
          country: 'US',
        },
        recipient: {
          id: 'cust_002',
          name: 'Jane Smith',
          country: 'US',
        },
      };

      const result = await client.compliance.check({
        transaction,
        rulesets: ['aml-basic'],
      });

      expect(result.status).toBe('COMPLIANT');
      expect(result.violations).toHaveLength(0);
      expect(result.rulesExecuted).toBeGreaterThan(0);
    });

    it('should flag high-value transaction', async () => {
      const transaction: Transaction = {
        id: 'txn_high_value_001',
        type: 'wire_transfer',
        amount: 50000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: {
          id: 'cust_001',
          name: 'John Doe',
          country: 'US',
        },
        recipient: {
          id: 'cust_002',
          name: 'Jane Smith',
          country: 'GB',
        },
      };

      const result = await client.compliance.check({
        transaction,
        rulesets: ['aml-basic'],
      });

      expect(result.status).toBe('NON_COMPLIANT');
      expect(result.violations.length).toBeGreaterThan(0);
      expect(result.violations[0].severity).toBe('high');
    });

    it('should detect sanctioned entity', async () => {
      // Add test entity to watchlist
      await client.watchlist.addEntry({
        name: 'Blocked Entity Corp',
        type: 'entity',
        listType: 'sanctions',
        programs: ['OFAC-SDN'],
      });

      const transaction: Transaction = {
        id: 'txn_sanctioned_001',
        type: 'wire_transfer',
        amount: 1000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: {
          id: 'cust_001',
          name: 'John Doe',
          country: 'US',
        },
        recipient: {
          id: 'ext_001',
          name: 'Blocked Entity Corp',
          country: 'RU',
        },
      };

      const result = await client.compliance.check({
        transaction,
        rulesets: ['sanctions-screening'],
      });

      expect(result.status).toBe('NON_COMPLIANT');
      expect(result.violations).toContainEqual(
        expect.objectContaining({
          category: 'sanctions',
          severity: 'critical',
        })
      );
    });

    it('should create audit trail for compliance check', async () => {
      const transaction: Transaction = {
        id: 'txn_audit_001',
        type: 'wire_transfer',
        amount: 5000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: { id: 'cust_001', name: 'John Doe', country: 'US' },
        recipient: { id: 'cust_002', name: 'Jane Smith', country: 'US' },
      };

      const result = await client.compliance.check({
        transaction,
        rulesets: ['aml-basic'],
        options: { includeAuditTrail: true },
      });

      expect(result.auditTrail).toBeDefined();
      expect(result.auditTrail.id).toBeTruthy();
      expect(result.auditTrail.hash).toBeTruthy();

      // Verify audit entry was persisted
      const auditEntry = await client.audit.get(result.auditTrail.id);
      expect(auditEntry).toBeDefined();
      expect(auditEntry.action).toBe('compliance.check');
      expect(auditEntry.resourceId).toBe(transaction.id);
    });
  });

  describe('Batch Transaction Check', () => {
    it('should process batch of transactions', async () => {
      const transactions = Array.from({ length: 10 }, (_, i) => ({
        id: `txn_batch_${i}`,
        type: 'wire_transfer',
        amount: 1000 + i * 1000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: { id: 'cust_001', name: 'John Doe', country: 'US' },
        recipient: { id: 'cust_002', name: 'Jane Smith', country: 'US' },
      }));

      const results = await client.compliance.batchCheck({
        transactions,
        rulesets: ['aml-basic'],
        options: { parallelism: 5 },
      });

      expect(results).toHaveLength(10);
      results.forEach((result, i) => {
        expect(result.transactionId).toBe(`txn_batch_${i}`);
      });
    });

    it('should continue processing on individual failures', async () => {
      const transactions = [
        {
          id: 'txn_valid',
          type: 'wire_transfer',
          amount: 1000,
          currency: 'USD',
          timestamp: new Date().toISOString(),
          sender: { id: 'cust_001', name: 'John Doe', country: 'US' },
          recipient: { id: 'cust_002', name: 'Jane Smith', country: 'US' },
        },
        {
          id: 'txn_invalid',
          type: 'wire_transfer',
          amount: -100, // Invalid amount
          currency: 'USD',
          timestamp: new Date().toISOString(),
          sender: { id: 'cust_001', name: 'John Doe', country: 'US' },
          recipient: { id: 'cust_002', name: 'Jane Smith', country: 'US' },
        },
      ];

      const results = await client.compliance.batchCheck({
        transactions,
        rulesets: ['aml-basic'],
        options: { continueOnError: true },
      });

      expect(results).toHaveLength(2);
      expect(results[0].status).toBe('COMPLIANT');
      expect(results[1].status).toBe('ERROR');
    });
  });

  describe('Caching Behavior', () => {
    it('should cache rule compilation', async () => {
      const transaction: Transaction = {
        id: 'txn_cache_001',
        type: 'wire_transfer',
        amount: 5000,
        currency: 'USD',
        timestamp: new Date().toISOString(),
        sender: { id: 'cust_001', name: 'John Doe', country: 'US' },
        recipient: { id: 'cust_002', name: 'Jane Smith', country: 'US' },
      };

      // First call - cache miss
      const start1 = Date.now();
      await client.compliance.check({ transaction, rulesets: ['aml-basic'] });
      const duration1 = Date.now() - start1;

      // Second call - cache hit
      const start2 = Date.now();
      await client.compliance.check({
        transaction: { ...transaction, id: 'txn_cache_002' },
        rulesets: ['aml-basic'],
      });
      const duration2 = Date.now() - start2;

      // Second call should be faster due to caching
      expect(duration2).toBeLessThan(duration1);
    });
  });
});
```

### Database Integration Tests

```typescript
// tests/integration/database/rules-repository.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { RulesRepository } from '../../../src/repositories/rules';
import { createTestDatabase, TestDatabase } from '../../helpers/test-database';

describe('RulesRepository', () => {
  let db: TestDatabase;
  let repository: RulesRepository;

  beforeAll(async () => {
    db = await createTestDatabase();
    repository = new RulesRepository(db.pool);
  });

  afterAll(async () => {
    await db.close();
  });

  beforeEach(async () => {
    await db.truncate(['rules', 'rule_versions']);
  });

  describe('create()', () => {
    it('should create a new rule', async () => {
      const rule = await repository.create({
        name: 'Test Rule',
        category: 'aml',
        severity: 'high',
        definition: {
          type: 'condition',
          condition: { field: 'amount', operator: 'gt', value: 10000 },
          actions: [],
        },
      });

      expect(rule.id).toBeTruthy();
      expect(rule.version).toBe('1.0.0');
      expect(rule.status).toBe('draft');
    });

    it('should enforce unique rule names within tenant', async () => {
      await repository.create({
        name: 'Unique Rule',
        category: 'aml',
        severity: 'high',
        definition: { type: 'condition', condition: {}, actions: [] },
      });

      await expect(
        repository.create({
          name: 'Unique Rule',
          category: 'aml',
          severity: 'high',
          definition: { type: 'condition', condition: {}, actions: [] },
        })
      ).rejects.toThrow('Rule name already exists');
    });
  });

  describe('update()', () => {
    it('should create new version on update', async () => {
      const rule = await repository.create({
        name: 'Versioned Rule',
        category: 'aml',
        severity: 'high',
        definition: {
          type: 'condition',
          condition: { field: 'amount', operator: 'gt', value: 10000 },
          actions: [],
        },
      });

      const updated = await repository.update(rule.id, {
        definition: {
          type: 'condition',
          condition: { field: 'amount', operator: 'gt', value: 20000 },
          actions: [],
        },
      });

      expect(updated.version).toBe('1.1.0');

      const versions = await repository.getVersions(rule.id);
      expect(versions).toHaveLength(2);
    });
  });

  describe('query()', () => {
    beforeEach(async () => {
      // Seed test data
      await Promise.all([
        repository.create({ name: 'AML Rule 1', category: 'aml', severity: 'high', definition: {}, status: 'active' }),
        repository.create({ name: 'AML Rule 2', category: 'aml', severity: 'medium', definition: {}, status: 'active' }),
        repository.create({ name: 'Sanctions Rule', category: 'sanctions', severity: 'critical', definition: {}, status: 'active' }),
        repository.create({ name: 'Draft Rule', category: 'aml', severity: 'low', definition: {}, status: 'draft' }),
      ]);
    });

    it('should filter by category', async () => {
      const rules = await repository.query({ category: 'aml' });
      expect(rules.data).toHaveLength(3);
      rules.data.forEach(rule => expect(rule.category).toBe('aml'));
    });

    it('should filter by status', async () => {
      const rules = await repository.query({ status: 'active' });
      expect(rules.data).toHaveLength(3);
    });

    it('should paginate results', async () => {
      const page1 = await repository.query({ limit: 2, offset: 0 });
      const page2 = await repository.query({ limit: 2, offset: 2 });

      expect(page1.data).toHaveLength(2);
      expect(page2.data).toHaveLength(2);
      expect(page1.data[0].id).not.toBe(page2.data[0].id);
    });
  });
});
```

### Running Integration Tests

```bash
# Start test infrastructure
docker-compose -f docker-compose.test.yml up -d

# Wait for services to be ready
pnpm run test:wait-for-services

# Run integration tests
pnpm run test:integration

# Run with specific database
TEST_DATABASE_URL=postgresql://user:pass@localhost:5433/test pnpm run test:integration

# Cleanup
docker-compose -f docker-compose.test.yml down -v
```

---

## End-to-End Tests

### E2E Test Framework

ARKA Engine uses Playwright for E2E testing of APIs and UI components.

```typescript
// tests/e2e/api/compliance-workflow.test.ts
import { test, expect } from '@playwright/test';
import { ApiClient } from '../../helpers/api-client';

test.describe('Compliance Workflow E2E', () => {
  let api: ApiClient;

  test.beforeAll(async () => {
    api = new ApiClient({
      baseUrl: process.env.E2E_API_URL || 'http://localhost:8080',
      apiKey: process.env.E2E_API_KEY,
    });
  });

  test('complete compliance check workflow', async () => {
    // 1. Create a new rule
    const rule = await api.rules.create({
      name: 'E2E Test Rule',
      category: 'aml',
      severity: 'high',
      definition: {
        type: 'condition',
        condition: {
          field: 'transaction.amount',
          operator: 'gt',
          value: 10000,
        },
        actions: [
          { type: 'flag', severity: 'high' },
        ],
      },
    });
    expect(rule.id).toBeTruthy();

    // 2. Activate the rule
    const activatedRule = await api.rules.activate(rule.id);
    expect(activatedRule.status).toBe('active');

    // 3. Perform compliance check
    const result = await api.compliance.check({
      transaction: {
        id: 'e2e_txn_001',
        type: 'wire_transfer',
        amount: 50000,
        currency: 'USD',
        sender: { id: 'cust_001', name: 'Test Sender', country: 'US' },
        recipient: { id: 'cust_002', name: 'Test Recipient', country: 'GB' },
      },
      rulesets: ['aml'],
    });

    expect(result.status).toBe('NON_COMPLIANT');
    expect(result.violations).toContainEqual(
      expect.objectContaining({
        ruleId: rule.id,
      })
    );

    // 4. Verify audit trail
    const auditEntry = await api.audit.get(result.auditTrail.id);
    expect(auditEntry.action).toBe('compliance.check');

    // 5. Cleanup
    await api.rules.delete(rule.id);
  });

  test('webhook delivery on violation', async () => {
    // Create webhook endpoint (using mock server)
    const webhookUrl = `${process.env.E2E_WEBHOOK_MOCK}/compliance-alerts`;

    const webhook = await api.webhooks.create({
      name: 'E2E Test Webhook',
      url: webhookUrl,
      events: ['compliance.violated'],
      secret: 'test-secret',
    });

    // Trigger a violation
    await api.compliance.check({
      transaction: {
        id: 'e2e_webhook_txn',
        type: 'wire_transfer',
        amount: 100000,
        currency: 'USD',
        sender: { id: 'cust_001', name: 'Test Sender', country: 'US' },
        recipient: { id: 'cust_002', name: 'Test Recipient', country: 'CU' }, // Sanctioned country
      },
      rulesets: ['sanctions'],
    });

    // Wait for webhook delivery
    await expect.poll(
      async () => {
        const deliveries = await api.webhooks.getDeliveries(webhook.id);
        return deliveries.find(d => d.status === 'delivered');
      },
      { timeout: 30000 }
    ).toBeTruthy();

    // Cleanup
    await api.webhooks.delete(webhook.id);
  });
});
```

### E2E Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['json', { outputFile: 'test-results.json' }],
    ['junit', { outputFile: 'junit-results.xml' }],
  ],
  use: {
    baseURL: process.env.E2E_BASE_URL || 'http://localhost:8080',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'api',
      testMatch: /.*api.*\.test\.ts/,
    },
    {
      name: 'ui-chromium',
      testMatch: /.*ui.*\.test\.ts/,
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'ui-firefox',
      testMatch: /.*ui.*\.test\.ts/,
      use: { ...devices['Desktop Firefox'] },
    },
  ],
  globalSetup: require.resolve('./tests/e2e/global-setup.ts'),
  globalTeardown: require.resolve('./tests/e2e/global-teardown.ts'),
});
```

### Running E2E Tests

```bash
# Run all E2E tests
pnpm run test:e2e

# Run API tests only
pnpm run test:e2e -- --project=api

# Run with UI
pnpm run test:e2e -- --ui

# Run specific test
pnpm run test:e2e -- tests/e2e/api/compliance-workflow.test.ts

# Debug mode
pnpm run test:e2e -- --debug
```

---

## Contract Tests

### API Contract Testing with Pact

```typescript
// tests/contract/consumer/compliance-api.pact.ts
import { pactWith } from 'jest-pact';
import { Matchers } from '@arka-foundation/pact';

const { like, eachLike, iso8601DateTime } = Matchers;

pactWith(
  {
    consumer: 'ComplianceConsumer',
    provider: 'ArkaEngineAPI',
  },
  (interaction) => {
    interaction('compliance check request', ({ provider, execute }) => {
      beforeEach(() =>
        provider.addInteraction({
          state: 'rules exist for aml-basic ruleset',
          uponReceiving: 'a request to check compliance',
          withRequest: {
            method: 'POST',
            path: '/api/v1/compliance/check',
            headers: {
              'Content-Type': 'application/json',
              Authorization: like('Bearer token'),
            },
            body: {
              transaction: {
                id: like('txn_123'),
                type: like('wire_transfer'),
                amount: like(5000),
                currency: like('USD'),
              },
              rulesets: eachLike('aml-basic'),
            },
          },
          willRespondWith: {
            status: 200,
            headers: {
              'Content-Type': 'application/json',
            },
            body: {
              id: like('cc_abc123'),
              status: like('COMPLIANT'),
              score: like(95),
              violations: [],
              warnings: eachLike({
                id: like('warn_001'),
                ruleId: like('rule_001'),
                message: like('Warning message'),
                severity: like('low'),
              }),
              rulesExecuted: like(10),
              executionTime: like(150),
              auditTrail: {
                id: like('audit_xyz'),
                timestamp: iso8601DateTime(),
                hash: like('sha256:abc123'),
              },
            },
          },
        })
      );

      execute('performs compliance check', async (mockServer) => {
        const client = new ComplianceClient({
          baseUrl: mockServer.url,
          apiKey: 'test-token',
        });

        const result = await client.check({
          transaction: {
            id: 'txn_123',
            type: 'wire_transfer',
            amount: 5000,
            currency: 'USD',
          },
          rulesets: ['aml-basic'],
        });

        expect(result.status).toBe('COMPLIANT');
      });
    });
  }
);
```

---

## Test Infrastructure

### Test Data Factories

```typescript
// tests/helpers/factories.ts
import { faker } from '@faker-js/faker';
import { Transaction, Customer, Rule } from '../../src/types';

export const transactionFactory = {
  build(overrides: Partial<Transaction> = {}): Transaction {
    return {
      id: faker.string.uuid(),
      type: faker.helpers.arrayElement(['wire_transfer', 'ach', 'card_payment']),
      amount: faker.number.float({ min: 100, max: 100000, fractionDigits: 2 }),
      currency: faker.helpers.arrayElement(['USD', 'EUR', 'GBP']),
      timestamp: faker.date.recent().toISOString(),
      sender: {
        id: faker.string.uuid(),
        name: faker.person.fullName(),
        country: faker.location.countryCode(),
      },
      recipient: {
        id: faker.string.uuid(),
        name: faker.company.name(),
        country: faker.location.countryCode(),
      },
      ...overrides,
    };
  },

  buildList(count: number, overrides: Partial<Transaction> = {}): Transaction[] {
    return Array.from({ length: count }, () => this.build(overrides));
  },

  highValue(overrides: Partial<Transaction> = {}): Transaction {
    return this.build({
      amount: faker.number.float({ min: 50000, max: 1000000, fractionDigits: 2 }),
      ...overrides,
    });
  },

  international(overrides: Partial<Transaction> = {}): Transaction {
    const senderCountry = faker.location.countryCode();
    let recipientCountry = faker.location.countryCode();
    while (recipientCountry === senderCountry) {
      recipientCountry = faker.location.countryCode();
    }

    return this.build({
      sender: { ...this.build().sender, country: senderCountry },
      recipient: { ...this.build().recipient, country: recipientCountry },
      ...overrides,
    });
  },
};

export const ruleFactory = {
  build(overrides: Partial<Rule> = {}): Rule {
    return {
      id: `rule_${faker.string.alphanumeric(8)}`,
      name: faker.lorem.words(3),
      version: '1.0.0',
      status: 'active',
      category: faker.helpers.arrayElement(['aml', 'sanctions', 'kyc', 'fraud']),
      severity: faker.helpers.arrayElement(['low', 'medium', 'high', 'critical']),
      definition: {
        type: 'condition',
        condition: {
          field: 'transaction.amount',
          operator: 'gt',
          value: faker.number.int({ min: 1000, max: 100000 }),
        },
        actions: [
          { type: 'flag', severity: 'high' },
        ],
      },
      ...overrides,
    };
  },
};
```

### Test Utilities

```typescript
// tests/helpers/test-utils.ts
import { expect } from 'vitest';

export async function waitFor<T>(
  fn: () => Promise<T>,
  options: { timeout?: number; interval?: number } = {}
): Promise<T> {
  const { timeout = 5000, interval = 100 } = options;
  const start = Date.now();

  while (Date.now() - start < timeout) {
    try {
      return await fn();
    } catch (error) {
      await new Promise(resolve => setTimeout(resolve, interval));
    }
  }

  throw new Error(`waitFor timed out after ${timeout}ms`);
}

export function expectTimestamp(value: string) {
  expect(value).toMatch(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}/);
}

export function expectUUID(value: string) {
  expect(value).toMatch(
    /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
  );
}

export async function measureExecutionTime<T>(
  fn: () => Promise<T>
): Promise<{ result: T; duration: number }> {
  const start = performance.now();
  const result = await fn();
  const duration = performance.now() - start;
  return { result, duration };
}
```

---

## CI/CD Integration

### GitHub Actions Test Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm run test:unit --coverage

      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: pact_test
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: pact_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile
      - run: pnpm run db:migrate
        env:
          DATABASE_URL: postgresql://pact_test:test_password@localhost:5432/pact_test

      - run: pnpm run test:integration
        env:
          DATABASE_URL: postgresql://pact_test:test_password@localhost:5432/pact_test
          REDIS_URL: redis://localhost:6379

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v2
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      - name: Start services
        run: docker-compose -f docker-compose.test.yml up -d

      - name: Wait for services
        run: pnpm run test:wait-for-services

      - name: Install Playwright
        run: pnpm exec playwright install --with-deps

      - name: Run E2E tests
        run: pnpm run test:e2e

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.test.yml down -v
```

---

## Best Practices

### Test Naming Conventions

```typescript
// Good: Descriptive, behavior-focused
describe('RuleEngine', () => {
  describe('execute()', () => {
    it('should return triggered status when threshold is exceeded', () => {});
    it('should include all matched actions in result', () => {});
    it('should timeout after maxExecutionTime milliseconds', () => {});
  });
});

// Bad: Implementation-focused
describe('RuleEngine', () => {
  it('test execute', () => {});
  it('works', () => {});
});
```

### Arrange-Act-Assert Pattern

```typescript
it('should flag transaction when amount exceeds threshold', async () => {
  // Arrange
  const engine = new RuleEngine();
  const rule = ruleFactory.build({
    definition: {
      type: 'condition',
      condition: { field: 'amount', operator: 'gt', value: 10000 },
      actions: [{ type: 'flag' }],
    },
  });
  const transaction = transactionFactory.build({ amount: 50000 });

  // Act
  const result = await engine.execute(rule, { transaction });

  // Assert
  expect(result.status).toBe('triggered');
  expect(result.actions).toContainEqual(expect.objectContaining({ type: 'flag' }));
});
```

### Test Isolation

```typescript
describe('RulesRepository', () => {
  let repository: RulesRepository;

  // Setup fresh repository for each test
  beforeEach(async () => {
    repository = new RulesRepository(testDb.pool);
    await testDb.truncate(['rules']); // Clean slate
  });

  // Tests are independent
  it('test 1', async () => { /* ... */ });
  it('test 2', async () => { /* ... */ });
});
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Engineering Team | Initial release |

**Review Schedule:** Quarterly
**Next Review:** March 2025
