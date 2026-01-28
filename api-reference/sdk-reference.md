# SDK Reference

The ARKA Engine provides official SDKs for multiple programming languages, enabling seamless integration of compliance evaluation into your applications.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    ARKA SDK Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Application Code                      ││
│  └─────────────────────────────────────────────────────────┘│
│                            │                                 │
│                            ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                     ARKA SDK                             ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────────┐  ││
│  │  │ Client  │  │  Rule   │  │  Proof  │  │ Governance│  ││
│  │  │ Config  │  │  Builder│  │ Verifier│  │  Client   │  ││
│  │  └─────────┘  └─────────┘  └─────────┘  └───────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
│                            │                                 │
│              ┌─────────────┼─────────────┐                  │
│              │             │             │                  │
│              ▼             ▼             ▼                  │
│         ┌────────┐    ┌────────┐    ┌────────┐              │
│         │  REST  │    │  gRPC  │    │  Local │              │
│         │  API   │    │  API   │    │  Agent │              │
│         └────────┘    └────────┘    └────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Python SDK

### Installation

```bash
pip install arka-engine-sdk

# With optional dependencies
pip install arka-engine-sdk[grpc,async]
```

### Quick Start

```python
from pact import ARKAClient, Transaction

# Initialize client
client = ARKAClient(
    endpoint="https://api.arka.io",
    api_key="your-api-key"
)

# Evaluate transaction
result = client.evaluate(
    transaction=Transaction(
        id="txn-123",
        type="wire_transfer",
        amount=15000,
        currency="USD",
        source={"account": "acct-001", "country": "US"},
        destination={"account": "acct-002", "country": "UK"}
    ),
    jurisdictions=["US", "UK"],
    domains=["AML", "SANCTIONS"]
)

print(f"Compliant: {result.compliant}")
print(f"Proof ID: {result.proof_id}")
```

### Configuration

```python
from pact import ARKAClient, ClientConfig, RetryConfig, TLSConfig

# Full configuration
config = ClientConfig(
    endpoint="https://api.arka.io",
    api_key="your-api-key",

    # Connection settings
    timeout=30.0,
    max_connections=100,
    connection_timeout=10.0,

    # Retry configuration
    retry=RetryConfig(
        max_retries=3,
        backoff_factor=0.5,
        retry_statuses=[502, 503, 504]
    ),

    # TLS configuration
    tls=TLSConfig(
        verify=True,
        cert_path="/path/to/client.crt",
        key_path="/path/to/client.key",
        ca_path="/path/to/ca.crt"
    ),

    # Logging
    log_level="INFO",
    log_requests=True
)

client = ARKAClient(config=config)
```

### Evaluation API

```python
from pact import (
    ARKAClient,
    Transaction,
    EvaluationOptions,
    Jurisdiction,
    Domain
)

client = ARKAClient(endpoint="https://api.arka.io", api_key="key")

# Basic evaluation
result = client.evaluate(
    transaction=Transaction(
        id="txn-123",
        type="wire_transfer",
        amount=15000,
        currency="USD",
        data={
            "source": {"account": "acct-001", "country": "US"},
            "destination": {"account": "acct-002", "country": "UK"},
            "purpose": "business_payment"
        }
    ),
    jurisdictions=["US", "UK"],
    domains=["AML"]
)

# Evaluation with options
result = client.evaluate(
    transaction=transaction,
    jurisdictions=[
        Jurisdiction(code="US", subdivision="NY"),
        Jurisdiction(code="UK")
    ],
    domains=[Domain.AML, Domain.SANCTIONS, Domain.KYC],
    options=EvaluationOptions(
        generate_proof=True,
        include_trace=True,
        fail_fast=False,
        timeout_ms=5000,
        rules=["rule/aml/high-value", "rule/sanctions/ofac"]
    )
)

# Access results
print(f"Evaluation ID: {result.evaluation_id}")
print(f"Status: {result.status}")
print(f"Compliant: {result.compliant}")

# Check violations
if not result.compliant:
    for violation in result.violations:
        print(f"Rule: {violation.rule_id}")
        print(f"Severity: {violation.severity}")
        print(f"Description: {violation.description}")
        print(f"Details: {violation.details}")

# Access rule results
for rule_result in result.rule_results:
    print(f"Rule: {rule_result.rule_id} v{rule_result.version}")
    print(f"Status: {rule_result.status}")
    print(f"Execution time: {rule_result.execution_time_us}µs")

# Access proof
if result.proof:
    print(f"Proof ID: {result.proof.proof_id}")
    print(f"Proof hash: {result.proof.hash}")
    print(f"Signature: {result.proof.signature}")
```

