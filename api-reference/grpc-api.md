# gRPC API Reference

The ARKA Engine provides a high-performance gRPC API for low-latency compliance evaluation and streaming operations.

## Overview

The gRPC API is designed for high-throughput scenarios where latency and performance are critical. It supports both unary and streaming RPCs.

```
┌─────────────────────────────────────────────────────────────┐
│                     gRPC Services                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Evaluation     │  │  Rules          │                   │
│  │  Service        │  │  Service        │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Governance     │  │  Agent          │                   │
│  │  Service        │  │  Service        │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │  Anchoring      │  │  Health         │                   │
│  │  Service        │  │  Service        │                   │
│  └─────────────────┘  └─────────────────┘                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Connection

### Server Configuration

```yaml
# grpc-server-config.yaml
grpc:
  server:
    port: 50051
    maxConnectionAge: 30m
    maxConnectionAgeGrace: 5m
    keepaliveTime: 10s
    keepaliveTimeout: 3s
    maxRecvMsgSize: 16Mi
    maxSendMsgSize: 16Mi

  tls:
    enabled: true
    certFile: /certs/server.crt
    keyFile: /certs/server.key
    caFile: /certs/ca.crt
    clientAuth: require

  interceptors:
    - authentication
    - rateLimit
    - logging
    - metrics
```

### Client Connection

```python
import grpc
from pact.proto import evaluation_pb2_grpc

# Secure connection with mTLS
credentials = grpc.ssl_channel_credentials(
    root_certificates=open('/certs/ca.crt', 'rb').read(),
    private_key=open('/certs/client.key', 'rb').read(),
    certificate_chain=open('/certs/client.crt', 'rb').read()
)

channel = grpc.secure_channel(
    'arka.company.com:50051',
    credentials,
    options=[
        ('grpc.max_receive_message_length', 16 * 1024 * 1024),
        ('grpc.max_send_message_length', 16 * 1024 * 1024),
        ('grpc.keepalive_time_ms', 10000),
        ('grpc.keepalive_timeout_ms', 3000),
    ]
)

# Create stub
stub = evaluation_pb2_grpc.EvaluationServiceStub(channel)
```

## Protocol Buffers

### Common Types

```protobuf
// common.proto
syntax = "proto3";

package arka.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/struct.proto";

// Jurisdiction identifier
message Jurisdiction {
  string code = 1;           // e.g., "US", "EU", "UK"
  string subdivision = 2;    // e.g., "NY", "CA" (optional)
}

// Domain identifier
enum Domain {
  DOMAIN_UNSPECIFIED = 0;
  DOMAIN_AML = 1;
  DOMAIN_KYC = 2;
  DOMAIN_SANCTIONS = 3;
  DOMAIN_PEP = 4;
  DOMAIN_GDPR = 5;
  DOMAIN_HIPAA = 6;
}

// Rule reference
message RuleRef {
  string rule_id = 1;
  string version = 2;        // Specific version or "latest"
}

// Compliance result
enum ComplianceStatus {
  COMPLIANCE_STATUS_UNSPECIFIED = 0;
  COMPLIANCE_STATUS_COMPLIANT = 1;
  COMPLIANCE_STATUS_NON_COMPLIANT = 2;
  COMPLIANCE_STATUS_REQUIRES_REVIEW = 3;
  COMPLIANCE_STATUS_ERROR = 4;
}

// Violation details
message Violation {
  string rule_id = 1;
  string rule_version = 2;
  string description = 3;
  string severity = 4;        // "low", "medium", "high", "critical"
  google.protobuf.Struct details = 5;
}

// Trust proof reference
message TrustProof {
  string proof_id = 1;
  string proof_hash = 2;
  string agent_did = 3;
  string signature = 4;
  google.protobuf.Timestamp timestamp = 5;
}

// Pagination
message PageRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message PageResponse {
  string next_page_token = 1;
  int32 total_count = 2;
}
```

## Evaluation Service

### Service Definition

```protobuf
// evaluation.proto
syntax = "proto3";

package arka.v1;

import "common.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

service EvaluationService {
  // Evaluate a single transaction
  rpc Evaluate(EvaluateRequest) returns (EvaluateResponse);

  // Evaluate multiple transactions in batch
  rpc EvaluateBatch(EvaluateBatchRequest) returns (EvaluateBatchResponse);

  // Stream transactions for real-time evaluation
  rpc EvaluateStream(stream EvaluateRequest) returns (stream EvaluateResponse);

  // Bidirectional streaming for high-throughput scenarios
  rpc EvaluateBidirectional(stream EvaluateRequest) returns (stream EvaluateResponse);

  // Evaluate against historical rules (time-travel)
  rpc EvaluateHistorical(EvaluateHistoricalRequest) returns (EvaluateResponse);

  // Get evaluation status for async evaluation
  rpc GetEvaluationStatus(GetEvaluationStatusRequest) returns (GetEvaluationStatusResponse);
}

