# ARKA Blockchain

## Overview

The ARKA Blockchain is a purpose-built distributed ledger for recording immutable compliance events. It provides cryptographic proof of compliance decisions, audit trails, and regulatory evidence that cannot be tampered with.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          ARKA Blockchain Layer                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                        arka-chain-node-api                               │ │
│  │                    HTTP REST API - Port 4040                             │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                          @arka/chain-node                                │ │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────────┐    │ │
│  │  │    Mempool    │  │    Miner      │  │    Transaction Pool       │    │ │
│  │  │  (Pending TX) │  │  (Block Prod) │  │    (Validation)           │    │ │
│  │  └───────────────┘  └───────────────┘  └───────────────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                     │                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                          @arka/chain-core                                │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────┐   │ │
│  │  │  Hash   │  │ Merkle  │  │  Block  │  │  Chain  │  │   Storage   │   │ │
│  │  │ SHA-256 │  │  Tree   │  │ Builder │  │  State  │  │ Memory/File │   │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────┐  ┌────────────────────────────────────────┐ │
│  │    @arka/chain-client      │  │         arka-chain-explorer            │ │
│  │  - Local Client            │  │       Web UI - Port 4041               │ │
│  │  - HTTP Client             │  │  - Block browser                       │ │
│  │  - Cloud Integration       │  │  - Transaction viewer                  │ │
│  └────────────────────────────┘  └────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Key Concepts

### Blocks

Blocks are containers for transactions, linked together in a chain through cryptographic hashes.

```typescript
interface Block {
  header: {
    chainId: string;        // Chain identifier
    height: number;         // Block number (0 = genesis)
    previousHash: string;   // SHA-256 of previous block
    timestamp: string;      // ISO 8601 timestamp
    merkleRoot: string;     // Merkle root of transactions
    proposerId: string;     // Node that created block
    txCount: number;        // Number of transactions
  };
  transactions: Transaction[];
  hash: string;             // SHA-256 of block
}
```

### Transactions

Transactions represent compliance events recorded on the chain.

```typescript
// ARKA Event Transaction
interface ArkaEventTransaction {
  type: 'ARKA_EVENT';
  hash: string;           // SHA-256 transaction hash
  timestamp: string;      // When created
  eventId: string;        // Unique event identifier
  entityId: string;       // Related entity
  ruleId?: string;        // Rule that was evaluated
  decisionId?: string;    // Decision reference
  source?: string;        // Source system/plugin
  eventType?: string;     // Event classification
  payload: unknown;       // Event data
}

// Governance Transaction
interface GovernanceTransaction {
  type: 'GOVERNANCE';
  hash: string;
  timestamp: string;
  action: 'PROPOSAL_CREATED' | 'VOTE_CAST' | 'PROPOSAL_EXECUTED' | 'RULE_UPDATED';
  actorId: string;
  proposalId?: string;
  payload: unknown;
}

// Audit Transaction
interface AuditTransaction {
  type: 'AUDIT';
  hash: string;
  timestamp: string;
  action: 'ACCESS' | 'MODIFICATION' | 'EXPORT' | 'QUERY';
  actorId: string;
  resourceId: string;
  resourceType: string;
  payload: unknown;
}
```

### Merkle Trees

Transactions are organized in a Merkle tree for efficient verification.

```
                    Merkle Root
                    /         \
               Hash(0,1)    Hash(2,3)
               /     \      /     \
            Hash0  Hash1  Hash2  Hash3
              |      |      |      |
            Tx0    Tx1    Tx2    Tx3
```

---

## Packages

### @arka/chain-core

The foundational blockchain engine providing cryptographic primitives, block creation, and storage.

**Key Components:**

| Component | Description |
|-----------|-------------|
| `hash.ts` | SHA-256 hashing utilities |
| `merkle.ts` | Merkle tree implementation |
| `block.ts` | Block creation and validation |
| `chain.ts` | Chain state management |
| `storage.ts` | Pluggable storage backends |
| `transaction.ts` | Transaction creation and validation |

**Usage:**