### Batch Evaluation

```python
from pact import ARKAClient, Transaction, BatchOptions

client = ARKAClient(endpoint="https://api.arka.io", api_key="key")

# Prepare transactions
transactions = [
    Transaction(id=f"txn-{i}", type="payment", amount=1000 * i, currency="USD")
    for i in range(100)
]

# Batch evaluation
batch_result = client.evaluate_batch(
    transactions=transactions,
    jurisdictions=["US"],
    domains=["AML"],
    options=BatchOptions(
        max_parallelism=10,
        stop_on_error=False,
        timeout_ms=60000
    )
)

print(f"Total: {batch_result.total}")
print(f"Succeeded: {batch_result.succeeded}")
print(f"Failed: {batch_result.failed}")
print(f"Duration: {batch_result.duration_ms}ms")

# Process results
for result in batch_result.results:
    if not result.compliant:
        print(f"Transaction {result.transaction_id}: Non-compliant")
        for v in result.violations:
            print(f"  - {v.description}")
```

### Async API

```python
import asyncio
from pact import AsyncPACTClient, Transaction

async def main():
    # Async client
    client = AsyncPACTClient(
        endpoint="https://api.arka.io",
        api_key="your-api-key"
    )

    # Single evaluation
    result = await client.evaluate(
        transaction=Transaction(id="txn-1", type="payment", amount=5000),
        jurisdictions=["US"],
        domains=["AML"]
    )

    # Concurrent evaluations
    tasks = [
        client.evaluate(
            transaction=Transaction(id=f"txn-{i}", type="payment", amount=1000*i),
            jurisdictions=["US"],
            domains=["AML"]
        )
        for i in range(10)
    ]
    results = await asyncio.gather(*tasks)

    # Streaming evaluation
    async for result in client.evaluate_stream(transaction_generator()):
        print(f"Result: {result.status}")

    await client.close()

asyncio.run(main())
```

### Rule Management

```python
from pact import ARKAClient, RuleBuilder, RuleMetadata
from datetime import date

client = ARKAClient(endpoint="https://api.arka.io", api_key="key")

# List rules
rules = client.rules.list(
    jurisdictions=["US"],
    domains=["AML"],
    tags=["high-priority"],
    page_size=50
)

for rule in rules:
    print(f"Rule: {rule.rule_id} v{rule.version}")

# Get specific rule
rule = client.rules.get("rule/aml/transaction-threshold", version="v1.2.0")
print(f"DSL: {rule.dsl_source}")

# Create rule with builder
builder = RuleBuilder()
rule_dsl = (
    builder
    .name("high_value_transfer")
    .description("Flag high-value wire transfers")
    .jurisdiction("US")
    .domain("AML")
    .condition("transaction.amount > 50000")
    .condition("transaction.type == 'wire_transfer'")
    .action("flag_for_review")
    .action("notify", channel="compliance-alerts")
    .build()
)

# Create rule
new_rule = client.rules.create(
    rule_id="rule/aml/high-value-transfer",
    name="High Value Transfer",
    description="Flag high-value wire transfers for review",
    jurisdiction="US",
    domain="AML",
    dsl_source=rule_dsl,
    metadata=RuleMetadata(
        author="compliance-team",
        tags=["high-value", "wire-transfer"],
        regulatory_reference="BSA/AML",
        effective_date=date(2024, 4, 1)
    )
)

# Validate rule before creation
validation = client.rules.validate(rule_dsl, jurisdiction="US", domain="AML")
if not validation.valid:
    for error in validation.errors:
        print(f"Error at line {error.line}: {error.message}")

# Get rule history
history = client.rules.get_history("rule/aml/transaction-threshold")
for version in history:
    print(f"Version {version.version}: {version.change_summary}")
```

