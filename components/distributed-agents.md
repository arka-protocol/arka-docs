# Distributed Compliance Agents

Distributed Compliance Agents provide edge-based regulatory compliance enforcement with cryptographically verifiable execution and offline capabilities.

## Overview

The ARKA Engine's distributed agent architecture enables compliance evaluation at the edge, close to where transactions occur. Agents operate autonomously while maintaining cryptographic proof of their actions that can be independently verified.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Distributed Agent Architecture                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│                        ┌─────────────────┐                              │
│                        │  Control Plane  │                              │
│                        │    (Central)    │                              │
│                        └────────┬────────┘                              │
│                                 │                                        │
│              ┌──────────────────┼──────────────────┐                    │
│              │                  │                  │                    │
│      ┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐           │
│      │   Region A    │  │   Region B    │  │   Region C    │           │
│      │   Cluster     │  │   Cluster     │  │   Cluster     │           │
│      └───────┬───────┘  └───────┬───────┘  └───────┬───────┘           │
│              │                  │                  │                    │
│       ┌──────┼──────┐    ┌──────┼──────┐    ┌──────┼──────┐            │
│       │      │      │    │      │      │    │      │      │            │
│      ┌▼┐    ┌▼┐    ┌▼┐  ┌▼┐    ┌▼┐    ┌▼┐  ┌▼┐    ┌▼┐    ┌▼┐          │
│      │A│    │A│    │A│  │A│    │A│    │A│  │A│    │A│    │A│          │
│      └─┘    └─┘    └─┘  └─┘    └─┘    └─┘  └─┘    └─┘    └─┘          │
│     Agent  Agent  Agent Agent Agent Agent Agent Agent Agent            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### Agent Architecture

Each distributed agent runs a complete compliance evaluation stack:

```
┌─────────────────────────────────────────────────────────────┐
│                    Compliance Agent                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Agent Runtime                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │   │
│  │  │   RTVM     │  │   Rule     │  │    Proof       │  │   │
│  │  │  Instance  │  │   Cache    │  │   Generator    │  │   │
│  │  └────────────┘  └────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Local Storage                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │   │
│  │  │   Rules    │  │   Proofs   │  │    Sync        │  │   │
│  │  │   Store    │  │   Queue    │  │    State       │  │   │
│  │  └────────────┘  └────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 Communication Layer                   │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │   │
│  │  │   Sync     │  │ Heartbeat  │  │   Attestation  │  │   │
│  │  │  Manager   │  │  Service   │  │   Service      │  │   │
│  │  └────────────┘  └────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Identity Layer                     │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │   │
│  │  │    DID     │  │    Key     │  │  Credential    │  │   │
│  │  │  Manager   │  │   Store    │  │    Wallet      │  │   │
│  │  └────────────┘  └────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Agent Configuration

```yaml
# agent-config.yaml
apiVersion: arka.io/v1
kind: AgentConfig
metadata:
  name: compliance-agent-001
  region: us-east-1
  cluster: production-a
spec:
  # Identity
  identity:
    did: did:arka:agent:node-001
    keyStore:
      type: hsm
      config:
        provider: aws-cloudhsm
        clusterId: cluster-abc123

  # RTVM configuration
  rtvm:
    maxConcurrentEvaluations: 100
    evaluationTimeout: 30s
    memoryLimit: 2Gi
    deterministic: true
    timeTravelEnabled: true

  # Rule synchronization
  rules:
    syncInterval: 5m
    cacheSize: 10000
    preloadJurisdictions:
      - US
      - EU
      - UK
    preloadDomains:
      - AML
      - KYC
      - SANCTIONS

  # Local storage
  storage:
    rulesPath: /data/rules
    proofsPath: /data/proofs
    syncStatePath: /data/sync
    retentionDays: 30

  # Communication
  communication:
    controlPlane:
      endpoint: https://control.arka.io
      heartbeatInterval: 30s
      reconnectBackoff: 5s
    peers:
      discoveryMethod: dns
      dnsEndpoint: _pact._tcp.agents.arka.io

  # Offline mode
  offline:
    enabled: true
    maxOfflineDuration: 24h
    proofQueueSize: 10000
    syncOnReconnect: true

  # Monitoring
  monitoring:
    metricsPort: 9090
    healthPort: 8080
    tracingEnabled: true
    tracingEndpoint: https://tracing.arka.io

  # Security
  security:
    mtls:
      enabled: true
      certPath: /certs/agent.crt
      keyPath: /certs/agent.key
      caPath: /certs/ca.crt
    attestation:
      enabled: true
      tpmEnabled: true
