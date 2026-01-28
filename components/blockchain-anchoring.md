# Blockchain Anchoring System

The Blockchain Anchoring System provides immutable proof of rule execution, version history, and compliance attestations through multi-ledger anchoring strategies.

## Overview

The ARKA Engine uses blockchain technology to create tamper-evident records of regulatory rules, their execution, and compliance outcomes. This provides cryptographic proof that can be independently verified by regulators, auditors, and counterparties.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Blockchain Anchoring System                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────────┐   │
│  │   Merkle     │    │    Rule      │    │      DID                 │   │
│  │   Anchoring  │    │    NFTs      │    │      Registry            │   │
│  └──────────────┘    └──────────────┘    └──────────────────────────┘   │
│         │                   │                        │                   │
│         └───────────────────┼────────────────────────┘                   │
│                             ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                     Multi-Ledger Adapter                             ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   ││
│  │  │Ethereum │  │ Polygon │  │Hyperledg│  │  Solana │  │  Custom │   ││
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘   ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### Merkle Tree Anchoring

The Merkle Tree Anchoring system batches multiple proofs into a single blockchain transaction for cost efficiency.

#### Architecture

```
                    ┌─────────────────┐
                    │   Merkle Root   │
                    │     (Anchor)    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
        ┌─────┴─────┐                 ┌─────┴─────┐
        │  Hash AB  │                 │  Hash CD  │
        └─────┬─────┘                 └─────┬─────┘
              │                             │
       ┌──────┴──────┐               ┌──────┴──────┐
       │             │               │             │
   ┌───┴───┐    ┌────┴───┐      ┌────┴───┐   ┌────┴───┐
   │Proof A│    │Proof B │      │Proof C │   │Proof D │
   └───────┘    └────────┘      └────────┘   └────────┘
```

#### Proof Structure

```python
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

@dataclass
class TrustProof:
    """Cryptographic proof of compliant execution."""
    proof_id: str
    timestamp: datetime
    rule_id: str
    rule_version: str
    input_hash: str
    output_hash: str
    execution_hash: str
    rtvm_state_hash: str
    jurisdiction: str
    agent_did: str
    signature: str

@dataclass
class MerkleProof:
    """Proof of inclusion in Merkle tree."""
    leaf_hash: str
    proof_path: List[str]
    proof_index: int
    root_hash: str

@dataclass
class AnchorRecord:
    """Blockchain anchor record."""
    anchor_id: str
    merkle_root: str
    chain_id: str
    transaction_hash: str
    block_number: int
    block_timestamp: datetime
    proof_count: int
    anchor_timestamp: datetime
```

#### Anchoring Configuration

```yaml
# anchoring-config.yaml
anchoring:
  # Batching settings
  batching:
    maxBatchSize: 1000
    maxBatchAge: 1h
    minBatchSize: 10
    batchTrigger:
      - type: size
        threshold: 500
      - type: time
        interval: 30m
      - type: priority
        levels: [critical, high]

  # Merkle tree settings
  merkle:
    hashAlgorithm: sha3-256
    leafPrefix: "0x00"
    nodePrefix: "0x01"
    sortLeaves: true

  # Chain selection
  chains:
    primary: ethereum
    fallback: polygon
    selectionStrategy: cost_optimized

  # Retention
  retention:
    proofRetention: 7y
    anchorRetention: permanent
    archiveAfter: 1y
```

#### Using the Anchoring API

```python
from pact.blockchain import AnchoringService, TrustProof

anchoring = AnchoringService(config_path="anchoring-config.yaml")

# Create trust proof
proof = TrustProof(
    proof_id="proof-123",
    timestamp=datetime.utcnow(),
    rule_id="rule/aml/transaction-threshold",
    rule_version="v1.2.0",
    input_hash=hash_input(transaction_data),
    output_hash=hash_output(evaluation_result),
    execution_hash=rtvm.execution_hash,
    rtvm_state_hash=rtvm.state_hash,
    jurisdiction="US",
    agent_did="did:arka:agent:node-001",
    signature=sign_proof(proof_data)
)

# Submit for anchoring
anchor_request = await anchoring.submit(proof)
print(f"Proof queued: {anchor_request.id}")
print(f"Estimated anchor time: {anchor_request.estimated_anchor_time}")

# Wait for anchoring (async)
anchor_result = await anchoring.wait_for_anchor(anchor_request.id)
print(f"Anchored in block: {anchor_result.block_number}")
print(f"Transaction: {anchor_result.transaction_hash}")
```