### Governance API

```python
from pact import ARKAClient, ProposalBuilder, RuleChange

client = ARKAClient(endpoint="https://api.arka.io", api_key="key")

# Create proposal
proposal = client.governance.create_proposal(
    title="Update AML Threshold",
    description="Lower transaction monitoring threshold per new guidance",
    changes=[
        RuleChange(
            type="update",
            rule_id="rule/aml/transaction-threshold",
            new_dsl="""
                rule transaction_threshold {
                    when {
                        transaction.amount > 3000
                    }
                    then {
                        flag_for_review()
                    }
                }
            """,
            rationale="FinCEN guidance 2024-01"
        )
    ],
    metadata={
        "priority": "high",
        "regulatory_reference": "FinCEN-2024-01",
        "deadline": "2024-06-15"
    }
)

print(f"Proposal ID: {proposal.proposal_id}")

# Run simulation
simulation = client.governance.run_simulation(
    proposal_id=proposal.proposal_id,
    mode="historical_replay",
    config={
        "start_date": "2024-01-01",
        "end_date": "2024-03-31",
        "sample_size": 100000
    }
)

# Wait for simulation completion
result = client.governance.wait_for_simulation(simulation.simulation_id)
print(f"New flags: {result.metrics.new_flags}")
print(f"False positive rate: {result.metrics.false_positive_rate}")

# Submit review
client.governance.submit_review(
    proposal_id=proposal.proposal_id,
    decision="approved",
    comments="Simulation results acceptable"
)

# List proposals
proposals = client.governance.list_proposals(
    statuses=["pending_review"],
    page_size=20
)
```

### Proof Verification

```python
from pact import ARKAClient, ProofVerifier

client = ARKAClient(endpoint="https://api.arka.io", api_key="key")

# Get proof
proof = client.proofs.get("proof-123")

# Verify proof
verifier = ProofVerifier(client)
verification = verifier.verify(proof)

print(f"Valid: {verification.valid}")
print(f"Signature valid: {verification.signature_valid}")
print(f"Hash valid: {verification.hash_valid}")

# Verify against blockchain
anchor_verification = verifier.verify_anchor(
    proof,
    chain="ethereum"
)

print(f"Anchored: {anchor_verification.anchored}")
print(f"Block: {anchor_verification.block_number}")
print(f"Merkle proof valid: {anchor_verification.merkle_valid}")

# Get Merkle proof for external verification
merkle_proof = client.proofs.get_merkle_proof("proof-123")
print(f"Leaf hash: {merkle_proof.leaf_hash}")
print(f"Root hash: {merkle_proof.root_hash}")
print(f"Proof path: {merkle_proof.proof_path}")
```

### Webhooks

```python
from pact import WebhookHandler, WebhookEvent
from flask import Flask, request

app = Flask(__name__)
handler = WebhookHandler(secret="your-webhook-secret")

@app.route('/webhooks/pact', methods=['POST'])
def handle_webhook():
    event = handler.verify_and_parse(
        payload=request.data,
        signature=request.headers.get('X-PACT-Signature')
    )

    if event.type == WebhookEvent.EVALUATION_COMPLETED:
        print(f"Evaluation {event.data.evaluation_id} completed")
    elif event.type == WebhookEvent.RULE_UPDATED:
        print(f"Rule {event.data.rule_id} updated to {event.data.version}")
    elif event.type == WebhookEvent.PROPOSAL_APPROVED:
        print(f"Proposal {event.data.proposal_id} approved")
    elif event.type == WebhookEvent.ANCHOR_CONFIRMED:
        print(f"Proof {event.data.proof_id} anchored")

    return '', 200
```

## JavaScript/TypeScript SDK

### Installation

```bash
npm install @arka/sdk

# or with yarn
yarn add @arka/sdk
```

### Quick Start

