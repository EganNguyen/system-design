# High-Level Design: Automated DevOps & Chaos Engineering Control Plane
## Non-LLM Multi-Agent System (MAS) for Cluster Recovery & Chaos Orchestration

---

**Document Version:** 1.0.0
**Status:** Draft — Production-Ready Architecture
**Target Stack:** Go 1.22+, Apache Kafka / NATS JetStream, gRPC + Protobuf, Redis Cluster, Raft Consensus
**Classification:** Principal Architecture Reference

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Goals & Non-Goals](#2-system-goals--non-goals)
3. [Architectural Constraints & Principles](#3-architectural-constraints--principles)
4. [High-Level Architecture Overview](#4-high-level-architecture-overview)
5. [Agent Taxonomy & Brain Engines](#5-agent-taxonomy--brain-engines)
6. [Event-Driven Communication Layer](#6-event-driven-communication-layer)
7. [Distributed State Management](#7-distributed-state-management)
8. [Agent Interaction Flows](#8-agent-interaction-flows)
9. [Chaos Engineering Orchestration Subsystem](#9-chaos-engineering-orchestration-subsystem)
10. [Cluster Recovery Subsystem](#10-cluster-recovery-subsystem)
11. [Anomaly Detection Engine](#11-anomaly-detection-engine)
12. [Control Plane & Operator API](#12-control-plane--operator-api)
13. [Observability & Audit Trail](#13-observability--audit-trail)
14. [Security Model](#14-security-model)
15. [Deployment Architecture](#15-deployment-architecture)
16. [Data Models & Protobuf Contracts](#16-data-models--protobuf-contracts)
17. [Performance Targets & SLOs](#17-performance-targets--slos)
18. [Failure Modes & Mitigations](#18-failure-modes--mitigations)
19. [Technology Stack Summary](#19-technology-stack-summary)
20. [Roadmap & Phasing](#20-roadmap--phasing)

---

## 1. Executive Summary

This document defines the High-Level Design for a fully deterministic, event-driven **Multi-Agent System (MAS)** that automates two critical platform reliability domains:

1. **Cluster Recovery Automation** — autonomous detection, triage, and remediation of degraded Kubernetes clusters without human intervention.
2. **Chaos Engineering Orchestration** — scheduled and triggered injection of controlled failure scenarios, hypothesis validation, and automated blast-radius containment.

The system is architecturally grounded in **zero LLM dependency**. All agent intelligence is derived from Finite State Machines (FSM), Rule Engines, Directed Acyclic Graphs (DAG), Z-Score anomaly detection, and Isolation Forest classifiers operating on structured telemetry streams. The target language is **Go**, chosen for its goroutine model, sub-millisecond scheduling, and compile-time type safety.

The platform is designed to operate at production scale: 1,000+ node clusters, 10,000+ events/second ingestion, and sub-10ms agent-to-agent communication latency.

---

## 2. System Goals & Non-Goals

### 2.1 Goals

| ID | Goal |
|----|------|
| G-01 | Detect cluster anomalies within 30 seconds of metric deviation |
| G-02 | Execute recovery runbooks autonomously with <5 minute MTTR for known failure classes |
| G-03 | Orchestrate chaos experiments with configurable blast radius, auto-rollback, and hypothesis tracking |
| G-04 | Guarantee exactly-once or at-least-once semantics on all remediation actions via idempotency keys |
| G-05 | Prevent split-brain in multi-agent decision-making using distributed consensus |
| G-06 | Provide a full audit trail of every agent decision, state transition, and action executed |
| G-07 | Allow operators to inject, pause, dry-run, or override any agent at runtime |
| G-08 | Support multi-cluster federation from a single control plane |

### 2.2 Non-Goals

| ID | Non-Goal |
|----|----------|
| NG-01 | No LLM, NLP, or generative AI components — ever |
| NG-02 | Not a general-purpose workflow engine (Temporal, Argo) — purpose-built domain logic only |
| NG-03 | Does not replace Prometheus/Alertmanager for raw alerting — consumes their output |
| NG-04 | No self-modifying rule sets without operator approval (no autonomous policy learning) |

---

## 3. Architectural Constraints & Principles

### 3.1 Core Constraints

**C-01 — Zero LLM Dependency**
All decision logic must be reducible to formal specifications: FSM transition tables, YAML/Rego rule sets, DAG execution plans, or mathematical scoring functions. No component may invoke an inference endpoint.

**C-02 — Determinism**
Given the same input signal and system state, any agent must produce the same output action. This enables reproducible chaos experiments and audit-provable recovery decisions.

**C-03 — Sub-10ms Inter-Agent Overhead**
Synchronous agent-to-agent calls use gRPC over Unix Domain Sockets (same host) or mTLS TCP (cross-host). Protobuf encoding overhead is bounded at <2ms at P99 for payloads under 16KB.

**C-04 — Eventual Consistency with Strong Leases**
Agents acquire distributed leases via Redis `SET NX PX` or Raft-backed locks before mutating cluster state. No optimistic mutation without a lease.

**C-05 — Fail-Safe by Default**
Any agent reaching an undefined state transition defaults to `HALT_AND_ALERT`, never to an uncontrolled action.

### 3.2 Design Principles

- **Single Responsibility per Agent** — each agent owns exactly one concern domain.
- **Immutable Event Log** — all state changes are sourced from an append-only Kafka topic.
- **Blast Radius Budgets** — every chaos experiment carries an explicit `BlastRadiusBudget` struct that the Chaos Governor enforces before approval.
- **Idempotency Keys** — all remediation RPCs carry a UUID idempotency key; executors reject duplicate invocations within a TTL window.
- **Circuit Breakers on Actuators** — actuator agents (those that touch live infrastructure) wrap all external calls in a circuit breaker with exponential backoff.

---

## 4. High-Level Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OPERATOR CONTROL PLANE                              │
│         REST/gRPC API  ·  Dashboard  ·  CLI (cectl)  ·  GitOps Sync        │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │  Control Events / Policy Push
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATION TIER                                   │
│                                                                             │
│  ┌─────────────────────┐    ┌──────────────────────┐                        │
│  │  Recovery Governor  │    │   Chaos Governor     │                        │
│  │  (FSM + DAG Engine) │    │  (FSM + Rule Engine) │                        │
│  └──────────┬──────────┘    └──────────┬───────────┘                        │
│             │  gRPC                    │  gRPC                              │
│  ┌──────────▼──────────────────────────▼───────────┐                        │
│  │            Agent Coordinator (Raft Leader)       │                        │
│  │        Global Lock Registry  ·  Work Queue       │                        │
│  └──────────────────────────┬──────────────────────┘                        │
└─────────────────────────────│───────────────────────────────────────────────┘
                              │  Async Events (Kafka/NATS)
┌─────────────────────────────▼───────────────────────────────────────────────┐
│                        INTELLIGENCE TIER                                    │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐    │
│  │ Anomaly Detector │  │  Triage Agent    │  │  Hypothesis Validator  │    │
│  │ (Z-Score + IF)   │  │  (Rule Engine)   │  │  (Statistical Engine)  │    │
│  └──────────┬───────┘  └──────────┬───────┘  └────────────┬───────────┘    │
└────────────-│──────────────────────│───────────────────────│────────────────┘
              │                      │                       │
              └──────────────────────┼───────────────────────┘
                                     │  Kafka Topics
┌────────────────────────────────────▼────────────────────────────────────────┐
│                         EXECUTION TIER                                      │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────────────────┐   │
│  │  Recovery       │  │  Chaos Injector │  │   Rollback Agent          │   │
│  │  Actuator Agent │  │  Agent          │  │   (DAG Reverse Traversal) │   │
│  └────────┬────────┘  └────────┬────────┘  └────────────┬──────────────┘   │
│           │  Kubernetes API    │  Fault Injection SDK   │                  │
└───────────│────────────────────│────────────────────────│──────────────────┘
            │                    │                        │
┌───────────▼────────────────────▼────────────────────────▼──────────────────┐
│                      INFRASTRUCTURE LAYER                                   │
│                                                                             │
│     Kubernetes Clusters        Telemetry Pipeline       Redis Cluster       │
│     (multi-cluster federation) (Prometheus + OTEL)      (State + Leases)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Tier Responsibilities

| Tier | Agents | Primary Protocol | State Scope |
|------|--------|-----------------|-------------|
| Orchestration | Recovery Governor, Chaos Governor, Agent Coordinator | gRPC (sync) | Global — Raft |
| Intelligence | Anomaly Detector, Triage Agent, Hypothesis Validator | Kafka (async) | Local + Redis |
| Execution | Recovery Actuator, Chaos Injector, Rollback Agent | Kafka + K8s API | Ephemeral |

---

## 5. Agent Taxonomy & Brain Engines

### 5.1 Agent Registry

| Agent ID | Name | Brain Engine | Tier | Concurrency Model |
|----------|------|--------------|------|-------------------|
| AG-01 | Anomaly Detector | Z-Score + Isolation Forest | Intelligence | Goroutine per metric stream |
| AG-02 | Triage Agent | Drools-equivalent Rule Engine (OPA/Rego) | Intelligence | Single consumer per partition |
| AG-03 | Recovery Governor | FSM (6 states) + DAG Planner | Orchestration | Single elected leader |
| AG-04 | Chaos Governor | FSM (7 states) + Rule Engine | Orchestration | Single elected leader |
| AG-05 | Agent Coordinator | Raft Consensus | Orchestration | Raft cluster (3 nodes) |
| AG-06 | Recovery Actuator | FSM (4 states) + Circuit Breaker | Execution | Worker pool (configurable) |
| AG-07 | Chaos Injector | DAG Executor | Execution | Worker pool |
| AG-08 | Hypothesis Validator | Statistical Engine (t-test, SLO diff) | Intelligence | Event-driven |
| AG-09 | Rollback Agent | DAG Reverse Traversal | Execution | On-demand goroutine |
| AG-10 | Blast Radius Guard | Threshold Rule Engine | Orchestration | Inline middleware |

### 5.2 FSM Specifications

#### AG-03: Recovery Governor FSM

```
States: IDLE → OBSERVING → TRIAGING → PLANNING → EXECUTING → VALIDATING → IDLE
        Any state → HALTED (on error or operator override)

Transitions:
  IDLE        --[AnomalyEvent received]-->          OBSERVING
  OBSERVING   --[TriageComplete, severity >= P2]-->  PLANNING
  OBSERVING   --[TriageComplete, severity < P2]-->   IDLE
  PLANNING    --[DAGPlanApproved]-->                 EXECUTING
  PLANNING    --[NoPlanFound]-->                     HALTED (alert)
  EXECUTING   --[AllActionsComplete]-->              VALIDATING
  EXECUTING   --[ActionFailed, retries_exhausted]-->  HALTED
  VALIDATING  --[HealthCheckPass]-->                 IDLE
  VALIDATING  --[HealthCheckFail]-->                 EXECUTING (escalate plan)
  Any         --[OperatorHalt]-->                    HALTED
  HALTED      --[OperatorResume]-->                  IDLE
```

#### AG-04: Chaos Governor FSM

```
States: IDLE → SCHEDULING → BLAST_RADIUS_CHECK → APPROVED → RUNNING
        → MONITORING → CONCLUDING → IDLE
        Any → ABORTED (rollback triggered)

Transitions:
  IDLE                --[ExperimentScheduled]-->     SCHEDULING
  SCHEDULING          --[PreflightPass]-->            BLAST_RADIUS_CHECK
  BLAST_RADIUS_CHECK  --[BudgetSatisfied]-->          APPROVED
  BLAST_RADIUS_CHECK  --[BudgetExceeded]-->           ABORTED
  APPROVED            --[InjectionStarted]-->         RUNNING
  RUNNING             --[MonitoringWindowOpen]-->     MONITORING
  MONITORING          --[HypothesisValidated]-->      CONCLUDING
  MONITORING          --[SLOBreach detected]-->       ABORTED
  MONITORING          --[TimeoutExceeded]-->           ABORTED
  CONCLUDING          --[ReportGenerated]-->           IDLE
  ABORTED             --[RollbackComplete]-->          IDLE
```

### 5.3 Rule Engine Design (AG-02: Triage Agent)

The Triage Agent uses an **OPA (Open Policy Agent)** instance embedded as a Go library (`github.com/open-policy-agent/opa/rego`). Rules are authored in Rego and loaded from a versioned ConfigMap at startup, with hot-reload support via inotify.

**Example Rego Rule (simplified):**
```rego
package triage

default severity = "P4"

severity = "P1" {
    input.anomaly_type == "OOMKill"
    input.affected_pods > 5
    input.namespace == "production"
}

severity = "P2" {
    input.anomaly_type == "HighLatency"
    input.p99_ms > 2000
    input.duration_seconds > 300
}

remediation_class = "rolling_restart" {
    input.anomaly_type == "MemoryLeak"
    input.severity != "P1"
}

remediation_class = "node_drain_and_replace" {
    input.anomaly_type == "NodeNotReady"
    input.node_age_hours > 720
}
```

### 5.4 DAG Execution Engine (AG-03, AG-07, AG-09)

DAGs represent remediation or injection plans as a directed graph of `ActionNode`s. The engine:

- Builds the DAG from a registered `RunbookTemplate` (stored in etcd/ConfigMap).
- Performs topological sort via Kahn's algorithm before execution.
- Executes independent branches concurrently using a bounded goroutine pool (semaphore pattern).
- Persists node execution state to Redis with TTL for crash recovery and idempotency.
- Supports rollback by reversing the completed subgraph (only nodes with `rollback_fn` defined).

```
DAG Example — Rolling Restart Runbook:

[CordonNode] ──► [DrainNode] ──► [DeletePod] ──► [WaitForHealthy]
                                 [ScaleUp]   ──►     │
                                 (parallel)  ──►     ▼
                                                [UncordonNode] ──► [ValidateEndpoint]
```

---

## 6. Event-Driven Communication Layer

### 6.1 Kafka Topic Architecture

| Topic | Partitions | Replication | Retention | Consumers |
|-------|-----------|-------------|-----------|-----------|
| `telemetry.raw` | 64 | 3 | 1h | Anomaly Detector |
| `anomaly.detected` | 16 | 3 | 24h | Triage Agent |
| `triage.result` | 8 | 3 | 24h | Recovery Governor, Chaos Governor |
| `recovery.plan` | 4 | 3 | 72h | Recovery Actuator |
| `chaos.experiment` | 4 | 3 | 168h | Chaos Injector |
| `action.executed` | 16 | 3 | 72h | Hypothesis Validator, Audit Log |
| `rollback.trigger` | 4 | 3 | 72h | Rollback Agent |
| `agent.heartbeat` | 8 | 3 | 10m | Agent Coordinator |
| `operator.command` | 2 | 3 | 24h | All agents (broadcast) |

**Partition Strategy:** Events are keyed by `cluster_id + namespace` to maintain ordering guarantees within a namespace while allowing cross-namespace parallelism.

### 6.2 NATS JetStream (Alternative / Supplementary)

For environments where Kafka is not available or for sub-millisecond internal pub/sub between co-located agents, NATS JetStream subjects mirror the Kafka topic structure:

```
ce.telemetry.raw.{cluster_id}.{node_id}
ce.anomaly.{cluster_id}.{severity}
ce.recovery.plan.{cluster_id}.{runbook_id}
ce.chaos.experiment.{cluster_id}.{experiment_id}
```

### 6.3 gRPC Service Definitions

Synchronous calls between governors and actuators use gRPC with Protobuf. All services implement deadline propagation and carry a `trace_id` in metadata.

```protobuf
// Core RPC contracts (summary)

service RecoveryGovernorService {
  rpc SubmitAnomalyForTriage (AnomalyEvent) returns (TriageResult);
  rpc ApprovePlan (PlanApprovalRequest) returns (PlanApprovalResponse);
  rpc GetGovernorState (Empty) returns (GovernorStateResponse);
  rpc OverrideState (OperatorOverrideRequest) returns (OperatorOverrideResponse);
}

service ChaosGovernorService {
  rpc ScheduleExperiment (ExperimentSpec) returns (ExperimentScheduleResponse);
  rpc AbortExperiment (AbortRequest) returns (AbortResponse);
  rpc GetExperimentStatus (ExperimentStatusRequest) returns (ExperimentStatus);
}

service ActuatorService {
  rpc ExecuteAction (ActionRequest) returns (ActionResult);
  rpc DryRunAction (ActionRequest) returns (DryRunResult);
  rpc RollbackAction (RollbackRequest) returns (RollbackResult);
}

service AgentCoordinatorService {
  rpc AcquireLease (LeaseRequest) returns (LeaseResponse);
  rpc ReleaseLease (LeaseReleaseRequest) returns (Empty);
  rpc ElectLeader (LeaderElectionRequest) returns (LeaderElectionResponse);
  rpc Heartbeat (HeartbeatRequest) returns (Empty);
}
```

### 6.4 Message Envelope Schema

Every Kafka/NATS message uses a standard envelope:

```protobuf
message EventEnvelope {
  string  event_id        = 1;  // UUID v4
  string  event_type      = 2;  // e.g., "anomaly.detected"
  string  source_agent_id = 3;
  int64   timestamp_ns    = 4;  // Unix nanoseconds
  string  cluster_id      = 5;
  string  trace_id        = 6;  // OpenTelemetry trace propagation
  bytes   payload         = 7;  // Serialized inner Protobuf message
  map<string, string> labels = 8;
}
```

---

## 7. Distributed State Management

### 7.1 State Classification

| State Class | Store | TTL | Consistency Model |
|-------------|-------|-----|-------------------|
| Agent FSM state | Redis Hash | Session | Strong (lease-gated write) |
| DAG execution progress | Redis Hash | 4h | At-least-once with idempotency |
| Global agent leases | Redis `SET NX PX` | 30s (auto-renew) | Exclusive |
| Experiment metadata | Redis + Kafka compacted topic | 7 days | Eventual (Kafka) + Cache (Redis) |
| Cluster health snapshot | Redis Sorted Set | 5m rolling | Eventual |
| Raft cluster membership | Embedded Raft (hashicorp/raft) | Permanent | Strong |
| Runbook templates | etcd / Kubernetes ConfigMap | Permanent | Operator-managed |
| Audit log | Kafka + ClickHouse | 90 days | Append-only |

### 7.2 Redis State Schema

```
# Agent FSM state
HSET agent:{agent_id}:state  current_state  "OBSERVING"
                             entered_at     "1719000000"
                             cluster_id     "prod-us-east-1"
                             trace_id       "abc123"

# DAG node progress
HSET dag:{plan_id}:node:{node_id}  status        "COMPLETED"
                                    started_at    "1719000001"
                                    completed_at  "1719000010"
                                    output        "{...json...}"

# Distributed lease
SET lease:recovery_governor:{cluster_id}  {agent_instance_id}  NX PX 30000

# Cluster health snapshot (Sorted Set — score = timestamp)
ZADD cluster:{cluster_id}:health  {timestamp}  "{health_snapshot_json}"
```

### 7.3 Raft Consensus Layer (Agent Coordinator)

The Agent Coordinator runs as a 3-node Raft cluster using `hashicorp/raft` embedded in Go. Responsibilities:

- **Leader Election** for Recovery Governor and Chaos Governor (only one instance active per cluster at a time).
- **Distributed Lock Registry** for cross-agent coordination.
- **Configuration Propagation** for runtime policy updates.

Raft log entries are typed commands:

```go
type RaftCommand struct {
    Type      CommandType  // ELECT_LEADER | ACQUIRE_LOCK | RELEASE_LOCK | UPDATE_POLICY
    AgentID   string
    ClusterID string
    Payload   []byte       // Protobuf-encoded command-specific data
    Timestamp int64
}
```

### 7.4 Split-Brain Prevention Protocol

When a network partition is detected:
1. The Agent Coordinator demotes any non-quorum governor to `STANDBY` state via gRPC.
2. Governors in `STANDBY` reject all incoming triage/recovery events (return `NOOP`).
3. Actuators refuse to execute if they cannot confirm governor liveness within 5 seconds.
4. Partition healed → Raft re-elects → governor re-promoted → backlog replayed from Kafka offset.

---

## 8. Agent Interaction Flows

### 8.1 Flow 1: Anomaly Detection → Cluster Recovery

```
Prometheus Alertmanager
        │  Webhook (HTTP POST)
        ▼
[Telemetry Ingestion Sidecar]
        │  Publish: ce.telemetry.raw
        ▼
[AG-01: Anomaly Detector]
  · Compute Z-Score on metric window
  · If |Z| > threshold → flag anomaly
  · Run Isolation Forest on multivariate features
  · Publish: ce.anomaly.detected (with anomaly_type, features, score)
        │
        ▼
[AG-02: Triage Agent]
  · Evaluate OPA/Rego rules against anomaly event
  · Assign severity (P1–P4), remediation_class, affected_scope
  · Publish: ce.triage.result
        │
        ▼
[AG-03: Recovery Governor]  (holds Raft-elected lease)
  · FSM: IDLE → OBSERVING
  · Acquire Redis lease: lease:recovery_governor:{cluster_id}
  · Load matching RunbookTemplate from etcd
  · Build DAG execution plan
  · FSM: OBSERVING → PLANNING → EXECUTING
  · gRPC: ExecuteAction → AG-06: Recovery Actuator
        │
        ▼
[AG-06: Recovery Actuator]
  · Validate idempotency key (Redis lookup)
  · Check circuit breaker state
  · Execute K8s API calls (Drain, Delete, Scale, Restart)
  · Report result via Kafka: ce.action.executed
        │
        ▼
[AG-03: Recovery Governor]
  · FSM: EXECUTING → VALIDATING
  · Poll cluster health endpoint
  · Health check pass → FSM: VALIDATING → IDLE
  · Health check fail → escalate plan or HALTED
```

### 8.2 Flow 2: Chaos Experiment Lifecycle

```
Operator / GitOps (ExperimentSpec YAML)
        │  REST API POST /experiments
        ▼
[AG-04: Chaos Governor]
  · FSM: IDLE → SCHEDULING
  · Preflight checks: maintenance window, freeze flags, active incidents
  · FSM: SCHEDULING → BLAST_RADIUS_CHECK
        │
        ▼
[AG-10: Blast Radius Guard]  (inline middleware)
  · Evaluate: affected_pods / total_pods ≤ budget.max_pod_pct
  · Evaluate: experiment does not overlap with P1 incident
  · Evaluate: SLO error budget remaining ≥ experiment risk estimate
  · Return: APPROVED or REJECTED with reason
        │
        ▼
[AG-04: Chaos Governor]
  · FSM: BLAST_RADIUS_CHECK → APPROVED → RUNNING
  · Publish: ce.chaos.experiment (ExperimentSpec + ApprovalToken)
        │
        ▼
[AG-07: Chaos Injector]
  · Parse DAG from ExperimentSpec.fault_graph
  · Execute faults sequentially/in-parallel per DAG topology:
      - PodKill, NetworkPartition, CPUHog, MemoryPressure, DiskFill, LatencyInjection
  · Stream execution events to ce.action.executed
        │
        ▼
[AG-04: Chaos Governor] (monitoring loop)
  · FSM: RUNNING → MONITORING
  · Subscribe to ce.action.executed + cluster SLO metrics
  · Evaluate hypothesis conditions every 10s
        │
     ┌──┴──┐
SLO breach?  No SLO breach
     │              │
     ▼              ▼
[AG-09: Rollback]  [AG-08: Hypothesis Validator]
  · Traverse DAG    · Compute statistical significance
  · Reverse actions   (Welch's t-test on SLO baseline vs experiment window)
  · FSM: ABORTED    · Generate ExperimentReport
                    · FSM: CONCLUDING → IDLE
```

---

## 9. Chaos Engineering Orchestration Subsystem

### 9.1 Experiment Specification Model

```protobuf
message ExperimentSpec {
  string   experiment_id     = 1;
  string   name              = 2;
  string   cluster_id        = 3;
  string   namespace         = 4;
  
  Hypothesis hypothesis      = 5;
  FaultGraph fault_graph     = 6;
  BlastRadiusBudget budget   = 7;
  Schedule schedule          = 8;
  SteadyStateProbe probes    = 9;
  
  Duration monitoring_window = 10;
  Duration timeout           = 11;
  bool     dry_run           = 12;
  string   rollback_policy   = 13; // "auto" | "manual" | "never"
}

message Hypothesis {
  string description        = 1;
  repeated SLOThreshold slos = 2;  // e.g., p99_latency_ms < 500
}

message BlastRadiusBudget {
  float  max_pod_percentage    = 1;  // 0.0–1.0
  int32  max_nodes_affected    = 2;
  bool   allow_production      = 3;
  float  min_error_budget_pct  = 4;  // minimum SLO error budget remaining
}

message FaultGraph {
  repeated FaultNode nodes = 1;
  repeated FaultEdge edges = 2;  // DAG topology
}

message FaultNode {
  string    node_id    = 1;
  FaultType fault_type = 2;  // POD_KILL | NETWORK_PARTITION | CPU_HOG | etc.
  map<string, string> params = 3;
  Duration  duration   = 4;
}
```

### 9.2 Supported Fault Types

| Fault Type | Mechanism | Scope | Rollback |
|-----------|-----------|-------|----------|
| `POD_KILL` | K8s Delete Pod | Pod | Auto (K8s recreates) |
| `NETWORK_PARTITION` | tc/iptables via DaemonSet | Node/Namespace | tc rule removal |
| `CPU_HOG` | stress-ng via Job | Container | Job deletion |
| `MEMORY_PRESSURE` | stress-ng --vm | Container | Job deletion |
| `DISK_FILL` | dd via Job | Node | File cleanup Job |
| `LATENCY_INJECTION` | tc netem via DaemonSet | Pod network | tc rule removal |
| `PACKET_LOSS` | tc netem | Pod network | tc rule removal |
| `NODE_DRAIN` | K8s Cordon + Drain | Node | Uncordon |
| `CLOCK_SKEW` | chronyc via DaemonSet | Node | chronyc reset |
| `DEPENDENCY_KILL` | Network policy injection | Service | Policy removal |

### 9.3 Hypothesis Validation Engine (AG-08)

The validator uses a sliding window statistical comparison:

```
Input:
  · baseline_window: SLO metrics from T-10min to T-0 (pre-experiment)
  · experiment_window: SLO metrics during fault injection

Algorithm:
  1. For each SLOThreshold in Hypothesis.slos:
     a. Collect metric samples in both windows
     b. Compute mean, stddev, sample size
     c. Perform Welch's t-test (unequal variance): t = (μ₁ - μ₂) / sqrt(σ₁²/n₁ + σ₂²/n₂)
     d. Compute p-value from t-distribution (lookup table, no external lib)
     e. If p < 0.05 AND metric_in_experiment > threshold → SLO_BREACH

  2. Score experiment:
     · PASS: all SLOs within threshold, p-values non-significant
     · DEGRADED: SLOs within threshold but approaching limits
     · FAIL: one or more SLO breaches detected

Output: HypothesisResult {outcome, slo_results[], confidence_score, raw_stats}
```

---

## 10. Cluster Recovery Subsystem

### 10.1 Runbook Template Registry

Runbooks are DAG templates stored as versioned Protobuf in etcd. Each runbook maps to one or more `(anomaly_type, severity, remediation_class)` tuples.

| Runbook ID | Anomaly Class | Severity | Actions |
|-----------|--------------|----------|---------|
| RB-001 | `OOMKill` | P1-P3 | Scale Deployment, Adjust ResourceLimits, Alert |
| RB-002 | `NodeNotReady` | P1-P2 | Cordon, Drain, Terminate Node, Provision Replacement |
| RB-003 | `HighErrorRate` | P2-P3 | Rolling Restart, Canary Traffic Shift, Validate |
| RB-004 | `DiskPressure` | P2 | Evict Completed Jobs, Clean Image Cache, Alert if >80% |
| RB-005 | `CrashLoopBackOff` | P1-P2 | Capture Logs, Scale to 0, Scale to N, Validate |
| RB-006 | `ServiceUnreachable` | P1 | DNS Check, Network Policy Audit, Restart CoreDNS |
| RB-007 | `HPA_MaxReached` | P2 | Burst Node Pool, Adjust HPA Max, Notify Capacity |

### 10.2 Recovery Actuator Safety Gates

Before executing any K8s mutation, the Recovery Actuator evaluates a **safety gate chain**:

```
Gate 1: Freeze Check
  → Is cluster_id in global freeze list? (Redis key: freeze:{cluster_id})
  → Reject if true

Gate 2: Idempotency Check
  → Has action_id been executed within TTL? (Redis: idempotent:{action_id})
  → Return cached result if true

Gate 3: Concurrent Mutation Lock
  → Acquire Redis lease: mutlock:{cluster_id}:{namespace}
  → Reject if cannot acquire within 2s

Gate 4: Quota Check
  → Actions per cluster per hour ≤ configured max (default: 50)
  → Reject if exceeded

Gate 5: Dry Run Mode
  → If cluster or action is in dry_run mode → log action, return NOOP
```

### 10.3 MTTR Target Breakdown

| Phase | Target Duration | Agent |
|-------|----------------|-------|
| Anomaly detection | ≤ 30s | AG-01 |
| Triage | ≤ 10s | AG-02 |
| Plan generation | ≤ 5s | AG-03 |
| Action execution (simple) | ≤ 120s | AG-06 |
| Validation | ≤ 60s | AG-03 |
| **Total P50 MTTR** | **≤ 4 min** | |
| **Total P99 MTTR** | **≤ 10 min** | |

---

## 11. Anomaly Detection Engine

### 11.1 AG-01 Internal Architecture

The Anomaly Detector runs a streaming pipeline per metric type:

```
Metric Stream (Kafka Consumer Group: anomaly-detector)
    │
    ▼
[Feature Extractor]
  · Rolling window (1m, 5m, 15m) for mean, stddev, rate
  · Feature vector: [value, rate_of_change, z_score, window_mean, window_std]
    │
    ▼
[Z-Score Detector]
  · σ = (x - μ) / σ_window
  · Threshold: |σ| > 3.0 for "spike", |σ| > 2.0 for "drift"
  · Adaptive baseline: EWMA (α = 0.1) to avoid stale baselines
    │
    ▼
[Isolation Forest Detector]
  · Feature matrix: multivariate (CPU%, memory%, latency_p99, error_rate)
  · Trained offline on 30-day historical baseline (Go implementation, no Python)
  · Anomaly score = avg_path_length / avg_path_length_normal
  · Threshold: score > 0.65 → anomalous
    │
    ▼
[Ensemble Voter]
  · Vote: Z-Score result (weight 0.4) + IF result (weight 0.6)
  · Weighted sum > 0.5 → emit AnomalyEvent
  · Below threshold → discard (no event)
```

### 11.2 Isolation Forest — Go Implementation Notes

The Isolation Forest model is:
- **Trained** offline using a Go service with `gonum.org/v1/gonum/mat` for matrix operations.
- **Serialized** to a compact binary format (Protobuf + msgpack trees) and loaded at agent startup.
- **Inference** runs in O(log n) per sample — fits within 1ms at P99 for standard feature vectors.
- **Retrained** on a 7-day rolling basis by a scheduled batch job (Kubernetes CronJob), with hot-swap via Redis key pointing to the latest model version.

### 11.3 Metric Feature Matrix

| Feature | Source | Window | Type |
|---------|--------|--------|------|
| `cpu_utilization_pct` | cAdvisor | 1m/5m | Gauge |
| `memory_rss_bytes` | cAdvisor | 5m | Gauge |
| `http_request_error_rate` | Prometheus | 1m | Rate |
| `http_p99_latency_ms` | Prometheus | 1m | Histogram |
| `pod_restart_count_delta` | K8s API | 5m | Counter delta |
| `node_disk_pressure` | K8s Events | instant | Boolean |
| `network_rx_errors` | Node exporter | 1m | Rate |
| `gc_pause_ms` | JVM/Go runtime | 1m | Histogram |

---

## 12. Control Plane & Operator API

### 12.1 REST API Surface

All endpoints require mTLS + JWT authentication (RBAC-enforced).

```
# Recovery Management
GET    /v1/clusters/{cluster_id}/recovery/status
POST   /v1/clusters/{cluster_id}/recovery/freeze
DELETE /v1/clusters/{cluster_id}/recovery/freeze
GET    /v1/clusters/{cluster_id}/recovery/history

# Chaos Experiments
POST   /v1/experiments                        # Schedule new experiment
GET    /v1/experiments/{experiment_id}        # Get status
DELETE /v1/experiments/{experiment_id}        # Abort experiment
GET    /v1/experiments/{experiment_id}/report # Get final report
POST   /v1/experiments/{experiment_id}/dry-run

# Runbook Management
GET    /v1/runbooks
POST   /v1/runbooks                           # Register new runbook
PUT    /v1/runbooks/{runbook_id}              # Update runbook
GET    /v1/runbooks/{runbook_id}/validate

# Agent Management
GET    /v1/agents                             # List all agents + FSM state
POST   /v1/agents/{agent_id}/halt
POST   /v1/agents/{agent_id}/resume
GET    /v1/agents/{agent_id}/state

# Policy Management
GET    /v1/policies/blast-radius-budget
PUT    /v1/policies/blast-radius-budget
GET    /v1/policies/triage-rules
PUT    /v1/policies/triage-rules             # Hot-reload OPA rules
```

### 12.2 CLI Tool: `cectl`

```bash
# cectl — Chaos Engineering Control Tool

cectl agent list --cluster prod-us-east-1
cectl agent halt recovery-governor --cluster prod-us-east-1
cectl experiment schedule --file experiment.yaml --dry-run
cectl experiment status exp-abc123
cectl experiment abort exp-abc123 --reason "unexpected SLO degradation"
cectl cluster freeze prod-us-east-1 --duration 2h --reason "deployment window"
cectl runbook list
cectl runbook validate --file runbook.yaml
cectl policy update triage-rules --file rules.rego
```

### 12.3 GitOps Integration

ExperimentSpec and RunbookTemplate CRDs are synced via Flux/ArgoCD:

```yaml
# ExperimentSpec CRD example
apiVersion: chaos.control-plane.io/v1alpha1
kind: ChaosExperiment
metadata:
  name: pod-kill-payment-service
  namespace: chaos-system
spec:
  cluster_id: prod-us-east-1
  namespace: payments
  hypothesis:
    description: "Payment service maintains <200ms p99 with 20% pod failure"
    slos:
      - metric: http_p99_latency_ms
        threshold: 200
        comparison: less_than
  fault_graph:
    nodes:
      - id: kill-pods
        fault_type: POD_KILL
        params:
          selector: app=payment-service
          kill_percentage: "0.2"
        duration: 5m
  blast_radius_budget:
    max_pod_percentage: 0.25
    allow_production: true
    min_error_budget_pct: 0.30
  monitoring_window: 10m
  rollback_policy: auto
```

---

## 13. Observability & Audit Trail

### 13.1 Metrics (Prometheus)

Each agent exposes a `/metrics` endpoint on port 9090:

```
# Agent FSM state transitions
ce_agent_state_transitions_total{agent_id, from_state, to_state, cluster_id}

# Action execution latency
ce_action_execution_duration_seconds{agent_id, action_type, cluster_id}

# Anomaly detection throughput
ce_anomaly_detection_events_total{detector_type, anomaly_type, severity}

# Recovery MTTR
ce_recovery_duration_seconds{cluster_id, runbook_id, outcome}

# Chaos experiment outcomes
ce_chaos_experiment_outcomes_total{cluster_id, outcome, fault_type}

# Agent-to-agent gRPC latency
ce_grpc_call_duration_seconds{caller, callee, method, status}

# Kafka consumer lag
ce_kafka_consumer_lag{topic, consumer_group, partition}
```

### 13.2 Distributed Tracing (OpenTelemetry)

Every event envelope carries a `trace_id`. Agents propagate `traceparent` headers across:
- gRPC metadata
- Kafka message headers
- Kubernetes API server calls (as `X-Request-ID`)

Traces are exported to **Jaeger** / **Tempo** via OTLP exporter.

### 13.3 Audit Log Schema

All agent decisions are written to Kafka topic `ce.audit.log` and ingested into **ClickHouse**:

```sql
CREATE TABLE audit_log (
    event_id        UUID,
    timestamp       DateTime64(9),
    agent_id        String,
    agent_type      Enum('ANOMALY_DETECTOR', 'TRIAGE', 'RECOVERY_GOVERNOR', ...),
    cluster_id      String,
    namespace       String,
    action_type     String,
    input_payload   String,  -- JSON
    output_payload  String,  -- JSON
    fsm_from_state  String,
    fsm_to_state    String,
    trace_id        String,
    outcome         Enum('SUCCESS', 'FAILURE', 'NOOP', 'HALTED'),
    duration_ms     UInt32
) ENGINE = MergeTree()
ORDER BY (cluster_id, timestamp)
PARTITION BY toYYYYMM(timestamp)
TTL timestamp + INTERVAL 90 DAY;
```

---

## 14. Security Model

### 14.1 Authentication & Authorization

| Surface | Mechanism |
|---------|-----------|
| Operator REST API | mTLS + JWT (OIDC provider) |
| Agent-to-Agent gRPC | mTLS with per-agent client certificates (SPIFFE/SPIRE) |
| Kafka | SASL/SCRAM-SHA-512 + TLS in-transit |
| Redis | TLS + AUTH token |
| K8s API access | ServiceAccount + RBAC (least-privilege) |
| etcd | mTLS (embedded with K8s or standalone) |

### 14.2 RBAC Roles

| Role | Permissions |
|------|-------------|
| `chaos-viewer` | Read experiments, reports, agent state |
| `chaos-operator` | Schedule/abort experiments, freeze clusters |
| `chaos-admin` | All above + update policies, manage runbooks |
| `system-agent` | Internal agent-to-agent service account |

### 14.3 Kubernetes RBAC (Recovery Actuator)

The Recovery Actuator ServiceAccount is bound to a minimal ClusterRole:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "patch", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "patch"]  # for cordon/uncordon
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
```

### 14.4 Secrets Management

All credentials are injected via Kubernetes Secrets (sourced from Vault via vault-agent-injector). No credentials in environment variables, ConfigMaps, or compiled binaries.

---

## 15. Deployment Architecture

### 15.1 Component Topology

```
┌─────────────────────────────────────────────────────────┐
│              chaos-system Namespace                      │
│                                                         │
│  StatefulSet: agent-coordinator (3 replicas — Raft)     │
│  Deployment:  recovery-governor  (2 replicas — 1 active)│
│  Deployment:  chaos-governor     (2 replicas — 1 active)│
│  Deployment:  anomaly-detector   (N replicas — sharded) │
│  Deployment:  triage-agent       (N replicas)           │
│  Deployment:  recovery-actuator  (N replicas)           │
│  Deployment:  chaos-injector     (N replicas)           │
│  Deployment:  hypothesis-validator (2 replicas)         │
│  Deployment:  rollback-agent     (2 replicas)           │
│  Deployment:  control-plane-api  (3 replicas + HPA)     │
│                                                         │
│  DaemonSet:   chaos-daemon (fault injection agents)     │
│                                                         │
│  StatefulSet: redis-cluster (6 nodes: 3 primary + 3 replica)│
│                                                         │
│  External: Kafka Cluster (MSK / Confluent / Self-hosted)│
└─────────────────────────────────────────────────────────┘
```

### 15.2 Resource Budgets

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|------------|-----------|---------------|--------------|
| anomaly-detector | 500m | 2000m | 256Mi | 1Gi |
| triage-agent | 200m | 500m | 128Mi | 512Mi |
| recovery-governor | 500m | 1000m | 256Mi | 512Mi |
| chaos-governor | 500m | 1000m | 256Mi | 512Mi |
| agent-coordinator | 500m | 1000m | 512Mi | 1Gi |
| recovery-actuator | 200m | 500m | 128Mi | 256Mi |
| chaos-injector | 200m | 500m | 128Mi | 256Mi |
| control-plane-api | 500m | 2000m | 256Mi | 512Mi |

### 15.3 Multi-Cluster Federation

Each target cluster runs lightweight **collection agents** (telemetry forwarder + chaos daemon). The control plane is centralized in a dedicated **management cluster**:

```
Management Cluster (Control Plane)
    ├── All governor and coordinator agents
    └── Kafka, Redis, ClickHouse

Target Cluster: prod-us-east-1
    ├── chaos-daemon (DaemonSet)
    ├── telemetry-forwarder (Deployment)
    └── recovery-actuator (Deployment — remote-mode)

Target Cluster: prod-eu-west-1
    └── (same as above)
```

---

## 16. Data Models & Protobuf Contracts

### 16.1 Core Protobuf Messages

```protobuf
syntax = "proto3";
package ce.v1;

// ── Anomaly Detection ──────────────────────────────────────────

message AnomalyEvent {
  string   event_id      = 1;
  string   cluster_id    = 2;
  string   namespace     = 3;
  string   anomaly_type  = 4;
  double   anomaly_score = 5;
  double   z_score       = 6;
  int64    detected_at   = 7;
  repeated Feature features = 8;
  string   trace_id      = 9;
}

message Feature {
  string name  = 1;
  double value = 2;
}

// ── Triage ─────────────────────────────────────────────────────

message TriageResult {
  string   triage_id          = 1;
  string   anomaly_event_id   = 2;
  string   severity           = 3;  // P1|P2|P3|P4
  string   remediation_class  = 4;
  string   runbook_id         = 5;
  repeated string affected_resources = 6;
  string   reasoning_rule_id  = 7;   // Which Rego rule fired
  int64    triaged_at         = 8;
}

// ── Recovery Plan ──────────────────────────────────────────────

message RecoveryPlan {
  string   plan_id       = 1;
  string   triage_id     = 2;
  string   cluster_id    = 3;
  string   runbook_id    = 4;
  repeated ActionNode actions = 5;
  repeated ActionEdge dag_edges = 6;
  int64    created_at    = 7;
}

message ActionNode {
  string   node_id      = 1;
  string   action_type  = 2;
  map<string,string> params = 3;
  string   idempotency_key  = 4;
  bool     has_rollback  = 5;
}

message ActionEdge {
  string from_node_id = 1;
  string to_node_id   = 2;
}

// ── Action Execution ───────────────────────────────────────────

message ActionRequest {
  string   request_id       = 1;
  string   idempotency_key  = 2;
  string   cluster_id       = 3;
  ActionNode action         = 4;
  bool     dry_run          = 5;
  string   trace_id         = 6;
}

message ActionResult {
  string   request_id  = 1;
  string   status      = 2;  // SUCCESS|FAILURE|NOOP|CIRCUIT_OPEN
  string   message     = 3;
  int64    executed_at = 4;
  int64    duration_ms = 5;
  bytes    output      = 6;
}
```

---

## 17. Performance Targets & SLOs

### 17.1 Latency Targets

| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| Telemetry ingestion → AnomalyEvent | 5ms | 15ms | 30ms |
| AnomalyEvent → TriageResult | 10ms | 25ms | 50ms |
| TriageResult → RecoveryPlan built | 50ms | 150ms | 500ms |
| gRPC ActuatorService.ExecuteAction (network only) | 2ms | 5ms | 10ms |
| Redis lease acquire | 1ms | 3ms | 8ms |
| Raft log commit (3-node) | 5ms | 15ms | 30ms |
| Full anomaly → action start | 200ms | 500ms | 1s |

### 17.2 Throughput Targets

| Metric | Target |
|--------|--------|
| Telemetry events ingested/sec | 50,000 |
| Anomaly evaluations/sec (Z-Score) | 10,000 |
| Concurrent recovery plans | 100 (across all clusters) |
| Concurrent chaos experiments | 20 |
| Audit log writes/sec | 5,000 |

### 17.3 Availability SLOs

| Component | SLO |
|-----------|-----|
| Control Plane API | 99.9% |
| Recovery Governor (per cluster) | 99.95% |
| Chaos Governor | 99.9% |
| Agent Coordinator (Raft cluster) | 99.99% |
| Anomaly Detector | 99.9% |

---

## 18. Failure Modes & Mitigations

| Failure Scenario | Detection | Mitigation |
|-----------------|-----------|------------|
| Recovery Governor crashes mid-execution | Kafka heartbeat timeout + Raft leader re-election | Standby governor takes over; replays DAG from last persisted node in Redis |
| Redis cluster partition | Redis Sentinel / Cluster redirect | Agents fall back to optimistic execution with full logging; re-sync post-partition |
| Kafka consumer lag spike | `ce_kafka_consumer_lag` metric alert | Scale out consumer group; circuit breaker on upstream Prometheus scrape |
| Actuator circuit breaker open (K8s API down) | Circuit breaker state metric | Queue actions locally; retry with backoff; alert operator |
| Chaos experiment SLO breach undetected | Hypothesis Validator timeout | Monitoring loop has max TTL; hard abort triggered at `experiment.timeout` |
| OPA rule evaluation timeout | `rego.eval` deadline context | Default to highest severity + HALT action on timeout |
| Raft split-brain (net partition) | Quorum loss detection | All governors demoted to STANDBY; recovery blocked until quorum restored |
| Action idempotency key collision | Redis duplicate detection | Return cached result, log warning, increment collision counter metric |
| Runbook template not found for anomaly | Triage result with no runbook_id | Escalate to `HUMAN_ESCALATION` action; PagerDuty alert; FSM → HALTED |
| Anomaly detector model stale | Model version TTL check | Alert + flag; continue with Z-Score only until model refreshed |

---

## 19. Technology Stack Summary

| Layer | Technology | Version | Justification |
|-------|-----------|---------|---------------|
| Language | Go | 1.22+ | Goroutine model, compile-time safety, sub-ms latency |
| Message Bus | Apache Kafka | 3.7+ | Durable, partitioned, exactly-once semantics |
| Message Bus (alt) | NATS JetStream | 2.10+ | Lower latency for co-located agents |
| RPC | gRPC + Protobuf | grpc-go 1.63 | Binary efficiency, streaming, deadline propagation |
| State Store | Redis Cluster | 7.2+ | Sub-ms reads, SETNX leases, TTL expiry |
| Consensus | hashicorp/raft | 1.6 | Embedded Go Raft; no ZooKeeper dependency |
| Rule Engine | OPA/Rego | 0.63+ | Declarative, testable, hot-reloadable policies |
| Matrix Math | gonum | 0.15 | Pure Go; Isolation Forest training/inference |
| Metrics | Prometheus | 2.50+ | Industry standard; Go client first-class |
| Tracing | OpenTelemetry Go | 1.26 | Vendor-neutral trace propagation |
| Audit Store | ClickHouse | 24.x | Columnar; 10M+ row/s insert; TTL partitioning |
| Config Store | etcd / K8s ConfigMap | 3.5+ | Runbook and policy templates |
| Container Orchestration | Kubernetes | 1.29+ | Target runtime; operator CRDs |
| GitOps | Flux v2 / ArgoCD | — | ExperimentSpec and policy sync |
| Secrets | HashiCorp Vault | 1.16+ | Dynamic credentials; vault-agent-injector |
| Service Identity | SPIFFE/SPIRE | 1.9 | mTLS cert rotation for agent-to-agent |
| Fault Injection SDK | Chaos Mesh / LitmusChaos | — | DaemonSet-based fault execution |

---

## 20. Roadmap & Phasing

### Phase 1 — Foundation (Months 1–3)

- [ ] Core agent scaffolding: FSM library, event envelope, Kafka consumers
- [ ] Agent Coordinator with embedded Raft
- [ ] Anomaly Detector: Z-Score pipeline + Kafka integration
- [ ] Triage Agent: OPA integration + 10 baseline rules
- [ ] Recovery Governor: FSM + basic DAG executor
- [ ] Recovery Actuator: K8s client + 3 runbooks (OOMKill, CrashLoop, NodeNotReady)
- [ ] Control Plane API: core endpoints + CLI skeleton
- [ ] Redis state management + lease protocol
- [ ] Basic Prometheus metrics + structured JSON logging

### Phase 2 — Chaos Engineering Core (Months 4–6)

- [ ] Chaos Governor: full FSM + BlastRadiusBudget enforcement
- [ ] Chaos Injector: DAG executor + 5 fault types (PodKill, NetworkPartition, CPUHog, MemoryPressure, LatencyInjection)
- [ ] Hypothesis Validator: Welch's t-test engine + SLO result reporting
- [ ] Rollback Agent: DAG reverse traversal
- [ ] ExperimentSpec CRD + GitOps sync
- [ ] Full audit log pipeline → ClickHouse
- [ ] OpenTelemetry distributed tracing

### Phase 3 — Production Hardening (Months 7–9)

- [ ] Isolation Forest model training pipeline + hot-swap
- [ ] Multi-cluster federation architecture
- [ ] Full RBAC + SPIFFE/SPIRE mTLS
- [ ] Vault secrets integration
- [ ] Load testing: 10K events/sec, 100 concurrent recovery plans
- [ ] Runbook library: 15+ runbooks
- [ ] Operator dashboard (read-only React UI)
- [ ] Chaos experiment scheduling (cron-based)

### Phase 4 — Advanced Capabilities (Months 10–12)

- [ ] Adaptive blast radius budgets (SLO error budget integration)
- [ ] Experiment templates library (GameDays, DR drills)
- [ ] Automated GameDay report generation
- [ ] Cross-cluster chaos (multi-region partition experiments)
- [ ] Predictive anomaly pre-warnings (forecast-based Z-Score on trends)
- [ ] `cectl` v2 with interactive TUI

---

## Appendix A: Key Design Decisions Log

| Decision | Alternatives Considered | Rationale |
|----------|------------------------|-----------|
| OPA/Rego for rule engine | Custom DSL, Drools | OPA is Go-native, testable, hot-reloadable; Drools is JVM |
| Kafka over NATS as primary bus | RabbitMQ, Pulsar | Kafka provides replay, consumer group offsets, and log compaction needed for event sourcing |
| hashicorp/raft over etcd for coordinator | etcd client, ZooKeeper | Embedded in-process; no external dependency for single cluster; etcd used only for config |
| Redis SETNX leases over Raft log | Raft-based locks | Redis locks are simpler and faster for short-lived leases; Raft reserved for membership/config |
| Welch's t-test over Mann-Whitney | Mann-Whitney U, KS-test | Welch's is sufficient for normal-ish SLO distributions; easily implementable in Go without external libs |
| DaemonSet-based fault injection | In-cluster Job, Sidecar | DaemonSet allows node-level and network-level faults that container-scoped approaches cannot reach |

---

*End of HLD Document — Automated DevOps & Chaos Engineering Control Plane v1.0.0*