#### Batch Anchoring

```python
# Submit multiple proofs
proofs = [proof1, proof2, proof3, ...]

batch_request = await anchoring.submit_batch(
    proofs=proofs,
    priority="normal",
    callback_url="https://api.company.com/anchor-callback"
)

# Monitor batch status
status = await anchoring.get_batch_status(batch_request.id)
print(f"Batch size: {status.proof_count}")
print(f"Status: {status.state}")
print(f"Progress: {status.anchored_count}/{status.proof_count}")
```

### Multi-Ledger Support

The system supports multiple blockchain networks with automatic failover and cost optimization.

#### Supported Chains

| Chain | Type | Use Case | Cost | Finality |
|-------|------|----------|------|----------|
| Ethereum | Public | High-value anchors | High | ~15 min |
| Polygon | Public L2 | Standard anchors | Low | ~2 min |
| Arbitrum | Public L2 | High throughput | Low | ~1 min |
| Hyperledger Fabric | Private | Enterprise | None | Immediate |
| Solana | Public | High frequency | Very Low | ~1 sec |
| Custom | Private | Specialized | Variable | Variable |

#### Chain Configuration

```yaml
# chains-config.yaml
chains:
  ethereum:
    enabled: true
    type: public
    rpcEndpoints:
      - https://mainnet.infura.io/v3/${INFURA_KEY}
      - https://eth-mainnet.alchemyapi.io/v2/${ALCHEMY_KEY}
    contractAddress: "0x1234567890abcdef..."
    confirmations: 12
    gasStrategy: eip1559
    maxGasPrice: 100gwei
    priority: high

  polygon:
    enabled: true
    type: public_l2
    rpcEndpoints:
      - https://polygon-rpc.com
      - https://rpc-mainnet.matic.network
    contractAddress: "0xabcdef1234567890..."
    confirmations: 128
    gasStrategy: legacy
    maxGasPrice: 500gwei
    priority: normal

  hyperledger:
    enabled: true
    type: private
    connectionProfile: /config/hyperledger-connection.yaml
    channelName: compliance-channel
    chaincodeName: arka-anchor
    identity:
      mspId: Org1MSP
      certPath: /certs/user.pem
      keyPath: /certs/user.key
    priority: enterprise

  solana:
    enabled: true
    type: public
    rpcEndpoints:
      - https://api.mainnet-beta.solana.com
    programId: "Pact1111111111111111111111111111111111111"
    commitment: finalized
    priority: low_latency
```

#### Chain Selection Strategy

```python
from pact.blockchain import ChainSelector, SelectionStrategy

selector = ChainSelector(
    strategy=SelectionStrategy.COST_OPTIMIZED,
    constraints={
        "max_latency": "5m",
        "min_security": "high",
        "budget_per_day": 100  # USD
    }
)

# Automatic chain selection
selected_chain = await selector.select(
    proof_count=100,
    priority="normal",
    required_finality="15m"
)

print(f"Selected chain: {selected_chain.name}")
print(f"Estimated cost: ${selected_chain.estimated_cost}")
print(f"Estimated time: {selected_chain.estimated_time}")

# Override with specific chain
anchor_result = await anchoring.anchor(
    proofs=proofs,
    chain="ethereum",
    override_selection=True
)
```

### Rule NFTs

Rules can be tokenized as NFTs to establish ownership, versioning, and licensing.

#### NFT Structure