```typescript
import { ARKAClient, Transaction } from '@arka/sdk';

// Initialize client
const client = new ARKAClient({
  endpoint: 'https://api.arka.io',
  apiKey: 'your-api-key'
});

// Evaluate transaction
const result = await client.evaluate({
  transaction: {
    id: 'txn-123',
    type: 'wire_transfer',
    amount: 15000,
    currency: 'USD',
    source: { account: 'acct-001', country: 'US' },
    destination: { account: 'acct-002', country: 'UK' }
  },
  jurisdictions: ['US', 'UK'],
  domains: ['AML', 'SANCTIONS']
});

console.log(`Compliant: ${result.compliant}`);
console.log(`Proof ID: ${result.proof?.proofId}`);
```

### Configuration

```typescript
import { ARKAClient, ClientConfig } from '@arka/sdk';

const config: ClientConfig = {
  endpoint: 'https://api.arka.io',
  apiKey: 'your-api-key',

  // Connection settings
  timeout: 30000,
  maxRetries: 3,
  retryDelay: 1000,

  // TLS settings
  tls: {
    rejectUnauthorized: true,
    cert: fs.readFileSync('/path/to/client.crt'),
    key: fs.readFileSync('/path/to/client.key'),
    ca: fs.readFileSync('/path/to/ca.crt')
  },

  // Interceptors
  interceptors: {
    request: async (config) => {
      config.headers['X-Request-ID'] = generateRequestId();
      return config;
    },
    response: async (response) => {
      console.log(`Request took ${response.duration}ms`);
      return response;
    }
  }
};

const client = new ARKAClient(config);
```

### TypeScript Types

```typescript
import {
  Transaction,
  EvaluationResult,
  Violation,
  TrustProof,
  Rule,
  Proposal,
  ComplianceStatus,
  Domain,
  Jurisdiction
} from '@arka/sdk';

// Strongly typed transaction
interface WireTransfer extends Transaction {
  type: 'wire_transfer';
  data: {
    source: {
      account: string;
      bank: string;
      country: string;
    };
    destination: {
      account: string;
      bank: string;
      country: string;
    };
    purpose: string;
    reference: string;
  };
}

// Type-safe evaluation
const result: EvaluationResult = await client.evaluate({
  transaction: wireTransfer,
  jurisdictions: [{ code: 'US' }, { code: 'UK' }],
  domains: [Domain.AML, Domain.SANCTIONS]
});

// Handle violations with type safety
result.violations.forEach((violation: Violation) => {
  console.log(`Rule: ${violation.ruleId}`);
  console.log(`Severity: ${violation.severity}`);
});
```

### Streaming API

```typescript
import { ARKAClient, TransactionStream } from '@arka/sdk';

const client = new ARKAClient({
  endpoint: 'https://api.arka.io',
  apiKey: 'your-api-key'
});

// Create stream
const stream = client.createStream({
  jurisdictions: ['US'],
  domains: ['AML']
});

// Send transactions
stream.on('ready', () => {
  transactions.forEach(txn => stream.send(txn));
});

// Receive results
stream.on('result', (result) => {
  console.log(`Transaction ${result.transactionId}: ${result.status}`);

  if (!result.compliant) {
    result.violations.forEach(v => {
      console.log(`  Violation: ${v.description}`);
    });
  }
});

// Handle errors
stream.on('error', (error) => {
  console.error('Stream error:', error);
});

// Close stream when done
stream.close();
```

### React Hooks

```typescript
import { usePACT, useEvaluation, useRules } from '@arka/sdk/react';

function ComplianceCheck() {
  const { client } = usePACT();

  const {
    evaluate,
    result,
    loading,
    error
  } = useEvaluation();

  const handleSubmit = async (transaction: Transaction) => {
    await evaluate({
      transaction,
      jurisdictions: ['US'],
      domains: ['AML']
    });
  };

  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;

  return (
    <div>
      {result && (
        <ComplianceResult
          compliant={result.compliant}
          violations={result.violations}
        />
      )}
    </div>
  );
}

// Provider setup
function App() {
  return (
    <PACTProvider
      config={{
        endpoint: 'https://api.arka.io',
        apiKey: process.env.ARKA_API_KEY
      }}
    >
      <ComplianceCheck />
    </PACTProvider>
  );
}
```

