# EAAGF Governance Standard Specification

## Overview

The Enterprise AI Agent Governance Framework (EAAGF) is a protocol-level governance standard — analogous to CNI, CSI, and MCP — that defines *how* AI agents are governed across the enterprise, independent of the platform they run on. This document serves as the normative specification that all product teams, platform owners, and AI governance stakeholders reference when building, deploying, and operating AI agents.

This specification does not prescribe a specific software implementation. Instead, it defines:

1. **Governance Protocols** — The rules, decision flows, and enforcement points that govern agent behavior
2. **Data Schemas** — The canonical data models for agent identity, classification, conformance, and audit
3. **Process Flows** — The step-by-step governance processes for registration, authorization, oversight, and lifecycle management
4. **Conformance Criteria** — The verifiable requirements that platforms and agents must satisfy to be EAAGF-compliant
5. **Compliance Mappings** — The linkage between EAAGF controls and EU AI Act, NIST AI RMF, and ISO 42001 obligations

The framework is owned and enforced by the AI Governance Team. It establishes binding standards that all product teams must conform to, with tiered oversight requirements based on agent risk classification.

### Scope

This specification covers governance of AI agents across seven enterprise platforms:
- Databricks
- Salesforce AgentForce
- Snowflake Cortex
- Microsoft Copilot Studio
- AWS Bedrock
- Azure AI Foundry
- GCP Vertex AI

### Audience

- AI Governance Team members
- Platform engineering teams
- Product teams deploying AI agents
- Security and compliance teams
- External auditors

---

## Architecture

### Governance Architecture Overview

The EAAGF governance architecture is structured in three layers. Each layer defines a set of responsibilities and interfaces that must be satisfied by any conforming implementation.

```mermaid
graph TB
    subgraph "Enterprise AI Agent Governance Framework"
        subgraph "Protocol Layer"
            SPEC[EAAGF Specification<br/>Schemas · Conformance Criteria · Protocols]
        end

        subgraph "Governance Layer"
            GC[Governance Controller<br/>Policy Enforcement Point]
            AR[Agent Registry<br/>Identity & Lifecycle Authority]
            PE[Policy Engine<br/>Authorization Decisions]
            TE[Telemetry Emitter<br/>Audit & Observability]
            HOG[Human Oversight Gate<br/>Approval Workflows]
        end

        subgraph "Platform Integration Layer"
            PA_DB[Databricks Adapter]
            PA_SF[Salesforce AgentForce Adapter]
            PA_SN[Snowflake Cortex Adapter]
            PA_MS[Copilot Studio Adapter]
            PA_AWS[AWS Bedrock Adapter]
            PA_AZ[Azure AI Foundry Adapter]
            PA_GCP[GCP Vertex AI Adapter]
        end
    end

    subgraph "Enterprise Infrastructure"
        IDP[Identity Provider<br/>Azure AD · Okta · LDAP]
        SIEM[SIEM / SOAR<br/>Splunk · Sentinel · Chronicle]
        CICD[CI/CD<br/>GitHub Actions · Azure DevOps]
        DC[Data Catalog<br/>Atlas · Collibra · Alation]
        OTEL[Observability Backend<br/>Datadog · Splunk · CloudWatch]
        MQ[Message Queue<br/>Kafka · Event Hub · Kinesis]
    end

    subgraph "Agent Platforms"
        DB[Databricks]
        SF[Salesforce AgentForce]
        SN[Snowflake Cortex]
        MS[Microsoft Copilot Studio]
        AWS[AWS Bedrock]
        AZ[Azure AI Foundry]
        GCP[GCP Vertex AI]
    end

    DB --> PA_DB --> GC
    SF --> PA_SF --> GC
    SN --> PA_SN --> GC
    MS --> PA_MS --> GC
    AWS --> PA_AWS --> GC
    AZ --> PA_AZ --> GC
    GCP --> PA_GCP --> GC

    GC --> AR
    GC --> PE
    GC --> TE
    GC --> HOG

    AR --> IDP
    PE --> DC
    TE --> OTEL
    TE --> MQ
    MQ --> SIEM
    HOG --> SIEM
    CICD --> GC
```

### Governance Decision Flow

Every agent action passes through the following governance decision flow. This is the core enforcement pattern that all platform adapters must implement.

```mermaid
sequenceDiagram
    participant Agent
    participant PlatformAdapter as Platform Adapter
    participant GovController as Governance Controller
    participant PolicyEngine as Policy Engine
    participant HOGate as Human Oversight Gate
    participant TelemetryEmitter as Telemetry Emitter
    participant TargetSystem as Target System

    Agent->>PlatformAdapter: Action Request (tool call / data access / delegation)
    PlatformAdapter->>GovController: Normalized EAAGF Action Request
    GovController->>GovController: Validate Agent_Identity
    GovController->>PolicyEngine: Evaluate Policy (agent_id, action, resource, context)
    PolicyEngine-->>GovController: PERMIT / DENY / GATE

    alt DENY
        GovController->>TelemetryEmitter: Emit BLOCKED event
        GovController-->>PlatformAdapter: Denied + reason code
    else GATE (Human Oversight Required)
        GovController->>HOGate: Trigger approval workflow
        HOGate-->>GovController: APPROVED / REJECTED (async)
        GovController->>TelemetryEmitter: Emit GATE_TRIGGERED / GATE_RESOLVED event
    else PERMIT
        GovController->>TargetSystem: Execute action
        TargetSystem-->>GovController: Action result
        GovController->>TelemetryEmitter: Emit ACTION_COMPLETED event
        GovController-->>PlatformAdapter: Action result
    end

    PlatformAdapter-->>Agent: Response
```