```typescript
import {
  createChain,
  createGenesisBlock,
  createArkaEventTransaction,
  hashString,
  computeMerkleRoot,
} from '@arka/chain-core';

// Create a new chain
const chain = await createChain({
  chainId: 'arka-main-chain',
  storage: 'memory', // or 'file'
});

// Create a transaction
const tx = createArkaEventTransaction({
  type: 'ARKA_EVENT',
  eventId: 'evt-001',
  entityId: 'customer-123',
  eventType: 'KYC_VERIFIED',
  payload: { status: 'approved' },
});

// Append block with transactions
const block = await chain.appendBlock([tx], 'node-1');

// Query chain
const blockByHeight = await chain.getBlockByHeight(0);
const blockByHash = await chain.getBlockByHash(block.hash);
const transaction = await chain.getTransaction(tx.hash);
```

**Hash Utilities:**

```typescript
import { hashString, hashObject, randomHex } from '@arka/chain-core';

// Hash a string
const hash = hashString('hello world');
// -> 'b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9'

// Hash an object (deterministic)
const objHash = hashObject({ foo: 'bar', num: 42 });

// Generate random hex
const nonce = randomHex(16); // 16 bytes = 32 hex chars
```

**Merkle Tree:**

```typescript
import { computeMerkleRoot, generateMerkleProof, verifyMerkleProof } from '@arka/chain-core';

const txHashes = ['hash1', 'hash2', 'hash3', 'hash4'];

// Compute root
const root = computeMerkleRoot(txHashes);

// Generate proof for a transaction
const proof = generateMerkleProof(txHashes, 1); // proof for hash2

// Verify proof
const isValid = verifyMerkleProof('hash2', proof, root);
```

**Storage Backends:**

```typescript
import { createStorage, MemoryStorage, FileStorage } from '@arka/chain-core';

// In-memory (default, for testing)
const memStorage = createStorage({ type: 'memory' });

// File-based (persistent)
const fileStorage = createStorage({
  type: 'file',
  path: './data/blockchain',
});

// Use with chain
const chain = await createChain({
  chainId: 'my-chain',
  storage: fileStorage,
});
```

---

### @arka/chain-node

Node service providing mempool management, block production, and transaction processing.

**Usage:**

```typescript
import { createChainNode, createAndInitNode } from '@arka/chain-node';

// Create and initialize node
const node = await createAndInitNode({
  nodeId: 'node-1',
  chainId: 'arka-dev-chain',
  maxTxPerBlock: 1000,
  maxMempoolSize: 10000,
});

// Submit transaction
const txHash = await node.submitTransaction(tx);

// Mine a block
const block = await node.mineBlock();

// Query
const state = node.getChainState();
const status = node.getStatus();
const pendingCount = node.getPendingTxCount();

// Graceful shutdown
await node.shutdown();
```

**Mempool:**

```typescript
import { Mempool } from '@arka/chain-node';

const mempool = new Mempool({ maxSize: 10000 });

// Add transaction
const added = mempool.add(tx);

// Take transactions for block
const txsForBlock = mempool.take(100); // Take up to 100 txs

// Check if transaction exists
const exists = mempool.has(txHash);

// Get pending count
const pending = mempool.size();
```

**Node Status:**

```typescript
interface NodeStatus {
  nodeId: string;
  chainId: string;
  height: number;
  pendingTxCount: number;
  isSyncing: boolean;
  peerCount: number;
  uptimeMs: number;
  version: string;
}
```

---

### @arka/chain-client

Client library for interacting with blockchain nodes.

**Local Client (in-process):**

```typescript
import { createLocalPactChainClient } from '@arka/chain-client';

const client = createLocalPactChainClient(node);

// Submit transaction
const result = await client.submitEventTransaction({
  type: 'ARKA_EVENT',
  eventId: 'evt-001',
  entityId: 'entity-123',
  payload: { data: 'example' },
});
console.log(result.txHash);

// Query
const state = await client.getChainState();
const stats = await client.getChainStats();
const block = await client.getBlockByHeight(0);
const txLookup = await client.lookupTransaction(txHash);

// Mine (dev only)
const newBlock = await client.triggerMine();
```