## Go SDK

### Installation

```bash
go get github.c../arka-io/arka-sdk-go
```

### Quick Start

```go
package main

import (
    "context"
    "fmt"
    "log"

    pact "github.c../arka-io/arka-sdk-go"
)

func main() {
    // Create client
    client, err := pact.NewClient(
        pact.WithEndpoint("https://api.arka.io"),
        pact.WithAPIKey("your-api-key"),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()

    // Create transaction
    transaction := &pact.Transaction{
        ID:       "txn-123",
        Type:     "wire_transfer",
        Amount:   15000,
        Currency: "USD",
        Data: map[string]interface{}{
            "source": map[string]string{
                "account": "acct-001",
                "country": "US",
            },
            "destination": map[string]string{
                "account": "acct-002",
                "country": "UK",
            },
        },
    }

    // Evaluate
    result, err := client.Evaluate(context.Background(), &pact.EvaluateRequest{
        Transaction:   transaction,
        Jurisdictions: []string{"US", "UK"},
        Domains:       []pact.Domain{pact.DomainAML, pact.DomainSanctions},
    })
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Compliant: %v\n", result.Compliant)
    fmt.Printf("Proof ID: %s\n", result.Proof.ProofID)
}
```

### Configuration

```go
package main

import (
    "crypto/tls"
    "time"

    pact "github.c../arka-io/arka-sdk-go"
)

func main() {
    // Load TLS certificates
    cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")

    // Create client with full configuration
    client, _ := pact.NewClient(
        pact.WithEndpoint("https://api.arka.io"),
        pact.WithAPIKey("your-api-key"),

        // Timeouts
        pact.WithTimeout(30 * time.Second),
        pact.WithConnectTimeout(10 * time.Second),

        // Retries
        pact.WithMaxRetries(3),
        pact.WithRetryBackoff(time.Second),

        // TLS
        pact.WithTLSConfig(&tls.Config{
            Certificates: []tls.Certificate{cert},
            MinVersion:   tls.VersionTLS13,
        }),

        // Connection pool
        pact.WithMaxConnections(100),
        pact.WithMaxIdleConnections(10),

        // Logging
        pact.WithLogger(customLogger),
    )
    defer client.Close()
}
```

### Batch Evaluation

```go
func batchEvaluate(client *pact.Client, transactions []*pact.Transaction) error {
    ctx := context.Background()

    // Batch evaluate
    results, err := client.EvaluateBatch(ctx, &pact.EvaluateBatchRequest{
        Transactions:  transactions,
        Jurisdictions: []string{"US"},
        Domains:       []pact.Domain{pact.DomainAML},
        Options: &pact.BatchOptions{
            MaxParallelism: 10,
            StopOnError:    false,
        },
    })
    if err != nil {
        return err
    }

    fmt.Printf("Total: %d, Succeeded: %d, Failed: %d\n",
        results.Total, results.Succeeded, results.Failed)

    for _, result := range results.Results {
        if !result.Compliant {
            fmt.Printf("Transaction %s: Non-compliant\n", result.TransactionID)
            for _, v := range result.Violations {
                fmt.Printf("  - %s: %s\n", v.RuleID, v.Description)
            }
        }
    }

    return nil
}
```

### gRPC Client

```go
package main

import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"

    pb "github.c../arka-io/arka-sdk-go/proto"
)

func main() {
    // Load certificates
    cert, _ := tls.LoadX509KeyPair("client.crt", "client.key")
    caCert, _ := ioutil.ReadFile("ca.crt")
    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    // Create TLS credentials
    creds := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        RootCAs:      caCertPool,
    })

    // Create gRPC connection
    conn, _ := grpc.Dial(
        "api.arka.io:50051",
        grpc.WithTransportCredentials(creds),
        grpc.WithDefaultCallOptions(
            grpc.MaxCallRecvMsgSize(16*1024*1024),
            grpc.MaxCallSendMsgSize(16*1024*1024),
        ),
    )
    defer conn.Close()

    // Create client
    client := pb.NewEvaluationServiceClient(conn)

    // Evaluate
    resp, _ := client.Evaluate(context.Background(), &pb.EvaluateRequest{
        RequestId: "req-123",
        Transaction: &structpb.Struct{
            // ... transaction data
        },
        Jurisdictions: []*pb.Jurisdiction{{Code: "US"}},
        Domains:       []pb.Domain{pb.Domain_DOMAIN_AML},
    })

    fmt.Printf("Status: %v\n", resp.Status)
}
```

