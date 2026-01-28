# Integration Patterns

This guide covers common patterns for integrating the ARKA Engine into your applications and infrastructure.

## Overview

The ARKA Engine supports multiple integration patterns to fit different architectural needs:

```
┌─────────────────────────────────────────────────────────────┐
│                   Integration Patterns                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Synchronous │  │    Event    │  │      Embedded       │  │
│  │   Request   │  │   Driven    │  │       Agent         │  │
│  │  Response   │  │             │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Batch     │  │    Sidecar  │  │      Gateway        │  │
│  │ Processing  │  │    Proxy    │  │   Integration       │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Synchronous Request-Response

### REST API Integration

The most common pattern for real-time compliance checks:

```python
import requests
from typing import Optional

class ComplianceClient:
    """Synchronous compliance client using REST API."""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        })

    def evaluate(
        self,
        transaction: dict,
        jurisdictions: list[str],
        domains: list[str],
        timeout: float = 30.0
    ) -> dict:
        """
        Evaluate a transaction synchronously.

        Args:
            transaction: Transaction data
            jurisdictions: Target jurisdictions
            domains: Compliance domains
            timeout: Request timeout in seconds

        Returns:
            Evaluation result
        """
        response = self.session.post(
            f"{self.base_url}/api/v1/evaluate",
            json={
                "transaction": transaction,
                "jurisdictions": jurisdictions,
                "domains": domains,
                "options": {
                    "generate_proof": True
                }
            },
            timeout=timeout
        )
        response.raise_for_status()
        return response.json()

# Usage in transaction processing
class PaymentService:
    def __init__(self, compliance_client: ComplianceClient):
        self.compliance = compliance_client

    def process_payment(self, payment: dict) -> dict:
        # Check compliance before processing
        result = self.compliance.evaluate(
            transaction={
                "id": payment["id"],
                "type": "payment",
                "amount": payment["amount"],
                "currency": payment["currency"],
                "source": payment["from_account"],
                "destination": payment["to_account"]
            },
            jurisdictions=["US"],
            domains=["AML", "SANCTIONS"]
        )

        if not result["compliant"]:
            return {
                "status": "rejected",
                "reason": "compliance_check_failed",
                "violations": result["violations"]
            }

        # Process payment
        return self.execute_payment(payment)
```

### gRPC Integration

For low-latency, high-throughput scenarios:

```python
import grpc
from pact.proto import evaluation_pb2, evaluation_pb2_grpc

class GRPCComplianceClient:
    """High-performance gRPC compliance client."""

    def __init__(self, endpoint: str, credentials: grpc.ChannelCredentials):
        self.channel = grpc.secure_channel(endpoint, credentials)
        self.stub = evaluation_pb2_grpc.EvaluationServiceStub(self.channel)

    def evaluate(self, transaction: dict, jurisdictions: list, domains: list) -> dict:
        request = evaluation_pb2.EvaluateRequest(
            transaction=dict_to_struct(transaction),
            jurisdictions=[evaluation_pb2.Jurisdiction(code=j) for j in jurisdictions],
            domains=[getattr(evaluation_pb2, f"DOMAIN_{d}") for d in domains]
        )

        response = self.stub.Evaluate(request, timeout=10)
        return struct_to_dict(response)

    def evaluate_stream(self, transactions):
        """Stream transactions for evaluation."""
        def request_generator():
            for txn in transactions:
                yield evaluation_pb2.EvaluateRequest(
                    transaction=dict_to_struct(txn),
                    jurisdictions=[evaluation_pb2.Jurisdiction(code="US")],
                    domains=[evaluation_pb2.DOMAIN_AML]
                )

        for response in self.stub.EvaluateStream(request_generator()):
            yield struct_to_dict(response)
```

## Event-Driven Integration

### Kafka Integration

Process transactions asynchronously with Apache Kafka:

```python
from kafka import KafkaConsumer, KafkaProducer
import json

