# EAAGF Specification — Observability and Audit Trail Standard

**Document ID:** EAAGF-SPEC-05  
**Version:** 1.0.0  
**Status:** Draft  
**Last Updated:** 2025-07-14  
**Owner:** AI Governance Team

---

## 1. Purpose

This document defines the normative standard for observability and audit trail within the Enterprise AI Agent Governance Framework (EAAGF). It specifies how the Telemetry_Emitter produces standardized audit events, how the Audit_Log guarantees immutability and retention, and how the Governance_Controller exposes audit data for querying, SIEM integration, and compliance reporting.

Every governance decision — PERMIT, DENY, or GATE — MUST produce a standardized audit event. The audit trail is the authoritative record of all agent actions, policy decisions, and oversight events across the enterprise. It serves as the foundation for incident investigation, regulatory compliance demonstration, and agent behavior analysis.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. Scope

This standard applies to:

- All AI agents deployed on any enterprise-supported platform (Databricks, Salesforce AgentForce, Snowflake Cortex, Microsoft Copilot Studio, AWS Bedrock, Azure AI Foundry, GCP Vertex AI)
- The Telemetry_Emitter component and its event emission interfaces
- The Audit_Log storage layer and its immutability guarantees
- The Governance_Controller component responsible for audit event query and SIEM integration
- All teams that develop, deploy, or operate AI agents within the enterprise

For related standards, see:

| Related Domain | Document |
|---|---|
| Agent Identity | [02 — Agent Identity Standard](./02-agent-identity-standard.md) |
| Risk Classification | [03 — Risk Classification Standard](./03-risk-classification-standard.md) |
| Authorization | [04 — Authorization Standard](./04-authorization-standard.md) |
| Human Oversight | [06 — Human Oversight Standard](./06-human-oversight-standard.md) |
| Security | [09 — Security Standard](./09-security-standard.md) |
| Compliance | [10 — Compliance Standard](./10-compliance-standard.md) |

---

## 3. Audit Event Emission

### 3.1 Mandatory Event Emission

The Telemetry_Emitter SHALL emit an audit event for every agent action. No agent action — whether permitted, denied, or gated — SHALL proceed without a corresponding audit event being generated.

**Normative rules:**

1. The Telemetry_Emitter SHALL emit an audit event for every agent action, including the following required fields:
   - **Agent ID** (`eaagf.agent.id`) — The agent's UUID v4 identifier.
   - **Action Type** (`eaagf.action.type`) — The category of action performed (e.g., `TOOL_CALL`, `DATA_ACCESS`, `AGENT_DELEGATION`, `EXTERNAL_CONNECTION`).
   - **Target Resource** (`eaagf.action.target`) — The URI of the resource the action targets.
   - **Input Summary** (`eaagf.action.input_summary`) — A structured summary of the action's input parameters. For data privacy, this field SHALL be redacted or masked according to the data classification rules in [08 — Data Governance Standard](./08-data-governance-standard.md).
   - **Output Summary** (`eaagf.action.output_summary`) — A structured summary of the action's result. Subject to the same redaction rules as the input summary.
   - **Timestamp** (`timestamp`) — The UTC timestamp of the event in ISO 8601 format.
   - **Risk Tier** (`eaagf.agent.risk_tier`) — The agent's assigned Risk Tier (T1, T2, T3, or T4).
   - **Platform** (`eaagf.agent.platform`) — The platform the agent is deployed on.
   - **Outcome** (`eaagf.action.outcome`) — The result of the governance decision: `PERMITTED`, `DENIED`, `GATED`, or `BLOCKED`.
2. The Telemetry_Emitter SHALL emit events synchronously with the governance decision. The event MUST be generated before the action result is returned to the agent.
3. IF the Telemetry_Emitter fails to emit an event, the failure SHALL NOT silently suppress the event. The buffering rules in Section 8 apply.

> **Validates: Requirement 4.1** — THE Telemetry_Emitter SHALL emit an audit event for every agent action, including: agent ID, action type, target resource, input summary, output summary, timestamp (UTC), Risk_Tier, platform, and outcome (success/failure/blocked).

---

## 4. OTLP Event Format

### 4.1 OpenTelemetry Compatibility

The Telemetry_Emitter SHALL produce telemetry in OpenTelemetry-compatible format (OTLP) so that events can be ingested by any OTEL-compatible backend.

**Normative rules:**

1. All audit events SHALL be emitted as OTLP spans conforming to the [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/).
2. EAAGF-specific data SHALL be encoded as span attributes using the `eaagf.*` namespace prefix.
3. The Telemetry_Emitter SHALL support export via the following OTLP transport protocols:
   - OTLP/gRPC (REQUIRED)
   - OTLP/HTTP (REQUIRED)
   - OTLP/JSON (RECOMMENDED)
4. The Telemetry_Emitter SHALL be compatible with the following observability backends:
   - Datadog
   - Splunk
   - Azure Monitor
   - AWS CloudWatch
   - Any OTEL-compatible collector or backend
5. Conforming implementations MAY support additional export formats beyond OTLP, but OTLP support is REQUIRED.
6. The OTLP exporter SHALL be configurable via standard OpenTelemetry SDK environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, etc.).

> **Validates: Requirement 4.2** — THE Telemetry_Emitter SHALL produce telemetry in OpenTelemetry-compatible format (OTLP) so that events can be ingested by any OTEL-compatible backend (Datadog, Splunk, Azure Monitor, AWS CloudWatch).

---

## 5. Audit Event Schema

### 5.1 Complete Audit Event Schema (OTLP Span Attributes)

The following JSON schema defines the complete set of OTLP span attributes that MUST be present on every audit event. All attributes use the `eaagf.*` namespace prefix to avoid collisions with other OpenTelemetry instrumentation.