```

### Agent Deployment

#### Kubernetes Deployment

```yaml
# agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arka-agent
  namespace: compliance
spec:
  replicas: 3
  selector:
    matchLabels:
      app: arka-agent
  template:
    metadata:
      labels:
        app: arka-agent
    spec:
      serviceAccountName: arka-agent
      containers:
        - name: agent
          image: pact/compliance-agent:latest
          ports:
            - containerPort: 8080
              name: api
            - containerPort: 9090
              name: metrics
          env:
            - name: AGENT_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: AGENT_REGION
              value: "us-east-1"
            - name: CONTROL_PLANE_URL
              value: "https://control.arka.io"
          volumeMounts:
            - name: config
              mountPath: /config
            - name: certs
              mountPath: /certs
              readOnly: true
            - name: data
              mountPath: /data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: arka-agent-config
        - name: certs
          secret:
            secretName: arka-agent-certs
        - name: data
          persistentVolumeClaim:
            claimName: arka-agent-data
---
apiVersion: v1
kind: Service
metadata:
  name: arka-agent
  namespace: compliance
spec:
  selector:
    app: arka-agent
  ports:
    - port: 8080
      name: api
    - port: 9090
      name: metrics
```

#### Docker Compose (Development)

```yaml
# docker-compose.yaml
version: '3.8'
services:
  agent:
    image: pact/compliance-agent:latest
    environment:
      - AGENT_ID=dev-agent-001
      - AGENT_MODE=development
      - CONTROL_PLANE_URL=http://control-plane:8080
      - LOG_LEVEL=debug
    ports:
      - "8080:8080"
      - "9090:9090"
    volumes:
      - ./config:/config
      - ./data:/data
      - ./certs:/certs:ro
    depends_on:
      - control-plane
      - redis

  control-plane:
    image: pact/control-plane:latest
    ports:
      - "8081:8080"
    environment:
      - DATABASE_URL=postgres://pact:pact@postgres:5432/pact
      - REDIS_URL=redis://redis:6379

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=pact
      - POSTGRES_PASSWORD=pact
      - POSTGRES_DB=pact
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## Agent Operations

### Compliance Evaluation

```python
from pact.agent import ComplianceAgent

agent = ComplianceAgent(config_path="/config/agent.yaml")

# Initialize agent
await agent.initialize()

# Evaluate transaction
result = await agent.evaluate(
    transaction={
        "id": "txn-123",
        "type": "wire_transfer",
        "amount": 15000,
        "currency": "USD",
        "source": {
            "account": "acct-001",
            "country": "US"
        },
        "destination": {
            "account": "acct-002",
            "country": "UK"
        }
    },
    jurisdictions=["US", "UK"],
    domains=["AML", "SANCTIONS"]
)

print(f"Compliant: {result.compliant}")
print(f"Rules evaluated: {result.rules_evaluated}")
print(f"Proof ID: {result.proof_id}")

# Result includes cryptographic proof
print(f"Proof hash: {result.proof.hash}")
print(f"Agent signature: {result.proof.signature}")
```

### Batch Evaluation