---

## Governance Domains and Standards

### Domain 1: Agent Identity and Registration

The Agent Registry is the authoritative source of truth for all agent identities and lifecycle states. Every agent deployed in the enterprise MUST be registered before it can perform any governed action.

#### Agent Registration Flow

```mermaid
flowchart TD
    A[Team submits Agent Manifest] --> B{Manifest valid?}
    B -->|No| C[Return validation errors]
    B -->|Yes| D[Assign UUID v4 identifier]
    D --> E[Record agent metadata<br/>name, team, platform, capabilities]
    E --> F[Run Risk Classification]
    F --> G{Risk Tier assigned?}
    G -->|No| H[Block: CLASSIFICATION_REQUIRED]
    G -->|Yes| I[Issue Agent_Identity credential<br/>X.509 or OAuth 2.0]
    I --> J[Validate Conformance Profile<br/>against EAAGF schema]
    J --> K{Profile valid?}
    K -->|No| L[Return schema errors]
    K -->|Yes| M[Set lifecycle state: DEVELOPMENT]
    M --> N[Emit AGENT_REGISTERED audit event]
    N --> O[Agent registered successfully]
```

#### Agent Record Schema

```json
{
  "agent_id": "uuid-v4",
  "name": "string",
  "version": "semver",
  "owning_team": "string",
  "platform": "DATABRICKS | SALESFORCE | SNOWFLAKE | COPILOT_STUDIO | AWS | AZURE | GCP",
  "risk_tier": "T1 | T2 | T3 | T4",
  "lifecycle_state": "DEVELOPMENT | STAGING | PRODUCTION | DECOMMISSIONED",
  "conformance_profile": { "$ref": "#/ConformanceProfile" },
  "identity": {
    "credential_type": "X509 | OAUTH2",
    "credential_id": "string",
    "issued_at": "ISO8601",
    "expires_at": "ISO8601"
  },
  "created_at": "ISO8601",
  "created_by": "string",
  "last_modified_at": "ISO8601",
  "last_modified_by": "string",
  "compliance_flags": ["EU_AI_ACT_HIGH_RISK", "GDPR_SCOPED", "CCPA_SCOPED"]
}
```

#### Credential Lifecycle Flow

```mermaid
flowchart TD
    A[Credential Issued] --> B[Active]
    B --> C{80% TTL reached?}
    C -->|No| B
    C -->|Yes| D[Auto-rotate credential]
    D --> E[Issue new credential]
    E --> F[Revoke old credential]
    F --> B
    B --> G{Agent decommissioned?}
    G -->|Yes| H[Revoke all credentials<br/>within 60 seconds]
    H --> I[Emit CREDENTIAL_REVOKED event]
    B --> J{Task completed/timed out?}
    J -->|Yes| K[Revoke session credentials]
```

#### Conformance Profile Schema

```json
{
  "schema_version": "1.0",
  "agent_id": "uuid-v4",
  "capabilities": ["TOOL_CALL", "DATA_READ", "DATA_WRITE", "AGENT_DELEGATION", "EXTERNAL_CONNECTION"],
  "declared_permissions": [
    { "resource": "snowflake://db/schema/table", "actions": ["SELECT"] },
    { "resource": "salesforce://sobject/Account", "actions": ["READ", "UPDATE"] }
  ],
  "approved_mcp_servers": ["mcp://enterprise-catalog/salesforce-crm"],
  "approved_egress_endpoints": ["api.internal.company.com", "*.salesforce.com"],
  "data_classifications_accessed": ["INTERNAL", "CONFIDENTIAL"],
  "oversight_mode": "SUPERVISED | FULL_AUTO | APPROVAL_REQUIRED | HUMAN_IN_LOOP",
  "max_session_duration_seconds": 3600,
  "max_actions_per_minute": 100,
  "context_compartments": ["crm-context", "finance-context"],
  "geographic_constraints": ["EU", "US"],
  "protocols_supported": ["MCP_1_0", "A2A_1_0"],
  "disclosure_mode": "SESSION_START | PERIODIC | CONTINUOUS",
  "disclosure_interval_seconds": 300,
  "content_provenance_mode": "LABEL_ONLY | METADATA_ONLY | FULL"
}
```

---

### Domain 2: Risk Classification and Tiering

All agents MUST be classified into exactly one of four Risk Tiers based on three dimensions: autonomy level, data sensitivity, and action scope. The Risk Tier determines the governance controls applied.

#### Risk Classification Flow