```json
{
  "eaagf.agent.id": {
    "type": "string",
    "format": "uuid-v4",
    "description": "The agent's globally unique identifier assigned at registration.",
    "required": true
  },
  "eaagf.agent.risk_tier": {
    "type": "string",
    "enum": ["T1", "T2", "T3", "T4"],
    "description": "The agent's assigned Risk Tier at the time of the event.",
    "required": true
  },
  "eaagf.agent.platform": {
    "type": "string",
    "enum": ["DATABRICKS", "SALESFORCE", "SNOWFLAKE", "COPILOT_STUDIO", "AWS", "AZURE", "GCP"],
    "description": "The platform the agent is deployed on.",
    "required": true
  },
  "eaagf.action.type": {
    "type": "string",
    "enum": ["TOOL_CALL", "DATA_ACCESS", "AGENT_DELEGATION", "EXTERNAL_CONNECTION"],
    "description": "The category of action performed by the agent.",
    "required": true
  },
  "eaagf.action.target": {
    "type": "string",
    "description": "The URI of the target resource (e.g., 'snowflake://db/schema/table', 'salesforce://sobject/Account').",
    "required": true
  },
  "eaagf.action.outcome": {
    "type": "string",
    "enum": ["PERMITTED", "DENIED", "GATED", "BLOCKED"],
    "description": "The result of the governance decision for this action.",
    "required": true
  },
  "eaagf.action.reason_code": {
    "type": "string",
    "description": "The reason code when the outcome is DENIED or BLOCKED (e.g., 'PERMISSION_NOT_DECLARED', 'COMPARTMENT_VIOLATION'). NULL for PERMITTED outcomes.",
    "required": false
  },
  "eaagf.action.input_summary": {
    "type": "string",
    "description": "A structured summary of the action's input parameters. Subject to PII redaction rules.",
    "required": true
  },
  "eaagf.action.output_summary": {
    "type": "string",
    "description": "A structured summary of the action's result. Subject to PII redaction rules.",
    "required": true
  },
  "eaagf.task.id": {
    "type": "string",
    "format": "uuid-v4",
    "description": "The unique identifier of the task execution context.",
    "required": true
  },
  "eaagf.task.correlation_id": {
    "type": "string",
    "format": "uuid-v4",
    "description": "A correlation ID shared by all events within a single agent task execution. Used for end-to-end trace reconstruction.",
    "required": true
  },
  "eaagf.gate.id": {
    "type": "string",
    "format": "uuid-v4",
    "description": "The unique identifier of the Human_Oversight_Gate instance, if the action was gated.",
    "required": false
  },
  "eaagf.gate.approver": {
    "type": "string",
    "description": "The identity of the human approver, if the action was gated.",
    "required": false
  },
  "eaagf.gate.decision": {
    "type": "string",
    "enum": ["APPROVED", "REJECTED"],
    "description": "The approval decision, if the action was gated and resolved.",
    "required": false
  },
  "eaagf.gate.trigger_time": {
    "type": "string",
    "format": "ISO8601",
    "description": "The UTC timestamp when the Human_Oversight_Gate was triggered.",
    "required": false
  },
  "eaagf.gate.decision_time": {
    "type": "string",
    "format": "ISO8601",
    "description": "The UTC timestamp when the approver made the approval/rejection decision.",
    "required": false
  },
  "eaagf.data.classification": {
    "type": "string",
    "enum": ["PUBLIC", "INTERNAL", "CONFIDENTIAL", "RESTRICTED"],
    "description": "The data classification label of the target resource.",
    "required": false
  },
  "eaagf.security.prompt_injection_score": {
    "type": "float",
    "description": "The prompt injection detection confidence score for the input, if evaluated.",
    "required": false
  },
  "eaagf.compliance.eu_ai_act_applicable": {
    "type": "boolean",
    "description": "Whether the agent is flagged as EU_AI_ACT_HIGH_RISK.",
    "required": false
  },
  "eaagf.reasoning.chain_summary": {
    "type": "string",
    "description": "A structured summary of the agent's reasoning chain (chain-of-thought), if available. See Section 10.",
    "required": false
  },
  "timestamp": {
    "type": "string",
    "format": "ISO8601",
    "description": "The UTC timestamp of the event in ISO 8601 format.",
    "required": true
  }
}
```

### 5.2 Attribute Requirements by Outcome

The following table specifies which attributes are REQUIRED versus OPTIONAL based on the event outcome:

| Attribute | PERMITTED | DENIED / BLOCKED | GATED |
|---|---|---|---|
| `eaagf.agent.id` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.agent.risk_tier` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.agent.platform` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.action.type` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.action.target` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.action.outcome` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.action.reason_code` | OPTIONAL | REQUIRED | OPTIONAL |
| `eaagf.action.input_summary` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.action.output_summary` | REQUIRED | OPTIONAL | OPTIONAL |
| `eaagf.task.id` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.task.correlation_id` | REQUIRED | REQUIRED | REQUIRED |
| `eaagf.gate.id` | N/A | N/A | REQUIRED |
| `eaagf.gate.approver` | N/A | N/A | REQUIRED (on resolution) |
| `eaagf.gate.decision` | N/A | N/A | REQUIRED (on resolution) |
| `eaagf.gate.trigger_time` | N/A | N/A | REQUIRED |
| `eaagf.gate.decision_time` | N/A | N/A | REQUIRED (on resolution) |
| `eaagf.data.classification` | RECOMMENDED | RECOMMENDED | RECOMMENDED |
| `eaagf.reasoning.chain_summary` | RECOMMENDED | OPTIONAL | OPTIONAL |
| `timestamp` | REQUIRED | REQUIRED | REQUIRED |

---

## 6. Blocked Event Emission SLA

### 6.1 500-Millisecond Emission Requirement

WHEN an agent action is blocked by the Policy_Engine, the Telemetry_Emitter SHALL emit a BLOCKED event within 500 milliseconds of the block decision.

**Normative rules:**