```
┌─────────────────────────────────────────────────────────────┐
│                       Rule NFT                               │
├─────────────────────────────────────────────────────────────┤
│  Token ID: 12345                                             │
│  Owner: did:arka:org:company-xyz                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Metadata                           │    │
│  │  - Rule ID: rule/aml/transaction-threshold          │    │
│  │  - Version: v1.2.0                                  │    │
│  │  - Jurisdiction: US                                 │    │
│  │  - Domain: AML                                      │    │
│  │  - Created: 2024-01-15T10:30:00Z                   │    │
│  │  - Author: did:arka:agent:author-001               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Version History                      │    │
│  │  v1.0.0 -> v1.1.0 -> v1.2.0 (current)              │    │
│  │     │         │         │                           │    │
│  │    IPFS     IPFS      IPFS                         │    │
│  │    Hash     Hash      Hash                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Licensing                          │    │
│  │  - License Type: Enterprise                         │    │
│  │  - Allowed Uses: [evaluation, audit]               │    │
│  │  - Restrictions: [modification, redistribution]    │    │
│  │  - Expiry: 2025-01-15                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### NFT Contract Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract ARKARuleNFT is ERC721, AccessControl {
    struct RuleMetadata {
        string ruleId;
        string version;
        string jurisdiction;
        string domain;
        string contentHash;  // IPFS hash
        uint256 createdAt;
        address author;
    }

    struct License {
        LicenseType licenseType;
        uint256 expiry;
        bool transferable;
        string[] allowedUses;
    }

    enum LicenseType { OpenSource, Commercial, Enterprise, Regulatory }

    mapping(uint256 => RuleMetadata) public ruleMetadata;
    mapping(uint256 => License) public licenses;
    mapping(uint256 => uint256[]) public versionHistory;

    event RuleMinted(
        uint256 indexed tokenId,
        string ruleId,
        string version,
        address author
    );

    event RuleUpdated(
        uint256 indexed tokenId,
        string newVersion,
        string newContentHash
    );

    function mintRule(
        address to,
        string memory ruleId,
        string memory version,
        string memory jurisdiction,
        string memory domain,
        string memory contentHash,
        License memory license
    ) external returns (uint256) {
        uint256 tokenId = uint256(keccak256(abi.encodePacked(ruleId)));

        _safeMint(to, tokenId);

        ruleMetadata[tokenId] = RuleMetadata({
            ruleId: ruleId,
            version: version,
            jurisdiction: jurisdiction,
            domain: domain,
            contentHash: contentHash,
            createdAt: block.timestamp,
            author: msg.sender
        });

        licenses[tokenId] = license;

        emit RuleMinted(tokenId, ruleId, version, msg.sender);

        return tokenId;
    }

    function updateRule(
        uint256 tokenId,
        string memory newVersion,
        string memory newContentHash
    ) external {
        require(ownerOf(tokenId) == msg.sender, "Not owner");

        versionHistory[tokenId].push(
            uint256(keccak256(abi.encodePacked(
                ruleMetadata[tokenId].contentHash
            )))
        );

        ruleMetadata[tokenId].version = newVersion;
        ruleMetadata[tokenId].contentHash = newContentHash;

        emit RuleUpdated(tokenId, newVersion, newContentHash);
    }

    function verifyRule(
        uint256 tokenId,
        string memory contentHash
    ) external view returns (bool) {
        return keccak256(bytes(ruleMetadata[tokenId].contentHash)) ==
               keccak256(bytes(contentHash));
    }
}
```

#### NFT API

```python
from pact.blockchain import RuleNFTManager

nft_manager = RuleNFTManager(
    chain="polygon",
    contract_address="0x1234..."
)

# Mint rule NFT
token_id = await nft_manager.mint(
    rule_id="rule/aml/transaction-threshold",
    version="v1.2.0",
    jurisdiction="US",
    domain="AML",
    content_hash=ipfs_hash,
    license={
        "type": "enterprise",
        "expiry": "2025-01-15",
        "transferable": True,
        "allowed_uses": ["evaluation", "audit"]
    }
)

print(f"Minted NFT: {token_id}")

# Update rule version
await nft_manager.update(
    token_id=token_id,
    new_version="v1.3.0",
    new_content_hash=new_ipfs_hash
)

# Verify rule authenticity
is_valid = await nft_manager.verify(
    token_id=token_id,
    content_hash=claimed_hash
)

# Get version history
history = await nft_manager.get_history(token_id)
for version in history:
    print(f"Version: {version.version}")
    print(f"Hash: {version.content_hash}")
    print(f"Timestamp: {version.timestamp}")
```

### DID Registry

Decentralized Identifiers (DIDs) provide cryptographic identity for agents, organizations, and rules.

#### DID Methods