class KafkaComplianceProcessor:
    """Kafka-based asynchronous compliance processor."""

    def __init__(
        self,
        bootstrap_servers: list[str],
        input_topic: str,
        output_topic: str,
        compliance_client: ComplianceClient
    ):
        self.consumer = KafkaConsumer(
            input_topic,
            bootstrap_servers=bootstrap_servers,
            group_id="compliance-processor",
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            enable_auto_commit=False
        )

        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda m: json.dumps(m).encode('utf-8')
        )

        self.output_topic = output_topic
        self.compliance = compliance_client

    def process(self):
        """Process transactions from Kafka."""
        for message in self.consumer:
            transaction = message.value

            try:
                # Evaluate compliance
                result = self.compliance.evaluate(
                    transaction=transaction,
                    jurisdictions=transaction.get("jurisdictions", ["US"]),
                    domains=transaction.get("domains", ["AML"])
                )

                # Publish result
                self.producer.send(
                    self.output_topic,
                    key=transaction["id"].encode(),
                    value={
                        "transaction_id": transaction["id"],
                        "compliant": result["compliant"],
                        "violations": result.get("violations", []),
                        "proof_id": result.get("proof", {}).get("proof_id")
                    }
                )

                # Commit offset on success
                self.consumer.commit()

            except Exception as e:
                # Send to dead letter queue
                self.producer.send(
                    f"{self.output_topic}.dlq",
                    value={
                        "transaction": transaction,
                        "error": str(e)
                    }
                )

# Usage
processor = KafkaComplianceProcessor(
    bootstrap_servers=["kafka:9092"],
    input_topic="transactions.pending",
    output_topic="transactions.compliance-results",
    compliance_client=compliance_client
)
processor.process()
```

### AWS EventBridge Integration

```python
import boto3
import json
from datetime import datetime

class EventBridgeComplianceHandler:
    """AWS EventBridge compliance event handler."""

    def __init__(self, compliance_client: ComplianceClient):
        self.events = boto3.client('events')
        self.compliance = compliance_client

    def handle_transaction_event(self, event: dict, context: dict) -> dict:
        """Lambda handler for transaction events."""
        transaction = event['detail']['transaction']

        # Evaluate compliance
        result = self.compliance.evaluate(
            transaction=transaction,
            jurisdictions=event['detail'].get('jurisdictions', ['US']),
            domains=event['detail'].get('domains', ['AML'])
        )

        # Publish compliance result event
        self.events.put_events(
            Entries=[{
                'Source': 'arka.compliance',
                'DetailType': 'ComplianceEvaluationCompleted',
                'Detail': json.dumps({
                    'transaction_id': transaction['id'],
                    'compliant': result['compliant'],
                    'violations': result.get('violations', []),
                    'proof_id': result.get('proof', {}).get('proof_id'),
                    'evaluated_at': datetime.utcnow().isoformat()
                }),
                'EventBusName': 'compliance-events'
            }]
        )

        return result

# AWS Lambda handler
def lambda_handler(event, context):
    handler = EventBridgeComplianceHandler(compliance_client)
    return handler.handle_transaction_event(event, context)
```

### RabbitMQ Integration

```python
import pika
import json

class RabbitMQComplianceProcessor:
    """RabbitMQ-based compliance processor."""

    def __init__(
        self,
        connection_url: str,
        input_queue: str,
        output_exchange: str,
        compliance_client: ComplianceClient
    ):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(connection_url)
        )
        self.channel = self.connection.channel()

        # Declare queues
        self.channel.queue_declare(queue=input_queue, durable=True)
        self.channel.exchange_declare(
            exchange=output_exchange,
            exchange_type='topic',
            durable=True
        )

        self.input_queue = input_queue
        self.output_exchange = output_exchange
        self.compliance = compliance_client

    def process_message(self, ch, method, properties, body):
        """Process a single message."""
        transaction = json.loads(body)

        result = self.compliance.evaluate(
            transaction=transaction,
            jurisdictions=["US"],
            domains=["AML"]
        )

        # Route based on result
        routing_key = "compliant" if result["compliant"] else "non_compliant"

        self.channel.basic_publish(
            exchange=self.output_exchange,
            routing_key=f"compliance.{routing_key}",
            body=json.dumps({
                "transaction_id": transaction["id"],
                "result": result
            }),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent
                correlation_id=properties.correlation_id
            )
        )

        ch.basic_ack(delivery_tag=method.delivery_tag)

    def start(self):
        """Start consuming messages."""
        self.channel.basic_qos(prefetch_count=10)
        self.channel.basic_consume(
            queue=self.input_queue,
            on_message_callback=self.process_message
        )
        self.channel.start_consuming()