1. The 500-millisecond window begins at the timestamp when the Policy_Engine returns a DENY decision and ends when the BLOCKED audit event is durably committed to the telemetry pipeline.
2. "Durably committed" means the event has been accepted by the OTLP exporter or written to the local write-ahead log (WAL), whichever occurs first.
3. The BLOCKED event SHALL include:
   - All REQUIRED attributes defined in Section 5.2 for the DENIED/BLOCKED outcome.
   - The `eaagf.action.reason_code` attribute with the specific denial reason (e.g., `PERMISSION_NOT_DECLARED`, `COMPARTMENT_VIOLATION`, `RATE_LIMIT_EXCEEDED`).
4. IF the Telemetry_Emitter cannot emit the BLOCKED event within 500 milliseconds due to backend unavailability, the event SHALL be written to the local WAL (see Section 8) and the 500ms SLA is considered met once the WAL write completes.
5. Conforming implementations SHALL monitor and report the p99 latency of BLOCKED event emission. The p99 latency SHOULD remain below 500 milliseconds under normal operating conditions.
6. The 500ms SLA applies to all BLOCKED events regardless of the agent's Risk Tier or platform.

> **Validates: Requirement 4.3** — WHEN an agent action is blocked by the Policy_Engine, THE Telemetry_Emitter SHALL emit a BLOCKED event within 500 milliseconds of the block decision.

---

## 7. Audit Log Immutability

### 7.1 Immutability Guarantee

The Audit_Log SHALL be immutable — once written, audit records SHALL NOT be modifiable or deletable by any agent, platform adapter, or application team.

**Normative rules:**

1. Once an audit event is written to the Audit_Log, no entity — including agents, Platform Adapters, application teams, or automated processes — SHALL be able to modify, overwrite, or delete the record.
2. The Audit_Log storage layer SHALL enforce write-once semantics. Conforming implementations MUST use storage mechanisms that prevent in-place modification (e.g., append-only logs, immutable object storage, WORM-compliant storage).
3. The AI Governance Team SHALL have read-only access to the Audit_Log. Even the AI Governance Team SHALL NOT have the ability to modify or delete audit records.
4. The Audit_Log SHALL implement tamper-evidence mechanisms. Conforming implementations SHOULD use one or more of the following:
   - Cryptographic hash chains linking sequential audit records.
   - Digital signatures on audit event batches.
   - Merkle tree verification for audit log integrity.
5. IF a tamper attempt is detected (e.g., hash chain verification failure), the Audit_Log SHALL emit an `AUDIT_INTEGRITY_VIOLATION` security alert and notify the AI Governance Team immediately.
6. Audit records MAY be archived to cold storage after a configurable period (default: 1 year), but archived records SHALL retain the same immutability guarantees as active records.
7. The only permitted operation that removes audit data is the data subject deletion process defined in [08 — Data Governance Standard](./08-data-governance-standard.md), which applies PII redaction to specific fields while preserving the audit record structure and integrity chain.

> **Validates: Requirement 4.4** — THE Audit_Log SHALL be immutable — once written, audit records SHALL NOT be modifiable or deletable by any agent, platform adapter, or application team.

---

## 8. Retention and Archival

### 8.1 Seven-Year Retention Requirement

The Audit_Log SHALL retain all records for a minimum of 7 years to satisfy EU AI Act and enterprise compliance requirements.

**Normative rules:**

1. ALL audit events — regardless of outcome, Risk Tier, platform, or action type — SHALL be retained for a minimum of 7 years from the event timestamp.
2. The 7-year retention period is a minimum. Conforming implementations MAY retain records for longer periods based on enterprise policy.
3. Retained records SHALL remain queryable via the Query API (Section 11) for the full retention period. Archived records MAY have higher query latency than active records, but SHALL be retrievable within 24 hours of a query request.
4. The retention policy SHALL apply uniformly across all platforms. Platform-specific storage limitations SHALL NOT reduce the retention period below 7 years.
5. Conforming implementations SHALL implement a tiered storage strategy:
   - **Hot storage** (0–90 days): Low-latency query access. Records available within seconds.
   - **Warm storage** (91 days–1 year): Medium-latency query access. Records available within minutes.
   - **Cold storage** (1–7 years): High-latency query access. Records available within 24 hours.
6. Storage tier transitions SHALL NOT affect record immutability or integrity. Records SHALL retain their cryptographic integrity verification (hash chains, signatures) across all storage tiers.
7. At the end of the 7-year retention period, records MAY be purged. Purge operations SHALL be logged and SHALL require AI Governance Team authorization.

> **Validates: Requirement 4.5** — THE Audit_Log SHALL retain all records for a minimum of 7 years to satisfy EU AI Act and enterprise compliance requirements.

---

## 9. Human Oversight Gate Event Emission

### 9.1 Gate Event Requirements

WHEN a Human_Oversight_Gate is triggered, the Telemetry_Emitter SHALL emit events capturing the full gate lifecycle.

**Normative rules:**

1. The Telemetry_Emitter SHALL emit a `GATE_TRIGGERED` event when a Human_Oversight_Gate is activated. This event SHALL include:
   - `eaagf.gate.id` — The unique identifier of the gate instance.
   - `eaagf.gate.trigger_time` — The UTC timestamp when the gate was triggered.
   - `eaagf.agent.id` — The agent whose action triggered the gate.
   - `eaagf.action.type` — The action type that triggered the gate.
   - `eaagf.action.target` — The target resource of the gated action.
   - `eaagf.agent.risk_tier` — The agent's Risk Tier.
2. The Telemetry_Emitter SHALL emit a `GATE_RESOLVED` event when the gate is resolved (approved or rejected). This event SHALL include:
   - `eaagf.gate.id` — The gate instance identifier (matching the `GATE_TRIGGERED` event).
   - `eaagf.gate.approver` — The identity of the human who made the decision.
   - `eaagf.gate.decision` — `APPROVED` or `REJECTED`.
   - `eaagf.gate.decision_time` — The UTC timestamp of the decision.
   - `eaagf.gate.trigger_time` — The original trigger timestamp (for duration calculation).