```
┌─────────────────────────────────────────────────────────────┐
│                      DID Registry                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  did:arka:agent:node-001                                    │
│  ├── Controller: did:arka:org:company-xyz                   │
│  ├── Public Key: ed25519:abc123...                         │
│  ├── Service Endpoints:                                     │
│  │   ├── compliance: https://node-001.company.com/api      │
│  │   └── attestation: https://node-001.company.com/attest  │
│  └── Capabilities:                                          │
│      ├── rule_execution                                     │
│      ├── proof_generation                                   │
│      └── attestation_signing                                │
│                                                              │
│  did:arka:org:company-xyz                                   │
│  ├── Controller: self                                       │
│  ├── Public Keys:                                           │
│  │   ├── master: secp256k1:def456...                       │
│  │   └── signing: ed25519:ghi789...                        │
│  ├── Verification Methods:                                  │
│  │   ├── authentication                                     │
│  │   └── assertionMethod                                    │
│  └── Managed Agents: [node-001, node-002, ...]             │
│                                                              │
│  did:arka:rule:aml-threshold-v1                             │
│  ├── Controller: did:arka:org:regulator-abc                │
│  ├── Content Hash: QmXyz...                                │
│  ├── Jurisdiction: US                                       │
│  └── Attestations: [attestation-1, attestation-2, ...]     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### DID Document

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://arka.io/ns/did/v1"
  ],
  "id": "did:arka:agent:node-001",
  "controller": "did:arka:org:company-xyz",
  "verificationMethod": [
    {
      "id": "did:arka:agent:node-001#keys-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:arka:agent:node-001",
      "publicKeyMultibase": "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
    }
  ],
  "authentication": [
    "did:arka:agent:node-001#keys-1"
  ],
  "assertionMethod": [
    "did:arka:agent:node-001#keys-1"
  ],
  "service": [
    {
      "id": "did:arka:agent:node-001#compliance",
      "type": "ComplianceEndpoint",
      "serviceEndpoint": "https://node-001.company.com/api"
    },
    {
      "id": "did:arka:agent:node-001#attestation",
      "type": "AttestationService",
      "serviceEndpoint": "https://node-001.company.com/attest"
    }
  ],
  "pact:capabilities": [
    "rule_execution",
    "proof_generation",
    "attestation_signing"
  ],
  "pact:attestations": [
    {
      "type": "SecurityAudit",
      "issuer": "did:arka:org:auditor-xyz",
      "issuanceDate": "2024-01-15T00:00:00Z",
      "expirationDate": "2025-01-15T00:00:00Z",
      "proof": "..."
    }
  ]
}
```

#### DID API

```python
from pact.blockchain import DIDRegistry, DIDDocument

registry = DIDRegistry(
    method="arka",
    resolver_endpoints=[
        "https://resolver.arka.io",
        "https://resolver-backup.arka.io"
    ]
)

# Create agent DID
agent_did = await registry.create(
    type="agent",
    controller="did:arka:org:company-xyz",
    public_key=agent_public_key,
    capabilities=["rule_execution", "proof_generation"],
    service_endpoints={
        "compliance": "https://agent.company.com/api",
        "attestation": "https://agent.company.com/attest"
    }
)

print(f"Created DID: {agent_did.id}")

# Resolve DID
did_document = await registry.resolve("did:arka:agent:node-001")
print(f"Controller: {did_document.controller}")
print(f"Capabilities: {did_document.capabilities}")

# Update DID document
await registry.update(
    did="did:arka:agent:node-001",
    updates={
        "add_service": {
            "id": "#metrics",
            "type": "MetricsEndpoint",
            "serviceEndpoint": "https://agent.company.com/metrics"
        }
    },
    signature=sign_update(update_data)
)

# Revoke DID
await registry.revoke(
    did="did:arka:agent:node-001",
    reason="Agent decommissioned",
    signature=sign_revocation(revocation_data)
)
```

### Zero-Knowledge Proofs

ZK proofs enable privacy-preserving compliance verification.

#### ZK Proof Types

| Proof Type | Use Case | Privacy Level |
|------------|----------|---------------|
| Range Proof | Amount within threshold | High |
| Set Membership | Jurisdiction validation | High |
| Computation Proof | Rule execution verification | Medium |
| Aggregate Proof | Batch compliance | High |

#### ZK Circuit Example

