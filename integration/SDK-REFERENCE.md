# ARKA Engine SDK Reference

**Version:** 1.0.0
**Last Updated:** December 2024
**Classification:** Technical / Integration

---

## Overview

ARKA Engine provides official SDKs for multiple programming languages, enabling seamless integration with your applications. All SDKs provide consistent APIs for rule execution, compliance checking, audit logging, and event handling.

---

## Supported SDKs

| SDK | Language | Version | Package Manager | Status |
|-----|----------|---------|-----------------|--------|
| [TypeScript/JavaScript](#typescriptjavascript-sdk) | TypeScript, JavaScript | 1.0.0 | npm/yarn/pnpm | Stable |
| [Python](#python-sdk) | Python 3.9+ | 1.0.0 | pip/poetry | Stable |
| [Go](#go-sdk) | Go 1.21+ | 1.0.0 | go modules | Stable |
| [Java](#java-sdk) | Java 17+ | 1.0.0 | Maven/Gradle | Stable |
| [.NET](#net-sdk) | C# .NET 6+ | 1.0.0 | NuGet | Stable |
| [Rust](#rust-sdk) | Rust 1.70+ | 1.0.0 | Cargo | Beta |

---

## TypeScript/JavaScript SDK

### Installation

```bash
# npm
npm install @arka-engine/sdk

# yarn
yarn add @arka-engine/sdk

# pnpm
pnpm add @arka-engine/sdk
```

### Quick Start

```typescript
import { PactClient, Rule, Transaction } from '@arka-engine/sdk';

// Initialize client
const pact = new PactClient({
  apiUrl: 'https://api.arka-engine.io',
  apiKey: process.env.ARKA_API_KEY,
  tenantId: 'your-tenant-id'
});

// Execute a compliance check
async function checkCompliance(transaction: Transaction) {
  const result = await arka.compliance.check({
    transaction,
    rulesets: ['aml-kyc', 'sanctions'],
    options: {
      includeExplanation: true,
      timeout: 5000
    }
  });

  if (result.status === 'COMPLIANT') {
    console.log('Transaction approved');
  } else {
    console.log('Violations:', result.violations);
  }

  return result;
}

// Listen for events
pact.events.on('rule.executed', (event) => {
  console.log('Rule executed:', event.ruleId, event.result);
});

// Graceful shutdown
process.on('SIGTERM', () => pact.disconnect());
```

### API Reference

#### PactClient

```typescript
interface PactClientConfig {
  apiUrl: string;
  apiKey: string;
  tenantId: string;
  timeout?: number;          // Default: 30000ms
  retries?: number;          // Default: 3
  retryDelay?: number;       // Default: 1000ms
  logger?: Logger;           // Custom logger
  tlsConfig?: TLSConfig;     // Custom TLS
}

class PactClient {
  constructor(config: PactClientConfig);

  // Sub-clients
  readonly rules: RulesClient;
  readonly compliance: ComplianceClient;
  readonly audit: AuditClient;
  readonly events: EventsClient;
  readonly webhooks: WebhooksClient;

  // Methods
  async connect(): Promise<void>;
  async disconnect(): Promise<void>;
  async healthCheck(): Promise<HealthStatus>;
}
```

#### Rules Client

```typescript
class RulesClient {
  // List rules
  async list(options?: ListOptions): Promise<PaginatedResponse<Rule>>;

  // Get rule by ID
  async get(ruleId: string): Promise<Rule>;

  // Create rule
  async create(rule: CreateRuleRequest): Promise<Rule>;

  // Update rule
  async update(ruleId: string, updates: UpdateRuleRequest): Promise<Rule>;

  // Delete rule
  async delete(ruleId: string): Promise<void>;

  // Execute rule
  async execute(ruleId: string, input: RuleInput): Promise<RuleResult>;

  // Validate rule syntax
  async validate(ruleDefinition: string): Promise<ValidationResult>;

  // Get rule versions
  async versions(ruleId: string): Promise<RuleVersion[]>;
}

interface Rule {
  id: string;
  name: string;
  version: string;
  status: 'active' | 'inactive' | 'draft';
  category: string;
  jurisdiction: string[];
  definition: RuleDefinition;
  metadata: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
}
```

#### Compliance Client

```typescript
class ComplianceClient {
  // Check compliance
  async check(request: ComplianceRequest): Promise<ComplianceResult>;

  // Batch check
  async batchCheck(requests: ComplianceRequest[]): Promise<ComplianceResult[]>;

  // Get compliance report
  async getReport(reportId: string): Promise<ComplianceReport>;

  // Generate report
  async generateReport(options: ReportOptions): Promise<ComplianceReport>;
}

interface ComplianceRequest {
  transaction: Transaction;
  rulesets: string[];
  options?: {
    includeExplanation?: boolean;
    timeout?: number;
    failFast?: boolean;
    context?: Record<string, unknown>;
  };
}

interface ComplianceResult {
  id: string;
  status: 'COMPLIANT' | 'NON_COMPLIANT' | 'PENDING_REVIEW' | 'ERROR';
  score: number;                      // 0-100
  violations: Violation[];
  warnings: Warning[];
  rulesExecuted: number;
  executionTime: number;              // milliseconds
  explanation?: AIExplanation;
  auditTrail: AuditEntry;
}
```

#### Audit Client

```typescript
class AuditClient {
  // Query audit logs
  async query(query: AuditQuery): Promise<PaginatedResponse<AuditEntry>>;

  // Get specific entry
  async get(entryId: string): Promise<AuditEntry>;

  // Export audit logs
  async export(query: AuditQuery, format: 'json' | 'csv' | 'pdf'): Promise<ExportResult>;

  // Verify audit entry integrity
  async verify(entryId: string): Promise<VerificationResult>;
}

interface AuditQuery {
  startTime?: Date;
  endTime?: Date;
  entityType?: string;
  entityId?: string;
  action?: string;
  userId?: string;
  limit?: number;
  offset?: number;
}
```

---

## Python SDK

### Installation

```bash
# pip
pip install arka-engine-sdk

# poetry
poetry add arka-engine-sdk

# conda
conda install -c arka-engine arka-sdk
```

### Quick Start

```python
from pact_sdk import PactClient, Transaction
from pact_sdk.models import ComplianceRequest
import asyncio

# Initialize client
client = PactClient(
    api_url="https://api.arka-engine.io",
    api_key=os.environ["ARKA_API_KEY"],
    tenant_id="your-tenant-id"
)

async def check_compliance():
    # Create transaction
    transaction = Transaction(
        id="txn-123",
        type="wire_transfer",
        amount=50000.00,
        currency="USD",
        sender={
            "id": "cust-001",
            "name": "John Doe",
            "country": "US"
        },
        recipient={
            "id": "cust-002",
            "name": "Acme Corp",
            "country": "GB"
        }
    )

    # Check compliance
    result = await client.compliance.check(
        ComplianceRequest(
            transaction=transaction,
            rulesets=["aml-kyc", "sanctions", "fatf"],
            options={
                "include_explanation": True,
                "timeout": 5000
            }
        )
    )

    print(f"Status: {result.status}")
    print(f"Score: {result.score}")

    if result.violations:
        for violation in result.violations:
            print(f"Violation: {violation.rule_id} - {violation.message}")

    return result

# Run
if __name__ == "__main__":
    asyncio.run(check_compliance())
```

### API Reference

```python
class PactClient:
    """Main ARKA Engine client."""

    def __init__(
        self,
        api_url: str,
        api_key: str,
        tenant_id: str,
        timeout: int = 30000,
        retries: int = 3,
        logger: Optional[Logger] = None
    ) -> None: ...

    @property
    def rules(self) -> RulesClient: ...

    @property
    def compliance(self) -> ComplianceClient: ...

    @property
    def audit(self) -> AuditClient: ...

    @property
    def events(self) -> EventsClient: ...

    async def connect(self) -> None: ...
    async def disconnect(self) -> None: ...
    async def health_check(self) -> HealthStatus: ...


class RulesClient:
    """Rules management client."""

    async def list(
        self,
        page: int = 1,
        limit: int = 50,
        status: Optional[str] = None,
        category: Optional[str] = None
    ) -> PaginatedResponse[Rule]: ...

    async def get(self, rule_id: str) -> Rule: ...

    async def create(self, rule: CreateRuleRequest) -> Rule: ...

    async def update(
        self,
        rule_id: str,
        updates: UpdateRuleRequest
    ) -> Rule: ...

    async def delete(self, rule_id: str) -> None: ...

    async def execute(
        self,
        rule_id: str,
        input_data: Dict[str, Any]
    ) -> RuleResult: ...


class ComplianceClient:
    """Compliance checking client."""

    async def check(
        self,
        request: ComplianceRequest
    ) -> ComplianceResult: ...

    async def batch_check(
        self,
        requests: List[ComplianceRequest],
        parallel: bool = True
    ) -> List[ComplianceResult]: ...

    async def generate_report(
        self,
        options: ReportOptions
    ) -> ComplianceReport: ...
```

### Type Definitions

```python
from dataclasses import dataclass
from typing import List, Dict, Any, Optional
from datetime import datetime
from enum import Enum

class ComplianceStatus(Enum):
    COMPLIANT = "COMPLIANT"
    NON_COMPLIANT = "NON_COMPLIANT"
    PENDING_REVIEW = "PENDING_REVIEW"
    ERROR = "ERROR"

@dataclass
class Transaction:
    id: str
    type: str
    amount: float
    currency: str
    sender: Dict[str, Any]
    recipient: Dict[str, Any]
    timestamp: Optional[datetime] = None
    metadata: Optional[Dict[str, Any]] = None

@dataclass
class ComplianceResult:
    id: str
    status: ComplianceStatus
    score: float
    violations: List['Violation']
    warnings: List['Warning']
    rules_executed: int
    execution_time: int
    explanation: Optional['AIExplanation'] = None

@dataclass
class Violation:
    rule_id: str
    rule_name: str
    severity: str  # 'critical', 'high', 'medium', 'low'
    message: str
    details: Dict[str, Any]
    remediation: Optional[str] = None
```

---

## Go SDK

### Installation

```bash
go get github.com/arka-engine/arka-sdk-go
```

### Quick Start

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    pact "github.com/arka-engine/arka-sdk-go"
)

func main() {
    // Initialize client
    client, err := pact.NewClient(pact.Config{
        APIURL:   "https://api.arka-engine.io",
        APIKey:   os.Getenv("ARKA_API_KEY"),
        TenantID: "your-tenant-id",
        Timeout:  30 * time.Second,
    })
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    defer client.Close()

    ctx := context.Background()

    // Create transaction
    transaction := &pact.Transaction{
        ID:       "txn-123",
        Type:     "wire_transfer",
        Amount:   50000.00,
        Currency: "USD",
        Sender: pact.Party{
            ID:      "cust-001",
            Name:    "John Doe",
            Country: "US",
        },
        Recipient: pact.Party{
            ID:      "cust-002",
            Name:    "Acme Corp",
            Country: "GB",
        },
    }

    // Check compliance
    result, err := client.Compliance.Check(ctx, &pact.ComplianceRequest{
        Transaction: transaction,
        Rulesets:    []string{"aml-kyc", "sanctions"},
        Options: &pact.CheckOptions{
            IncludeExplanation: true,
            Timeout:            5 * time.Second,
        },
    })
    if err != nil {
        log.Fatalf("Compliance check failed: %v", err)
    }

    fmt.Printf("Status: %s\n", result.Status)
    fmt.Printf("Score: %.2f\n", result.Score)

    for _, v := range result.Violations {
        fmt.Printf("Violation: %s - %s\n", v.RuleID, v.Message)
    }
}
```

### API Reference

```go
package pact

// Client is the main ARKA Engine client
type Client struct {
    Rules      *RulesService
    Compliance *ComplianceService
    Audit      *AuditService
    Events     *EventsService
    Webhooks   *WebhooksService
}

// Config contains client configuration
type Config struct {
    APIURL      string
    APIKey      string
    TenantID    string
    Timeout     time.Duration
    MaxRetries  int
    RetryDelay  time.Duration
    TLSConfig   *tls.Config
    HTTPClient  *http.Client
    Logger      Logger
}

// NewClient creates a new ARKA client
func NewClient(config Config) (*Client, error)

// Close closes the client connection
func (c *Client) Close() error

// HealthCheck verifies the API is available
func (c *Client) HealthCheck(ctx context.Context) (*HealthStatus, error)


// RulesService handles rule operations
type RulesService struct{}

func (s *RulesService) List(ctx context.Context, opts *ListOptions) (*RuleList, error)
func (s *RulesService) Get(ctx context.Context, ruleID string) (*Rule, error)
func (s *RulesService) Create(ctx context.Context, rule *CreateRuleRequest) (*Rule, error)
func (s *RulesService) Update(ctx context.Context, ruleID string, updates *UpdateRuleRequest) (*Rule, error)
func (s *RulesService) Delete(ctx context.Context, ruleID string) error
func (s *RulesService) Execute(ctx context.Context, ruleID string, input map[string]interface{}) (*RuleResult, error)


// ComplianceService handles compliance checks
type ComplianceService struct{}

func (s *ComplianceService) Check(ctx context.Context, req *ComplianceRequest) (*ComplianceResult, error)
func (s *ComplianceService) BatchCheck(ctx context.Context, reqs []*ComplianceRequest) ([]*ComplianceResult, error)
func (s *ComplianceService) GenerateReport(ctx context.Context, opts *ReportOptions) (*ComplianceReport, error)
```

### Types

```go
// Transaction represents a financial transaction
type Transaction struct {
    ID        string                 `json:"id"`
    Type      string                 `json:"type"`
    Amount    float64                `json:"amount"`
    Currency  string                 `json:"currency"`
    Sender    Party                  `json:"sender"`
    Recipient Party                  `json:"recipient"`
    Timestamp time.Time              `json:"timestamp,omitempty"`
    Metadata  map[string]interface{} `json:"metadata,omitempty"`
}

// ComplianceResult contains the result of a compliance check
type ComplianceResult struct {
    ID            string          `json:"id"`
    Status        ComplianceStatus `json:"status"`
    Score         float64         `json:"score"`
    Violations    []Violation     `json:"violations"`
    Warnings      []Warning       `json:"warnings"`
    RulesExecuted int             `json:"rules_executed"`
    ExecutionTime int64           `json:"execution_time"`
    Explanation   *AIExplanation  `json:"explanation,omitempty"`
}

// ComplianceStatus represents the compliance check result
type ComplianceStatus string

const (
    StatusCompliant     ComplianceStatus = "COMPLIANT"
    StatusNonCompliant  ComplianceStatus = "NON_COMPLIANT"
    StatusPendingReview ComplianceStatus = "PENDING_REVIEW"
    StatusError         ComplianceStatus = "ERROR"
)
```

---

## Java SDK

### Installation

**Maven:**
```xml
<dependency>
    <groupId>io.arka-engine</groupId>
    <artifactId>arka-sdk</artifactId>
    <version>1.0.0</version>
</dependency>
```

**Gradle:**
```groovy
implementation 'io.arka-engine:arka-sdk:1.0.0'
```

### Quick Start

```java
import io.pactengine.sdk.*;
import io.pactengine.sdk.models.*;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class PactExample {
    public static void main(String[] args) {
        // Initialize client
        PactClient client = PactClient.builder()
            .apiUrl("https://api.arka-engine.io")
            .apiKey(System.getenv("ARKA_API_KEY"))
            .tenantId("your-tenant-id")
            .timeout(Duration.ofSeconds(30))
            .build();

        try {
            // Create transaction
            Transaction transaction = Transaction.builder()
                .id("txn-123")
                .type("wire_transfer")
                .amount(new BigDecimal("50000.00"))
                .currency("USD")
                .sender(Party.builder()
                    .id("cust-001")
                    .name("John Doe")
                    .country("US")
                    .build())
                .recipient(Party.builder()
                    .id("cust-002")
                    .name("Acme Corp")
                    .country("GB")
                    .build())
                .build();

            // Check compliance
            ComplianceResult result = client.compliance()
                .check(ComplianceRequest.builder()
                    .transaction(transaction)
                    .rulesets(List.of("aml-kyc", "sanctions"))
                    .options(CheckOptions.builder()
                        .includeExplanation(true)
                        .timeout(Duration.ofSeconds(5))
                        .build())
                    .build())
                .get();

            System.out.println("Status: " + result.getStatus());
            System.out.println("Score: " + result.getScore());

            for (Violation v : result.getViolations()) {
                System.out.println("Violation: " + v.getRuleId() + " - " + v.getMessage());
            }

        } finally {
            client.close();
        }
    }
}
```

### API Reference

```java
public class PactClient implements Closeable {

    public static Builder builder() { ... }

    public RulesClient rules();
    public ComplianceClient compliance();
    public AuditClient audit();
    public EventsClient events();
    public WebhooksClient webhooks();

    public CompletableFuture<HealthStatus> healthCheck();
    public void close();

    public static class Builder {
        public Builder apiUrl(String apiUrl);
        public Builder apiKey(String apiKey);
        public Builder tenantId(String tenantId);
        public Builder timeout(Duration timeout);
        public Builder maxRetries(int maxRetries);
        public Builder httpClient(HttpClient httpClient);
        public Builder objectMapper(ObjectMapper objectMapper);
        public PactClient build();
    }
}

public interface RulesClient {
    CompletableFuture<PaginatedResponse<Rule>> list(ListOptions options);
    CompletableFuture<Rule> get(String ruleId);
    CompletableFuture<Rule> create(CreateRuleRequest request);
    CompletableFuture<Rule> update(String ruleId, UpdateRuleRequest request);
    CompletableFuture<Void> delete(String ruleId);
    CompletableFuture<RuleResult> execute(String ruleId, Map<String, Object> input);
}

public interface ComplianceClient {
    CompletableFuture<ComplianceResult> check(ComplianceRequest request);
    CompletableFuture<List<ComplianceResult>> batchCheck(List<ComplianceRequest> requests);
    CompletableFuture<ComplianceReport> generateReport(ReportOptions options);
}
```

---

## .NET SDK

### Installation

```bash
# .NET CLI
dotnet add package ArkaEngine.Sdk

# Package Manager
Install-Package ArkaEngine.Sdk
```

### Quick Start

```csharp
using ArkaEngine.Sdk;
using ArkaEngine.Sdk.Models;

// Initialize client
var client = new PactClient(new PactClientOptions
{
    ApiUrl = "https://api.arka-engine.io",
    ApiKey = Environment.GetEnvironmentVariable("ARKA_API_KEY"),
    TenantId = "your-tenant-id",
    Timeout = TimeSpan.FromSeconds(30)
});

// Create transaction
var transaction = new Transaction
{
    Id = "txn-123",
    Type = "wire_transfer",
    Amount = 50000.00m,
    Currency = "USD",
    Sender = new Party
    {
        Id = "cust-001",
        Name = "John Doe",
        Country = "US"
    },
    Recipient = new Party
    {
        Id = "cust-002",
        Name = "Acme Corp",
        Country = "GB"
    }
};

// Check compliance
var result = await client.Compliance.CheckAsync(new ComplianceRequest
{
    Transaction = transaction,
    Rulesets = new[] { "aml-kyc", "sanctions" },
    Options = new CheckOptions
    {
        IncludeExplanation = true,
        Timeout = TimeSpan.FromSeconds(5)
    }
});

Console.WriteLine($"Status: {result.Status}");
Console.WriteLine($"Score: {result.Score}");

foreach (var violation in result.Violations)
{
    Console.WriteLine($"Violation: {violation.RuleId} - {violation.Message}");
}

// Cleanup
await client.DisposeAsync();
```

### API Reference

```csharp
public class PactClient : IAsyncDisposable
{
    public PactClient(PactClientOptions options);

    public IRulesClient Rules { get; }
    public IComplianceClient Compliance { get; }
    public IAuditClient Audit { get; }
    public IEventsClient Events { get; }
    public IWebhooksClient Webhooks { get; }

    public Task<HealthStatus> HealthCheckAsync(CancellationToken ct = default);
    public ValueTask DisposeAsync();
}

public interface IRulesClient
{
    Task<PaginatedResponse<Rule>> ListAsync(ListOptions? options = null, CancellationToken ct = default);
    Task<Rule> GetAsync(string ruleId, CancellationToken ct = default);
    Task<Rule> CreateAsync(CreateRuleRequest request, CancellationToken ct = default);
    Task<Rule> UpdateAsync(string ruleId, UpdateRuleRequest request, CancellationToken ct = default);
    Task DeleteAsync(string ruleId, CancellationToken ct = default);
    Task<RuleResult> ExecuteAsync(string ruleId, IDictionary<string, object> input, CancellationToken ct = default);
}

public interface IComplianceClient
{
    Task<ComplianceResult> CheckAsync(ComplianceRequest request, CancellationToken ct = default);
    Task<IReadOnlyList<ComplianceResult>> BatchCheckAsync(IEnumerable<ComplianceRequest> requests, CancellationToken ct = default);
    Task<ComplianceReport> GenerateReportAsync(ReportOptions options, CancellationToken ct = default);
}
```

---

## Rust SDK

### Installation

```toml
# Cargo.toml
[dependencies]
arka-sdk = "1.0.0"
tokio = { version = "1", features = ["full"] }
```

### Quick Start

```rust
use pact_sdk::{PactClient, Transaction, ComplianceRequest, Party};
use std::env;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize client
    let client = PactClient::builder()
        .api_url("https://api.arka-engine.io")
        .api_key(&env::var("ARKA_API_KEY")?)
        .tenant_id("your-tenant-id")
        .build()?;

    // Create transaction
    let transaction = Transaction::builder()
        .id("txn-123")
        .transaction_type("wire_transfer")
        .amount(50000.00)
        .currency("USD")
        .sender(Party {
            id: "cust-001".to_string(),
            name: "John Doe".to_string(),
            country: "US".to_string(),
        })
        .recipient(Party {
            id: "cust-002".to_string(),
            name: "Acme Corp".to_string(),
            country: "GB".to_string(),
        })
        .build()?;

    // Check compliance
    let result = client.compliance()
        .check(ComplianceRequest {
            transaction,
            rulesets: vec!["aml-kyc".to_string(), "sanctions".to_string()],
            options: Some(CheckOptions {
                include_explanation: true,
                timeout_ms: Some(5000),
                ..Default::default()
            }),
        })
        .await?;

    println!("Status: {:?}", result.status);
    println!("Score: {}", result.score);

    for violation in &result.violations {
        println!("Violation: {} - {}", violation.rule_id, violation.message);
    }

    Ok(())
}
```

---

## Common Patterns

### Error Handling

```typescript
// TypeScript
import { PactError, RateLimitError, ValidationError } from '@arka-engine/sdk';

try {
  const result = await arka.compliance.check(request);
} catch (error) {
  if (error instanceof RateLimitError) {
    // Wait and retry
    await delay(error.retryAfter);
    return retry();
  } else if (error instanceof ValidationError) {
    console.error('Invalid request:', error.validationErrors);
  } else if (error instanceof PactError) {
    console.error('ARKA API error:', error.code, error.message);
  } else {
    throw error;
  }
}
```

### Retry Logic

```python
# Python
from pact_sdk import PactClient, RetryConfig

client = PactClient(
    api_url="https://api.arka-engine.io",
    api_key=os.environ["ARKA_API_KEY"],
    tenant_id="your-tenant-id",
    retry_config=RetryConfig(
        max_retries=3,
        initial_delay=1.0,
        max_delay=30.0,
        exponential_base=2.0,
        retry_on=[429, 500, 502, 503, 504]
    )
)
```

### Batch Processing

```go
// Go
results, err := client.Compliance.BatchCheck(ctx, transactions,
    pact.WithConcurrency(10),
    pact.WithFailFast(false),
    pact.WithTimeout(30*time.Second),
)
```

### Event Streaming

```typescript
// TypeScript
const stream = pact.events.stream({
  topics: ['compliance.checked', 'rule.violated'],
  filter: {
    severity: ['critical', 'high']
  }
});

for await (const event of stream) {
  console.log('Event:', event.type, event.data);
}
```

---

## Authentication

All SDKs support multiple authentication methods:

```typescript
// API Key (recommended for server-to-server)
const client = new PactClient({
  apiKey: 'pact_key_...'
});

// JWT Token (for user context)
const client = new PactClient({
  accessToken: 'eyJhbGciOiJSUzI1NiIs...'
});

// OAuth Client Credentials
const client = new PactClient({
  clientId: 'your_client_id',
  clientSecret: 'your_client_secret'
});
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Engineering Team | Initial release |

**Review Schedule:** Monthly
**Next Review:** January 2025