3. IF the gate times out and escalates, the Telemetry_Emitter SHALL emit a `GATE_TIMEOUT_ESCALATION` event containing the gate ID, the original approver identity, the escalation target, and the timeout duration.
4. The `GATE_TRIGGERED` and `GATE_RESOLVED` events SHALL share the same `eaagf.task.correlation_id` as the original action event, enabling end-to-end trace reconstruction across the gate lifecycle.
5. Gate events SHALL be emitted in addition to (not instead of) the standard action audit event. A gated action produces a minimum of two audit events: the action event (with outcome `GATED`) and the gate resolution event.

> **Validates: Requirement 4.6** — WHEN a Human_Oversight_Gate is triggered, THE Telemetry_Emitter SHALL emit events capturing: gate trigger time, approver identity, approval/rejection decision, and decision timestamp.

---

## 10. Correlation ID Requirements

### 10.1 End-to-End Trace Correlation

The Telemetry_Emitter SHALL include a correlation ID in all events emitted within a single agent task execution, enabling end-to-end trace reconstruction.

**Normative rules:**

1. The correlation ID SHALL be a UUID v4 value assigned at the start of each agent task execution.
2. ALL audit events emitted during a single task execution SHALL carry the same `eaagf.task.correlation_id` value. This includes:
   - Action events (PERMITTED, DENIED, BLOCKED, GATED).
   - Gate lifecycle events (GATE_TRIGGERED, GATE_RESOLVED, GATE_TIMEOUT_ESCALATION).
   - Credential events (SESSION_CREDENTIAL_ISSUED, SESSION_CREDENTIAL_REVOKED).
   - Security events (PROMPT_INJECTION_DETECTED, OUTPUT_VALIDATION_FAILURE).
   - Reasoning chain events (see Section 12).
3. WHEN an agent delegates a sub-task to another agent via A2A protocol, the delegated task SHALL receive a new `eaagf.task.id` but SHALL propagate the parent task's `eaagf.task.correlation_id`. This enables trace reconstruction across agent delegation chains.
4. The correlation ID SHALL be propagated through all EAAGF components: Platform Adapter → Governance Controller → Policy Engine → Telemetry Emitter → Human Oversight Gate.
5. The correlation ID SHALL be included in the OTLP span context as a span attribute, and SHOULD also be set as the OpenTelemetry `trace_id` where the OTEL trace context is available.
6. Conforming implementations SHALL support querying all events by correlation ID via the Query API (Section 11).

> **Validates: Requirement 4.7** — THE Telemetry_Emitter SHALL include a correlation ID in all events emitted within a single agent task execution, enabling end-to-end trace reconstruction.

---

## 11. Query API Specification

### 11.1 Audit Event Query API

The Governance_Controller SHALL provide a query API that allows authorized users to retrieve audit events filtered by multiple dimensions.

**Normative rules:**

1. The Query API SHALL support filtering by the following dimensions:
   - **Agent ID** (`agent_id`) — Filter events for a specific agent.
   - **Time Range** (`start_time`, `end_time`) — Filter events within a UTC time window. Both bounds are inclusive.
   - **Action Type** (`action_type`) — Filter by action category (TOOL_CALL, DATA_ACCESS, AGENT_DELEGATION, EXTERNAL_CONNECTION).
   - **Risk Tier** (`risk_tier`) — Filter by agent Risk Tier (T1, T2, T3, T4).
   - **Outcome** (`outcome`) — Filter by governance decision outcome (PERMITTED, DENIED, GATED, BLOCKED).
   - **Correlation ID** (`correlation_id`) — Retrieve all events within a single task execution trace.
   - **Platform** (`platform`) — Filter by agent platform.
   - **Reason Code** (`reason_code`) — Filter by denial reason code.
2. The Query API SHALL support combining multiple filters using AND logic. All specified filters MUST match for an event to be included in the result set.
3. The Query API SHALL support pagination for large result sets. The default page size SHALL be 100 events, with a configurable maximum of 1,000 events per page.
4. The Query API SHALL return results in JSON format, with each event containing all attributes defined in the Audit Event Schema (Section 5).
5. Access to the Query API SHALL be restricted to authorized users. The following roles SHALL be supported:
   - **AI Governance Team** — Full query access across all agents, platforms, and time ranges.
   - **Team Leads** — Query access scoped to agents owned by their team.
   - **External Auditors** — Read-only query access as defined in [10 — Compliance Standard](./10-compliance-standard.md).
6. The Query API SHALL enforce rate limiting to prevent abuse. The default rate limit is 100 queries per minute per authenticated user.
7. Query results from hot storage (0–90 days) SHALL be returned within 5 seconds. Query results from warm storage (91 days–1 year) SHALL be returned within 60 seconds. Query results from cold storage (1–7 years) SHALL be initiated within 60 seconds and delivered within 24 hours.

### 11.2 Query API Request Schema

```json
{
  "filters": {
    "agent_id": "uuid-v4 (optional)",
    "start_time": "ISO8601 UTC (required)",
    "end_time": "ISO8601 UTC (required)",
    "action_type": "TOOL_CALL | DATA_ACCESS | AGENT_DELEGATION | EXTERNAL_CONNECTION (optional)",
    "risk_tier": "T1 | T2 | T3 | T4 (optional)",
    "outcome": "PERMITTED | DENIED | GATED | BLOCKED (optional)",
    "correlation_id": "uuid-v4 (optional)",
    "platform": "DATABRICKS | SALESFORCE | SNOWFLAKE | COPILOT_STUDIO | AWS | AZURE | GCP (optional)",
    "reason_code": "string (optional)"
  },
  "pagination": {
    "page_size": "integer (default: 100, max: 1000)",
    "page_token": "string (optional, for subsequent pages)"
  },
  "sort": {
    "field": "timestamp (default)",
    "order": "ASC | DESC (default: DESC)"
  }
}
```

### 11.3 Query API Response Schema