// Evaluation request
message EvaluateRequest {
  string request_id = 1;                    // Idempotency key
  google.protobuf.Struct transaction = 2;   // Transaction data
  repeated Jurisdiction jurisdictions = 3;   // Target jurisdictions
  repeated Domain domains = 4;               // Compliance domains
  repeated RuleRef rules = 5;                // Specific rules (optional)
  EvaluationOptions options = 6;
}

message EvaluationOptions {
  bool generate_proof = 1;           // Generate trust proof
  bool include_trace = 2;            // Include execution trace
  bool fail_fast = 3;                // Stop on first violation
  int32 timeout_ms = 4;              // Evaluation timeout
  string correlation_id = 5;         // For tracing
}

// Evaluation response
message EvaluateResponse {
  string evaluation_id = 1;
  ComplianceStatus status = 2;
  repeated Violation violations = 3;
  repeated RuleResult rule_results = 4;
  TrustProof proof = 5;
  ExecutionMetrics metrics = 6;
  google.protobuf.Timestamp timestamp = 7;
}

message RuleResult {
  string rule_id = 1;
  string rule_version = 2;
  ComplianceStatus status = 3;
  google.protobuf.Struct output = 4;
  int64 execution_time_us = 5;
}

message ExecutionMetrics {
  int32 rules_evaluated = 1;
  int64 total_time_us = 2;
  int64 rtvm_time_us = 3;
  int64 proof_time_us = 4;
  int32 cache_hits = 5;
  int32 cache_misses = 6;
}

// Batch evaluation
message EvaluateBatchRequest {
  repeated EvaluateRequest requests = 1;
  BatchOptions options = 2;
}

message BatchOptions {
  int32 max_parallelism = 1;
  bool stop_on_error = 2;
  int32 timeout_ms = 3;
}

message EvaluateBatchResponse {
  repeated EvaluateResponse responses = 1;
  BatchMetrics metrics = 2;
}

message BatchMetrics {
  int32 total = 1;
  int32 succeeded = 2;
  int32 failed = 3;
  int64 total_time_ms = 4;
  int64 avg_time_us = 5;
}

// Historical evaluation
message EvaluateHistoricalRequest {
  EvaluateRequest request = 1;
  google.protobuf.Timestamp as_of_time = 2;
}

// Async evaluation status
message GetEvaluationStatusRequest {
  string evaluation_id = 1;
}

message GetEvaluationStatusResponse {
  string evaluation_id = 1;
  EvaluationState state = 2;
  EvaluateResponse result = 3;  // Populated when complete
  string error = 4;              // Populated on failure
}

enum EvaluationState {
  EVALUATION_STATE_UNSPECIFIED = 0;
  EVALUATION_STATE_PENDING = 1;
  EVALUATION_STATE_RUNNING = 2;
  EVALUATION_STATE_COMPLETED = 3;
  EVALUATION_STATE_FAILED = 4;
}
```

### Client Usage

```python
from pact.proto import evaluation_pb2, evaluation_pb2_grpc
from google.protobuf.struct_pb2 import Struct

# Create request
transaction = Struct()
transaction.update({
    "id": "txn-123",
    "type": "wire_transfer",
    "amount": 15000,
    "currency": "USD",
    "source": {"account": "acct-001", "country": "US"},
    "destination": {"account": "acct-002", "country": "UK"}
})

request = evaluation_pb2.EvaluateRequest(
    request_id="req-123",
    transaction=transaction,
    jurisdictions=[
        evaluation_pb2.Jurisdiction(code="US"),
        evaluation_pb2.Jurisdiction(code="UK")
    ],
    domains=[
        evaluation_pb2.DOMAIN_AML,
        evaluation_pb2.DOMAIN_SANCTIONS
    ],
    options=evaluation_pb2.EvaluationOptions(
        generate_proof=True,
        timeout_ms=5000
    )
)

# Unary call
response = stub.Evaluate(request)
print(f"Status: {response.status}")
print(f"Violations: {len(response.violations)}")
print(f"Proof ID: {response.proof.proof_id}")
```

### Streaming Evaluation

```python
import asyncio
from typing import AsyncIterator