```python
# Evaluate multiple transactions
transactions = [txn1, txn2, txn3, ...]

batch_results = await agent.evaluate_batch(
    transactions=transactions,
    parallel=True,
    max_concurrency=50
)

print(f"Total: {batch_results.total}")
print(f"Compliant: {batch_results.compliant_count}")
print(f"Non-compliant: {batch_results.non_compliant_count}")
print(f"Processing time: {batch_results.duration_ms}ms")

for result in batch_results.results:
    if not result.compliant:
        print(f"Transaction {result.transaction_id}:")
        for violation in result.violations:
            print(f"  Rule: {violation.rule_id}")
            print(f"  Reason: {violation.reason}")
```

### Real-time Streaming

```python
from pact.agent import TransactionStream

# Create transaction stream
stream = TransactionStream(
    agent=agent,
    source="kafka://transactions.compliance.topic",
    config={
        "consumer_group": "compliance-agents",
        "auto_offset_reset": "latest"
    }
)

# Process transactions in real-time
async for transaction in stream:
    result = await agent.evaluate(transaction)

    if not result.compliant:
        await stream.send_alert(
            topic="alerts.compliance",
            alert={
                "transaction_id": transaction.id,
                "violations": result.violations,
                "proof_id": result.proof_id
            }
        )

    # Acknowledge processing
    await stream.ack(transaction)
```

## Trust Proofs

### Proof Generation

Every compliance evaluation generates a cryptographic trust proof:

```
┌─────────────────────────────────────────────────────────────┐
│                      Trust Proof                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   Proof Header                          │ │
│  │  Proof ID: proof-abc123                                │ │
│  │  Timestamp: 2024-03-15T10:30:00Z                       │ │
│  │  Agent DID: did:arka:agent:node-001                    │ │
│  │  Rule Version: v1.2.0                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                 Execution Context                       │ │
│  │  Input Hash: 0xabc123...                               │ │
│  │  Rule Hash: 0xdef456...                                │ │
│  │  RTVM State Hash: 0xghi789...                          │ │
│  │  Output Hash: 0xjkl012...                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   Attestations                          │ │
│  │  Agent Attestation:                                    │ │
│  │    Type: ed25519                                       │ │
│  │    Signature: 0xmno345...                              │ │
│  │  Hardware Attestation (optional):                      │ │
│  │    Type: tpm2                                          │ │
│  │    Quote: 0xpqr678...                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                 Verification Data                       │ │
│  │  Merkle Path: [hash1, hash2, hash3]                   │ │
│  │  Anchor Reference: anchor-xyz789                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Proof Structure

```python
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

@dataclass
class TrustProof:
    """Cryptographic proof of compliant execution."""

    # Header
    proof_id: str
    timestamp: datetime
    agent_did: str

    # Rule context
    rule_id: str
    rule_version: str
    jurisdiction: str

    # Execution hashes
    input_hash: str
    rule_hash: str
    rtvm_state_hash: str
    output_hash: str
    execution_trace_hash: str

    # Attestations
    agent_signature: str
    hardware_attestation: Optional[str]

    # Verification
    merkle_path: List[str]
    anchor_reference: Optional[str]

    def compute_proof_hash(self) -> str:
        """Compute deterministic hash of proof."""
        data = (
            self.proof_id +
            self.timestamp.isoformat() +
            self.agent_did +
            self.rule_id +
            self.rule_version +
            self.input_hash +
            self.output_hash +
            self.execution_trace_hash
        )
        return sha3_256(data.encode()).hexdigest()

    def verify_signature(self, public_key: bytes) -> bool:
        """Verify agent signature."""
        proof_hash = self.compute_proof_hash()
        return verify_ed25519(
            public_key,
            bytes.fromhex(self.agent_signature),
            bytes.fromhex(proof_hash)
        )
```

### Proof Verification

```python
from pact.agent import ProofVerifier

verifier = ProofVerifier()

# Verify proof locally
verification = await verifier.verify(proof)