```json
{
  "events": [
    {
      "eaagf.agent.id": "uuid",
      "eaagf.agent.risk_tier": "T1|T2|T3|T4",
      "eaagf.agent.platform": "string",
      "eaagf.action.type": "string",
      "eaagf.action.target": "string",
      "eaagf.action.outcome": "string",
      "eaagf.action.reason_code": "string|null",
      "eaagf.action.input_summary": "string",
      "eaagf.action.output_summary": "string|null",
      "eaagf.task.id": "uuid",
      "eaagf.task.correlation_id": "uuid",
      "eaagf.gate.id": "uuid|null",
      "eaagf.gate.approver": "string|null",
      "eaagf.gate.decision": "string|null",
      "eaagf.data.classification": "string|null",
      "eaagf.reasoning.chain_summary": "string|null",
      "timestamp": "ISO8601 UTC"
    }
  ],
  "pagination": {
    "total_count": "integer",
    "page_token": "string|null",
    "has_more": "boolean"
  }
}
```

> **Validates: Requirement 4.8** — THE Governance_Controller SHALL provide a query API that allows authorized users to retrieve audit events filtered by agent ID, time range, action type, Risk_Tier, and outcome.

---

## 12. Telemetry Buffering and Resilience

### 12.1 Write-Ahead Log (WAL) Buffering

IF the telemetry backend is unavailable, THEN the Governance_Controller SHALL buffer audit events locally and replay them when connectivity is restored, without dropping events.

**Normative rules:**

1. The Governance_Controller SHALL maintain a local write-ahead log (WAL) that buffers audit events when the primary telemetry backend is unreachable.
2. The WAL SHALL have a minimum capacity of 10,000 events. Conforming implementations MAY support a larger WAL capacity based on available local storage.
3. Events written to the WAL SHALL be persisted to durable local storage (disk). In-memory-only buffering is NOT sufficient — events MUST survive process restarts.
4. WHEN the telemetry backend becomes available again, the Governance_Controller SHALL replay all buffered events from the WAL in chronological order (oldest first).
5. Replayed events SHALL retain their original timestamps and all original attribute values. The replay process MUST NOT modify event content.
6. The Governance_Controller SHALL add a `eaagf.telemetry.replayed` boolean attribute (set to `true`) to replayed events so that downstream consumers can distinguish replayed events from real-time events.
7. IF the WAL reaches its capacity limit (10,000 events) while the backend remains unavailable, the Governance_Controller SHALL:
   a. Emit a `TELEMETRY_WAL_FULL` alert to the AI Governance Team.
   b. Continue accepting new events by evicting the oldest events from the WAL using a FIFO strategy.
   c. Log the count of evicted events for post-incident reconciliation.
8. The Governance_Controller SHALL NOT block or delay agent action processing due to telemetry backend unavailability. The WAL write is the fallback path, and it SHALL complete within the same latency budget as a normal telemetry emission.
9. The WAL SHALL implement the same tamper-evidence mechanisms as the primary Audit_Log (hash chains or equivalent) to ensure buffered events cannot be modified before replay.

> **Validates: Requirement 4.9** — IF the telemetry backend is unavailable, THEN THE Governance_Controller SHALL buffer audit events locally and replay them when connectivity is restored, without dropping events.

---

## 13. SIEM Integration

### 13.1 Message Queue Publishing

The Telemetry_Emitter SHALL support SIEM integration by publishing audit events to a configurable webhook or message queue.

**Normative rules:**

1. The Telemetry_Emitter SHALL support publishing audit events to the following message queue systems:
   - **Apache Kafka** — Events SHALL be published to a configurable Kafka topic.
   - **Azure Event Hub** — Events SHALL be published to a configurable Event Hub namespace and event hub.
   - **AWS Kinesis** — Events SHALL be published to a configurable Kinesis data stream.
2. The Telemetry_Emitter SHALL additionally support publishing to a configurable webhook endpoint (HTTPS POST) for integration with SIEM platforms that do not consume from message queues directly.
3. Events published to message queues SHALL use the same JSON serialization as the Audit Event Schema defined in Section 5. The message key SHALL be the `eaagf.agent.id` to enable partition-based ordering per agent.
4. The Telemetry_Emitter SHALL support configuring multiple simultaneous SIEM destinations. For example, events MAY be published to both a Kafka topic and an Azure Event Hub concurrently.
5. SIEM publishing SHALL be asynchronous and SHALL NOT block the governance decision path. If a SIEM destination is temporarily unavailable, the WAL buffering rules in Section 12 apply.
6. The Telemetry_Emitter SHALL support configurable event filtering for SIEM destinations. Teams MAY configure filters to publish only specific event types (e.g., only BLOCKED and GATED events) to a given SIEM destination.
7. Authentication to message queue systems SHALL use the following mechanisms:
   - **Kafka**: SASL/SCRAM or mTLS.
   - **Azure Event Hub**: Azure AD managed identity or SAS token.
   - **AWS Kinesis**: IAM role-based authentication.
   - **Webhook**: OAuth 2.0 bearer token or mutual TLS.
8. The Telemetry_Emitter SHALL guarantee at-least-once delivery to SIEM destinations. Duplicate events MAY occur during replay scenarios; downstream consumers SHOULD implement idempotent processing using the event's `eaagf.task.id` and `timestamp` as a deduplication key.

### 13.2 SIEM Configuration Schema

```json
{
  "siem_destinations": [
    {
      "type": "KAFKA",
      "config": {
        "bootstrap_servers": "kafka-broker-1:9092,kafka-broker-2:9092",
        "topic": "eaagf-audit-events",
        "authentication": "SASL_SCRAM | MTLS",
        "compression": "gzip | snappy | lz4 | none"
      },
      "filter": {
        "outcomes": ["DENIED", "BLOCKED", "GATED"],
        "risk_tiers": ["T3", "T4"]
      }
    },
    {
      "type": "AZURE_EVENT_HUB",
      "config": {
        "namespace": "eaagf-telemetry",
        "event_hub": "audit-events",
        "authentication": "MANAGED_IDENTITY | SAS_TOKEN"
      },
      "filter": null
    },
    {
      "type": "AWS_KINESIS",
      "config": {
        "stream_name": "eaagf-audit-events",
        "region": "us-east-1",
        "authentication": "IAM_ROLE"
      },
      "filter": null
    },
    {
      "type": "WEBHOOK",
      "config": {
        "endpoint": "https://siem.internal.company.com/api/v1/events",
        "authentication": "OAUTH2 | MTLS",
        "retry_policy": {
          "max_retries": 3,
          "backoff_seconds": [1, 5, 30]
        }
      },
      "filter": {
        "outcomes": ["BLOCKED"],
        "reason_codes": ["PROMPT_INJECTION_DETECTED", "DATA_EXFILTRATION_ATTEMPT"]
      }
    }
  ]
}
```