```

## Embedded Agent Pattern

Deploy compliance evaluation locally within your service:

```python
from pact.agent import EmbeddedAgent, AgentConfig

class EmbeddedComplianceService:
    """Embedded ARKA agent for local compliance evaluation."""

    def __init__(self):
        self.agent = EmbeddedAgent(
            config=AgentConfig(
                control_plane_url="https://control.arka.io",
                api_key=os.environ["ARKA_API_KEY"],
                sync_interval=300,  # 5 minutes
                cache_size=10000,
                offline_enabled=True
            )
        )

    async def initialize(self):
        """Initialize the embedded agent."""
        await self.agent.start()
        await self.agent.sync_rules()

    async def evaluate(self, transaction: dict) -> dict:
        """Evaluate locally using embedded agent."""
        return await self.agent.evaluate(
            transaction=transaction,
            jurisdictions=["US"],
            domains=["AML"]
        )

    async def shutdown(self):
        """Shutdown the agent."""
        await self.agent.stop()

# Usage in FastAPI application
from fastapi import FastAPI
from contextlib import asynccontextmanager

compliance_service = EmbeddedComplianceService()

@asynccontextmanager
async def lifespan(app: FastAPI):
    await compliance_service.initialize()
    yield
    await compliance_service.shutdown()

app = FastAPI(lifespan=lifespan)

@app.post("/transactions")
async def process_transaction(transaction: dict):
    result = await compliance_service.evaluate(transaction)

    if not result["compliant"]:
        raise HTTPException(status_code=400, detail=result["violations"])

    return {"status": "processed", "proof_id": result["proof_id"]}
```

## Sidecar Proxy Pattern

Deploy ARKA as a sidecar container for transparent compliance:

```yaml
# Kubernetes deployment with ARKA sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        # Main application
        - name: payment-service
          image: payment-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: COMPLIANCE_URL
              value: "http://localhost:9080"

        # ARKA sidecar
        - name: arka-sidecar
          image: pact/sidecar:latest
          ports:
            - containerPort: 9080
          env:
            - name: ARKA_CONTROL_PLANE
              value: "https://control.arka.io"
            - name: ARKA_API_KEY
              valueFrom:
                secretKeyRef:
                  name: arka-secrets
                  key: api-key
          volumeMounts:
            - name: arka-config
              mountPath: /config
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"

      volumes:
        - name: arka-config
          configMap:
            name: arka-sidecar-config
```

```yaml
# Sidecar configuration
# arka-sidecar-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: arka-sidecar-config
data:
  config.yaml: |
    server:
      port: 9080
      timeout: 30s

    agent:
      syncInterval: 5m
      cacheSize: 5000
      offlineEnabled: true

    interceptor:
      enabled: true
      rules:
        - path: /api/payments
          method: POST
          domains: [AML, SANCTIONS]
          jurisdictions: [US]
          action: block_on_violation

        - path: /api/transfers
          method: POST
          domains: [AML]
          action: log_only
```

```python
# Application code - no changes needed for compliance
import httpx

class PaymentClient:
    def __init__(self, base_url: str):
        self.base_url = base_url

    async def create_payment(self, payment: dict) -> dict:
        # Compliance check happens transparently via sidecar
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/api/payments",
                json=payment
            )
            return response.json()