```python
from pact.blockchain.zk import ZKCircuit, ZKProver, ZKVerifier

# Define compliance circuit
class AMLThresholdCircuit(ZKCircuit):
    """ZK circuit for AML threshold compliance."""

    def define(self):
        # Private inputs (not revealed)
        amount = self.private_input("amount")
        account_balance = self.private_input("account_balance")

        # Public inputs (revealed)
        threshold = self.public_input("threshold")
        jurisdiction = self.public_input("jurisdiction")

        # Constraints
        self.assert_less_than(amount, threshold)
        self.assert_greater_than(account_balance, 0)

        # Output commitment
        self.output_commitment(
            hash(amount, account_balance, threshold, jurisdiction)
        )

# Generate proving/verification keys
circuit = AMLThresholdCircuit()
proving_key, verification_key = circuit.setup()

# Create prover
prover = ZKProver(circuit, proving_key)

# Generate proof
proof = await prover.prove(
    private_inputs={
        "amount": 5000,
        "account_balance": 50000
    },
    public_inputs={
        "threshold": 10000,
        "jurisdiction": "US"
    }
)

# Verify proof (anyone can verify with public inputs)
verifier = ZKVerifier(circuit, verification_key)

is_valid = await verifier.verify(
    proof=proof,
    public_inputs={
        "threshold": 10000,
        "jurisdiction": "US"
    }
)

print(f"Proof valid: {is_valid}")
# Output: Proof valid: True
# Note: The actual amount (5000) was never revealed
```

#### ZK Compliance Attestation

```python
from pact.blockchain.zk import ZKAttestationService

zk_service = ZKAttestationService()

# Generate ZK attestation
attestation = await zk_service.create_attestation(
    rule_id="rule/aml/transaction-threshold",
    compliance_result=True,
    private_data={
        "transaction_amount": 5000,
        "customer_id": "cust-123",
        "account_number": "acct-456"
    },
    public_data={
        "jurisdiction": "US",
        "rule_version": "v1.2.0",
        "timestamp": datetime.utcnow().isoformat()
    }
)

# Attestation can be shared without revealing private data
print(f"Attestation ID: {attestation.id}")
print(f"Public commitment: {attestation.commitment}")
print(f"Proof: {attestation.proof}")

# Third-party verification
verification = await zk_service.verify_attestation(attestation)
print(f"Verified: {verification.valid}")
print(f"Compliance: {verification.compliant}")
# Private data (amount, customer_id) never revealed
```

## Cross-Chain Operations

### Cross-Chain Verification

```python
from pact.blockchain import CrossChainVerifier

verifier = CrossChainVerifier()

# Verify anchor across chains
verification = await verifier.verify_anchor(
    anchor_id="anchor-123",
    chains=["ethereum", "polygon", "hyperledger"]
)

print(f"Verified on {len(verification.confirmed_chains)} chains")
for chain in verification.confirmed_chains:
    print(f"  {chain.name}: Block {chain.block_number}")

# Cross-chain proof aggregation
aggregated_proof = await verifier.aggregate_proofs(
    proof_ids=["proof-1", "proof-2", "proof-3"],
    target_chain="ethereum"
)
```

### Bridge Operations

```python
from pact.blockchain import ChainBridge

bridge = ChainBridge(
    source_chain="polygon",
    target_chain="ethereum"
)

# Bridge proof to another chain
bridge_tx = await bridge.bridge_proof(
    proof_id="proof-123",
    reason="regulatory_audit"
)

print(f"Bridge transaction: {bridge_tx.hash}")
print(f"Expected confirmation: {bridge_tx.estimated_time}")

# Wait for bridge completion
result = await bridge.wait_for_completion(bridge_tx.id)
print(f"Bridged to {result.target_chain}: {result.target_tx_hash}")
```

## Verification APIs

### Proof Verification

```python
from pact.blockchain import ProofVerifier

verifier = ProofVerifier()

# Verify single proof
result = await verifier.verify(
    proof_id="proof-123",
    expected_rule="rule/aml/transaction-threshold",
    expected_version="v1.2.0"
)

print(f"Proof valid: {result.valid}")
print(f"Anchored: {result.anchored}")
print(f"Chain: {result.chain}")
print(f"Block: {result.block_number}")
print(f"Merkle path valid: {result.merkle_valid}")

# Verify with full audit trail
audit = await verifier.full_audit(
    proof_id="proof-123",
    include_execution_trace=True
)

print(f"Execution verified: {audit.execution_valid}")
print(f"Input hash matches: {audit.input_hash_valid}")
print(f"Output hash matches: {audit.output_hash_valid}")
print(f"Agent signature valid: {audit.signature_valid}")
```

### Batch Verification