> **Validates: Requirement 4.10** — THE Telemetry_Emitter SHALL support SIEM integration by publishing audit events to a configurable webhook or message queue (Kafka, Azure Event Hub, AWS Kinesis).

---

## 14. Reasoning Chain Logging

### 14.1 Structured Reasoning Summary

WHEN an agent reasoning chain is available (e.g., chain-of-thought), the Telemetry_Emitter SHALL log a structured summary of the reasoning steps alongside the action event.

**Normative rules:**

1. IF the agent's platform or runtime provides access to the agent's reasoning chain (chain-of-thought, step-by-step reasoning, tool selection rationale), the Telemetry_Emitter SHALL capture and log a structured summary.
2. The reasoning chain summary SHALL be stored in the `eaagf.reasoning.chain_summary` span attribute as a JSON-encoded string.
3. The reasoning chain summary SHALL include the following structure:

```json
{
  "reasoning_steps": [
    {
      "step_index": 1,
      "step_type": "OBSERVATION | REASONING | TOOL_SELECTION | ACTION_PLAN | DECISION",
      "summary": "Brief description of the reasoning step",
      "confidence": 0.95,
      "timestamp": "ISO8601 UTC"
    }
  ],
  "total_steps": 3,
  "final_decision": "Description of the agent's final decision",
  "model_id": "string (optional — the model identifier if available)"
}
```

4. The reasoning chain summary SHALL be a condensed representation, NOT a verbatim transcript of the agent's internal reasoning. Conforming implementations SHALL limit the summary to a maximum of 4,096 characters to prevent excessive storage consumption.
5. The reasoning chain summary SHALL be subject to the same PII redaction rules as other audit event fields. Sensitive data appearing in reasoning steps SHALL be masked before logging.
6. Reasoning chain logging is REQUIRED for T3 and T4 agents when the reasoning chain is available from the platform. For T1 and T2 agents, reasoning chain logging is RECOMMENDED but OPTIONAL.
7. IF the agent's platform does not expose reasoning chain data, the `eaagf.reasoning.chain_summary` attribute SHALL be omitted from the audit event. The absence of this attribute SHALL NOT be treated as a conformance violation.
8. The reasoning chain summary SHALL be included in the same OTLP span as the action event. It SHALL NOT be emitted as a separate event.

> **Validates: Requirement 4.11** — WHEN an agent reasoning chain is available (e.g., chain-of-thought), THE Telemetry_Emitter SHALL log a structured summary of the reasoning steps alongside the action event.

---

## 15. Verifiable Action Receipts

### 15.1 Cryptographically Signed Action Receipts

WHEN a T3 or T4 agent completes a consequential action (write operation, external connection, agent delegation, or any action performed under a Constrained_Delegation_Mandate), the Governance_Controller SHALL produce a Verifiable_Action_Receipt — a cryptographically signed, tamper-evident record that binds the agent's identity, the authorizing human's identity, and the action outcome into a single non-repudiable artifact.

**Normative rules:**

1. A Verifiable_Action_Receipt is a structured JSON object signed by the Governance_Controller's signing key. It serves as deterministic proof that a specific action was authorized, executed, and recorded under governance controls. Unlike audit events (which are observability records), receipts are portable evidence artifacts that can be shared with external parties, regulators, or dispute resolution processes.
2. The Governance_Controller SHALL produce a Verifiable_Action_Receipt for the following action types when performed by T3 or T4 agents:
   - Write operations on Confidential or Restricted data.
   - External connections to non-enterprise endpoints.
   - Agent delegations via A2A protocol.
   - Any action performed under a Constrained_Delegation_Mandate.
3. A Verifiable_Action_Receipt SHALL contain the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `receipt_id` | UUID v4 | REQUIRED | Unique identifier for this receipt |
| `agent_id` | UUID v4 | REQUIRED | The agent that performed the action |
| `agent_risk_tier` | enum | REQUIRED | The agent's Risk_Tier at time of action |
| `action_type` | enum | REQUIRED | The type of action performed |
| `target_resource` | string | REQUIRED | The URI of the target resource |
| `action_outcome` | enum | REQUIRED | PERMITTED, GATED_APPROVED |
| `authorizer` | object | REQUIRED | Identity of the authorizing party: `type` (HUMAN_GATE, MANDATE, POLICY_AUTO), `identity` (human identity or mandate_id), `authorization_timestamp` |
| `task_correlation_id` | UUID v4 | REQUIRED | The EAAGF correlation ID for end-to-end tracing |
| `mandate_id` | UUID v4 | OPTIONAL | The Constrained_Delegation_Mandate ID, if the action was performed under a mandate |
| `intent_description` | string | OPTIONAL | The mandate's intent_description, if applicable |
| `data_classification` | enum | OPTIONAL | The data classification of the target resource |
| `timestamp` | ISO 8601 | REQUIRED | UTC timestamp of action completion |
| `governance_controller_id` | string | REQUIRED | Identifier of the Governance_Controller instance that produced the receipt |
| `signature` | string | REQUIRED | Cryptographic signature (RS256 or ES256) over all fields except `signature` |