```

## API Gateway Integration

### Kong Gateway Plugin

```lua
-- kong/plugi../arka-compliance/handler.lua
local ARKAHandler = {
    VERSION = "1.0.0",
    PRIORITY = 1000,
}

function ARKAHandler:access(conf)
    local body = kong.request.get_body()

    if not body then
        return
    end

    -- Call ARKA Engine
    local httpc = require("resty.http").new()
    local res, err = httpc:request_uri(conf.pact_endpoint .. "/api/v1/evaluate", {
        method = "POST",
        headers = {
            ["Content-Type"] = "application/json",
            ["Authorization"] = "Bearer " .. conf.api_key,
        },
        body = cjson.encode({
            transaction = body,
            jurisdictions = conf.jurisdictions,
            domains = conf.domains,
        }),
    })

    if not res then
        kong.log.err("ARKA request failed: ", err)
        if conf.fail_open then
            return -- Allow request to proceed
        end
        return kong.response.exit(503, { message = "Compliance check unavailable" })
    end

    local result = cjson.decode(res.body)

    if not result.compliant then
        -- Store proof for audit
        kong.ctx.shared.pact_proof_id = result.proof.proof_id
        kong.ctx.shared.pact_violations = result.violations

        if conf.action == "block" then
            return kong.response.exit(403, {
                message = "Compliance check failed",
                violations = result.violations,
                proof_id = result.proof.proof_id
            })
        end
    end

    -- Add compliance headers
    kong.service.request.set_header("X-Compliance-Status", result.compliant and "pass" or "fail")
    kong.service.request.set_header("X-Compliance-Proof-ID", result.proof.proof_id)
end

return ARKAHandler
```

```yaml
# Kong plugin configuration
plugins:
  - name: arka-compliance
    config:
      pact_endpoint: "https://pact.internal:8080"
      api_key: "${ARKA_API_KEY}"
      jurisdictions:
        - US
        - EU
      domains:
        - AML
        - SANCTIONS
      action: block
      fail_open: false
    service: payment-service
    routes:
      - /api/payments
      - /api/transfers
```

### AWS API Gateway Integration

```yaml
# AWS SAM template
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ComplianceAuthorizer:
    Type: AWS::Serverless::Function
    Properties:
      Handler: compliance_authorizer.handler
      Runtime: python3.11
      Timeout: 30
      Environment:
        Variables:
          ARKA_ENDPOINT: !Ref ARKAEndpoint
          ARKA_API_KEY: !Ref ARKAApiKey

  PaymentsApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: ComplianceAuthorizer
        Authorizers:
          ComplianceAuthorizer:
            FunctionArn: !GetAtt ComplianceAuthorizer.Arn
            FunctionPayloadType: REQUEST
            Identity:
              Headers:
                - Authorization
              ReauthorizeEvery: 0
```

```python
# compliance_authorizer.py
import json
import os
import httpx

ARKA_ENDPOINT = os.environ['ARKA_ENDPOINT']
ARKA_API_KEY = os.environ['ARKA_API_KEY']

async def handler(event, context):
    # Extract transaction from request
    body = json.loads(event.get('body', '{}'))

    # Evaluate compliance
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{ARKA_ENDPOINT}/api/v1/evaluate",
            headers={
                "Authorization": f"Bearer {ARKA_API_KEY}",
                "Content-Type": "application/json"
            },
            json={
                "transaction": body,
                "jurisdictions": ["US"],
                "domains": ["AML", "SANCTIONS"]
            }
        )
        result = response.json()

    # Build policy
    if result['compliant']:
        return generate_policy('Allow', event['methodArn'], result)
    else:
        return generate_policy('Deny', event['methodArn'], result)

def generate_policy(effect, resource, compliance_result):
    return {
        'principalId': 'compliance-check',
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': resource
            }]
        },
        'context': {
            'compliant': str(compliance_result['compliant']),
            'proofId': compliance_result.get('proof', {}).get('proof_id', '')
        }
    }
```

## Batch Processing Pattern

### Daily Batch Processing

```python
from datetime import datetime, timedelta
import pandas as pd