```mermaid
flowchart TD
    A[Agent submitted for classification] --> B[Evaluate Autonomy Level]
    B --> C{Autonomy?}
    C -->|Human approves every action| D[Autonomy: LOW]
    C -->|Human-in-loop for writes| E[Autonomy: MEDIUM]
    C -->|Autonomous multi-step| F[Autonomy: HIGH]
    C -->|Fully autonomous| G[Autonomy: CRITICAL]

    D --> H[Evaluate Data Sensitivity]
    E --> H
    F --> H
    G --> H

    H --> I{Data Sensitivity?}
    I -->|Public / Internal| J[Data: LOW]
    I -->|Internal / Confidential| K[Data: MEDIUM]
    I -->|Confidential| L[Data: HIGH]
    I -->|Restricted| M[Data: CRITICAL]

    J --> N[Evaluate Action Scope]
    K --> N
    L --> N
    M --> N

    N --> O{Action Scope?}
    O -->|Read-only| P[Scope: LOW]
    O -->|Read + Write internal| Q[Scope: MEDIUM]
    O -->|Read + Write + Multi-step| R[Scope: HIGH]
    O -->|Write + External + Financial| S[Scope: CRITICAL]

    P --> T[Apply Classification Matrix]
    Q --> T
    R --> T
    S --> T

    T --> U{Assign Risk Tier}
    U --> V[T1: Informational]
    U --> W[T2: Transactional]
    U --> X[T3: Autonomous]
    U --> Y[T4: Critical]
```

#### Risk Tier Classification Matrix

| Dimension | T1 (Informational) | T2 (Transactional) | T3 (Autonomous) | T4 (Critical) |
|---|---|---|---|---|
| Autonomy | Human approves every action | Human-in-loop for writes | Autonomous multi-step | Fully autonomous |
| Data Sensitivity | Public / Internal | Internal / Confidential | Confidential | Restricted |
| Action Scope | Read-only | Read + Write (internal) | Read + Write + Multi-step | Write + External + Financial |
| Default Oversight Mode | HUMAN_IN_LOOP | SUPERVISED | APPROVAL_REQUIRED | APPROVAL_REQUIRED |
| Credential TTL | 1 hour | 1 hour | 15 minutes | 15 minutes |
| Re-validation Period | 180 days | 180 days | 90 days | 90 days |
| AI Governance Approval Required | No | No | Yes | Yes |
| EU AI Act High-Risk Candidate | No | Possible | Likely | Yes |

#### Multi-Platform Tier Resolution

When an agent spans multiple platforms, the assigned tier MUST equal the maximum tier across all platform contexts. For example, if an agent is T2 on Salesforce and T3 on Databricks, the agent is classified as T3.

---

### Domain 3: Authorization and Least Privilege

The Policy Engine evaluates every agent action request against governance policies using an attribute-based access control (ABAC) model.

#### Authorization Decision Flow

```mermaid
flowchart TD
    A[Action Request received] --> B{Agent_Identity valid?}
    B -->|No| C[DENY: IDENTITY_UNREGISTERED]
    B -->|Yes| D{Permission declared<br/>in Conformance Profile?}
    D -->|No| E[DENY: PERMISSION_NOT_DECLARED]
    D -->|Yes| F{Resource in declared<br/>Context_Compartment?}
    F -->|No| G[DENY: COMPARTMENT_VIOLATION]
    F -->|Yes| H{Egress endpoint<br/>on allowlist?}
    H -->|No| I[DENY: EGRESS_NOT_ALLOWED]
    H -->|Yes| J{Rate limit<br/>exceeded?}
    J -->|Yes| K[DENY: RATE_LIMIT_EXCEEDED]
    J -->|No| L{T3/T4 write on<br/>restricted data?}
    L -->|Yes| M[GATE: Route to<br/>Human Oversight]
    L -->|No| N[PERMIT: Execute action]
```

#### Policy Evaluation Model

```
Decision = f(agent_attributes, action_attributes, resource_attributes, environment_attributes)

agent_attributes    = { agent_id, risk_tier, lifecycle_state, conformance_profile }
action_attributes   = { action_type, declared_in_profile, rate_within_limit }
resource_attributes = { classification, compartment, geographic_region }
environment_attributes = { time_of_day, active_incidents, platform }
```

#### Credential TTL Rules

| Risk Tier | Maximum Credential TTL | Session Behavior |
|---|---|---|
| T1 | 3600 seconds (1 hour) | Revoke on task completion |
| T2 | 3600 seconds (1 hour) | Revoke on task completion |
| T3 | 900 seconds (15 minutes) | Revoke on task completion or timeout |
| T4 | 900 seconds (15 minutes) | Revoke on task completion or timeout |

---

### Domain 4: Observability and Audit Trail

Every governance decision MUST produce a standardized audit event. The audit trail is immutable and retained for a minimum of 7 years.

#### Audit Event Schema (OTLP Span Attributes)

```json
{
  "eaagf.agent.id": "uuid",
  "eaagf.agent.risk_tier": "T1|T2|T3|T4",
  "eaagf.agent.platform": "DATABRICKS|SALESFORCE|...",
  "eaagf.action.type": "TOOL_CALL|DATA_ACCESS|AGENT_DELEGATION|EXTERNAL_CONNECTION",
  "eaagf.action.target": "resource URI",
  "eaagf.action.outcome": "PERMITTED|DENIED|GATED|BLOCKED",
  "eaagf.action.reason_code": "string",
  "eaagf.task.id": "uuid",
  "eaagf.task.correlation_id": "uuid",
  "eaagf.gate.id": "uuid (if gated)",
  "eaagf.gate.approver": "string (if gated)",
  "eaagf.gate.decision": "APPROVED|REJECTED (if gated)",
  "eaagf.data.classification": "PUBLIC|INTERNAL|CONFIDENTIAL|RESTRICTED",
  "eaagf.security.prompt_injection_score": "float",
  "eaagf.compliance.eu_ai_act_applicable": "bool",
  "timestamp": "ISO8601 UTC"
}
```