4. The Governance_Controller SHALL sign each receipt using a dedicated signing key managed by the enterprise PKI. The signing key SHALL be rotated at minimum every 90 days. The public key SHALL be published in the enterprise trust registry so that receipt signatures can be independently verified.
5. Verifiable_Action_Receipts SHALL be stored alongside the corresponding audit event in the Audit_Log. The receipt SHALL be linked to the audit event via the `task_correlation_id` and `receipt_id`.
6. Verifiable_Action_Receipts SHALL be queryable via the Query API (Section 11) using the `receipt_id`, `agent_id`, `mandate_id`, or `task_correlation_id` as filter dimensions.
7. The `authorizer` field provides non-repudiable evidence of who authorized the action:
   - `type: HUMAN_GATE` — A human approved the action via a Human_Oversight_Gate. The `identity` field contains the approver's enterprise identity.
   - `type: MANDATE` — The action was authorized by a Constrained_Delegation_Mandate. The `identity` field contains the mandate_id, and the `mandate_id` and `intent_description` fields provide the full mandate context.
   - `type: POLICY_AUTO` — The action was authorized by the Policy_Engine without human intervention (FULL_AUTO or SUPERVISED mode for non-gated actions). The `identity` field contains the policy version that authorized the action.
8. Verifiable_Action_Receipts are designed to support dispute resolution and regulatory evidence scenarios. When a question arises about whether an agent was authorized to perform a specific action, the receipt provides a self-contained, cryptographically verifiable answer that does not require access to the full Audit_Log.
9. Receipt generation SHALL be synchronous with the action completion. The receipt MUST be produced before the action result is returned to the agent. Receipt generation latency SHALL NOT exceed 100 milliseconds.
10. Conforming implementations MAY extend the receipt schema with additional fields for domain-specific evidence (e.g., financial transaction references, data lineage hashes), provided the core fields defined above are always present.

> **Validates: Requirement 4.12** — WHEN a T3 or T4 agent completes a consequential action, THE Governance_Controller SHALL produce a Verifiable_Action_Receipt that cryptographically binds the agent's identity, the authorizing party, and the action outcome into a non-repudiable artifact.