async def evaluate_stream(
    stub: evaluation_pb2_grpc.EvaluationServiceStub,
    transactions: AsyncIterator[dict]
) -> AsyncIterator[evaluation_pb2.EvaluateResponse]:
    """Stream transactions for evaluation."""

    async def request_iterator():
        async for txn in transactions:
            transaction = Struct()
            transaction.update(txn)

            yield evaluation_pb2.EvaluateRequest(
                request_id=f"req-{txn['id']}",
                transaction=transaction,
                jurisdictions=[evaluation_pb2.Jurisdiction(code="US")],
                domains=[evaluation_pb2.DOMAIN_AML],
                options=evaluation_pb2.EvaluationOptions(
                    generate_proof=True
                )
            )

    async for response in stub.EvaluateStream(request_iterator()):
        yield response

# Usage
async def process_transactions():
    async for response in evaluate_stream(stub, transaction_source):
        if response.status != evaluation_pb2.COMPLIANCE_STATUS_COMPLIANT:
            await handle_non_compliant(response)
```

## Rules Service

### Service Definition

```protobuf
// rules.proto
syntax = "proto3";

package arka.v1;

import "common.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

service RulesService {
  // Get a rule by ID
  rpc GetRule(GetRuleRequest) returns (Rule);

  // List rules with filtering
  rpc ListRules(ListRulesRequest) returns (ListRulesResponse);

  // Get rule version history
  rpc GetRuleHistory(GetRuleHistoryRequest) returns (GetRuleHistoryResponse);

  // Create a new rule
  rpc CreateRule(CreateRuleRequest) returns (Rule);

  // Update an existing rule
  rpc UpdateRule(UpdateRuleRequest) returns (Rule);

  // Delete a rule
  rpc DeleteRule(DeleteRuleRequest) returns (DeleteRuleResponse);

  // Validate rule DSL
  rpc ValidateRule(ValidateRuleRequest) returns (ValidateRuleResponse);

  // Compile rule to bytecode
  rpc CompileRule(CompileRuleRequest) returns (CompileRuleResponse);

  // Watch for rule changes (streaming)
  rpc WatchRules(WatchRulesRequest) returns (stream RuleEvent);
}