class BatchComplianceProcessor:
    """Batch compliance processing for historical analysis."""

    def __init__(
        self,
        compliance_client: ComplianceClient,
        batch_size: int = 1000
    ):
        self.compliance = compliance_client
        self.batch_size = batch_size

    async def process_daily_batch(
        self,
        date: datetime,
        data_source: callable
    ) -> dict:
        """
        Process all transactions for a given date.

        Args:
            date: Date to process
            data_source: Function to fetch transactions

        Returns:
            Processing summary
        """
        start_time = datetime.utcnow()
        total_processed = 0
        compliant_count = 0
        non_compliant_count = 0
        errors = []

        # Fetch transactions in batches
        offset = 0
        while True:
            transactions = await data_source(
                date=date,
                offset=offset,
                limit=self.batch_size
            )

            if not transactions:
                break

            # Process batch
            batch_results = await self.compliance.evaluate_batch(
                transactions=transactions,
                jurisdictions=["US"],
                domains=["AML"]
            )

            for result in batch_results['results']:
                total_processed += 1
                if result['compliant']:
                    compliant_count += 1
                else:
                    non_compliant_count += 1

                if result.get('error'):
                    errors.append({
                        'transaction_id': result['transaction_id'],
                        'error': result['error']
                    })

            offset += self.batch_size

        return {
            'date': date.isoformat(),
            'total_processed': total_processed,
            'compliant': compliant_count,
            'non_compliant': non_compliant_count,
            'error_count': len(errors),
            'errors': errors[:100],  # Limit errors in response
            'processing_time_seconds': (datetime.utcnow() - start_time).total_seconds()
        }

    async def generate_compliance_report(
        self,
        start_date: datetime,
        end_date: datetime
    ) -> pd.DataFrame:
        """Generate compliance report for date range."""
        results = []

        current_date = start_date
        while current_date <= end_date:
            day_result = await self.process_daily_batch(
                date=current_date,
                data_source=self.fetch_transactions
            )
            results.append(day_result)
            current_date += timedelta(days=1)

        return pd.DataFrame(results)
```

### Spark Integration for Large-Scale Processing

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import udf, struct, col
from pyspark.sql.types import StructType, StructField, StringType, BooleanType, ArrayType

# Initialize Spark session
spark = SparkSession.builder \
    .appName("PACTComplianceProcessor") \
    .config("spark.jars.packages", "io.pact:arka-spark:1.0.0") \
    .getOrCreate()

# Define UDF for compliance evaluation
@udf(returnType=StructType([
    StructField("compliant", BooleanType()),
    StructField("proof_id", StringType()),
    StructField("violations", ArrayType(StringType()))
]))
def evaluate_compliance(transaction):
    """UDF to evaluate compliance for each transaction."""
    client = get_compliance_client()  # Connection pooled client

    result = client.evaluate(
        transaction=transaction.asDict(),
        jurisdictions=["US"],
        domains=["AML"]
    )

    return {
        "compliant": result["compliant"],
        "proof_id": result.get("proof", {}).get("proof_id"),
        "violations": [v["description"] for v in result.get("violations", [])]
    }

# Process large dataset
df = spark.read.parquet("s3://data-lake/transactions/2024/")

# Add compliance results
df_with_compliance = df.withColumn(
    "compliance_result",
    evaluate_compliance(struct(df.columns))
)

# Filter non-compliant transactions
non_compliant = df_with_compliance.filter(
    col("compliance_result.compliant") == False
)

# Write results
non_compliant.write.parquet("s3://data-lake/compliance-results/2024/")
```

## Webhook Integration

### Receiving Compliance Events