## Java SDK

### Installation

```xml
<!-- Maven -->
<dependency>
    <groupId>io.pact</groupId>
    <artifactId>arka-sdk</artifactId>
    <version>1.0.0</version>
</dependency>
```

```gradle
// Gradle
implementation 'io.pact:arka-sdk:1.0.0'
```

### Quick Start

```java
import io.pact.sdk.*;
import io.pact.sdk.model.*;

public class QuickStart {
    public static void main(String[] args) {
        // Create client
        ARKAClient client = ARKAClient.builder()
            .endpoint("https://api.arka.io")
            .apiKey("your-api-key")
            .build();

        // Create transaction
        Transaction transaction = Transaction.builder()
            .id("txn-123")
            .type("wire_transfer")
            .amount(15000)
            .currency("USD")
            .data(Map.of(
                "source", Map.of("account", "acct-001", "country", "US"),
                "destination", Map.of("account", "acct-002", "country", "UK")
            ))
            .build();

        // Evaluate
        EvaluationResult result = client.evaluate(
            EvaluateRequest.builder()
                .transaction(transaction)
                .jurisdictions(List.of("US", "UK"))
                .domains(List.of(Domain.AML, Domain.SANCTIONS))
                .build()
        );

        System.out.println("Compliant: " + result.isCompliant());
        System.out.println("Proof ID: " + result.getProof().getProofId());

        // Handle violations
        for (Violation violation : result.getViolations()) {
            System.out.println("Rule: " + violation.getRuleId());
            System.out.println("Description: " + violation.getDescription());
        }

        client.close();
    }
}
```

### Async API

```java
import io.pact.sdk.*;
import java.util.concurrent.CompletableFuture;

public class AsyncExample {
    public static void main(String[] args) {
        ARKAAsyncClient client = ARKAAsyncClient.builder()
            .endpoint("https://api.arka.io")
            .apiKey("your-api-key")
            .build();

        // Async evaluation
        CompletableFuture<EvaluationResult> future = client.evaluateAsync(
            EvaluateRequest.builder()
                .transaction(transaction)
                .jurisdictions(List.of("US"))
                .domains(List.of(Domain.AML))
                .build()
        );

        future.thenAccept(result -> {
            System.out.println("Compliant: " + result.isCompliant());
        }).exceptionally(error -> {
            System.err.println("Error: " + error.getMessage());
            return null;
        });

        // Batch async
        List<CompletableFuture<EvaluationResult>> futures = transactions.stream()
            .map(txn -> client.evaluateAsync(
                EvaluateRequest.builder()
                    .transaction(txn)
                    .jurisdictions(List.of("US"))
                    .domains(List.of(Domain.AML))
                    .build()
            ))
            .collect(Collectors.toList());

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenRun(() -> System.out.println("All evaluations complete"));
    }
}
```

### Spring Boot Integration

```java
import io.pact.sdk.spring.*;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@EnablePACT
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Configuration
@Configuration
public class ARKAConfig {
    @Bean
    public ARKAClient pactClient(PACTProperties properties) {
        return ARKAClient.builder()
            .endpoint(properties.getEndpoint())
            .apiKey(properties.getApiKey())
            .build();
    }
}

// Service usage
@Service
public class ComplianceService {
    private final ARKAClient pactClient;

    public ComplianceService(PACTClient pactClient) {
        this.pactClient = pactClient;
    }

    public EvaluationResult evaluateTransaction(Transaction transaction) {
        return pactClient.evaluate(
            EvaluateRequest.builder()
                .transaction(transaction)
                .jurisdictions(List.of("US"))
                .domains(List.of(Domain.AML))
                .build()
        );
    }
}

// application.yml
pact:
  endpoint: https://api.arka.io
  api-key: ${ARKA_API_KEY}
  timeout: 30s
  max-connections: 100
```

