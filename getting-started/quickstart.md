# Quick Start Guide

Get ARKA Engine running and evaluate your first compliance rule in under 15 minutes.

## Prerequisites

- Docker 20.10+ and Docker Compose 2.0+
- 8GB RAM minimum (16GB recommended)
- 20GB available disk space

## Step 1: Clone and Start

```bash
# Clone the repository
git clone https://github.com/arka-engine/pact.git
cd pact

# Start ARKA Engine with Docker Compose
docker-compose up -d

# Verify services are running
docker-compose ps
```

Expected output:
```
NAME                    STATUS    PORTS
arka-core               running   0.0.0.0:8080->8080/tcp
arka-rtvm               running   0.0.0.0:8081->8081/tcp
arka-postgres           running   0.0.0.0:5432->5432/tcp
arka-redis              running   0.0.0.0:6379->6379/tcp
```

## Step 2: Verify Installation

```bash
# Check system health
curl http://localhost:8080/health

# Expected response:
# {"status":"healthy","version":"1.0.0","components":{"rtvm":"healthy","database":"healthy","cache":"healthy"}}
```

## Step 3: Create Your First Rule

Create a simple transaction amount threshold rule:

```bash
# Create a rule using the CLI
curl -X POST http://localhost:8080/api/v1/rules \
  -H "Content-Type: application/json" \
  -d '{
    "name": "transaction-amount-limit",
    "description": "Flag transactions exceeding $10,000",
    "jurisdiction": "US",
    "domain": "AML",
    "rule_dsl": {
      "conditions": [
        {
          "field": "transaction.amount",
          "operator": "greater_than",
          "value": 10000
        }
      ],
      "actions": [
        {
          "type": "flag",
          "severity": "high",
          "message": "Transaction exceeds reporting threshold"
        }
      ]
    }
  }'
```

Response:
```json
{
  "rule_id": "rule_abc123",
  "version": "1.0.0",
  "status": "active",
  "created_at": "2024-01-15T10:30:00Z"
}
```

## Step 4: Evaluate Compliance

Test your rule against sample data:

```bash
# Evaluate a transaction
curl -X POST http://localhost:8080/api/v1/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "context": {
      "jurisdiction": "US",
      "domain": "AML"
    },
    "input": {
      "transaction": {
        "id": "txn_001",
        "amount": 15000,
        "currency": "USD",
        "sender": "customer_123",
        "recipient": "merchant_456"
      }
    }
  }'
```

Response:
```json
{
  "evaluation_id": "eval_xyz789",
  "timestamp": "2024-01-15T10:35:00Z",
  "result": {
    "compliant": false,
    "flags": [
      {
        "rule_id": "rule_abc123",
        "severity": "high",
        "message": "Transaction exceeds reporting threshold",
        "field": "transaction.amount",
        "value": 15000
      }
    ]
  },
  "trust_proof": {
    "proof_id": "proof_def456",
    "rule_version": "1.0.0",
    "input_hash": "sha256:abc...",
    "output_hash": "sha256:def..."
  }
}
```

## Step 5: View the Dashboard

Open your browser and navigate to:

```
http://localhost:8080/dashboard
```

The dashboard provides:
- Real-time compliance evaluation metrics
- Rule management interface
- Audit trail viewer
- System health monitoring

## Next Steps

Now that you have ARKA Engine running:

1. **Learn the Concepts** - Read [Concepts & Terminology](./concepts.md) to understand ARKA's architecture
2. **Write More Rules** - Follow the [Rule Authoring Guide](../guides/rule-authoring.md) for complex rules
3. **Integrate Your Systems** - See [Integration Patterns](../guides/integration-patterns.md)
4. **Deploy to Production** - Follow the [Deployment Guide](../deployment/deployment-guide.md)

## Troubleshooting

### Services won't start

```bash
# Check logs
docker-compose logs arka-core

# Ensure ports are available
lsof -i :8080
```

### Database connection errors

```bash
# Verify PostgreSQL is running
docker-compose exec arka-postgres pg_isready

# Check database logs
docker-compose logs arka-postgres
```

### Memory issues

```bash
# Increase Docker memory limit to 8GB minimum
# Docker Desktop: Settings -> Resources -> Memory
```

## Clean Up

```bash
# Stop all services
docker-compose down

# Remove volumes (deletes all data)
docker-compose down -v
```