```python
from fastapi import FastAPI, Request, HTTPException
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.hmac import HMAC
import base64

app = FastAPI()

WEBHOOK_SECRET = os.environ["ARKA_WEBHOOK_SECRET"]

def verify_signature(payload: bytes, signature: str) -> bool:
    """Verify webhook signature."""
    h = HMAC(WEBHOOK_SECRET.encode(), hashes.SHA256())
    h.update(payload)
    expected = base64.b64encode(h.finalize()).decode()
    return signature == expected

@app.post("/webhooks/pact")
async def handle_pact_webhook(request: Request):
    # Verify signature
    signature = request.headers.get("X-PACT-Signature")
    payload = await request.body()

    if not verify_signature(payload, signature):
        raise HTTPException(status_code=401, detail="Invalid signature")

    event = await request.json()

    # Handle different event types
    handlers = {
        "evaluation.completed": handle_evaluation_completed,
        "rule.updated": handle_rule_updated,
        "proposal.approved": handle_proposal_approved,
        "anchor.confirmed": handle_anchor_confirmed,
    }

    handler = handlers.get(event["type"])
    if handler:
        await handler(event["data"])

    return {"status": "received"}

async def handle_evaluation_completed(data: dict):
    """Handle evaluation completed event."""
    if not data["compliant"]:
        # Alert compliance team
        await send_alert(
            severity="high",
            message=f"Non-compliant transaction: {data['transaction_id']}",
            details=data["violations"]
        )

async def handle_rule_updated(data: dict):
    """Handle rule update event."""
    # Invalidate caches
    await cache.invalidate(f"rule:{data['rule_id']}")

    # Notify downstream systems
    await notify_rule_change(data["rule_id"], data["version"])

async def handle_proposal_approved(data: dict):
    """Handle proposal approval event."""
    # Log for audit
    await audit_log.record(
        event_type="proposal_approved",
        details=data
    )

async def handle_anchor_confirmed(data: dict):
    """Handle blockchain anchor confirmation."""
    # Update proof status
    await proof_store.update_status(
        proof_id=data["proof_id"],
        status="anchored",
        anchor_details={
            "chain": data["chain"],
            "block": data["block_number"],
            "tx_hash": data["transaction_hash"]
        }
    )
```

## Database Integration Patterns

### Transaction Interception

```python
from sqlalchemy import event
from sqlalchemy.orm import Session

class TransactionInterceptor:
    """Intercept database transactions for compliance."""

    def __init__(self, compliance_client: ComplianceClient):
        self.compliance = compliance_client

    def register(self, engine):
        """Register event listeners on SQLAlchemy engine."""
        event.listen(engine, "before_cursor_execute", self.before_execute)

    def before_execute(self, conn, cursor, statement, parameters, context, executemany):
        """Check compliance before executing sensitive operations."""
        if self._is_financial_operation(statement):
            transaction_data = self._extract_transaction_data(statement, parameters)

            result = self.compliance.evaluate(
                transaction=transaction_data,
                jurisdictions=["US"],
                domains=["AML"]
            )

            if not result["compliant"]:
                raise ComplianceViolationError(
                    violations=result["violations"],
                    proof_id=result["proof"]["proof_id"]
                )

    def _is_financial_operation(self, statement: str) -> bool:
        """Determine if statement involves financial data."""
        financial_tables = ['payments', 'transfers', 'transactions']
        return any(table in statement.lower() for table in financial_tables)
```

### Change Data Capture

```python
from debezium import Debezium

class CDCComplianceProcessor:
    """Process compliance via Change Data Capture."""

    def __init__(self):
        self.debezium = Debezium(
            connector_config={
                "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
                "database.hostname": "db.company.com",
                "database.port": "5432",
                "database.user": "debezium",
                "database.password": "${DB_PASSWORD}",
                "database.dbname": "transactions",
                "table.include.list": "public.payments,public.transfers"
            }
        )
        self.compliance = ComplianceClient(...)

    async def process_changes(self):
        """Process database changes for compliance."""
        async for change in self.debezium.stream():
            if change.operation in ['c', 'u']:  # Create or Update
                await self.evaluate_change(change)

    async def evaluate_change(self, change):
        """Evaluate compliance for database change."""
        result = await self.compliance.evaluate(
            transaction={
                "id": change.after.get("id"),
                "type": change.table,
                "amount": change.after.get("amount"),
                "data": change.after
            },
            jurisdictions=["US"],
            domains=["AML"]
        )

        if not result["compliant"]:
            # Flag record in database
            await self.flag_record(
                table=change.table,
                record_id=change.after["id"],
                violations=result["violations"]
            )
```