**HTTP Client (remote):**

```typescript
import { createHttpPactChainClient } from '@arka/chain-client';

const client = createHttpPactChainClient({
  baseUrl: 'http://localhost:4040',
  timeout: 30000,
  retryCount: 3,
});

// Same API as local client
const result = await client.submitEventTransaction({
  type: 'ARKA_EVENT',
  eventId: 'evt-002',
  entityId: 'entity-456',
  payload: { action: 'approved' },
});
```

**Cloud Integration:**

```typescript
import { PactCloudChainIntegration } from '@arka/chain-client';

const integration = new PactCloudChainIntegration({
  chainClient: client,
  eventMapping: {
    'kyc.verified': 'KYC_VERIFICATION',
    'transaction.screened': 'TRANSACTION_SCREENED',
    'alert.generated': 'COMPLIANCE_ALERT',
  },
});

// Record ARKA Cloud event on blockchain
await integration.recordEvent({
  type: 'kyc.verified',
  entityId: 'customer-123',
  data: { level: 'enhanced', score: 95 },
});

// Batch record events
await integration.recordEventBatch([
  { type: 'transaction.screened', entityId: 'tx-001', data: {...} },
  { type: 'transaction.screened', entityId: 'tx-002', data: {...} },
]);
```

---

## arka-chain-node-api

HTTP REST API for the blockchain node.

**Base URL:** `http://localhost:4040`

### Endpoints

#### Health Check
```
GET /health
```
```json
{
  "status": "ok",
  "nodeId": "node-local",
  "chainId": "arka-dev-chain",
  "height": 5,
  "pendingTx": 0,
  "uptimeMs": 123456,
  "version": "0.1.0"
}
```

#### Node Status
```
GET /status
```
```json
{
  "nodeId": "node-local",
  "chainId": "arka-dev-chain",
  "height": 5,
  "pendingTxCount": 3,
  "isSyncing": false,
  "peerCount": 0,
  "uptimeMs": 123456,
  "version": "0.1.0"
}
```

#### Submit ARKA Event Transaction
```
POST /chain/tx/arka-event
Content-Type: application/json

{
  "eventId": "evt-kyc-001",
  "entityId": "customer-12345",
  "eventType": "KYC_VERIFICATION",
  "ruleId": "rule-kyc-basic",
  "source": "arka-kyc-plugin",
  "payload": {
    "verificationLevel": "enhanced",
    "riskScore": 25,
    "status": "approved"
  }
}
```
Response:
```json
{
  "txHash": "a515a9b6c664e7b5023a8b98c2992d2ef6721aaf...",
  "status": "pending"
}
```

#### Look Up Transaction
```
GET /chain/tx/:txHash
```
```json
{
  "transaction": {
    "type": "ARKA_EVENT",
    "hash": "a515a9b6c664e7b5...",
    "eventId": "evt-kyc-001",
    "entityId": "customer-12345",
    ...
  },
  "block": {
    "height": 3,
    "hash": "b94d27b9934d3e08...",
    "timestamp": "2024-01-10T10:00:00.000Z",
    "txCount": 5,
    "proposerId": "node-local"
  },
  "confirmations": 2
}
```

#### List Blocks
```
GET /chain/blocks?offset=0&limit=10
```
```json
{
  "blocks": [
    {
      "height": 5,
      "hash": "abc123...",
      "timestamp": "2024-01-10T10:05:00.000Z",
      "txCount": 3,
      "proposerId": "node-local"
    },
    ...
  ],
  "total": 6,
  "offset": 0,
  "limit": 10,
  "hasMore": false
}
```

#### Get Block by Height
```
GET /chain/blocks/:height
```
```json
{
  "header": {
    "chainId": "arka-dev-chain",
    "height": 3,
    "previousHash": "def456...",
    "timestamp": "2024-01-10T10:03:00.000Z",
    "merkleRoot": "ghi789...",
    "proposerId": "node-local",
    "txCount": 5
  },
  "transactions": [...],
  "hash": "abc123..."
}
```