print(f"Valid: {verification.valid}")
print(f"Signature valid: {verification.signature_valid}")
print(f"Input hash matches: {verification.input_valid}")
print(f"Output hash matches: {verification.output_valid}")
print(f"Execution deterministic: {verification.execution_valid}")

# Verify against blockchain anchor
anchor_verification = await verifier.verify_anchor(
    proof,
    chain="ethereum"
)

print(f"Anchored: {anchor_verification.anchored}")
print(f"Block: {anchor_verification.block_number}")
print(f"Merkle proof valid: {anchor_verification.merkle_valid}")
```

### Proof Aggregation

```python
from pact.agent import ProofAggregator

aggregator = ProofAggregator()

# Aggregate multiple proofs
aggregated = await aggregator.aggregate(
    proofs=[proof1, proof2, proof3, ...],
    method="merkle"
)

print(f"Aggregated root: {aggregated.root_hash}")
print(f"Proof count: {aggregated.proof_count}")
print(f"Individual proofs verifiable: {aggregated.verifiable}")

# Verify individual proof within aggregate
individual_verification = await aggregator.verify_individual(
    aggregated,
    proof_id="proof-123"
)
```

## Rule Synchronization

### Sync Protocol

```
┌─────────────────────────────────────────────────────────────┐
│                    Sync Protocol                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Agent                           Control Plane               │
│    │                                   │                     │
│    │──── SYNC_REQUEST ────────────────▶│                    │
│    │     (last_sync_version)           │                    │
│    │                                   │                     │
│    │◀─── SYNC_RESPONSE ────────────────│                    │
│    │     (rule_updates, deletions)     │                    │
│    │                                   │                     │
│    │──── SYNC_ACK ────────────────────▶│                    │
│    │     (applied_version)             │                    │
│    │                                   │                     │
│    │◀─── SYNC_CONFIRM ────────────────│                    │
│    │     (next_sync_time)              │                    │
│    │                                   │                     │
└─────────────────────────────────────────────────────────────┘
```

### Sync Configuration

```yaml
# sync-config.yaml
sync:
  # Sync intervals
  intervals:
    normal: 5m
    priority: 30s
    emergency: 5s

  # Conflict resolution
  conflicts:
    strategy: control_plane_wins
    notifyOnConflict: true

  # Bandwidth optimization
  optimization:
    deltaSync: true
    compression: zstd
    maxBatchSize: 100

  # Offline handling
  offline:
    maxQueueSize: 10000
    syncOnReconnect: true
    prioritizeRecent: true

  # Integrity verification
  verification:
    verifySignatures: true
    verifyHashes: true
    rejectInvalid: true
```

### Sync API

```python
from pact.agent import RuleSyncManager

sync_manager = RuleSyncManager(agent)

# Force immediate sync
await sync_manager.sync_now()

# Get sync status
status = await sync_manager.get_status()
print(f"Last sync: {status.last_sync}")
print(f"Rules synced: {status.rule_count}")
print(f"Pending updates: {status.pending_updates}")

# Subscribe to sync events
@sync_manager.on("sync_complete")
async def handle_sync(event):
    print(f"Sync completed: {event.rules_updated} rules updated")
    print(f"New version: {event.version}")

@sync_manager.on("sync_conflict")
async def handle_conflict(event):
    print(f"Conflict detected: {event.rule_id}")
    print(f"Resolution: {event.resolution}")

# Manual rule fetch
rule = await sync_manager.fetch_rule(
    rule_id="rule/aml/transaction-threshold",
    version="v1.2.0"
)
```

## Offline Mode

### Offline Capabilities

Agents can operate fully offline while maintaining compliance guarantees:

```python
from pact.agent import OfflineManager

offline = OfflineManager(agent)

# Check offline status
status = offline.get_status()
print(f"Online: {status.online}")
print(f"Offline duration: {status.offline_duration}")
print(f"Queued proofs: {status.queued_proofs}")