## Monitoring Integration

### Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

# Define metrics
evaluation_requests = Counter(
    'pact_evaluation_requests_total',
    'Total compliance evaluation requests',
    ['jurisdiction', 'domain', 'result']
)

evaluation_latency = Histogram(
    'pact_evaluation_latency_seconds',
    'Compliance evaluation latency',
    ['jurisdiction', 'domain'],
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

active_evaluations = Gauge(
    'pact_active_evaluations',
    'Currently active evaluations'
)

class InstrumentedComplianceClient:
    """Compliance client with Prometheus metrics."""

    def __init__(self, base_client: ComplianceClient):
        self.client = base_client

    def evaluate(self, transaction, jurisdictions, domains):
        active_evaluations.inc()
        start_time = time.time()

        try:
            result = self.client.evaluate(transaction, jurisdictions, domains)

            # Record metrics
            for j in jurisdictions:
                for d in domains:
                    evaluation_requests.labels(
                        jurisdiction=j,
                        domain=d,
                        result='compliant' if result['compliant'] else 'non_compliant'
                    ).inc()

                    evaluation_latency.labels(
                        jurisdiction=j,
                        domain=d
                    ).observe(time.time() - start_time)

            return result

        except Exception as e:
            evaluation_requests.labels(
                jurisdiction=jurisdictions[0],
                domain=domains[0],
                result='error'
            ).inc()
            raise

        finally:
            active_evaluations.dec()
```

### OpenTelemetry Tracing

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer("arka-compliance")

class TracedComplianceClient:
    """Compliance client with OpenTelemetry tracing."""

    def __init__(self, base_client: ComplianceClient):
        self.client = base_client

    def evaluate(self, transaction, jurisdictions, domains):
        with tracer.start_as_current_span(
            "pact.evaluate",
            kind=SpanKind.CLIENT,
            attributes={
                "pact.transaction_id": transaction.get("id"),
                "pact.jurisdictions": ",".join(jurisdictions),
                "pact.domains": ",".join(domains),
            }
        ) as span:
            result = self.client.evaluate(transaction, jurisdictions, domains)

            span.set_attribute("arka.compliant", result["compliant"])
            span.set_attribute("pact.proof_id", result.get("proof", {}).get("proof_id"))
            span.set_attribute("pact.violations_count", len(result.get("violations", [])))

            if not result["compliant"]:
                span.set_status(trace.Status(trace.StatusCode.OK, "Non-compliant"))

            return result
```

## Best Practices

### Resilience

1. **Circuit breakers**: Implement circuit breakers for external calls
2. **Timeouts**: Set appropriate timeouts for all operations
3. **Retries**: Use exponential backoff for transient failures
4. **Fallbacks**: Define fallback behavior for degraded mode

### Performance

1. **Connection pooling**: Reuse connections
2. **Caching**: Cache rule metadata and frequent lookups
3. **Batching**: Batch multiple evaluations when possible
4. **Async**: Use async operations for I/O-bound work

### Security

1. **mTLS**: Use mutual TLS for all communications
2. **Secrets**: Use secret management for API keys
3. **Audit**: Log all compliance decisions
4. **Encryption**: Encrypt sensitive data in transit and at rest

### Observability

1. **Metrics**: Track request rates, latencies, and error rates
2. **Tracing**: Implement distributed tracing
3. **Logging**: Structured logging with correlation IDs
4. **Alerting**: Set up alerts for anomalies

## Related Documentation

- [SDK Reference](../api-reference/sdk-reference.md)
- [REST API Reference](../api-reference/rest-api.md)
- [Deployment Guide](../deployment/deployment-guide.md)
- [Security Model](../security/security-model.md)