## Error Handling

### Error Types

```python
from pact.exceptions import (
    ARKAError,
    AuthenticationError,
    RateLimitError,
    ValidationError,
    RuleNotFoundError,
    EvaluationTimeoutError,
    NetworkError
)

try:
    result = client.evaluate(request)
except AuthenticationError as e:
    print(f"Authentication failed: {e.message}")
except RateLimitError as e:
    print(f"Rate limited. Retry after {e.retry_after} seconds")
except ValidationError as e:
    print(f"Validation error: {e.message}")
    for error in e.errors:
        print(f"  Field: {error.field}, Message: {error.message}")
except RuleNotFoundError as e:
    print(f"Rule not found: {e.rule_id}")
except EvaluationTimeoutError as e:
    print(f"Evaluation timed out after {e.timeout_ms}ms")
except NetworkError as e:
    print(f"Network error: {e.message}")
except ARKAError as e:
    print(f"ARKA error: {e.code} - {e.message}")
```

### Retry Strategies

```python
from pact import ARKAClient, RetryConfig, RetryStrategy

# Exponential backoff
client = ARKAClient(
    endpoint="https://api.arka.io",
    api_key="key",
    retry=RetryConfig(
        strategy=RetryStrategy.EXPONENTIAL_BACKOFF,
        max_retries=5,
        base_delay=1.0,
        max_delay=30.0,
        jitter=True
    )
)

# Custom retry logic
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
def evaluate_with_retry(client, request):
    return client.evaluate(request)
```

## Testing

### Mock Client

```python
from pact.testing import MockPACTClient, MockResponse

# Create mock client
mock_client = MockPACTClient()

# Configure mock responses
mock_client.mock_evaluate(
    request_matcher=lambda r: r.transaction.type == "wire_transfer",
    response=MockResponse(
        compliant=True,
        proof_id="mock-proof-123"
    )
)

mock_client.mock_evaluate(
    request_matcher=lambda r: r.transaction.amount > 50000,
    response=MockResponse(
        compliant=False,
        violations=[
            {"rule_id": "rule/aml/high-value", "description": "High value transfer"}
        ]
    )
)

# Use in tests
def test_compliance_check():
    service = ComplianceService(mock_client)
    result = service.check_transaction(transaction)
    assert result.compliant == True
```

### Integration Testing

```python
import pytest
from pact import ARKAClient
from pact.testing import ARKATestServer

@pytest.fixture
def pact_client():
    # Start local test server
    server = ARKATestServer()
    server.start()

    client = ARKAClient(
        endpoint=server.endpoint,
        api_key="test-key"
    )

    yield client

    client.close()
    server.stop()

def test_evaluation(pact_client):
    result = pact_client.evaluate(
        transaction=test_transaction,
        jurisdictions=["US"],
        domains=["AML"]
    )

    assert result.evaluation_id is not None
    assert result.status in ["compliant", "non_compliant"]
```

## Best Practices

### Connection Management

1. **Reuse clients**: Create one client instance and reuse it
2. **Connection pooling**: Configure appropriate pool sizes
3. **Graceful shutdown**: Always close clients when done

### Performance

1. **Batch operations**: Use batch APIs for multiple transactions
2. **Async/streaming**: Use async APIs for high throughput
3. **Caching**: Cache rule metadata locally

### Security

1. **Secure credentials**: Never hardcode API keys
2. **TLS verification**: Always verify TLS certificates
3. **Audit logging**: Log all compliance decisions

### Error Handling

1. **Retry transient errors**: Use exponential backoff
2. **Handle rate limits**: Respect retry-after headers
3. **Monitor errors**: Track error rates and patterns

## Related Documentation

- [REST API Reference](./rest-api.md)
- [gRPC API Reference](./grpc-api.md)
- [Authentication Guide](../security/authentication.md)
- [Integration Patterns](../guides/integration-patterns.md)