# Offline evaluation
result = await agent.evaluate(
    transaction=transaction,
    allow_offline=True
)

if result.offline_evaluation:
    print("Evaluated offline")
    print(f"Proof queued for sync: {result.proof_id}")

# Manual sync when back online
await offline.sync_queued_proofs()
```

### Offline Rule Caching

```yaml
# offline-cache-config.yaml
offline:
  cache:
    # Rules to preload
    preload:
      jurisdictions: [US, EU, UK, SG, HK]
      domains: [AML, KYC, SANCTIONS, PEP]
      minVersions: 3  # Keep last 3 versions

    # Cache limits
    limits:
      maxRules: 10000
      maxSizeMB: 500
      evictionPolicy: lru

    # Freshness
    freshness:
      maxAge: 24h
      warnAge: 12h
      forceRefreshAge: 48h

    # Persistence
    persistence:
      path: /data/rule-cache
      encryption: aes-256-gcm
      integrity: sha256
```

### Offline Proof Queue

```python
# Configure proof queue for offline operation
queue_config = {
    "max_size": 10000,
    "persistence": "/data/proof-queue",
    "priority_levels": ["critical", "high", "normal", "low"],
    "retry_policy": {
        "max_attempts": 3,
        "backoff_multiplier": 2,
        "max_backoff": "1h"
    }
}

# Queue management
queue = agent.proof_queue

# Get queue statistics
stats = await queue.get_stats()
print(f"Queued: {stats.total}")
print(f"By priority: {stats.by_priority}")
print(f"Oldest: {stats.oldest_timestamp}")

# Process queue when online
async for proof in queue.drain():
    try:
        await anchoring.submit(proof)
        await queue.ack(proof.id)
    except Exception as e:
        await queue.nack(proof.id, reason=str(e))
```

## Attestation Framework

### Hardware Attestation

```python
from pact.agent import HardwareAttestation

# Initialize TPM-based attestation
attestation = HardwareAttestation(
    type="tpm2",
    config={
        "device": "/dev/tpm0",
        "key_handle": "0x81000001"
    }
)

# Generate attestation quote
quote = await attestation.generate_quote(
    nonce=random_bytes(32),
    pcr_selection=[0, 1, 2, 7]  # PCR registers to include
)

print(f"Quote: {quote.quote_data}")
print(f"Signature: {quote.signature}")
print(f"PCR values: {quote.pcr_values}")

# Include in proof
proof.hardware_attestation = quote.serialize()
```

### Software Attestation

```python
from pact.agent import SoftwareAttestation

# Software-based attestation for cloud environments
attestation = SoftwareAttestation(
    type="nitro",  # AWS Nitro Enclaves
    config={
        "enclave_cid": 16,
        "attestation_doc_version": "1.0"
    }
)

# Generate attestation document
doc = await attestation.generate(
    user_data=proof.compute_proof_hash(),
    public_key=agent.public_key
)

print(f"Attestation document: {doc.document}")
print(f"PCRs: {doc.pcrs}")
```

### Attestation Verification

```python
from pact.agent import AttestationVerifier

verifier = AttestationVerifier()

# Verify TPM attestation
tpm_result = await verifier.verify_tpm(
    quote=quote,
    expected_pcrs=expected_pcr_values,
    ca_certificates=tpm_ca_certs
)

print(f"TPM attestation valid: {tpm_result.valid}")
print(f"PCRs match: {tpm_result.pcrs_match}")
print(f"Certificate chain valid: {tpm_result.cert_chain_valid}")

# Verify Nitro attestation
nitro_result = await verifier.verify_nitro(
    attestation_doc=doc,
    expected_pcrs=expected_nitro_pcrs,
    root_cert=aws_nitro_root_cert
)
```

## Monitoring and Observability

### Agent Metrics

```python
from pact.agent import MetricsCollector

metrics = MetricsCollector(agent)