#### Get Block by Hash
```
GET /chain/blocks/hash/:hash
```
Same response as get by height.

#### Get Chain State
```
GET /chain/state
```
```json
{
  "chainId": "arka-dev-chain",
  "head": {...},
  "height": 5,
  "totalTransactions": 25,
  "genesisTimestamp": "2024-01-10T10:00:00.000Z"
}
```

#### Get Chain Statistics
```
GET /chain/stats
```
```json
{
  "chainId": "arka-dev-chain",
  "totalBlocks": 6,
  "totalTransactions": 25,
  "txLast24h": 25,
  "avgBlockTimeMs": 60000,
  "avgTxPerBlock": 4.17,
  "genesisTime": "2024-01-10T10:00:00.000Z",
  "lastBlockTime": "2024-01-10T10:05:00.000Z"
}
```

#### Mine Block (Dev Only)
```
POST /chain/mine
```
```json
{
  "header": {
    "height": 6,
    "txCount": 3,
    ...
  },
  "transactions": [...],
  "hash": "newblockhash..."
}
```

---

## arka-chain-explorer

Web-based block explorer for browsing the blockchain.

**URL:** `http://localhost:4041`

**Features:**
- Real-time chain statistics dashboard
- Block listing with pagination
- Block detail view with transactions
- Transaction detail view
- Connection status indicator
- Auto-refresh every 10 seconds

**Environment Variables:**
| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Explorer server port | `4041` |
| `CHAIN_API_URL` | Blockchain node URL | `http://localhost:4040` |

---

## Local Development

### Quick Start

```bash
# Terminal 1 - Start blockchain node
pnpm chain:start

# Terminal 2 - Start explorer
pnpm chain:explorer

# Terminal 3 - Run demo
pnpm chain:demo
```

### Demo Script Output

```
=======================================================
  ARKA Blockchain Local Demo
=======================================================

  [OK] Connected to chain: arka-dev-chain
  [..] Current height: -1 blocks, 0 txs

  Submitting ARKA compliance events...

    KYC_VERIFICATION       -> a515a9b6c664e7b5...
    TRANSACTION_SCREENED   -> 63d0bf351ac61953...
    SANCTIONS_CHECK        -> 5ba56e56e3691573...
    AUDIT_LOG              -> a091058f6dbebc7b...
    REGULATORY_REPORT      -> 2668f6be94895f5e...

  [OK] Submitted 5 transactions to mempool
  [OK] Mined block #0 with 5 transactions

=======================================================
  ARKA Blockchain Demo Complete!
=======================================================
```

---

## Configuration

### Node Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | HTTP API port | `4040` |
| `ARKA_CHAIN_ID` | Chain identifier | `arka-dev-chain` |
| `ARKA_NODE_ID` | Node identifier | `node-local` |
| `ARKA_MAX_TX_PER_BLOCK` | Max transactions per block | `1000` |
| `CORS_ENABLED` | Enable CORS | `true` |
| `CORS_ORIGINS` | Allowed origins | `*` |

### Storage Configuration

```typescript
// Memory storage (default)
const chain = await createChain({
  chainId: 'my-chain',
  storage: 'memory',
});

// File storage
const chain = await createChain({
  chainId: 'my-chain',
  storage: 'file',
  storagePath: './data/chain',
});
```

---

## AWS Deployment

The `infra/arka-chain-cdk` directory contains AWS CDK infrastructure for production deployment.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                           AWS VPC                                │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │   Public Subnet     │    │   Public Subnet     │            │
│  │   (AZ-a)            │    │   (AZ-b)            │            │
│  │   ┌─────────────┐   │    │   ┌─────────────┐   │            │
│  │   │ Chain Node  │   │    │   │ Chain Node  │   │   Future   │
│  │   │ (Fargate)   │   │    │   │ (Fargate)   │   │            │
│  │   └─────────────┘   │    │   └─────────────┘   │            │
│  └─────────────────────┘    └─────────────────────┘            │
│                                                                  │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │  Isolated Subnet    │    │  Isolated Subnet    │            │
│  │  (Databases)        │    │  (Databases)        │   Future   │
│  └─────────────────────┘    └─────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### CDK Commands