---

### Domain 5: Human Oversight Controls

The framework defines four oversight modes and a gate-based approval workflow for high-stakes agent decisions.

#### Oversight Modes

| Mode | Description | Default For |
|---|---|---|
| FULL_AUTO | No human gates | — (requires explicit authorization) |
| SUPERVISED | Gates on write operations | T2 agents |
| APPROVAL_REQUIRED | Gates on all non-trivial actions | T3, T4 agents |
| HUMAN_IN_LOOP | Human approves every action | T1 agents |

#### Human Oversight Gate Workflow

```mermaid
stateDiagram-v2
    [*] --> PENDING : Gate triggered
    PENDING --> APPROVED : Approver approves
    PENDING --> REJECTED : Approver rejects
    PENDING --> ESCALATED : Timeout (default 4h)
    ESCALATED --> APPROVED : Secondary approver approves
    ESCALATED --> REJECTED : Secondary approver rejects
    ESCALATED --> AUTO_REJECTED : Secondary timeout
    APPROVED --> [*]
    REJECTED --> [*]
    AUTO_REJECTED --> [*]
```

#### Human Oversight Decision Flow

```mermaid
flowchart TD
    A[Gate Triggered] --> B[Notify designated approver<br/>via configured channel<br/>within 30 seconds]
    B --> C{Response received?}
    C -->|Approved| D[Resume agent execution]
    C -->|Rejected| E[Block action, emit GATE_REJECTED]
    C -->|No response within timeout| F[Escalate to secondary approver]
    F --> G{Secondary response?}
    G -->|Approved| D
    G -->|Rejected| E
    G -->|No response| H[AUTO_REJECT, emit GATE_TIMEOUT]
```

---

### Domain 6: Interoperability Standards

All agents MUST communicate using MCP (Model Context Protocol) for tool connections and A2A (Agent-to-Agent Protocol) for peer delegation. Platform adapters bridge platform-native APIs to EAAGF-compliant interfaces.

#### Platform Adapter Compatibility Matrix

| Platform | Integration Point | MCP Support | A2A Support |
|---|---|---|---|
| Databricks | MLflow Model Serving, Unity Catalog | Via compatibility shim | Via compatibility shim |
| Salesforce AgentForce | AgentForce Command Center API | Native | Native |
| Snowflake Cortex | Cortex Agent API, Snowpark | Via compatibility shim | Via compatibility shim |
| Microsoft Copilot Studio | Power Platform connectors | Native | Via compatibility shim |
| AWS Bedrock | Bedrock Agents API, Lambda | Via compatibility shim | Via compatibility shim |
| Azure AI Foundry | AI Foundry SDK, Azure API Management | Native | Via compatibility shim |
| GCP Vertex AI | Vertex AI Agent Builder API | Via compatibility shim | Via compatibility shim |

#### Agent-to-Agent Delegation Flow

```mermaid
flowchart TD
    A[Agent A initiates delegation] --> B{A2A protocol used?}
    B -->|No| C[DENY: Protocol violation]
    B -->|Yes| D{Signed Agent Card present?}
    D -->|No| E[DENY: Missing Agent Card]
    D -->|Yes| F{Card agent_id matches<br/>registered record?}
    F -->|No| G[DENY: Identity mismatch]
    F -->|Yes| H{Combined Risk_Tier<br/>within task authorization?}
    H -->|No| I[DENY: Tier exceeded]
    H -->|Yes| J[PERMIT delegation]
    J --> K[Emit A2A_DELEGATION audit event]
```

#### MCP Enterprise Directory Entry Schema

```json
{
  "mcp_server_id": "mcp://enterprise-catalog/salesforce-crm",
  "display_name": "Salesforce CRM MCP Server",
  "provider": "Salesforce",
  "version": "3.1.0",
  "security_attestation": {
    "last_reviewed": "2025-01-15",
    "reviewed_by": "AI Governance Team",
    "vulnerability_scan_passed": true,
    "data_classification_max": "CONFIDENTIAL"
  },
  "approved_risk_tiers": ["T1", "T2", "T3"],
  "capabilities": ["account_read", "opportunity_read", "case_create"],
  "endpoint": "https://mcp.salesforce.internal.company.com"
}
```

---

### Domain 7: Data Governance and Privacy

Agents MUST respect enterprise data classification policies. Sensitive data MUST be compartmentalized within task contexts.

#### Data Governance Decision Flow