// Rule message
message Rule {
  string rule_id = 1;
  string version = 2;
  string name = 3;
  string description = 4;
  Jurisdiction jurisdiction = 5;
  Domain domain = 6;
  string dsl_source = 7;
  bytes bytecode = 8;
  RuleMetadata metadata = 9;
  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

message RuleMetadata {
  string author = 1;
  repeated string tags = 2;
  string regulatory_reference = 3;
  google.protobuf.Timestamp effective_date = 4;
  google.protobuf.Timestamp expiration_date = 5;
  map<string, string> labels = 6;
}

// Get rule
message GetRuleRequest {
  string rule_id = 1;
  string version = 2;  // Optional, defaults to latest
}

// List rules
message ListRulesRequest {
  repeated Jurisdiction jurisdictions = 1;
  repeated Domain domains = 2;
  repeated string tags = 3;
  string search_query = 4;
  PageRequest pagination = 5;
}

message ListRulesResponse {
  repeated Rule rules = 1;
  PageResponse pagination = 2;
}

// Rule history
message GetRuleHistoryRequest {
  string rule_id = 1;
  PageRequest pagination = 2;
}

message GetRuleHistoryResponse {
  repeated RuleVersion versions = 1;
  PageResponse pagination = 2;
}

message RuleVersion {
  string version = 1;
  string change_summary = 2;
  string author = 3;
  google.protobuf.Timestamp created_at = 4;
}

// Create rule
message CreateRuleRequest {
  string rule_id = 1;
  string name = 2;
  string description = 3;
  Jurisdiction jurisdiction = 4;
  Domain domain = 5;
  string dsl_source = 6;
  RuleMetadata metadata = 7;
}

// Update rule
message UpdateRuleRequest {
  string rule_id = 1;
  string name = 2;
  string description = 3;
  string dsl_source = 4;
  RuleMetadata metadata = 5;
  string change_summary = 6;
}

// Delete rule
message DeleteRuleRequest {
  string rule_id = 1;
  string reason = 2;
}

message DeleteRuleResponse {
  bool success = 1;
  string message = 2;
}

// Validate rule
message ValidateRuleRequest {
  string dsl_source = 1;
  Jurisdiction jurisdiction = 2;
  Domain domain = 3;
}

message ValidateRuleResponse {
  bool valid = 1;
  repeated ValidationError errors = 2;
  repeated ValidationWarning warnings = 3;
}

message ValidationError {
  int32 line = 1;
  int32 column = 2;
  string message = 3;
  string code = 4;
}

message ValidationWarning {
  int32 line = 1;
  int32 column = 2;
  string message = 3;
  string code = 4;
}

// Compile rule
message CompileRuleRequest {
  string dsl_source = 1;
  CompileOptions options = 2;
}

message CompileOptions {
  bool optimize = 1;
  bool debug_info = 2;
}

message CompileRuleResponse {
  bytes bytecode = 1;
  CompileMetrics metrics = 2;
  repeated CompileWarning warnings = 3;
}

message CompileMetrics {
  int32 bytecode_size = 1;
  int64 compile_time_ms = 2;
  int32 instruction_count = 3;
}

message CompileWarning {
  string message = 1;
  string code = 2;
}

// Watch rules
message WatchRulesRequest {
  repeated string rule_ids = 1;       // Specific rules or empty for all
  repeated Jurisdiction jurisdictions = 2;
  repeated Domain domains = 3;
}

message RuleEvent {
  RuleEventType type = 1;
  Rule rule = 2;
  google.protobuf.Timestamp timestamp = 3;
}

enum RuleEventType {
  RULE_EVENT_TYPE_UNSPECIFIED = 0;
  RULE_EVENT_TYPE_CREATED = 1;
  RULE_EVENT_TYPE_UPDATED = 2;
  RULE_EVENT_TYPE_DELETED = 3;
  RULE_EVENT_TYPE_ACTIVATED = 4;
  RULE_EVENT_TYPE_DEACTIVATED = 5;
}
```

### Client Usage

```python
from pact.proto import rules_pb2, rules_pb2_grpc

stub = rules_pb2_grpc.RulesServiceStub(channel)

# List rules
response = stub.ListRules(rules_pb2.ListRulesRequest(
    jurisdictions=[rules_pb2.Jurisdiction(code="US")],
    domains=[rules_pb2.DOMAIN_AML],
    pagination=rules_pb2.PageRequest(page_size=50)
))

for rule in response.rules:
    print(f"Rule: {rule.rule_id} v{rule.version}")
    print(f"  Domain: {rule.domain}")
    print(f"  Description: {rule.description}")

# Watch for rule changes
async def watch_rules():
    request = rules_pb2.WatchRulesRequest(
        jurisdictions=[rules_pb2.Jurisdiction(code="US")],
        domains=[rules_pb2.DOMAIN_AML]
    )

    async for event in stub.WatchRules(request):
        print(f"Rule event: {event.type}")
        print(f"Rule: {event.rule.rule_id}")
        await handle_rule_update(event)
```

## Governance Service

### Service Definition

```protobuf
// governance.proto
syntax = "proto3";

package arka.v1;

import "common.proto";
import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

service GovernanceService {
  // Submit a proposal
  rpc SubmitProposal(SubmitProposalRequest) returns (Proposal);

  // Get proposal by ID
  rpc GetProposal(GetProposalRequest) returns (Proposal);

  // List proposals
  rpc ListProposals(ListProposalsRequest) returns (ListProposalsResponse);

  // Submit approval/rejection
  rpc SubmitReview(SubmitReviewRequest) returns (SubmitReviewResponse);

  // Run simulation
  rpc RunSimulation(RunSimulationRequest) returns (SimulationResult);

  // Get simulation status (streaming for progress)
  rpc WatchSimulation(WatchSimulationRequest) returns (stream SimulationProgress);

  // Get proposal diff
  rpc GetProposalDiff(GetProposalDiffRequest) returns (ProposalDiff);

  // Watch proposal updates
  rpc WatchProposals(WatchProposalsRequest) returns (stream ProposalEvent);
}

// Proposal
message Proposal {
  string proposal_id = 1;
  string title = 2;
  string description = 3;
  ProposalStatus status = 4;
  ProposalStage current_stage = 5;
  repeated RuleChange changes = 6;
  ProposalMetadata metadata = 7;
  repeated Review reviews = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

enum ProposalStatus {
  PROPOSAL_STATUS_UNSPECIFIED = 0;
  PROPOSAL_STATUS_DRAFT = 1;
  PROPOSAL_STATUS_PENDING_REVIEW = 2;
  PROPOSAL_STATUS_APPROVED = 3;
  PROPOSAL_STATUS_REJECTED = 4;
  PROPOSAL_STATUS_DEPLOYED = 5;
  PROPOSAL_STATUS_WITHDRAWN = 6;
}

enum ProposalStage {
  PROPOSAL_STAGE_UNSPECIFIED = 0;
  PROPOSAL_STAGE_DRAFT = 1;
  PROPOSAL_STAGE_TECHNICAL_REVIEW = 2;
  PROPOSAL_STAGE_COMPLIANCE_REVIEW = 3;
  PROPOSAL_STAGE_LEGAL_REVIEW = 4;
  PROPOSAL_STAGE_STAGING = 5;
  PROPOSAL_STAGE_PRODUCTION = 6;
}

message RuleChange {
  ChangeType type = 1;
  string rule_id = 2;
  string old_version = 3;
  string new_version = 4;
  string dsl_diff = 5;
  string rationale = 6;
}

enum ChangeType {
  CHANGE_TYPE_UNSPECIFIED = 0;
  CHANGE_TYPE_CREATE = 1;
  CHANGE_TYPE_UPDATE = 2;
  CHANGE_TYPE_DELETE = 3;
}

message ProposalMetadata {
  string author = 1;
  string priority = 2;
  string regulatory_reference = 3;
  google.protobuf.Timestamp deadline = 4;
  map<string, string> labels = 5;
}

message Review {
  string reviewer = 1;
  string role = 2;
  ReviewDecision decision = 3;
  string comments = 4;
  ProposalStage stage = 5;
  google.protobuf.Timestamp timestamp = 6;
}

enum ReviewDecision {
  REVIEW_DECISION_UNSPECIFIED = 0;
  REVIEW_DECISION_APPROVED = 1;
  REVIEW_DECISION_REJECTED = 2;
  REVIEW_DECISION_REQUEST_CHANGES = 3;
}

// Submit proposal
message SubmitProposalRequest {
  string title = 1;
  string description = 2;
  repeated RuleChange changes = 3;
  ProposalMetadata metadata = 4;
}

// Get proposal
message GetProposalRequest {
  string proposal_id = 1;
}

// List proposals
message ListProposalsRequest {
  repeated ProposalStatus statuses = 1;
  repeated ProposalStage stages = 2;
  string author = 3;
  PageRequest pagination = 4;
}

message ListProposalsResponse {
  repeated Proposal proposals = 1;
  PageResponse pagination = 2;
}

// Submit review
message SubmitReviewRequest {
  string proposal_id = 1;
  ReviewDecision decision = 2;
  string comments = 3;
  repeated string required_changes = 4;
}

message SubmitReviewResponse {
  Proposal proposal = 1;
  bool stage_advanced = 2;
}

// Run simulation
message RunSimulationRequest {
  string proposal_id = 1;
  SimulationMode mode = 2;
  SimulationConfig config = 3;
}

enum SimulationMode {
  SIMULATION_MODE_UNSPECIFIED = 0;
  SIMULATION_MODE_HISTORICAL_REPLAY = 1;
  SIMULATION_MODE_SYNTHETIC = 2;
  SIMULATION_MODE_AB_COMPARISON = 3;
}

message SimulationConfig {
  google.protobuf.Timestamp start_date = 1;
  google.protobuf.Timestamp end_date = 2;
  int32 sample_size = 3;
  string sampling_strategy = 4;
  google.protobuf.Struct synthetic_config = 5;
}

message SimulationResult {
  string simulation_id = 1;
  SimulationStatus status = 2;
  SimulationMetrics metrics = 3;
  repeated SimulationFinding findings = 4;
  google.protobuf.Timestamp completed_at = 5;
}

enum SimulationStatus {
  SIMULATION_STATUS_UNSPECIFIED = 0;
  SIMULATION_STATUS_RUNNING = 1;
  SIMULATION_STATUS_COMPLETED = 2;
  SIMULATION_STATUS_FAILED = 3;
}

message SimulationMetrics {
  int32 total_evaluated = 1;
  int32 new_flags = 2;
  int32 removed_flags = 3;
  float false_positive_rate = 4;
  float coverage = 5;
}

message SimulationFinding {
  string type = 1;
  string severity = 2;
  string description = 3;
  google.protobuf.Struct details = 4;
}

// Watch simulation
message WatchSimulationRequest {
  string simulation_id = 1;
}

message SimulationProgress {
  string simulation_id = 1;
  float progress = 2;  // 0.0 to 1.0
  int32 processed = 3;
  int32 total = 4;
  string current_phase = 5;
  google.protobuf.Timestamp estimated_completion = 6;
}

// Get proposal diff
message GetProposalDiffRequest {
  string proposal_id = 1;
  DiffType diff_type = 2;
}

enum DiffType {
  DIFF_TYPE_UNSPECIFIED = 0;
  DIFF_TYPE_TEXTUAL = 1;
  DIFF_TYPE_STRUCTURAL = 2;
  DIFF_TYPE_BEHAVIORAL = 3;
  DIFF_TYPE_ALL = 4;
}

message ProposalDiff {
  repeated TextualDiff textual = 1;
  repeated StructuralDiff structural = 2;
  repeated BehavioralDiff behavioral = 3;
}

message TextualDiff {
  string rule_id = 1;
  repeated DiffHunk hunks = 2;
}

message DiffHunk {
  int32 old_start = 1;
  int32 old_count = 2;
  int32 new_start = 3;
  int32 new_count = 4;
  repeated DiffLine lines = 5;
}

message DiffLine {
  DiffLineType type = 1;
  string content = 2;
}

enum DiffLineType {
  DIFF_LINE_TYPE_UNSPECIFIED = 0;
  DIFF_LINE_TYPE_CONTEXT = 1;
  DIFF_LINE_TYPE_ADDED = 2;
  DIFF_LINE_TYPE_REMOVED = 3;
}

message StructuralDiff {
  string rule_id = 1;
  string element = 2;
  string change_type = 3;
  string path = 4;
  google.protobuf.Struct old_value = 5;
  google.protobuf.Struct new_value = 6;
}

message BehavioralDiff {
  string rule_id = 1;
  string scenario = 2;
  ComplianceStatus old_result = 3;
  ComplianceStatus new_result = 4;
  string significance = 5;
}

// Watch proposals
message WatchProposalsRequest {
  repeated string proposal_ids = 1;
  repeated ProposalStatus statuses = 2;
}

message ProposalEvent {
  ProposalEventType type = 1;
  Proposal proposal = 2;
  google.protobuf.Timestamp timestamp = 3;
}

enum ProposalEventType {
  PROPOSAL_EVENT_TYPE_UNSPECIFIED = 0;
  PROPOSAL_EVENT_TYPE_CREATED = 1;
  PROPOSAL_EVENT_TYPE_UPDATED = 2;
  PROPOSAL_EVENT_TYPE_REVIEWED = 3;
  PROPOSAL_EVENT_TYPE_STAGE_CHANGED = 4;
  PROPOSAL_EVENT_TYPE_DEPLOYED = 5;
}
```

## Agent Service

### Service Definition

```protobuf
// agent.proto
syntax = "proto3";

package arka.v1;

import "common.proto";
import "google/protobuf/timestamp.proto";

service AgentService {
  // Register agent with control plane
  rpc RegisterAgent(RegisterAgentRequest) returns (RegisterAgentResponse);

  // Send heartbeat
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);

  // Sync rules from control plane
  rpc SyncRules(SyncRulesRequest) returns (stream SyncRulesResponse);

  // Submit proofs for anchoring
  rpc SubmitProofs(stream SubmitProofsRequest) returns (SubmitProofsResponse);

  // Get agent status
  rpc GetAgentStatus(GetAgentStatusRequest) returns (AgentStatus);

  // List agents
  rpc ListAgents(ListAgentsRequest) returns (ListAgentsResponse);
}

// Register agent
message RegisterAgentRequest {
  string agent_did = 1;
  AgentCapabilities capabilities = 2;
  AgentConfig config = 3;
}

message AgentCapabilities {
  repeated string supported_domains = 1;
  repeated string supported_jurisdictions = 2;
  bool hardware_attestation = 3;
  int32 max_throughput = 4;
}

message AgentConfig {
  string region = 1;
  string cluster = 2;
  map<string, string> labels = 3;
}

message RegisterAgentResponse {
  string agent_id = 1;
  string agent_did = 2;
  bytes certificate = 3;
  SyncConfig sync_config = 4;
}

message SyncConfig {
  int32 sync_interval_seconds = 1;
  int32 heartbeat_interval_seconds = 2;
  repeated string preload_rules = 3;
}

// Heartbeat
message HeartbeatRequest {
  string agent_id = 1;
  AgentMetrics metrics = 2;
}

message AgentMetrics {
  int32 active_evaluations = 1;
  int32 queued_proofs = 2;
  int32 cached_rules = 3;
  float cpu_usage = 4;
  float memory_usage = 5;
  bool online = 6;
}

message HeartbeatResponse {
  bool ack = 1;
  repeated AgentCommand commands = 2;
}

message AgentCommand {
  AgentCommandType type = 1;
  bytes payload = 2;
}

enum AgentCommandType {
  AGENT_COMMAND_TYPE_UNSPECIFIED = 0;
  AGENT_COMMAND_TYPE_SYNC_NOW = 1;
  AGENT_COMMAND_TYPE_ROTATE_KEYS = 2;
  AGENT_COMMAND_TYPE_UPDATE_CONFIG = 3;
  AGENT_COMMAND_TYPE_SHUTDOWN = 4;
}

// Sync rules
message SyncRulesRequest {
  string agent_id = 1;
  string last_sync_version = 2;
  repeated string jurisdictions = 3;
  repeated string domains = 4;
}

message SyncRulesResponse {
  SyncType type = 1;
  Rule rule = 2;
  string current_version = 3;
  bool complete = 4;
}

enum SyncType {
  SYNC_TYPE_UNSPECIFIED = 0;
  SYNC_TYPE_ADD = 1;
  SYNC_TYPE_UPDATE = 2;
  SYNC_TYPE_DELETE = 3;
}

// Submit proofs
message SubmitProofsRequest {
  string agent_id = 1;
  repeated TrustProof proofs = 2;
}

message SubmitProofsResponse {
  int32 accepted = 1;
  int32 rejected = 2;
  repeated ProofRejection rejections = 3;
}

message ProofRejection {
  string proof_id = 1;
  string reason = 2;
}

// Agent status
message GetAgentStatusRequest {
  string agent_id = 1;
}

message AgentStatus {
  string agent_id = 1;
  string agent_did = 2;
  AgentState state = 3;
  AgentMetrics metrics = 4;
  string last_sync_version = 5;
  google.protobuf.Timestamp last_heartbeat = 6;
  google.protobuf.Timestamp registered_at = 7;
}

enum AgentState {
  AGENT_STATE_UNSPECIFIED = 0;
  AGENT_STATE_HEALTHY = 1;
  AGENT_STATE_DEGRADED = 2;
  AGENT_STATE_OFFLINE = 3;
  AGENT_STATE_MAINTENANCE = 4;
}

// List agents
message ListAgentsRequest {
  repeated AgentState states = 1;
  string region = 2;
  string cluster = 3;
  PageRequest pagination = 4;
}

message ListAgentsResponse {
  repeated AgentStatus agents = 1;
  PageResponse pagination = 2;
}
```

## Anchoring Service

### Service Definition

```protobuf
// anchoring.proto
syntax = "proto3";

package arka.v1;

import "common.proto";
import "google/protobuf/timestamp.proto";

service AnchoringService {
  // Submit proof for anchoring
  rpc SubmitAnchor(SubmitAnchorRequest) returns (SubmitAnchorResponse);

  // Get anchor status
  rpc GetAnchor(GetAnchorRequest) returns (Anchor);

  // Verify proof anchor
  rpc VerifyAnchor(VerifyAnchorRequest) returns (VerifyAnchorResponse);

  // Get Merkle proof for verification
  rpc GetMerkleProof(GetMerkleProofRequest) returns (MerkleProof);

  // Watch anchor status
  rpc WatchAnchor(WatchAnchorRequest) returns (stream AnchorEvent);
}

// Submit anchor
message SubmitAnchorRequest {
  string proof_id = 1;
  bytes proof_hash = 2;
  AnchorPriority priority = 3;
  string preferred_chain = 4;
}

enum AnchorPriority {
  ANCHOR_PRIORITY_UNSPECIFIED = 0;
  ANCHOR_PRIORITY_LOW = 1;
  ANCHOR_PRIORITY_NORMAL = 2;
  ANCHOR_PRIORITY_HIGH = 3;
  ANCHOR_PRIORITY_CRITICAL = 4;
}

message SubmitAnchorResponse {
  string anchor_id = 1;
  string batch_id = 2;
  google.protobuf.Timestamp estimated_anchor_time = 3;
}

// Get anchor
message GetAnchorRequest {
  string anchor_id = 1;
}

message Anchor {
  string anchor_id = 1;
  string proof_id = 2;
  AnchorState state = 3;
  string chain = 4;
  string transaction_hash = 5;
  int64 block_number = 6;
  bytes merkle_root = 7;
  google.protobuf.Timestamp anchored_at = 8;
}

enum AnchorState {
  ANCHOR_STATE_UNSPECIFIED = 0;
  ANCHOR_STATE_PENDING = 1;
  ANCHOR_STATE_BATCHED = 2;
  ANCHOR_STATE_SUBMITTED = 3;
  ANCHOR_STATE_CONFIRMED = 4;
  ANCHOR_STATE_FAILED = 5;
}

// Verify anchor
message VerifyAnchorRequest {
  string proof_id = 1;
  bytes proof_hash = 2;
  string chain = 3;
}

message VerifyAnchorResponse {
  bool verified = 1;
  Anchor anchor = 2;
  MerkleProof merkle_proof = 3;
  VerificationDetails details = 4;
}

message VerificationDetails {
  bool hash_matches = 1;
  bool merkle_valid = 2;
  bool chain_confirmed = 3;
  int32 confirmations = 4;
}

// Get Merkle proof
message GetMerkleProofRequest {
  string proof_id = 1;
  string anchor_id = 2;
}

message MerkleProof {
  bytes leaf_hash = 1;
  repeated bytes proof_path = 2;
  repeated bool proof_directions = 3;  // true = right, false = left
  bytes root_hash = 4;
}

// Watch anchor
message WatchAnchorRequest {
  string anchor_id = 1;
}

message AnchorEvent {
  string anchor_id = 1;
  AnchorState state = 2;
  string message = 3;
  google.protobuf.Timestamp timestamp = 4;
}
```

## Health Service

### Service Definition

```protobuf
// health.proto
syntax = "proto3";

package arka.v1;

service HealthService {
  // Standard gRPC health check
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  // Watch health status
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);

  // Deep health check with component status
  rpc DeepCheck(DeepHealthCheckRequest) returns (DeepHealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  ServingStatus status = 1;
}

enum ServingStatus {
  SERVING_STATUS_UNKNOWN = 0;
  SERVING_STATUS_SERVING = 1;
  SERVING_STATUS_NOT_SERVING = 2;
  SERVING_STATUS_SERVICE_UNKNOWN = 3;
}

message DeepHealthCheckRequest {
  bool include_metrics = 1;
}

message DeepHealthCheckResponse {
  ServingStatus overall_status = 1;
  map<string, ComponentHealth> components = 2;
  SystemMetrics metrics = 3;
}

message ComponentHealth {
  string name = 1;
  ServingStatus status = 2;
  string message = 3;
  int64 latency_ms = 4;
}

message SystemMetrics {
  float cpu_usage = 1;
  float memory_usage = 2;
  int32 active_connections = 3;
  int32 pending_requests = 4;
  float request_rate = 5;
  float error_rate = 6;
}
```

## Error Handling

### Error Codes

```protobuf
// errors.proto
syntax = "proto3";

package arka.v1;

// ARKA-specific error details
message PactErrorDetails {
  string error_code = 1;
  string error_type = 2;
  string message = 3;
  map<string, string> metadata = 4;
  repeated PactErrorDetails causes = 5;
}

// Error codes
// ARKA_INVALID_RULE: Invalid rule DSL or bytecode
// ARKA_RULE_NOT_FOUND: Rule not found
// ARKA_EVALUATION_TIMEOUT: Evaluation exceeded timeout
// ARKA_JURISDICTION_NOT_SUPPORTED: Jurisdiction not supported
// ARKA_PROOF_INVALID: Trust proof validation failed
// ARKA_ANCHOR_FAILED: Blockchain anchoring failed
// ARKA_AGENT_OFFLINE: Agent is offline
// ARKA_SYNC_CONFLICT: Rule sync conflict
// ARKA_QUOTA_EXCEEDED: Rate limit or quota exceeded
```

### Client Error Handling

```python
import grpc
from pact.proto import evaluation_pb2, errors_pb2

try:
    response = stub.Evaluate(request)
except grpc.RpcError as e:
    status_code = e.code()
    details = e.details()

    # Extract ARKA-specific error details
    for detail in e.trailing_metadata():
        if detail.key == 'arka-error-bin':
            pact_error = errors_pb2.PactErrorDetails()
            pact_error.ParseFromString(detail.value)
            print(f"ARKA Error: {pact_error.error_code}")
            print(f"Message: {pact_error.message}")

    if status_code == grpc.StatusCode.INVALID_ARGUMENT:
        print("Invalid request")
    elif status_code == grpc.StatusCode.NOT_FOUND:
        print("Rule not found")
    elif status_code == grpc.StatusCode.DEADLINE_EXCEEDED:
        print("Evaluation timeout")
    elif status_code == grpc.StatusCode.RESOURCE_EXHAUSTED:
        print("Rate limit exceeded")
```

## Best Practices

### Connection Management

1. **Reuse Channels**: Create channels once and reuse them
2. **Connection Pooling**: Use load balancing for multiple backends
3. **Keepalive**: Configure appropriate keepalive settings
4. **Graceful Shutdown**: Handle shutdown signals properly

### Performance

1. **Streaming**: Use streaming for high-throughput scenarios
2. **Compression**: Enable gzip compression for large payloads
3. **Batching**: Use batch APIs when possible
4. **Async**: Use async stubs for non-blocking operations

### Security

1. **mTLS**: Always use mTLS in production
2. **Certificate Rotation**: Implement certificate rotation
3. **Authentication**: Use proper authentication metadata
4. **Rate Limiting**: Implement client-side rate limiting

## Related Documentation

- [REST API Reference](./rest-api.md)
- [SDK Reference](./sdk-reference.md)
- [Authentication Guide](../security/authentication.md)