# Available metrics
"""
pact_agent_evaluations_total - Total evaluations
pact_agent_evaluation_duration_seconds - Evaluation latency
pact_agent_proofs_generated_total - Proofs generated
pact_agent_proofs_queued - Currently queued proofs
pact_agent_rules_cached - Cached rules count
pact_agent_sync_duration_seconds - Sync latency
pact_agent_online_status - Online/offline status
pact_agent_memory_usage_bytes - Memory usage
"""

# Export to Prometheus
from prometheus_client import start_http_server
start_http_server(9090)

# Custom metrics
metrics.record_custom(
    name="custom_business_metric",
    value=100,
    labels={"domain": "aml", "jurisdiction": "US"}
)
```

### Health Checks

```python
from pact.agent import HealthChecker

health = HealthChecker(agent)

# Liveness check
liveness = await health.check_liveness()
# Returns 200 if agent process is running

# Readiness check
readiness = await health.check_readiness()
print(f"Ready: {readiness.ready}")
print(f"RTVM ready: {readiness.components['rtvm']}")
print(f"Rules cached: {readiness.components['rules']}")
print(f"Control plane connected: {readiness.components['control_plane']}")

# Deep health check
deep = await health.check_deep()
print(f"Rule evaluation test: {deep.evaluation_test}")
print(f"Proof generation test: {deep.proof_test}")
print(f"Sync connectivity test: {deep.sync_test}")
```

### Distributed Tracing

```python
from pact.agent import TracingConfig

tracing = TracingConfig(
    enabled=True,
    exporter="otlp",
    endpoint="https://tracing.company.com:4317",
    service_name="arka-agent",
    sample_rate=0.1  # 10% sampling
)

agent.configure_tracing(tracing)

# Traces include:
# - Evaluation spans with rule execution details
# - Sync operations
# - Proof generation and anchoring
# - Network communication
```

## Security

### Agent Security Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Agent Security Layers                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Network Security                           │ │
│  │  - mTLS for all communications                         │ │
│  │  - Certificate pinning                                 │ │
│  │  - Network isolation                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Identity Security                          │ │
│  │  - DID-based authentication                            │ │
│  │  - Hardware-backed key storage                         │ │
│  │  - Key rotation policies                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Runtime Security                           │ │
│  │  - RTVM sandboxing                                     │ │
│  │  - Resource limits                                     │ │
│  │  - Execution monitoring                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Data Security                              │ │
│  │  - Encryption at rest                                  │ │
│  │  - Secure proof storage                                │ │
│  │  - Audit logging                                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### mTLS Configuration

```yaml
# mtls-config.yaml
security:
  mtls:
    enabled: true

    # Agent certificate
    certificate:
      path: /certs/agent.crt
      keyPath: /certs/agent.key
      keyPassword: ${CERT_KEY_PASSWORD}

    # CA certificates
    ca:
      path: /certs/ca.crt
      verifyClient: true
      verifyServer: true

    # Certificate validation
    validation:
      checkRevocation: true
      ocspEnabled: true
      crlEndpoints:
        - https://crl.arka.io/agents.crl

    # Cipher suites
    cipherSuites:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256

    # Protocol version
    minVersion: TLS1.3
```

### Key Rotation

```python
from pact.agent import KeyManager

key_manager = KeyManager(agent)

# Schedule key rotation
await key_manager.schedule_rotation(
    interval="30d",
    overlap_period="7d"  # Old key valid during transition
)

# Manual rotation
new_did = await key_manager.rotate_keys(
    reason="scheduled_rotation"
)

print(f"New DID: {new_did}")
print(f"Old DID deprecated: {key_manager.deprecated_dids}")