```mermaid
flowchart TD
    A[Agent requests data access] --> B[Resolve data classification<br/>from enterprise data catalog]
    B --> C{Classification level?}
    C -->|Public / Internal| D[Standard access controls]
    C -->|Confidential| E[Create Context_Compartment]
    C -->|Restricted| F[Create Context_Compartment<br/>+ enhanced controls]

    E --> G[Grant access within compartment]
    F --> G

    G --> H{Cross-scope transfer<br/>attempted?}
    H -->|Yes, Restricted data| I[DENY: DATA_EXFILTRATION_ATTEMPT]
    H -->|No| J[Continue execution]

    J --> K{Task complete?}
    K -->|Yes| L[Purge compartment<br/>within 60 seconds]
    L --> M[Record data lineage]

    D --> N{PII detected?}
    N -->|Yes| O[Mask PII in audit logs]
    N -->|No| P[Standard audit logging]

    G --> N
```

#### Data Classification Controls

| Classification | Compartment Required | Cross-Scope Transfer | PII Masking | Geographic Constraints |
|---|---|---|---|---|
| Public | No | Allowed | If PII detected | None |
| Internal | No | Allowed | If PII detected | None |
| Confidential | Yes | Restricted | Required | If GDPR/CCPA scoped |
| Restricted | Yes | Blocked | Required | Enforced |

---

### Domain 8: Security Controls

The framework defines defense-in-depth security controls applied at agent action boundaries.

#### Security Control Chain

```mermaid
flowchart TD
    A[Incoming Input] --> B[Prompt Injection Classifier]
    B --> C{Score > threshold?}
    C -->|Yes| D[BLOCK: PROMPT_INJECTION_DETECTED<br/>Quarantine for review]
    C -->|No| E[Content Filter<br/>T3/T4 agents only]
    E --> F{Harmful/biased content?}
    F -->|Yes| G[BLOCK: Content policy violation]
    F -->|No| H[Agent processes input]

    H --> I[Agent produces output]
    I --> J[Output Schema Validation]
    J --> K{Schema valid?}
    K -->|No| L[BLOCK: OUTPUT_VALIDATION_FAILURE]
    K -->|Yes| M[Content Filter<br/>T3/T4 agents only]
    M --> N{Policy violation?}
    N -->|Yes| O[BLOCK: Content policy violation]
    N -->|No| P[Deliver output]
```

#### Rate Limiting Rules

| Risk Tier | Max Actions/Minute | Breach Action |
|---|---|---|
| T1 | 100 | Throttle + RATE_LIMIT_EXCEEDED event |
| T2 | 100 | Throttle + RATE_LIMIT_EXCEEDED event |
| T3 | 20 | Throttle + RATE_LIMIT_EXCEEDED event |
| T4 | 20 | Throttle + RATE_LIMIT_EXCEEDED event |

#### Self-Modification Prevention

Any agent action targeting its own Conformance_Profile, Risk_Tier, oversight mode, or governance controls MUST be denied with reason code SELF_MODIFICATION_ATTEMPT.

---

### Domain 9: Compliance and Regulatory Alignment

The framework provides built-in mappings to EU AI Act, NIST AI RMF, and ISO 42001.

#### Compliance Mapping Registry Schema

```json
{
  "control_id": "EAAGF-AUTH-001",
  "control_name": "Least Privilege Authorization",
  "requirement_refs": ["3.1", "3.2", "3.3"],
  "regulatory_mappings": {
    "eu_ai_act": ["Article 9 - Risk Management", "Article 13 - Transparency"],
    "nist_ai_rmf": ["GOVERN 1.1", "MANAGE 2.2", "MEASURE 2.5"],
    "iso_42001": ["Clause 6.1 - Risk Assessment", "Clause 8.4 - AI System Operation"]
  },
  "evidence_sources": ["policy_engine_logs", "credential_issuance_records", "permission_denial_events"]
}
```

---

### Domain 10: Agent Lifecycle Management

All agents follow a four-stage lifecycle with governance gates at each transition.

#### Agent Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> DEVELOPMENT : Agent registered
    DEVELOPMENT --> STAGING : Conformance profile complete<br/>+ tests pass + Risk_Tier assigned
    STAGING --> PRODUCTION : Security scan + data governance review<br/>+ (T3/T4: AI Gov approval)
    PRODUCTION --> STAGING : Re-validation overdue<br/>OR compliance drift
    PRODUCTION --> DECOMMISSIONED : Decommission authorized<br/>+ dependencies migrated
    STAGING --> DECOMMISSIONED : Decommission authorized
    DEVELOPMENT --> DECOMMISSIONED : Decommission authorized
    DECOMMISSIONED --> [*] : Records retained 7 years
```

#### Lifecycle Transition Gates

| Transition | Prerequisites |
|---|---|
| DEVELOPMENT → STAGING | Completed Conformance_Profile, passing conformance tests, Risk_Tier assigned |
| STAGING → PRODUCTION | Security scan attestation, data governance review, AI Gov approval (T3/T4) |
| PRODUCTION → STAGING | Triggered by re-validation overdue or compliance drift |
| Any → DECOMMISSIONED | Decommission authorization, dependency migration verified (or override) |

#### Agent Lifecycle Management Flow

```mermaid
flowchart TD
    A[Agent in DEVELOPMENT] --> B{Conformance Profile<br/>complete?}
    B -->|No| C[Block transition]
    B -->|Yes| D{Conformance tests<br/>passing?}
    D -->|No| C
    D -->|Yes| E{Risk_Tier<br/>assigned?}
    E -->|No| C
    E -->|Yes| F[Transition to STAGING]

    F --> G{Security scan<br/>attestation?}
    G -->|No| H[Block transition]
    G -->|Yes| I{Data governance<br/>review complete?}
    I -->|No| H
    I -->|Yes| J{T3/T4 agent?}
    J -->|Yes| K{AI Gov Team<br/>approval?}
    K -->|No| H
    K -->|Yes| L[Transition to PRODUCTION]
    J -->|No| L

    L --> M{Re-validation<br/>overdue?}
    M -->|Yes| N[Downgrade to STAGING]
    M -->|No| O{Decommission<br/>requested?}
    O -->|Yes| P{Dependencies<br/>migrated?}
    P -->|No| Q{Override<br/>authorized?}
    Q -->|No| R[Block decommission]
    Q -->|Yes| S[Transition to DECOMMISSIONED]
    P -->|Yes| S