```python
# Verify multiple proofs efficiently
batch_result = await verifier.verify_batch(
    proof_ids=["proof-1", "proof-2", "proof-3", ...],
    parallel=True
)

print(f"Total: {batch_result.total}")
print(f"Valid: {batch_result.valid_count}")
print(f"Invalid: {batch_result.invalid_count}")

for invalid in batch_result.invalid_proofs:
    print(f"  {invalid.proof_id}: {invalid.reason}")
```

## Configuration Reference

### Complete Configuration

```yaml
# blockchain-config.yaml
blockchain:
  # General settings
  enabled: true
  defaultChain: polygon

  # Anchoring settings
  anchoring:
    enabled: true
    strategy: merkle_batch
    batching:
      maxSize: 1000
      maxAge: 1h
      minSize: 10
    retry:
      maxAttempts: 3
      backoffMs: 1000

  # NFT settings
  nft:
    enabled: true
    chain: polygon
    contractAddress: "0x..."
    ipfsGateway: https://ipfs.io/ipfs/

  # DID settings
  did:
    method: pact
    registry: https://registry.arka.io
    resolvers:
      - https://resolver.arka.io
      - https://resolver-backup.arka.io
    cache:
      enabled: true
      ttl: 1h

  # ZK settings
  zk:
    enabled: true
    provingBackend: rapidsnark
    verificationBackend: snarkjs
    circuitsPath: /circuits
    trustedSetup: /setup/powers_of_tau.ptau

  # Chain configurations
  chains:
    ethereum:
      enabled: true
      rpcEndpoints:
        - ${ETH_RPC_ENDPOINT}
      contractAddress: ${ETH_CONTRACT_ADDRESS}
      privateKey: ${ETH_PRIVATE_KEY}
      confirmations: 12
      gasLimit: 500000

    polygon:
      enabled: true
      rpcEndpoints:
        - ${POLYGON_RPC_ENDPOINT}
      contractAddress: ${POLYGON_CONTRACT_ADDRESS}
      privateKey: ${POLYGON_PRIVATE_KEY}
      confirmations: 128

    hyperledger:
      enabled: true
      connectionProfile: /config/hyperledger.yaml
      identity:
        mspId: ${HL_MSP_ID}
        certPath: ${HL_CERT_PATH}
        keyPath: ${HL_KEY_PATH}

  # Monitoring
  monitoring:
    metricsEnabled: true
    alerting:
      anchorFailure:
        threshold: 3
        window: 5m
      chainLatency:
        threshold: 10m
        chains: [ethereum, polygon]
```

## Best Practices

### Anchoring Strategy

1. **Batch Appropriately**: Balance cost vs. latency based on requirements
2. **Multi-Chain Redundancy**: Anchor critical proofs to multiple chains
3. **Monitor Gas Prices**: Implement dynamic gas strategies
4. **Handle Reorgs**: Account for blockchain reorganizations

### Security

1. **Key Management**: Use HSM for production private keys
2. **Signature Verification**: Always verify signatures before anchoring
3. **DID Validation**: Verify DID documents before trusting
4. **ZK Circuit Audit**: Have ZK circuits professionally audited

### Performance

1. **Parallel Verification**: Use batch and parallel verification
2. **Cache DID Documents**: Implement caching with appropriate TTL
3. **Optimize ZK Proofs**: Use efficient proof systems (Groth16, PLONK)
4. **Connection Pooling**: Pool RPC connections to blockchain nodes

## Troubleshooting

### Common Issues

**Anchor Transaction Failed**
```python
# Check transaction status
status = await anchoring.get_transaction_status(tx_hash)
if status.failed:
    print(f"Failure reason: {status.error}")
    # Retry with higher gas
    await anchoring.retry(anchor_id, gas_multiplier=1.5)
```

**DID Resolution Failed**
```python
# Try backup resolvers
did_doc = await registry.resolve(
    did,
    use_backup=True,
    timeout=30
)
```

**ZK Proof Generation Slow**
```python
# Use hardware acceleration
prover = ZKProver(
    circuit,
    proving_key,
    backend="cuda"  # Use GPU
)
```

## Related Documentation

- [RTVM Reference](./rtvm.md)
- [Distributed Agents](./distributed-agents.md)
- [Security Model](../security/security-model.md)
- [REST API Reference](../api-reference/rest-api.md)