```bash
cd infra/arka-chain-cdk

# Install dependencies
npm install

# Synthesize CloudFormation
npm run synth

# Deploy (when ready)
cdk deploy --all

# View diff
cdk diff
```

### Cost Optimization

The skeleton infrastructure is designed to minimize costs:

| Resource | Cost | Status |
|----------|------|--------|
| VPC | Free | Created |
| NAT Gateway | ~$32/month/AZ | **Disabled** |
| Fargate Service | Pay per use | **Disabled** |
| Load Balancer | ~$16/month | **Disabled** |
| CloudWatch Logs | Minimal | 1 week retention |

---

## Testing

### Run All Chain Tests

```bash
# All chain packages
pnpm test -- packages/arka-chain-core packages/arka-chain-node packages/arka-chain-client apps/arka-chain-node-api

# Individual packages
pnpm --filter @arka/chain-core test
pnpm --filter @arka/chain-node test
pnpm --filter @arka/chain-client test
pnpm --filter arka-chain-node-api test
```

### Test Coverage

| Package | Tests | Coverage |
|---------|-------|----------|
| @arka/chain-core | 19 | Block, chain, merkle, storage |
| @arka/chain-node | 14 | Mempool, node operations |
| @arka/chain-client | 11 | Local and HTTP clients |
| arka-chain-node-api | 15 | All API endpoints |
| **Total** | **59** | - |

---

## Security Considerations

### Transaction Integrity
- All transactions are SHA-256 hashed
- Blocks include Merkle root for transaction verification
- Blocks reference previous block hash for chain integrity

### Immutability
- Once a block is mined, it cannot be modified
- Changing any transaction would invalidate the Merkle root
- Changing any block would invalidate subsequent block hashes

### Audit Trail
- All compliance events are timestamped
- Event source and actor are recorded
- Full payload preserved for regulatory evidence

### Access Control
- Production deployments should use authentication
- CORS configuration limits API access
- Rate limiting recommended for public endpoints

---

## Integration with ARKA Cloud

### Recording Compliance Events

```typescript
import { PactCloudChainIntegration } from '@arka/chain-client';

// Initialize integration
const chainIntegration = new PactCloudChainIntegration({
  chainClient: httpClient,
});

// In your compliance workflow
async function handleKycVerification(result) {
  // Process KYC result...

  // Record on blockchain
  await chainIntegration.recordEvent({
    type: 'kyc.verified',
    entityId: result.customerId,
    data: {
      level: result.verificationLevel,
      score: result.riskScore,
      timestamp: new Date().toISOString(),
    },
  });
}

// In your transaction monitoring
async function handleTransactionScreening(tx, result) {
  await chainIntegration.recordEvent({
    type: 'transaction.screened',
    entityId: tx.id,
    data: {
      amount: tx.amount,
      riskLevel: result.riskLevel,
      flagged: result.flagged,
      rules: result.triggeredRules,
    },
  });
}
```

### Querying Compliance History

```typescript
// Get all events for an entity
const entityHistory = await chainClient.getTransactionsByEntity('customer-123');

// Get events in date range
const recentEvents = await chainClient.getTransactionsByDateRange(
  new Date('2024-01-01'),
  new Date('2024-01-31')
);

// Verify transaction exists and is confirmed
const lookup = await chainClient.lookupTransaction(txHash);
if (lookup.confirmations >= 6) {
  console.log('Transaction is confirmed');
}
```

---

## Roadmap

### v0.2.0
- [ ] Peer-to-peer networking
- [ ] Multi-node consensus
- [ ] Persistent file storage

### v0.3.0
- [ ] Transaction indexing (by entity, date)
- [ ] Event subscriptions (WebSocket)
- [ ] Block explorer enhancements

### v1.0.0
- [ ] Production consensus mechanism
- [ ] Horizontal scaling
- [ ] Enterprise SLA support
- [ ] Regulatory certification