```

#### Incident Response Flow

```mermaid
flowchart TD
    A[Incident declared] --> B[Collect evidence bundle]
    B --> C[Audit logs for incident window]
    B --> D[Telemetry snapshots]
    B --> E[Configuration state]
    B --> F[Credential issuance records]

    C --> G[Package evidence bundle]
    D --> G
    E --> G
    F --> G

    G --> H{Risk Tier?}
    H -->|T1/T2| I[Follow T1/T2 runbook<br/>Team-level response]
    H -->|T3| J[Follow T3 runbook<br/>AI Gov Team notified]
    H -->|T4| K[Follow T4 runbook<br/>Emergency response<br/>AI Gov Team leads]

    I --> L[Document resolution]
    J --> L
    K --> L
    L --> M[Update compliance records]
```

---

### Domain 11: Transparency and AI Disclosure Controls

Agents MUST disclose their AI nature to users. The Governance_Controller enforces disclosure requirements as an output-level control, analogous to output schema validation. Disclosure cadence is configured per agent via the `disclosure_mode` field in the Conformance_Profile.

#### Disclosure Mode Configuration

| Disclosure Mode | Behavior | Default For |
|---|---|---|
| SESSION_START | Disclosure at session initiation only | T1, T2 agents |
| PERIODIC | Disclosure at session start + configurable intervals | T3 agents |
| CONTINUOUS | Disclosure attached to every output | T4 agents, AI companions |

#### Disclosure Enforcement Flow

```mermaid
flowchart TD
    A[Agent produces output] --> B{Disclosure required?}
    B --> C[Check disclosure_mode<br/>from Conformance_Profile]
    C --> D{Mode?}
    D -->|SESSION_START| E{First output<br/>in session?}
    E -->|Yes| F[Inject disclosure notification]
    E -->|No| G[Pass through]
    D -->|PERIODIC| H{Time since last<br/>disclosure > interval?}
    H -->|Yes| F
    H -->|No| G
    D -->|CONTINUOUS| F

    F --> I{User is minor?}
    I -->|Yes| J[Apply enhanced disclosure:<br/>simplified language +<br/>prominent visual indicator]
    I -->|No| K[Apply standard disclosure]

    J --> L{AI companion agent?}
    K --> L
    L -->|Yes| M[Prepend explicit disclaimer<br/>+ force CONTINUOUS mode]
    L -->|No| N[Proceed to output delivery]
    M --> N

    G --> N
    N --> O[Emit AI_DISCLOSURE_EVENT<br/>audit event]
    O --> P[Deliver output]

    B -->|Disclosure missing<br/>and required| Q[BLOCK: DISCLOSURE_MISSING]
    Q --> R[Emit OUTPUT_VALIDATION_FAILURE<br/>audit event]
```

#### AI Disclosure Event Schema

```json
{
  "eaagf.disclosure.type": "SESSION_START | PERIODIC | CONTINUOUS",
  "eaagf.disclosure.format": "TEXT | AUDIO | VISUAL",
  "eaagf.disclosure.enhanced": "bool (true if minor or AI companion)",
  "eaagf.disclosure.agent_id": "uuid",
  "eaagf.disclosure.session_id": "uuid",
  "eaagf.disclosure.interval_seconds": "int (configured interval)",
  "eaagf.disclosure.last_disclosure_at": "ISO8601",
  "timestamp": "ISO8601 UTC"
}
```

#### Disclosure Suppression Prevention

Any agent action that attempts to suppress, obscure, or misrepresent the AI nature of the agent MUST be denied with reason code SELF_MODIFICATION_ATTEMPT. This includes:
- Removing disclosure text from outputs
- Claiming to be human
- Instructing users to ignore disclosure notices

---

### Domain 12: Synthetic Content Identification and Provenance

All AI-generated content MUST carry provenance markers identifying it as synthetic. The Governance_Controller enforces provenance marking as an output-level control. Provenance strategy is configured per agent via the `content_provenance_mode` field in the Conformance_Profile.

#### Content Provenance Mode Configuration

| Provenance Mode | Explicit Marker | Implicit Marker (C2PA) | Default For |
|---|---|---|---|
| LABEL_ONLY | Yes (visible label) | No | T1 text-only agents |
| METADATA_ONLY | No | Yes (C2PA embedded) | Internal-use agents |
| FULL | Yes (visible label) | Yes (C2PA embedded) | T3/T4 agents, deepfake-relevant |

#### Provenance Enforcement Flow

```mermaid
flowchart TD
    A[Agent produces output] --> B[Determine content type<br/>text / image / audio / video]
    B --> C{content_provenance_mode<br/>from Conformance_Profile?}

    C -->|LABEL_ONLY| D[Apply explicit marker<br/>visible label/watermark]
    C -->|METADATA_ONLY| E[Embed C2PA metadata]
    C -->|FULL| F[Apply explicit marker<br/>+ embed C2PA metadata]

    D --> G{Visual/audio/video content?}
    E --> G
    F --> G

    G -->|Yes| H{Deepfake-relevant?}
    H -->|Yes| I[Force FULL mode<br/>regardless of profile setting]
    I --> J[Apply watermark + C2PA]
    H -->|No| K[Continue with configured mode]
    G -->|No| K

    J --> L[Validate provenance markers present]
    K --> L

    L --> M{Markers valid?}
    M -->|No| N[BLOCK: PROVENANCE_MISSING]
    N --> O[Emit OUTPUT_VALIDATION_FAILURE<br/>audit event]
    M -->|Yes| P[Record C2PA manifest<br/>in provenance registry]
    P --> Q[Emit PROVENANCE_APPLIED<br/>audit event]
    Q --> R[Deliver output]
