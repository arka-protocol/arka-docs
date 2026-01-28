# Plugin Framework

The ARKA Plugin Framework enables extensibility through modular, isolated plugins that extend the core compliance engine with domain-specific logic, custom integrations, and specialized functionality.

## Table of Contents

- [Overview](#overview)
- [Plugin Types](#plugin-types)
- [Plugin Development](#plugin-development)
- [Plugin SDK](#plugin-sdk)
- [Plugin Lifecycle](#plugin-lifecycle)
- [Sandbox Security](#sandbox-security)
- [Plugin Configuration](#plugin-configuration)
- [Best Practices](#best-practices)

---

## Overview

### Why Plugins?

The plugin framework addresses key challenges:

| Challenge | Solution |
|-----------|----------|
| Domain diversity | Domain-specific plugins (AML, KYC, GDPR) |
| Custom integrations | Data connector plugins |
| Specialized reporting | Reporter plugins |
| Organization-specific logic | Custom rule executor plugins |
| Third-party extensions | Plugin marketplace |

### Plugin Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PLUGIN FRAMEWORK                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                     PLUGIN MANAGER                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │ │
│  │  │   Loader     │  │   Registry   │  │  Lifecycle   │              │ │
│  │  │              │  │              │  │   Manager    │              │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                    │                                     │
│                                    ▼                                     │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                      PLUGIN HOST                                    │ │
│  │                                                                     │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │                    SANDBOX LAYER                             │   │ │
│  │  │                                                              │   │ │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │   │ │
│  │  │  │Sandbox 1│  │Sandbox 2│  │Sandbox 3│  │Sandbox N│        │   │ │
│  │  │  │         │  │         │  │         │  │         │        │   │ │
│  │  │  │┌───────┐│  │┌───────┐│  │┌───────┐│  │┌───────┐│        │   │ │
│  │  │  ││Plugin ││  ││Plugin ││  ││Plugin ││  ││Plugin ││        │   │ │
│  │  │  │└───────┘│  │└───────┘│  │└───────┘│  │└───────┘│        │   │ │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  │                                                                     │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │                    IPC LAYER                                 │   │ │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │ │
│  │  │  │  Message     │  │  Service     │  │  Event       │       │   │ │
│  │  │  │  Router      │  │  Proxy       │  │  Bus         │       │   │ │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘       │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Plugin Types

### Rule Executor Plugins

Execute domain-specific compliance logic.

```typescript
interface RuleExecutorPlugin extends BasePlugin {
  pluginType: 'RULE_EXECUTOR';

  // Execute a compiled rule against input
  executeRule(
    rule: CompiledRule,
    input: RuleInput,
    context: ExecutionContext
  ): Promise<RuleExecutionResult>;

  // Get supported rule types
  getSupportedRuleTypes(): RuleType[];

  // Validate a rule before compilation
  validateRule(rule: RuleDefinition): ValidationResult;
}
```

**Example**: AML Transaction Monitoring Plugin

```typescript
class AMLPlugin implements RuleExecutorPlugin {
  pluginType = 'RULE_EXECUTOR' as const;

  async executeRule(rule, input, context) {
    const transaction = input.transaction;

    // Transaction amount check
    if (rule.conditions.includes('amount_threshold')) {
      const threshold = rule.parameters.threshold;
      if (transaction.amount > threshold) {
        return {
          flags: [{
            type: 'AMOUNT_THRESHOLD_EXCEEDED',
            severity: 'high',
            value: transaction.amount,
            threshold: threshold
          }]
        };
      }
    }

    // Sanctions screening
    if (rule.conditions.includes('sanctions_check')) {
      const sanctionsResult = await this.screenSanctions(
        transaction.counterparty
      );
      if (sanctionsResult.matched) {
        return {
          flags: [{
            type: 'SANCTIONS_MATCH',
            severity: 'critical',
            matchedList: sanctionsResult.list,
            matchScore: sanctionsResult.score
          }]
        };
      }
    }

    return { flags: [], compliant: true };
  }
}
```

### Data Connector Plugins

Connect to external data sources.

```typescript
interface DataConnectorPlugin extends BasePlugin {
  pluginType: 'DATA_CONNECTOR';
  connectionType: 'REST_API' | 'DATABASE' | 'FILE' | 'STREAM';

  // Establish connection
  connect(config: ConnectionConfig): Promise<Connection>;

  // Query data
  query(connection: Connection, query: DataQuery): Promise<QueryResult>;

  // Stream data
  stream(connection: Connection, query: DataQuery): AsyncIterable<DataRecord>;

  // Get data schema
  getSchema(connection: Connection): Promise<DataSchema>;
}
```

**Example**: REST API Connector

```typescript
class RestApiConnector implements DataConnectorPlugin {
  pluginType = 'DATA_CONNECTOR' as const;
  connectionType = 'REST_API' as const;

  async connect(config) {
    return {
      baseUrl: config.baseUrl,
      headers: {
        'Authorization': `Bearer ${config.apiKey}`,
        'Content-Type': 'application/json'
      }
    };
  }

  async query(connection, query) {
    const response = await fetch(
      `${connection.baseUrl}${query.endpoint}`,
      {
        method: query.method || 'GET',
        headers: connection.headers,
        body: query.body ? JSON.stringify(query.body) : undefined
      }
    );

    return response.json();
  }
}
```

### Reporter Plugins

Generate compliance reports and filings.

```typescript
interface ReporterPlugin extends BasePlugin {
  pluginType: 'REPORTER';
  supportedFormats: ReportFormat[];

  // Generate report
  generateReport(
    template: ReportTemplate,
    data: ReportData,
    options: ReportOptions
  ): Promise<GeneratedReport>;

  // Deliver report
  deliverReport(
    report: GeneratedReport,
    destination: DeliveryDestination
  ): Promise<DeliveryResult>;
}
```

**Example**: XBRL Reporter

```typescript
class XBRLReporter implements ReporterPlugin {
  pluginType = 'REPORTER' as const;
  supportedFormats = ['XBRL', 'iXBRL'];

  async generateReport(template, data, options) {
    const taxonomy = await this.loadTaxonomy(template.taxonomyUrl);

    const xbrlDoc = new XBRLDocument();
    xbrlDoc.setTaxonomy(taxonomy);

    for (const fact of data.facts) {
      xbrlDoc.addFact({
        concept: fact.concept,
        value: fact.value,
        context: this.createContext(fact),
        unit: fact.unit
      });
    }

    return {
      format: 'XBRL',
      content: xbrlDoc.serialize(),
      metadata: {
        taxonomy: template.taxonomyUrl,
        factCount: data.facts.length
      }
    };
  }
}
```

### Validator Plugins

Validate input data or rule definitions.

```typescript
interface ValidatorPlugin extends BasePlugin {
  pluginType: 'VALIDATOR';

  // Validate input data
  validate(
    data: unknown,
    schema: ValidationSchema
  ): Promise<ValidationResult>;

  // Get supported schemas
  getSupportedSchemas(): SchemaDescriptor[];
}
```

### Transform Plugins

Transform data between formats.

```typescript
interface TransformPlugin extends BasePlugin {
  pluginType: 'TRANSFORM';

  // Transform data
  transform(
    input: unknown,
    mapping: TransformMapping
  ): Promise<unknown>;

  // Get supported transformations
  getSupportedTransforms(): TransformDescriptor[];
}
```

---

## Plugin Development

### Project Structure

```
my-plugin/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # Plugin entry point
│   ├── plugin.ts         # Main plugin class
│   ├── handlers/         # Request handlers
│   │   ├── execute.ts
│   │   └── validate.ts
│   └── utils/            # Utility functions
├── config/
│   └── schema.json       # Configuration schema
├── test/
│   ├── plugin.test.ts
│   └── fixtures/
└── dist/                 # Compiled output
```

### Plugin Manifest

```json
{
  "name": "@arka/aml-plugin",
  "version": "1.0.0",
  "arka": {
    "pluginType": "RULE_EXECUTOR",
    "apiVersion": "1.0",
    "entryPoint": "dist/index.js",
    "capabilities": {
      "domains": ["AML", "BSA"],
      "jurisdictions": ["US", "EU"],
      "features": [
        "transaction_monitoring",
        "sanctions_screening",
        "risk_scoring"
      ]
    },
    "dependencies": {
      "arka-core": "^1.0.0"
    },
    "resources": {
      "memory": "256MB",
      "cpu": "0.5"
    },
    "permissions": [
      "network:sanctions-api.example.com",
      "storage:read",
      "config:read"
    ]
  }
}
```

### Implementing a Plugin

```typescript
// src/index.ts
import { Plugin, PluginContext } from '@arka/plugin-sdk';
import { MyPlugin } from './plugin';

export default function createPlugin(): Plugin {
  return new MyPlugin();
}
```

```typescript
// src/plugin.ts
import {
  BasePlugin,
  PluginContext,
  PluginConfig,
  HealthStatus,
  PluginMetrics
} from '@arka/plugin-sdk';

export class MyPlugin implements BasePlugin {
  readonly pluginId = 'com.example.my-plugin';
  readonly name = 'My Custom Plugin';
  readonly version = '1.0.0';
  readonly vendor = {
    name: 'Example Corp',
    url: 'https://example.com'
  };

  private context: PluginContext;
  private config: MyPluginConfig;

  async initialize(context: PluginContext): Promise<void> {
    this.context = context;
    this.context.logger.info('Initializing plugin...');

    // Load configuration
    this.config = await context.config.get<MyPluginConfig>();

    // Initialize resources
    await this.initializeResources();
  }

  async start(): Promise<void> {
    this.context.logger.info('Starting plugin...');
    // Start background tasks, open connections, etc.
  }

  async stop(): Promise<void> {
    this.context.logger.info('Stopping plugin...');
    // Clean up resources
  }

  async dispose(): Promise<void> {
    // Final cleanup
  }

  getHealth(): HealthStatus {
    return {
      status: 'healthy',
      details: {
        lastCheck: new Date().toISOString(),
        connections: 'ok'
      }
    };
  }

  getMetrics(): PluginMetrics {
    return {
      requestsTotal: this.requestCount,
      errorsTotal: this.errorCount,
      latencyMs: {
        p50: 10,
        p95: 50,
        p99: 100
      }
    };
  }

  getConfigSchema(): ConfigSchema {
    return {
      type: 'object',
      properties: {
        apiEndpoint: { type: 'string', format: 'uri' },
        timeout: { type: 'number', default: 30000 },
        retries: { type: 'number', default: 3 }
      },
      required: ['apiEndpoint']
    };
  }

  async configure(config: PluginConfig): Promise<void> {
    this.config = config as MyPluginConfig;
    await this.reconfigure();
  }

  validateConfig(config: PluginConfig): ValidationResult {
    // Validate configuration
    const errors = [];

    if (!config.apiEndpoint) {
      errors.push({ field: 'apiEndpoint', message: 'Required' });
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  getCapabilities(): PluginCapabilities {
    return {
      supportedDomains: ['AML'],
      supportedJurisdictions: ['US', 'EU'],
      features: ['transaction_monitoring']
    };
  }

  getDependencies(): PluginDependency[] {
    return [];
  }
}
```

---

## Plugin SDK

### Installation

```bash
npm install @arka/plugin-sdk
```

### Core Interfaces

```typescript
// Plugin Context - provided to plugins by the host
interface PluginContext {
  // Logging
  readonly logger: Logger;

  // Configuration
  readonly config: ConfigService;

  // Secrets management
  readonly secrets: SecretsService;

  // Caching
  readonly cache: CacheService;

  // Event bus for pub/sub
  readonly eventBus: EventBus;

  // Plugin registry for inter-plugin communication
  readonly pluginRegistry: PluginRegistry;

  // Core services (rate-limited)
  readonly ruleEngine: RuleEngineAPI;
  readonly auditLog: AuditLogAPI;

  // Resource constraints
  readonly resourceLimits: ResourceLimits;
  readonly permissions: PermissionSet;
}
```

### Logging

```typescript
// Structured logging
context.logger.info('Processing request', {
  requestId: '123',
  operation: 'evaluate'
});

context.logger.error('Operation failed', {
  error: err.message,
  stack: err.stack
});

// Log levels: debug, info, warn, error
```

### Configuration

```typescript
// Get typed configuration
const config = await context.config.get<MyConfig>();

// Watch for changes
context.config.onChange((newConfig) => {
  this.reconfigure(newConfig);
});

// Get specific value with default
const timeout = await context.config.get('timeout', 30000);
```

### Secrets

```typescript
// Get secret value
const apiKey = await context.secrets.get('api_key');

// Secrets are never logged
// Secrets can be rotated without plugin restart
```

### Caching

```typescript
// Set cache value
await context.cache.set('key', value, { ttl: 3600 });

// Get cache value
const cached = await context.cache.get<MyType>('key');

// Delete cache value
await context.cache.delete('key');
```

### Events

```typescript
// Subscribe to events
context.eventBus.subscribe('rule.created', (event) => {
  this.handleRuleCreated(event);
});

// Publish events
context.eventBus.publish('my-plugin.processed', {
  inputId: '123',
  result: 'success'
});
```

---

## Plugin Lifecycle

### State Machine

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PLUGIN LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                         ┌───────────┐                                    │
│                         │DISCOVERED │                                    │
│                         └─────┬─────┘                                    │
│                               │ load()                                   │
│                               ▼                                          │
│                         ┌───────────┐                                    │
│                         │  LOADED   │                                    │
│                         └─────┬─────┘                                    │
│                               │ validate()                               │
│                               ▼                                          │
│                         ┌───────────┐                                    │
│                         │ VALIDATED │                                    │
│                         └─────┬─────┘                                    │
│                               │ initialize()                             │
│                               ▼                                          │
│                         ┌───────────┐                                    │
│              ┌─────────▶│INITIALIZED│◀─────────┐                        │
│              │          └─────┬─────┘          │                        │
│              │                │ start()        │ stop()                 │
│              │                ▼                │                        │
│              │          ┌───────────┐          │                        │
│    restart() │          │  RUNNING  │──────────┘                        │
│              │          └─────┬─────┘                                    │
│              │                │ error                                    │
│              │                ▼                                          │
│              │          ┌───────────┐                                    │
│              └──────────│  ERRORED  │                                    │
│                         └─────┬─────┘                                    │
│                               │ unload()                                 │
│                               ▼                                          │
│                         ┌───────────┐                                    │
│                         │ UNLOADED  │                                    │
│                         └───────────┘                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Lifecycle Hooks

```typescript
class MyPlugin implements BasePlugin {
  // Called when plugin is first loaded
  async initialize(context: PluginContext): Promise<void> {
    // Setup resources, connections, etc.
  }

  // Called when plugin should begin processing
  async start(): Promise<void> {
    // Start accepting requests
  }

  // Called for graceful shutdown
  async stop(): Promise<void> {
    // Stop accepting new requests
    // Finish in-flight requests
  }

  // Called for final cleanup
  async dispose(): Promise<void> {
    // Release all resources
    // Close all connections
  }

  // Called when configuration changes (optional)
  async onConfigurationChanged?(config: PluginConfig): Promise<void> {
    // Apply new configuration
  }
}
```

### Hot Reload

Plugins can be updated without system restart:

```bash
# Deploy new plugin version
arka-cli plugin upgrade my-plugin --version 1.1.0

# Process:
# 1. Load new version in parallel sandbox
# 2. Initialize new version
# 3. Migrate state from old to new
# 4. Atomic switch: route requests to new version
# 5. Drain old version
# 6. Unload old version
```

---

## Sandbox Security

### Isolation Model

Each plugin runs in an isolated sandbox:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SANDBOX ISOLATION                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                        HOST PROCESS                                 │ │
│  │                                                                     │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │                    SANDBOX SUPERVISOR                        │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  │                              │                                      │ │
│  │         ┌────────────────────┼────────────────────┐                │ │
│  │         │                    │                    │                │ │
│  │         ▼                    ▼                    ▼                │ │
│  │  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐        │ │
│  │  │  SANDBOX A  │      │  SANDBOX B  │      │  SANDBOX C  │        │ │
│  │  │             │      │             │      │             │        │ │
│  │  │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │        │ │
│  │  │ │ Plugin  │ │      │ │ Plugin  │ │      │ │ Plugin  │ │        │ │
│  │  │ │    A    │ │      │ │    B    │ │      │ │    C    │ │        │ │
│  │  │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │        │ │
│  │  │             │      │             │      │             │        │ │
│  │  │ Memory: 256M│      │ Memory: 512M│      │ Memory: 256M│        │ │
│  │  │ CPU: 0.5    │      │ CPU: 1.0    │      │ CPU: 0.5    │        │ │
│  │  │ Network: Y  │      │ Network: N  │      │ Network: Y  │        │ │
│  │  └─────────────┘      └─────────────┘      └─────────────┘        │ │
│  │         │                    │                    │                │ │
│  │         └────────────────────┼────────────────────┘                │ │
│  │                              │                                      │ │
│  │                     IPC (encrypted)                                │ │
│  │                                                                     │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Resource Limits

```yaml
# Sandbox resource configuration
sandbox:
  resources:
    memory:
      limit: 256MB
      swap: 0
    cpu:
      shares: 512
      quota: 50000  # 50% of one core
    io:
      read_bps: 100MB/s
      write_bps: 50MB/s
    network:
      bandwidth_in: 10MB/s
      bandwidth_out: 10MB/s
```

### Permission Model

```yaml
# Plugin permissions
permissions:
  # Network access
  network:
    allowed_hosts:
      - "api.example.com"
      - "*.internal.corp"
    allowed_ports: [443, 8443]

  # File system access
  filesystem:
    read:
      - "/config/*"
      - "/data/shared/*"
    write:
      - "/data/plugin-specific/*"

  # Service access
  services:
    - "cache:read,write"
    - "config:read"
    - "secrets:read"
    - "events:publish,subscribe"

  # Prohibited
  denied:
    - "shell:*"
    - "process:*"
    - "filesystem:/etc/*"
```

### Security Scanning

Plugins are scanned before deployment:

```bash
# Security scan results
arka-cli plugin scan my-plugin.tar.gz

# Output:
# Scanning plugin artifact...
#
# Vulnerability Scan:
# ├── Critical: 0
# ├── High: 0
# ├── Medium: 2
# │   ├── CVE-2024-1234: lodash < 4.17.21 (dependency)
# │   └── CVE-2024-5678: axios < 1.6.0 (dependency)
# └── Low: 5
#
# Permission Analysis:
# ├── Network: api.example.com:443 (approved)
# ├── Storage: read:/config, write:/data/plugin
# └── No elevated permissions requested
#
# Code Analysis:
# ├── No hardcoded secrets detected
# ├── No unsafe eval() usage
# └── No process spawning
#
# Result: PASS (with warnings)
```

---

## Plugin Configuration

### Configuration Schema

Define configuration schema in your plugin:

```typescript
getConfigSchema(): ConfigSchema {
  return {
    type: 'object',
    properties: {
      apiEndpoint: {
        type: 'string',
        format: 'uri',
        description: 'API endpoint URL'
      },
      apiKey: {
        type: 'string',
        secret: true,  // Marked as secret
        description: 'API authentication key'
      },
      timeout: {
        type: 'number',
        default: 30000,
        minimum: 1000,
        maximum: 120000,
        description: 'Request timeout in milliseconds'
      },
      retryPolicy: {
        type: 'object',
        properties: {
          maxRetries: { type: 'number', default: 3 },
          backoffMs: { type: 'number', default: 1000 }
        }
      },
      features: {
        type: 'object',
        properties: {
          sanctionsScreening: { type: 'boolean', default: true },
          riskScoring: { type: 'boolean', default: true }
        }
      }
    },
    required: ['apiEndpoint', 'apiKey']
  };
}
```

### Runtime Configuration

```yaml
# Plugin configuration in ARKA config
plugins:
  aml-plugin:
    enabled: true
    version: "1.0.0"
    config:
      apiEndpoint: "https://api.sanctions.example.com"
      apiKey: "${SANCTIONS_API_KEY}"  # Environment variable
      timeout: 30000
      retryPolicy:
        maxRetries: 3
        backoffMs: 1000
      features:
        sanctionsScreening: true
        riskScoring: true
```

### Configuration Validation

```typescript
validateConfig(config: PluginConfig): ValidationResult {
  const errors: ValidationError[] = [];

  // Validate API endpoint
  try {
    new URL(config.apiEndpoint);
  } catch {
    errors.push({
      field: 'apiEndpoint',
      message: 'Must be a valid URL'
    });
  }

  // Validate timeout range
  if (config.timeout < 1000 || config.timeout > 120000) {
    errors.push({
      field: 'timeout',
      message: 'Must be between 1000 and 120000 ms'
    });
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

---

## Best Practices

### Error Handling

```typescript
async executeRule(rule, input, context) {
  try {
    const result = await this.processRule(rule, input);
    return result;
  } catch (error) {
    // Log error with context
    this.context.logger.error('Rule execution failed', {
      ruleId: rule.id,
      error: error.message,
      stack: error.stack
    });

    // Return structured error
    if (error instanceof ValidationError) {
      return {
        success: false,
        error: {
          type: 'VALIDATION_ERROR',
          message: error.message,
          details: error.details
        }
      };
    }

    // Re-throw unexpected errors
    throw error;
  }
}
```

### Performance

```typescript
class OptimizedPlugin implements BasePlugin {
  private cache: Map<string, CachedResult>;
  private connectionPool: ConnectionPool;

  async initialize(context) {
    // Use connection pooling
    this.connectionPool = new ConnectionPool({
      min: 2,
      max: 10,
      idleTimeout: 30000
    });

    // Initialize cache
    this.cache = new Map();
  }

  async executeRule(rule, input, context) {
    // Check cache first
    const cacheKey = this.computeCacheKey(rule, input);
    const cached = this.cache.get(cacheKey);
    if (cached && !this.isExpired(cached)) {
      return cached.result;
    }

    // Get connection from pool
    const conn = await this.connectionPool.acquire();
    try {
      const result = await this.doExecute(conn, rule, input);

      // Cache result
      this.cache.set(cacheKey, {
        result,
        timestamp: Date.now()
      });

      return result;
    } finally {
      // Always release connection
      this.connectionPool.release(conn);
    }
  }
}
```

### Testing

```typescript
// test/plugin.test.ts
import { createTestContext } from '@arka/plugin-sdk/testing';
import { MyPlugin } from '../src/plugin';

describe('MyPlugin', () => {
  let plugin: MyPlugin;
  let context: TestPluginContext;

  beforeEach(async () => {
    context = createTestContext({
      config: {
        apiEndpoint: 'https://test.example.com',
        apiKey: 'test-key'
      }
    });

    plugin = new MyPlugin();
    await plugin.initialize(context);
    await plugin.start();
  });

  afterEach(async () => {
    await plugin.stop();
    await plugin.dispose();
  });

  it('should execute rule successfully', async () => {
    const result = await plugin.executeRule(
      mockRule,
      mockInput,
      mockContext
    );

    expect(result.success).toBe(true);
    expect(result.flags).toHaveLength(0);
  });

  it('should flag high-risk transactions', async () => {
    const result = await plugin.executeRule(
      mockRule,
      { ...mockInput, amount: 1000000 },
      mockContext
    );

    expect(result.flags).toContainEqual(
      expect.objectContaining({
        type: 'HIGH_RISK_AMOUNT'
      })
    );
  });
});
```

### Documentation

Include comprehensive documentation:

```markdown
# My Plugin

## Overview
Brief description of what the plugin does.

## Configuration

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| apiEndpoint | string | Yes | - | API endpoint URL |
| timeout | number | No | 30000 | Request timeout |

## Features

### Transaction Monitoring
Description of the feature...

### Sanctions Screening
Description of the feature...

## Examples

### Basic Usage
```yaml
plugins:
  my-plugin:
    config:
      apiEndpoint: "https://api.example.com"
```

## Troubleshooting

### Common Issues
...
```

---

## Next Steps

- [Plugin Development Guide](../guides/plugin-development.md) - Detailed development tutorial
- [Plugin API Reference](../api-reference/plugin-api.md) - Complete API documentation
- [RTVM](./rtvm.md) - Understand the execution engine
- [Distributed Agents](./distributed-agents.md) - Deploy plugins at the edge