# Verify key rotation completed
status = await key_manager.verify_rotation()
print(f"Rotation complete: {status.complete}")
print(f"New key active: {status.new_key_active}")
print(f"Old key deprecated: {status.old_key_deprecated}")
```

## Configuration Reference

### Complete Agent Configuration

```yaml
# complete-agent-config.yaml
agent:
  # Basic settings
  id: agent-001
  name: "US East Compliance Agent"
  region: us-east-1
  cluster: production

  # Identity
  identity:
    did: did:arka:agent:${AGENT_ID}
    keyStore:
      type: hsm
      provider: aws-cloudhsm
      clusterId: ${HSM_CLUSTER_ID}
      keyLabel: agent-signing-key

  # RTVM
  rtvm:
    maxConcurrency: 100
    timeout: 30s
    memoryLimit: 2Gi
    cpuLimit: 2
    deterministic: true
    timeTravelEnabled: true
    plugins:
      enabled: true
      allowList:
        - pact/aml-plugin
        - pact/kyc-plugin

  # Rules
  rules:
    syncInterval: 5m
    cacheSize: 10000
    preload:
      jurisdictions: [US, EU, UK]
      domains: [AML, KYC, SANCTIONS]
    verification:
      signatures: true
      hashes: true

  # Storage
  storage:
    type: persistent
    path: /data
    encryption:
      enabled: true
      algorithm: aes-256-gcm
      keySource: hsm

  # Communication
  communication:
    controlPlane:
      endpoint: https://control.arka.io
      timeout: 30s
      retries: 3
    grpc:
      maxMessageSize: 10Mi
      keepalive: 30s

  # Offline
  offline:
    enabled: true
    maxDuration: 24h
    proofQueueSize: 10000

  # Security
  security:
    mtls:
      enabled: true
      certPath: /certs/agent.crt
      keyPath: /certs/agent.key
      caPath: /certs/ca.crt
    attestation:
      hardware: true
      tpmDevice: /dev/tpm0

  # Monitoring
  monitoring:
    metrics:
      enabled: true
      port: 9090
    tracing:
      enabled: true
      endpoint: ${TRACING_ENDPOINT}
      sampleRate: 0.1
    logging:
      level: info
      format: json
```

## Best Practices

### Deployment

1. **High Availability**: Deploy multiple agents per region
2. **Resource Isolation**: Use dedicated compute for compliance
3. **Network Segmentation**: Isolate agent network traffic
4. **Regular Updates**: Keep agents updated with latest rules

### Security

1. **Hardware Keys**: Use HSM for production key storage
2. **Certificate Management**: Implement automated certificate rotation
3. **Audit Logging**: Enable comprehensive audit logging
4. **Least Privilege**: Minimize agent permissions

### Performance

1. **Rule Caching**: Pre-cache frequently used rules
2. **Batch Processing**: Use batch evaluation for high throughput
3. **Connection Pooling**: Pool connections to control plane
4. **Resource Limits**: Set appropriate memory and CPU limits

### Resilience

1. **Offline Capability**: Always enable offline mode
2. **Proof Queuing**: Configure adequate queue size
3. **Retry Logic**: Implement exponential backoff
4. **Health Monitoring**: Monitor agent health continuously

## Troubleshooting

### Common Issues

**Agent Not Connecting to Control Plane**
```bash
# Check connectivity
curl -v https://control.arka.io/health

# Verify certificates
openssl s_client -connect control.arka.io:443

# Check agent logs
kubectl logs -f deployme../arka-agent -n compliance
```

**Rule Sync Failing**
```python
# Force full sync
await sync_manager.force_full_sync()

# Check sync status
status = await sync_manager.get_status()
print(f"Last error: {status.last_error}")
```

**Proof Queue Growing**
```python
# Check queue status
stats = await queue.get_stats()
print(f"Queue size: {stats.total}")
print(f"Failed proofs: {stats.failed}")

# Retry failed proofs
await queue.retry_failed()
```

## Related Documentation

- [RTVM Reference](./rtvm.md)
- [Blockchain Anchoring](./blockchain-anchoring.md)
- [Security Model](../security/security-model.md)
- [Deployment Guide](../deployment/deployment-guide.md)