```

#### Provenance Event Schema

```json
{
  "eaagf.provenance.marker_type": "EXPLICIT | IMPLICIT | BOTH",
  "eaagf.provenance.content_type": "TEXT | IMAGE | AUDIO | VIDEO | MULTIMODAL",
  "eaagf.provenance.c2pa_manifest_id": "string (C2PA manifest reference)",
  "eaagf.provenance.agent_id": "uuid",
  "eaagf.provenance.mode": "LABEL_ONLY | METADATA_ONLY | FULL",
  "eaagf.provenance.deepfake_relevant": "bool",
  "eaagf.provenance.forced_full_mode": "bool",
  "timestamp": "ISO8601 UTC"
}
```

#### Provenance Marker Integrity

Any agent action that attempts to remove, alter, or strip provenance markers from content MUST be denied with reason code SELF_MODIFICATION_ATTEMPT. This applies to:
- Markers on the agent's own outputs
- Markers on content received from other agents via A2A delegation
- Markers on content retrieved from external sources

#### Provenance Registry

The Governance_Controller maintains a provenance registry that stores the C2PA manifest for every piece of AI-generated content. This registry enables:
- Downstream verification of content origin
- Chain of custody tracking across agent delegations
- Regulatory audit of synthetic content production

```json
{
  "provenance_record_id": "uuid-v4",
  "agent_id": "uuid-v4",
  "content_hash": "SHA-256 hash of content",
  "c2pa_manifest": { "$ref": "#/C2PA_Manifest" },
  "content_type": "TEXT | IMAGE | AUDIO | VIDEO | MULTIMODAL",
  "created_at": "ISO8601",
  "platform": "DATABRICKS | SALESFORCE | SNOWFLAKE | COPILOT_STUDIO | AWS | AZURE | GCP",
  "downstream_consumers": ["agent_id_1", "agent_id_2"]
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

The following properties cover the two new governance domains (11 and 12). Properties for Domains 1–10 are covered by the existing specification.

### Property 1: Session-start disclosure is always present

*For any* agent session and any disclosure_mode setting (SESSION_START, PERIODIC, or CONTINUOUS), the first output of the session shall contain a disclosure notification.

**Validates: Requirements 11.1, 11.5**

### Property 2: Periodic disclosure cadence enforcement

*For any* agent with disclosure_mode PERIODIC and any sequence of outputs, if the elapsed time since the last disclosure exceeds the configured interval, the next output shall contain a disclosure notification.

**Validates: Requirements 11.2, 11.5**

### Property 3: Continuous disclosure completeness

*For any* agent with disclosure_mode CONTINUOUS, every output produced by the agent shall contain a disclosure indicator.

**Validates: Requirements 11.3, 11.5**

### Property 4: Minor enhanced disclosure enforcement

*For any* agent session where the user is identified as a minor, the disclosure notification shall include enhanced formatting (simplified language and prominent visual indicator), regardless of the base disclosure_mode.

**Validates: Requirements 11.4**

### Property 5: AI companion forced continuous mode

*For any* agent classified as an AI companion, the effective disclosure_mode shall be CONTINUOUS regardless of the Conformance_Profile setting, and every output shall be prepended with an explicit disclaimer.

**Validates: Requirements 11.9**

### Property 6: Disclosure blocking on missing notification

*For any* agent output that requires a disclosure notification (per the agent's disclosure_mode and elapsed time), if the disclosure is absent, the output shall be blocked and an OUTPUT_VALIDATION_FAILURE event with reason code DISCLOSURE_MISSING shall be emitted.

**Validates: Requirements 11.7**

### Property 7: Disclosure audit trail completeness

*For any* disclosure notification delivered to a user, a corresponding AI_DISCLOSURE_EVENT audit event shall be emitted containing the disclosure type, format, recipient context, and timestamp.

**Validates: Requirements 11.8**

### Property 8: Explicit provenance marker presence

*For any* agent output where content_provenance_mode is LABEL_ONLY or FULL, the output shall contain a visible provenance marker indicating AI origin.

**Validates: Requirements 12.1, 12.4**

### Property 9: Implicit provenance marker presence

*For any* agent output where content_provenance_mode is METADATA_ONLY or FULL, the output shall contain embedded C2PA metadata identifying AI origin.

**Validates: Requirements 12.2, 12.4**

### Property 10: Deepfake-relevant content forced full mode

*For any* agent generating visual, audio, or video content that could be mistaken for human-created content, the effective content_provenance_mode shall be FULL regardless of the Conformance_Profile setting, and both watermarking and C2PA metadata shall be applied.

**Validates: Requirements 12.3, 12.9**

### Property 11: Provenance blocking on missing markers

*For any* agent output that requires provenance markers (per the agent's content_provenance_mode), if the markers are absent, the output shall be blocked and an OUTPUT_VALIDATION_FAILURE event with reason code PROVENANCE_MISSING shall be emitted.

**Validates: Requirements 12.5**

### Property 12: Provenance marker immutability

*For any* agent action that attempts to remove, alter, or strip provenance markers from any content, the action shall be denied and a SELF_MODIFICATION_ATTEMPT security event shall be emitted.

**Validates: Requirements 12.6**

### Property 13: Provenance audit trail completeness

*For any* provenance marker application, a corresponding PROVENANCE_APPLIED audit event shall be emitted containing the marker type, content type, C2PA manifest reference, and timestamp.

**Validates: Requirements 12.7**

### Property 14: Provenance registry round-trip

*For any* AI-generated content with a C2PA manifest recorded in the provenance registry, querying the registry by content hash shall return the original C2PA manifest and agent origin.

**Validates: Requirements 12.10**

---

## Error Code Registry

| Code | Category | Description | Recovery |
|---|---|---|---|
| IDENTITY_UNREGISTERED | Identity | Agent has no valid registered identity | Register agent before use |
| PERMISSION_NOT_DECLARED | Authorization | Requested permission not in Conformance_Profile | Update Conformance_Profile |
| COMPARTMENT_VIOLATION | Authorization | Access to resource outside declared compartment | Declare compartment in profile |
| EGRESS_NOT_ALLOWED | Authorization | Outbound connection to non-allowlisted endpoint | Add endpoint to approved_egress_endpoints |
| CLASSIFICATION_REQUIRED | Lifecycle | Agent has no Risk_Tier assignment | Complete risk classification |
| UNAPPROVED_MCP_SERVER | Interoperability | MCP server not in enterprise directory | Request MCP server approval |
| PROMPT_INJECTION_DETECTED | Security | Input classified as prompt injection | Review and sanitize input |
| OUTPUT_VALIDATION_FAILURE | Security | Agent output failed schema validation | Review agent output logic |
| SELF_MODIFICATION_ATTEMPT | Security | Agent attempted to modify own governance controls | Governance controls are immutable by agents |
| DATA_EXFILTRATION_ATTEMPT | Data Governance | Restricted data transfer outside authorized scope | Review data handling logic |
| RATE_LIMIT_EXCEEDED | Security | Agent exceeded action rate limit | Reduce action frequency or request limit increase |
| PLAN_DEVIATION | Oversight | Agent action sequence deviated from declared plan | Human review required |
| COMPLIANCE_DRIFT | Compliance | Agent compliance posture has degraded | Remediate within 24 hours |
| GATE_TIMEOUT_ESCALATION | Oversight | Human gate not responded to within timeout | Secondary approver notified |
| EMERGENCY_STOPPED | Oversight | Agent has been emergency stopped | Requires AI Governance Team review |
| DISCLOSURE_MISSING | Transparency | Agent output missing required AI disclosure notification | Add disclosure to output or check disclosure_mode configuration |
| PROVENANCE_MISSING | Provenance | Agent output missing required content provenance markers | Apply provenance markers per content_provenance_mode |

---

## Agent Manifest Format

The canonical agent manifest format used for registration:

```yaml
apiVersion: eaagf/v1
kind: AgentManifest
metadata:
  name: "sales-forecast-agent"
  version: "1.2.0"
  owning_team: "revenue-analytics"
  platform: "DATABRICKS"
spec:
  risk_tier: "T2"
  capabilities:
    - TOOL_CALL
    - DATA_READ
    - DATA_WRITE
  declared_permissions:
    - resource: "snowflake://analytics/sales/forecast"
      actions: ["SELECT", "INSERT"]
    - resource: "salesforce://sobject/Opportunity"
      actions: ["READ"]
  approved_mcp_servers:
    - "mcp://enterprise-catalog/snowflake-query"
    - "mcp://enterprise-catalog/salesforce-crm"
  approved_egress_endpoints:
    - "api.internal.company.com"
  data_classifications_accessed:
    - "INTERNAL"
    - "CONFIDENTIAL"
  oversight_mode: "SUPERVISED"
  max_session_duration_seconds: 3600
  max_actions_per_minute: 100
  context_compartments:
    - "sales-analytics-context"
  geographic_constraints:
    - "US"
    - "EU"
  protocols_supported:
    - "MCP_1_0"
  disclosure_mode: "SESSION_START"
  disclosure_interval_seconds: 300
  content_provenance_mode: "LABEL_ONLY"
```