### 15.2 Verifiable_Action_Receipt Schema (JSON)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://eaagf.enterprise.com/schemas/verifiable-action-receipt/v1",
  "title": "EAAGF Verifiable Action Receipt",
  "description": "A cryptographically signed, tamper-evident record binding agent identity, authorization, and action outcome.",
  "type": "object",
  "required": [
    "receipt_id", "agent_id", "agent_risk_tier", "action_type",
    "target_resource", "action_outcome", "authorizer",
    "task_correlation_id", "timestamp", "governance_controller_id", "signature"
  ],
  "properties": {
    "receipt_id": { "type": "string", "format": "uuid" },
    "agent_id": { "type": "string", "format": "uuid" },
    "agent_risk_tier": { "type": "string", "enum": ["T1", "T2", "T3", "T4"] },
    "action_type": { "type": "string", "enum": ["TOOL_CALL", "DATA_ACCESS", "AGENT_DELEGATION", "EXTERNAL_CONNECTION"] },
    "target_resource": { "type": "string" },
    "action_outcome": { "type": "string", "enum": ["PERMITTED", "GATED_APPROVED"] },
    "authorizer": {
      "type": "object",
      "required": ["type", "identity", "authorization_timestamp"],
      "properties": {
        "type": { "type": "string", "enum": ["HUMAN_GATE", "MANDATE", "POLICY_AUTO"] },
        "identity": { "type": "string" },
        "authorization_timestamp": { "type": "string", "format": "date-time" }
      }
    },
    "task_correlation_id": { "type": "string", "format": "uuid" },
    "mandate_id": { "type": "string", "format": "uuid" },
    "intent_description": { "type": "string", "maxLength": 4096 },
    "data_classification": { "type": "string", "enum": ["PUBLIC", "INTERNAL", "CONFIDENTIAL", "RESTRICTED"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "governance_controller_id": { "type": "string" },
    "signature": { "type": "string" }
  }
}
```

---

## 16. Observability Event Catalog

The following table catalogs all audit events defined by the observability standard. Events from other governance domains that flow through the Telemetry_Emitter are also listed for completeness.

| Event Type | Trigger | Source Domain | Required Attributes |
|---|---|---|---|
| `ACTION_PERMITTED` | Agent action authorized | Authorization | agent_id, action_type, target, risk_tier, correlation_id, timestamp |
| `ACTION_BLOCKED` | Agent action denied | Authorization | agent_id, action_type, target, risk_tier, reason_code, correlation_id, timestamp |
| `GATE_TRIGGERED` | Human_Oversight_Gate activated | Oversight | agent_id, gate_id, action_type, target, risk_tier, trigger_time, correlation_id |
| `GATE_RESOLVED` | Gate approved or rejected | Oversight | gate_id, approver, decision, decision_time, correlation_id |
| `GATE_TIMEOUT_ESCALATION` | Gate timeout, escalated | Oversight | gate_id, original_approver, escalation_target, timeout_duration, correlation_id |
| `SESSION_CREDENTIAL_ISSUED` | Scope-bound credential issued | Authorization | agent_id, credential_id, scope, ttl_seconds, timestamp |
| `SESSION_CREDENTIAL_REVOKED` | Session credential revoked | Authorization | agent_id, credential_id, task_id, revocation_reason, timestamp |
| `AGENT_REGISTERED` | New agent registered | Identity | agent_id, name, owning_team, platform, risk_tier, timestamp |
| `CREDENTIAL_ROTATED` | Agent credential auto-rotated | Identity | agent_id, old_credential_id, new_credential_id, timestamp |
| `PROMPT_INJECTION_DETECTED` | Prompt injection blocked | Security | agent_id, injection_score, input_hash, timestamp |
| `OUTPUT_VALIDATION_FAILURE` | Agent output failed schema validation | Security | agent_id, validation_errors, timestamp |
| `DATA_EXFILTRATION_ATTEMPT` | Restricted data transfer blocked | Data Governance | agent_id, source_resource, target, data_classification, timestamp |
| `EMERGENCY_STOP` | Agent emergency stopped | Oversight | agent_id, stopped_by, reason, timestamp |
| `POLICY_UPDATED` | Permission policy hot-reloaded | Authorization | policy_type, previous_version, new_version, timestamp |
| `COMPLIANCE_DRIFT` | Agent compliance posture degraded | Compliance | agent_id, drift_type, details, timestamp |
| `TELEMETRY_WAL_FULL` | WAL buffer capacity reached | Observability | wal_capacity, events_buffered, timestamp |
| `AUDIT_INTEGRITY_VIOLATION` | Tamper attempt detected | Observability | affected_records, detection_method, timestamp |

---

## 17. Conformance Requirements

### 17.1 Implementation Requirements

Any conforming implementation of the EAAGF observability standard MUST satisfy the following:

1. The implementation SHALL emit an audit event for every agent action with all REQUIRED attributes defined in Section 5, as specified in Section 3.
2. The implementation SHALL produce telemetry in OTLP format with support for OTLP/gRPC and OTLP/HTTP transports, as specified in Section 4.
3. The implementation SHALL emit BLOCKED events within 500 milliseconds of the block decision, as specified in Section 6.
4. The implementation SHALL enforce Audit_Log immutability with tamper-evidence mechanisms, as specified in Section 7.
5. The implementation SHALL retain all audit records for a minimum of 7 years with tiered storage, as specified in Section 8.
6. The implementation SHALL emit gate lifecycle events (GATE_TRIGGERED, GATE_RESOLVED) with all required fields, as specified in Section 9.
7. The implementation SHALL include a correlation ID in all events within a single task execution, as specified in Section 10.
8. The implementation SHALL provide a Query API supporting multi-dimensional filtering, as specified in Section 11.
9. The implementation SHALL buffer events in a durable WAL (minimum 10,000 events) when the telemetry backend is unavailable, as specified in Section 12.
10. The implementation SHALL support SIEM integration via Kafka, Azure Event Hub, AWS Kinesis, and webhook, as specified in Section 13.
11. The implementation SHALL log structured reasoning chain summaries for T3/T4 agents when available, as specified in Section 14.
12. The implementation SHALL produce Verifiable_Action_Receipts for T3/T4 consequential actions with cryptographic signatures, as specified in Section 15.

### 17.2 Conformance Test Cases

The Conformance_Test_Suite SHALL include the following test cases for the observability domain:

| Test ID | Description | Validates |
|---|---|---|
| OBS-001 | Verify audit event emitted for every agent action with all required fields | Section 3 |
| OBS-002 | Verify events are emitted in OTLP format and ingestible by an OTEL collector | Section 4 |
| OBS-003 | Verify BLOCKED event emitted within 500ms of block decision | Section 6 |
| OBS-004 | Verify audit records cannot be modified or deleted after write | Section 7 |
| OBS-005 | Verify audit records are retained and queryable for 7 years | Section 8 |
| OBS-006 | Verify GATE_TRIGGERED and GATE_RESOLVED events contain all required fields | Section 9 |
| OBS-007 | Verify all events in a task execution share the same correlation ID | Section 10 |
| OBS-008 | Verify Query API returns correct results for each filter dimension | Section 11 |
| OBS-009 | Verify events are buffered in WAL during backend outage and replayed on recovery | Section 12 |
| OBS-010 | Verify events are published to configured SIEM destinations (Kafka, Event Hub, Kinesis) | Section 13 |
| OBS-011 | Verify reasoning chain summary is logged for T3/T4 agents when available | Section 14 |
| OBS-012 | Verify Verifiable_Action_Receipt is produced for T3/T4 consequential actions with valid signature | Section 15 |
| OBS-013 | Verify receipts are queryable via Query API by receipt_id, agent_id, mandate_id, and correlation_id | Section 15 |

---

## 18. Regulatory Alignment

### 18.1 EU AI Act Mapping

| EAAGF Control | EU AI Act Article | Alignment |
|---|---|---|
| Audit event emission (Section 3) | Article 12 — Record-keeping | Comprehensive logging of all agent actions satisfies record-keeping obligations |
| Immutability (Section 7) | Article 12 — Record-keeping | Tamper-evident logs ensure record integrity for regulatory inspection |
| 7-year retention (Section 8) | Article 12 — Record-keeping | Retention period exceeds EU AI Act minimum requirements |
| Reasoning chain logging (Section 14) | Article 13 — Transparency | Structured reasoning summaries support transparency obligations for high-risk AI |
| Query API (Section 11) | Article 13 — Transparency | Enables authorized access to agent decision records |

### 18.2 NIST AI RMF Mapping

| EAAGF Control | NIST AI RMF Function | Alignment |
|---|---|---|
| OTLP event format (Section 4) | MEASURE 2.5 — Monitoring and evaluation | Standardized telemetry enables consistent monitoring across platforms |
| Correlation ID (Section 10) | MEASURE 2.5 — Monitoring and evaluation | End-to-end tracing supports comprehensive evaluation of agent behavior |
| SIEM integration (Section 13) | MANAGE 4.1 — Risk treatment and response | Real-time event streaming enables automated incident response |
| Telemetry buffering (Section 12) | GOVERN 1.1 — Legal and regulatory requirements | Resilient event capture ensures no audit gaps during outages |

### 18.3 ISO 42001 Mapping

| EAAGF Control | ISO 42001 Clause | Alignment |
|---|---|---|
| Audit event emission (Section 3) | Clause 9.1 — Monitoring, measurement, analysis, and evaluation | Provides the measurement data required for AI system evaluation |
| Query API (Section 11) | Clause 9.2 — Internal audit | Enables internal audit teams to query and analyze agent behavior |
| 7-year retention (Section 8) | Clause 7.5 — Documented information | Satisfies documented information retention requirements |
| Immutability (Section 7) | Clause 9.2 — Internal audit | Ensures audit evidence integrity for ISO certification |

For the complete control-by-control compliance mapping, see the [Compliance Mapping Registry](../reference/compliance-mapping-registry.md).
